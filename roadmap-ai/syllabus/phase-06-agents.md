# Phase 06 — Agents

> Bagian dari [AI Engineering / Agent Roadmap](../README.md)

---

## 24. What is an AI Agent?

### Apa itu?
AI Agent adalah pergeseran cara LLM dipakai: dari sekadar "jawab pertanyaan sekali jalan" jadi "mikir, bertindak, amati hasilnya, lalu putuskan langkah berikutnya" — berulang-ulang sampai tugasnya benar-benar kelar. Di topik 8 (Phase 2) kita udah lihat LLM bisa manggil satu tool buat dapat data. Agent adalah generalisasi dari itu: bukan cuma satu kali panggil-tool-lalu-jawab, tapi sebuah **loop** di mana LLM sendiri yang memutuskan tool apa yang dipakai, kapan berhenti manggil tool, dan kapan dia udah cukup yakin buat kasih jawaban final.

### Kenapa dibutuhkan?
Banyak permintaan customer SupportPilot itu **gak bisa diselesaikan dalam satu langkah**. Contoh: "gimana status order saya, dan kalau ternyata telat tolong bukain tiket ya" — ini butuh: (1) cek status order dulu, (2) baru berdasarkan hasilnya, putuskan apakah perlu bikin tiket atau enggak, (3) baru kasih jawaban final ke customer. Plain LLM call (satu kali kirim pertanyaan, satu kali dapat jawaban) gak punya cara buat "berhenti sejenak, cari tau sesuatu, lalu lanjut mikir berdasarkan hasil itu". Agent menyelesaikan ini dengan membungkus proses "mikir → tindakan → amati → mikir lagi" jadi sebuah loop yang jalan otomatis sampai tugasnya selesai.

### Cara Kerja
Siklus kerja agent, dikontraskan dengan plain LLM call:
```
Plain LLM call (satu putaran, gak ada loop):
  User pertanyaan → LLM → Jawaban langsung

AI Agent (loop, berulang sampai selesai):
  User pertanyaan
      → LLM: OBSERVE (lihat pertanyaan + hasil tool sebelumnya, kalau ada)
      → LLM: THINK/DECIDE (perlu tool gak buat lanjut? tool mana? atau udah cukup info?)
      → kalau perlu tool: ACT (tool itu benar-benar dieksekusi oleh kode, BUKAN oleh LLM)
      → OBSERVE RESULT (hasil tool dikasih balik ke LLM sebagai konteks baru)
      → REPEAT (balik ke THINK/DECIDE) ... sampai LLM yakin bisa kasih Jawaban Final
      → Jawaban Final
```
Bedanya secara mendasar: plain LLM call itu satu putaran doang (input masuk, output keluar, selesai). Agent adalah **loop** dari putaran-putaran kecil semacam itu, dengan tool sebagai "tangan" yang dipakai LLM buat berinteraksi dengan dunia di luar dirinya (database, API eksternal, dst) di antara tiap putaran mikirnya. Implementasi konkret loop ini — fungsi `run_agent_loop` — dibahas detail di topik 25.

### Trade-off & Pitfall
- **Agent lebih lambat dan lebih mahal** dibanding plain LLM call — tiap "putaran mikir" tambahan berarti panggilan API tambahan, dan customer harus nunggu semua putaran itu selesai sebelum dapat jawaban final.
- **Debugging agent lebih ribet** — kalau jawabannya salah, kita harus telusuri di putaran (step) mana LLM mulai "salah mikir" atau salah pilih tool, bukan cuma cek satu request/response tunggal kayak plain LLM call.
- **Agent bisa "nyasar"** — misal terus-terusan manggil tool yang sama tanpa progress, atau gak pernah yakin buat berhenti — kalau gak dibatasi jumlah langkah maksimal (dibahas di topik 25 lewat `max_steps`).
- **Gak semua kebutuhan cocok pakai agent** — task yang urutan tool-nya udah pasti/predictable biasanya cukup diselesaikan lewat tool calling biasa (Phase 2 topik 8) tanpa perlu loop penuh; ini dibahas lebih detail di topik 26.

### Kapan Dipakai
- Pakai agent kalau task-nya butuh **jumlah/urutan langkah yang gak bisa ditentuin di awal** — baru ketauan pas berjalan, tergantung hasil tool sebelumnya (misal: cek order dulu, baru tau perlu bikin tiket atau enggak).
- Kalau task-nya cuma butuh satu-dua tool call yang urutannya udah pasti, tool calling manual (Phase 2 topik 8) sudah cukup — gak perlu bikin agent loop penuh.
- Bedanya lebih detail sama pola "workflow" (yang urutannya dikontrol developer, bukan LLM) dibahas di topik 26.

### Sering Ditanya Saat Interview
- **Apa perbedaan mendasar antara plain LLM call dan AI agent?** — plain LLM call adalah satu putaran (input → output langsung); agent adalah loop dari beberapa putaran "mikir → bertindak → amati hasil" yang berulang sampai LLM sendiri memutuskan tugasnya selesai.
- **Sebutkan siklus kerja utama sebuah agent.** — Observe (lihat konteks & hasil tool sebelumnya) → Think/Decide (perlu tool lagi atau udah cukup) → Act (eksekusi tool) → Observe Result (hasil tool jadi konteks baru) → Repeat sampai selesai.
- **Kenapa agent butuh tool untuk berfungsi?** — karena LLM sendiri gak bisa "bertindak" di dunia nyata (mengambil data real-time, memanggil API, dsb) — tool adalah jembatan yang memungkinkan itu (lihat Phase 2 topik 8).
- **Apa risiko utama kalau agent gak dibatasi jumlah langkahnya?** — bisa terus muter tanpa henti (atau nyaris gak pernah berhenti) kalau LLM gak pernah "yakin" buat mengakhiri, yang berujung ke biaya dan latency yang membengkak.

---

## 25. Agent Loop

