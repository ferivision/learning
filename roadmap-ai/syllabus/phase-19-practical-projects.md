# Phase 19 — Practical Projects

> Bagian dari [AI Engineering / Agent Roadmap](../README.md)

---

Fase ini beda dari 18 fase sebelumnya: gak ada topik baru, gak ada blok kode contoh, gak ada konsep yang belum pernah disinggung. Isinya cuma 6 project yang mem-framing ulang semua yang udah dibangun sepanjang Phase 1-18 jadi 6 bentuk aplikasi nyata yang biasa muncul di dunia kerja (dan di interview). Sebagian besar "kode"-nya udah ada — SupportPilot sendiri (Phase 2-18) adalah salah satu dari 6 project ini (Project 4) yang udah selesai dibangun potongan demi potongan. Tujuan fase ini cuma satu: liat gambar besarnya, dan paham komponen mana dari fase mana yang dipakai buat merakit tiap project.

---

## Project 1 — Basic LLM API

Ini adalah bentuk paling sederhana dari "aplikasi AI" — satu backend yang jadi perantara antara client dan LLM provider, tanpa RAG, tanpa agent, tanpa tool. SupportPilot sendiri mulai dari sini: `POST /chat` di Phase 5 topik 19, lalu di-refactor manggil lewat `LLMGateway` (Phase 5 topik 20) supaya provider LLM-nya gak di-hardcode di banyak tempat. Project ini pada dasarnya adalah checklist "hal-hal yang bikin `LLMGateway` + `POST /chat` versi topik 19-20 siap production", bukan membangun dari nol.

```
Client → Python Backend (FastAPI) → LLM API → Response
              │
              ├── LLMGateway.generate()  (Phase 5, topik 20)
              └── trace_llm_call()       (Phase 15, topik 42 — logging/observability)
```

Implementation checklist:
- [ ] Endpoint dasar `POST /chat` (Phase 5 topik 19) yang nerima pesan customer dan balikin jawaban JSON.
- [ ] Semua panggilan LLM lewat `LLMGateway` (Phase 5 topik 20), bukan manggil `OpenAI()`/`Anthropic()` langsung di endpoint.
- [ ] Streaming response pakai SSE (`chat_stream`, Phase 5 topik 21) supaya customer liat jawaban muncul token-by-token, bukan nunggu full response.
- [ ] Token tracking & cost estimation lewat `track_usage` (Phase 5 topik 22), di-wire ke `LLMGateway.generate()` supaya SETIAP request otomatis ke-log biayanya.
- [ ] Error handling: request ke LLM provider yang gagal (timeout, rate limit, response error) gak boleh bikin seluruh endpoint crash — dibungkus try/except dan balikin error response yang jelas ke client.
- [ ] Timeout eksplisit di panggilan LLM (jangan andalkan default library) supaya request yang macet gak nge-hang selamanya.
- [ ] Rate limiting di sisi backend (per customer/per API key), biar satu client gak bisa menghabiskan quota buat semua orang.
- [ ] Logging tiap panggilan LLM pakai `trace_llm_call` (Phase 15 topik 42) — input, output, latency, token count — supaya ada jejak buat debugging dan analisis biaya belakangan.

---

## Project 2 — RAG System

Ini adalah pipeline RAG penuh, dari dokumen mentah sampai jawaban yang di-ground ke sumber. Semua komponennya udah dibangun di Phase 3-4: `generate_embedding` (Phase 3) buat ubah teks jadi vector, `ingest_document` + `chunk_text` (Phase 4) buat masukin dokumen ke knowledge base, `retrieve_relevant_chunks` (Phase 4) buat nyari chunk yang relevan, dan `rerank_chunks` (Phase 4) buat mengurutkan ulang hasil retrieval sebelum dikirim ke LLM. Project ini adalah checklist buat mastiin pipeline itu lengkap dan bisa diukur kualitasnya — bukan menulis ulang retrieval dari nol.

```
PDF → Text Extraction → Chunking → Embedding → pgvector → Retriever → LLM → Answer
        (ingest_document)  (chunk_text)  (generate_embedding)   (retrieve_relevant_chunks
                                                                   + rerank_chunks)
```

Implementation checklist:
- [ ] Ekstraksi teks dari PDF/dokumen sumber, lalu `chunk_text` (Phase 4) buat pecah jadi potongan `Chunk` yang ukurannya wajar buat di-embed.
- [ ] `ingest_document` (Phase 4) buat nyimpen tiap `Chunk` beserta embedding-nya (`generate_embedding`, Phase 3) ke pgvector.
- [ ] Metadata per chunk (sumber dokumen, halaman, tanggal) disimpan bareng embedding-nya — bukan cuma teksnya doang — supaya bisa dipakai buat filtering dan citation belakangan.
- [ ] Top-K retrieval yang bisa disetel (`retrieve_relevant_chunks`, Phase 4) — jangan hardcode `k`, biar bisa di-tuning sesuai kebutuhan.
- [ ] Hybrid search (gabungan similarity search vector lewat `cosine_similarity`/pgvector dengan keyword search biasa) buat query yang lebih cocok dijawab lewat exact match daripada semantic similarity.
- [ ] Reranking hasil retrieval pakai `rerank_chunks` (Phase 4) sebelum chunk-nya dikirim ke LLM, supaya chunk paling relevan yang naik ke atas, bukan cuma urutan similarity mentah.
- [ ] Citation di jawaban akhir — jawaban LLM harus nunjuk balik ke chunk/dokumen sumber mana yang dipakai, bukan cuma teks polos tanpa sumber.
- [ ] Evaluasi pipeline pakai `evaluate_rag_pipeline` (Phase 4 topik 18) buat sinyal kasar (retrieval hit rate + answer correctness), atau versi yang lebih tajam `evaluate_rag_pipeline_v2` (Phase 15) yang misahin `retrieval_score` (precision) dan `generation_score` (correctness + faithfulness) kalau butuh tau persis sisi mana yang bermasalah.

