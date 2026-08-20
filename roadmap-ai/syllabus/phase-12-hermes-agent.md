# Phase 12 — Hermes Agent

> Bagian dari [AI Engineering / Agent Roadmap](../README.md)

---

## 46. What to Understand About Hermes

### Apa itu?
Hermes Agent adalah agent/runtime otonom modern dari Nous Research — bukan sekadar satu skrip `run_agent_loop` (Phase 6), tapi sebuah **produk runtime siap pakai** yang sudah menyatukan hampir semua lapisan yang dibahas di Phase 11 (session, memory, permission, sandboxing) plus kemampuan tambahan: persistent memory, skills, tool use, web search, browser control, code execution, scheduling, subagents, MCP (Phase 9), banyak channel komunikasi, dan sandboxed execution environment. Fase ini **bukan tutorial instalasi** — tujuannya cuma satu: paham ARSITEKTUR di baliknya, supaya kalau ditanya di interview "gimana cara kerja agent runtime seperti Hermes?", bisa jawab dari prinsip, bukan dari hafalan command.
Source resmi: https://hermes-agent.nousresearch.com/docs/

### Kenapa dibutuhkan?
Fase 11 sudah menjelaskan KENAPA sebuah bare agent loop butuh dibungkus jadi runtime (session management, permission, sandboxing, human approval, dst) begitu dipakai production. Hermes Agent relevan buat dipelajari karena dia adalah **contoh nyata** runtime yang sudah menyediakan semua lapisan itu sebagai produk jadi — bukan sesuatu yang tim SupportPilot harus bangun dari nol. Paham arsitektur Hermes membantu mengenali pola yang sama di runtime pihak ketiga lain (termasuk OpenClaw, Phase 13): agent inti (LLM + loop) dikelilingi lapisan tools, skills, memory, browser, code execution, dan subagent — persis pola "harness" yang sudah dibahas di Phase 11.

### Cara Kerja
Diagram arsitektur berikut sama dengan yang ada di `roadmap-ai/README.md`, topik 46 — hafalkan bentuknya, bukan detail implementasinya:
```
User → Hermes Agent → LLM → Agent Loop
                              ├── Tools
                              ├── Skills
                              ├── Memory
                              ├── Browser
                              ├── Code Execution
                              └── Subagents
```
Cara bacanya: request dari User masuk ke Hermes Agent (lapisan produk/runtime), yang meneruskannya ke LLM untuk diproses lewat Agent Loop (pola OBSERVE → THINK/DECIDE → ACT → REPEAT yang sama seperti Phase 6, topik 25). Setiap kali loop itu memutuskan perlu "ACT", dia bercabang ke salah satu dari enam kapabilitas: Tools (Phase 6, Phase 9 lewat MCP), Skills (topik 47 di bawah, sama konsepnya dengan Phase 8), Memory (topik 48, sama konsepnya dengan Phase 7), Browser (kontrol browser buat browsing web), Code Execution (mirip `SandboxedExecutor` di Phase 11 topik 43, tapi versi production Hermes), dan Subagents (topik 49, sama konsepnya dengan Phase 10). Setelah cabang itu selesai, hasilnya kembali ke Agent Loop buat diproses lagi (mungkin butuh langkah tambahan, mungkin sudah cukup buat jawaban akhir).

