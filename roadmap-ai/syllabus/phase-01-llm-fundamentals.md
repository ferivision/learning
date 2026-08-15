# Phase 01 — LLM Fundamentals

> Bagian dari [AI Engineering / Agent Roadmap](../README.md)

---

## 1. LLM Basics

### Apa itu?
LLM (Large Language Model) adalah model statistik raksasa yang dilatih buat memprediksi **token berikutnya** berdasarkan konteks yang sudah ada. Bukan database yang "nyari jawaban", bukan juga program yang punya aturan if-else — dia model probabilistik: dikasih urutan teks, dia hitung "kira-kira token apa yang paling mungkin muncul selanjutnya?", terus ulangi proses itu token demi token sampai selesai.

Beberapa istilah dasar yang harus lu kenal sebelum lanjut ke phase lain:
- **Token** — unit terkecil teks yang diproses model (bukan selalu satu kata utuh, bisa subword atau bahkan bagian dari kata). Detail lebih dalam ada di topik 4.
- **Context window** — total jumlah token (input + output digabung) yang bisa "dilihat" model dalam satu request. Kalau history percakapan kepanjangan, bagian terlama bisa "hilang" dari pandangan model.
- **Parameters** — angka/"bobot" di dalam neural network model itu sendiri (model dengan lebih banyak parameter biasanya — tapi gak selalu — lebih capable). Ini beda konsep sama "parameters" API kayak `temperature`.
- **Training vs Inference** — training adalah proses melatih model (mahal, dilakukan sekali oleh provider model kayak OpenAI/Anthropic — detail di topik 3). Inference adalah proses "memakai" model yang sudah jadi buat generate jawaban (ini yang lu lakuin tiap kali manggil API).

### Kenapa dibutuhkan?
Karena hampir semua yang bakal lu bangun di phase-phase selanjutnya (prompting, RAG, agent, tool calling) itu cuma lapisan di atas satu operasi dasar: manggil LLM buat generate teks. Kalau lu gak paham gimana LLM menghasilkan output-nya — kenapa kadang jawabannya beda padahal pertanyaan sama, kenapa ada batas panjang percakapan, kenapa ada parameter `temperature` — lu bakal susah debug perilaku yang aneh atau nge-tune behavior sesuai kebutuhan production. Ini fondasi, bukan detail yang bisa dilewatin.

### Cara Kerja
Alur dasarnya:
```
[System Message] + [User Message] → LLM → Token₁ → Token₂ → Token₃ → ... → Response
```

Tiga role pesan yang penting dipahami:
- **system** — instruksi "aturan main" buat model (persona, batasan, gaya jawaban). Contoh: "Kamu adalah asisten customer support yang ramah dan ringkas."
- **user** — input dari pengguna.
- **assistant** — jawaban dari model (juga dipakai buat merepresentasikan history jawaban sebelumnya kalau percakapan berlanjut).

Model men-generate response **satu token pada satu waktu**: di tiap langkah, model menghitung distribusi probabilitas atas seluruh kemungkinan token berikutnya, lalu memilih salah satu token berdasarkan probabilitas itu. Dua parameter yang paling sering dipakai buat mengontrol proses pemilihan ini:
- **temperature** — mengatur seberapa "berani" model milih token yang probabilitasnya rendah. `temperature=0` → hampir selalu pilih token paling mungkin (jawaban lebih konsisten/deterministic). `temperature` tinggi (misal 1.0+) → lebih random/kreatif, tapi juga lebih gampang ngasih jawaban yang gak konsisten dari run ke run.
- **top-p** (nucleus sampling) — daripada mempertimbangkan semua kemungkinan token, model cuma mempertimbangkan sekumpulan token teratas yang probabilitas kumulatifnya mencapai `p`. `top_p=0.1` misalnya, cuma mempertimbangkan token-token yang totalnya mencakup 10% probabilitas teratas — hasilnya jawaban lebih fokus/gak melebar ke pilihan kata yang aneh.

### Contoh Kode — Python
Ini adalah pemanggilan LLM API pertama di seri ini — jadi kita bikin sesederhana mungkin. Kode di bawah manggil OpenAI API buat jawab pertanyaan customer, meniru kasus SupportPilot yang paling dasar: kasih instruksi lewat system message, kasih pertanyaan user lewat user message, terus print jawabannya.

