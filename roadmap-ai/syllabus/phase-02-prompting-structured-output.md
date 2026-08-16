# Phase 02 — Prompting & Structured Output

> Bagian dari [AI Engineering / Agent Roadmap](../README.md)

---

## 6. Prompt Engineering

### Apa itu?
Prompt engineering adalah skill nyusun instruksi (prompt) ke LLM supaya jawabannya konsisten, relevan, dan sesuai format yang lu mau. Ini bukan soal "nulis kalimat yang sopan ke AI" — ini soal ngasih model sebanyak mungkin sinyal yang jelas: siapa dia (role), apa yang harus dikerjain (task), info tambahan yang relevan (context), batasan yang gak boleh dilanggar (constraints), dan bentuk output yang diharapkan (output format). Kalau salah satu bagian ini hilang atau ambigu, model bakal "nebak-nebak" — dan tebakannya sering meleset dari yang lu maksud.

### Kenapa dibutuhkan?
Karena LLM itu pada dasarnya generator teks yang sangat fleksibel — dia bisa jawab apa aja, dalam format apa aja, sepanjang apa aja. Fleksibilitas ini bagus buat chatbot bebas, tapi jadi masalah kalau kode SupportPilot butuh output yang **bisa diproses program** (misal: label urgency buat auto-routing tiket). Tanpa prompt yang terstruktur, dua request dengan pertanyaan yang sama secara esensial bisa menghasilkan format jawaban yang beda-beda — kadang satu kata, kadang satu paragraf penjelasan, kadang malah nanya balik. Itu bikin kode yang manggil LLM jadi rapuh (fragile) karena harus nebak-nebak cara parsing jawabannya.

### Cara Kerja
Pola yang paling sering dipakai buat nyusun prompt yang solid adalah lima komponen berikut — biasa disingkat **Role + Task + Context + Constraints + Output Format**:

- **Role** — siapa persona model itu ("kamu adalah classifier urgency untuk tiket customer support"). Ini men-set "kacamata" yang dipakai model buat menjawab.
- **Task** — instruksi konkret, satu kalimat, tentang apa yang harus dikerjain.
- **Context** — informasi tambahan yang relevan biar model gak menjawab secara generik (misal: domain bisnisnya apa, contoh kasus yang biasa muncul).
- **Constraints** — batasan eksplisit (jangan jelasin alasan, jangan pakai tanda baca, harus salah satu dari beberapa pilihan tertentu, dll).
- **Output Format** — bentuk jawaban yang diharapkan secara persis (satu kata, JSON, list, dll).

Selain itu ada dua gaya pemberian contoh ke model:
- **Zero-shot** — langsung kasih instruksi tanpa contoh jawaban. Cocok buat task sederhana yang polanya jelas dari instruksi doang.
- **Few-shot** — kasih beberapa contoh pasangan input→output di dalam prompt sebelum pertanyaan sebenarnya. Berguna kalau task-nya nuanced atau formatnya susah dijelaskan cuma lewat kalimat (model "belajar dari contoh" di dalam satu prompt yang sama, bukan lewat training ulang).

```
[Role] + [Task] + [Context] + [Constraints] + [Output Format] → Prompt yang solid
```

### Contoh Kode — Python
Bayangin kita mau bikin fitur SupportPilot yang otomatis klasifikasi urgency dari pesan customer (buat nentuin tiket mana yang harus ditangani duluan). Di bawah ini dua versi prompt buat task yang sama persis — satu ditulis asal-asalan, satu ditulis pakai struktur Role+Task+Context+Constraints+Output Format.

Prompt yang **kurang terstruktur** — cuma nanya langsung tanpa ngasih tau format jawaban yang diharapkan:
```python
messages_poor = [
    {
        "role": "user",
        "content": "Ini pesan urgent gak: 'aplikasi saya error terus gak bisa checkout, saldo kepotong 2x'?",
    },
]
```

