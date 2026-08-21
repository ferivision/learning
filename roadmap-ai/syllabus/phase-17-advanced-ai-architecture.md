# Phase 17 — Advanced AI Architecture

> Bagian dari [AI Engineering / Agent Roadmap](../README.md)

---

## 65. Agentic RAG

### Apa itu?
Agentic RAG adalah RAG (Phase 4) yang "dibungkus" jadi agent loop (Phase 6): bukan cuma satu kali retrieve-lalu-generate, tapi ada langkah tambahan di tengah di mana sistem sendiri **menilai** apakah hasil retrieval udah cukup buat menjawab, dan kalau belum, dia **mencoba lagi** dengan query pencarian yang dirumuskan ulang — sebelum akhirnya menyusun jawaban final.

Bedanya kelihatan jelas kalau dikontraskan alurnya:
```
Plain RAG (topik 12, Phase 4) — satu jalur, gak ada evaluasi:
  Query customer → Retrieve → LLM jawab langsung

Agentic RAG — ada evaluasi & kemungkinan ulang:
  Goal (pertanyaan customer)
      → Agent decide: query apa yang mau dicari?
      → Retrieve (retrieve_relevant_chunks + rerank_chunks, Phase 4)
      → Evaluate: chunk ini udah cukup buat jawab, belum?
      → kalau belum DAN masih ada kesempatan: reformulasi query → search again
      → kalau udah cukup (atau kesempatan habis): Answer
```

### Kenapa dibutuhkan?
Plain RAG (Phase 4 topik 12) itu bagus buat kasus di mana pertanyaan customer kebetulan "cocok" secara istilah dengan isi artikel help-center — tapi customer sering nulis pertanyaan pakai bahasa sehari-hari yang beda jauh dari istilah teknis di artikel (misal "duitku kapan balik" vs artikel yang nulisnya "kebijakan refund"). Kalau retrieval pertama gagal nemuin chunk yang relevan karena mismatch istilah kayak gini, plain RAG gak punya cara buat sadar dan coba lagi — dia cuma sekali retrieve, lalu LLM tetap dipaksa jawab berdasarkan chunk yang sebenarnya gak relevan (atau malah bilang gak tau, padahal jawabannya sebenarnya ADA di knowledge base, cuma gak ketemu di percobaan pertama).

Agentic RAG menyelesaikan ini dengan menambahkan langkah **evaluate**: sebelum jawab, cek dulu apakah chunk yang ada beneran menjawab pertanyaannya. Kalau enggak, LLM sendiri yang merumuskan ulang query pencarian (misal ganti "duitku kapan balik" jadi "berapa lama proses refund SupportPilot") dan coba retrieve sekali lagi — meniru cara manusia yang gak langsung nyerah kalau pencarian pertama gak nemuin apa-apa, tapi coba kata kunci lain.

### Cara Kerja
`agentic_rag_loop(client, user_query)` menjalankan siklus retrieve → evaluate → (reformulate → retrieve lagi) sebanyak `max_retries` kali sebelum menyerah dan menyusun jawaban dari apa pun konteks yang berhasil terkumpul:
```
agentic_rag_loop(user_query):
    query_saat_ini = user_query

    ulangi sampai (max_retries + 1) kali:
        chunks = retrieve_relevant_chunks(query_saat_ini) → rerank_chunks(...)

        kalau ini percobaan TERAKHIR yang diizinkan:
            berhenti evaluasi, langsung pakai chunks apa adanya

        judge = LLM menilai: chunks ini cukup gak buat jawab user_query?
        kalau CUKUP → berhenti, pakai chunks ini
        kalau BELUM cukup → judge juga kasih query baru (reformulasi) →
                             query_saat_ini = query baru itu, lanjut iterasi berikutnya

    → susun jawaban final dari chunks yang terkumpul (LLM generate, sama seperti topik 12)
```
Dua hal yang bikin loop ini genuinely bounded dan gak asal prosa:
- **Evaluate-nya eksplisit dan terpisah dari generate akhir** — `_judge_and_maybe_reformulate` cuma punya satu tugas: balikin `{"sufficient": bool, "reformulated_query": str}`. Keputusan "lanjut retry atau berhenti" murni berdasarkan field `sufficient` ini, bukan ditebak dari teks jawaban akhir.
- **`max_retries` adalah batas keras**, persis spirit-nya `max_steps` di `run_agent_loop` (Phase 6 topik 25) — loop punya jumlah percobaan retrieval maksimum yang pasti (`max_retries + 1` kali retrieve), walau LLM judge terus-terusan bilang "belum cukup".

