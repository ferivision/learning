# Phase 15 — AI Observability & Evaluation

> Bagian dari [AI Engineering / Agent Roadmap](../README.md)

---

## 59. LLM Observability

### Apa itu?
LLM Observability adalah praktik mencatat (trace) SETIAP hal penting yang terjadi selama satu request SupportPilot diproses lewat LLM/agent — bukan cuma jawaban akhirnya, tapi juga **prompt** yang dikirim, **response** mentah yang diterima, **tokens** (input/output), **latency**, **cost**, **model** yang dipakai, **tool calls** (tool apa, argumen apa, hasil apa), **errors**, dan **retrieval results** (chunk apa yang di-retrieve, kalau ada RAG yang terlibat). Alurnya:
```
User → Agent → Trace → LLM call / Tool call / DB call / Retrieval
```
Trace ini bukan cuma satu log baris — dia adalah "rekaman lengkap" dari SATU eksekusi, yang nantinya jadi bahan mentah buat evaluation (topik 60-62) dan debugging kalau ada yang salah.

### Kenapa dibutuhkan?
Tanpa observability, begitu ada customer yang komplain "SupportPilot jawabnya aneh/salah", engineer cuma bisa nebak-nebak apa yang sebenarnya terjadi di balik layar — prompt apa yang dikirim? Model pilih tool apa? Kenapa jawabannya kayak gitu? Berapa biayanya? Ini makin parah begitu `run_agent_loop` (Phase 6, topik 25) punya beberapa step tool call sebelum jawaban final — tanpa trace, kita gak tau step ke berapa yang mulai "salah mikir". Observability menyelesaikan ini dengan mencatat SETIAP detail relevan dari SETIAP panggilan, supaya bisa direplay/diaudit belakangan kapan aja, bukan cuma diandalkan lewat print debugging manual yang cuma muncul sekali lalu hilang.

### Cara Kerja
```
Tanpa tracing:
  LLMGateway.generate(messages, model) dipanggil -> jawaban langsung dibalikin
      -> gak ada jejak APAPUN soal prompt/tokens/latency/cost yang dipakai

Dengan tracing (trace_llm_call sebagai decorator):
  LLMGateway.generate(messages, model) dipanggil
      -> trace_llm_call "membungkus" panggilan itu:
           - catat waktu mulai
           - jalankan fungsi asli SEPERTI BIASA (gak diubah perilakunya)
           - catat waktu selesai -> hitung latency
           - estimasi input/output tokens & cost dari prompt+response
           - simpan semuanya sebagai satu entry di TRACE_STORE
      -> jawaban tetap dibalikin ke pemanggil, TANPA berubah sama sekali
```
`trace_llm_call` di bawah adalah **decorator** — sintaks `@sesuatu` yang sama seperti `@app.post("/chat")` yang sudah dikenalin di Phase 5 topik 19 (Catatan Python di situ menjelaskan konsep dasarnya kalau belum familiar). Ini pemakaian KEDUA-nya di syllabus ini, jadi di sini gak diulang dari nol — bedanya, decorator ini gak dipakai buat mendaftarkan route HTTP, tapi buat "menyelipkan" logging observability di sekeliling fungsi LLM call APAPUN, tanpa mengubah isi fungsi itu sendiri.

### Contoh Kode — Python
`trace_llm_call` dibuktikan dulu pakai fungsi simulasi tanpa network (`fake_llm_call`), supaya perilakunya beneran bisa dijalankan dan dicek — sebelum dipasang ke `LLMGateway.generate` (Phase 5, topik 20) yang butuh API key sungguhan:
```python
import functools
import time

# Trace store sederhana: list of dict, disimpan in-memory. Di production
# beneran ini biasanya database/observability platform (Langfuse, LangSmith,
# dst) -- di sini list biasa dipakai supaya fokus ke KONSEPnya dulu, bukan ke
# infrastrukturnya.
TRACE_STORE: list[dict] = []

PRICE_PER_1K_TOKENS = {
    # Sama seperti Phase 5 topik 22, disederhanakan buat contoh di sini.
    "gpt-4o-mini": {"input": 0.00015, "output": 0.0006},
    "claude-3-5-sonnet-20241022": {"input": 0.003, "output": 0.015},
}


def _estimate_tokens(text: str) -> int:
    """
    Estimasi KASAR jumlah token dari panjang teks (bukan tokenizer asli model
    tertentu) -- lihat Trade-off & Pitfall soal batasannya.
    """
    if not text:
        return 0
    return max(1, round(len(text.split()) / 0.75))


def trace_llm_call(func):
    """
    Decorator generik buat membungkus fungsi LLM-calling APAPUN (di sini
    dipasang ke `LLMGateway.generate`, Phase 5 topik 20) supaya SETIAP
    pemanggilannya otomatis tercatat ke TRACE_STORE -- prompt, response,
    estimasi tokens & cost, latency, model, dan error kalau ada -- tanpa
    perlu nulis logging manual berulang-ulang di tiap tempat yang manggil
    fungsi itu.

    `functools.wraps(func)` di bawah PENTING: tanpa itu, `wrapper.__name__`
    dan `wrapper.__doc__` bakal jadi "wrapper" / kosong, bukan nama/docstring
    fungsi ASLI yang dibungkus -- itu bikin debugging dan introspeksi (misal
    `help(traced_fake_llm_call)`) jadi membingungkan. `functools.wraps`
    menyalin metadata fungsi asli ke wrapper-nya, jadi dari luar, fungsi yang
    sudah di-trace tetap "kelihatan" seperti fungsi aslinya.
    """
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        # messages & model diambil dari kwargs -- sesuai gaya pemanggilan
        # LLMGateway.generate di seluruh syllabus ini (selalu keyword,
        # lihat Phase 5 topik 20-22).
        messages = kwargs.get("messages", [])
        model = kwargs.get("model") or "unknown"
        prompt_text = messages[-1]["content"] if messages else ""

        mulai = time.perf_counter()
        error = None
        response_text = None
        try:
            response_text = func(*args, **kwargs)
            return response_text
        except Exception as e:
            # Tetap catat trace-nya walau fungsinya gagal, lalu re-raise
            # supaya pemanggil asli tetap tau ada error -- tracing gak boleh
            # "menyembunyikan" kegagalan.
            error = str(e)
            raise
        finally:
            selesai = time.perf_counter()
            input_tokens = _estimate_tokens(prompt_text)
            output_tokens = _estimate_tokens(response_text or "")
            harga = PRICE_PER_1K_TOKENS.get(model, {"input": 0.0, "output": 0.0})
            estimated_cost = (
                (input_tokens / 1000) * harga["input"]
                + (output_tokens / 1000) * harga["output"]
            )
            TRACE_STORE.append({
                "function": func.__name__,
                "model": model,
                "prompt": prompt_text,
                "response": response_text,
                "input_tokens": input_tokens,
                "output_tokens": output_tokens,
                "estimated_cost_usd": round(estimated_cost, 6),
                "latency_seconds": round(selesai - mulai, 4),
                "error": error,
            })
    return wrapper


def fake_llm_call(messages: list[dict], model: str = "gpt-4o-mini") -> str:
    """Simulasi LLM call TANPA network -- buat membuktikan trace_llm_call
    beneran jalan, tanpa perlu API key sungguhan."""
    time.sleep(0.01)
    return f"Balasan simulasi untuk: {messages[-1]['content']}"


traced_fake_llm_call = trace_llm_call(fake_llm_call)

# functools.wraps membuat nama & docstring ASLI tetap kepakai, bukan "wrapper"
print(traced_fake_llm_call.__name__)
# -> "fake_llm_call"

reply = traced_fake_llm_call(
    messages=[{"role": "user", "content": "gimana status order O-456?"}],
    model="gpt-4o-mini",
)
print(reply)
# -> "Balasan simulasi untuk: gimana status order O-456?"

print(len(TRACE_STORE), TRACE_STORE[-1]["function"], TRACE_STORE[-1]["model"])
# -> 1 fake_llm_call gpt-4o-mini
```
Tiga print di atas nunjukin `trace_llm_call` genuinely bekerja, bukan cuma diklaim di prosa: nama fungsi asli tetap kepakai berkat `functools.wraps` (bukan `"wrapper"`), jawaban dari `fake_llm_call` tetap dibalikin TANPA berubah, dan satu entry baru beneran nambah ke `TRACE_STORE` berisi nama fungsi & model yang benar.