Prompt yang **terstruktur** dengan lima komponen di atas:
```python
messages_good = [
    {
        "role": "system",
        "content": (
            # Role: siapa persona model ini
            "Kamu adalah classifier urgency untuk tiket customer support SupportPilot.\n\n"
            # Task: instruksi konkret satu kalimat
            "Task: klasifikasikan urgency pesan customer ke salah satu dari: low, medium, high, critical.\n\n"
            # Context: info domain yang relevan biar model gak menjawab generik
            "Context: SupportPilot melayani customer e-commerce. Pesan yang menyebut uang hilang, "
            "transaksi gagal, atau akun terblokir biasanya masuk kategori high atau critical.\n\n"
            # Constraints: batasan eksplisit soal apa yang boleh/gak boleh ada di jawaban
            "Constraints: jawab HANYA dengan satu kata urgency-nya, tanpa penjelasan tambahan, "
            "tanpa tanda baca, huruf kecil semua.\n\n"
            # Output format: bentuk jawaban yang diharapkan secara persis
            "Output format: satu kata, salah satu dari: low, medium, high, critical."
        ),
    },
    {
        "role": "user",
        "content": "Aplikasi saya error terus gak bisa checkout, padahal saldo sudah kepotong 2 kali.",
    },
]
```

Manggil salah satu dari dua prompt di atas persis sama caranya (lihat Phase 1) — bedanya cuma di `messages` yang dikirim:
```python
from openai import OpenAI

client = OpenAI()

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=messages_good,
    temperature=0,  # rendah karena ini task klasifikasi, butuh konsisten
)

print(response.choices[0].message.content)  # diharapkan: "high" (satu kata doang)
```

Bedanya di praktik: `messages_poor` gak ngasih tau model harus jawab dalam bentuk apa — jadi model bisa aja jawab `"Ya, ini urgent."`, atau `"Ini kelihatannya cukup mendesak karena menyangkut uang."`, atau bahkan format lain lagi di run berikutnya untuk pertanyaan yang mirip. Kode yang manggil LLM ini gak bisa dengan aman melakukan `if jawaban == "high":` karena jawabannya gak konsisten bentuknya. `messages_good` secara eksplisit ngunci output ke satu kata dari vocabulary tertutup (`low|medium|high|critical`), jadi hasilnya predictable dan gampang diproses lebih lanjut oleh kode.

### Trade-off & Pitfall
- **Prompt yang makin detail = makin banyak token = makin mahal & makin lambat.** Ada titik "cukup" — jangan overload prompt dengan instruksi yang gak relevan ke task-nya.
- **Few-shot examples makan context window.** Kalau contohnya kebanyakan atau kepanjangan, bisa habisin budget token yang harusnya buat data/history yang sebenarnya penting.
- **Constraint yang terlalu ketat kadang bikin model "maksa" jawab** meski gak yakin — misal dipaksa pilih satu dari 4 kategori padahal kasusnya ambigu. Perlu dipikirkan apakah butuh kategori "unknown/unclear" sebagai fallback.
- **Prompt yang jalan bagus di satu model belum tentu jalan sama bagusnya di model lain** — tiap model punya "kebiasaan" mengikuti instruksi yang sedikit beda.

### Kapan Dipakai
- Selalu pakai struktur Role+Task+Context+Constraints+Output Format buat task yang **outputnya bakal diproses program** (klasifikasi, ekstraksi, routing) — bukan cuma buat chat bebas ke user.
- Pakai **few-shot** kalau task-nya susah dijelasin lewat instruksi doang, atau kalau format outputnya nuanced dan lu punya beberapa contoh baik yang representatif.
- Pakai **zero-shot** buat task sederhana yang polanya cukup jelas dari instruksi (contoh di atas sebenarnya masih jalan baik dengan zero-shot karena tugasnya cukup straightforward).

### Sering Ditanya Saat Interview
- **Apa itu struktur Role+Task+Context+Constraints+Output Format, dan kenapa penting?** — kerangka buat nyusun prompt supaya model punya sinyal yang jelas soal persona, tugas, info relevan, batasan, dan bentuk jawaban — mengurangi variasi/ambiguitas output.
- **Apa beda zero-shot dan few-shot prompting?** — zero-shot cuma kasih instruksi tanpa contoh, few-shot menyertakan beberapa contoh input→output di dalam prompt yang sama biar model "meniru pola" contoh tersebut.
- **Kenapa prompt yang gak menyebutkan format output itu berisiko di production?** — karena jawaban model jadi gak konsisten bentuknya antar request, sehingga kode yang parsing jawabannya jadi rapuh dan gampang gagal.
- **Apakah nambah instruksi ke prompt selalu memperbaiki hasil?** — enggak — instruksi berlebihan/gak relevan nambah token (biaya & latency) dan kadang malah bikin model bingung; instruksi harus relevan dan seperlunya.