### Contoh Kode — Python
`_judge_and_maybe_reformulate` — helper yang minta LLM menilai kecukupan chunk dan (kalau perlu) merumuskan ulang query pencariannya:
```python
import json
from openai import OpenAI

client = OpenAI()


def _judge_and_maybe_reformulate(client, user_query: str, chunks: list[dict]) -> dict:
    """
    Minta LLM menilai apakah `chunks` (hasil retrieve_relevant_chunks + rerank_chunks,
    Phase 4) cukup buat menjawab `user_query` secara lengkap. Kalau belum cukup,
    LLM juga diminta merumuskan ulang query pencarian yang lebih spesifik/pakai
    istilah berbeda, supaya percobaan retrieval berikutnya punya kemungkinan
    lebih besar nemuin chunk yang relevan.

    Balikan: {"sufficient": bool, "reformulated_query": str}
    """
    konteks = (
        "\n\n".join(f"- {c['content']}" for c in chunks)
        if chunks
        else "(tidak ada chunk yang ditemukan)"
    )

    prompt = (
        "Kamu adalah judge internal untuk sistem RAG SupportPilot. Nilai apakah "
        "konteks di bawah ini CUKUP untuk menjawab pertanyaan customer secara "
        "lengkap dan akurat.\n\n"
        f"Pertanyaan customer: {user_query}\n\n"
        f"Konteks yang ditemukan:\n{konteks}\n\n"
        "Balas HANYA dengan JSON persis format ini, tanpa teks lain di luar JSON:\n"
        '{"sufficient": true, "reformulated_query": ""}\n\n'
        "Isi reformulated_query dengan query pencarian BARU yang lebih spesifik "
        "atau memakai istilah/sudut pandang berbeda dari pertanyaan customer asli, "
        "supaya retrieval berikutnya lebih mungkin nemuin chunk yang relevan. "
        "Kalau sufficient bernilai true, reformulated_query boleh dikosongkan."
    )

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0,
        response_format={"type": "json_object"},
    )
    return json.loads(response.choices[0].message.content)
```

`agentic_rag_loop` — fungsi utamanya, menjahit retrieve, evaluate, reformulate, dan generate jadi satu loop yang dibatasi `max_retries`:
```python
def agentic_rag_loop(client, user_query: str, max_retries: int = 2) -> str:
    """
    Agent-controlled RAG: bukan sekadar satu kali retrieve-lalu-generate (topik 12),
    tapi loop yang bisa memutuskan sendiri buat mencari lagi kalau hasil retrieval
    pertama dinilai belum cukup buat menjawab pertanyaan customer.

    Goal -> Agent decide what to search -> Retrieve -> Evaluate
        -> search again if needed (dibatasi max_retries) -> Answer
    """
    query_saat_ini = user_query
    chunk_terbaik: list[dict] = []

    for percobaan in range(max_retries + 1):
        kandidat = retrieve_relevant_chunks(conn, query_saat_ini, top_k=10)
        chunk_terbaik = rerank_chunks(query_saat_ini, kandidat, top_k=3)

        # Percobaan terakhir yang diizinkan -> gak perlu evaluasi lagi,
        # apa pun hasilnya, ini yang bakal dipakai buat jawab
        if percobaan == max_retries:
            break

        judge = _judge_and_maybe_reformulate(client, user_query, chunk_terbaik)
        if judge["sufficient"]:
            break

        # Belum cukup DAN masih ada kesempatan retry -> pakai query hasil
        # reformulasi buat percobaan berikutnya
        query_baru = judge.get("reformulated_query", "").strip()
        if query_baru:
            query_saat_ini = query_baru

    konteks = "\n\n".join(c["content"] for c in chunk_terbaik) if chunk_terbaik else ""
    prompt = (
        "Jawab pertanyaan customer berikut HANYA berdasarkan konteks di bawah ini. "
        "Kalau konteksnya gak cukup buat jawab, bilang terus terang gak tau, "
        "jangan mengarang.\n\n"
        f"Konteks:\n{konteks}\n\nPertanyaan customer: {user_query}"
    )
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0,
    )
    return response.choices[0].message.content


# Contoh pemakaian
jawaban = agentic_rag_loop(client, "duitku kapan balik ya kalau barangnya aku retur")
print(jawaban)
```
Yang terjadi di dalam loop itu buat pertanyaan di atas, langkah demi langkah (`max_retries=2`, default):
1. **Percobaan 1** — `retrieve_relevant_chunks("duitku kapan balik ya kalau barangnya aku retur")` nemuin chunk yang kurang nyambung (embedding gak cukup kuat menangkap "duit balik" ~ "refund"). `_judge_and_maybe_reformulate` menilai `sufficient: false`, dan kasih `reformulated_query: "berapa lama proses refund setelah barang diretur"`.
2. **Percobaan 2** — `retrieve_relevant_chunks` dijalankan lagi pakai query hasil reformulasi itu, kali ini nemuin chunk kebijakan refund yang tepat. `_judge_and_maybe_reformulate` menilai `sufficient: true` — loop berhenti di sini, gak sampai ke percobaan ke-3.
3. **Generate jawaban akhir** dari chunk yang ditemukan di percobaan 2: *"Refund kamu bakal diproses dalam 3-5 hari kerja setelah barang diterima gudang."*