### Apa itu?
Agent Loop adalah implementasi konkret dari siklus Observe → Think/Decide → Act → Observe Result → Repeat yang dijelasin di topik 24, dibungkus jadi satu fungsi generic yang bisa dipakai ulang: `run_agent_loop(client, user_message, tools, max_steps=5) -> str`. Fungsi ini nge-loop manggil `call_llm_with_tools` (Phase 2) — kalau model minta tool call, kode kita eksekusi tool-nya dan kasih hasilnya balik ke model; kalau model udah kasih jawaban teks biasa, loop berhenti dan jawaban itu yang dibalikin ke customer.

### Kenapa dibutuhkan?
Tanpa fungsi generic ini, tiap kali SupportPilot butuh alur multi-step (cek order → mungkin bikin tiket → jawab customer), engineer harus nulis manual round-trip tool calling (topik 8) satu-satu, ngulang boilerplate yang sama, dan gampang lupa salah satu bagian penting — misalnya lupa kasih batas jumlah langkah maksimal, atau lupa nangkep error kalau tool-nya gagal. `run_agent_loop` menyatukan seluruh pola ini di satu tempat: sekali ditulis dengan benar, dipakai ulang buat skenario apa aja yang butuh sejumlah tool call yang gak pasti di awal.

### Cara Kerja
```
run_agent_loop(user_message):
    messages = [system prompt, user_message]

    ulangi sampai max_steps kali:
        result = call_llm_with_tools(messages, tools)

        kalau result adalah "message"   → SELESAI. return isi jawabannya.
        kalau result adalah "tool_call" → eksekusi tool yang diminta,
                                            catat tool call + hasilnya ke messages,
                                            lanjut ke iterasi berikutnya

    (kalau max_steps habis dan model MASIH minta tool call)
        → return pesan fallback, JANGAN diulang lagi tanpa batas
```
Dua hal yang bikin loop ini "genuinely" berhenti dengan benar, bukan cuma prosa:
- **Deteksi selesai** — `call_llm_with_tools` cuma punya dua kemungkinan balikan: `{"type": "message", ...}` (model udah final, gak minta tool lagi) atau `{"type": "tool_call", ...}` (model masih butuh data/aksi tambahan). `run_agent_loop` cukup cek `result["type"]` buat tau kapan harus berhenti.
- **Batas keras (`max_steps`)** — loop dibungkus `for step in range(max_steps)`, jadi walau model gak pernah mengembalikan `"message"`, loop tetap otomatis berhenti setelah `max_steps` iterasi dan balikin fallback — mencegah loop jalan tanpa henti (yang berarti biaya API dan latency yang gak terkendali).

### Contoh Kode — Python
Dua tool baru buat SupportPilot, mock function dengan gaya yang sama seperti `get_ticket_status` (Phase 2):
```python
def get_order_status(order_id: str) -> dict:
    """
    Mock function, gaya sama seperti get_ticket_status (Phase 2) — belum
    nyambung ke sistem order beneran, cukup buat belajar alur agent loop.
    """
    return {
        "order_id": order_id,
        "status": "delayed",
        "estimated_delivery": "2026-08-20",
        "carrier": "JNE",
    }


def create_support_ticket(customer_id: str, subject: str) -> dict:
    """
    Mock function: pura-pura bikin tiket baru di sistem SupportPilot.
    """
    return {
        "ticket_id": "T-789",
        "customer_id": customer_id,
        "subject": subject,
        "status": "open",
        "created_at": "2026-08-14T10:00:00Z",
    }
```

Tool definition (schema) buat kedua fungsi di atas, format yang sama seperti `tools` di Phase 2:
```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_order_status",
            "description": "Ambil status pengiriman sebuah order berdasarkan order_id.",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {
                        "type": "string",
                        "description": "ID order customer, contohnya 'O-456'.",
                    }
                },
                "required": ["order_id"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "create_support_ticket",
            "description": "Buka tiket customer support baru untuk sebuah customer.",
            "parameters": {
                "type": "object",
                "properties": {
                    "customer_id": {
                        "type": "string",
                        "description": "ID customer yang tiketnya mau dibuka.",
                    },
                    "subject": {
                        "type": "string",
                        "description": "Judul singkat tiket, misal 'Order O-456 telat'.",
                    },
                },
                "required": ["customer_id", "subject"],
            },
        },
    },
]
```

Registry: mapping dari nama tool (string yang dikasih model) ke fungsi Python asli yang benar-benar dieksekusi. `run_agent_loop` cuma perlu tau NAMA tool dari model, lalu lookup ke sini buat nemuin fungsi mana yang harus dipanggil:
```python
TOOL_REGISTRY = {
    "get_order_status": get_order_status,
    "create_support_ticket": create_support_ticket,
}
```

Fungsi inti `run_agent_loop`:
```python
import json


def run_agent_loop(
    client, user_message: str, tools: list[dict], max_steps: int = 5
) -> str:
    """
    Agent loop generik: berulang kali panggil call_llm_with_tools (Phase 2),
    eksekusi tool yang diminta model (kalau ada), kirim hasilnya balik ke
    model, sampai model mengembalikan jawaban teks final ATAU max_steps habis.
    """
    messages = [
        {
            "role": "system",
            "content": (
                "Kamu adalah agent customer support SupportPilot. Gunakan tool "
                "yang tersedia buat cari data yang kamu butuh. Kalau informasi "
                "sudah cukup, jawab customer secara langsung tanpa manggil tool lagi."
            ),
        },
        {"role": "user", "content": user_message},
    ]

    for step in range(max_steps):
        result = call_llm_with_tools(client, messages, tools)

        if result["type"] == "message":
            # Model gak minta tool call lagi -> ini jawaban final, loop selesai
            return result["content"]

        # result["type"] == "tool_call": model masih butuh data/aksi tambahan
        tool_name = result["tool_name"]
        tool_arguments = result["tool_arguments"]

        if tool_name not in TOOL_REGISTRY:
            # Model minta tool yang gak ada di registry -> jangan crash,
            # laporkan sebagai error lewat tool result supaya model bisa
            # menjelaskan situasinya ke customer
            tool_result = {"error": f"Tool '{tool_name}' tidak dikenal"}
        else:
            try:
                tool_result = TOOL_REGISTRY[tool_name](**tool_arguments)
            except Exception as e:
                # Tool gagal dieksekusi (argumen invalid, dsb) -> tangkap,
                # jangan biarin seluruh agent loop crash
                tool_result = {"error": str(e)}

        # Catat dulu bahwa assistant tadi minta tool call ini...
        messages.append(
            {
                "role": "assistant",
                "tool_calls": [
                    {
                        "id": result["tool_call_id"],
                        "type": "function",
                        "function": {
                            "name": tool_name,
                            "arguments": json.dumps(tool_arguments),
                        },
                    }
                ],
            }
        )
        # ...lalu masukin hasil eksekusinya, supaya putaran berikutnya
        # model "inget" apa yang barusan terjadi
        messages.append(
            {
                "role": "tool",
                "tool_call_id": result["tool_call_id"],
                "content": json.dumps(tool_result),
            }
        )
        # Lanjut ke iterasi berikutnya: panggil call_llm_with_tools lagi,
        # sekarang dengan hasil tool ini sudah ada di messages

    # max_steps habis tapi model masih minta tool call terus -> jangan biarin
    # loop jalan tanpa henti, keluar paksa dengan fallback
    return (
        "Maaf, saya butuh lebih banyak langkah untuk menyelesaikan permintaan ini. "
        "Tim support kami akan bantu proses lebih lanjut, ya."
    )
```