Penjelasan tiap baris sebelum baca kodenya:
- `from openai import OpenAI` — import class client resmi dari SDK OpenAI.
- `client = OpenAI()` — bikin instance client. Secara default, SDK ini otomatis baca API key dari environment variable `OPENAI_API_KEY` (gak perlu ditulis manual di kode — praktik yang baik biar secret gak ke-commit ke git).
- `client.chat.completions.create(...)` — ini pemanggilan API yang sebenarnya, mengirim request ke server OpenAI dan nunggu response-nya (synchronous, blocking).
- `model="gpt-4o-mini"` — nama model yang mau dipakai (model kecil & murah, cocok buat contoh kayak gini — detail pemilihan model ada di topik 5).
- `messages=[...]` — daftar pesan percakapan, urutannya penting: system dulu (aturan main), baru user (pertanyaan).
- `temperature=0.3` — dibikin rendah biar jawaban customer support lebih konsisten dan gak "ngarang-ngarang".
- `response.choices[0].message.content` — struktur response API-nya berupa daftar `choices` (biasanya cuma ada 1 kalau `n` default), dan jawaban teksnya ada di `.message.content`.

```python
from openai import OpenAI

client = OpenAI()  # baca API key dari environment variable OPENAI_API_KEY

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {
            "role": "system",
            "content": "Kamu adalah asisten customer support yang ramah dan ringkas.",
        },
        {
            "role": "user",
            "content": "Kenapa order saya belum sampai padahal sudah 5 hari?",
        },
    ],
    temperature=0.3,
)

print(response.choices[0].message.content)
```

### Trade-off & Pitfall
- **Temperature tinggi = kreatif tapi kurang predictable.** Kalau lu butuh output yang konsisten (misal buat structured output di Phase 2), jangan pakai temperature tinggi.
- **Non-determinism tetap ada walau `temperature=0`.** Karena infrastruktur inference provider (batching, floating-point non-determinism di GPU, dll), jawaban masih bisa sedikit beda antar request meski parameternya identik. Jangan asumsikan `temperature=0` = 100% reproducible.
- **Context window terbatas.** Semua history percakapan + system prompt + jawaban harus muat dalam batas token model. Kelewatan → request bisa error atau history lama otomatis "kepotong" tergantung implementasi.
- **Biaya dihitung per token** (input maupun output), bukan per request atau per karakter. Prompt yang panjang = biaya lebih mahal per call.

### Kapan Dipakai
- Pakai **temperature rendah (0–0.3)** buat task yang butuh konsistensi: klasifikasi, ekstraksi data, jawaban customer support yang harus akurat.
- Pakai **temperature lebih tinggi (0.7–1.0)** buat task kreatif: brainstorming, penulisan konten, variasi respons.
- Selalu isi **system message** buat set persona dan batasan — jangan andalkan user message doang buat "mendidik" perilaku model tiap request.
- **top_p** biasanya cukup dibiarkan default (1.0) kalau lu udah atur `temperature` — jarang perlu tuning keduanya bersamaan.

### Sering Ditanya Saat Interview
- **Apa bedanya `temperature` dan `top_p`?** — `temperature` mengatur seberapa "tajam" atau "rata" distribusi probabilitas token (makin tinggi makin rata/random). `top_p` membatasi jumlah kandidat token yang dipertimbangkan berdasarkan probabilitas kumulatif. Keduanya bisa dipakai bareng, tapi biasanya cukup tuning salah satu.
- **Kenapa LLM kadang ngasih jawaban beda walau pertanyaan sama persis?** — karena proses sampling token itu probabilistik (kecuali di-set benar-benar deterministic), dan bahkan pada `temperature=0` masih ada sedikit non-determinism dari sisi infrastruktur.
- **Apa itu context window dan kenapa penting?** — batas total token (input+output) yang bisa diproses model dalam satu request; penting karena membatasi berapa banyak history/dokumen yang bisa dikasih ke model sekaligus.
- **Apa beda training dan inference?** — training itu proses melatih model (dilakukan provider, sangat mahal, sekali per model), inference itu proses "pakai" model yang sudah jadi buat generate jawaban (yang kita lakukan tiap kali manggil API).

---

## 2. Transformer Basics

### Apa itu?
Transformer adalah arsitektur neural network yang jadi fondasi hampir semua LLM modern (GPT, Claude, Gemini, Llama, dll), diperkenalkan lewat paper "Attention Is All You Need" (2017). Intinya, Transformer adalah cara buat memproses urutan teks (sequence) dengan mekanisme yang disebut **attention** — bukan lagi memproses kata satu-satu secara berurutan kayak arsitektur lama (RNN/LSTM).

