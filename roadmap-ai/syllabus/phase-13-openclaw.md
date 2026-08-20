# Phase 13 — OpenClaw

> Bagian dari [AI Engineering / Agent Roadmap](../README.md)

---

## 50. OpenClaw Concepts

### Apa itu?
OpenClaw adalah contoh lain dari personal AI agent system/runtime — mirip Hermes Agent (Phase 12) dalam hal *kategori* produk (agent runtime siap pakai buat dipasang jadi asisten personal), tapi punya riwayat asal yang penting diketahui: dulu bernama **Clawdbot**, sempat rebrand jadi **Moltbot**, sebelum akhirnya settle jadi nama **OpenClaw**. Sama seperti Phase 12, fase ini **BUKAN tutorial instalasi/command** — tujuannya paham ARSITEKTUR-nya: Gateway, Agent, Skills, Tools, Channels, Sessions, Memory, Scheduling, Model providers, Local execution, Permissions, Sandboxing. Nama command spesifik gampang basi (apalagi produk yang riwayat rebranding-nya sendiri sudah tiga kali) — arsitekturnya yang lebih tahan lama buat dipelajari.

### Kenapa dibutuhkan?
Phase 11 sudah menjelaskan kenapa bare agent loop butuh dibungkus jadi runtime production (session, memory, permission, sandboxing, dst). OpenClaw relevan dipelajari karena dia contoh KEDUA (setelah Hermes di Phase 12) dari pola yang sama, tapi dengan penekanan yang agak berbeda: OpenClaw secara historis lebih dikenal sebagai runtime personal yang dijalankan sendiri (self-hosted) oleh penggunanya, sering di server/mesin pribadi, dan terhubung ke banyak channel komunikasi personal (Telegram, WhatsApp, dst — topik 51). Ini penting dipelajari BUKAN cuma buat nambah referensi arsitektur, tapi karena OpenClaw juga adalah studi kasus nyata soal apa yang terjadi kalau lapisan security di sekitar agent runtime diabaikan — relevan langsung ke Phase 14 (Agent Security) di bawah.

> **Catatan penting (studi kasus nyata):** OpenClaw (dulu bernama Clawdbot, sempat jadi Moltbot sebelum settle jadi OpenClaw) pernah viral banget di awal 2026, tapi juga sempat mengalami insiden security serius — ratusan instance yang ke-expose ke internet **tanpa authentication** (bocor API key & chat history), dan marketplace skill komunitasnya sempat kemasukan **ribuan skill yang malicious**. Ini bukan cuma trivia — ini adalah studi kasus nyata untuk topik-topik di Phase 14 (Agent Security): kegagalan authentication dasar dan risiko supply-chain dari skill/tool pihak ketiga.

### Cara Kerja
Diagram arsitektur:
```
User → OpenClaw Gateway → Agent → Tools / Skills / Channels / Memory / Model
```
Cara bacanya: request dari user (lewat channel apapun — topik 51) masuk lewat **Gateway**, lapisan yang menerima raw message dari channel, melakukan routing/otentikasi awal, lalu meneruskannya ke instance **Agent** yang sesuai. Agent ini adalah LLM + agent loop (Phase 6) yang, begitu butuh "ACT", bisa bercabang ke beberapa lapisan pendukung:
- **Tools** (Phase 6, Phase 9 lewat MCP) — kemampuan konkret yang bisa dieksekusi.
- **Skills** (Phase 8) — paket instruksi/tool per domain task, dimuat sesuai kebutuhan.
- **Channels** (topik 51) — jalur komunikasi masuk/keluar; satu agent bisa dipasang ke banyak channel sekaligus.
- **Memory** (Phase 7) — persistent context lintas sesi.
- **Model** — lapisan **Model providers**, abstraksi supaya agent bisa dikonfigurasi pakai LLM provider yang berbeda-beda (mirip LLM Gateway di Phase 5 topik 20), tanpa mengubah logika agent-nya.

Beberapa konsep tambahan yang relevan buat OpenClaw sebagai runtime personal:
- **Sessions** — state percakapan aktif per user/channel (setara short-term memory, Phase 7 topik 29), supaya agent tahu ini lanjutan percakapan yang mana.
- **Scheduling** — kemampuan menjalankan task terjadwal/berkala tanpa user harus trigger manual (misal reminder harian) — konsep yang sama dengan background execution di Phase 11 topik 42.
- **Local execution** — karena sering dijalankan self-hosted, OpenClaw bisa memberi agent akses eksekusi LOKAL di mesin tempat dia jalan (file, shell, dst) — ini yang membuat lapisan **Permissions** dan **Sandboxing** (Phase 11 topik 43 & 45) jadi krusial, bukan opsional. Insiden security yang disebut di atas adalah contoh konkret apa yang terjadi kalau lapisan ini lemah/hilang.

