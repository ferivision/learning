# Phase 10 — Agent Orchestration

> Bagian dari [AI Engineering / Agent Roadmap](../README.md)

---

## 38. Single Agent

### Apa itu?
Single agent adalah pola yang udah kita bangun lengkap di Phase 6: **satu** agent loop (`run_agent_loop`, topik 25), dibekali **satu** katalog tool (`tools` + `TOOL_REGISTRY`, topik 28), yang menangani seluruh percakapan customer dari awal sampai jawaban final. Alurnya persis: `User → Agent → Search/Database/Email` — satu LLM yang mikir, manggil tool mana pun yang relevan (`get_ticket_status`, `get_order_status`, `create_support_ticket`, `escalate_to_human`, `search_knowledge_base`), dan menyusun jawaban, tanpa ada agent lain yang terlibat.

### Kenapa dibutuhkan?
Sebelum ngomongin "multi-agent", penting ditegasin dulu: single agent itu bukan versi "belum lengkap" dari multi-agent — dia adalah **baseline** yang sudah cukup buat sebagian besar kebutuhan SupportPilot. Satu agent dengan katalog tool yang cukup lengkap (seperti yang dibangun di Phase 6) bisa menjawab pertanyaan soal status tiket, status order, buka tiket baru, eskalasi, sampai cari jawaban di knowledge base — semua dalam satu loop, satu system prompt, satu context yang koheren. Paham ini penting karena godaan buat langsung lompat ke arsitektur multi-agent (topik 39-40) itu besar padahal sering kali gak perlu.

### Cara Kerja
```
Single Agent (baseline dari Phase 6):

  User → [Agent: run_agent_loop] → Search (search_knowledge_base)
                                  → Database (get_ticket_status, get_order_status)
                                  → Email/Aksi (create_support_ticket, escalate_to_human)
         (satu LLM, satu system prompt, satu katalog tool, satu loop)
```
Satu agent, satu konteks percakapan yang koheren — tanpa perlu "serah-terima" pekerjaan ke agent lain di tengah jalan.

### Trade-off & Pitfall
- **Katalog tool yang menggelembung di satu agent bikin system prompt makin panjang dan model makin gampang bingung** milih tool yang tepat (lihat topik 28: deskripsi tool yang tumpang tindih bikin salah pilih) — ini salah satu sinyal awal kapan mulai mikirin spesialisasi (topik 39).
- **Satu agent yang menangani domain yang sangat berbeda** (misal customer support umum DAN billing yang detail soal invoice/refund) bisa berujung ke system prompt yang "generalis" dan kurang tajam di tiap domain dibanding kalau dipisah.
- **Godaan sebaliknya juga bahaya**: pecah jadi banyak agent padahal single agent + tool yang cukup udah menjawab kebutuhan — nambah kompleksitas tanpa manfaat nyata (dibahas detail di topik 41).

### Kapan Dipakai
- **Mulai dari sini** buat hampir semua kebutuhan baru — single agent + katalog tool yang relevan (Phase 6) adalah default yang paling murah, paling gampang di-debug, dan paling gampang dites.
- Baru pertimbangkan multi-agent (topik 39) kalau ketemu domain yang genuinely butuh spesialisasi berbeda (system prompt/tool set yang beda jauh) DAN kompleksitas koordinasinya sepadan sama manfaatnya.

### Sering Ditanya Saat Interview
- **Apa itu "single agent" dalam konteks arsitektur agent?** — satu agent loop (`run_agent_loop`) yang dibekali satu katalog tool, menangani seluruh percakapan dari awal sampai jawaban final tanpa melibatkan agent lain.
- **Kenapa harus mulai dari single agent, bukan langsung multi-agent?** — single agent lebih murah, lebih gampang di-debug (satu context, satu alur), dan buat kebanyakan kasus udah cukup — multi-agent nambah kompleksitas dan biaya yang cuma sepadan kalau ada kebutuhan spesialisasi yang jelas.
- **Apa sinyal paling umum bahwa sebuah agent sebaiknya dipecah jadi beberapa agent?** — ketika katalog tool-nya menggelembung mencakup domain-domain yang sangat berbeda (misal general support vs billing) sampai system prompt-nya jadi generalis dan model mulai gampang salah pilih tool.

---

## 39. Multi-Agent