Kalau di percobaan 2 pun `judge` masih bilang belum cukup, loop bakal lanjut ke percobaan ke-3 (`percobaan == max_retries`) TANPA evaluasi lagi — langsung pakai chunk apa pun yang ketemu di percobaan itu buat menyusun jawaban, supaya loop pasti berhenti dan gak nge-retry tanpa batas.

### Trade-off & Pitfall
- **Tiap percobaan nambah minimal satu panggilan LLM tambahan** (buat `_judge_and_maybe_reformulate`), di atas panggilan generate akhir — dibanding plain RAG (topik 12) yang cuma satu panggilan LLM total, agentic RAG bisa 2-3x lebih mahal dan lebih lambat kalau sampai retry beberapa kali.
- **`max_retries` wajib ada** — tanpa batas ini, kalau LLM judge terus-menerus (secara keliru) bilang "belum cukup", loop bisa jalan berkali-kali tanpa henti, membengkakkan biaya dan latency tanpa jaminan hasilnya bakal jadi lebih baik.
- **Judge-nya sendiri bisa salah** — LLM yang menilai "cukup/belum cukup" bisa aja terlalu pede bilang cukup padahal chunk-nya sebenarnya kurang, atau kebalikannya, terlalu strict sampai selalu minta retry walau chunk yang ada sudah relevan. Kualitas evaluasi ini gak sempurna, sama seperti keterbatasan LLM-as-judge secara umum.
- **Reformulasi query gak menjamin hasil retrieval lebih baik** — LLM bisa aja merumuskan ulang query jadi versi yang justru sama-sama gak cocok dengan istilah di knowledge base; agentic RAG cuma nambah KESEMPATAN buat nemuin chunk yang lebih relevan, bukan garansi.

### Kapan Dipakai
- Pakai agentic RAG kalau knowledge base SupportPilot besar/heterogen dan pertanyaan customer sering ditulis pakai bahasa sehari-hari yang jauh beda dari istilah teknis di artikel — di sinilah kesempatan "coba lagi dengan query lain" paling bermanfaat.
- Kalau knowledge base-nya kecil dan pertanyaan customer biasanya udah cukup mirip dengan istilah di artikel, plain RAG (topik 12, Phase 4) sudah cukup dan jauh lebih murah — overhead evaluasi & kemungkinan retry gak sepadan manfaatnya.
- Cocok dipasang di jalur customer-facing yang toleran sedikit latency tambahan demi akurasi jawaban yang lebih tinggi; kalau butuh respons instan, pertimbangkan trade-off ini sebelum menambahkan langkah evaluate.

### Sering Ditanya Saat Interview
- **Apa beda mendasar plain RAG dan agentic RAG?** — plain RAG cuma satu kali retrieve-lalu-generate tanpa evaluasi; agentic RAG menambahkan langkah evaluate setelah retrieval, dan bisa mencoba retrieve lagi dengan query yang dirumuskan ulang kalau hasil pertama dinilai belum cukup, sebelum akhirnya menjawab.
- **Kenapa `agentic_rag_loop` butuh `max_retries`?** — sebagai batas keras supaya loop gak jalan tanpa henti kalau LLM judge terus-terusan menilai hasil retrieval belum cukup; tanpa batas ini biaya dan latency bisa membengkak gak terkendali.
- **Apa yang menentukan agentic RAG berhenti retry lebih awal (sebelum `max_retries` habis)?** — begitu `_judge_and_maybe_reformulate` mengembalikan `sufficient: true`, artinya LLM sudah menilai chunk yang ada cukup buat menjawab, loop langsung berhenti dan lanjut ke generate jawaban final.
- **Apa risiko utama dari langkah "evaluate" di agentic RAG?** — evaluasinya dilakukan oleh LLM juga, jadi bisa salah (terlalu longgar atau terlalu strict); dan reformulasi query yang dihasilkan gak selalu benar-benar memperbaiki hasil retrieval berikutnya.

---

## 66. Multi-Step Research Agent

### Apa itu?
Multi-Step Research Agent adalah pola agent yang tugasnya bukan menjawab SATU pertanyaan spesifik (seperti agentic RAG di topik 65), tapi melakukan **riset** lintas banyak sumber buat menyusun satu laporan sintesis. Siklusnya: **Search** (cari sumber-sumber yang mungkin relevan) → **Read sources** (baca isi tiap sumber yang ketemu) → **Extract data** (ambil fakta/pattern penting dari situ) → **Search for missing info** (kalau ternyata ada gap/pertanyaan susulan, cari lagi pakai keyword lain) → **Compare** (bandingkan temuan dari berbagai sumber, cari pattern-nya) → **Generate report** (susun jadi satu ringkasan akhir).