### Contoh Kode — Python
**Catatan penting:** blok di bawah ini ILUSTRATIF/KONSEPTUAL saja — sekadar nunjukin BENTUK/SHAPE seperti apa sebuah konfigurasi gateway bergaya OpenClaw secara konsep mungkin terlihat (channel mana yang disambungkan ke agent mana, model provider apa yang dipakai). Ini BUKAN skema config yang diverifikasi terhadap OpenClaw yang sesungguhnya, dan BUKAN command atau API call nyata. Field, nama key, dan format persisnya bisa saja sangat berbeda — **selalu cek dokumentasi resmi/repository resmi OpenClaw** buat skema config yang benar-benar berlaku saat ini.
```python
# ILUSTRATIF -- sketsa konseptual, BUKAN skema config OpenClaw yang terverifikasi.
openclaw_gateway_config_sketch = {
    "gateway": {
        # Daftar channel yang di-listen gateway ini -- lihat topik 51.
        "channels": ["telegram", "whatsapp", "web"],
        # PENTING secara konsep: gateway harus mewajibkan authentication
        # sebelum meneruskan request ke agent -- insiden security yang
        # dibahas di atas terjadi ketika lapisan ini absen/lemah.
        "require_authentication": True,
    },
    "agent": {
        "agent_name": "personal-assistant",
        "model_provider": "<pilihan-provider-sesuai-dokumentasi-resmi>",
        "capabilities_enabled": {
            "skills": True,
            "memory": True,
            "scheduling": True,
            "local_execution": False,  # off by default -- baca Trade-off di bawah
        },
    },
    # Field lain (permission scoping per-skill, sandbox profile, dst)
    # sengaja gak disertakan -- bentuk pastinya perlu dicek ke dokumentasi
    # resmi, bukan ditebak dari sketsa ini.
}
```
Cara membacanya: konfigurasi seperti ini secara konsep memisahkan "siapa yang boleh masuk lewat channel apa" (gateway) dari "apa yang agent boleh lakukan begitu request masuk" (capabilities) — pemisahan yang sama pentingnya dengan permission scoping di Phase 11 topik 45, dan jadi makin krusial justru karena runtime seperti ini sering dijalankan self-hosted oleh individu yang belum tentu punya pengalaman hardening server production.

### Trade-off & Pitfall
- **Self-hosted/personal runtime memindahkan tanggung jawab security ke operator individu** — beda dengan SaaS terkelola yang authentication-nya sudah dipaksakan platform, runtime yang dijalankan sendiri (seperti OpenClaw) menaruh keputusan "apakah expose ke internet dengan aman" di tangan orang yang menjalankannya — insiden ratusan instance tanpa authentication adalah akibat langsung dari asumsi yang keliru soal ini.
- **"Local execution" itu kapabilitas dua sisi mata pisau** — sangat berguna (agent bisa benar-benar bantu kerjaan di mesin sendiri), tapi kalau gateway di depannya gak terautentikasi dengan benar, kapabilitas ini yang paling berbahaya buat di-abuse oleh pihak luar.
- **Marketplace skill pihak ketiga = attack surface supply-chain** — insiden ribuan skill malicious di marketplace komunitas OpenClaw adalah contoh nyata dari risiko yang dibahas lebih detail di Phase 14 topik 57 (Skill/Tool Supply Chain Security): instal skill dari sumber yang belum di-review sama saja dengan menjalankan kode pihak ketiga yang gak diverifikasi.
- **Jangan hafalin nama/command OpenClaw** — riwayat rebranding-nya sendiri (Clawdbot → Moltbot → OpenClaw) adalah sinyal bahwa detail permukaan produk ini berubah-ubah; yang stabil buat dipelajari adalah pola arsitekturnya (gateway + agent + lapisan pendukung), bukan syntax spesifiknya.

### Kapan Dipakai
- Pelajari arsitektur OpenClaw (diagram di atas) kalau butuh contoh KEDUA — setelah Hermes di Phase 12 — buat memperkuat pemahaman bahwa pola "agent loop dikelilingi lapisan pendukung" itu berulang di banyak runtime pihak ketiga, bukan cuma satu produk.
- Pelajari studi kasus security-nya SEBELUM (atau bersamaan dengan) mempertimbangkan pemakaian runtime self-hosted serupa untuk keperluan apapun — Phase 14 di bawah membahas mitigasinya secara eksplisit (authentication, tool permission, supply chain security).
- Jangan mulai dari mencari command instalasi OpenClaw sebelum paham Phase 6 (agent loop) dan Phase 11 (runtime/harness) — tanpa kerangka itu, detail OpenClaw cuma jadi hafalan kosong.