### Apa itu?
Multi-agent adalah pola di mana beberapa agent yang **terspesialisasi** — masing-masing dengan system prompt dan katalog tool yang lebih sempit dan fokus ke satu domain — dikoordinasikan oleh sebuah **manager**. Manager gak menjawab pertanyaan customer sendiri; tugasnya cuma memutuskan agent spesialis mana yang paling cocok menangani permintaan itu, lalu meneruskan (dan kalau perlu, menggabungkan hasilnya).

### Kenapa dibutuhkan?
Begitu SupportPilot mulai menangani domain yang genuinely berbeda kebutuhannya — misalnya pertanyaan customer support umum (status tiket, status order) versus urusan billing (invoice, refund, yang butuh ketelitian ekstra karena menyangkut uang) — mencampur semuanya jadi satu agent (topik 38) bikin system prompt-nya generalis dan katalog tool-nya menggelembung. Dua agent yang lebih sempit dan fokus — satu buat support umum, satu lagi khusus billing dengan tool yang lebih dibatasi (sesuai prinsip allowlisting di topik 28: model gak perlu ditawarin tool `issue_refund` kalau pertanyaannya cuma soal status tiket) — lebih gampang dijaga akurat dan lebih aman, asal ada satu "manager" yang tau kapan harus mengarahkan ke agent yang mana.

### Cara Kerja
Pola umum multi-agent (contoh generik: manager membagi kerja ke agent yang beda spesialisasi):
```
                        ┌──────────────────┐
        Request ──────► │   Manager Agent   │
                        └─────────┬─────────┘
                                  │ (memutuskan agent mana yang relevan)
              ┌───────────────────┼───────────────────┐
              ▼                   ▼                   ▼
      ┌───────────────┐   ┌───────────────┐   ┌───────────────┐
      │ Research Agent │   │  Coding Agent  │   │ Analysis Agent │
      └───────────────┘   └───────────────┘   └───────────────┘
```
Versi konkret di SupportPilot — manager cukup **router**, bukan agent LLM tersendiri, karena keputusannya sederhana (dua pilihan, bisa dijawab keyword matching):
```
                        ┌────────────────────────┐
  user_message ───────► │  run_multi_agent_flow   │
                        │  (classify_route: cek   │
                        │   keyword billing)      │
                        └───────────┬────────────┘
                     "support" ◄────┴────► "billing"
                          ▼                    ▼
                ┌──────────────────┐  ┌──────────────────┐
                │  Support Agent    │  │  Billing Agent    │
                │ run_agent_loop(   │  │ run_agent_loop(   │
                │   tools=          │  │   tools=          │
                │   SUPPORT_TOOLS)  │  │   BILLING_TOOLS)  │
                └──────────────────┘  └──────────────────┘
```
Baik support agent maupun billing agent sama-sama cuma pemanggil `run_agent_loop` (topik 25) yang sudah ada — bedanya cuma **katalog tool** (`tools`, schema yang ditawarkan ke model) yang dipakai tiap agent, jauh lebih sempit dan spesifik ke domainnya masing-masing.

### Contoh Kode — Python
Dua tool baru khusus billing, mock function dengan gaya yang sama seperti tool-tool di Phase 6 — sengaja **tidak** dicampur ke katalog tool support umum, karena efeknya (refund menyangkut uang) butuh scope lebih sempit:
```python
def get_invoice_details(customer_id: str) -> dict:
    """
    Mock function: pura-pura ambil detail invoice terakhir seorang customer.
    """
    return {
        "customer_id": customer_id,
        "invoice_id": "INV-2201",
        "amount": 150000,
        "status": "paid",
        "billed_at": "2026-08-01",
    }


def issue_refund(invoice_id: str, amount: int) -> dict:
    """
    Mock function: pura-pura memproses refund untuk sebuah invoice.
    """
    return {
        "invoice_id": invoice_id,
        "refunded_amount": amount,
        "status": "refund_processed",
        "processed_at": "2026-08-14T11:00:00Z",
    }
```