Contoh menjalankan `run_agent_loop` buat skenario: *"Halo, gimana status order saya O-456? Kalau ternyata telat, tolong bukain tiket juga ya, customer_id saya C-99"*:
```python
from openai import OpenAI

client = OpenAI()

final_answer = run_agent_loop(
    client,
    user_message=(
        "Halo, gimana status order saya O-456? Kalau ternyata telat, "
        "tolong bukain tiket juga ya, customer_id saya C-99"
    ),
    tools=tools,
    max_steps=5,
)
print(final_answer)
```
Yang terjadi di dalam loop itu, langkah demi langkah:
1. **Step 1** — `call_llm_with_tools` balikin `{"type": "tool_call", "tool_name": "get_order_status", "tool_arguments": {"order_id": "O-456"}}`. Kode kita eksekusi `get_order_status("O-456")` lewat `TOOL_REGISTRY`, dapat `{"status": "delayed", ...}`, hasilnya dimasukin ke `messages`.
2. **Step 2** — Model sekarang udah "liat" order-nya `delayed`, jadi dia balikin `{"type": "tool_call", "tool_name": "create_support_ticket", "tool_arguments": {"customer_id": "C-99", "subject": "Order O-456 telat"}}`. Kode kita eksekusi, dapat `{"ticket_id": "T-789", ...}`, dimasukin lagi ke `messages`.
3. **Step 3** — Model sekarang punya cukup info (status order + tiket yang udah dibuka), jadi dia balikin `{"type": "message", "content": "Order O-456 kamu memang lagi delayed, estimasi sampai 20 Agustus. Saya udah bukain tiket T-789 buat follow-up ya."}`. `run_agent_loop` mendeteksi `type == "message"`, langsung `return` — loop berhenti di step ke-3, gak sampai ke `max_steps=5`.

Kalau order-nya ternyata **gak** telat, model kemungkinan langsung balikin `"message"` di step ke-2 (gak perlu manggil `create_support_ticket` sama sekali) — jumlah step yang dipakai memang gak pasti di awal, itulah yang bikin task ini butuh agent loop, bukan tool calling manual dengan urutan yang di-hardcode.

### Trade-off & Pitfall
- **`max_steps` yang kekecilan bisa motong task di tengah jalan** — customer dapat fallback message walau sebenarnya task-nya bisa selesai kalau dikasih 1-2 langkah lagi. `max_steps` yang kegedean sebaliknya bisa bikin biaya dan latency membengkak kalau model "muter-muter" gak jelas arahnya.
- **`TOOL_REGISTRY` harus selalu sinkron dengan `tools` (schema yang dikasih ke model)** — kalau model "ditawarin" tool tertentu lewat schema tapi tool itu gak ada di registry (atau namanya beda dikit), eksekusinya bakal gagal; makanya cabang `if tool_name not in TOOL_REGISTRY` di atas wajib ada, jangan asumsikan tool yang diminta model pasti ketemu.
- **Urutan append ke `messages` harus benar** — pesan `role: "assistant"` (tool call request) harus ditambahkan SEBELUM pesan `role: "tool"` (hasilnya); kebalik urutannya bisa bikin API menolak request atau model jadi bingung soal request mana yang dijawab hasil yang mana.
- **Tool error yang gak ditangkap bisa bikin seluruh loop crash** — `try/except` di sekitar eksekusi tool wajib ada (seperti di atas), supaya kegagalan satu tool (misal `order_id` gak valid) gak menghentikan seluruh percakapan, tapi cukup dilaporkan balik ke model sebagai tool result biasa.

### Kapan Dipakai
- Pakai `run_agent_loop` kapan pun task customer butuh **0 sampai beberapa tool call yang jumlah/urutannya gak pasti di awal** — baru ketauan pas berjalan, tergantung hasil tool sebelumnya (seperti skenario order + tiket di atas).
- Kalau prosesnya udah pasti urutannya (misal selalu tepat satu tool call tanpa follow-up apapun), tool calling manual tanpa loop (Phase 2 topik 8) sudah cukup dan lebih sederhana.
- `max_steps` sebaiknya disesuaikan dengan kompleksitas task riil-nya — task simpel (1-2 tool) cukup `max_steps` kecil (3-5), task yang butuh eksplorasi lebih panjang mungkin butuh lebih banyak, tapi tetap harus ada batasnya.