### Sering Ditanya Saat Interview
- **Apa itu OpenClaw, secara arsitektur?** — personal AI agent runtime dengan Gateway (entry point multi-channel), Agent (LLM + loop), dan lapisan pendukung: Skills, Tools, Channels, Sessions, Memory, Scheduling, Model providers, Local execution, Permissions, dan Sandboxing.
- **Apa insiden security yang pernah terjadi pada OpenClaw, dan kenapa relevan buat dipelajari?** — ratusan instance yang ke-expose ke internet tanpa authentication (bocor API key & chat history), plus marketplace skill komunitasnya kemasukan ribuan skill malicious — studi kasus nyata buat dua topik di Phase 14: kegagalan authentication dasar dan risiko supply-chain skill/tool pihak ketiga.
- **Kenapa "local execution" jadi kapabilitas yang perlu diwaspadai khusus di runtime seperti OpenClaw?** — karena runtime ini sering dijalankan self-hosted dengan akses ke mesin/lingkungan nyata milik penggunanya; kalau gateway di depannya gak terautentikasi, kapabilitas eksekusi lokal itu yang paling berbahaya buat diakses pihak luar.
- **Apa pelajaran utama dari insiden OpenClaw buat siapapun yang membangun/menjalankan agent runtime sendiri?** — authentication di gateway itu bukan opsional, dan skill/tool pihak ketiga dari marketplace komunitas harus diperlakukan sebagai supply-chain risk yang butuh review & permission scoping — bukan diinstal begitu saja karena kelihatan populer.

---

## 51. Agent Channels

### Apa itu?
Channel adalah jalur/interface tempat user berinteraksi dengan sebuah agent — Telegram, Discord, Slack, WhatsApp, Email, Web, CLI, dan lain-lain. Poin pentingnya: **channel cuma interface-nya aja**. Logika inti agent (agent loop, tools, memory, model) tetap SATU dan SAMA, terlepas dari lewat channel mana request itu datang.

### Kenapa dibutuhkan?
Tanpa pemisahan ini, developer akan tergoda menulis ulang logika agent buat setiap platform baru yang mau didukung (versi Telegram-nya sendiri, versi Slack-nya sendiri, dst) — duplikasi yang gampang jadi gak konsisten begitu ada perubahan di satu tempat tapi lupa diterapkan ke tempat lain. Dengan memisahkan "channel adapter" (yang paham format spesifik tiap platform) dari "agent core" (yang gak peduli platform apa yang manggil dia), satu agent bisa dipasang ke banyak channel sekaligus tanpa duplikasi logika inti — mirip prinsip abstraksi yang sama dengan LLM Gateway di Phase 5 topik 20 (satu interface, banyak backend yang bisa ditukar).

### Cara Kerja
Diagram:
```
WhatsApp → Gateway → Agent → Tools / Memory / LLM
```
Cara bacanya: pesan WhatsApp masuk dalam format spesifik WhatsApp (payload, webhook, dst), diterima oleh **Gateway** yang punya adapter khusus buat "menerjemahkan" format itu jadi representasi generik yang dipahami **Agent** (misal: `{sender, text, attachments}` generik, bukan struktur payload WhatsApp mentah). Agent core lalu memproses request itu SAMA PERSIS seperti kalau request itu datang dari channel lain — menjalankan agent loop, memanggil Tools/Memory/LLM sesuai kebutuhan (Phase 6). Setelah agent selesai, hasilnya dikembalikan ke Gateway, yang lalu menerjemahkan balik hasil generik itu ke format yang cocok buat dikirim lewat WhatsApp. Kalau request itu datang dari Slack, cuma langkah "terjemahan" di ujung-ujung (masuk & keluar) yang berbeda — Agent di tengah gak berubah sama sekali.