Wiring-nya ke `LLMGateway.generate` (Phase 5, topik 20) — cukup tambah satu baris `@trace_llm_call` di atas method-nya, isi method-nya SAMA SEKALI gak berubah:
```python
from openai import OpenAI
from anthropic import Anthropic


class LLMGateway:
    """Sama seperti Phase 5 topik 20, tapi method generate() sekarang
    dibungkus trace_llm_call supaya SETIAP panggilannya otomatis tercatat."""

    def __init__(self, provider: str = "openai"):
        self.provider = provider
        self.openai_client = OpenAI()
        self.anthropic_client = Anthropic()

    @trace_llm_call
    def generate(self, messages: list[dict], model: str | None = None) -> str:
        if self.provider == "openai":
            return self._generate_openai(messages, model or "gpt-4o-mini")
        elif self.provider == "anthropic":
            return self._generate_anthropic(messages, model or "claude-3-5-sonnet-20241022")
        else:
            raise ValueError(f"Provider tidak dikenal: {self.provider}")

    def _generate_openai(self, messages: list[dict], model: str) -> str:
        response = self.openai_client.chat.completions.create(model=model, messages=messages)
        return response.choices[0].message.content

    def _generate_anthropic(self, messages: list[dict], model: str) -> str:
        response = self.anthropic_client.messages.create(model=model, max_tokens=1024, messages=messages)
        return response.content[0].text
```

### Trade-off & Pitfall
- **Estimasi token di `_estimate_tokens` itu APPROXIMATION, bukan tokenizer asli** — `LLMGateway.generate` cuma balikin `str` (teks jawaban), bukan objek response mentah yang punya `.usage` (beda dengan `track_usage` di Phase 5 topik 22 yang langsung dikasih objek response asli) — production system sebaiknya ubah `generate()` supaya ikut mengembalikan/menyimpan usage asli dari provider, atau pakai tokenizer resmi (`tiktoken` buat OpenAI), bukan cuma hitung kata dibagi 0.75.
- **In-memory `TRACE_STORE` hilang begitu proses restart** — cukup buat belajar/demo, tapi production butuh trace store yang persisten (database, atau platform observability khusus kayak Langfuse/LangSmith) supaya trace lama tetap bisa diaudit.
- **Tracing nambah sedikit overhead ke tiap panggilan** — biasanya kecil (hitungan milidetik), tapi kalau logic di dalam `wrapper` makin berat (misal nulis ke database sinkron di tiap call), itu bisa nambah latency yang dirasakan customer; production biasanya nge-log secara async/batch, bukan blocking di jalur request utama.
- **Prompt & response mentah yang di-trace bisa mengandung data sensitif customer** — trace store itu sendiri jadi tempat baru yang perlu diamankan (access control) dan sebaiknya di-redact PII-nya (Phase 14 topik 58) SEBELUM disimpan permanen, bukan cuma dianggap "aman karena internal".

### Kapan Dipakai
- Pasang `trace_llm_call` (atau setara) di SEMUA fungsi yang benar-benar manggil LLM provider — `LLMGateway.generate` (Phase 5) adalah titik yang paling tepat karena SEMUA panggilan LLM SupportPilot lewat situ, jadi satu decorator otomatis mencakup semuanya.
- Wajib dipasang SEBELUM SupportPilot punya trafik production nyata — trace yang baru dipasang SETELAH ada insiden gak bisa "melihat ke belakang" ke kejadian yang sudah lewat.
- Kombinasikan dengan evaluation (topik 60-62) — trace yang terkumpul di sini adalah bahan mentah buat membangun eval dataset dan menganalisis kegagalan agent secara sistematis, bukan cuma buat debugging manual satu-satu.