---

## 7. Structured Output

### Apa itu?
Structured output adalah pendekatan supaya jawaban LLM **selalu** mengikuti skema/struktur data tertentu (misal JSON dengan field dan tipe data yang sudah ditentukan), bukan sekadar teks bebas. Alih-alih model jawab kalimat "Urgency-nya tinggi ya, kira-kira 90% yakin", kita minta model balikin sesuatu seperti `{"urgency": "high", "category": "billing", "confidence": 0.9}` yang bentuknya sudah pasti dan bisa langsung dipakai kode.

### Kenapa dibutuhkan?
Free-text output itu rapuh (fragile) buat production karena kode yang manggil LLM biasanya butuh **field spesifik** buat ambil keputusan (misal: `if urgency == "high": route_to_senior_agent()`). Kalau outputnya cuma teks bebas, kode harus nge-parse teks itu pakai regex/string matching yang gampang jebol tiap kali model sedikit ubah gaya bahasanya (padahal secara semantik jawabannya sama). Structured output memindahkan tanggung jawab "menjaga format konsisten" dari kode manual (parsing rapuh) ke lapisan yang lebih andal (schema + validasi eksplisit).

### Cara Kerja
Alurnya:
```
Definisikan schema → Minta LLM jawab sesuai schema → Parse output → Validasi → (kalau invalid: retry)
```

- **JSON Schema** — deskripsi formal soal field apa aja yang harus ada di output, tipe datanya apa, mana yang wajib (required). Beberapa provider LLM punya fitur "structured output" native yang menjamin (atau nyaris menjamin) output mengikuti schema ini.
- **Validation** — proses ngecek apakah JSON yang dibalikin model beneran sesuai schema (field-nya lengkap, tipenya benar, dll) sebelum dipakai lebih lanjut oleh kode.
- **Retry-on-invalid-output** — kalau validasi gagal (model ngasih JSON yang salah format atau field-nya gak lengkap), strategi paling umum adalah panggil ulang model, kasih tau apa yang salah dari output sebelumnya, dan minta perbaiki — bukan langsung nyerah di percobaan pertama.

Di Python, cara paling umum buat mendefinisikan schema dan sekaligus dapet validasi otomatis adalah pakai library **pydantic**. Kalau ini pertama kalinya lu lihat pola ini: pydantic adalah library yang biarin lu definisikan "bentuk data" pakai `class` biasa dengan anotasi tipe (`field_name: tipe`), terus otomatis nge-validasi dan mengonversi data mentah (misal dict dari JSON) ke instance class itu — kalau ada field yang hilang atau tipenya salah, dia otomatis melempar error yang jelas, tanpa kita harus nulis validasi manual satu-satu.

### Contoh Kode — Python
Pertama, definisikan schema klasifikasi tiket pakai pydantic:
```python
from pydantic import BaseModel


class TicketClassification(BaseModel):
    urgency: str      # salah satu dari: low, medium, high, critical
    category: str     # misal: billing, shipping, technical, account
    confidence: float  # skor keyakinan model, dari 0.0 sampai 1.0
```

Kalau provider/SDK yang dipakai mendukung structured output secara native (misal OpenAI SDK versi terbaru punya `client.beta.chat.completions.parse`), kita bisa langsung minta modelnya jawab sesuai schema `TicketClassification` di atas, dan SDK-nya yang urus parsing + validasi:
```python
from openai import OpenAI

client = OpenAI()

response = client.beta.chat.completions.parse(
    model="gpt-4o-mini",
    messages=[
        {
            "role": "system",
            "content": "Kamu adalah classifier tiket customer support SupportPilot. "
            "Klasifikasikan urgency (low/medium/high/critical) dan category "
            "(billing/shipping/technical/account) dari pesan customer, plus confidence score-mu.",
        },
        {
            "role": "user",
            "content": "Saldo saya kepotong 2 kali tapi order gagal terus, tolong bantu secepatnya!",
        },
    ],
    response_format=TicketClassification,  # SDK otomatis parse & validasi ke schema ini
)

# .parsed sudah berupa instance TicketClassification, bukan dict/string mentah
classification: TicketClassification = response.choices[0].message.parsed
print(classification.urgency, classification.category, classification.confidence)
```