### Kenapa dibutuhkan?
Beberapa kebutuhan di SupportPilot itu bentuknya bukan "jawab pertanyaan customer A", tapi "cari tau ADA APA sebenarnya di balik banyak tiket/komplain" — misal tim ops nanya *"kenapa belakangan banyak komplain soal refund yang lambat? ada pattern-nya gak?"*. Ini beda karakter dari agentic RAG (topik 65): agentic RAG berhenti begitu satu pertanyaan customer terjawab dari 1-3 chunk; riset semacam ini butuh **menjelajahi banyak tiket**, ngambil fakta dari masing-masing, ngebandingin satu sama lain buat nemuin pattern (misal "ternyata korelasinya sama masalah gudang & kurir"), baru nyusun laporan. Satu kali retrieve gak akan cukup — riset ini butuh beberapa putaran search yang saling menyambung, di mana temuan dari putaran sebelumnya nentuin apa yang perlu dicari selanjutnya.

### Cara Kerja
Alur ini dibangun **di atas** `run_agent_loop` (Phase 6 topik 25) yang sudah ada — gak perlu bikin mesin loop baru dari nol, cukup kasih dia set tool yang cocok buat riset:
```
run_agent_loop(user_message="riset kenapa banyak komplain X", tools=[...riset tools...])
    → LLM: cari tiket pakai keyword awal (search_tickets_by_keyword)
    → LLM: baca hasilnya, nemuin keyword/pattern baru yang perlu ditelusuri lebih lanjut
    → LLM: search_tickets_by_keyword lagi pakai keyword baru itu (search for missing info)
    → ... berulang sampai LLM merasa cukup data buat dibandingkan ...
    → LLM: summarize_findings(catatan-catatan yang udah dikumpulkan)
    → LLM: susun jawaban final (laporan ringkasan) → loop selesai
```
Dua tool baru buat skenario ini: `search_tickets_by_keyword` (versi "Search" + "Read sources" sekaligus — mengembalikan cuplikan tiket yang mengandung suatu keyword) dan `summarize_findings` (versi "Generate report" dari catatan yang sudah dikumpulkan LLM sepanjang percakapan).

### Contoh Kode — Python
Dua tool baru, mock function dengan gaya yang sama seperti tool-tool di Phase 6:
```python
def search_tickets_by_keyword(keyword: str) -> list[dict]:
    """
    Mock function: pura-pura mencari tiket customer SupportPilot yang isinya
    mengandung `keyword` tertentu. Data di-hardcode di dictionary ini cuma buat
    ilustrasi; real case bakal query full-text search ke database tiket asli.
    """
    tiket_per_keyword = {
        "refund lambat": [
            {
                "ticket_id": "T-201",
                "excerpt": "refund saya udah 10 hari belum cair, katanya cuma 3-5 hari kerja",
            },
            {
                "ticket_id": "T-244",
                "excerpt": "kok refund lama banget ya, udah 2 minggu belum cair juga",
            },
        ],
        "gudang": [
            {
                "ticket_id": "T-201",
                "excerpt": "barang saya katanya belum diterima gudang, padahal kurir bilang udah delivered",
            },
        ],
        "kurir": [
            {
                "ticket_id": "T-201",
                "excerpt": "kurir bilang udah delivered tapi gudang bilang belum terima barangnya",
            },
            {
                "ticket_id": "T-267",
                "excerpt": "paket ilang di tengah jalan sama kurir, refund saya jadi ketunda gara-gara ini",
            },
        ],
    }
    return tiket_per_keyword.get(keyword, [])


def summarize_findings(notes: list[str]) -> str:
    """
    Mock function: pura-pura menyusun catatan-catatan riset yang sudah dikumpulkan
    LLM sepanjang percakapan jadi satu ringkasan terstruktur. Real case bakal
    manggil LLM lagi khusus buat merangkum (atau minimal deduplikasi & pengelompokan).
    """
    return "Ringkasan temuan riset:\n" + "\n".join(f"- {catatan}" for catatan in notes)
```

Tool schema dan `TOOL_REGISTRY` khusus buat skenario riset ini:
```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "search_tickets_by_keyword",
            "description": "Cari tiket customer SupportPilot yang mengandung keyword tertentu.",
            "parameters": {
                "type": "object",
                "properties": {
                    "keyword": {
                        "type": "string",
                        "description": "Kata kunci yang mau dicari di isi tiket, misal 'refund lambat'.",
                    }
                },
                "required": ["keyword"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "summarize_findings",
            "description": "Susun catatan-catatan temuan riset yang sudah dikumpulkan jadi satu ringkasan.",
            "parameters": {
                "type": "object",
                "properties": {
                    "notes": {
                        "type": "array",
                        "items": {"type": "string"},
                        "description": "Daftar catatan temuan riset yang sudah dikumpulkan sejauh ini.",
                    }
                },
                "required": ["notes"],
            },
        },
    },
]

TOOL_REGISTRY = {
    "search_tickets_by_keyword": search_tickets_by_keyword,
    "summarize_findings": summarize_findings,
}
```