### Sering Ditanya Saat Interview
- **Apa saja yang biasanya dicatat dalam LLM observability?** — prompt, response, tokens, latency, cost, model, tool calls, errors, dan retrieval results (kalau ada RAG yang terlibat).
- **Kenapa `trace_llm_call` ditulis sebagai decorator, bukan dipanggil manual di tiap tempat?** — supaya logging-nya terpasang SEKALI di titik pusat (`LLMGateway.generate`) dan otomatis berlaku buat SEMUA panggilan LLM, tanpa perlu duplikasi kode logging di tiap endpoint/fungsi yang manggil LLM — konsep yang sama dengan kenapa `@app.post("/chat")` dipakai di Phase 5 topik 19.
- **Kenapa `functools.wraps` penting dipakai di dalam decorator seperti `trace_llm_call`?** — tanpa itu, fungsi yang sudah dibungkus "kehilangan identitasnya" (nama & docstring-nya jadi `"wrapper"`/kosong), yang bikin debugging dan introspeksi jadi membingungkan; `functools.wraps` menyalin metadata fungsi asli ke wrapper-nya.
- **Kenapa estimasi token & cost di contoh ini disebut approximation, bukan angka pasti?** — karena `LLMGateway.generate` cuma mengembalikan teks jawaban (`str`), bukan objek response mentah yang punya info `.usage` asli dari provider — beda dengan `track_usage` (Phase 5 topik 22) yang langsung diberi objek response asli buat dihitung persis.

---

## 60. LLM Evaluation

### Apa itu?
LLM Evaluation adalah cara mengukur kualitas jawaban LLM/agent secara SISTEMATIS dan terukur — bukan cuma developer nge-tes beberapa pertanyaan manual lalu bilang "kelihatannya udah bagus". Fondasinya adalah **eval dataset**: kumpulan baris berisi empat kolom — **Input** (pertanyaan/pesan test), **Expected Output** (jawaban yang seharusnya, atau kriteria yang harus terpenuhi), **Actual Output** (jawaban asli yang dihasilkan sistem), dan **Score** (seberapa cocok Actual dengan Expected). Dimensi yang biasa dievaluasi: **accuracy**, **relevance**, **faithfulness**, **tool correctness**, **retrieval quality**, dan **safety**.

### Kenapa dibutuhkan?
"Kelihatannya bagus" itu subjektif, gak konsisten antar orang, dan gak bisa dijalankan otomatis tiap kali ada perubahan kode (ganti prompt, ganti model, nambah tool baru). Tanpa eval dataset yang jelas, SupportPilot gak punya cara buat menjawab pertanyaan sesederhana "apakah perubahan prompt kemarin bikin jawaban jadi lebih baik atau lebih buruk?" secara objektif — cuma bisa nebak dari beberapa contoh yang dicoba manual, yang gampang bias ke kasus yang "kebetulan kelihatan oke". Eval dataset yang konsisten menyelesaikan ini: sama seperti unit test buat kode biasa, tapi buat perilaku LLM/agent yang sifatnya non-deterministik.

### Cara Kerja
```
Siapkan test_cases: [{"input": ..., "expected_keywords": [...]}, ...]
    -> untuk tiap test case: jalankan run_agent_loop (Phase 6, topik 25)
         -> dapat actual_output (jawaban asli)
    -> score_response_keywords(actual_output, expected_keywords)
         -> proporsi expected_keywords yang MUNCUL di actual_output
    -> kumpulkan semua baris (Input, Expected, Actual, Score)
    -> agregat jadi avg_score buat SELURUH eval dataset
```
Versi di bawah pakai **keyword overlap** (mirip `evaluate_rag_pipeline` di Phase 4 topik 18) buat scoring — bukan embedding similarity atau LLM-judge yang lebih canggih, tapi cukup buat sinyal kasar dan gampang dipahami tanpa dependency tambahan.

### Contoh Kode — Python
`score_response_keywords` adalah fungsi murni (gak butuh network), jadi bisa dibuktikan langsung jalan dengan benar:
```python
def score_response_keywords(actual_response: str, expected_keywords: list[str]) -> float:
    """
    Skor sederhana: proporsi expected_keywords yang MUNCUL (case-insensitive)
    di actual_response. Filosofinya sama dengan skor correctness di
    evaluate_rag_pipeline (Phase 4 topik 18), tapi dipakai di sini buat
    menilai jawaban agent secara umum, gak cuma jawaban RAG.
    """
    if not expected_keywords:
        return 0.0
    teks = actual_response.lower()
    ditemukan = sum(1 for kw in expected_keywords if kw.lower() in teks)
    return ditemukan / len(expected_keywords)


# Jawaban yang mengandung SEMUA keyword yang diharapkan -> skor penuh
print(score_response_keywords(
    "Tiket T-555 kamu sudah kami eskalasikan ke tim finance untuk refund.",
    ["eskalasi", "refund"],
))
# -> 1.0

# Jawaban yang gak mengandung SATU PUN keyword yang diharapkan -> skor 0
print(score_response_keywords(
    "Tiket T-555 statusnya masih menunggu review.",
    ["eskalasi", "refund"],
))
# -> 0.0
```
Dua hasil di atas nunjukin `score_response_keywords` genuinely membedakan jawaban yang relevan dari yang enggak: jawaban yang menyebut "eskalasi" dan "refund" (dua kata yang diharapkan) dapat skor `1.0`, sementara jawaban yang gak menyebut sama sekali dapat `0.0` — bukan angka statis yang diklaim di prosa.