---

## Project 3 — Tool-Calling Agent

Ini adalah agent paling dasar: satu LLM yang dibekali beberapa tool (bukan RAG, bukan memory, bukan multi-agent), dan bisa mutusin sendiri tool mana yang perlu dipanggil buat menjawab satu permintaan. Fondasinya udah ada dari Phase 2 (`call_llm_with_tools`) dan Phase 6 (`run_agent_loop` + tool schema, topik 28) — project ini cuma nunjukin gimana fondasi itu dipakai buat kasus yang butuh LEBIH DARI SATU tool dipanggil berurutan dalam satu loop.

```
User → Agent → Search / Database / Calculator
          │
          └── run_agent_loop()   (Phase 6, topik 25)
              tool schema         (Phase 6, topik 28)
```

Implementation checklist:
- [ ] Definisikan tool schema (nama, deskripsi, parameter) buat tiap tool sesuai format yang dibahas di Phase 6 topik 28 — model cuma boleh "liat" tool yang memang relevan buat domainnya (prinsip allowlisting).
- [ ] Pakai `run_agent_loop` (Phase 6 topik 25) sebagai loop utama — model mikir, manggil tool, dapat tool result, mikir lagi, sampai dapat jawaban final.
- [ ] Tool `get_order_status` (Phase 6) sebagai contoh tool yang udah ada, buat nunjukin gimana satu tool call terintegrasi ke loop.
- [ ] Tambahkan satu tool baru bergaya `calculate_total` (kalkulator sederhana: jumlahin angka dari data yang dibalikin tool lain) buat nunjukin gimana MULTIPLE tools dipanggil berantai dalam satu permintaan.
- [ ] Worked example end-to-end: customer nanya *"cari order terakhir customer ini dan hitung total pengeluarannya"* → model manggil `get_order_status` buat dapat daftar order → model manggil `calculate_total` dengan data order itu → model susun jawaban final dari hasil kedua tool.
- [ ] Error handling per tool call (try/except di sekitar eksekusi tool, Phase 6 topik 25) — satu tool yang gagal gak boleh bikin seluruh loop crash.

---

## Project 4 — Customer Support Agent

Project ini **adalah SupportPilot itu sendiri** — running example yang udah dibangun sepanjang seluruh syllabus ini, dari Phase 2 sampai Phase 18. Gak ada yang perlu dimulai dari nol di sini; bagian ini cuma ngumpulin balik semua potongan yang udah selesai dibangun jadi satu gambar arsitektur utuh, biar kelihatan gimana semuanya nyambung satu sama lain.

```
                         ┌── RAG (Phase 4: retrieve_relevant_chunks + rerank_chunks)
                         ├── CRM/Order Tool (Phase 6: get_order_status)
User → API → Agent ──────┼── Ticket Tool (Phase 6: create_support_ticket)
        (Phase 5)  (Phase 6:            └── Human Escalation (Phase 6: escalate_to_human)
        POST /chat  run_agent_loop)
```

Implementation checklist:
- [ ] Authentication & authorization di layer API (Phase 5) — siapa yang boleh manggil `POST /chat`, dan sebagai customer mana.
- [ ] Tool permission lewat `check_permission` (Phase 11) — pastikan agent cuma bisa manggil tool yang memang diizinkan buat role/context yang lagi jalan (misal customer biasa gak boleh trigger tool yang harusnya cuma buat admin).
- [ ] RAG buat jawab pertanyaan umum lewat knowledge base (`retrieve_relevant_chunks` + `rerank_chunks`, Phase 4), sebelum jatuh ke tool-tool lain.
- [ ] Tool-tool operasional: `get_order_status`, `create_support_ticket`, `escalate_to_human` (semua Phase 6) buat aksi yang butuh data/side-effect nyata.
- [ ] Memory lintas sesi lewat `ConversationMemory` + `save_long_term_memory`/`retrieve_memories` (Phase 7) — agent inget histori dan fakta soal customer yang sama di sesi berikutnya.
- [ ] Skill loading lewat `load_skill` (Phase 8) — instruksi khusus (misal `refund_policy_skill`) baru dimuat pas memang relevan, biar system prompt gak menggelembung.
- [ ] Guardrail keamanan: `detect_prompt_injection`, `validate_tool_call`, `redact_pii` (semua Phase 14) diterapkan di sekitar input customer dan tool call sebelum dieksekusi.
- [ ] Logging & tracing tiap panggilan LLM lewat `trace_llm_call` (Phase 15).
- [ ] Evaluasi kualitas end-to-end lewat `evaluate_agent_run` (Phase 15) — bukan cuma evaluasi RAG-nya doang, tapi seluruh transcript agent (tool call yang tepat, jawaban akhir yang benar, dst).