### Contoh Kode — Python
**Catatan penting:** blok di bawah ini ILUSTRATIF/KONSEPTUAL saja — sekadar nunjukin BENTUK/SHAPE seperti apa dua adapter channel yang berbeda bisa secara konsep memanggil satu fungsi agent core yang sama. Ini BUKAN integrasi SDK Telegram/Slack yang diverifikasi resmi, dan BUKAN command atau API call nyata dari OpenClaw. **Selalu cek dokumentasi resmi platform channel terkait (Telegram Bot API, Slack API, dst) dan dokumentasi resmi OpenClaw** buat integrasi yang benar-benar berlaku.
```python
# ILUSTRATIF -- sketsa konseptual, BUKAN integrasi SDK/API yang terverifikasi.

def run_agent_core(message_text: str, session_id: str) -> str:
    # Ini "otak" agent -- sama persis dipanggil dari channel manapun.
    # Di real case, ini menjalankan agent loop penuh: Phase 6 topik 25.
    return f"[agent core] menjawab untuk sesi {session_id}: {message_text}"


def telegram_adapter_sketch(telegram_update: dict) -> dict:
    # ILUSTRATIF -- bentuk payload asli Telegram Bot API bisa sangat berbeda,
    # cek dokumentasi resmi Telegram buat struktur sesungguhnya.
    text = telegram_update.get("message", {}).get("text", "")
    session_id = f"telegram:{telegram_update.get('chat_id')}"

    reply_text = run_agent_core(text, session_id)

    # Terjemahan balik ke bentuk yang (secara konsep) cocok buat dikirim ke Telegram.
    return {"chat_id": telegram_update.get("chat_id"), "text": reply_text}


def slack_adapter_sketch(slack_event: dict) -> dict:
    # ILUSTRATIF -- bentuk payload asli Slack Events API bisa sangat berbeda,
    # cek dokumentasi resmi Slack buat struktur sesungguhnya.
    text = slack_event.get("event", {}).get("text", "")
    session_id = f"slack:{slack_event.get('event', {}).get('channel')}"

    reply_text = run_agent_core(text, session_id)

    # Terjemahan balik ke bentuk yang (secara konsep) cocok buat dikirim ke Slack.
    return {"channel": slack_event.get("event", {}).get("channel"), "text": reply_text}
```
Perhatikan bahwa `run_agent_core` dipanggil identik dari kedua adapter — cuma bagian "menerjemahkan payload masuk" dan "membentuk balasan keluar" yang berbeda per channel. Ini inti dari kenapa channel disebut "cuma interface aja": logika keputusan/agent loop-nya gak pernah tahu (dan gak perlu tahu) dia sedang dipanggil dari Telegram atau Slack.

### Trade-off & Pitfall
- **Setiap channel punya batasan/format berbeda yang harus ditangani adapter-nya** — panjang pesan maksimum, dukungan attachment/media, threading, rate limit platform — kalau adapter gak menangani perbedaan ini, hasil generik dari agent core bisa gagal dikirim atau terpotong di channel tertentu.
- **Lebih banyak channel = lebih banyak permukaan yang perlu diamankan** — tiap channel biasanya punya mekanisme authentication/verification sendiri (webhook signature, token, dst); menambah channel baru berarti menambah satu lagi titik yang harus diverifikasi dengan benar sebelum request-nya dipercaya masuk ke agent core (relevan langsung ke insiden OpenClaw di topik 50).
- **Pemetaan identitas/session lintas channel bisa rumit** — kalau user yang sama bisa mengakses agent dari WhatsApp DAN Web, harus ada keputusan sadar: apakah keduanya dianggap satu identitas dengan memory yang sama, atau dua session yang terpisah — keputusan ini berdampak langsung ke desain Sessions & Memory (topik 50 & Phase 7).
- **Jangan asumsikan "menambah channel baru" itu murah** — walau agent core-nya gak berubah, tiap channel baru tetap butuh adapter baru yang perlu ditulis, diuji, dan diamankan sendiri — bukan sekadar toggle konfigurasi.

### Kapan Dipakai
- Pisahkan channel dari agent core begitu ada rencana (atau kemungkinan) agent yang sama akan diakses dari lebih dari satu platform — investasi di lapisan adapter ini akan terbayar begitu channel kedua/ketiga ditambahkan.
- Kalau agent cuma akan dipakai lewat satu channel tunggal (misal cuma internal CLI tools) dan gak ada rencana ekspansi, memisahkan channel dari agent core secara eksplisit mungkin belum perlu — jangan over-engineer abstraksi yang belum dibutuhkan.
- Pertimbangkan lapisan authentication per-channel SEJAK AWAL desain, bukan ditambahkan belakangan — studi kasus di topik 50 menunjukkan konsekuensi kalau lapisan ini diabaikan.

### Sering Ditanya Saat Interview
- **Apa maksud "channel cuma interface-nya aja"?** — logika inti agent (agent loop, tools, memory, model) tetap satu dan sama; yang berbeda per channel cuma cara menerjemahkan format pesan masuk/keluar spesifik platform itu.
- **Kenapa perlu memisahkan channel adapter dari agent core, bukan menulis logika terpisah per platform?** — supaya gak ada duplikasi logika inti yang gampang jadi gak konsisten; satu agent bisa dipasang ke banyak channel tanpa menulis ulang keputusan/behavior-nya di tiap tempat.
- **Apa risiko security yang bertambah begitu jumlah channel bertambah?** — tiap channel butuh mekanisme authentication/verification sendiri sebelum request-nya dipercaya; menambah channel tanpa mengamankan masing-masing dengan benar memperbesar attack surface — persis pelajaran dari insiden OpenClaw di topik 50.

---

**Selanjutnya:** [Phase 14 — Agent Security](./phase-14-agent-security.md)