### Kenapa dibutuhkan?
Sebelum Transformer, model bahasa (RNN/LSTM) memproses teks kata demi kata **secara berurutan** — kata ke-100 baru bisa diproses setelah kata ke-99 selesai. Ini punya dua masalah besar: (1) lambat karena gak bisa diparalelkan, dan (2) sering "lupa" konteks dari kata-kata yang jauh di awal kalimat (long-range dependency problem). Transformer menyelesaikan keduanya lewat **self-attention**, yang memungkinkan tiap token "melihat" semua token lain sekaligus secara paralel — inilah yang bikin training model raksasa jadi feasible secara komputasi.

### Cara Kerja
Alur pemrosesan teks secara garis besar:
```
Text → Tokens → Embeddings (+ Positional Encoding) → Transformer Layers → Next-Token Prediction
```

Penjelasan tiap tahap:
- **Tokens** — teks dipecah jadi unit-unit kecil (lihat topik 4).
- **Embeddings** — tiap token diubah jadi vector angka yang merepresentasikan makna semantiknya (konsep ini dibahas lebih dalam di Phase 3 — di sini cukup paham bahwa token diubah jadi angka dulu sebelum diproses).
- **Positional encoding** — karena self-attention memproses semua token "sekaligus" (bukan berurutan kayak RNN), model butuh cara buat tau **urutan** token. Positional encoding menambahkan informasi posisi ke tiap embedding, supaya "kucing makan ikan" gak keliatan sama kayak "ikan makan kucing".
- **Self-attention** — mekanisme inti Transformer. Tiap token menghitung seberapa "relevan" dirinya terhadap setiap token lain di dalam sequence yang sama, lalu menimbang (weight) informasi dari token-token relevan itu buat membentuk representasi baru. Contoh sederhana: dalam kalimat "Order dia belum sampai, tolong dicek statusnya", kata "nya" di "statusnya" perlu "attend" ke kata "order" biar model tau yang dimaksud status apa.

  ```
  Token:      Order   dia   belum   sampai   ,   tolong   dicek   status-nya
  Attention:    ▲───────────────────────────────────────────────────┘
              (token "status-nya" memberi bobot atensi tinggi ke "Order")
  ```

- **Encoder vs Decoder** — Transformer awalnya didesain dengan dua bagian:
  - **Encoder** — membaca seluruh input sekaligus secara **bidirectional** (tiap token bisa "lihat" token sebelum dan sesudahnya). Cocok buat tugas yang butuh pemahaman utuh atas teks, misalnya model embedding.
  - **Decoder** — men-generate output token demi token, dan tiap token **hanya boleh** "melihat" token-token sebelumnya, gak boleh "curi lihat" token yang belum di-generate. Ini disebut **causal masking**.
  - LLM modern seperti GPT-style model umumnya **decoder-only** — makanya disebut juga **causal language model**: modelnya secara arsitektur memang dibatasi supaya prediksi token berikutnya cuma boleh bergantung pada token-token sebelumnya, persis seperti gimana dia dipakai buat generate teks satu arah.

### Trade-off & Pitfall
- **Biaya komputasi self-attention itu kuadratik** terhadap panjang sequence (makin panjang context, makin berat/mahal secara komputasi) — ini salah satu alasan kenapa context window gak bisa "sebesar apapun" secara gratis.
- **Miskonsepsi umum:** attention **bukan** berarti model "mengerti" makna kayak manusia. Attention cuma mekanisme matematis buat menimbang relevansi antar token berdasarkan pola statistik yang dipelajari saat training — hasilnya kelihatan seperti "pemahaman", tapi itu emergent behavior dari skala, bukan kesadaran (lihat topik 3).
- **Decoder-only model gak natural buat tugas yang butuh pemahaman dua arah** (misal butuh tau kata di akhir kalimat buat pahami kata di awal) — untuk itu arsitektur encoder (atau encoder-decoder) biasanya lebih cocok.

### Kapan Dipakai
Ini konsep arsitektur, bukan sesuatu yang lu "pilih pakai atau nggak" — tapi paham ini berguna waktu: (1) milih jenis model buat kebutuhan tertentu (model embedding biasanya berbasis encoder, model chat/generasi berbasis decoder-only — relevan di Phase 3 & 5), dan (2) debugging isu terkait context length/cost yang berakar dari cara kerja self-attention.