Kalau provider/model yang dipakai gak punya dukungan structured output native, kita bisa lakuin manual: minta model balas JSON lewat instruksi prompt biasa, lalu parse + validasi sendiri pakai pydantic, dan **retry** kalau gagal:
```python
import json
from pydantic import ValidationError


def classify_ticket_with_retry(
    client, user_message: str, max_retries: int = 2
) -> TicketClassification:
    """
    Fallback manual buat provider yang gak dukung structured output native.
    Kalau output model invalid (bukan JSON, atau field-nya gak sesuai schema),
    kita kasih tau error-nya ke model dan minta perbaiki di percobaan berikutnya.
    """
    messages = [
        {
            "role": "system",
            "content": (
                "Kamu adalah classifier tiket customer support. Balas HANYA dengan JSON valid "
                'berformat persis: {"urgency": "...", "category": "...", "confidence": 0.0} '
                "tanpa teks tambahan apapun di luar JSON."
            ),
        },
        {"role": "user", "content": user_message},
    ]

    last_error: Exception | None = None

    for _ in range(max_retries + 1):
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=messages,
            temperature=0,
        )
        raw_content = response.choices[0].message.content

        try:
            parsed_json = json.loads(raw_content)
            return TicketClassification(**parsed_json)  # validasi terjadi di sini
        except (json.JSONDecodeError, ValidationError) as e:
            last_error = e
            # tambahin jawaban model yang gagal + feedback error-nya ke history,
            # supaya percobaan berikutnya modelnya "tau" apa yang harus diperbaiki
            messages.append({"role": "assistant", "content": raw_content})
            messages.append(
                {
                    "role": "user",
                    "content": f"Output kamu invalid ({e}). Perbaiki dan balas ulang HANYA dengan JSON valid.",
                }
            )

    raise ValueError(f"Gagal dapat output valid setelah beberapa kali percobaan: {last_error}")
```

### Trade-off & Pitfall
- **Structured output native gak dijamin ada di semua provider/model** — fallback manual (prompt + parse + retry) tetap perlu dikuasai buat kasus model yang gak dukung fitur ini.
- **Retry nambah latency dan biaya.** Tiap kali validasi gagal dan kita retry, itu berarti panggilan API tambahan — kalau model sering gagal format, ini bisa jadi mahal. Batasi `max_retries` supaya gak infinite loop.
- **Schema yang terlalu longgar (misal semua field `str` tanpa batasan nilai) gak banyak membantu.** Field `urgency: str` masih bisa diisi model dengan nilai di luar `low/medium/high/critical` — validasi lebih ketat (misal pakai `Literal["low", "medium", "high", "critical"]` di pydantic) sebaiknya dipakai kalau butuh jaminan lebih kuat.
- **Confidence score dari model bukan probabilitas statistik yang presisi** — itu cuma perkiraan yang di-generate model, bukan angka yang dihitung dari distribusi probabilitas token secara langsung. Jangan terlalu percaya diri menganggapnya akurat sampai 2 desimal.

### Kapan Dipakai
- Pakai structured output tiap kali jawaban LLM **bakal diproses lebih lanjut oleh kode** (routing, filtering, disimpan ke database) — jangan andalkan parsing teks bebas buat kasus ini.
- Kalau provider/SDK yang dipakai punya dukungan native (kayak `parse()` di contoh atas), prioritaskan itu — lebih sedikit kode manual dan lebih reliable.
- Kalau gak ada dukungan native, selalu siapkan **validasi + retry** — jangan asumsikan model akan selalu patuh 100% ke instruksi format walau sudah diminta dengan jelas.