Eval harness lengkapnya menjalankan `score_response_keywords` di atas SETIAP test case lewat `run_agent_loop` (Phase 6, topik 25) yang beneran manggil LLM/agent:
```python
def run_llm_eval_harness(client, test_cases: list[dict], tools: list[dict]) -> dict:
    """
    Jalankan tiap test case lewat run_agent_loop (Phase 6, topik 25),
    bandingkan jawabannya terhadap expected_keywords, dan bangun "eval
    dataset" -- Input, Expected Output, Actual Output, Score -- buat SEMUA
    kasus sekaligus, bukan cuma nge-eyeball satu-dua contoh manual.

    test_cases: list of {"input": str, "expected_keywords": list[str]}
    """
    results = []
    for kasus in test_cases:
        actual_output = run_agent_loop(client, kasus["input"], tools, max_steps=5)
        score = score_response_keywords(actual_output, kasus["expected_keywords"])
        results.append({
            "input": kasus["input"],
            "expected_keywords": kasus["expected_keywords"],
            "actual_output": actual_output,
            "score": score,
        })

    avg_score = sum(r["score"] for r in results) / len(results) if results else 0.0
    return {"results": results, "avg_score": round(avg_score, 4)}


# Contoh pemakaian (butuh client & tools sungguhan dari Phase 6):
supportpilot_test_cases = [
    {
        "input": "Tiket T-555 saya kok belum ada respon dari kemarin, gimana ya?",
        "expected_keywords": ["eskalasi", "tiket"],
    },
    {
        "input": "Order O-456 saya kapan sampai?",
        "expected_keywords": ["order", "status"],
    },
]
# eval_result = run_llm_eval_harness(client, supportpilot_test_cases, tools)
# print(eval_result["avg_score"])
# diharapkan: skor mendekati 1.0 kalau agent konsisten menyebut kata kunci
# yang relevan di jawabannya
```

### Trade-off & Pitfall
- **Keyword overlap gampang salah nilai jawaban yang benar tapi pakai kata berbeda** — jawaban yang secara makna SAMA PERSIS tapi gak menyebut kata kunci literal yang diharapkan (misal jawab "sudah kami naikkan ke tim terkait" tanpa nyebut kata "eskalasi") bakal dapat skor rendah walau sebenarnya benar; embedding similarity atau LLM-judge lebih toleran terhadap variasi kata seperti ini, tapi butuh dependency tambahan (model embedding/LLM judge terpisah).
- **`run_llm_eval_harness` manggil LLM sungguhan buat tiap test case** — ini artinya eval punya cost & latency-nya sendiri (bukan gratis), dan hasilnya bisa sedikit berbeda antar run kalau `temperature` model gak diset ke 0 — pertimbangkan set `temperature=0` di panggilan yang dievaluasi supaya hasilnya lebih konsisten dibandingkan antar percobaan.
- **`accuracy`, `relevance`, `faithfulness`, `tool correctness`, `retrieval quality`, dan `safety` gak semuanya bisa diukur dengan satu fungsi keyword overlap sederhana** — `score_response_keywords` di sini cuma menyentuh sebagian kecil dimensi itu (kira-kira relevance/accuracy dasar); retrieval quality dibahas lebih dalam di topik 61, tool correctness di topik 62, dan faithfulness/safety butuh pendekatan lain (LLM-judge, guardrails Phase 14 topik 58) yang gak dibahas detail di sini.
- **Eval dataset yang statis bisa jadi basi** — begitu SupportPilot nambah fitur/tool baru, test case lama mungkin gak lagi merepresentasikan pertanyaan customer yang paling relevan; dataset butuh direview & diperbarui berkala, bukan ditulis sekali lalu dipakai selamanya.

### Kapan Dipakai
- Bangun eval dataset SEJAK AWAL, begitu SupportPilot mulai punya fitur agent/LLM yang nontrivial — jangan tunggu sampai ada insiden customer dulu buat mulai menulis test case.
- Jalankan `run_llm_eval_harness` (atau setara) SETIAP kali ada perubahan yang berpotensi mengubah perilaku jawaban — ganti prompt, ganti model, ganti versi tool — supaya perubahan itu bisa dibandingkan secara objektif terhadap baseline sebelumnya.
- Kalau keyword overlap mulai kelihatan sering salah nilai jawaban yang secara makna benar, itu sinyal buat upgrade ke embedding similarity atau LLM-judge, bukan terus menambah keyword secara manual tanpa henti.

### Sering Ditanya Saat Interview
- **Apa empat kolom utama dalam sebuah eval dataset?** — Input, Expected Output, Actual Output, dan Score.
- **Kenapa "kelihatannya bagus" gak cukup buat mengevaluasi LLM/agent?** — subjektif, gak konsisten antar orang, gak bisa dijalankan otomatis, dan gampang bias ke kasus yang kebetulan kelihatan oke — sama seperti kenapa kode butuh unit test daripada cuma "dicoba-coba manual terus keliatannya jalan".
- **Sebutkan enam dimensi umum yang dievaluasi dalam LLM evaluation.** — accuracy, relevance, faithfulness, tool correctness, retrieval quality, dan safety.
- **Apa batasan utama scoring berbasis keyword overlap dibanding embedding similarity/LLM-judge?** — keyword overlap gagal menilai jawaban yang secara makna benar tapi memakai kata-kata berbeda dari yang diharapkan; embedding similarity/LLM-judge lebih toleran terhadap variasi kata, tapi butuh dependency tambahan (model embedding atau LLM judge terpisah).

---

## 61. RAG Evaluation

### Apa itu?
RAG Evaluation di topik ini adalah PERLUASAN dari `evaluate_rag_pipeline` (Phase 4, topik 18): daripada cuma melaporkan `retrieval_hit_rate` (binary: ada/enggak chunk relevan ke-retrieve) dan `avg_answer_correctness` secara terpisah seperti sebelumnya, kita bikin eksplisit DUA skor yang masing-masing lebih tajam — **retrieval_score** (precision: dari chunk yang benar-benar TERPILIH, berapa PERSEN yang relevan — bukan cuma binary ada/enggak) dan **generation_score** (gabungan **answer correctness** DAN **faithfulness** — apakah jawaban model setia ke konteks yang diberikan, bukan ngarang dari luar konteks). Dua skor ini SENGAJA dilaporkan terpisah, gak dirata-ratakan jadi satu angka gabungan.