### Sering Ditanya Saat Interview
- **Bagaimana `run_agent_loop` tau kapan harus berhenti?** — begitu `call_llm_with_tools` mengembalikan `{"type": "message", ...}`, itu tandanya model udah gak minta tool call lagi dan siap kasih jawaban final; loop langsung `return` isi jawabannya.
- **Kenapa `max_steps` penting?** — sebagai pengaman keras (hard limit) supaya loop gak berjalan tanpa henti kalau model gak pernah "yakin" buat berhenti — tanpa ini, biaya API dan latency bisa membengkak gak terkendali.
- **Apa yang terjadi kalau tool yang diminta model gagal dieksekusi?** — error-nya ditangkap (`try/except`) dan dikirim balik ke model sebagai tool result berisi pesan error, bukan bikin seluruh program crash — model bisa lanjut menjelaskan situasinya ke customer.
- **Apa peran `TOOL_REGISTRY` dalam `run_agent_loop`?** — mapping dari nama tool (string yang dikasih model lewat tool call) ke fungsi Python asli yang harus dieksekusi, supaya kode gak perlu nulis `if/elif` manual satu-satu buat tiap kemungkinan tool.

---

## 26. Agent vs Workflow

### Apa itu?
Workflow dan agent sama-sama bisa melibatkan banyak langkah dan tool, tapi bedanya ada di **siapa yang menentukan urutan langkahnya**. Di workflow, developer yang menentukan urutan itu secara eksplisit di kode (langkah 1 selalu diikuti langkah 2, dst — fixed, gak berubah tiap request). Di agent, model (LLM) sendiri yang memutuskan urutan langkah dan tool apa yang dipakai secara dinamis, berdasarkan konteks yang ada saat itu — urutannya bisa beda-beda tiap request, tergantung apa yang sebenarnya dibutuhkan.

### Kenapa dibutuhkan?
Godaan buat selalu pakai "agent" itu besar karena istilahnya lagi tren — padahal banyak proses SupportPilot yang urutan langkahnya sebenarnya **udah pasti** dan gak butuh keputusan dinamis dari LLM sama sekali (misal: "terima tiket baru → klasifikasi urgency → routing ke antrian yang sesuai" — tiga langkah ini selalu sama urutannya, gak peduli isi tiketnya apa). Memaksakan pola agent ke proses yang sebenarnya fixed cuma nambahin kompleksitas, biaya, dan hasil yang kurang predictable, tanpa manfaat nyata. Paham beda keduanya penting supaya keputusan "pakai agent atau enggak" didasarkan pada kebutuhan riil proses itu, bukan sekadar ikut tren.

### Cara Kerja
```
Workflow (developer kontrol urutan langkah, FIXED tiap request):
  Request → Step 1 (kode nentuin) → Step 2 (kode nentuin) → Step 3 (kode nentuin) → Selesai
  (urutan & tool yang dipakai SAMA persis di setiap request, apapun isinya)

Agent (model kontrol urutan langkah, DINAMIS tiap request):
  Request → LLM mikir: "aku butuh apa dulu buat request INI?"
       → pilih tool A, atau tool B, atau langsung jawab kalau udah cukup info
       → LLM mikir lagi: "sekarang aku butuh apa?"
       → pilih tool lain, atau selesai
  (urutan & tool yang dipakai BISA BEDA tiap request, tergantung keputusan LLM saat itu)
```
Workflow lebih **predictable** (gampang di-test, hasilnya konsisten, biaya lebih terkontrol karena jumlah langkahnya fix) tapi kurang **fleksibel** (gak bisa menyesuaikan diri kalau kasusnya di luar pola yang udah ditentukan). Agent lebih **fleksibel** (bisa menyesuaikan diri ke variasi kasus yang tinggi) tapi kurang **predictable** dan lebih mahal (jumlah langkah/biaya bisa beda tiap request, lebih susah di-test secara deterministic).

### Trade-off & Pitfall
- **Memaksakan "agent" ke proses yang sebenarnya fixed/predictable** cuma nambah kompleksitas, biaya, dan latency tanpa manfaat nyata — workflow biasa udah cukup dan jauh lebih murah/predictable buat kasus semacam ini.
- **Agent lebih susah di-test secara deterministic** — hasil (dan jumlah tool call yang dipakai) bisa beda-beda tiap run, sedangkan workflow gampang di-unit-test karena urutan langkahnya fix dan bisa diprediksi persis.
- **Sebaliknya, memaksakan workflow kaku ke proses yang sebenarnya butuh keputusan dinamis** bikin sistemnya gak bisa menangani variasi kasus yang tinggi — developer harus nambah percabangan if/else manual buat tiap kemungkinan kasus baru, yang lama-lama gak scalable.
- **Observability jadi lebih ribet di agent** — karena urutan langkahnya gak fix, debugging butuh telusuri "kenapa model milih tool ini di step ini" buat request SPESIFIK itu, bukan cuma baca satu diagram alur yang berlaku buat semua request (seperti di workflow).

### Kapan Dipakai
- Pakai **workflow** kalau urutan langkahnya udah pasti/predictable dan gak butuh keputusan dinamis dari LLM — misal: "tiket masuk → klasifikasi urgency (topik 6/7) → routing ke antrian" selalu tiga langkah yang sama, urutannya gak pernah berubah.
- Pakai **agent** kalau proses genuinely butuh keputusan dinamis — misal: "customer nanya sesuatu yang bisa butuh 0 sampai beberapa tool berbeda, tergantung apa yang sebenarnya mereka tanyakan" (seperti skenario `run_agent_loop` di topik 25).
- **Jangan pakai agent hanya karena lagi trend** — kalau workflow sederhana udah cukup buat kebutuhan riilnya, itu pilihan yang lebih murah, lebih cepat, dan lebih gampang dijaga kualitasnya.

### Sering Ditanya Saat Interview
- **Apa beda utama antara agent dan workflow?** — di workflow, developer yang menentukan urutan langkah secara eksplisit di kode (fixed); di agent, model (LLM) yang memutuskan urutan langkah dan tool secara dinamis berdasarkan konteks saat itu.
- **Kapan sebaiknya pakai workflow, bukan agent?** — kalau urutan langkahnya udah predictable/tetap dan gak butuh keputusan dinamis dari LLM — workflow lebih murah, lebih gampang di-test, dan hasilnya lebih konsisten.
- **Apa risiko memaksakan pola agent ke proses yang sebenarnya fixed?** — nambah kompleksitas, biaya, dan latency tanpa manfaat nyata, plus hasil yang jadi kurang predictable dibanding kalau dikerjain sebagai workflow biasa.
- **Apakah agent selalu "lebih canggih" dibanding workflow?** — enggak — ini bukan soal mana yang lebih canggih, tapi soal kecocokan sama kebutuhan proses; pilih yang paling sesuai dengan seberapa dinamis keputusan yang dibutuhkan proses itu.

