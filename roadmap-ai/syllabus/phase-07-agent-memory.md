# Phase 07 — Agent Memory

> Bagian dari [AI Engineering / Agent Roadmap](../README.md)

---

## 29. Short-Term Memory

### Apa itu?
Short-term memory adalah memori yang cuma hidup selama **satu sesi percakapan aktif** — begitu customer bilang "nama saya John" di awal chat, terus beberapa pesan kemudian nanya "siapa nama saya?", agent harus bisa jawab "John" karena dia masih "inget" apa yang barusan terjadi DI SESI YANG SAMA. Begitu sesi itu selesai (browser ditutup, chat widget di-refresh, dst), short-term memory ini hilang — beda dari long-term memory (topik 30) yang memang didesain buat bertahan lintas sesi.

### Kenapa dibutuhkan?
`run_agent_loop` (Phase 6 topik 25) menerima `user_message: str` **tunggal** — begitu fungsi itu `return`, gak ada state apapun yang otomatis tersisa buat turn berikutnya. Tanpa short-term memory, tiap kali customer kirim pesan baru, SupportPilot bakal panggil `run_agent_loop` dari nol tanpa tau apa-apa soal pesan-pesan sebelumnya di percakapan yang sama — customer yang baru aja bilang namanya bakal ditanya ulang "siapa nama Anda?" di pesan berikutnya, yang jelas bikin pengalaman customer terasa aneh dan gak natural. Short-term memory menyelesaikan ini dengan menyimpan riwayat percakapan di level aplikasi (bukan di `run_agent_loop` itu sendiri), lalu menyertakannya lagi tiap kali turn baru datang.

### Cara Kerja
```
Turn 1: customer "nama saya John"
    → memory.add_message("user", "nama saya John")
    → run_agent_loop(client, riwayat + pesan ini, tools) → jawaban
    → memory.add_message("assistant", jawaban)

Turn 2: customer "siapa nama saya?"
    → memory.add_message("user", "siapa nama saya?")
    → run_agent_loop(client, RIWAYAT LENGKAP (termasuk turn 1) + pesan ini, tools)
    → jawaban sekarang bisa nyebut "John" karena riwayat turn 1 ikut disertakan
```
Kuncinya: `memory.get_history()` dipanggil ULANG tiap turn, dan hasilnya disertakan ke `run_agent_loop` — bukan cuma pesan customer yang paling baru saja.

### Contoh Kode — Python
`ConversationMemory` cuma pembungkus tipis di atas list biasa — sengaja simpel, karena tanggung jawabnya cuma satu: mencatat pesan secara berurutan dan mengembalikannya lagi saat diminta.
```python
class ConversationMemory:
    """
    Short-term memory: hidup selama proses aplikasi jalan & sesi percakapan
    masih aktif (disimpan di memory Python biasa di sini, buat contoh) — TIDAK
    dipersist ke database, jadi hilang begitu sesi/proses berakhir. Bandingkan
    dengan save_long_term_memory (topik 30) yang memang didesain buat bertahan.
    """

    def __init__(self):
        self._messages: list[dict] = []

    def add_message(self, role: str, content: str) -> None:
        self._messages.append({"role": role, "content": content})

    def get_history(self) -> list[dict]:
        # Kembalikan COPY-nya (bukan reference langsung ke self._messages),
        # supaya caller gak bisa gak sengaja mengubah isi memory dari luar
        return list(self._messages)
```

Karena `run_agent_loop` (Phase 6) cuma menerima `user_message: str` tunggal (bukan `messages: list[dict]`), riwayat dari `ConversationMemory` perlu diubah dulu jadi satu blok teks sebelum disertakan sebagai bagian dari `user_message`:
```python
def format_history_as_text(history: list[dict]) -> str:
    """
    Ubah riwayat percakapan (list of dict) jadi satu blok teks yang gampang
    dibaca model, dengan urutan turn tetap terjaga.
    """
    lines = []
    for msg in history:
        speaker = "Customer" if msg["role"] == "user" else "Agent"
        lines.append(f"{speaker}: {msg['content']}")
    return "\n".join(lines)


def chat_turn(
    client, memory: ConversationMemory, tools: list[dict], user_input: str
) -> str:
    """
    Satu putaran chat SupportPilot: catat pesan customer, sertakan SELURUH
    riwayat sejauh ini ke run_agent_loop (topik 25, Phase 6), lalu catat juga
    jawaban agent-nya supaya turn berikutnya bisa "inget" turn ini juga.
    """
    memory.add_message("user", user_input)

    riwayat_sejauh_ini = format_history_as_text(memory.get_history())
    prompt_dengan_riwayat = (
        f"Riwayat percakapan sejauh ini (urut dari yang paling lama):\n"
        f"{riwayat_sejauh_ini}\n\n"
        f"Balas pesan TERAKHIR dari customer di atas, dengan memperhatikan "
        f"seluruh riwayat sebelumnya kalau relevan."
    )

    jawaban = run_agent_loop(client, prompt_dengan_riwayat, tools, max_steps=5)
    memory.add_message("assistant", jawaban)
    return jawaban
```

Membuktikan klaim di "Apa itu?" — agent beneran "inget" nama customer dari turn sebelumnya:
```python
from openai import OpenAI

client = OpenAI()
memory = ConversationMemory()

jawaban_1 = chat_turn(client, memory, tools=[], user_input="Halo, nama saya John.")
print(jawaban_1)
# diharapkan: sapaan balik yang menyebut sudah mencatat nama "John"

jawaban_2 = chat_turn(client, memory, tools=[], user_input="Eh, siapa nama saya tadi?")
print(jawaban_2)
# diharapkan: jawaban menyebut "John" — model bisa jawab ini karena
# format_history_as_text(memory.get_history()) menyertakan turn pertama
# (yang berisi "nama saya John") ke dalam prompt turn kedua ini
```