### Kenapa dibutuhkan?
Kalau retrieval dan generation dicampur jadi satu angka "overall score", satu skor jelek gak ngasih tau APA yang perlu diperbaiki — apakah `retrieve_relevant_chunks`/`rerank_chunks` (Phase 4) yang gagal narik chunk relevan, atau LLM-nya yang gagal menyusun jawaban yang benar DARI chunk yang sebenarnya sudah relevan? Dua penyebab itu butuh perbaikan yang SANGAT berbeda (tuning retrieval vs tuning prompt/model generasi). Dengan melaporkan `retrieval_score` dan `generation_score` sebagai dua angka terpisah — dan masing-masing dibuat lebih tajam dari versi Phase 4 (precision proper buat retrieval, faithfulness ditambahkan buat generation) — tim bisa langsung tau lapisan mana yang perlu diprioritaskan buat diperbaiki duluan.

### Cara Kerja
```
Untuk tiap test case (query + expected_answer):
    retrieve_relevant_chunks -> rerank_chunks -> chunk_terpilih (top_k)

    retrieval_score (precision) per test case:
        proporsi chunk_terpilih yang mengandung kata kunci dari expected_answer
        (bukan cuma "ada satu chunk relevan", tapi BERAPA PERSEN dari yang
        di-retrieve itu relevan)

    generation_score per test case:
        LLM generate jawaban dari konteks chunk_terpilih
        correctness  = proporsi kata kunci expected_answer yang muncul di jawaban
        faithfulness = proporsi kata DI JAWABAN yang juga muncul di konteks
                        (jawaban yang banyak "ngarang" di luar konteks -> rendah)
        generation_score = rata-rata (correctness, faithfulness)

    -> rata-ratakan retrieval_score & generation_score SECARA TERPISAH
       ke seluruh test_cases, JANGAN digabung jadi satu angka
```

### Contoh Kode — Python
```python
def evaluate_rag_pipeline_v2(test_cases: list[dict]) -> dict:
    """
    Versi lanjutan evaluate_rag_pipeline (Phase 4, topik 18): melaporkan
    retrieval_score (precision) dan generation_score (correctness +
    faithfulness) sebagai DUA angka terpisah -- bukan digabung jadi satu
    overall score -- supaya jelas lapisan mana (retrieval atau generation)
    yang perlu diperbaiki kalau skornya jelek.

    test_cases: list of {"query": str, "expected_answer": str}

    Catatan: `conn` dan `client` di sini memakai koneksi/instance yang sama
    yang sudah dibuka di awal modul, sama seperti evaluate_rag_pipeline di
    Phase 4 topik 18.
    """
    retrieval_precisions = []
    generation_scores = []

    for kasus in test_cases:
        query = kasus["query"]
        expected_answer = kasus["expected_answer"]
        kata_kunci = [kata for kata in expected_answer.lower().split() if kata]

        kandidat = retrieve_relevant_chunks(conn, query, top_k=10)
        chunk_terpilih = rerank_chunks(query, kandidat, top_k=3)

        # retrieval_score: PRECISION -- proporsi chunk TERPILIH yang mengandung
        # minimal satu kata kunci dari expected_answer, bukan cuma binary
        # "ada satu chunk relevan atau enggak" seperti retrieval_hit_rate
        # di evaluate_rag_pipeline (Phase 4).
        chunk_relevan = sum(
            1 for chunk in chunk_terpilih
            if any(kata in chunk["content"].lower() for kata in kata_kunci)
        )
        precision = chunk_relevan / len(chunk_terpilih) if chunk_terpilih else 0.0
        retrieval_precisions.append(precision)

        konteks = "\n\n".join(chunk["content"] for chunk in chunk_terpilih)
        prompt = (
            "Jawab pertanyaan berikut HANYA berdasarkan konteks di bawah ini.\n\n"
            f"Konteks:\n{konteks}\n\nPertanyaan: {query}"
        )
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt}],
            temperature=0,
        )
        jawaban_model = response.choices[0].message.content

        # correctness: proporsi kata kunci expected_answer yang muncul di jawaban
        if kata_kunci:
            kata_muncul = sum(1 for kata in kata_kunci if kata in jawaban_model.lower())
            correctness = kata_muncul / len(kata_kunci)
        else:
            correctness = 0.0

        # faithfulness (proxy sederhana): proporsi kata DI JAWABAN yang juga
        # muncul di konteks yang ditempel -- jawaban yang banyak "ngarang" kata
        # yang gak ada di konteks bakal dapat skor faithfulness rendah.
        kata_jawaban = [kata for kata in jawaban_model.lower().split() if kata]
        if kata_jawaban:
            kata_grounded = sum(1 for kata in kata_jawaban if kata in konteks.lower())
            faithfulness = kata_grounded / len(kata_jawaban)
        else:
            faithfulness = 0.0

        generation_scores.append((correctness + faithfulness) / 2)

    return {
        "jumlah_test_case": len(test_cases),
        "retrieval_score": round(sum(retrieval_precisions) / len(test_cases), 4),
        "generation_score": round(sum(generation_scores) / len(test_cases), 4),
    }


# Contoh pemakaian (butuh conn & client sungguhan dari Phase 4):
test_cases = [
    {
        "query": "berapa lama proses refund SupportPilot",
        "expected_answer": "refund diproses dalam 3-5 hari kerja setelah barang diterima gudang",
    },
    {
        "query": "gimana caranya reset password ya",
        "expected_answer": "klik lupa password di halaman login lalu ikuti instruksi di email",
    },
]
# hasil = evaluate_rag_pipeline_v2(test_cases)
# print(hasil)
# diharapkan: {"jumlah_test_case": 2, "retrieval_score": <0.0-1.0>, "generation_score": <0.0-1.0>}
# -- dua angka TERPISAH, bukan satu overall_score gabungan
```