---

## 27. LangGraph — Membangun Agent sebagai Graph

### Apa itu?
LangGraph (dari tim yang sama dengan LangChain, topik 23) adalah framework buat membangun agent loop (topik 25) sebagai **graph/state machine** eksplisit — bukan cuma rantai linear kayak LangChain `|` biasa. Komponen intinya: **State** (dictionary/typed struktur yang membawa data selama graph berjalan — di SupportPilot, misal pertanyaan customer dan jawaban yang lagi disusun), **Node** (fungsi yang menerima state, melakukan sesuatu, dan mengembalikan state yang sudah diperbarui), dan **Conditional Edge** (fungsi yang menentukan, berdasarkan state saat ini, node mana yang harus dijalankan berikutnya — termasuk kemungkinan balik lagi ke node yang sama, alias loop).

### Kenapa dibutuhkan?
Agent loop (topik 25) yang kompleks butuh **percabangan** (kadang harus tool A, kadang tool B, tergantung state), **loop** (balik ke langkah sebelumnya kalau task belum selesai), dan **state** yang eksplisit dan bisa di-inspect di tiap langkah — hal-hal yang susah direpresentasikan sebagai rantai linear `|` LangChain biasa (topik 23), yang didesain buat alur satu-arah tanpa percabangan atau loop balik. LangGraph menyediakan struktur `StateGraph` yang secara native mendukung percabangan dan loop lewat conditional edges, plus **checkpointing** bawaan (simpan state di tengah jalan — penting buat human-in-the-loop di Phase 11) dan dukungan **multi-agent** (Phase 10, tiap agent jadi node terpisah di graph yang sama).

### Cara Kerja
```
StateGraph(AgentState)
    → add_node("nama_node", fungsi_node)      # daftarkan node dan fungsinya
    → set_entry_point("nama_node")             # node mana yang jalan pertama kali
    → add_conditional_edges("nama_node", fn)   # fn menentukan node berikutnya
                                                # (bisa balik ke node yang sama = loop,
                                                #  atau END = graph selesai)
    → compile() → app
    → app.invoke(initial_state) → jalanin graph dari entry point sampai END
```

### Contoh Kode — Python
Graph sederhana buat SupportPilot: satu node yang mencoba mengekstrak `ticket_id` dari pertanyaan customer dan menyusun jawaban; kalau `ticket_id` belum ketemu, conditional edge memutuskan buat **balik lagi** ke node yang sama (loop, di real case: minta klarifikasi ke customer), kalau udah ketemu, graph berhenti (`END`) dengan jawaban final:
```python
from langgraph.graph import StateGraph, END
from typing import TypedDict


class AgentState(TypedDict):
    question: str    # pertanyaan customer
    ticket_id: str    # ticket_id yang berhasil diekstrak (kosong kalau belum ketemu)
    answer: str       # jawaban akhir (kosong selama belum final)


def extract_or_answer(state: AgentState) -> AgentState:
    """
    Satu-satunya node di graph ini. Di real case, bagian "cari ticket_id" dan
    "susun jawaban" masing-masing bisa manggil LLM/tool (get_ticket_status,
    topik 25) — di sini disederhanakan biar fokus ke struktur graph-nya.
    """
    if not state["ticket_id"]:
        if "T-" in state["question"]:
            # simulasikan berhasil menemukan ticket_id dari pertanyaan customer
            state["ticket_id"] = "T-" + state["question"].split("T-")[1][:3]
        else:
            # belum ketemu ticket_id -> answer dibiarkan kosong,
            # conditional edge di bawah bakal nyuruh balik ke node ini lagi
            return state

    state["answer"] = f"Tiket {state['ticket_id']} kamu statusnya sedang diproses."
    return state


def should_continue(state: AgentState) -> str:
    # Conditional edge: kalau jawaban belum final, balik lagi ke node yang sama
    # (loop); kalau udah final, keluar dari graph
    return END if state["answer"] else "extract_or_answer"


graph = StateGraph(AgentState)
graph.add_node("extract_or_answer", extract_or_answer)
graph.set_entry_point("extract_or_answer")
graph.add_conditional_edges("extract_or_answer", should_continue)

app = graph.compile()
result = app.invoke(
    {"question": "Gimana status tiket T-123 saya?", "ticket_id": "", "answer": ""}
)
print(result["answer"])
# diharapkan: "Tiket T-123 kamu statusnya sedang diproses."
```
Kenapa ini penting dibanding LangChain biasa (topik 23): agent loop butuh percabangan dan loop yang eksplisit kayak di atas (`should_continue` yang bisa balik ke `"extract_or_answer"` lagi) — sesuatu yang gak natural direpresentasikan sebagai rantai `prompt | model | parser` satu arah.

### Contoh Kode — Node.js
Graph yang sama, versi LangGraph JS:
```javascript
import { StateGraph, END } from "@langchain/langgraph";

const graph = new StateGraph({
  channels: {
    question: null,
    ticketId: null,
    answer: null,
  },
});

function extractOrAnswer(state) {
  let { question, ticketId, answer } = state;

  if (!ticketId) {
    if (question.includes("T-")) {
      // simulasikan berhasil menemukan ticketId dari pertanyaan customer
      ticketId = "T-" + question.split("T-")[1].slice(0, 3);
    } else {
      // belum ketemu ticketId -> answer tetap kosong, conditional edge
      // bakal nyuruh balik ke node ini lagi
      return { ticketId, answer: "" };
    }
  }

  answer = `Tiket ${ticketId} kamu statusnya sedang diproses.`;
  return { ticketId, answer };
}

function shouldContinue(state) {
  return state.answer ? END : "extractOrAnswer";
}

graph.addNode("extractOrAnswer", extractOrAnswer);
graph.setEntryPoint("extractOrAnswer");
graph.addConditionalEdges("extractOrAnswer", shouldContinue);

const app = graph.compile();
const result = await app.invoke({
  question: "Gimana status tiket T-123 saya?",
  ticketId: "",
  answer: "",
});
console.log(result.answer);
// diharapkan: "Tiket T-123 kamu statusnya sedang diproses."
```