### Sering Ditanya Saat Interview
- **Apa itu self-attention?** — mekanisme di mana tiap token menghitung relevansinya terhadap token lain dalam sequence yang sama, lalu menimbang informasi dari token-token relevan itu.
- **Apa beda encoder dan decoder model?** — encoder membaca input secara bidirectional (lihat semua token sekaligus), decoder generate output secara causal (cuma boleh lihat token sebelumnya).
- **Apa itu causal language model?** — model yang dibatasi arsitekturnya supaya prediksi token berikutnya cuma bergantung pada token-token sebelumnya, bukan token yang akan datang.
- **Kenapa Transformer lebih baik dari RNN/LSTM untuk LLM?** — karena self-attention bisa diproses paralel (bukan sekuensial) dan lebih baik menangkap long-range dependency, sehingga training model raksasa jadi feasible.

---

## 3. Bagaimana LLM Diciptakan (Training Pipeline)

### Apa itu?
Ini jawaban buat pertanyaan "LLM itu dibuat gimana sih dari nol?" — dipahami sebagai satu pipeline besar dengan 6 tahap:
```
1. Data Collection → 2. Tokenization → 3. Pretraining
→ 4. Base Model → 5. Supervised Fine-Tuning (SFT) → 6. RLHF/DPO → Model Final
```

### Kenapa dibutuhkan?
Paham pipeline ini penting buat ngerti **kenapa base model beda perilakunya** dari model yang udah di-"chat-tuning", dan kenapa **fine-tuning** (Phase 16) itu tujuannya beda dari **RAG** (Phase 4) — dua hal yang sering ketuker padahal menyelesaikan masalah yang berbeda (fine-tuning mengubah *perilaku* model, RAG memberi *pengetahuan* tambahan tanpa mengubah model-nya sama sekali).

### Cara Kerja
Enam tahap secara berurutan:

**1. Data Collection** — scrape triliunan kata dari internet, buku, kode, dll. Ini teks mentah, bukan "kurikulum" yang disusun manual oleh manusia.

**2. Tokenization** — teks dipecah jadi token (unit kecil yang direpresentasikan sebagai angka), karena neural network cuma bisa memproses angka, bukan teks mentah.

**3. Pretraining** — jantung dari proses ini. Model (dengan arsitektur Transformer, lihat topik 2) dikasih potongan teks dan disuruh nebak "token berikutnya apa?", diulang triliunan kali. Tiap kali tebakannya salah, miliaran parameter di dalam model digeser sedikit lewat **gradient descent** biar next time tebakannya lebih akurat. Gak ada "pengertian sadar" di sini — murni pembelajaran pola statistik bahasa dalam skala masif. **Attention mechanism** (topik 2) berperan penting di sini: tiap token "melihat" dan menimbang relevansi token lain buat prediksi saat ini.

**4. Base Model** — hasil dari pretraining. Model ini jago nebak lanjutan teks, tapi belum tentu jago "menjawab" pertanyaan — ditanya sesuatu, base model bisa aja malah nerusin jadi bentuk teks lain (misal jadi soal ujian tambahan), karena yang dia pelajari cuma "nebak kelanjutan teks", bukan "jawab pertanyaan user".

**5. Supervised Fine-Tuning (SFT)** — model dilatih ulang pakai dataset percakapan tanya-jawab berkualitas tinggi (jumlahnya jauh lebih sedikit dibanding data pretraining) yang ditulis oleh manusia, biar model belajar format dan gaya menjawab yang membantu (helpful assistant behavior).

**6. RLHF / DPO (Reinforcement Learning from Human Feedback / Direct Preference Optimization)** — manusia memberi rating ke beberapa kandidat jawaban model (mana yang lebih disukai dibanding yang lain), lalu model disesuaikan lagi supaya condong menghasilkan jawaban yang lebih disukai manusia. Ini juga tahap di mana **safety/refusal behavior** dibentuk (misal model belajar menolak permintaan berbahaya).

**Miskonsepsi yang sering muncul:** LLM tidak "mengerti" secara sadar seperti manusia — ini model statistik raksasa yang, karena skalanya begitu besar, memunculkan kemampuan yang *terlihat* seperti reasoning (disebut *emergent capability*). Ini bukan kesadaran atau pemahaman dalam artian manusiawi, tapi pola statistik bahasa yang dipelajari dalam skala yang sangat masif.