### Sering Ditanya Saat Interview
- **Kenapa free-text output dianggap fragile buat production?** — karena kode harus parsing teks bebas yang formatnya gampang berubah-ubah antar request, sehingga logic parsing-nya gampang jebol.
- **Apa peran pydantic di structured output?** — mendefinisikan schema data lewat class dengan anotasi tipe, lalu otomatis memvalidasi dan mengonversi data mentah (misal dict dari JSON) ke instance class tersebut, melempar error jelas kalau ada field yang gak sesuai.
- **Gimana strategi handling kalau output LLM gak valid sesuai schema?** — retry: panggil ulang model, kasih tau apa yang salah dari output sebelumnya, minta perbaiki, dengan batas jumlah percobaan supaya gak infinite loop.
- **Apa bedanya structured output native vs manual JSON parsing?** — native (kalau didukung provider) menjamin/mendekati jaminan model mengikuti schema di level API, sedangkan manual berarti kita minta lewat instruksi prompt biasa lalu validasi sendiri di sisi kode — lebih rentan gagal tapi bekerja di semua provider.

---

## 8. Function / Tool Calling

### Apa itu?
Function/tool calling adalah kemampuan LLM buat "meminta" kode di luar dirinya buat menjalankan sebuah fungsi tertentu, lalu memakai hasilnya buat menyusun jawaban akhir. Ini pergeseran penting: dari sekadar chatbot yang cuma bisa "ngomong" berdasarkan apa yang udah dia tau dari training, jadi sistem yang bisa **bertindak** — ambil data real-time, panggil API eksternal, jalanin operasi — lewat perantaraan kode yang kita tulis sendiri. Ini adalah fondasi dari apa yang bakal disebut "agent" di phase-phase berikutnya.

### Kenapa dibutuhkan?
LLM gak punya akses langsung ke data SupportPilot (database tiket, status order, dll) — pengetahuannya cuma dari data training yang statis. Kalau customer nanya "status tiket T-123 saya gimana?", model gak bisa jawab akurat cuma dari "pengetahuan umum"-nya — dia butuh cara buat **manggil fungsi nyata** yang bisa ambil data tiket itu, baru menyusun jawaban berdasarkan data real tersebut. Tool calling adalah mekanisme resmi yang disediakan provider LLM buat menjembatani ini, tanpa kita harus nebak-nebak dari teks bebas kapan model "pengen" manggil fungsi apa.

### Cara Kerja
Ada beberapa istilah kunci yang perlu dipahami sebelum masuk ke kode:

- **Tool definition** — deskripsi sebuah fungsi yang "ditawarkan" ke model: nama fungsi, deskripsi kegunaannya, dan skema parameter yang dibutuhkan (mirip JSON Schema dari topik sebelumnya).
- **Tool schema** — bagian dari tool definition yang mendeskripsikan parameter apa aja yang dibutuhkan fungsi itu (nama parameter, tipe data, mana yang wajib).
- **Tool arguments** — nilai konkret yang "diisi" model buat parameter-parameter itu, berdasarkan konteks percakapan (misal model nentuin `ticket_id="T-123"` dari pesan user).
- **Tool execution** — proses beneran menjalankan fungsi itu dengan argument yang dikasih model. **Ini bagian yang PENTING dipahami: model sendiri gak bisa menjalankan kode apapun** — dia cuma bisa "meminta" (request) supaya sebuah tool dipanggil dengan argument tertentu. Kode kita (si caller) yang bertanggung jawab benar-benar menjalankan fungsinya.
- **Tool result** — hasil dari eksekusi fungsi tadi, yang kemudian dikirim balik ke model (sebagai pesan baru dengan role khusus) supaya model bisa menyusun jawaban akhir berdasarkan data itu.
- **Tool error handling** — kalau eksekusi tool gagal (misal ticket_id gak ketemu), hasil error itu juga sebaiknya dikirim balik ke model (bukan bikin program crash), supaya model bisa menjelaskan situasinya ke user dengan wajar.
- **Tool-calling loop** — karena satu pertanyaan user kadang butuh beberapa kali panggil-tool-lalu-panggil-model-lagi secara berurutan, alur ini sering dibungkus jadi sebuah loop: kirim ke model → cek apakah dia minta tool call → kalau ya, eksekusi lalu kirim hasilnya balik dan ulangi → kalau enggak (dia udah kasih jawaban teks), keluar dari loop.

Alur satu putaran (round-trip) tool calling secara garis besar:
```
User message → LLM → "aku mau manggil tool X dengan argument Y"
     → Caller eksekusi tool X(Y) → hasil dikirim balik ke LLM
     → LLM susun jawaban akhir berdasarkan hasil tool → Response ke user
```