### Contoh Kode — Python
**Catatan penting:** blok di bawah ini ILUSTRATIF/KONSEPTUAL saja — sekadar nunjukin BENTUK/SHAPE seperti apa sebuah konfigurasi agent Hermes secara konsep mungkin terlihat (nama agent, kapabilitas yang diaktifkan, pilihan model). Ini BUKAN skema config yang diverifikasi terhadap Hermes yang sesungguhnya, dan BUKAN command atau API call nyata. Field, nama key, dan format persisnya bisa saja berbeda — **selalu cek dokumentasi resmi Hermes** (https://hermes-agent.nousresearch.com/docs/) buat skema config yang benar-benar berlaku saat ini.
```python
# ILUSTRATIF -- sketsa konseptual, bukan skema config Hermes yang terverifikasi.
hermes_agent_config = {
    "agent_name": "support-triage-agent",
    "model": "<pilihan-model-sesuai-dokumentasi-resmi>",
    "capabilities_enabled": {
        "tools": True,
        "skills": True,
        "memory": True,
        "browser": False,
        "code_execution": False,
        "subagents": True,
    },
    # Bagian ini sekadar menunjukkan IDE bahwa agent dikonfigurasi lewat
    # deklarasi kapabilitas, bukan lewat kode imperatif -- persis seperti
    # pola konfigurasi deklaratif di banyak agent runtime modern.
}
```
Cara membacanya: konfigurasi seperti ini secara konsep memberi tahu runtime kapabilitas mana yang harus tersedia buat agent loop tertentu — mirip prinsip permission scoping di Phase 11 topik 45 (bukan setiap agent perlu semua kapabilitas aktif sekaligus).

### Trade-off & Pitfall
- **Menghafal nama command/CLI Hermes cepat basi** — produk pihak ketiga yang aktif dikembangkan bisa mengubah command, flag, atau skema config antar rilis; yang lebih tahan lama buat dipahami adalah ARSITEKTURnya (agent loop + enam cabang kapabilitas), bukan syntax spesifiknya.
- **Runtime siap pakai bukan berarti "kotak hitam tanpa trade-off"** — mengaktifkan browser control atau code execution di Hermes tetap butuh pemahaman tentang risiko yang sama dibahas di Phase 11 (topik 43 sandboxing, topik 44 human-in-the-loop) — runtime pihak ketiga menyediakan mekanismenya, tapi keputusan kapan mengaktifkan kapabilitas berisiko tetap ada di tangan yang mengonfigurasi.
- **Jangan salah kira semua runtime agent punya arsitektur identik** — Hermes punya kombinasi kapabilitas spesifik (browser, subagents, MCP, scheduling, multi-channel); runtime lain (termasuk OpenClaw di Phase 13) bisa punya penekanan atau susunan lapisan yang berbeda, walau prinsip dasarnya (agent loop dikelilingi lapisan pendukung) tetap serupa.

### Kapan Dipakai
- Pelajari arsitektur Hermes (diagram di atas) kalau butuh referensi konkret buat mendiskusikan "bagaimana agent runtime production modern biasanya disusun" di luar konteks proyek belajar sendiri (SupportPilot).
- Jangan mulai dari menghafal instalasi/CLI Hermes sebelum paham prinsip di Phase 6 (agent loop) dan Phase 11 (runtime/harness) — tanpa itu, detail Hermes cuma jadi hafalan kosong tanpa kerangka buat menempatkannya.
- Untuk detail command yang benar-benar terverifikasi dan terdokumentasi (bukan ilustratif), rujuk catatan di `roadmap-ai/README.md` (misalnya command migrasi `hermes claw migrate` yang memang didokumentasikan resmi) dan dokumentasi resmi Hermes langsung — jangan berasumsi command lain di luar itu ada tanpa verifikasi.

### Sering Ditanya Saat Interview
- **Apa itu Hermes Agent, secara arsitektur?** — agent runtime otonom dari Nous Research yang membungkus LLM + agent loop dengan enam kapabilitas terintegrasi: tools, skills, memory, browser, code execution, dan subagents, plus dukungan MCP, scheduling, dan multi-channel communication.
- **Kenapa penting paham arsitektur Hermes walau gak pernah menginstalnya?** — karena polanya (agent loop dikelilingi lapisan pendukung) representatif buat banyak agent runtime production modern; paham prinsipnya lebih tahan lama daripada menghafal command yang bisa berubah antar rilis.
- **Apa hubungan Hermes dengan konsep yang sudah dibahas di Phase 6 dan Phase 11?** — Hermes adalah contoh konkret hasil akhir dari ide "bare agent loop (Phase 6) dibungkus jadi runtime production (Phase 11)" — bedanya, Hermes adalah produk pihak ketiga siap pakai, bukan sesuatu yang dibangun sendiri dari nol.
- **Kalau ditanya detail command spesifik Hermes yang gak familiar, apa jawaban yang aman?** — jujur bahwa command spesifik perlu dicek ke dokumentasi resmi Hermes (https://hermes-agent.nousresearch.com/docs/) karena bisa berubah antar rilis, tapi tetap bisa menjelaskan ARSITEKTUR dan PRINSIP di baliknya dengan percaya diri.

---

## 47. Hermes Skills

### Apa itu?
Hermes Skills adalah penerapan konsep Skill (Phase 8, topik 34) di dalam runtime Hermes: paket instruksi + tool yang relevan buat satu kategori task, dimuat oleh agent HANYA saat dibutuhkan — bukan selalu menempel di setiap request. Kalau di Phase 8 skill diilustrasikan sebagai file YAML lokal (`skills/refund_policy_skill.yaml`) yang dibaca sendiri oleh kode `load_skill`, di Hermes pola yang sama diterapkan sebagai bagian dari arsitektur runtime-nya sendiri — modul skill yang bisa "dipasang" ke agent supaya agent itu jadi mampu menangani domain task baru tanpa mengubah instruksi inti agentnya.

### Kenapa dibutuhkan?
Alasan intinya identik dengan Phase 8: kalau SEMUA instruksi buat SEMUA kategori task dijejalkan ke satu system prompt raksasa, prompt jadi bengkak, biaya/latency naik karena instruksi yang gak relevan tetap ikut terkirim, dan model bisa "ke-distract" oleh instruksi yang gak nyambung dengan request saat itu. Pola "Agent → determine task → load relevant skill → use skill instructions/tools → execute" menyelesaikan ini di level runtime: agent memutuskan skill apa yang relevan buat task yang sedang dihadapi, memuat HANYA instruksi dan tool skill itu, lalu menjalankannya — modularitas yang sama seperti Phase 8, tapi diberikan sebagai fitur runtime yang sudah jadi, bukan sesuatu yang perlu dibangun manual lewat `load_skill` custom.

### Cara Kerja
Diagram ini sama dengan yang ada di `roadmap-ai/README.md`, topik 47:
```
Agent → determine task → load relevant skill → use skill instructions/tools → execute
```
Bandingkan langsung dengan pola `load_skill` di Phase 8: "determine task" setara dengan menentukan kategori request masuk (misal: keyword "refund" terdeteksi), "load relevant skill" setara dengan `load_skill("refund_policy_skill")` yang membaca file skill dan mem-parsenya, "use skill instructions/tools" setara dengan menambahkan instructions skill itu ke system prompt DAN mempersempit katalog tool ke tool milik skill itu saja, dan "execute" setara dengan melanjutkan ke agent loop seperti biasa (Phase 6, topik 25) dengan instruksi & tool yang sudah dipersempit itu. Bedanya: di Hermes, seluruh mekanisme "determine → load → use → execute" ini adalah bagian dari runtime itu sendiri, bukan kode custom yang ditulis dari nol seperti contoh di Phase 8.

### Contoh Kode — Python
**Catatan penting:** blok di bawah ini ILUSTRATIF saja — sekadar nunjukin BENTUK konseptual apa yang mungkin dikandung sebuah file skill bergaya Hermes (nama, deskripsi, instruksi, tool terkait), meniru struktur yang sudah dikenal dari Phase 8. Ini BUKAN skema file skill Hermes yang diverifikasi resmi. **Selalu cek dokumentasi resmi Hermes** buat format file skill yang benar-benar berlaku.
```python
# ILUSTRATIF -- sketsa konseptual struktur skill bergaya Hermes,
# BUKAN skema resmi yang terverifikasi terhadap dokumentasi Hermes.
hermes_skill_sketch = {
    "name": "refund_policy_skill",
    "description": "Menangani permintaan refund customer sesuai kebijakan.",
    "instructions": (
        "Ketika customer meminta refund, cek dulu status order, "
        "lalu ikuti kebijakan refund yang berlaku sebelum bertindak."
    ),
    "tools": ["get_order_status", "issue_refund"],
    # Field lain (versi, trigger/kondisi pemuatan, dependency ke skill
    # lain, dst) sengaja gak disertakan di sini karena bentuk pastinya
    # perlu dicek langsung ke dokumentasi resmi Hermes.
}
```
Perhatikan kesamaannya dengan struktur skill di Phase 8 (`name`, `description`, `instructions`, `tools`) — bentuk konseptualnya sama, walau format file dan cara Hermes memuatnya secara teknis bisa berbeda dan harus diverifikasi ke dokumentasi resmi.

### Trade-off & Pitfall
- **Sama seperti Phase 8: skill yang gak jelas cakupannya bikin "determine task" ambigu** — kalau dua skill punya deskripsi yang tumpang-tindih, mekanisme pemilihan skill (baik custom seperti Phase 8, maupun bawaan runtime seperti Hermes) bisa salah memuat skill yang gak tepat buat request tertentu.
- **Jangan asumsikan format file skill Hermes identik dengan contoh YAML di Phase 8** — kesamaannya cuma di level KONSEP (name/description/instructions/tools), bukan di level implementasi teknis; format persisnya (ekstensi file, struktur nested, cara registrasi) perlu dicek ke dokumentasi resmi, bukan ditebak dari kemiripan konsep.
- **Skill yang terlalu granular (satu skill per micro-task) menambah overhead "determine task"** sementara skill yang terlalu luas (satu skill buat banyak domain berbeda) mengalahkan tujuan modularitas — trade-off granularity ini sama seperti yang dibahas di Phase 8, dan berlaku juga buat skill di runtime manapun, termasuk Hermes.

### Kapan Dipakai
- Pikirkan skill Hermes sebagai cara memodulasi agent yang menangani banyak domain task berbeda-beda — mirip alasan kapan skill dipakai di Phase 8: begitu instruksi buat satu kategori task mulai terasa "asing" atau gak relevan buat kategori task lain.
- Kalau agentnya cuma menangani satu domain task sempit dan sederhana, menambahkan lapisan skill (baik custom maupun bawaan runtime) belum tentu perlu — sama seperti prinsip di Phase 8, jangan over-engineer modularitas sebelum ada kebutuhan nyata buat memisahkan domain task.

### Sering Ditanya Saat Interview
- **Apa hubungan konsep "skill" di Hermes dengan konsep skill yang dibahas di Phase 8?** — konsepnya sama persis: paket instruksi + tool yang relevan buat satu kategori task, dimuat sesuai kebutuhan, bukan selalu menempel di system prompt; bedanya, Hermes menyediakan mekanisme ini sebagai fitur runtime siap pakai, bukan sesuatu yang dibangun manual.
- **Kenapa loading skill "sesuai kebutuhan" lebih baik dibanding menjejalkan semua instruksi di satu system prompt?** — mengurangi ukuran prompt yang gak relevan buat request tertentu, menurunkan biaya/latency, dan mengurangi risiko model "ke-distract" oleh instruksi yang gak nyambung dengan task yang sedang dihadapi — alasan yang sama seperti Phase 8.
- **Apakah format file skill Hermes pasti sama dengan contoh YAML skill di Phase 8?** — tidak bisa diasumsikan begitu; kesamaannya cuma di level konsep (name, description, instructions, tools), format teknis persisnya harus dicek ke dokumentasi resmi Hermes.

---

## 48. Hermes Memory

### Apa itu?
Hermes Memory adalah penerapan konsep memory (Phase 7) di dalam satu runtime terintegrasi: gabungan dari **current conversation** (setara short-term memory, topik 29), **persistent memory** (setara long-term memory, topik 30 — bertahan lintas sesi), dan **retrieved context** (setara memory retrieval, topik 31 — potongan informasi relevan yang "diambil" dari penyimpanan persistent berdasarkan konteks percakapan saat ini, bukan seluruh isi memory dikirim mentah-mentah).

### Kenapa dibutuhkan?
Alasan dasarnya identik dengan Phase 7: sebuah agent tanpa memory apapun "amnesia" tiap kali sesi baru dimulai — gak inget preferensi customer, keputusan yang sudah dibuat sebelumnya, atau fakta penting lain dari interaksi lampau. Tapi Phase 7 juga sudah menekankan bahwa cuma menyimpan SEMUA memory dan mengirim semuanya mentah-mentah ke setiap request itu boros token dan bisa membebani context window (Phase 1, topik 4) — makanya perlu retrieval (topik 31) yang hanya mengambil bagian yang RELEVAN. Kombinasi tiga elemen ini di Hermes (current conversation + persistent memory + retrieved context) memungkinkan agent mempertahankan pengetahuan lintas sesi TANPA harus mengirim seluruh riwayat mentah setiap kali — pola yang sama persis dengan alasan kenapa `compact_context` (topik 32) dan memory retrieval (topik 31) dibahas terpisah di Phase 7.

### Cara Kerja
Diagram ini sama dengan yang ada di `roadmap-ai/README.md`, topik 48:
```
Current conversation + Persistent memory + Retrieved context
```
Cara bacanya, dipetakan langsung ke konsep Phase 7: "current conversation" adalah riwayat percakapan sesi aktif saat ini (setara `ConversationMemory` di topik 29 — hilang begitu sesi berakhir, kecuali sebagian isinya sengaja dipersist). "Persistent memory" adalah penyimpanan fakta yang memang didesain bertahan lintas sesi (setara `save_long_term_memory` di topik 30 — misalnya preferensi customer atau keputusan yang pernah diambil). "Retrieved context" adalah potongan dari persistent memory itu yang secara aktif DIAMBIL dan disuntikkan ke context saat ini karena relevan dengan apa yang sedang dibahas (setara memory retrieval di topik 31 — bukan seluruh isi persistent memory dikirim, cuma yang relevan). Ketiganya digabung jadi satu context yang dikirim ke LLM di setiap langkah agent loop.

### Contoh Kode — Python
**Catatan penting:** blok di bawah ini ILUSTRATIF saja — sekadar nunjukin BENTUK konseptual bagaimana satu entri memory mungkin secara konsep direpresentasikan (bukan API Hermes yang diverifikasi resmi, dan bukan skema database sungguhan). **Selalu cek dokumentasi resmi Hermes** buat struktur memory yang benar-benar dipakai runtime-nya.
```python
# ILUSTRATIF -- sketsa konseptual entri memory bergaya Hermes,
# BUKAN struktur/API memory Hermes yang terverifikasi resmi.
hermes_memory_entry_sketch = {
    "memory_type": "persistent",  # atau "current_conversation" / "retrieved_context"
    "content": "Customer lebih suka dihubungi lewat email, bukan telepon.",
    "source_session_id": "<id-sesi-tempat-fakta-ini-pertama-muncul>",
    "relevance_tags": ["preferensi_komunikasi"],
    # Field seperti timestamp, embedding buat retrieval, atau mekanisme
    # expiry sengaja gak disertakan -- strukturnya perlu dicek langsung
    # ke dokumentasi resmi Hermes, bukan ditebak dari sketsa ini.
}
```
Perhatikan bahwa `memory_type` di sketsa ini sengaja dipetakan ke tiga elemen di diagram ("current_conversation", "persistent", "retrieved_context") supaya hubungannya jelas — bukan klaim bahwa Hermes benar-benar memakai nama field ini.

### Trade-off & Pitfall
- **Jangan campur "persistent memory" dengan "retrieved context" seolah sama** — sama seperti pitfall di Phase 7 (jangan campur short-term dengan long-term memory), persistent memory adalah SELURUH penyimpanan lintas sesi, sedangkan retrieved context cuma POTONGAN yang relevan buat percakapan saat ini; keduanya elemen berbeda walau saling berkaitan.
- **Retrieval yang gak akurat membebani context tanpa manfaat** — kalau mekanisme "retrieved context" mengambil potongan memory yang gak relevan (analog dengan masalah retrieval quality di Phase 3-4 buat RAG), context yang dikirim ke LLM jadi penuh noise tanpa benar-benar membantu jawaban.
- **Skema penyimpanan memory yang sesungguhnya perlu dicek ke dokumentasi resmi** — sketsa dict di atas cuma menunjukkan konsep, bukan API atau format storage Hermes yang sesungguhnya; jangan berasumsi field seperti `memory_type` di atas benar-benar ada di produk nyatanya.

### Kapan Dipakai
- Pikirkan Hermes Memory sebagai kombinasi tiga konsep yang sudah dipelajari terpisah di Phase 7 (short-term, long-term, retrieval) — tapi disatukan sebagai satu lapisan runtime, bukan tiga komponen yang harus dijahit manual sendiri.
- Kalau perlu mendiskusikan "gimana agent runtime modern biasanya menangani memory lintas sesi", diagram tiga elemen ini (current conversation + persistent memory + retrieved context) adalah kerangka yang berguna buat dipakai — terlepas dari runtime spesifik apapun yang dibahas.

### Sering Ditanya Saat Interview
- **Apa tiga elemen memory yang digabung di Hermes, dan masing-masing setara dengan konsep Phase 7 apa?** — current conversation (setara short-term memory, topik 29), persistent memory (setara long-term memory, topik 30), dan retrieved context (setara memory retrieval, topik 31).
- **Kenapa gak cukup cuma punya persistent memory tanpa retrieval?** — karena mengirim SELURUH isi persistent memory di setiap request membebani context window dan menambah biaya/latency secara gak perlu (Phase 1, topik 4); retrieval memastikan cuma bagian yang relevan yang disuntikkan.
- **Apakah struktur data memory yang dipakai Hermes sungguhan sama dengan sketsa dict yang dipelajari di fase ini?** — tidak bisa diasumsikan; sketsa itu cuma ilustrasi konsep, struktur data dan API yang sesungguhnya harus dicek ke dokumentasi resmi Hermes.

---

## 49. Hermes Subagents

### Apa itu?
Hermes Subagents adalah penerapan pola main-agent-spawn-subagent (Phase 10) di dalam runtime Hermes: satu agent utama (main agent) yang sedang berjalan bisa **spawn** (memunculkan) subagent lain buat menangani sepotong pekerjaan yang butuh riset/eksekusi terisolasi, menunggu hasilnya, lalu melanjutkan pekerjaannya sendiri dengan hasil itu di tangan.

### Kenapa dibutuhkan?
Ini sama persis dengan motivasi di balik agent delegation (Phase 10, topik 40): begitu satu tugas kompleks butuh riset/eksekusi mendalam yang kalau dilakukan LANGSUNG di dalam loop main agent akan membengkakkan context-nya (semua langkah riset itu ikut numpuk sebagai riwayat di loop utama), lebih baik pekerjaan itu didelegasikan ke subagent yang punya context TERISOLASI sendiri — subagent itu bebas melakukan berapa pun langkah internalnya sendiri, dan main agent cuma menerima HASIL akhirnya, bukan seluruh proses berpikirnya. Ini juga membuka kemungkinan eksekusi paralel (beberapa subagent jalan bersamaan buat sub-tugas berbeda) dan agregasi hasil — pola yang sama seperti yang dibahas di Phase 10 topik 39-40 soal multi-agent dan delegation.

### Cara Kerja
Diagram ini sama dengan yang ada di `roadmap-ai/README.md`, topik 49:
```
Main Agent → spawn Subagent → Research → Return Result → Main Agent continues
```
Petakan ke Phase 10: "Main Agent → spawn Subagent" setara dengan main agent memanggil tool delegasi (mirip `delegate_to_billing_agent` di topik 40) yang alih-alih mock function biasa, benar-benar menjalankan agent loop LAIN secara penuh. "Research" adalah subagent itu mengerjakan sub-tugasnya sendiri secara terisolasi — bisa beberapa langkah internal sendiri, persis seperti billing agent di topik 40 yang bisa memanggil `get_invoice_details` dst di dalam loop internalnya sendiri sebelum kembali. "Return Result" adalah hasil AKHIR (bukan seluruh proses berpikir) subagent itu dikembalikan ke main agent sebagai semacam tool result. "Main Agent continues" adalah main agent melanjutkan loop-nya sendiri dengan hasil itu sebagai informasi tambahan, sama seperti support agent di topik 40 yang melanjutkan setelah menerima jawaban dari billing agent.

### Contoh Kode — Python
**Catatan penting:** blok di bawah ini ILUSTRATIF saja — sekadar nunjukin BENTUK konseptual permintaan spawn subagent dan hasil yang dikembalikan, meniru pola delegasi yang sudah dikenal dari Phase 10. Ini BUKAN API atau protokol spawn subagent Hermes yang diverifikasi resmi. **Selalu cek dokumentasi resmi Hermes** buat mekanisme spawn subagent yang benar-benar berlaku.
```python
# ILUSTRATIF -- sketsa konseptual permintaan & hasil spawn subagent
# bergaya Hermes, BUKAN API Hermes yang terverifikasi resmi.
subagent_spawn_request_sketch = {
    "task_description": "Riset perbandingan harga kompetitor untuk produk X.",
    "isolated_context": True,   # subagent gak "melihat" seluruh riwayat main agent
    "tools_allowed": ["web_search"],
}

subagent_result_sketch = {
    "status": "completed",
    "result_summary": (
        "Kompetitor A dan B menjual produk sejenis 10% lebih murah, "
        "tapi tanpa garansi tambahan."
    ),
    # Detail langkah internal riset subagent SENGAJA gak ikut dikembalikan
    # ke main agent -- main agent cuma butuh hasil akhirnya, bukan seluruh
    # proses berpikir subagent, persis prinsip context isolation di Phase 10.
}
```
Perhatikan bahwa `subagent_result_sketch` sengaja HANYA berisi ringkasan hasil, bukan seluruh riwayat langkah subagent — ini mencerminkan alasan utama kenapa pola subagent dipakai (isolasi context), bukan detail API yang sesungguhnya.

### Trade-off & Pitfall
- **Spawn subagent menambah latency** — subagent yang menjalankan agent loop-nya sendiri (mungkin berapa langkah) sebelum mengembalikan hasil berarti main agent harus menunggu seluruh proses itu selesai; ini trade-off yang sama seperti delegasi di Phase 10 topik 40, cuma sekarang jadi fitur runtime bawaan.
- **Isolasi context ada harganya: subagent gak "tau" apa yang sudah dibahas main agent** — kalau task description yang dikirim ke subagent gak cukup jelas/lengkap, subagent bisa melakukan riset yang gak tepat sasaran karena dia gak punya akses ke riwayat percakapan penuh; task description yang dikirim ke subagent harus cukup mandiri (self-contained).
- **Subagent yang berjalan paralel butuh strategi agregasi hasil yang jelas** — kalau beberapa subagent di-spawn bersamaan buat sub-tugas berbeda, main agent perlu logika buat menggabungkan hasil-hasil itu jadi satu jawaban koheren, bukan cuma menumpuknya mentah-mentah — sama seperti isu agregasi hasil di pola multi-agent Phase 10 topik 39.
- **Jangan asumsikan protokol spawn subagent Hermes identik dengan sketsa dict di atas** — bentuk permintaan/hasil sesungguhnya (termasuk cara subagent diberi batasan tool, timeout, dst) perlu dicek ke dokumentasi resmi, bukan ditebak dari kemiripan konsep dengan Phase 10.

### Kapan Dipakai
- Pikirkan subagent Hermes sebagai penerapan pola delegasi (Phase 10, topik 40) di level runtime: dipakai begitu satu sub-tugas butuh riset/eksekusi yang cukup dalam sehingga akan membengkakkan context main agent kalau dikerjakan langsung di loopnya sendiri.
- Untuk sub-tugas sederhana yang gak butuh banyak langkah internal, memanggil tool biasa (tanpa spawn subagent penuh) tetap lebih murah dan lebih cepat — spawn subagent baru bermanfaat begitu ada isolasi context yang benar-benar dibutuhkan, sama seperti kapan pola delegasi dipakai di Phase 10.

### Sering Ditanya Saat Interview
- **Apa pola dasar di balik Hermes Subagents, dan gimana hubungannya dengan Phase 10?** — main agent menunda sepotong pekerjaan ke subagent terisolasi, menunggu hasil akhirnya, lalu melanjutkan loop-nya sendiri dengan hasil itu — pola yang sama dengan agent delegation di Phase 10 topik 40, diterapkan sebagai fitur bawaan runtime.
- **Kenapa subagent cuma mengembalikan HASIL akhir, bukan seluruh riwayat langkah internalnya?** — supaya context main agent gak membengkak dengan detail proses riset yang gak perlu dia lihat; ini prinsip context isolation yang sama seperti alasan delegasi dipakai di Phase 10.
- **Apa risiko utama kalau task description yang dikirim ke subagent kurang lengkap?** — subagent bisa melakukan riset yang gak tepat sasaran karena dia gak punya akses ke riwayat percakapan main agent — task description yang dikirim harus cukup mandiri/self-contained.

---

**Selanjutnya:** [Phase 13 — OpenClaw](./phase-13-openclaw.md)