Menjalankan riset lewat `run_agent_loop` yang sudah ada (Phase 6), `max_steps` dinaikkan dibanding contoh biasa karena riset butuh lebih banyak putaran search-then-refine:
```python
from openai import OpenAI

client = OpenAI()

laporan = run_agent_loop(
    client,
    user_message=(
        "Tolong investigasi kenapa belakangan banyak customer komplain soal "
        "refund yang lambat. Cari pattern-nya di beberapa tiket, lalu susun "
        "ringkasan laporan temuan kamu."
    ),
    tools=tools,
    max_steps=8,
)
print(laporan)
```
Kemungkinan yang terjadi di dalam loop, langkah demi langkah — bukan cuma satu putaran search, tapi berkali-kali search-then-refine sebelum menyusun laporan:
1. **Step 1** — model manggil `search_tickets_by_keyword("refund lambat")`, dapat dua tiket (T-201, T-244) yang komplain soal refund lama.
2. **Step 2** — dari isi tiket T-201, model notice ada penyebutan "gudang belum terima barang" — ini info baru yang belum ditelusuri, jadi model manggil `search_tickets_by_keyword("gudang")` buat cari tau lebih lanjut (**search for missing info**).
3. **Step 3** — hasil dari "gudang" masih menyebut soal kurir, model manggil `search_tickets_by_keyword("kurir")` lagi, dan nemuin tiket T-267 yang juga soal masalah kurir bikin refund ketunda — mulai kelihatan **pattern**: refund lambat sering berkorelasi sama masalah koordinasi gudang-kurir.
4. **Step 4** — model merasa udah cukup data buat dibandingkan (**compare**), manggil `summarize_findings` dengan catatan-catatan yang udah dikumpulkan dari 3 pencarian sebelumnya.
5. **Step 5** — model menyusun jawaban final (**generate report**) berdasarkan hasil `summarize_findings`, misalnya: *"Dari 3 tiket yang ditemukan, pattern utamanya adalah miskomunikasi status antara gudang dan kurir — kurir sudah bilang delivered tapi gudang belum konfirmasi terima, yang bikin proses refund ketahan. Rekomendasi: sinkronisasi status real-time antara sistem kurir dan gudang."* — loop berhenti di step ke-5, jauh sebelum `max_steps=8` habis.

Jumlah dan urutan keyword yang benar-benar dicari model gak fix di awal — itu keputusan model berdasarkan apa yang dia temukan di putaran sebelumnya, persis pola agent loop di Phase 6, cuma di sini tool-nya dikhususkan buat riset lintas sumber, bukan satu aksi customer service tunggal.

### Trade-off & Pitfall
- **Riset multi-step makan lebih banyak step/token dibanding agent customer-service biasa** — `max_steps` yang dipakai buat riset (di atas biasanya 6-10) perlu lebih besar dibanding agent tool-calling sederhana (Phase 6 topik 25 pakai 5), karena eksplorasi "search for missing info" berulang butuh beberapa putaran sebelum cukup data buat dibandingkan.
- **Model bisa berhenti riset lebih awal dari yang seharusnya** kalau prompt-nya gak eksplisit minta ketelitian ("cari pattern di BEBERAPA tiket", bukan cuma satu) — atau sebaliknya, bisa terus mengeksplorasi keyword yang gak produktif kalau gak dibatasi `max_steps`; batas ini tetap krusial persis seperti di `run_agent_loop` biasa.
- **`summarize_findings` cuma sebagus catatan yang dikumpulkan model sebelumnya** — kalau model salah menangkap detail penting dari hasil `search_tickets_by_keyword` (misal salah kutip angka/tanggal), ringkasan akhirnya ikut salah; ini bukan bug di `summarize_findings`-nya, tapi keterbatasan umum agent riset.
- **`search_tickets_by_keyword` di sini cuma exact-match dictionary buat ilustrasi** — real implementation butuh full-text search atau vector search (Phase 4) yang bisa menangkap variasi kata, bukan cuma keyword yang persis sama seperti yang di-hardcode.

### Kapan Dipakai
- Pakai pola multi-step research agent buat kebutuhan **riset/analisis lintas banyak sumber** (pattern di banyak tiket, tren komplain, dsb) yang hasilnya berupa laporan sintesis — bukan buat menjawab satu pertanyaan customer tunggal (itu cukup pakai agentic RAG topik 65, atau plain RAG Phase 4 topik 12).
- Cocok dijalankan sebagai task internal (misal dipicu tim ops/product yang butuh insight), bukan di jalur real-time customer-facing — riset semacam ini natural butuh lebih banyak waktu dan step dibanding respons customer service biasa.
- Kalau kebutuhannya cuma "cari 1 fakta spesifik dari 1 sumber", agent riset ini berlebihan — cukup tool calling sederhana atau agentic RAG.