### Trade-off & Pitfall
- **`retrieval_score` dan `generation_score` di sini masih heuristik keyword overlap**, sama seperti `evaluate_rag_pipeline` Phase 4 — precision "beneran" secara IR (Information Retrieval) biasanya butuh label relevansi manual per chunk (bukan cuma tebak dari kata kunci `expected_answer`); heuristik ini cukup buat sinyal kasar, bukan pengganti anotasi relevansi yang teliti.
- **Proxy faithfulness berbasis overlap kata itu KASAR** — jawaban yang menyusun ulang informasi dari konteks pakai SINONIM (bukan kata yang sama persis) bisa salah dinilai "gak faithful" walau sebenarnya tetap setia ke konteks; faithfulness yang lebih akurat biasanya butuh LLM-judge yang benar-benar membandingkan makna, bukan cuma kata literal.
- **Melaporkan dua skor terpisah butuh disiplin buat gak "menyembunyikan" masalah di salah satu sisi** — tim yang cuma lihat `generation_score` yang bagus bisa lupa cek `retrieval_score`-nya juga jelek atau enggak; dua angka ini WAJIB dibaca bersamaan, bukan pilih salah satu yang kelihatan lebih bagus.
- **Biaya evaluasi bertambah** dibanding versi Phase 4 — fungsi ini tetap manggil LLM sekali per test case (buat generation_score), jadi trade-off cost/latency yang sama dengan `evaluate_rag_pipeline` tetap berlaku, plus sedikit komputasi tambahan buat menghitung faithfulness.

### Kapan Dipakai
- Pakai `retrieval_score`/`generation_score` terpisah begitu tim mulai butuh MEMUTUSKAN prioritas perbaikan — kalau cuma butuh sinyal kasar "pipeline-nya jalan gak", `evaluate_rag_pipeline` (Phase 4) yang lebih sederhana sudah cukup.
- Jalankan evaluasi ini setiap kali ada perubahan ke komponen retrieval (`chunk_size`, `top_k`, model reranking, Phase 4 topik 15-16) ATAU komponen generation (prompt, model LLM) — bandingkan skor sebelum/sesudah buat memastikan perubahan itu benar-benar membantu SISI yang dituju, bukan cuma "kelihatan lebih baik" secara keseluruhan.
- Kombinasikan dengan RAG Failure Modes (Phase 4, topik 17) — skor `retrieval_score` yang jelek adalah sinyal buat langsung menelusuri failure mode di sisi retrieval, sementara `generation_score` yang jelek mengarahkan ke sisi prompt/model generasi.

### Sering Ditanya Saat Interview
- **Kenapa retrieval quality dan generation quality sebaiknya dilaporkan sebagai dua angka terpisah, bukan satu overall score?** — supaya jelas lapisan mana (retrieval atau generation) yang bermasalah kalau skornya jelek; kalau digabung jadi satu angka, satu skor rendah gak ngasih tau apakah retrieval-nya yang gagal narik chunk relevan atau LLM-nya yang gagal menyusun jawaban benar dari chunk yang sudah relevan.
- **Apa beda `retrieval_score` (topik ini) dengan `retrieval_hit_rate` di `evaluate_rag_pipeline` (Phase 4)?** — `retrieval_hit_rate` itu binary per test case (ada/enggak MINIMAL satu chunk relevan), sedangkan `retrieval_score` di sini adalah precision (berapa PERSEN dari chunk yang di-retrieve itu benar-benar relevan) — lebih tajam buat melihat seberapa "bersih" hasil retrieval-nya.
- **Apa itu faithfulness, dan gimana proxy sederhananya diukur di contoh ini?** — seberapa "setia" jawaban model terhadap konteks yang diberikan (bukan ngarang dari luar konteks); proxy sederhananya di sini adalah proporsi kata DI JAWABAN yang juga muncul di konteks yang ditempel.
- **Apa batasan utama proxy faithfulness berbasis keyword overlap?** — jawaban yang menyusun ulang informasi pakai SINONIM (bukan kata yang sama persis dengan konteks) bisa salah dinilai "gak faithful" walau sebenarnya tetap setia secara makna — faithfulness yang lebih akurat biasanya butuh LLM-judge, bukan cuma pencocokan kata literal.

---

## 62. Agent Evaluation

### Apa itu?
Agent Evaluation adalah evaluasi yang fokus ke KEPUTUSAN dan LANGKAH yang diambil agent sepanjang satu `run_agent_loop` (Phase 6, topik 25) — bukan cuma jawaban akhirnya. Dimensi yang dievaluasi: apakah agent **memilih tool yang benar**, apakah **argumen tool-nya benar** (kelihatan valid), apakah **task-nya selesai**, **berapa banyak step** yang dipakai, **berapa cost-nya**, dan apakah ada **pelanggaran permission** di sepanjang jalan. Metrik headline yang paling sering dipakai buat merangkum semua ini di level agregat: **Task Success Rate** — persentase run yang dianggap "berhasil" dari total run yang dievaluasi.

### Kenapa dibutuhkan?
`run_agent_loop` bisa saja menghasilkan JAWABAN yang kelihatan masuk akal secara teks, tapi sebenarnya agent-nya "mengambil jalan yang salah" buat sampai ke situ — misal manggil tool yang salah, mengirim argumen yang gak lengkap (yang seharusnya ditolak `validate_tool_call`, Phase 14 topik 54), butuh step yang jauh lebih banyak dari seharusnya, atau (paling parah) mencoba manggil tool yang sebenarnya gak diizinkan buat role itu (`check_permission`, Phase 11 topik 45). Evaluasi yang cuma lihat teks jawaban akhir gak akan menangkap masalah semacam ini. `evaluate_agent_run` di bawah menganalisis SELURUH transcript langkah demi langkah — bukan cuma baris terakhirnya — supaya masalah di tengah proses ikut kedeteksi.