### Trade-off & Pitfall
- **Riwayat yang disertakan ke `run_agent_loop` ikut menambah jumlah token per request** (Phase 1 topik 4) — makin panjang percakapan, makin mahal & makin lambat tiap turn berikutnya, karena SELURUH riwayat dikirim ulang tiap kali (bukan cuma pesan baru). Ini persis masalah yang diselesaikan `compact_context` di topik 32.
- **`run_agent_loop` menerima `user_message: str` tunggal, bukan `messages: list[dict]`** — jadi riwayat harus di-serialize jadi teks dulu (`format_history_as_text`) sebelum bisa disertakan; kalau nanti butuh riwayat benar-benar sebagai role message terpisah (system/user/assistant), signature `run_agent_loop` sendiri perlu diperluas buat menerima parameter riwayat opsional.
- **Short-term memory hilang kalau proses/sesi restart** — kalau `ConversationMemory` cuma disimpan di variable Python biasa (seperti contoh di atas) dan server-nya restart, seluruh riwayat sesi itu hilang. Buat production, `ConversationMemory` biasanya perlu dukungan penyimpanan sementara (misal Redis dengan TTL per sesi), bukan cuma in-process variable.
- **Jangan campur short-term dengan long-term memory** — short-term (topik ini) itu scoped ke SATU sesi aktif; kalau fakta yang muncul di sesi ini perlu diingat lintas sesi (misal preferensi customer), itu harus eksplisit disimpan lewat `save_long_term_memory` (topik 30) — gak otomatis "naik level" begitu aja.

### Kapan Dipakai
- Pakai `ConversationMemory` (atau pola serupa) di SETIAP fitur chat multi-turn SupportPilot — tanpa ini, agent bakal "amnesia" tiap pesan baru dan gak bisa merujuk balik ke apa yang barusan dibahas di sesi yang sama.
- Kalau percakapannya makin panjang dan riwayatnya mulai mendekati batas context window (Phase 1 topik 4), jangan terus-terusan kirim SELURUH riwayat mentah — lanjut ke `compact_context` (topik 32).
- Kalau ada fakta dari percakapan ini yang perlu diingat WALAU sesinya sudah berakhir, itu bukan tanggung jawab `ConversationMemory` — itu tanggung jawab `save_long_term_memory` (topik 30).

### Sering Ditanya Saat Interview
- **Apa itu short-term memory dalam konteks agent, dan gimana bedanya dengan long-term memory?** — short-term memory scoped ke SATU sesi percakapan aktif dan hilang begitu sesi berakhir; long-term memory (topik 30) memang didesain buat bertahan lintas sesi, biasanya disimpan permanen di database.
- **Kenapa `run_agent_loop` gak otomatis "inget" percakapan sebelumnya?** — karena signature-nya cuma menerima `user_message: str` tunggal per panggilan, dan tiap panggilan dimulai dari `messages` baru di dalam fungsi itu (Phase 6 topik 25) — riwayat harus disuntikkan dari luar lewat `ConversationMemory`, bukan otomatis tersimpan sendiri oleh `run_agent_loop`.
- **Apa risiko utama kalau riwayat percakapan terus disertakan penuh tiap turn?** — jumlah token per request terus bertambah (Phase 1 topik 4), yang berarti biaya dan latency ikut membengkak seiring percakapan makin panjang — ini yang dibahas solusinya di topik 32 (`compact_context`).
- **Kenapa `ConversationMemory` di contoh ini gak cocok langsung dipakai di production tanpa modifikasi?** — karena disimpan cuma sebagai variable Python biasa (in-process), jadi hilang kalau proses/server-nya restart; production butuh backing store sementara (misal Redis dengan TTL per sesi).

---

## 30. Long-Term Memory

### Apa itu?
Long-term memory adalah informasi tentang seorang customer yang **bertahan lintas sesi** — beda dari short-term memory (topik 29) yang cuma hidup selama satu percakapan aktif. Contohnya: preferensi cara dihubungi (email vs telepon), keputusan/kesepakatan yang pernah dibuat sebelumnya, atau fakta penting lain tentang customer itu yang tetap relevan biarpun sesi chat sebelumnya sudah lama ditutup.

### Kenapa dibutuhkan?
Tanpa long-term memory, tiap kali customer SupportPilot mulai sesi chat baru, agent-nya "amnesia total" — customer yang minggu lalu udah bilang "saya lebih suka dihubungi lewat email, jangan telpon" harus mengulang preferensi itu lagi dari nol tiap kali chat baru. Ini bukan cuma bikin pengalaman customer terasa buruk, tapi juga beresiko agent mengambil aksi yang bertentangan dengan preferensi/keputusan yang sudah pernah disepakati sebelumnya. Long-term memory menyelesaikan ini dengan menyimpan fakta-fakta penting itu secara permanen di database, terikat ke `customer_id` tertentu, supaya bisa diambil lagi di sesi manapun di masa depan.

### Cara Kerja
```
Percakapan mengungkap fakta penting
    → fact: str (mis. "Customer C-99 lebih suka dihubungi lewat email")
    → generate_embedding(fact) (Phase 3 topik 9)          -> vector 1536 dimensi
    → INSERT ke tabel customer_memories (fact + embedding + customer_id)
    → fakta ini sekarang bisa di-retrieve lagi kapan pun lewat retrieve_memories (topik 31)
```
Skema tabel `customer_memories` — sengaja mirip pola `knowledge_articles` (Phase 3 topik 11), tapi tiap barisnya terikat ke satu `customer_id` tertentu (bukan pengetahuan umum yang dibagi semua customer):
```sql
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE customer_memories (
    id SERIAL PRIMARY KEY,
    customer_id TEXT NOT NULL,
    fact TEXT NOT NULL,
    -- dimensi 1536 harus konsisten dengan generate_embedding (text-embedding-3-small,
    -- Phase 3 topik 9)
    embedding VECTOR(1536) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Index biasa buat mempercepat filter per customer (WHERE customer_id = ...)
CREATE INDEX customer_memories_customer_id_idx ON customer_memories (customer_id);

-- Index ANN buat mempercepat pencarian semantik (dipakai retrieve_memories, topik 31)
CREATE INDEX customer_memories_embedding_idx
ON customer_memories
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

### Contoh Kode — Python
```python
def save_long_term_memory(conn, customer_id: str, fact: str) -> None:
    """
    Simpan satu fakta tentang seorang customer secara PERMANEN (bertahan
    lintas sesi), di-embed dulu (Phase 3 topik 9-10) supaya nantinya bisa
    dicari berdasarkan makna lewat retrieve_memories (topik 31) — bukan cuma
    exact match teks.
    """
    embedding = generate_embedding(fact)
    embedding_literal = "[" + ",".join(str(x) for x in embedding) + "]"

    with conn.cursor() as cur:
        cur.execute(
            """
            INSERT INTO customer_memories (customer_id, fact, embedding)
            VALUES (%s, %s, %s::vector);
            """,
            (customer_id, fact, embedding_literal),
        )
    conn.commit()