Katalog tool tiap agent. `TOOL_REGISTRY` (eksekusi, topik 25/28) tetap **satu** dictionary gabungan berisi semua fungsi yang ada — yang bikin tiap agent jadi "sempit" bukan registry-nya, tapi `tools` (schema) yang ditawarkan ke model, karena model cuma bisa minta tool yang namanya ada di `tools` yang dikasih ke `call_llm_with_tools` (Phase 2):
```python
SUPPORT_TOOLS = [
    tool for tool in tools  # `tools` gabungan lengkap dari Phase 6 topik 28
    if tool["function"]["name"]
    in {"get_ticket_status", "get_order_status", "create_support_ticket", "escalate_to_human", "search_knowledge_base"}
]

BILLING_TOOLS = [
    {
        "type": "function",
        "function": {
            "name": "get_invoice_details",
            "description": "Ambil detail invoice (jumlah tagihan, status pembayaran) berdasarkan customer_id.",
            "parameters": {
                "type": "object",
                "properties": {
                    "customer_id": {"type": "string", "description": "ID customer, contohnya 'C-99'."}
                },
                "required": ["customer_id"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "issue_refund",
            "description": "Proses refund untuk sebuah invoice, sejumlah nominal tertentu.",
            "parameters": {
                "type": "object",
                "properties": {
                    "invoice_id": {"type": "string", "description": "ID invoice, contohnya 'INV-2201'."},
                    "amount": {"type": "integer", "description": "Nominal refund dalam rupiah."},
                },
                "required": ["invoice_id", "amount"],
            },
        },
    },
]

# TOOL_REGISTRY (topik 25/28) diperluas dengan dua fungsi billing di atas -- tetap
# SATU registry gabungan; yang membatasi tiap agent adalah `tools` (schema) di atas,
# bukan registry-nya
TOOL_REGISTRY["get_invoice_details"] = get_invoice_details
TOOL_REGISTRY["issue_refund"] = issue_refund
```

Router: klasifikasi sederhana pakai keyword matching (bukan LLM call tambahan) — cukup buat kasus dua-pilihan kayak gini, dan hasilnya deterministic (gampang dites):
```python
BILLING_KEYWORDS = ("tagihan", "invoice", "refund", "bayar", "pembayaran", "billing", "faktur")


def classify_route(user_message: str) -> str:
    """
    Router multi-agent paling sederhana: keyword matching. Kalau pesan customer
    mengandung salah satu kata kunci seputar billing, arahkan ke billing agent;
    selain itu default ke support agent (general).
    """
    lowered = user_message.lower()
    if any(keyword in lowered for keyword in BILLING_KEYWORDS):
        return "billing"
    return "support"
```

`run_multi_agent_flow`: manager function yang memanggil router, lalu meneruskan ke agent yang sesuai. `client` dibuat sekali di startup aplikasi (pola yang sama seperti `db_conn` di topik 28), dipakai bareng oleh semua agent:
```python
from openai import OpenAI

client = OpenAI()  # dibuat sekali saat startup, dipakai semua agent di bawah ini


def run_multi_agent_flow(user_message: str) -> str:
    """
    Manager: route pesan customer ke support agent (general) atau billing agent
    (spesialis), masing-masing cuma wrapper tipis di atas run_agent_loop (topik 25)
    dengan katalog tool yang berbeda.
    """
    route = classify_route(user_message)

    if route == "billing":
        return run_agent_loop(client, user_message, tools=BILLING_TOOLS, max_steps=5)

    # default: support agent, menangani pertanyaan umum
    return run_agent_loop(client, user_message, tools=SUPPORT_TOOLS, max_steps=5)
```

Contoh menjalankan dua skenario berbeda:
```python
# Skenario 1: pertanyaan umum -> classify_route("...") == "support"
jawaban_1 = run_multi_agent_flow("Gimana status tiket T-555 saya?")
print(jawaban_1)
# route ke Support Agent -> run_agent_loop dipanggil dengan tools=SUPPORT_TOOLS,
# model manggil get_ticket_status("T-555")

# Skenario 2: pertanyaan billing -> classify_route("...") == "billing"
jawaban_2 = run_multi_agent_flow("Saya mau tanya soal invoice terakhir, boleh refund gak ya?")
print(jawaban_2)
# route ke Billing Agent -> run_agent_loop dipanggil dengan tools=BILLING_TOOLS,
# model manggil get_invoice_details(...), lalu mungkin issue_refund(...) kalau
# kriteria refund terpenuhi
```
Perhatikan: `run_multi_agent_flow` sendiri **tidak pernah** manggil LLM langsung — dia cuma router (fungsi Python biasa) yang memutuskan agent mana yang dipanggil; LLM baru terlibat di dalam `run_agent_loop` masing-masing agent.