---

## Project 5 — Personal Agent

Beda dari Project 4 yang fokusnya customer support, project ini adalah asisten personal yang jalan lewat channel chat (misal Telegram) dan punya akses ke hal-hal yang lebih "pribadi": kalender, email, file lokal. Secara arsitektur, project ini persis pola yang udah dibahas di Phase 11 (agent runtime/harness) dan sangat dekat dengan apa yang Hermes Agent dan OpenClaw (Phase 12-13) sediakan sebagai produk runtime siap pakai — kalau mau bikin dari nol, komponen-komponen di bawah ini yang perlu dirakit sendiri; kalau mau pakai yang udah jadi, Hermes/OpenClaw udah menyatukan hampir semua lapisan ini.

```
Telegram → Agent Gateway → LLM → Web Search / Calendar / Email / Files / Memory
                              │
                              ├── SandboxedExecutor        (Phase 11)
                              └── require_human_approval    (Phase 11)
```

Implementation checklist:
- [ ] Gateway yang nerima pesan dari Telegram (atau channel lain) dan meneruskannya ke agent loop.
- [ ] Persistent memory lintas sesi (`ConversationMemory`, `save_long_term_memory`/`retrieve_memories`, Phase 7) — asisten personal harus inget preferensi dan konteks dari percakapan sebelumnya, bukan mulai dari nol tiap sesi.
- [ ] Skills (`load_skill`, Phase 8) buat instruksi khusus per kategori tugas (misal skill "cara jadwalin meeting", skill "cara balas email formal"), dimuat cuma pas relevan.
- [ ] Scheduling — kemampuan agent buat menjalankan aksi di waktu tertentu (reminder, follow-up terjadwal), bukan cuma react ke pesan masuk.
- [ ] Tool permission lewat `check_permission` (Phase 11) — akses ke file/email/kalender personal harus dibatasi per konteks, gak semua tool available buat semua situasi.
- [ ] Human approval lewat `require_human_approval` (Phase 11) buat aksi yang beresiko/gak reversibel (kirim email, hapus file, bikin janji di kalender orang lain) — agent berhenti dan minta konfirmasi dulu sebelum eksekusi.
- [ ] Sandboxing eksekusi kode/aksi sistem lewat `SandboxedExecutor` (Phase 11) — kalau agent bisa jalanin kode atau command shell (misal buat baca/tulis file), itu harus jalan di environment yang terisolasi.

---

## Project 6 — Multi-Agent System

Ini adalah pola orkestrasi multi-agent versi lebih kompleks dari yang udah dibangun di Phase 10. `run_multi_agent_flow` (Phase 10) udah nunjukin pola manager-router dengan dua spesialis (support agent + billing agent); project ini extend pola yang sama dengan menambah peran spesialis ketiga, dan fokusnya bergeser ke bagaimana MENGUKUR sistem multi-agent itu berhasil atau enggak — bukan cuma bikinnya jalan.

```
                    Manager Agent
                   /      |       \
            Research    Data     Writer
             Agent      Agent     Agent
                   \      |       /
                    Manager Agent → Final Report
```

Implementation checklist:
- [ ] Extend pola `run_multi_agent_flow` (Phase 10) dari dua spesialis (support/billing) jadi tiga peran baru yang lebih generik: Research Agent (nyari/ngumpulin informasi), Data Agent (mengolah/menganalisis data), Writer Agent (menyusun laporan akhir dari hasil dua agent lain).
- [ ] Manager agent yang me-routing tugas ke spesialis yang tepat, lalu menggabungkan hasilnya jadi satu output akhir yang koheren — pola delegasi yang sama seperti `delegate_to_billing_agent` (Phase 10), diperluas ke tiga peran.
- [ ] Ukur success rate — dari sekian banyak run, berapa persen yang menghasilkan laporan akhir yang benar/lengkap dibanding ground truth atau rubric.
- [ ] Ukur token cost — total token yang terpakai lintas SEMUA agent (manager + tiga spesialis) per run, bukan cuma satu LLM call.
- [ ] Ukur latency end-to-end — dari request masuk ke manager sampai laporan akhir keluar, termasuk waktu yang kepakai buat delegasi antar-agent.
- [ ] Ukur jumlah tool call per run — sistem multi-agent yang boros manggil tool secara gak perlu (looping, retry berlebihan) harus kelihatan dari metrik ini.
- [ ] Pakai `evaluate_agent_run` (Phase 15) sebagai basis pengukuran — diterapkan ke transcript gabungan lintas agent, bukan cuma transcript satu agent tunggal.