```

Dipakai setelah sebuah percakapan SupportPilot mengungkap sesuatu yang layak diingat lintas sesi (di real production, keputusan "fakta MANA yang layak disimpan" biasanya datang dari LLM call terpisah yang mengekstrak fakta dari percakapan — di sini kita fokus ke bagian PENYIMPANANNYA, bukan ekstraksinya):
```python
import psycopg2

conn = psycopg2.connect(
    dbname="supportpilot",
    user="supportpilot_app",
    password="...",
    host="localhost",
)

# Percakapan barusan (pakai chat_turn dari topik 29) mengungkap preferensi customer
jawaban = chat_turn(
    client,
    memory,
    tools=[],
    user_input=(
        "Tolong ya kalau ada update soal tiket saya, hubungi lewat email aja, "
        "jangan telpon, saya jarang angkat telpon dari nomor gak dikenal."
    ),
)

# Fakta ini layak diingat lintas sesi -> simpan sebagai long-term memory
save_long_term_memory(
    conn,
    customer_id="C-99",
    fact="Customer C-99 lebih suka dihubungi lewat email daripada telepon.",
)
```

### Trade-off & Pitfall
- **Nyimpen fakta itu murah, tapi menentukan fakta MANA yang layak disimpan itu yang susah.** Kalau semua kalimat customer disimpan mentah-mentah sebagai "fakta", tabel `customer_memories` cepat penuh sama noise (basa-basi, keluhan sesaat yang gak relevan jangka panjang) — butuh kriteria/filter (biasanya lewat LLM call ekstraksi terpisah) sebelum sesuatu layak lewat `save_long_term_memory`.
- **Fakta bisa jadi usang (stale) atau bahkan kontradiktif seiring waktu** — misal customer dulu bilang "suka dihubungi lewat telepon", tapi bulan berikutnya bilang sebaliknya. Skema di atas gak otomatis menangani "fakta lama vs fakta baru mana yang berlaku" (masalah ini yang jadi kekuatan utama pendekatan temporal knowledge graph seperti Zep/Graphiti — dibahas di topik 33) — versi sederhana ini butuh logic tambahan (misal timestamp + hanya ambil yang terbaru) kalau butuh presisi itu.
- **Privasi & retensi data adalah pertimbangan wajib, bukan opsional** — fakta yang disimpan lintas sesi seringkali data pribadi customer; perlu kebijakan jelas soal berapa lama data ini boleh disimpan, siapa yang boleh akses, dan bagaimana cara menghapusnya kalau customer minta (lihat juga pertanyaan-pertanyaan di topik 31).
- **Biaya embedding + storage bertambah linear seiring jumlah fakta yang disimpan** — beda dari `knowledge_articles` (Phase 3) yang jumlahnya relatif tetap, `customer_memories` bisa terus bertambah seiring jumlah customer & percakapan, jadi butuh strategi retensi/pembersihan (lihat topik 31).

### Kapan Dipakai
- Pakai `save_long_term_memory` buat fakta yang punya nilai jangka panjang: preferensi customer, keputusan/kesepakatan yang sudah dibuat, detail penting yang bakal relevan di sesi-sesi berikutnya.
- **Jangan** pakai buat informasi yang cuma relevan di sesi saat ini (misal "customer lagi bingung soal tiket T-555 ini") — itu cukup di `ConversationMemory` (topik 29), gak perlu dipermanenkan.
- Pertimbangkan proses ekstraksi fakta otomatis (LLM call terpisah yang menganalisis percakapan dan memutuskan "apakah ada fakta yang layak disimpan?") daripada memanggil `save_long_term_memory` manual untuk tiap pesan customer.

### Sering Ditanya Saat Interview
- **Apa itu long-term memory, dan kenapa perlu disimpan di database (bukan cuma di context percakapan)?** — informasi yang harus bertahan lintas sesi (preferensi, keputusan, fakta penting customer); disimpan di database (bukan cuma context percakapan aktif) supaya tetap ada walau sesi sebelumnya sudah lama berakhir atau proses aplikasi sudah restart.
- **Kenapa `save_long_term_memory` meng-embed `fact` sebelum menyimpannya?** — supaya nantinya fakta itu bisa dicari berdasarkan KEMIRIPAN MAKNA lewat `retrieve_memories` (topik 31), bukan cuma exact-match teks — customer bisa nanya dengan kalimat berbeda tapi maksud yang sama.
- **Apa risiko utama long-term memory kalau gak ada kriteria fakta mana yang disimpan?** — tabel jadi penuh noise dari basa-basi/keluhan sesaat yang gak relevan jangka panjang, bikin retrieval nantinya kurang presisi dan storage membengkak gak perlu.
- **Apa yang harus dipikirkan soal privasi saat menyimpan long-term memory customer?** — kebijakan retensi (berapa lama data disimpan), kontrol akses (siapa yang boleh baca), dan mekanisme penghapusan kalau customer minta datanya dihapus — ini bukan detail teknis tambahan, tapi kewajiban yang harus ada sejak awal.

---

## 31. Memory Retrieval

### Apa itu?
Memory retrieval adalah proses mengambil HANYA fakta-fakta yang relevan tentang seorang customer tertentu — berdasarkan apa yang sedang dia tanyakan SAAT INI — daripada men-dump SEMUA fakta yang pernah tersimpan tentang customer itu ke context sekaligus. Pola ini mirip banget sama retrieval di RAG (Phase 4), bedanya di sini yang dicari bukan pengetahuan umum (artikel help-center), tapi fakta yang di-scope ke SATU customer spesifik.

### Kenapa dibutuhkan?
Kalau seorang customer sudah punya puluhan/ratusan fakta tersimpan di `customer_memories` (topik 30) setelah bertahun-tahun jadi pelanggan, men-dump SEMUA fakta itu ke context tiap kali dia chat itu boros token (Phase 1 topik 4) dan justru bisa menurunkan kualitas jawaban — sebagian besar fakta itu gak relevan sama pertanyaan yang sedang diajukan saat itu, dan fenomena "lost in the middle" (Phase 1 topik 4) bikin fakta yang BENERAN relevan malah "tenggelam" di antara fakta-fakta gak relevan lainnya. Sebelum sampai ke pertanyaan teknis "gimana cara retrieve-nya", ada pertanyaan yang lebih mendasar yang wajib dijawab dulu buat sistem memory manapun:
- **Apa yang harus diingat?** — gak semua ucapan customer layak jadi memory permanen (lihat topik 30).
- **Apa yang harus dilupakan?** — fakta yang sudah usang/gak relevan lagi harus bisa dihapus atau ditandai gak berlaku.
- **Berapa lama disimpan?** — ada kebijakan retensi yang jelas, bukan disimpan selamanya tanpa batas.
- **Siapa yang boleh akses?** — fakta customer A gak boleh ke-retrieve buat sesi customer B (lihat WHERE `customer_id` di kode di bawah).
- **Apakah infonya masih akurat?** — fakta lama yang mungkin sudah berubah butuh mekanisme buat di-update/dikoreksi, bukan dianggap selalu berlaku selamanya.

### Cara Kerja
```
Pertanyaan customer saat ini
    → generate_embedding(query) (Phase 3 topik 9)          -> query vector
    → SQL: WHERE customer_id = ini SAJA
           ORDER BY embedding <=> query_vector
           LIMIT top_k
    → top_k fakta paling relevan MILIK customer ini (bukan customer lain)