### Trade-off & Pitfall
- **Router berbasis keyword itu rapuh** — pesan customer yang gak eksplisit mengandung kata kunci billing (misal "kenapa saya ditagih dua kali") bisa salah di-route ke support agent yang gak punya tool `get_invoice_details`/`issue_refund`; keyword list butuh terus dipelihara, atau diganti klasifikasi LLM (lebih fleksibel, tapi nambah satu API call sebelum agent yang sebenarnya jalan — biaya dan latency ekstra).
- **`TOOL_REGISTRY` gabungan tapi `tools` per-agent beda-beda** butuh disiplin: nambah tool baru ke satu agent doang berarti update `SUPPORT_TOOLS`/`BILLING_TOOLS` (schema) yang tepat, JANGAN taruh sembarangan di keduanya — kalau billing agent "ditawarin" `create_support_ticket` padahal gak perlu, itu melanggar prinsip allowlisting (topik 28).
- **Manager yang salah route bikin agent yang salah nyoba jawab** tanpa tool yang tepat — hasilnya customer dapat jawaban yang gak akurat atau agent mentok karena gak ada tool yang relevan; router yang gak akurat adalah titik kegagalan baru yang gak ada di single agent (topik 38).
- **Dua agent terpisah berarti dua kali system prompt + dua kali kemungkinan panggilan `run_agent_loop`** — kalau permintaan customer sebenarnya butuh KEDUANYA (misal pertanyaan campuran support+billing), router dua-pilihan ini gak cukup; itu yang dibahas di topik 40 lewat delegasi.

### Kapan Dipakai
- Pakai multi-agent dengan manager router kayak di atas begitu ada **domain yang genuinely berbeda kebutuhan tool-nya** (support umum vs billing) DAN klasifikasi ke domain yang tepat itu sendiri gampang/murah dilakukan (keyword atau LLM call ringan).
- Kalau router-nya sendiri jadi rumit (butuh banyak konteks buat memutuskan agent mana yang tepat), pertimbangkan apakah itu tandanya harusnya satu agent generalis dengan tool lengkap (topik 38) masih lebih simpel.
- Kalau permintaan customer sering butuh **lebih dari satu** agen sekaligus dalam satu percakapan, lanjut ke pola delegasi (topik 40), bukan cuma routing dua-pilihan yang eksklusif.

### Sering Ditanya Saat Interview
- **Apa peran "manager" dalam arsitektur multi-agent?** — memutuskan agent spesialis mana yang relevan buat sebuah permintaan (lewat routing/klasifikasi), bukan menjawab pertanyaan itu sendiri — jawaban tetap datang dari agent spesialis yang dipilih.
- **Kenapa `SUPPORT_TOOLS` dan `BILLING_TOOLS` dipisah padahal `TOOL_REGISTRY`-nya sama?** — karena yang membatasi tool apa yang bisa dipanggil model adalah `tools` (schema, dikasih ke `call_llm_with_tools`), bukan registry eksekusinya; ini menerapkan prinsip allowlisting (topik 28) per-agent.
- **Apa risiko router berbasis keyword?** — rapuh terhadap variasi bahasa customer yang gak eksplisit mengandung kata kunci yang didaftarkan, bisa salah route ke agent yang gak punya tool yang tepat.
- **Kapan sebaiknya pakai LLM call buat klasifikasi routing, dibanding keyword matching?** — kalau variasi cara customer bertanya terlalu tinggi buat ditangkap keyword list yang fixed; trade-off-nya nambah satu API call (biaya + latency) sebelum agent yang sebenarnya mulai bekerja.

---

## 40. Agent Delegation

### Apa itu?
Kalau di topik 39 manager memilih **satu** agent yang menangani seluruh permintaan dari awal sampai akhir, agent delegation adalah pola di mana **satu agent yang sedang berjalan** menyadari bagian dari tugasnya di luar keahliannya, lalu **mendelegasikan** sub-bagian itu ke agent lain di tengah percakapan, menunggu hasilnya, dan menggabungkan hasil itu ke jawaban akhir yang dia susun sendiri.

### Kenapa dibutuhkan?
Routing dua-pilihan di topik 39 berasumsi tiap permintaan customer murni satu domain: full support ATAU full billing. Kenyataannya banyak permintaan **campuran** — misal customer nanya soal tiket yang macet, tapi ternyata alasan sebenarnya tiket itu di-hold adalah karena ada dispute pembayaran yang belum kelar. Support agent yang menerima pertanyaan awal gak punya tool billing (`get_invoice_details`, `issue_refund`) buat mengecek itu sendiri — dia butuh cara buat "bertanya" ke billing agent di tengah proses mikirnya, dapat jawabannya, lalu meneruskan info itu ke customer sebagai bagian dari jawaban akhir yang koheren. Pola umum ini (Manager: "Analisis masalah churn customer ini" → Research Agent → Data Agent → Strategy Agent → Manager menggabungkan) sama persisnya: satu tugas kompleks dipecah, didelegasikan ke spesialis, hasilnya digabung jadi satu output akhir.