### Trade-off & Pitfall
- **Pretraining sangat mahal** — butuh compute (ribuan GPU/TPU), waktu (berminggu-minggu hingga berbulan-bulan), dan data dalam skala triliunan token. Ini alasan kenapa hampir gak pernah masuk akal buat sebuah tim (kecuali provider model besar) melatih LLM dari nol.
- **Base model itu "mentah" dan gak aman dipakai langsung** — perilakunya belum di-alignment, bisa menghasilkan output yang gak diinginkan atau berbahaya sebelum melalui SFT dan RLHF/DPO.
- **SFT dan RLHF bisa memasukkan bias** — tergantung siapa yang menulis data SFT dan siapa yang memberi rating di RLHF, model bisa condong ke preferensi/nilai tertentu, atau jadi terlalu sering menolak (over-refusal) permintaan yang sebenarnya wajar.

### Kapan Dipakai
Paham pipeline ini relevan terutama buat dua situasi: (1) menjelaskan konsep ini di interview, dan (2) memutuskan pendekatan yang tepat kalau mau mengubah perilaku model — apakah butuh **fine-tuning** dari base model provider (Phase 16, jarang perlu buat kebanyakan kasus backend/applied AI), atau cukup **RAG** (Phase 4, jauh lebih umum) buat nambah pengetahuan tanpa melatih ulang model.

### Sering Ditanya Saat Interview
- **LLM itu dibuat gimana dari nol?** — enam tahap: data collection, tokenization, pretraining, base model, SFT, lalu RLHF/DPO.
- **Apa beda base model dan chat model (misal "gpt-4o" vs versi base-nya)?** — base model cuma jago "nerusin teks", belum tentu jago menjawab pertanyaan secara membantu; chat model sudah melalui SFT dan RLHF/DPO supaya perilakunya cocok jadi asisten yang menjawab pertanyaan.
- **Apa itu RLHF?** — tahap training di mana manusia me-rating kandidat jawaban model, lalu model disesuaikan supaya condong ke jawaban yang lebih disukai manusia.
- **Kenapa fine-tuning beda dari RAG?** — fine-tuning mengubah *perilaku/parameter* model lewat training tambahan, sedangkan RAG memberi *pengetahuan* eksternal ke model lewat context saat inference, tanpa mengubah model itu sendiri sama sekali.

---

## 4. Tokens & Context Window

### Apa itu?
**Token** adalah unit terkecil teks yang diproses model — bukan selalu satu kata utuh, bisa berupa subword, potongan kata, atau bahkan satu karakter, tergantung tokenizer yang dipakai model tersebut. **Context window** adalah total jumlah token (gabungan input + output) yang bisa diproses model dalam satu request — ini adalah "batas memori aktif" model buat satu percakapan.

### Kenapa dibutuhkan?
Ini konsep praktis yang langsung berdampak ke production: estimasi **biaya** (karena API di-charge per token, bukan per kata atau karakter), mencegah **error kepotong** kalau history percakapan kelewat panjang, dan memahami kenapa chatbot dengan percakapan panjang (misal fitur `Conversation` di SupportPilot) butuh strategi khusus supaya gak "lupa" instruksi awal atau kehabisan context window.

### Cara Kerja
Sebuah **tokenizer** memecah teks jadi token menggunakan algoritma seperti Byte-Pair Encoding (BPE) — teks umum sering jadi satu token, sementara kata yang jarang/panjang bisa dipecah jadi beberapa token. Tiap keluarga model (GPT, Claude, dll) punya tokenizer/vocabulary sendiri, jadi jumlah token buat teks yang sama bisa beda antar model.

```
More context → more information → higher cost → potentially slower → kualitas context bisa menurun
```

Yang perlu diperhatikan soal context window:
- **Token dihitung dari input DAN output digabung**, bukan cuma salah satunya.
- Dalam percakapan panjang (banyak turn), **total token history bertambah terus** tiap kali ada balasan baru ditambahkan — kalau gak dikelola, lama-lama mendekati atau melewati batas context window model.
- Kalau limit terlewati, tergantung implementasi: request bisa error, atau history lama harus dipotong/diringkas dulu (strategi lebih detail ada di Phase 7 — Agent Memory & Context Engineering).