### Sering Ditanya Saat Interview
- **Apa beda multi-step research agent dengan agentic RAG (topik 65)?** — agentic RAG fokus menjawab SATU pertanyaan dengan mekanisme retry kalau retrieval pertama belum cukup; research agent menjelajahi BANYAK sumber lintas beberapa putaran search buat menyusun satu laporan sintesis, bukan satu jawaban tunggal.
- **Kenapa multi-step research agent bisa dibangun langsung di atas `run_agent_loop` (Phase 6) tanpa mesin loop baru?** — karena mekanisme intinya sama (loop tool-calling yang berhenti begitu model kasih jawaban final); yang beda cuma tool apa yang disediakan (`search_tickets_by_keyword`, `summarize_findings`) dan `max_steps` yang biasanya lebih besar buat riset.
- **Apa yang dimaksud "search for missing info" dalam siklus research agent?** — begitu model membaca hasil pencarian awal dan nemuin celah/pertanyaan susulan (misal keyword baru yang muncul dari hasil sebelumnya), model melakukan pencarian lagi pakai keyword itu, bukan langsung menyimpulkan dari data yang tidak lengkap.
- **Kenapa `max_steps` buat research agent biasanya perlu lebih besar dibanding agent customer-service biasa?** — riset lintas banyak sumber butuh beberapa putaran search-then-refine sebelum data yang terkumpul cukup buat dibandingkan dan dirangkum, beda dari task customer-service yang biasanya selesai dalam 1-3 tool call.

---

## 67. Coding Agent

### Apa itu?
Coding Agent adalah agent (Phase 6) yang tool-nya dikhususkan buat berinteraksi dengan kode: baca file/cari kode (**Read**), menyusun rencana perbaikan (**Plan** — ini terjadi di "kepala" LLM, gak perlu tool khusus), mengubah isi file (**Edit**), menjalankan test (**Test**), dan membaca hasilnya buat menentukan langkah berikutnya (**Observe**) — kalau test gagal, dia ulangi Edit dengan perbaikan baru (**Fix**), sampai test-nya lolos atau kesempatan (`max_steps`) habis.

### Kenapa dibutuhkan?
Debugging manual itu proses yang sama persis siklusnya: baca kode yang bermasalah, coba pahami bug-nya, ubah kodenya, jalankan test, lihat masih gagal atau enggak, ulangi kalau masih gagal. Ini persis pola agent loop (Phase 6 topik 25) — bedanya cuma tool yang dipakai (`read_file`, `write_file`, `run_tests`) gantinya tool customer-service (`get_ticket_status`, dst). Coding agent menyelesaikan masalah "engineer harus duduk manual ngulang siklus ini" dengan membiarkan LLM sendiri yang menjalankan siklus itu — baca bug, usulkan fix, tes, dan kalau masih gagal, coba fix lain, tanpa perlu manusia campur tangan di setiap iterasi.

### Cara Kerja
```
Bug report ("apply_discount hasilnya salah")
    → read_file (Read: lihat isi kode yang bermasalah)
    → LLM susun rencana fix (Plan, di kepala LLM, gak ada tool-nya)
    → write_file (Edit: tulis ulang kode dengan fix yang diusulkan)
    → run_tests (Test: jalankan test case terhadap kode yang baru)
    → LLM baca hasil run_tests (Observe)
        → kalau test PASS  → selesai, jawaban final
        → kalau test FAIL  → LLM baca pesan error, susun fix BARU (Fix),
                              balik lagi ke write_file → run_tests (Repeat)
    → (dibatasi max_steps dari run_agent_loop, sama seperti agent lain)
```
Ini dibangun di atas `run_agent_loop` (Phase 6) yang sama — `read_file`, `write_file`, `run_tests` didaftarkan sebagai tool biasa, dan LLM sendiri yang memutuskan kapan harus baca, kapan harus edit, kapan harus test, dan kapan udah cukup yakin buat berhenti.

### Contoh Kode — Python
"Project" yang disederhanakan jadi satu dictionary in-memory (gantinya baca/tulis file asli di disk — supaya syllabus ini gak perlu benar-benar bikin project buat ditest), berisi satu function yang ada bug-nya:
```python
FAKE_FILES: dict[str, str] = {
    "supportpilot/discount.py": (
        "def apply_discount(price: float, percent: float) -> float:\n"
        "    # BUG: harusnya price dikurangi (price * percent / 100),\n"
        "    # bukan dikurangi percent mentah-mentah\n"
        "    return price - percent\n"
    )
}


def read_file(file_path: str) -> str:
    """
    Mock: baca isi 'file' dari FAKE_FILES (in-memory), gantinya baca file asli
    di disk — cukup buat mengilustrasikan alur read-plan-edit-test tanpa perlu
    benar-benar menyiapkan project di sistem file.
    """
    if file_path not in FAKE_FILES:
        return f"# ERROR: file '{file_path}' tidak ditemukan"
    return FAKE_FILES[file_path]


def write_file(file_path: str, content: str) -> dict:
    """
    Mock: timpa isi 'file' di FAKE_FILES dengan `content` baru (hasil edit/fix
    yang diusulkan model). Model harus kirim ISI LENGKAP file, bukan cuma
    baris yang berubah, karena mock ini cuma menimpa, bukan mem-patch sebagian.
    """
    FAKE_FILES[file_path] = content
    return {"file_path": file_path, "written": True}


def run_tests(file_path: str) -> dict:
    """
    Mock test runner — TAPI hasil pass/fail-nya BENERAN dihitung dari isi
    FAKE_FILES saat dipanggil (bukan angka yang di-hardcode): `exec()` isi
    file itu, panggil apply_discount dengan satu test case konkret, dan
    bandingkan hasilnya dengan nilai yang diharapkan. Jadi kalau write_file
    mengubah isi kodenya, hasil run_tests berikutnya beneran ikut berubah.
    """
    namespace: dict = {}
    try:
        exec(FAKE_FILES[file_path], namespace)
        hasil = namespace["apply_discount"](price=200.0, percent=10.0)
        if hasil == 180.0:
            return {"passed": True, "error": None}
        return {
            "passed": False,
            "error": f"apply_discount(price=200, percent=10) diharapkan 180, tapi hasilnya {hasil}",
        }
    except Exception as e:
        return {"passed": False, "error": f"{type(e).__name__}: {e}"}
```