### Cara Kerja
Pola umum (ilustrasi generik, bukan spesifik SupportPilot):
```
Manager: "Analisis masalah churn customer ini"
    │
    ├──► Research Agent  (kumpulkan data historis relevan)
    │         │
    │         ▼ hasil riset
    ├──► Data Agent       (olah angka, cari pola dari hasil riset)
    │         │
    │         ▼ hasil analisis data
    └──► Strategy Agent   (susun rekomendasi dari hasil analisis)
              │
              ▼ rekomendasi
        Manager menggabungkan semua hasil → jawaban akhir
```
Versi konkret di SupportPilot: delegasi diimplementasikan sebagai **tool tambahan** yang ditawarkan ke support agent. Tool ini, kalau dipanggil model, gak menjalankan mock function biasa — dia menjalankan `run_agent_loop` LAGI (billing agent), lalu jawaban billing agent itu dikembalikan sebagai *tool result* ke support agent punya loop sendiri:
```
Support Agent punya loop (run_agent_loop, tools=SUPPORT_TOOLS_WITH_DELEGATION):

  Step 1: model manggil get_ticket_status("T-777")
          → hasil: tiket di-hold, alasan "menunggu konfirmasi pembayaran"
  Step 2: model menyadari ini butuh info billing yang gak dia punya,
          manggil delegate_to_billing_agent("Kenapa tiket T-777 di-hold ...?")
          → tool ini MENJALANKAN run_agent_loop lain (billing agent) secara
            penuh (bisa beberapa step internal sendiri: get_invoice_details, dst),
            baru hasil akhirnya (teks) dikembalikan sebagai tool result
          → tool result: {"billing_agent_answer": "Invoice INV-2201 belum ..."}
  Step 3: model support agent SEKARANG punya kedua info (status tiket +
          jawaban billing agent), gabungkan jadi satu jawaban final ke customer
```
Bedanya sama topik 39: di sini cuma **satu** `run_agent_loop` top-level yang customer "lihat" (support agent), dan billing agent jalan sebagai sub-panggilan di tengah loop itu — bukan dua panggilan `run_agent_loop` yang independen dan hasilnya digabung manual di luar.

### Contoh Kode — Python
Tool delegasi butuh akses ke `client` (buat manggil `run_agent_loop` billing agent), padahal `TOOL_REGISTRY[tool_name](**tool_arguments)` di topik 25 cuma mengoper argumen dari model — gak ada tempat buat "menyisipkan" `client` di situ. Solusinya: bikin tool function lewat **closure/factory**, `client` di-*capture* dari luar begitu dibuat, bukan diminta sebagai argumen dari model:
```python
def make_delegate_to_billing_agent(client):
    """
    Factory yang menghasilkan fungsi tool 'delegate_to_billing_agent', dengan
    `client` sudah ditutup (closure) di dalamnya. Model cuma perlu ngisi
    argumen `question` -- `client` gak pernah diminta/dilihat model sama sekali.
    """

    def delegate_to_billing_agent(question: str) -> dict:
        """
        Dipanggil support agent kalau nemu pertanyaan yang sebenarnya ranah
        billing agent. Menjalankan run_agent_loop (topik 25) PENUH dengan
        BILLING_TOOLS, lalu jawaban akhirnya dikembalikan sebagai tool result
        biasa -- supaya support agent punya loop bisa lanjut mikir dengan
        info itu di step berikutnya.
        """
        billing_answer = run_agent_loop(
            client, question, tools=BILLING_TOOLS, max_steps=5
        )
        return {"billing_agent_answer": billing_answer}

    return delegate_to_billing_agent
```

Schema tool delegasi ini, ditambahkan ke katalog tool support agent (di luar `SUPPORT_TOOLS` biasa dari topik 39, khusus dipakai di skenario delegasi):
```python
DELEGATE_TO_BILLING_TOOL = {
    "type": "function",
    "function": {
        "name": "delegate_to_billing_agent",
        "description": (
            "Delegasikan pertanyaan yang menyangkut billing (invoice, "
            "pembayaran, refund) ke billing agent, dan dapatkan jawabannya. "
            "Pakai ini kalau kamu (support agent) nemu bagian pertanyaan "
            "customer yang di luar keahlianmu dan butuh info dari sisi billing."
        ),
        "parameters": {
            "type": "object",
            "properties": {
                "question": {
                    "type": "string",
                    "description": "Pertanyaan billing yang mau didelegasikan, dalam bahasa natural.",
                }
            },
            "required": ["question"],
        },
    },
}

SUPPORT_TOOLS_WITH_DELEGATION = SUPPORT_TOOLS + [DELEGATE_TO_BILLING_TOOL]
```