```
Bedanya dengan `search_knowledge_base`/`retrieve_relevant_chunks` (Phase 3 & 4): di sana pencarian dilakukan ke SELURUH baris tabel (pengetahuan umum, dibagi semua customer); di sini WAJIB ada filter `WHERE customer_id = %s` supaya fakta yang di-retrieve terbatas ke SATU customer tertentu — kebocoran fakta antar customer di sini adalah bug keamanan/privasi, bukan cuma bug relevansi.

### Contoh Kode — Python
```python
def retrieve_memories(
    conn, customer_id: str, query: str, top_k: int = 3
) -> list[str]:
    """
    Cari top_k fakta paling relevan tentang SATU customer tertentu, berdasarkan
    makna `query` (pertanyaan/konteks customer saat ini) — mirip
    retrieve_relevant_chunks (Phase 4), tapi di-scope ketat ke satu
    customer_id lewat WHERE, gak dicari lintas semua customer.
    """
    query_embedding = generate_embedding(query)
    embedding_literal = "[" + ",".join(str(x) for x in query_embedding) + "]"

    with conn.cursor() as cur:
        cur.execute(
            """
            SELECT fact
            FROM customer_memories
            WHERE customer_id = %s
            ORDER BY embedding <=> %s::vector
            LIMIT %s;
            """,
            (customer_id, embedding_literal, top_k),
        )
        rows = cur.fetchall()

    # rows berupa list of tuple (satu kolom, "fact") -> ambil elemen pertama tiap baris
    return [row[0] for row in rows]
```

Dipakai buat menyuntikkan fakta yang relevan ke prompt sebelum manggil `run_agent_loop`, tanpa men-dump semua fakta customer itu sekaligus:
```python
pertanyaan_saat_ini = "Kalau tiket saya belum selesai, gimana cara update-nya nanti?"

memories = retrieve_memories(
    conn, customer_id="C-99", query=pertanyaan_saat_ini, top_k=3
)
# diharapkan: ["Customer C-99 lebih suka dihubungi lewat email daripada telepon."]
# (fakta yang disimpan di topik 30) — relevan karena query-nya soal "cara update"

prompt_dengan_memory = (
    f"Fakta yang diketahui tentang customer ini: {'; '.join(memories)}\n\n"
    f"Pertanyaan customer: {pertanyaan_saat_ini}"
)