### Cara Manual (From Scratch) — agent loop tanpa LangGraph
LangGraph itu intinya cuma **while-loop dengan state dictionary** yang dibungkus rapi. Ini versi manualnya, buat kasus SupportPilot yang sama persis:

**Python (manual, agent loop pakai while-loop biasa):**
```python
def extract_or_answer_manual(question: str, ticket_id: str) -> tuple[str, str]:
    # Setara node "extract_or_answer" di versi LangGraph
    if not ticket_id:
        if "T-" in question:
            ticket_id = "T-" + question.split("T-")[1][:3]
        else:
            return ticket_id, ""  # belum final, answer masih kosong

    answer = f"Tiket {ticket_id} kamu statusnya sedang diproses."
    return ticket_id, answer


def is_task_done(answer: str) -> bool:
    # Ini yang digantiin conditional edge should_continue di LangGraph
    return bool(answer)


def run_agent_manual(question: str) -> str:
    # "state" cuma dictionary biasa, gak ada abstraksi graph
    state = {"question": question, "ticket_id": "", "answer": ""}

    # Ini "graph"-nya: while-loop yang bisa balik lagi kalau task belum selesai
    while not is_task_done(state["answer"]):
        state["ticket_id"], state["answer"] = extract_or_answer_manual(
            state["question"], state["ticket_id"]
        )
        # Kalau butuh multi-node lain (misal panggil tool dulu baru jawab),
        # tinggal tambah percabangan if/elif di sini

    return state["answer"]


result = run_agent_manual("Gimana status tiket T-123 saya?")
print(result)
# diharapkan: "Tiket T-123 kamu statusnya sedang diproses."
```

**Node.js (manual, agent loop pakai while-loop biasa):**
```javascript
function extractOrAnswerManual(question, ticketId) {
  // Setara node "extractOrAnswer" di versi LangGraph
  if (!ticketId) {
    if (question.includes("T-")) {
      ticketId = "T-" + question.split("T-")[1].slice(0, 3);
    } else {
      return { ticketId, answer: "" }; // belum final, answer masih kosong
    }
  }

  const answer = `Tiket ${ticketId} kamu statusnya sedang diproses.`;
  return { ticketId, answer };
}

function isTaskDone(answer) {
  // Setara conditional edge shouldContinue di LangGraph
  return Boolean(answer);
}

function runAgentManual(question) {
  // "state" cuma object biasa, gak ada abstraksi graph
  let state = { question, ticketId: "", answer: "" };

  // Ini "graph"-nya: while-loop yang bisa balik lagi kalau task belum selesai
  while (!isTaskDone(state.answer)) {
    const { ticketId, answer } = extractOrAnswerManual(state.question, state.ticketId);
    state = { ...state, ticketId, answer };
    // Kalau butuh multi-node lain (misal panggil tool dulu baru jawab),
    // tinggal tambah percabangan if/else di sini
  }

  return state.answer;
}

console.log(runAgentManual("Gimana status tiket T-123 saya?"));
// diharapkan: "Tiket T-123 kamu statusnya sedang diproses."
```

**Insight penting:** kode manual di atas keliatan simpel buat satu node. Tapi begitu agent SupportPilot punya banyak node (cek tiket → cari tau di knowledge base → putuskan eskalasi → susun jawaban), banyak percabangan, dan butuh **checkpointing** (nyimpen state di tengah jalan buat human-in-the-loop atau resume setelah crash) — nulis manual jadi gampang berantakan dan gampang salah urutan. Itu yang LangGraph selesaikan: state machine yang terstruktur, bisa divisualisasikan, dan checkpoint-nya udah built-in, bukan lu bikin sendiri dari nol.

### Trade-off & Pitfall
- **Overhead belajar tambahan** — konsep `StateGraph`, node, conditional edges, dan `compile()` perlu dipahami dulu sebelum bisa dipakai efektif; buat agent loop yang cuma satu-dua langkah tanpa percabangan (seperti contoh dasar di topik 25), while-loop manual sudah cukup dan lebih transparan.
- **Debugging graph yang kompleks tetap butuh usaha** — walau state-nya eksplisit dan bisa di-inspect, begitu jumlah node dan conditional edge-nya banyak, menelusuri "kenapa graph ambil jalur ini" tetap butuh effort, gak otomatis jadi simpel cuma karena strukturnya lebih rapi.
- **Loop yang gak dibatasi tetap bisa jalan tanpa henti** — LangGraph gak otomatis mencegah loop infinite kalau conditional edge-nya gak pernah mengarah ke `END`; sama seperti `max_steps` di `run_agent_loop` (topik 25), batas jumlah iterasi tetap perlu dipikirkan secara eksplisit di logic node/conditional edge-nya.
- **Versi library yang sering update** — sama seperti LangChain (topik 23), API LangGraph juga berpotensi berubah antar versi rilis, jadi kode yang jalan mulus di satu versi belum tentu jalan sama persis di versi berikutnya.

### Kapan Dipakai
- Pakai LangGraph begitu agent SupportPilot butuh lebih dari satu langkah "mikir → tindakan → mikir lagi" dengan kemungkinan **bercabang** atau **balik ke langkah sebelumnya** — itu tandanya udah lewat batas yang nyaman buat while-loop manual atau LangChain chain linear biasa.
- Pakai LangGraph kalau butuh **checkpointing** (human-in-the-loop, Phase 11) atau bakal berkembang jadi **multi-agent** (Phase 10) — dua kebutuhan ini native didukung LangGraph tanpa harus dibangun sendiri dari nol.
- Kalau agent loop-nya masih sederhana (satu-dua node, gak ada percabangan berarti, seperti `run_agent_loop` dasar di topik 25) — while-loop manual sudah cukup dan lebih gampang dipahami tim yang belum familiar LangGraph.