### Contoh Kode — Python
Sebelum ke kode, kenalan dulu sama `tiktoken` — library resmi dari OpenAI buat menghitung berapa token yang dihasilkan dari sebuah teks, **tanpa perlu manggil API** (jadi bisa dipakai buat estimasi cepat sebelum kirim request beneran). Fungsinya sederhana: kasih teks, dia balikin daftar angka (token id), dan kita tinggal hitung panjangnya.

Contoh di bawah mensimulasikan history percakapan SupportPilot yang terus bertambah, lalu menghitung total token-nya dan mengecek apakah sudah mendekati batas context window:

```python
import tiktoken

# encoding_for_model otomatis pilih tokenizer yang sesuai buat model ini
encoding = tiktoken.encoding_for_model("gpt-4o-mini")


def count_tokens(text: str) -> int:
    # encode() mengubah teks jadi daftar token id; panjang daftarnya = jumlah token
    return len(encoding.encode(text))


conversation_history = [
    {"role": "system", "content": "Kamu adalah asisten customer support SupportPilot."},
    {"role": "user", "content": "Order saya #12345 belum sampai padahal sudah 5 hari."},
    {"role": "assistant", "content": "Baik, saya cek dulu status pengiriman order #12345 ya."},
    # bayangkan ratusan turn lagi menumpuk di sini seiring percakapan berlanjut...
]

total_tokens = sum(count_tokens(msg["content"]) for msg in conversation_history)
context_window_limit = 128_000  # contoh: batas context window untuk gpt-4o-mini

print(f"Total token dalam history saat ini: {total_tokens}")
print(f"Sisa kapasitas context window: {context_window_limit - total_tokens}")

if total_tokens > context_window_limit * 0.8:
    print("Peringatan: history percakapan sudah mendekati batas context window!")
```

### Trade-off & Pitfall
- **Token count ≠ jumlah kata ≠ jumlah karakter.** Estimasi ngasal (misal "1000 kata ≈ 1000 token") sering meleset — kata yang jarang dipakai bisa dipecah jadi beberapa token, sementara kata umum bisa jadi satu token.
- **Context lebih panjang bukan selalu lebih baik.** Selain lebih mahal dan lebih lambat, ada fenomena "lost in the middle" — model cenderung kurang memperhatikan informasi yang terkubur di tengah-tengah context yang sangat panjang, dibanding info di awal atau akhir.
- **Tokenizer beda per keluarga model** — jumlah token buat teks yang sama bisa beda kalau lu ganti provider model (misal dari OpenAI ke model lain), jadi jangan hardcode asumsi token count lintas provider.

### Kapan Dipakai
- Hitung token count **sebelum** kirim request kalau lu perlu estimasi biaya atau mau memastikan gak melebihi limit.
- Kalau history percakapan makin panjang dan mendekati batas context window, pertimbangkan strategi **compaction/summarization** atau **sliding window** (dibahas lebih detail di Phase 7 — Agent Memory).
- Kalau kebutuhannya "cari info relevan dari kumpulan dokumen besar" (bukan sekadar history percakapan), itu tandanya lu butuh **RAG** (Phase 3 & 4), bukan sekadar menjejalkan semua teks ke context.

### Sering Ditanya Saat Interview
- **Apa itu token, dan kenapa bukan sama dengan kata?** — token adalah unit terkecil teks yang diproses model (bisa subword), sehingga jumlahnya beda dari jumlah kata; kata yang jarang muncul bisa dipecah jadi beberapa token.
- **Apa itu context window, dan kenapa penting buat cost?** — batas total token (input+output) yang bisa diproses model per request; makin banyak token yang dikirim/dihasilkan, makin mahal biayanya (charge per token).
- **Gimana cara handle percakapan yang kelebihan context window?** — strategi umum: summarization/compaction history lama, sliding window (buang turn terlama), atau selective retrieval (ambil bagian relevan aja, mirip pola RAG).
- **Kenapa context yang terlalu panjang bisa menurunkan kualitas jawaban?** — fenomena "lost in the middle": model cenderung kurang memperhatikan informasi yang ada di tengah context yang sangat panjang.

---

## 5. Model Selection

### Apa itu?
Model selection adalah proses memilih model LLM mana yang paling cocok buat sebuah use case tertentu — bukan sekadar "pakai yang paling canggih/mahal", tapi mempertimbangkan trade-off antara kualitas, kecepatan, biaya, dan kebutuhan teknis lainnya.