### Cara Kerja
```
transcript (list of dict), step pertama berisi EKSPEKTASI test case:
    [0] {"type": "expected", "expected_tool": ..., "expected_argument_keys": [...], "success_keywords": [...]}
    [1..] hasil rekaman ASLI dari run_agent_loop:
          {"type": "tool_call", "tool_name": ..., "tool_arguments": {...}}
          {"type": "tool_result", "tool_name": ..., "result": {...}}
          {"type": "message", "content": "..."}   <- jawaban final

evaluate_agent_run(transcript):
    1. tool_called_correctly -> expected_tool ADA di antara tool_call yang terjadi?
    2. arguments_correct     -> panggilan ke expected_tool itu mengandung SEMUA
                                expected_argument_keys?
    3. task_complete         -> jawaban final (message terakhir) mengandung
                                SEMUA success_keywords?
    4. permission_violations -> ada tool_result yang error-nya nyebut "izin"
                                (pola pesan error check_permission, Phase 11)?
    5. step_count, total_cost_usd -> dihitung/dijumlah dari transcript
    -> success = SEMUA (1)-(4) lolos DAN gak ada permission_violations
```

### Contoh Kode — Python
```python
def evaluate_agent_run(transcript: list[dict]) -> dict:
    """
    Evaluasi SATU rekaman run_agent_loop (Phase 6, topik 25) buat sebuah eval
    test case. `transcript` diawali step ber-type "expected" (ekspektasi test
    case), diikuti step-step ASLI dari agent loop ("tool_call", "tool_result",
    diakhiri "message" berisi jawaban final).

    Fungsi ini BENAR-BENAR menganalisis isi transcript -- tool yang dipanggil,
    argumen yang dikirim, kata kunci di jawaban final, jumlah step, dan tanda
    pelanggaran permission -- bukan cuma mengembalikan dict statis.
    """
    if not transcript or transcript[0].get("type") != "expected":
        raise ValueError("transcript harus diawali step ber-type 'expected'")

    expected = transcript[0]
    steps = transcript[1:]

    tool_calls = [step for step in steps if step.get("type") == "tool_call"]
    tool_results = [step for step in steps if step.get("type") == "tool_result"]
    final_messages = [step for step in steps if step.get("type") == "message"]

    expected_tool = expected.get("expected_tool")
    expected_argument_keys = set(expected.get("expected_argument_keys", []))
    success_keywords = expected.get("success_keywords", [])

    # 1. Tool correctness: apakah tool yang DIHARAPKAN benar-benar dipanggil
    matching_calls = [tc for tc in tool_calls if tc.get("tool_name") == expected_tool]
    tool_called_correctly = len(matching_calls) > 0

    # 2. Argument correctness: dari panggilan ke tool yang benar, apakah SEMUA
    # field yang diharapkan ADA di argumen yang dikirim model (cek "kelihatan
    # benar" secara struktur -- bukan validasi tipe/schema penuh seperti
    # validate_tool_call, Phase 14 topik 54, yang jauh lebih ketat).
    arguments_correct = False
    if matching_calls:
        arguments_correct = any(
            expected_argument_keys.issubset(set(tc.get("tool_arguments", {}).keys()))
            for tc in matching_calls
        )

    # 3. Task completion: jawaban final mengandung SEMUA success_keywords
    final_answer = final_messages[-1]["content"] if final_messages else ""
    task_complete = bool(success_keywords) and all(
        kw.lower() in final_answer.lower() for kw in success_keywords
    )

    # 4. Permission violation: tool_result mana pun yang error-nya nyebut "izin"
    # (pola pesan error dari check_permission, Phase 11 topik 45)
    permission_violations = [
        tr for tr in tool_results
        if isinstance(tr.get("result"), dict)
        and "izin" in str(tr["result"].get("error", "")).lower()
    ]

    # 5. Cost & step count: total_cost_usd dijumlah dari field opsional yang
    # mungkin sudah dicatat trace_llm_call (topik 59) di tiap step, kalau ada
    total_cost_usd = sum(step.get("cost_usd", 0.0) for step in steps)
    step_count = len(tool_calls)

    success = (
        tool_called_correctly and arguments_correct and task_complete
        and not permission_violations
    )

    return {
        "tool_called_correctly": tool_called_correctly,
        "arguments_correct": arguments_correct,
        "task_complete": task_complete,
        "step_count": step_count,
        "permission_violations": len(permission_violations),
        "total_cost_usd": round(total_cost_usd, 6),
        "success": success,
    }


def compute_task_success_rate(runs: list[dict]) -> float:
    """Metrik headline Agent Evaluation: proporsi run yang 'success' dari
    total run yang dievaluasi (masing-masing hasil evaluate_agent_run)."""
    if not runs:
        return 0.0
    return sum(1 for r in runs if r["success"]) / len(runs)


# Run yang BENAR: tool tepat, argumen lengkap, jawaban mengandung keyword sukses
good_transcript = [
    {
        "type": "expected",
        "expected_tool": "get_order_status",
        "expected_argument_keys": ["order_id"],
        "success_keywords": ["diproses"],
    },
    {"type": "tool_call", "tool_name": "get_order_status", "tool_arguments": {"order_id": "O-456"}, "cost_usd": 0.0002},
    {"type": "tool_result", "tool_name": "get_order_status", "result": {"status": "diproses"}},
    {"type": "message", "content": "Order O-456 kamu sedang diproses.", "cost_usd": 0.0005},
]
print(evaluate_agent_run(good_transcript))
# -> {"tool_called_correctly": True, "arguments_correct": True, "task_complete": True,
#     "step_count": 1, "permission_violations": 0, "total_cost_usd": 0.0007, "success": True}

# Run yang SALAH TOOL: agent manggil search_knowledge_base padahal
# seharusnya get_order_status
wrong_tool_transcript = [
    {
        "type": "expected",
        "expected_tool": "get_order_status",
        "expected_argument_keys": ["order_id"],
        "success_keywords": ["diproses"],
    },
    {"type": "tool_call", "tool_name": "search_knowledge_base", "tool_arguments": {"query": "kebijakan refund"}},
    {"type": "tool_result", "tool_name": "search_knowledge_base", "result": {"content": "..."}},
    {"type": "message", "content": "Order kamu sedang diproses."},
]
print(evaluate_agent_run(wrong_tool_transcript))
# -> {"tool_called_correctly": False, ..., "success": False}

# Run dengan PELANGGARAN PERMISSION: tool yang benar dipanggil dengan
# argumen benar, tapi ditolak check_permission (Phase 11 topik 45)
permission_violation_transcript = [
    {
        "type": "expected",
        "expected_tool": "issue_refund",
        "expected_argument_keys": ["order_id"],
        "success_keywords": ["refund"],
    },
    {"type": "tool_call", "tool_name": "issue_refund", "tool_arguments": {"order_id": "O-456"}},
    {"type": "tool_result", "tool_name": "issue_refund", "result": {
        "error": "Role 'support_agent' tidak punya izin memanggil 'issue_refund'"
    }},
    {"type": "message", "content": "Maaf, refund kamu gagal diproses."},
]
print(evaluate_agent_run(permission_violation_transcript))
# -> {"tool_called_correctly": True, "arguments_correct": True, "task_complete": True,
#     "step_count": 1, "permission_violations": 1, "total_cost_usd": 0.0, "success": False}

print(compute_task_success_rate([
    evaluate_agent_run(good_transcript),
    evaluate_agent_run(wrong_tool_transcript),
    evaluate_agent_run(permission_violation_transcript),
]))
# -> 0.3333333333333333
```
Tiga hasil `evaluate_agent_run` di atas nunjukin fungsi ini genuinely membedakan run yang benar-benar berhasil dari yang gagal karena alasan BERBEDA-BEDA — bukan cuma `return {"success": True}` yang diklaim di prosa: run yang benar semua lolos dan `success=True`, run yang salah tool langsung ke-flag `tool_called_correctly=False`, dan run yang kena pelanggaran permission tetap ke-flag `success=False` lewat `permission_violations=1` walau tool & argumennya sendiri sudah benar — tiga jalur kegagalan yang berbeda, dan `evaluate_agent_run` menangkap masing-masingnya secara spesifik, bukan cuma bilang "gagal" tanpa alasan.