jawaban = run_agent_loop(client, prompt_dengan_memory, tools=[], max_steps=5)
print(jawaban)
# diharapkan: jawaban menyebutkan bakal update lewat EMAIL (bukan telepon),
# karena fakta relevan itu sudah disuntikkan ke prompt lewat retrieve_memories
```

### Trade-off & Pitfall
- **`top_k` yang kekecilan bisa membuang fakta yang sebenarnya relevan**, sementara `top_k` yang kegedean balik lagi ke masalah "dump semua memory" yang justru mau dihindari — nilai `top_k` biasanya perlu di-tuning berdasarkan data nyata (mirip pertimbangan `top_k` di RAG, Phase 4 topik 15).
- **Lupa filter `WHERE customer_id`** adalah bug paling berbahaya di fungsi ini — kalau sampai lolos, fakta milik customer lain bisa ke-retrieve dan bocor ke percakapan customer yang berbeda; ini WAJIB divalidasi di setiap tempat yang manggil `retrieve_memories`.
- **Fakta yang kontradiktif atau sudah usang tetap bisa ke-retrieve** kalau gak ada mekanisme "apakah infonya masih akurat" (lihat pertanyaan di atas) — retrieval semata-mata berdasarkan kemiripan makna, dia gak otomatis tau fakta mana yang paling baru/valid kalau ada dua fakta yang saling bertentangan tersimpan.
- **Sama seperti retrieval biasa (Phase 4 topik 17), retrieval failure tetap mungkin terjadi** — fakta yang relevan gak selalu ke-retrieve kalau kalimat query-nya jauh secara semantik dari cara fakta itu ditulis saat disimpan.

### Kapan Dipakai
- Pakai `retrieve_memories` SETIAP kali mau menyuntikkan fakta customer ke prompt sebelum `run_agent_loop` dipanggil — jangan pernah query SEMUA baris `customer_memories` milik satu customer tanpa filter relevansi (topik 30 & 31 sengaja dipisah: simpan semua yang relevan, tapi retrieve cuma yang relevan SAAT INI).
- Pakai bareng `ConversationMemory` (topik 29) — riwayat sesi aktif (short-term) dan fakta relevan lintas sesi (long-term, lewat retrieval ini) sama-sama disuntikkan ke prompt yang sama sebelum dikirim ke `run_agent_loop`.
- Kalau volume `customer_memories` per customer sudah sangat besar dan linear scan mulai lambat, pertimbangan index ANN (`ivfflat`/`hnsw`, Phase 3 topik 11) yang sudah dipasang di topik 30 jadi makin penting.

### Sering Ditanya Saat Interview
- **Kenapa gak dump semua memory customer ke context sekaligus?** — boros token (biaya & latency naik) dan bisa menurunkan kualitas jawaban lewat fenomena "lost in the middle" — fakta yang relevan bisa tenggelam di antara fakta yang gak relevan buat pertanyaan saat itu.
- **Apa 5 pertanyaan penting yang harus dijawab sebelum bangun sistem memory?** — apa yang harus diingat, apa yang harus dilupakan, berapa lama disimpan, siapa yang boleh akses, dan apakah infonya masih akurat.
- **Kenapa `retrieve_memories` WAJIB filter `customer_id`?** — supaya fakta milik satu customer gak pernah ke-retrieve dan bocor ke percakapan customer lain — ini isu keamanan/privasi, bukan cuma soal relevansi hasil.
- **Apa persamaan `retrieve_memories` dengan retrieval di RAG (Phase 4)?** — pola dasarnya sama (embed query, cari yang paling mirip secara makna, ambil top-k), bedanya scope-nya: RAG mencari pengetahuan umum yang dibagi semua customer, `retrieve_memories` di-scope ketat ke satu customer tertentu.

---

## 32. Context Engineering & Context Compaction

### Apa itu?
Context engineering adalah disiplin mengelola **seluruh isi context** yang dikirim ke LLM di setiap request — bukan cuma soal "bagaimana menyusun satu prompt yang bagus" (itu Prompt Engineering, Phase 2 topik 6), tapi soal bagaimana mengelola history percakapan yang TERUS BERTAMBAH selama sesi agent berjalan lama (banyak turn, banyak tool call, seperti riwayat dari `ConversationMemory` di topik 29). Context compaction adalah salah satu teknik intinya: begitu history sudah terlalu besar buat context window budget yang ditetapkan, ringkas bagian yang lama jadi satu summary, sambil tetap menyimpan turn-turn terbaru apa adanya.

### Kenapa dibutuhkan?
Prompt Engineering (Phase 2) menjawab pertanyaan "gimana cara menyusun SATU prompt yang efektif untuk SATU request?" — itu masalah yang beda dari yang dihadapi di sini: sebuah sesi agent SupportPilot yang berjalan lama (misal customer chat panjang, atau agent yang melakukan banyak tool call berturut-turut di topik 25) bikin `messages` yang dikirim ke LLM TERUS BERTAMBAH tiap turn. Kalau dibiarkan tumbuh tanpa batas, dua masalah muncul: (1) biaya & latency membengkak linear seiring panjang percakapan (Phase 1 topik 4, token dihitung dari input+output), dan (2) begitu total token mendekati/melebihi context window model, request bisa error atau — kalau history dipotong sembarangan — informasi penting dari awal percakapan (misal instruksi system, atau fakta yang disebut customer di awal) bisa hilang begitu saja. Context compaction menyelesaikan ini dengan strategi yang lebih pintar dari sekadar "buang yang lama": ringkas yang lama jadi summary singkat (tetap ada jejaknya), tapi simpan turn-turn terbaru utuh (karena itu yang paling relevan buat kelanjutan percakapan saat ini).

### Cara Kerja
Beberapa strategi context engineering yang saling melengkapi:
- **Context window budget** — tetapkan `max_tokens` sebagai batas aman (biasanya di bawah limit context window model sesungguhnya, nyisain ruang buat jawaban baru) — begitu history diperkirakan bakal melebihi batas ini, ambil tindakan (compaction).
- **Summarization/compaction** — ringkas pesan-pesan LAMA jadi satu pesan summary tunggal, supaya intinya tetap ada tanpa makan token sebanyak history mentahnya.
- **Sliding window** — pendekatan lebih sederhana: buang begitu saja turn paling lama, simpan cuma N turn paling baru (tanpa diringkas). Lebih murah secara komputasi dibanding summarization, tapi informasi dari turn yang dibuang benar-benar hilang, gak ada jejaknya sama sekali.
- **Selective retrieval** — daripada masukin SEMUA history, ambil cuma bagian yang relevan sama pertanyaan saat ini (pola yang sama persis dengan `retrieve_memories`, topik 31 — bedanya di sini yang di-retrieve adalah potongan history percakapan, bukan fakta customer yang tersimpan permanen).

`compact_context` di bawah ini menggabungkan pendekatan **summarization** (buat bagian lama) dengan **sliding window** (buat turn terbaru, disimpan verbatim) sekaligus:
```
estimasi total token messages
    kalau <= max_tokens -> gak perlu compaction, return messages apa adanya
    kalau > max_tokens:
        -> pisahkan system message (SELALU dipertahankan utuh, di posisi awal)
        -> dari non-system messages: ambil N_RECENT_TURNS_TO_KEEP pesan TERAKHIR,
           simpan VERBATIM (gak diringkas sama sekali)
        -> sisanya (yang lebih lama dari itu) diringkas jadi SATU pesan summary
        -> hasil akhir: [system messages] + [1 pesan summary] + [N pesan terbaru verbatim]
```

### Contoh Kode — Python
Estimasi token pakai `tiktoken`, sama seperti `count_tokens` di Phase 1 topik 4:
```python
import tiktoken

encoding = tiktoken.encoding_for_model("gpt-4o-mini")


def count_tokens(text: str) -> int:
    return len(encoding.encode(text))