Extend `run_multi_agent_flow` (topik 39): sekarang branch "support" mendaftarkan `delegate_to_billing_agent` ke `TOOL_REGISTRY` (dengan `client` sudah ter-*capture* lewat factory di atas) sebelum menjalankan support agent, supaya kalau model memutuskan manggil tool itu, eksekusinya udah siap:
```python
def run_multi_agent_flow(user_message: str) -> str:
    """
    Manager: route ke billing agent (kalau memang murni billing), atau ke
    support agent yang sekarang dibekali kemampuan delegasi mid-conversation
    ke billing agent lewat tool delegate_to_billing_agent.
    """
    route = classify_route(user_message)

    if route == "billing":
        return run_agent_loop(client, user_message, tools=BILLING_TOOLS, max_steps=5)

    # default: support agent, DIBEKALI kemampuan delegasi ke billing agent
    TOOL_REGISTRY["delegate_to_billing_agent"] = make_delegate_to_billing_agent(client)
    return run_agent_loop(
        client, user_message, tools=SUPPORT_TOOLS_WITH_DELEGATION, max_steps=5
    )
```

Contoh menjalankan skenario campuran: *"Tiket T-777 saya kok di-hold terus ya, padahal saya udah lama nunggu?"* — ternyata alasannya adalah dispute pembayaran, sesuatu yang cuma diketahui billing agent:
```python
jawaban = run_multi_agent_flow(
    "Tiket T-777 saya kok di-hold terus ya, padahal saya udah lama nunggu?"
)
print(jawaban)
```
Yang terjadi, langkah demi langkah, SEMUA di dalam SATU `run_agent_loop` milik support agent:
1. **Step 1** — model manggil `get_ticket_status("T-777")`, hasilnya `{"status": "on_hold", "hold_reason": "menunggu konfirmasi pembayaran"}` (field baru ini menjelaskan kenapa di-hold).
2. **Step 2** — model melihat `hold_reason` menyangkut pembayaran, sesuatu yang di luar tool support biasa, jadi model manggil `delegate_to_billing_agent("Kenapa ada dispute pembayaran untuk customer terkait tiket T-777?")`. Ini men-trigger `run_agent_loop` billing agent SECARA PENUH (billing agent bisa manggil `get_invoice_details` dulu di dalam sub-loop-nya sendiri sebelum kasih jawaban), dan hasil akhirnya — misal `"Invoice INV-2201 belum lunas karena kartu ditolak; refund tidak berlaku."` — dikembalikan sebagai tool result `{"billing_agent_answer": "Invoice INV-2201 belum lunas ..."}`.
3. **Step 3** — support agent sekarang punya DUA info: status tiket (dari step 1) DAN jawaban billing agent (dari step 2). Model menggabungkan keduanya jadi satu jawaban koheren ke customer, misal: *"Tiket T-777 kamu di-hold karena ada kendala pembayaran — invoice INV-2201 belum lunas karena kartu ditolak. Begitu pembayarannya berhasil, tiketnya bakal lanjut diproses lagi."*

Perhatikan: customer cuma "melihat" satu agent (support agent) yang menjawab — proses delegasi ke billing agent sepenuhnya terjadi di dalam satu tool call, tersembunyi dari customer, dan hasilnya benar-benar dipakai (bukan cuma dua jawaban terpisah yang ditempel manual) karena model support agent sendiri yang membaca tool result itu di step 3 dan menyusun ulang kalimatnya.