Tool schema dan `TOOL_REGISTRY` buat ketiga tool coding agent ini:
```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "Baca isi sebuah file kode dari project SupportPilot berdasarkan path-nya.",
            "parameters": {
                "type": "object",
                "properties": {
                    "file_path": {
                        "type": "string",
                        "description": "Path file yang mau dibaca, contohnya 'supportpilot/discount.py'.",
                    }
                },
                "required": ["file_path"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "write_file",
            "description": "Timpa isi sebuah file kode dengan konten baru (hasil edit/fix).",
            "parameters": {
                "type": "object",
                "properties": {
                    "file_path": {"type": "string", "description": "Path file yang mau ditulis ulang."},
                    "content": {
                        "type": "string",
                        "description": "Isi kode baru LENGKAP yang menggantikan seluruh isi file.",
                    },
                },
                "required": ["file_path", "content"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "run_tests",
            "description": "Jalankan test untuk memverifikasi sebuah file kode, mengembalikan status pass/fail beserta pesan error kalau gagal.",
            "parameters": {
                "type": "object",
                "properties": {
                    "file_path": {"type": "string", "description": "Path file yang mau ditest."}
                },
                "required": ["file_path"],
            },
        },
    },
]

TOOL_REGISTRY = {
    "read_file": read_file,
    "write_file": write_file,
    "run_tests": run_tests,
}
```

Menjalankan coding agent-nya lewat `run_agent_loop` (Phase 6) yang sama:
```python
from openai import OpenAI

client = OpenAI()

hasil_akhir = run_agent_loop(
    client,
    user_message=(
        "Ada bug di supportpilot/discount.py — function apply_discount "
        "hasilnya salah. Tolong cari bug-nya, perbaiki, dan pastikan test-nya "
        "lolos dulu sebelum bilang selesai."
    ),
    tools=tools,
    max_steps=8,
)
print(hasil_akhir)
```
Yang terjadi di dalam loop — perhatikan iterasi FAIL lalu FIX-nya beneran didorong oleh nilai `passed` yang dikembalikan `run_tests`, bukan cuma diasumsikan lewat prosa:
1. **Step 1** — model manggil `read_file("supportpilot/discount.py")`, lihat isinya `return price - percent` — kelihatan mencurigakan karena `percent` dipakai mentah-mentah, bukan sebagai persentase dari `price`.
2. **Step 2** — model mengusulkan fix PERTAMA (masih salah): `return price - (price * percent)` (lupa bagi 100), lalu manggil `write_file` dengan isi itu.
3. **Step 3** — model manggil `run_tests("supportpilot/discount.py")`. Karena `FAKE_FILES` sekarang berisi kode hasil Step 2, `run_tests` benar-benar meng-`exec` kode itu dan menghitung `apply_discount(200, 10)` = `200 - (200 * 10)` = `-1800` — jauh dari `180`, jadi mengembalikan `{"passed": False, "error": "apply_discount(price=200, percent=10) diharapkan 180, tapi hasilnya -1800.0"}`.
4. **Step 4** — model membaca `error` itu (**Observe**), menyadari fix-nya lupa bagi 100, lalu mengusulkan fix KEDUA: `return price - (price * percent / 100)`, dan manggil `write_file` lagi buat menimpa isi file dengan versi ini.
5. **Step 5** — model manggil `run_tests` lagi. Kali ini `exec` menjalankan kode versi terbaru: `apply_discount(200, 10)` = `200 - (200 * 10 / 100)` = `200 - 20` = `180` — cocok dengan yang diharapkan, jadi `run_tests` mengembalikan `{"passed": True, "error": None}`.
6. **Step 6** — model melihat `passed: True`, tau tugasnya selesai, dan mengembalikan jawaban final teks biasa, misalnya: *"Bug di apply_discount sudah diperbaiki — sebelumnya percent dikurangi mentah-mentah dari price, sekarang dihitung sebagai persentase (price * percent / 100). Test sudah lolos."* — loop berhenti di step ke-6.