### Sering Ditanya Saat Interview
- **Apa itu LangGraph, dan apa bedanya dengan LangChain biasa?** — framework buat membangun agent loop sebagai graph/state machine eksplisit (StateGraph, node, conditional edges); beda dari LangChain yang didesain buat rantai linear (`|`) satu arah tanpa percabangan/loop bawaan.
- **Apa yang sebenarnya terjadi di balik `StateGraph` sederhana?** — sama persis dengan pola manual: while-loop yang mengecek state lalu memutuskan mau lanjut ke node mana (termasuk balik ke node yang sama) — LangGraph menstandardisasi cara mendefinisikan dan menjalankan pola itu.
- **Kenapa agent loop butuh conditional edges, bukan cuma rantai `|` biasa?** — karena agent loop butuh percabangan (tool mana yang dipakai tergantung state) dan kemampuan balik ke langkah sebelumnya (loop) — dua hal yang gak natural direpresentasikan sebagai rantai satu arah.
- **Kapan LangGraph mulai worth-it dibanding while-loop manual?** — begitu agent punya banyak node dengan percabangan yang kompleks, dan/atau butuh checkpointing (human-in-the-loop) atau berkembang jadi multi-agent; buat satu-dua node sederhana, manual masih lebih transparan dan gampang di-debug.

---

## 28. Tools

### Apa itu?
Di konteks agent, tool adalah generalisasi dari tool calling single-step (Phase 2 topik 8): sebuah fungsi konkret plus schema-nya, yang dikumpulkan jadi satu **katalog** kemampuan yang bisa dipanggil model selama agent loop (topik 25) berjalan. Bedanya dengan tool calling dasar bukan di konsepnya (masih sama: tool definition, tool arguments, tool execution, tool result — lihat Phase 2 topik 8), tapi di skala: begitu SupportPilot punya lebih dari satu-dua tool, semuanya perlu didaftarkan bareng-bareng di satu tempat yang jelas, supaya agent loop yang sama bisa memilih tool mana pun yang relevan buat tiap request.

### Kenapa dibutuhkan?
Begitu SupportPilot punya banyak tool (`get_ticket_status`, `get_order_status`, `create_support_ticket`, `escalate_to_human`, `search_knowledge_base`) yang tersebar di berbagai tempat di codebase, jadi sulit tau **tool apa aja** yang sebenarnya bisa diakses model, siapa yang boleh manggil tool mana, dan gimana cara menambah/menghapus satu tool tanpa mengacaukan yang lain. Menyatukan definisi tool (schema `tools` + `TOOL_REGISTRY`, topik 25) di satu tempat menyelesaikan ini — sekaligus jadi titik yang tepat buat menegakkan **kontrol akses**: LLM sama sekali **gak boleh** dikasih akses tanpa batas ke semua fungsi yang ada di codebase (misal fungsi yang menghapus data, atau mengirim uang) — perlu ada **permission** (tool mana yang boleh dipanggil di konteks apa), **validation** (argumen dari model dicek dulu sebelum dieksekusi), **allowlisting** (cuma tool yang eksplisit didaftarkan yang bisa dipanggil, bukan sembarang fungsi Python), dan **sandboxing** (tool yang eksekusinya berisiko dijalankan di lingkungan terbatas). Ini baru sekilas — pembahasan detailnya ada di Phase 11 dan Phase 14.

### Cara Kerja
```
Definisikan tiap fungsi tool (get_ticket_status, get_order_status, dst)
    → definisikan schema-nya masing-masing (nama, description, parameters)
    → gabung semua schema jadi satu list `tools`      (dikasih ke model)
    → gabung semua fungsi jadi satu dict TOOL_REGISTRY (dieksekusi kode)
    → run_agent_loop(client, user_message, tools) dipanggil dengan
      katalog lengkap ini, model bebas pilih tool mana pun yang relevan
```

### Contoh Kode — Python
Tool baru yang belum pernah muncul: `escalate_to_human` (mock, gaya sama seperti tool lainnya), dan `search_knowledge_base` — wrapper tipis di atas `retrieve_relevant_chunks` (Phase 4), supaya fungsi RAG yang udah ada bisa dipanggil model sebagai tool biasa:
```python
def escalate_to_human(ticket_id: str) -> dict:
    """
    Mock function: pura-pura eskalasi sebuah tiket ke antrian agent manusia.
    """
    return {
        "ticket_id": ticket_id,
        "escalated": True,
        "assigned_to": "human-agent-queue",
        "escalated_at": "2026-08-14T10:05:00Z",
    }


def search_knowledge_base(query: str) -> list[dict]:
    """
    Wrapper tool di atas retrieve_relevant_chunks (Phase 4). `db_conn` di sini
    adalah koneksi database yang sudah dibuat sekali di startup aplikasi (bukan
    argumen yang diisi model) — polanya sama seperti gateway/registry lain:
    tool yang "ditawarkan" ke model gak perlu tau detail koneksi database.
    """
    chunks = retrieve_relevant_chunks(db_conn, query, top_k=3)
    return [{"source": c["source"], "content": c["content"]} for c in chunks]
```