### Trade-off & Pitfall
- **Delegasi nested berarti biaya dan latency dobel** — satu tool call `delegate_to_billing_agent` men-trigger seluruh `run_agent_loop` billing agent (yang sendirinya bisa berapa step), sebelum support agent bisa lanjut ke step berikutnya; customer nunggu total waktu KEDUA loop itu, bukan cuma satu.
- **`max_steps` billing agent yang jadi sub-panggilan tetap harus dijaga masuk akal** — kalau billing agent butuh banyak step buat selesai, support agent (yang menunggu tool result-nya) juga ikut ketahan; `max_steps` yang gede di level manapun ikut menambah worst-case latency total.
- **Closure `make_delegate_to_billing_agent(client)` harus dipanggil ULANG tiap kali `client` berubah** (misal beda API key per environment) — kalau `TOOL_REGISTRY["delegate_to_billing_agent"]` didaftarkan sekali lalu `client`-nya berubah tempat lain, closure lama tetap "mengingat" `client` yang sudah usang.
- **Delegasi berlapis-lapis (agent A delegasi ke B, B delegasi lagi ke C, dst) gampang jadi susah ditelusuri** — tiap lapis delegasi nambah satu level nested `run_agent_loop` yang harus di-debug terpisah kalau ada yang salah; batasi kedalaman delegasi (biasanya cukup satu lapis) kecuali benar-benar perlu.

### Kapan Dipakai
- Pakai delegasi (bukan cuma routing topik 39) begitu permintaan customer **genuinely campuran** antar domain dalam satu percakapan — agent utama butuh info dari agent lain buat menyelesaikan jawabannya sendiri, bukan sekadar "salah satu dari dua".
- Kalau ternyata SEBAGIAN BESAR permintaan yang datang ke support agent berakhir didelegasikan ke billing agent, itu sinyal classify_route (topik 39) mungkin perlu dipertajam supaya permintaan seperti itu langsung di-route ke billing agent sejak awal, bukan lewat delegasi tiap kali.
- Batasi delegasi ke satu lapis di awal (support → billing) sebelum menambah pola delegasi berantai yang lebih dalam — kompleksitas debugging naik cepat tiap lapis tambahan (lihat topik 41).

### Sering Ditanya Saat Interview
- **Apa beda mendasar antara routing (topik 39) dan delegation (topik 40)?** — routing memilih SATU agent yang menangani seluruh permintaan dari awal; delegation terjadi MID-CONVERSATION, di mana satu agent yang sedang berjalan meminta bantuan agent lain buat sub-bagian tugasnya, lalu menggabungkan hasilnya sendiri.
- **Kenapa `delegate_to_billing_agent` dibungkus lewat closure/factory, bukan langsung jadi fungsi biasa?** — karena `TOOL_REGISTRY[tool_name](**tool_arguments)` cuma mengoper argumen yang dikasih model; `client` bukan argumen yang model isi, jadi harus di-*capture* lewat closure supaya tersedia saat tool itu dieksekusi.
- **Gimana caranya hasil delegasi benar-benar "masuk" ke jawaban akhir, bukan cuma ditempel di luar?** — hasil billing agent dikembalikan sebagai tool result biasa ke messages support agent (sama seperti tool result lain di topik 25); support agent punya loop lanjut ke step berikutnya dan model-nya sendiri yang membaca tool result itu buat menyusun jawaban final.
- **Apa risiko latency terbesar dari pola delegasi ini?** — nested call: satu tool call bisa men-trigger seluruh `run_agent_loop` agent lain (yang sendirinya beberapa step), jadi customer menunggu total waktu kedua loop, bukan cuma satu.

---

## 41. Multi-Agent Tradeoffs

### Apa itu?
Setelah ngelihat pola routing (topik 39) dan delegation (topik 40), topik ini merangkum kapan multi-agent benar-benar sepadan dibanding tetap pakai single agent (topik 38) — bukan soal mana yang "lebih canggih", tapi soal kecocokan sama kebutuhan riil.

### Kenapa dibutuhkan?
Multi-agent kelihatan menarik karena tiap agent bisa fokus dan tajam di domainnya, tapi itu bukan berarti gratis — tiap agent tambahan berarti tiap potensi panggilan LLM tambahan, tiap potensi titik kegagalan tambahan, dan koordinasi (routing, delegasi) yang juga bisa salah. Tanpa kesadaran soal trade-off ini, tim gampang jatuh ke pola "pecah jadi banyak agent" padahal single agent yang dibekali beberapa tool tambahan (Phase 6 topik 28) sebenarnya sudah cukup dan jauh lebih murah dijaga.