### Kenapa dibutuhkan?
Karena model termahal/paling canggih **belum tentu pilihan terbaik** buat semua kasus. Task klasifikasi sederhana yang dipanggil jutaan kali gak butuh model reasoning paling kuat — itu cuma buang-buang biaya dan latency. Sebaliknya, task yang butuh reasoning kompleks kalau dipaksa pakai model murah/kecil bisa menghasilkan jawaban yang salah atau gak konsisten. Salah pilih model = salah satu penyebab paling umum sistem AI jadi mahal atau lambat tanpa alasan yang jelas.

### Cara Kerja
Pertimbangkan dimensi-dimensi berikut saat memilih model:

| Dimensi | Pertanyaan yang perlu dijawab |
|---|---|
| **Quality** | Seberapa akurat/relevan jawaban yang dibutuhkan? |
| **Latency** | Seberapa cepat respons harus muncul (real-time chat vs batch processing)? |
| **Cost** | Berapa volume request per hari/bulan, dan berapa budget per request? |
| **Context window** | Seberapa besar dokumen/history yang perlu diproses sekaligus? |
| **Reasoning ability** | Apakah task butuh penalaran multi-step (misal analisis kompleks), atau cukup task sederhana? |
| **Tool calling** | Apakah model perlu memanggil fungsi/tool eksternal (Phase 2, topik 8)? |
| **Structured output** | Apakah output harus mengikuti schema JSON yang ketat (Phase 2, topik 7)? |
| **Multimodal** | Apakah input berupa gambar/audio, bukan cuma teks? |
| **Privacy** | Apakah data yang diproses sensitif dan gak boleh keluar ke API pihak ketiga? |
| **Hosting requirement** | Apakah harus self-hosted (on-premise) karena kebijakan data, atau boleh pakai hosted API? |

Contoh pemetaan skenario ke kelas model:
```
Klasifikasi simpel        → model kecil/murah
Reasoning kompleks         → model reasoning kuat
Ekstraksi volume tinggi    → model cepat/murah
Data sensitif              → pertimbangkan self-hosted/private model
```

### Trade-off & Pitfall
- **Premature optimization ke arah yang salah** — banyak tim langsung pakai model termahal/paling canggih dari awal, padahal task-nya sebenarnya sederhana dan bisa dilayani model yang jauh lebih murah dan cepat.
- **Terlalu bergantung pada quirk satu model tertentu** — kalau prompt lu terlalu di-tuning buat gaya jawab satu model spesifik, migrasi ke model/provider lain nanti jadi lebih menyakitkan.
- **Trade-off privasi vs kemudahan** — model hosted API biasanya lebih gampang dipakai dan lebih capable, tapi data yang dikirim keluar ke infrastruktur pihak ketiga; self-hosted model lebih rumit dioperasikan tapi datanya gak pernah keluar.

### Kapan Dipakai
- **Task klasifikasi/ekstraksi sederhana, volume tinggi** → pilih model kecil dan cepat (biaya rendah, latency rendah).
- **Reasoning kompleks / analisis multi-step** → pilih model dengan reasoning ability kuat, meski lebih mahal/lambat.
- **Butuh tool calling atau structured output yang reliable** → pastikan model yang dipilih memang punya kemampuan itu secara native (gak semua model punya dukungan tool calling yang sama baiknya).
- **Data sensitif (misal data pelanggan di SupportPilot yang berisi info pribadi)** → pertimbangkan model self-hosted/private, atau minimal provider dengan jaminan data tidak dipakai buat training ulang.

### Sering Ditanya Saat Interview
- **Gimana cara milih model buat use case tertentu?** — timbang quality, latency, cost, context window, reasoning ability, tool calling, structured output, multimodal, privacy, dan hosting requirement sesuai kebutuhan use case-nya, bukan asal pilih yang paling canggih.
- **Kapan pakai model kecil vs model besar?** — model kecil/murah buat task sederhana bervolume tinggi (klasifikasi, ekstraksi); model besar/reasoning kuat buat task yang butuh penalaran kompleks.
- **Apa pertimbangan kalau data yang diproses sensitif?** — pertimbangkan self-hosted atau private model, atau pastikan provider punya jaminan kontraktual soal data tidak dipakai untuk training.
- **Kenapa gak selalu pakai model paling canggih buat semua task?** — karena lebih mahal dan lebih lambat; buat task sederhana, itu cuma overhead tanpa manfaat kualitas yang signifikan.

---

**Selanjutnya:** [Phase 02 — Prompting & Structured Output](./phase-02-prompting-structured-output.md)