Karena `run_tests` di atas beneran meng-`exec` isi `FAKE_FILES` yang sudah diubah `write_file`, hasil pass/fail di Step 3 dan Step 5 itu bukan angka yang ditentukan di prosa — kalau fix di Step 2 KEBETULAN sudah benar, `run_tests` di Step 3 bakal langsung `passed: True` dan loop berhenti lebih awal; kalau fix di Step 4 masih salah lagi, `run_tests` di Step 5 bakal `passed: False` lagi dan model akan mencoba fix ketiga di step berikutnya (selama `max_steps` masih tersisa).

### Trade-off & Pitfall
- **`run_tests` di atas cuma `exec()` string kode tanpa sandboxing** — ini disederhanakan buat ilustrasi; PANGGIL LANGSUNG `exec()` terhadap kode yang ditulis LLM itu berbahaya di production (kode itu bisa aja berisi apa pun, termasuk yang gak diinginkan) — coding agent asli wajib menjalankan test di lingkungan terisolasi/sandbox (container terpisah, resource & network terbatas — Phase 14).
- **`write_file` di atas menimpa SELURUH isi file** — kalau file aslinya panjang dan model cuma "lihat" sebagian (misal karena dipotong buat menghemat context window), dia bisa gak sengaja menghapus bagian kode lain yang gak dia maksud ubah; makanya `read_file` harus ngasih isi LENGKAP file, bukan potongan, supaya model punya konteks penuh sebelum menulis ulang.
- **Loop ini tetap dibatasi `max_steps` dari `run_agent_loop`** — bug yang genuinely rumit mungkin gak berhasil diperbaiki dalam jumlah percobaan yang dianggarkan; begitu `max_steps` habis, `run_agent_loop` balikin fallback message (Phase 6 topik 25), yang berarti coding agent ini butuh eskalasi ke manusia, bukan dipaksa lanjut retry tanpa batas.
- **Sinyal "Observe" di sini sepenuhnya bergantung pada kualitas test yang ada** — kalau `run_tests` cuma menutupi sebagian kecil skenario (misal cuma satu test case seperti contoh di atas), fix yang "lolos test" belum tentu benar-benar memperbaiki SEMUA kasus penggunaan `apply_discount`; test suite yang gak representatif bikin coding agent percaya diri secara keliru.

### Kapan Dipakai
- Pakai coding agent buat bug fix yang **scope-nya jelas dan punya sinyal test yang reliable** — kalau ada test yang bisa mengonfirmasi "sudah benar/belum" secara objektif, loop read-edit-test-observe-fix ini bisa berjalan otomatis sampai lolos.
- Kalau project-nya BELUM punya test sama sekali buat area kode yang bermasalah, coding agent gak punya cara mendeteksi "fix ini beneran benar atau enggak" — perlu ditambahkan test dulu (baik manual atau minta agent-nya sendiri nulis test tambahan) sebelum loop semacam ini bisa dipercaya.
- Kurang cocok buat perubahan pada business logic yang kritis/sensitif (misal logic pembayaran) tanpa review manusia — sinyal pass/fail dari test itu penting, tapi belum tentu menangkap SEMUA implikasi bisnis dari sebuah perubahan; human review tetap perlu jadi gate terakhir buat perubahan berisiko tinggi.

### Sering Ditanya Saat Interview
- **Sebutkan siklus utama coding agent.** — Read (baca file/cari kode) → Plan (susun rencana fix, di "kepala" LLM) → Edit (tulis ulang kode) → Test (jalankan test) → Observe (baca hasil test) → kalau gagal, Fix (ulangi Edit dengan perbaikan baru) → berulang sampai test lolos atau kesempatan habis.
- **Kenapa coding agent bisa dibangun di atas `run_agent_loop` yang sama dengan agent lain (Phase 6)?** — mekanisme dasarnya identik (loop tool-calling yang berhenti begitu model kasih jawaban final teks biasa); yang beda cuma tool yang disediakan (`read_file`, `write_file`, `run_tests`) yang dikhususkan buat interaksi kode, bukan customer service atau riset.
- **Kenapa `run_tests` di contoh ini benar-benar meng-`exec()` kode, bukan cuma mengembalikan status hardcoded?** — supaya hasil pass/fail-nya beneran merefleksikan isi kode TERKINI (yang bisa berubah tiap kali model manggil `write_file`) — kalau di-hardcode, loop retry-nya cuma kelihatan iterasi di prosa tapi gak benar-benar bereaksi ke perubahan kode yang dilakukan model.
- **Kenapa `exec()` langsung terhadap kode yang ditulis model itu berisiko, dan gimana solusinya di production?** — kode yang dihasilkan LLM bisa aja gak sesuai harapan atau bahkan berbahaya kalau dijalankan tanpa batasan; production coding agent perlu menjalankan test di sandbox terisolasi (container terpisah dengan resource/network terbatas) daripada `exec()` polos seperti di contoh sederhana ini (Phase 14).

---

**Selanjutnya:** [Phase 18 — AI Infrastructure](./phase-18-ai-infrastructure.md)