Di SupportPilot, kita bikin helper `call_llm_with_tools(client, messages, tools)` yang tugasnya **cuma** membungkus panggilan API dan mendeteksi apakah model minta tool call — dia **tidak** menjalankan tool-nya sendiri (itu tetap tanggung jawab kode yang manggil dia), supaya helper ini tetap generic dan gak perlu tau isi/implementasi tiap tool yang ada.

### Contoh Kode — Python
Pertama, definisikan tool pertama SupportPilot: `get_ticket_status`. Untuk sekarang ini masih mock (data palsu) — versi yang query ke database beneran baru muncul di Phase 3/4, di sini kita fokus dulu ke alur tool calling-nya:
```python
def get_ticket_status(ticket_id: str) -> dict:
    """
    Mock function: belum nyambung ke database beneran (itu baru di Phase 3/4).
    Untuk sekarang selalu balikin data palsu, supaya kita bisa fokus belajar
    alur tool calling tanpa perlu setup database dulu.
    """
    return {
        "ticket_id": ticket_id,
        "status": "open",
        "priority": "high",
        "last_updated": "2026-08-14T10:00:00Z",
    }
```

Lalu, tool definition-nya — ini yang dikasih ke model supaya dia tau tool ini ada dan gimana cara "manggilnya":
```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_ticket_status",
            "description": "Ambil status dan detail sebuah tiket customer support berdasarkan ticket_id.",
            "parameters": {
                "type": "object",
                "properties": {
                    "ticket_id": {
                        "type": "string",
                        "description": "ID tiket customer, contohnya 'T-123'.",
                    }
                },
                "required": ["ticket_id"],
            },
        },
    }
]
```

Sekarang helper `call_llm_with_tools`. Perhatikan: fungsi ini cuma mendeteksi & melaporkan permintaan tool call ke caller — dia tidak menjalankan tool-nya sendiri:
```python
import json


def call_llm_with_tools(client, messages: list[dict], tools: list[dict]) -> dict:
    """
    Bungkus panggilan LLM yang mendukung tool calling.

    Return value:
    - Kalau model minta manggil tool:
        {"type": "tool_call", "tool_call_id": ..., "tool_name": ..., "tool_arguments": {...}}
    - Kalau model langsung jawab teks biasa (gak butuh tool):
        {"type": "message", "content": "..."}
    """
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=messages,
        tools=tools,
    )

    message = response.choices[0].message

    if message.tool_calls:
        # Ambil tool call pertama saja untuk contoh ini
        # (model bisa saja minta beberapa tool call sekaligus, tapi kita sederhanakan dulu)
        tool_call = message.tool_calls[0]
        return {
            "type": "tool_call",
            "tool_call_id": tool_call.id,
            "tool_name": tool_call.function.name,
            # argument dari API balik dalam bentuk string JSON, jadi harus di-parse dulu
            "tool_arguments": json.loads(tool_call.function.arguments),
        }

    return {"type": "message", "content": message.content}
```

Terakhir, contoh lengkap round-trip: user nanya status tiket → `call_llm_with_tools` mendeteksi model minta manggil `get_ticket_status` → kode kita yang eksekusi mock function-nya → hasilnya dikirim balik ke model → model menyusun jawaban akhir dalam bahasa natural:
```python
from openai import OpenAI

client = OpenAI()

messages = [
    {
        "role": "system",
        "content": "Kamu adalah asisten customer support SupportPilot. "
        "Gunakan tool yang tersedia kalau butuh data spesifik soal tiket.",
    },
    {
        "role": "user",
        "content": "Halo, gimana status tiket T-123 saya?",
    },
]

# Langkah 1: tanya ke model, kasih tau tool apa aja yang tersedia
result = call_llm_with_tools(client, messages, tools)

if result["type"] == "tool_call" and result["tool_name"] == "get_ticket_status":
    # Langkah 2: caller (bukan call_llm_with_tools) yang benar-benar eksekusi tool-nya
    ticket_data = get_ticket_status(**result["tool_arguments"])

    # Langkah 3a: catat dulu ke history bahwa assistant tadi minta tool call ini
    messages.append(
        {
            "role": "assistant",
            "tool_calls": [
                {
                    "id": result["tool_call_id"],
                    "type": "function",
                    "function": {
                        "name": result["tool_name"],
                        "arguments": json.dumps(result["tool_arguments"]),
                    },
                }
            ],
        }
    )

    # Langkah 3b: masukin hasil eksekusi tool ke history, dengan role khusus "tool"
    # dan tool_call_id yang sama supaya model tau ini jawaban buat request yang mana
    messages.append(
        {
            "role": "tool",
            "tool_call_id": result["tool_call_id"],
            "content": json.dumps(ticket_data),
        }
    )

    # Langkah 4: panggil lagi modelnya dengan history yang sudah lengkap + hasil tool,
    # supaya dia bisa susun jawaban akhir dalam bahasa natural
    final_result = call_llm_with_tools(client, messages, tools)
    print(final_result["content"])
    # contoh output: "Tiket T-123 kamu statusnya masih open dengan priority high, ya."
else:
    # Model gak butuh tool sama sekali, langsung jawab teks biasa
    print(result["content"])
```