def estimate_messages_tokens(messages: list[dict]) -> int:
    # Estimasi kasar: jumlah token dari SELURUH isi `content` tiap message
    # (di production sebenarnya ada overhead tambahan per message dari format
    # chat API, tapi estimasi ini cukup akurat buat keputusan "perlu compact atau tidak")
    return sum(count_tokens(msg["content"]) for msg in messages)


def summarize_messages(messages: list[dict]) -> str:
    """
    Ringkas sekumpulan message LAMA jadi satu paragraf singkat yang UKURANNYA
    TERBATAS (gak ikut membesar sebanding jumlah pesan yang diringkas -- itu
    properti wajib sebuah ringkasan, kalau enggak, "ringkasan" ini gak
    menghemat token apa-apa). Di production sungguhan, bagian ini biasanya
    manggil LLM (model kecil/murah, temperature rendah -- Phase 1 topik 5)
    yang diinstruksikan buat menghasilkan ringkasan singkat dari isi
    percakapan. Di sini dipakai heuristik sederhana (bukan panggilan API)
    supaya contoh tetap deterministic dan runnable tanpa API key -- tapi
    heuristiknya sengaja dibuat bounded-size, persis properti yang harus
    dipenuhi versi LLM-nya juga.
    """
    jumlah_user = sum(1 for m in messages if m["role"] == "user")
    jumlah_assistant = sum(1 for m in messages if m["role"] == "assistant")
    cuplikan_awal = messages[0]["content"][:80]
    cuplikan_akhir = messages[-1]["content"][:80]
    return (
        f"[Ringkasan {len(messages)} pesan sebelumnya: {jumlah_user} dari "
        f"customer, {jumlah_assistant} dari agent] Diawali dengan: "
        f"\"{cuplikan_awal}\" ... berakhir (sebelum turn terbaru) dengan: "
        f"\"{cuplikan_akhir}\""
    )
```

`compact_context` — implementasi intinya:
```python
def compact_context(messages: list[dict], max_tokens: int = 2000) -> list[dict]:
    """
    Kalau estimasi total token `messages` melebihi `max_tokens`, ringkas
    pesan-pesan TERLAMA jadi satu pesan system tunggal, sambil tetap menjaga
    beberapa turn TERBARU apa adanya (verbatim) -- supaya history gak overflow
    context window (Phase 1 topik 4), tapi bagian paling relevan (turn
    terbaru) tetap utuh, gak ikut diringkas/hilang.
    """
    total_tokens = estimate_messages_tokens(messages)

    if total_tokens <= max_tokens:
        # Belum melebihi budget -> gak perlu compaction sama sekali,
        # kembalikan apa adanya
        return messages

    # System message (instruksi awal, Phase 1 topik 1) SELALU dipertahankan
    # utuh di posisi pertama -- jangan pernah ikut diringkas atau dibuang
    system_messages = [m for m in messages if m["role"] == "system"]
    non_system_messages = [m for m in messages if m["role"] != "system"]

    # Jaga N_RECENT_TURNS_TO_KEEP pesan terakhir tetap verbatim (gak
    # diringkas) -- ini yang paling relevan buat kelanjutan percakapan saat ini
    N_RECENT_TURNS_TO_KEEP = 4

    if len(non_system_messages) <= N_RECENT_TURNS_TO_KEEP:
        # Non-system messages-nya udah sedikit -> gak ada cukup pesan lama
        # buat diringkas tanpa ikut memakan turn terbaru, jadi kembalikan
        # apa adanya (menurunkan token count bukan tujuan yang lebih penting
        # dibanding menjaga turn terbaru tetap verbatim)
        return messages

    messages_to_summarize = non_system_messages[:-N_RECENT_TURNS_TO_KEEP]
    recent_messages = non_system_messages[-N_RECENT_TURNS_TO_KEEP:]

    summary_text = summarize_messages(messages_to_summarize)
    summary_message = {"role": "system", "content": summary_text}

    # Urutan akhir penting: system asli dulu, baru summary (mewakili history
    # lama), baru turn-turn terbaru verbatim -- persis urutan kronologis asli
    return system_messages + [summary_message] + recent_messages
```

Membuktikan `compact_context` beneran mengurangi ukuran history begitu melebihi `max_tokens`, sambil tetap menjaga turn terbaru:
```python
long_history = [
    {"role": "system", "content": "Kamu adalah asisten customer support SupportPilot."}
]
for i in range(30):
    long_history.append(
        {"role": "user", "content": f"Pertanyaan customer ke-{i}: kapan order saya sampai?"}
    )
    long_history.append(
        {"role": "assistant", "content": f"Jawaban ke-{i}: order kamu sedang diproses."}
    )

print(f"Jumlah pesan sebelum compaction: {len(long_history)}")
print(f"Estimasi token sebelum compaction: {estimate_messages_tokens(long_history)}")

compacted_history = compact_context(long_history, max_tokens=500)

print(f"Jumlah pesan setelah compaction: {len(compacted_history)}")
print(f"Estimasi token setelah compaction: {estimate_messages_tokens(compacted_history)}")
# diharapkan: jumlah pesan setelah compaction jauh lebih sedikit (cuma 1
# system message asli + 1 summary message + 4 pesan terbaru = 6 pesan,
# dibanding 61 pesan sebelum compaction), dan estimasi token setelahnya
# jauh di bawah sebelum compaction