Sekarang gabungkan **semua** tool SupportPilot sejauh ini — `get_ticket_status` (Phase 2), `get_order_status` dan `create_support_ticket` (topik 25), plus `escalate_to_human` dan `search_knowledge_base` yang baru dikenalin di atas — jadi satu `tools` list (schema, dikasih ke model) dan satu `TOOL_REGISTRY` (fungsi asli, dieksekusi kode):
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
                    "ticket_id": {"type": "string", "description": "ID tiket, contohnya 'T-123'."}
                },
                "required": ["ticket_id"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "get_order_status",
            "description": "Ambil status pengiriman sebuah order berdasarkan order_id.",
            "parameters": {
                "type": "object",
                "properties": {
                    "order_id": {"type": "string", "description": "ID order, contohnya 'O-456'."}
                },
                "required": ["order_id"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "create_support_ticket",
            "description": "Buka tiket customer support baru untuk sebuah customer.",
            "parameters": {
                "type": "object",
                "properties": {
                    "customer_id": {"type": "string", "description": "ID customer."},
                    "subject": {"type": "string", "description": "Judul singkat tiket."},
                },
                "required": ["customer_id", "subject"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "escalate_to_human",
            "description": "Eskalasi sebuah tiket yang sudah ada ke antrian agent manusia.",
            "parameters": {
                "type": "object",
                "properties": {
                    "ticket_id": {"type": "string", "description": "ID tiket yang mau dieskalasi."}
                },
                "required": ["ticket_id"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "search_knowledge_base",
            "description": "Cari artikel/dokumentasi SupportPilot yang relevan dengan sebuah pertanyaan.",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "Pertanyaan atau topik yang mau dicari."}
                },
                "required": ["query"],
            },
        },
    },
]

TOOL_REGISTRY = {
    "get_ticket_status": get_ticket_status,
    "get_order_status": get_order_status,
    "create_support_ticket": create_support_ticket,
    "escalate_to_human": escalate_to_human,
    "search_knowledge_base": search_knowledge_base,
}
```

Menjalankan `run_agent_loop` (topik 25) dengan katalog tool yang lengkap, buat request customer yang butuh beberapa tool berbeda sekaligus: *"Tiket T-555 saya belum selesai juga udah lama, kenapa ya? Kalau memang udah kelamaan, tolong eskalasi ke manusia aja"*:
```python
from openai import OpenAI

client = OpenAI()

final_answer = run_agent_loop(
    client,
    user_message=(
        "Tiket T-555 saya belum selesai juga udah lama, kenapa ya? "
        "Kalau memang udah kelamaan, tolong eskalasi ke manusia aja"
    ),
    tools=tools,
    max_steps=5,
)
print(final_answer)
```
Kemungkinan alurnya: model manggil `get_ticket_status("T-555")` dulu buat lihat statusnya; kalau hasilnya menunjukkan tiket itu emang udah lama nunggu, model mungkin manggil `search_knowledge_base("kebijakan eskalasi tiket lama")` buat cek kriteria kapan sebuah tiket layak dieskalasi; kalau kriterianya terpenuhi, model manggil `escalate_to_human("T-555")`; baru setelah itu model menyusun jawaban final ke customer. Jumlah dan urutan tool yang benar-benar dipakai gak fix — itu sepenuhnya keputusan model berdasarkan hasil tiap tool sebelumnya, persis pola agent loop yang dibahas di topik 25.

### Trade-off & Pitfall
- **LLM gak boleh punya akses tanpa batas ke semua tool** — tiap tool yang didaftarkan ke `TOOL_REGISTRY` pada dasarnya adalah "izin" buat model buat memicu efek nyata (bikin tiket, eskalasi, dst); makin banyak tool yang didaftarkan, makin besar juga permukaan risiko kalau model salah pilih atau salah ngisi argumen (lihat Phase 2 topik 8) — allowlisting eksplisit (cuma daftarkan tool yang benar-benar dimaksudkan buat dipakai model) adalah pertahanan pertama.
- **Validasi argumen sebelum eksekusi tetap wajib**, terutama buat tool yang mengubah data (`create_support_ticket`, `escalate_to_human`) — argumen dari model yang gak divalidasi bisa memicu aksi yang gak diinginkan (misal `subject` kosong, atau `ticket_id` format aneh).
- **Tool description yang tumpang tindih bikin model salah pilih** — kalau `get_ticket_status` dan `get_order_status` deskripsinya terlalu mirip, model bisa salah manggil satu buat kebutuhan yang sebenarnya perlu yang lain; deskripsi tiap tool perlu spesifik dan gampang dibedakan satu sama lain.
- **Katalog tool yang terus bertambah butuh perawatan** — `tools` (schema) dan `TOOL_REGISTRY` (eksekusi) adalah dua sumber kebenaran yang harus tetap sinkron; nambah satu tool baru berarti harus update KEDUANYA, lupa salah satu bikin tool itu "ditawarkan" ke model tapi gagal dieksekusi (atau sebaliknya).

### Kapan Dipakai
- Kumpulkan semua tool SupportPilot di satu tempat (satu `tools` list + satu `TOOL_REGISTRY`) begitu jumlahnya lebih dari satu-dua — supaya gampang direview tool apa aja yang boleh diakses model, dan gampang di-maintain begitu ada tool baru.
- Terapkan **allowlisting + validation** sejak tool pertama yang bisa mengubah data (bukan cuma baca data) didaftarkan — jangan tunggu sampai ada insiden dulu baru dipikirkan; detail lengkap permission/sandboxing dibahas di Phase 11 dan Phase 14.
- Kalau agent SupportPilot butuh akses ke data yang udah ada RAG pipeline-nya (Phase 4), bungkus sebagai tool (seperti `search_knowledge_base` di atas) daripada menduplikasi logic retrieval-nya di tempat lain.

### Sering Ditanya Saat Interview
- **Apa itu tool dalam konteks agent, dan gimana bedanya dengan tool calling dasar (Phase 2)?** — konsepnya sama (tool definition, argument, execution, result), bedanya di skala: tool di konteks agent dikumpulkan jadi satu katalog (`tools` + registry eksekusi) yang dipilih model secara dinamis selama agent loop berjalan, bukan cuma satu tool tunggal buat satu skenario.
- **Kenapa LLM gak boleh punya akses tanpa batas ke semua fungsi yang ada di codebase?** — tiap tool yang diberikan ke model pada dasarnya adalah izin buat memicu efek nyata; tanpa permission/validation/allowlisting/sandboxing, model yang salah pilih atau salah ngisi argumen bisa memicu aksi yang gak diinginkan.
- **Apa yang harus tetap sinkron begitu nambah tool baru?** — schema tool di `tools` (yang dikasih ke model) dan entry-nya di `TOOL_REGISTRY` (fungsi asli yang dieksekusi) — keduanya dua sumber kebenaran terpisah yang harus di-update bareng.
- **Kenapa deskripsi tiap tool penting kalau jumlah tool-nya banyak?** — deskripsi yang tumpang tindih/gak jelas bikin model gampang salah pilih tool yang relevan buat kebutuhan tertentu, terutama begitu ada banyak tool dengan fungsi yang mirip-mirip.

---

**Selanjutnya:** [Phase 07 — Agent Memory](./phase-07-agent-memory.md)