### Trade-off & Pitfall
- **Model gak selalu memilih tool yang "benar".** Kadang model salah nebak tool mana yang relevan, atau ngisi argument yang gak sesuai — validasi argument sebelum eksekusi (misal cek `ticket_id` formatnya benar) tetap penting, jangan asumsikan argument dari model selalu valid.
- **Lupa mengembalikan hasil tool ke model = model "buta" terhadap hasil eksekusi.** Kalau langkah 3a/3b di atas dilewatin, model gak akan pernah tau hasil tool-nya, dan gak bisa menyusun jawaban akhir yang benar.
- **Tool error harus ditangani secara eksplisit** — kalau `get_ticket_status` melempar exception (misal karena versi non-mock: ticket_id gak ketemu), sebaiknya error itu ditangkap dan hasilnya (pesan error) tetap dikirim balik ke model sebagai tool result, supaya model bisa menjelaskan ke user dengan wajar, bukan bikin keseluruhan aplikasi crash.
- **Banyak tool call berurutan = banyak panggilan API = lebih lambat & lebih mahal.** Kalau satu permintaan user butuh berkali-kali round-trip tool calling, latency totalnya bisa terasa signifikan buat user yang nunggu jawaban real-time.
- **Tool description yang gak jelas bikin model salah pilih tool** (kalau ada banyak tool tersedia) atau salah ngisi argument — deskripsi tool sama pentingnya kayak system prompt.

### Kapan Dipakai
- Pakai tool calling kalau LLM butuh **data yang gak ada di training-nya** dan berubah-ubah (status tiket, data order, harga terkini) — bukan pengetahuan umum yang statis.
- Pakai tool calling kalau LLM butuh **melakukan aksi** (bukan cuma jawab pertanyaan) — misal update status tiket, kirim notifikasi.
- Kalau task-nya cuma butuh klasifikasi/ekstraksi dari teks yang sudah dikasih di prompt (gak butuh data eksternal), structured output (topik 7) biasanya cukup — gak perlu tool calling.
- Ini adalah building block utama buat "agent" (Phase 6 dst) — paham alur ini di level dasar krusial sebelum lanjut ke sistem yang lebih kompleks (multi-tool, multi-step reasoning).

### Sering Ditanya Saat Interview
- **Apa itu tool/function calling, dan kenapa penting buat agent?** — kemampuan LLM meminta eksekusi fungsi eksternal lewat kode caller, lalu memakai hasilnya buat menyusun jawaban; ini fondasi yang membuat LLM bisa "bertindak", bukan cuma "ngomong" — dasar dari konsep agent.
- **Siapa yang benar-benar mengeksekusi tool: model atau kode kita?** — kode kita (caller). Model cuma "meminta" tool call dengan argument tertentu; dia gak punya kemampuan menjalankan kode apapun sendiri.
- **Kenapa hasil eksekusi tool harus dikirim balik ke model?** — supaya model bisa menyusun jawaban akhir berdasarkan data real hasil eksekusi, bukan cuma berdasarkan "tebakan" dari pengetahuan training-nya.
- **Gimana cara handle kalau tool call gagal dieksekusi (misal argument invalid atau error)?** — tangkap error-nya, kirim balik ke model sebagai tool result (bukan biarin program crash), supaya model bisa menjelaskan situasi tersebut ke user dengan wajar.

---

**Selanjutnya:** [Phase 03 — Embeddings](./phase-03-embeddings.md)