### Trade-off & Pitfall
- **`arguments_correct` di sini cuma cek field yang WAJIB ada, gak validasi tipe/format** — ini SENGAJA lebih ringan dari `validate_tool_call` (Phase 14 topik 54); kalau butuh pengecekan argumen yang benar-benar ketat (tipe data, format value), gunakan `validate_tool_call` di titik eksekusi, dan `evaluate_agent_run` cukup fokus ke "kelihatan lengkap secara struktur" buat kebutuhan evaluasi agregat.
- **Deteksi permission violation di sini cocok ke pola pesan error SPESIFIK** (kata "izin") — kalau format pesan error `check_permission` berubah di masa depan, deteksi ini ikut gagal mendeteksi; production sebaiknya pakai kode error terstruktur (misal `{"error_code": "PERMISSION_DENIED"}`) daripada mencocokkan substring pesan teks bebas.
- **`success_keywords` yang terlalu ketat bisa salah nge-flag run yang sebenarnya berhasil** — kalau agent menjawab dengan kata yang berbeda tapi maknanya tetap benar (mirip masalah keyword overlap di topik 60), `task_complete` bisa salah jadi `False`; sama seperti topik 60, LLM-judge lebih toleran tapi butuh dependency tambahan.
- **`total_cost_usd` cuma akurat kalau tiap step transcript beneran diisi `cost_usd`** dari `trace_llm_call` (topik 59) — kalau step-nya gak membawa info itu (misal transcript direkam manual tanpa tracing), field ini diam-diam jadi `0.0` alih-alih error, yang bisa menyesatkan kalau dibaca tanpa sadar sumber datanya gak lengkap.

### Kapan Dipakai
- Jalankan `evaluate_agent_run` buat SETIAP test case di eval dataset (topik 60) yang melibatkan tool call — bukan cuma test case yang jawabannya teks biasa tanpa tool.
- Pantau `Task Success Rate` (lewat `compute_task_success_rate`) sebagai metrik headline tiap kali ada perubahan ke prompt sistem agent, daftar tool, atau model — turunnya Task Success Rate adalah sinyal regresi yang harus segera diselidiki lewat field detail (`tool_called_correctly`, `arguments_correct`, dst) buat tau AKAR masalahnya.
- Prioritaskan investigasi manual buat run dengan `permission_violations > 0` — ini bukan sekadar "agent kurang pintar", tapi potensi tanda agent mencoba melakukan aksi yang seharusnya di luar kewenangannya (terkait langsung ke Phase 11 topik 45 dan Phase 14 topik 54).

### Sering Ditanya Saat Interview
- **Sebutkan dimensi-dimensi utama yang dievaluasi dalam Agent Evaluation.** — tool yang dipilih benar atau enggak, argumen yang benar, task completion, jumlah step, cost, dan pelanggaran permission — dengan Task Success Rate sebagai metrik headline yang merangkum semuanya di level agregat.
- **Kenapa evaluasi agent gak bisa cuma melihat jawaban teks akhirnya saja?** — agent bisa menghasilkan jawaban yang kelihatan masuk akal secara teks walau sebenarnya mengambil jalan yang salah (tool yang salah, argumen gak lengkap, atau mencoba aksi yang gak diizinkan) — evaluasi yang cuma lihat teks akhir gak akan menangkap masalah macam ini, makanya `evaluate_agent_run` menganalisis SELURUH transcript.
- **Apa itu Task Success Rate, dan gimana cara menghitungnya?** — proporsi run yang dianggap "berhasil" (`success=True`) dari total run yang dievaluasi; dihitung dengan menjalankan `evaluate_agent_run` ke setiap transcript lalu mengagregat hasilnya, seperti `compute_task_success_rate` di atas.
- **Kenapa run yang tool & argumennya sudah benar tetap bisa dianggap gagal (`success=False`)?** — kalau run itu kena permission violation (tool yang dipanggil ditolak `check_permission`, Phase 11 topik 45) — `evaluate_agent_run` menganggap pelanggaran permission sebagai kegagalan TERLEPAS dari seberapa benar tool/argumennya, karena itu adalah masalah keamanan yang lebih serius daripada sekadar salah pilih tool.

---

**Selanjutnya:** [Phase 16 — Model Fine-Tuning](./phase-16-model-fine-tuning.md)