### Cara Kerja
Perbandingan langsung, berdasarkan yang udah dibangun di topik 38-40:
```
Manfaat multi-agent:
  - Spesialisasi   -> tiap agent fokus satu domain (system prompt & tool lebih tajam)
  - Eksekusi paralel -> beberapa agent spesialis bisa jalan bersamaan (kalau tugasnya
                        independen, tidak seperti pola delegasi sekuensial topik 40)
  - Separation of concerns -> gagal/salah di satu agent gak otomatis merusak agent lain

Biaya multi-agent:
  - Lebih banyak token -> tiap agent = system prompt sendiri, konteks sendiri
  - Cost & latency lebih tinggi -> tiap agent tambahan = tiap potensi panggilan LLM
                                    tambahan (topik 40: nested call bikin ini lebih parah)
  - Kompleksitas koordinasi -> perlu router/manager (topik 39) atau mekanisme delegasi
                                (topik 40) yang sendirinya bisa salah
  - Lebih banyak failure mode -> agent salah pilih tool (topik 28), router salah route
                                  (topik 39), delegasi nested gagal di tengah (topik 40)
                                  -- masing-masing titik kegagalan baru yang gak ada
                                  di single agent
```
Rule of thumb: **jangan pakai 5 agent kalau 1 agent + 3 tool udah cukup.**

### Trade-off & Pitfall
- **Menambah agent itu gampang, tapi menghapus/menyederhanakan lagi jauh lebih susah** — begitu router/delegasi sudah tertanam di banyak tempat kode, "menyatukan lagi jadi satu agent" butuh refactor yang gak sepele; keputusan pecah jadi multi-agent sebaiknya didasarkan bukti kebutuhan riil, bukan asumsi di awal.
- **Tiap agent tambahan = tiap system prompt yang harus tetap konsisten satu sama lain** — support agent dan billing agent (topik 39) yang "gaya bicaranya" beda ke customer bikin pengalaman terasa gak nyambung kalau keduanya terlibat di satu percakapan (topik 40).
- **Failure mode terakumulasi, bukan cuma sejumlah agent yang dipakai** — router salah pilih (topik 39) DITAMBAH kemungkinan billing agent sendiri gagal di sub-loop-nya (topik 40) berarti total probabilitas kegagalan end-to-end lebih tinggi dibanding satu single agent yang mikirin semuanya sendiri.
- **"Multi-agent" gampang jadi trend yang dipaksakan** — sama kayak godaan "pakai agent padahal workflow cukup" (Phase 6 topik 26), godaan "pakai multi-agent padahal single agent + tool cukup" juga nyata; keduanya butuh justifikasi berbasis kebutuhan riil, bukan ikut tren arsitektur.

### Kapan Dipakai
- Pertimbangkan multi-agent HANYA kalau salah satu manfaatnya (spesialisasi, paralelisme, separation of concerns) benar-benar dibutuhkan DAN kompleksitas koordinasi tambahannya sepadan — bukan default pertama yang dicoba.
- Kalau ragu, mulai dari single agent + tool set yang relevan (topik 38); pecah jadi multi-agent baru kalau ada bukti konkret (system prompt yang generalis, tool yang tumpang tindih antar domain, atau kebutuhan paralelisme nyata) yang menunjukkan single agent sudah gak cukup.
- Kalau permintaan campuran (topik 40) cuma terjadi sesekali, delegasi ad-hoc lewat satu tool tambahan (seperti `delegate_to_billing_agent`) bisa lebih murah dan lebih simpel dibanding merombak jadi router penuh yang menangani tiap kombinasi domain.

### Sering Ditanya Saat Interview
- **Sebutkan manfaat utama arsitektur multi-agent dibanding single agent.** — spesialisasi (tiap agent fokus satu domain), eksekusi paralel (kalau tugasnya independen), dan separation of concerns (kegagalan di satu agent gak otomatis merusak agent lain).
- **Sebutkan biaya utama arsitektur multi-agent.** — lebih banyak token/biaya (tiap agent punya konteks sendiri), latency lebih tinggi (terutama pada delegasi nested), kompleksitas koordinasi (perlu router/manager), dan lebih banyak failure mode (router salah route, agent salah pilih tool, delegasi gagal di tengah).
- **Apa rule of thumb buat memutuskan jumlah agent yang tepat?** — jangan pakai 5 agent kalau 1 agent + 3 tool sudah cukup; jumlah agent harus mengikuti kebutuhan spesialisasi/paralelisme riil, bukan sekadar mengikuti tren.
- **Kenapa keputusan pecah jadi multi-agent sebaiknya berbasis bukti, bukan asumsi awal?** — karena menambah agent gampang, tapi menyederhanakan kembali (menyatukan router/delegasi yang sudah tertanam di kode) jauh lebih sulit; mulai dari single agent (topik 38) dan pecah cuma kalau ada bukti konkret kebutuhannya.

---

**Selanjutnya:** [Phase 11 — Agent Runtimes / Harnesses](./phase-11-agent-runtimes-harnesses.md)