print(compacted_history[-1])
# diharapkan: pesan TERAKHIR di compacted_history persis sama dengan pesan
# TERAKHIR di long_history (verbatim, gak berubah sama sekali) -- karena
# masuk ke 4 pesan terbaru yang dijaga tetap utuh
assert compacted_history[-1] == long_history[-1]
```

### Trade-off & Pitfall
- **`summarize_messages` di atas cuma heuristik (concat teks), bukan ringkasan yang benar-benar "paham" isi percakapan** — di production, fungsi ini semestinya manggil LLM buat menghasilkan summary yang genuinely menangkap inti percakapan, bukan cuma menggabungkan teks mentah; heuristik di sini dipilih supaya contoh tetap runnable tanpa API key, tapi bukan pola yang disarankan buat production.
- **Informasi bisa hilang lewat proses summarization** — meringkas berarti membuang detail; kalau ada detail penting di pesan lama yang gak masuk ke summary (apalagi kalau `summarize_messages` diganti pakai LLM call yang kualitasnya kurang baik), informasi itu efektif hilang dari context selanjutnya.
- **`N_RECENT_TURNS_TO_KEEP` yang hardcoded (4) adalah trade-off, bukan angka ajaib** — kekecilan berarti context lebih hemat tapi lebih sering "lupa" detail turn yang agak lama; kegedean berarti lebih banyak detail terjaga tapi kurang efektif menghemat token. Perlu disesuaikan sama kebutuhan riil (rata-rata panjang percakapan SupportPilot, seberapa sering customer merujuk balik ke turn yang jauh).
- **Compaction yang dipanggil berulang kali atas history yang SUDAH ter-compact sebelumnya bisa "meringkas ringkasan"** — makin sering ini terjadi di sesi yang sangat panjang, detail dari summary sebelumnya bisa makin terkikis tiap kali diringkas ulang; perlu strategi tambahan (misal simpan summary versi sebelumnya secara terpisah) kalau sesi diharapkan berjalan sangat lama.
- **Estimasi token dari `tiktoken` (Phase 1 topik 4) adalah APROKSIMASI**, bukan angka pasti yang dipakai API buat charging — cukup akurat buat keputusan "perlu compact atau nggak", tapi jangan dipakai sebagai satu-satunya sumber kebenaran buat billing/cost tracking yang presisi.

### Kapan Dipakai
- Panggil `compact_context` sebelum riwayat dari `ConversationMemory` (topik 29) atau `retrieve_memories` (topik 31) dikirim ke `run_agent_loop`, khususnya buat sesi yang diperkirakan bakal berjalan panjang (banyak turn, agent yang butuh banyak tool call berturut-turut).
- Pakai strategi **sliding window** sederhana (buang turn lama tanpa diringkas) kalau detail dari turn lama memang gak penting dipertahankan sama sekali dan kesederhanaan implementasi lebih diprioritaskan dibanding menjaga jejak informasi lama.
- Pakai **selective retrieval** (mirip `retrieve_memories`, topik 31) kalau yang dibutuhkan bukan "ringkasan seluruh history lama", tapi "potongan history TERTENTU yang relevan sama pertanyaan saat ini" — dua kebutuhan yang berbeda walau sama-sama soal mengelola context yang membesar.
- Jangan pakai context compaction sebagai pengganti `retrieve_memories`/`save_long_term_memory` (topik 30-31) buat fakta yang memang harus bertahan permanen — compaction cuma soal mengelola SATU sesi yang sedang berjalan, bukan mekanisme persistence lintas sesi.

### Sering Ditanya Saat Interview
- **Apa beda Context Engineering dengan Prompt Engineering (Phase 2)?** — Prompt Engineering soal menyusun SATU prompt yang efektif untuk satu request; Context Engineering soal mengelola SELURUH isi context (termasuk history yang terus bertambah) selama sesi multi-turn/agent yang panjang berjalan.
- **Bagaimana `compact_context` menentukan kapan compaction perlu dilakukan?** — dengan mengestimasi total token seluruh `messages` (pakai `tiktoken`, Phase 1 topik 4) dan membandingkannya dengan `max_tokens`; kalau masih di bawah batas, `messages` dikembalikan apa adanya tanpa perubahan.
- **Kenapa `compact_context` gak meringkas SEMUA pesan, termasuk yang terbaru?** — supaya turn-turn paling baru (paling relevan buat kelanjutan percakapan saat ini) tetap verbatim dan gak kehilangan detail; cuma pesan-pesan yang lebih lama dari `N_RECENT_TURNS_TO_KEEP` yang diringkas jadi satu summary.
- **Apa risiko utama dari summarization dalam context compaction?** — informasi detail dari pesan lama bisa hilang begitu diringkas, terutama kalau kualitas summary-nya kurang baik atau dilakukan berulang kali atas history yang sudah pernah diringkas sebelumnya ("meringkas ringkasan").

---

## 33. AI Memory Landscape — Tools & Produk Nyata

### Apa itu?
AI Memory Landscape adalah gambaran tools/produk nyata yang mengimplementasikan konsep-konsep memory yang sudah dibahas di topik 29-32 (short-term, long-term, retrieval, context compaction) secara siap pakai — supaya gak selalu harus membangun `ConversationMemory`, `save_long_term_memory`, `retrieve_memories`, dan `compact_context` dari nol tiap kali butuh memory di sebuah agent. Ada dua kubu besar pendekatan yang tujuannya beda: infrastruktur memory buat agent yang dibangun developer sendiri, versus pendekatan personal knowledge management (PKM) yang lebih relevan buat penggunaan personal.

### Kenapa dibutuhkan?
Membangun sistem memory yang benar-benar robust (menangani masalah seperti fakta yang kontradiktif seiring waktu, retrieval yang efisien di skala besar, atau memory yang bisa "mengatur dirinya sendiri" soal apa yang tetap di context aktif vs dipindah ke storage archival) itu bukan pekerjaan trivial — versi `save_long_term_memory`/`retrieve_memories` di topik 30-31 sengaja dibuat sederhana supaya konsepnya jelas, tapi versi production-grade biasanya butuh penanganan edge case yang jauh lebih banyak. Paham lanskap tools yang sudah ada penting buat bisa memutuskan: bangun sendiri (seperti pola di topik 29-32) atau adopsi salah satu tool yang sudah battle-tested, tergantung skala dan kebutuhan spesifik SupportPilot.

### Cara Kerja
**Developer/Agent Memory Infrastructure** (dipasang ke agent yang dibangun sendiri, mirip kebutuhan `run_agent_loop` Phase 6):
- **Mem0** — paling populer (60K+ GitHub stars), menyimpan memory berlapis (per-percakapan, per-user, per-organisasi), dan otomatis mengekstrak fakta penting dari percakapan (bagian yang di topik 30 masih manual lewat pemanggilan eksplisit `save_long_term_memory`). Pilihan umum kalau butuh "tempel" persistent memory ke agent yang sudah ada tanpa membangun seluruh pipeline dari nol.
- **Zep** (engine-nya disebut **Graphiti**) — pakai *temporal knowledge graph*: tiap fakta dicatat KAPAN berlaku dan kapan berubah, jadi gak ketuker fakta lama vs baru — ini persis menyelesaikan masalah "fakta yang jadi usang/kontradiktif seiring waktu" yang disebut sebagai keterbatasan di topik 30.
- **Letta** (sebelumnya bernama **MemGPT**) — agent yang bisa mengatur sendiri memorinya: apa yang tetap ada di memory block (context aktif), apa yang dipindah ke storage archival, dan apa yang di-recall lagi saat dibutuhkan — pendekatan yang lebih otonom dibanding `compact_context` (topik 32) yang aturannya masih ditulis manual oleh developer.
- **LangMem** — paling native buat yang sudah pakai **LangGraph** (Phase 6 topik 27), terintegrasi langsung sebagai long-term memory store di dalam graph yang sama.

**Personal Knowledge Management (PKM) sebagai Memory** — rute yang lebih relevan buat kebutuhan personal, bukan agent produksi:
- **Obsidian** sendiri **bukan** AI memory system — itu cuma kumpulan file markdown biasa di device. Yang bikin jadi "AI memory" adalah **plugin RAG** di atasnya, seperti **Smart Connections** atau **Copilot for Obsidian**: catatan di-embed (persis pola Phase 3 topik 9), dan saat ditanya sesuatu, plugin retrieve catatan yang relevan lalu dikasih ke LLM sebagai context — sama persis pola RAG di Phase 4, tapi sumbernya vault pribadi, bukan `knowledge_articles` SupportPilot.
- **Nuansa penting**: sebuah folder catatan TANPA RAG yang benar itu cuma **penyimpanan pasif**, bukan memory. Yang bikin sesuatu layak disebut memory adalah kombinasi *retrieval yang selektif* (topik 31), *persistence yang bertahan* (topik 30), dan *navigasi terstruktur* — bukan sekadar menumpuk file markdown tanpa cara mencari kembali yang cerdas.

### Trade-off & Pitfall
- **Adopsi tool pihak ketiga (Mem0/Zep/Letta/LangMem) berarti dependency infrastruktur tambahan** — perlu dievaluasi sama seperti keputusan build-vs-buy lainnya: seberapa matang tool-nya, seberapa mudah di-self-host atau diintegrasikan ke stack SupportPilot yang sudah ada (PostgreSQL + pgvector), dan risiko vendor lock-in kalau nanti mau pindah.
- **Temporal knowledge graph (Zep/Graphiti) menyelesaikan masalah "fakta usang" dengan baik, tapi menambah kompleksitas modeling** dibanding tabel `customer_memories` sederhana (topik 30) — worth-it kalau fakta yang berubah seiring waktu memang sering terjadi dan penting buat dilacak dengan presisi, overkill kalau kebutuhannya masih sederhana.
- **Memory yang "mengatur dirinya sendiri" (Letta/MemGPT) kurang predictable dibanding aturan eksplisit** (`compact_context` manual, topik 32) — trade-off yang sama persis dengan agent vs workflow (Phase 6 topik 26): lebih fleksibel, tapi lebih susah di-debug/diprediksi perilakunya persis.
- **PKM (Obsidian + plugin RAG) gak dirancang buat kebutuhan multi-user/production** — ini pendekatan personal, gak punya konsep `customer_id` scoping seperti `retrieve_memories` (topik 31) yang wajib buat sistem yang melayani banyak customer sekaligus.

### Kapan Dipakai
- Kalau tujuannya personal note-taking + tanya-jawab ke catatan sendiri → Obsidian + plugin RAG (Smart Connections/Copilot for Obsidian) sudah cukup, gak perlu infrastruktur memory agent yang lebih berat.
- Kalau tujuannya membangun agent produksi (seperti SupportPilot) yang butuh memory lintas sesi/customer dalam skala besar, dan pola sederhana di topik 29-32 mulai terasa gak cukup robust (fakta sering kontradiktif, butuh manajemen memory otomatis) → Mem0/Zep/Letta/LangMem lebih relevan dipelajari & dipertimbangkan lebih dulu dibanding membangun semuanya dari nol.
- Kalau kebutuhannya masih sesuai skala pola di topik 29-32 (short-term via `ConversationMemory`, long-term via tabel `customer_memories` sederhana) — gak perlu buru-buru adopsi tool pihak ketiga; tambahkan kompleksitas cuma kalau kebutuhan nyata sudah melampaui itu.

### Sering Ditanya Saat Interview
- **Sebutkan beberapa tool developer memory infrastructure yang populer, dan apa yang membedakan masing-masing.** — Mem0 (memory berlapis + ekstraksi fakta otomatis), Zep/Graphiti (temporal knowledge graph, melacak kapan fakta berlaku/berubah), Letta/MemGPT (agent yang mengatur sendiri apa yang di memory aktif vs archival), LangMem (paling native buat integrasi LangGraph).
- **Kenapa Obsidian sendiri bukan "AI memory system"?** — Obsidian cuma kumpulan file markdown biasa; yang membuatnya berfungsi sebagai memory adalah plugin RAG di atasnya (Smart Connections/Copilot for Obsidian) yang meng-embed catatan dan melakukan retrieval selektif — tanpa itu, itu cuma penyimpanan pasif.
- **Apa yang membedakan "penyimpanan pasif" dari "memory" yang sesungguhnya?** — memory sesungguhnya butuh kombinasi retrieval yang selektif, persistence yang bertahan lintas waktu, dan navigasi/struktur yang jelas — bukan sekadar menumpuk data tanpa cara mengambilnya kembali secara cerdas.
- **Kapan sebaiknya adopsi tool memory pihak ketiga dibanding membangun sendiri (seperti pola topik 29-32)?** — begitu kebutuhan sudah melampaui yang bisa ditangani pola sederhana (misal butuh menangani fakta yang sering kontradiktif seiring waktu, atau butuh manajemen memory otomatis skala besar) — sebelum itu, membangun sendiri dengan pola sederhana biasanya sudah cukup dan lebih mudah dikontrol.

---

**Selanjutnya:** [Phase 08 — Agent Skills](./phase-08-agent-skills.md)
