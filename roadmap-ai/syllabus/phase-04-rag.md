# Phase 04 — RAG

> Bagian dari [AI Engineering / Agent Roadmap](../README.md)

---

## 12. What is RAG?

### Apa itu?
RAG (Retrieval-Augmented Generation) adalah pola di mana LLM gak cuma mengandalkan pengetahuan yang "nempel" di parameternya dari training (yang bisa aja ketinggalan zaman atau gak spesifik ke bisnis kita), tapi dikasih dulu **dokumen relevan** sebagai konteks tambahan sebelum diminta jawab. Nama "retrieval-augmented" secara harfiah artinya: generation (jawaban LLM) yang di-augment (diperkuat) pakai hasil retrieval (pencarian dokumen).

Alurnya secara garis besar: pertanyaan customer → cari dokumen yang relevan (pakai embedding & vector similarity, Phase 3) → tempelkan isi dokumen itu ke dalam prompt → LLM jawab berdasarkan dokumen tersebut, bukan cuma dari "ingatan" trainingnya.

### Kenapa dibutuhkan?
LLM yang dipanggil polos (tanpa RAG) buat pertanyaan spesifik soal SupportPilot — misalnya "berapa lama proses refund SupportPilot" — bakal ngarang jawaban generik atau bahkan salah, karena kebijakan refund SupportPilot itu gak ada di data training model manapun (itu data internal perusahaan, bukan pengetahuan umum). LLM juga gak tau perubahan kebijakan terbaru yang baru diupdate minggu lalu, karena training-nya sudah "beku" sejak beberapa bulan/tahun lalu (knowledge cutoff).

RAG menyelesaikan ini dengan cara paling praktis: daripada training ulang/fine-tune model tiap kali ada artikel help-center baru (mahal dan lambat), kita cukup **taruh** info yang relevan langsung di prompt saat request terjadi. Model jadi jawab berdasarkan fakta yang benar-benar ada di artikel help-center SupportPilot, bukan tebakan.

### Cara Kerja
Versi paling sederhana dari RAG:
```
Pertanyaan customer
    → cari 1+ dokumen relevan (embedding + vector similarity, Phase 3)
    → tempel isi dokumen ke dalam prompt sebagai konteks
    → kirim ke LLM
    → LLM jawab berdasarkan konteks itu (grounded answer)
```

Fungsi `search_knowledge_base(conn, query, top_k=5)` dari Phase 3 sudah menghandle bagian "cari dokumen relevan" — dia sendiri yang memanggil `generate_embedding` buat pertanyaan customer, lalu query pgvector buat nemuin artikel yang paling mirip secara makna. Di topik ini kita pakai langsung fungsi itu apa adanya; topik-topik selanjutnya (13-18) bakal memperdalam tiap tahap dari pipeline ini (ingestion, chunking, retrieval, reranking) buat kasus yang lebih realistis (dokumen panjang, banyak artikel, dll).

### Contoh Kode — Python
Versi "hello world" RAG: retrieve satu artikel paling relevan, tempel ke prompt, minta LLM jawab berdasarkan artikel itu:

```python
from openai import OpenAI

client = OpenAI()


def answer_with_rag(conn, question: str) -> str:
    """
    Versi RAG paling sederhana: cari satu artikel help-center paling relevan
    (search_knowledge_base dari Phase 3), tempel ke prompt, minta LLM jawab
    berdasarkan artikel itu (bukan dari "ingatan" training LLM).
    """
    # search_knowledge_base sendiri yang men-generate_embedding pertanyaannya (lihat Phase 3)
    hasil = search_knowledge_base(conn, question, top_k=1)
    artikel_relevan = hasil[0]

    prompt = (
        "Jawab pertanyaan customer berikut HANYA berdasarkan artikel help-center di bawah ini. "
        "Kalau artikelnya gak cukup buat jawab, bilang terus terang gak tau.\n\n"
        f"Judul artikel: {artikel_relevan['title']}\n"
        f"Isi artikel: {artikel_relevan['content']}\n\n"
        f"Pertanyaan customer: {question}"
    )

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0,
    )
    return response.choices[0].message.content


jawaban = answer_with_rag(conn, "gimana caranya reset password ya?")
print(jawaban)
# diharapkan: jawaban yang isinya sesuai instruksi reset password di artikel help-center,
# bukan jawaban generik/ngarang
```

### Trade-off & Pitfall
- **RAG bukan solusi ajaib buat semua masalah akurasi LLM.** Kalau dokumen yang di-retrieve gak relevan (retrieval-nya jelek), LLM tetap bakal jawab ngaco walau formatnya "kelihatan" grounded — RAG cuma sebagus retrieval-nya.
- **Nambah 1+ artikel ke prompt = nambah token = nambah biaya & latency** dibanding manggil LLM polos. Perlu dipikirkan berapa banyak/panjang dokumen yang di-stuff ke prompt.
- **LLM bisa aja tetap "mengarang" (hallucinate) walau udah dikasih konteks**, terutama kalau instruksinya gak eksplisit bilang "jawab HANYA berdasarkan konteks ini" — makanya instruksi ini penting ditulis eksplisit di prompt.
- **Versi sederhana ini (top_k=1, satu artikel utuh) gak scalable buat dokumen panjang** — kalau artikelnya panjang banget, dia gak bakal muat semua ke satu embedding yang representatif, dan gak semua bagiannya relevan ke pertanyaan. Ini yang diselesaikan lewat chunking (topik 14).

### Kapan Dipakai
- Pakai RAG begitu jawaban LLM butuh **fakta spesifik ke bisnis/data internal** yang gak mungkin ada di training data model (kebijakan internal, data customer, artikel help-center terbaru).
- Versi sederhana (retrieve satu dokumen, top_k=1) cukup buat kasus di mana satu artikel biasanya sudah menjawab satu pertanyaan secara utuh dan artikelnya pendek.
- Kalau dokumennya panjang, jumlahnya banyak, atau satu jawaban butuh info dari beberapa sumber sekaligus — lanjut ke topik 13-16 (ingestion, chunking, retrieval multi-dokumen, reranking).

### Sering Ditanya Saat Interview
- **Apa itu RAG, dan kenapa dibutuhkan?** — pola yang menyuntikkan dokumen relevan ke prompt sebelum LLM menjawab, supaya jawabannya grounded ke fakta yang benar-benar ada (bukan cuma "ingatan" dari training), berguna buat data internal/spesifik bisnis yang gak ada di training data model manapun.
- **Kenapa gak fine-tune model aja daripada pakai RAG?** — fine-tune mahal, lambat, dan harus diulang tiap kali datanya berubah; RAG cukup update index pencarian (misal tambah artikel baru), jauh lebih murah dan cepat buat data yang sering berubah.
- **Apakah RAG menjamin LLM gak akan hallucinate?** — enggak — RAG cuma sebagus dokumen yang di-retrieve; kalau retrieval-nya salah/gak relevan, atau instruksinya gak eksplisit minta jawab berdasarkan konteks, LLM tetap bisa ngarang.
- **Apa komponen utama dari RAG pipeline?** — retrieval (cari dokumen relevan, biasanya pakai embedding & vector similarity) dan generation (LLM menjawab berdasarkan dokumen yang diretrieve).

---

## 13. RAG Ingestion Pipeline

### Apa itu?
Ingestion pipeline adalah proses mengubah dokumen mentah (misalnya file artikel help-center dalam bentuk PDF/text) jadi data yang siap dicari lewat vector similarity: **extract** (ambil teksnya dari file), **clean** (bersihin karakter aneh/formatting yang gak perlu), **chunk** (potong jadi bagian-bagian lebih kecil), **embed** (ubah tiap potongan jadi vector, Phase 3), lalu **store** (simpan ke pgvector). Ini kebalikan dari topik 12 tadi yang masih pakai artikel utuh — di sini kita siapkan pipeline yang lebih realistis buat dokumen yang lebih panjang/banyak.

### Kenapa dibutuhkan?
Di topik 12, `search_knowledge_base` bekerja di atas tabel `knowledge_articles` yang isinya satu embedding per satu artikel utuh (Phase 3). Ini cukup buat artikel pendek, tapi begitu SupportPilot punya artikel help-center yang panjang (misalnya panduan lengkap "cara integrasi API SupportPilot" yang berhalaman-halaman), satu embedding buat SELURUH artikel itu jadi terlalu "kabur" — dia mewakili rata-rata makna dari banyak topik berbeda sekaligus, jadi kurang presisi buat nemuin bagian spesifik yang relevan ke satu pertanyaan customer.

Ingestion pipeline menyelesaikan ini dengan memecah dokumen jadi potongan-potongan (chunk) yang lebih kecil dan fokus, lalu meng-embed **tiap chunk secara terpisah**. Hasilnya, saat customer nanya sesuatu yang spesifik, kita bisa nemuin chunk yang paling relevan — bukan cuma "artikel mana yang paling relevan", tapi "bagian mana persisnya dari artikel itu yang relevan".

### Cara Kerja
Alur ingestion pipeline buat satu file artikel SupportPilot:
```
File artikel (plain text)
    → baca isinya (extract)
    → potong jadi beberapa Chunk (chunk, topik 14)
    → generate_embedding tiap Chunk (Phase 3)
    → INSERT tiap Chunk + embedding-nya ke tabel article_chunks (pgvector)
```

Kita butuh struktur data buat "membawa" tiap potongan teks beserta info asalnya (dari file mana, urutan ke berapa) selama proses ini berjalan. Di sinilah `Chunk` diperkenalkan.

> **Catatan Python:** `@dataclass` adalah decorator bawaan Python (dari modul `dataclasses`) yang otomatis bikinin `__init__`, `__repr__`, dan beberapa method lain buat class yang isinya cuma "bundel data" (kumpulan field), jadi lu gak perlu nulis manual `def __init__(self, text, source, chunk_index): self.text = text; ...` — cukup deklarasikan field-nya dengan type hint, sisanya di-generate otomatis.

```python
from dataclasses import dataclass


@dataclass
class Chunk:
    """Satu potongan teks hasil chunking, beserta info dari mana asalnya."""
    text: str          # isi teks potongan ini
    source: str        # asalnya dari file/artikel mana (misal path file)
    chunk_index: int    # urutan potongan ini di dalam dokumen asalnya (mulai dari 0)
```

Tabel `article_chunks` di pgvector, dibikin dengan pola yang sama persis kayak `knowledge_articles` di Phase 3 (extension `vector` sudah aktif di database yang sama):
```sql
-- Tabel buat nyimpen potongan (chunk) artikel, satu baris per chunk (bukan per artikel utuh)
CREATE TABLE article_chunks (
    id SERIAL PRIMARY KEY,
    source TEXT NOT NULL,        -- asal dokumen (misal path file artikel)
    chunk_index INT NOT NULL,    -- urutan chunk ini di dalam dokumen asalnya
    content TEXT NOT NULL,       -- isi teks chunk (field `text` di dataclass Chunk)
    -- dimensi 1536 sama persis dengan knowledge_articles di Phase 3
    -- (text-embedding-3-small, konsisten satu model buat semua tabel)
    embedding VECTOR(1536) NOT NULL
);

-- Index ANN, pola identik dengan knowledge_articles_embedding_idx di Phase 3
CREATE INDEX article_chunks_embedding_idx
ON article_chunks
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

### Contoh Kode — Python
`ingest_document(conn, file_path)` membaca satu file artikel SupportPilot (plain text), memotongnya jadi beberapa `Chunk` lewat `chunk_text` (didefinisikan lengkap di topik 14 — di sini kita pakai duluan, forward reference, karena secara alur ingestion memang butuh chunking di tengah prosesnya), lalu meng-embed dan menyimpan tiap chunk:

```python
def ingest_document(conn, file_path: str) -> None:
    """
    Baca file artikel SupportPilot (plain text), potong jadi beberapa Chunk,
    generate embedding tiap Chunk (Phase 3), lalu simpan semuanya ke article_chunks.
    """
    with open(file_path, "r", encoding="utf-8") as f:
        raw_text = f.read()

    # chunk_text didefinisikan lengkap di topik 14 (Chunking); di sini dianggap sudah ada
    chunks = chunk_text(raw_text, chunk_size=500, overlap=50)

    with conn.cursor() as cur:
        for chunk in chunks:
            # chunk_text belum tau chunk ini berasal dari file mana, jadi kita isi di sini
            chunk.source = file_path

            embedding = generate_embedding(chunk.text)
            embedding_literal = "[" + ",".join(str(x) for x in embedding) + "]"

            cur.execute(
                """
                INSERT INTO article_chunks (source, chunk_index, content, embedding)
                VALUES (%s, %s, %s, %s::vector);
                """,
                (chunk.source, chunk.chunk_index, chunk.text, embedding_literal),
            )

    conn.commit()


# Contoh pemakaian: ingest satu file artikel help-center SupportPilot
ingest_document(conn, "artikel/cara-integrasi-api-supportpilot.txt")
```

Catatan: `Chunk` di atas bukan `frozen` dataclass, jadi field-nya boleh diubah setelah objeknya dibuat — makanya baris `chunk.source = file_path` di atas valid (kalau mau field-nya gak bisa diubah lagi setelah dibuat, tinggal tambahkan `@dataclass(frozen=True)`).

### Trade-off & Pitfall
- **Ingestion itu proses yang harus di-rerun kalau dokumen sumbernya berubah.** Kalau artikel help-center diedit, chunk & embedding lama yang sudah kadung tersimpan jadi basi (stale) — perlu strategi buat re-ingest (hapus chunk lama dari `source` yang sama, insert ulang) tiap kali ada update konten.
- **Extract yang jelek (misal dari PDF dengan layout kompleks) bisa menghasilkan teks berantakan** (kolom tercampur, tabel jadi teks acak) — chunk & embedding yang dihasilkan dari teks berantakan ini kualitasnya ikut jelek, walau proses chunking/embedding-nya sendiri jalan tanpa error.
- **Ingest dokumen besar itu banyak panggilan API (satu `generate_embedding` per chunk)** — kalau satu dokumen jadi ratusan chunk, ini banyak panggilan sekaligus; perlu dipikirkan rate limit API dan biaya totalnya, bukan cuma biaya per chunk.
- **Lupa `conn.commit()` di akhir bikin semua INSERT hilang** begitu koneksi ditutup (perilaku transaksi PostgreSQL biasa, sama kayak operasi database lain yang reader udah familiar).

### Kapan Dipakai
- Jalankan ingestion pipeline setiap kali ada artikel help-center baru ditambahkan, atau artikel lama diedit isinya.
- Cocok dijalankan sebagai batch job terpisah (bukan di request path customer) — customer gak perlu nunggu proses ingest selesai buat dapat jawaban dari RAG.
- Kalau volume dokumennya besar, ingestion biasanya dijadwalkan (misal cron job harian) atau di-trigger event-based (begitu artikel baru disimpan ke sistem CMS internal).

### Sering Ditanya Saat Interview
- **Apa isi dari RAG ingestion pipeline?** — extract (ambil teks dari file), clean (bersihin format), chunk (potong jadi bagian kecil), embed (ubah tiap chunk jadi vector), store (simpan ke vector database).
- **Kenapa gak embed satu artikel utuh aja kayak Phase 3?** — artikel panjang yang di-embed utuh jadi "kabur" (satu vector mewakili banyak topik sekaligus), kurang presisi buat nemuin bagian spesifik yang relevan ke satu pertanyaan.
- **Apa yang dilakukan `@dataclass` buat class `Chunk`?** — otomatis generate `__init__`, `__repr__`, dll berdasarkan field yang dideklarasikan, jadi gak perlu nulis boilerplate constructor manual buat class yang isinya cuma bundel data.
- **Kenapa ingestion perlu di-rerun kalau dokumen sumbernya berubah?** — chunk & embedding yang sudah tersimpan merepresentasikan versi lama dokumennya; kalau gak di-rerun, retrieval bakal nemuin/nampilin info yang sudah basi.

---

## 14. Chunking

### Apa itu?
Chunking adalah proses memotong satu dokumen panjang jadi beberapa potongan (chunk) teks yang lebih kecil, supaya tiap potongan bisa di-embed secara terpisah dan lebih presisi mewakili satu "unit makna" tertentu, bukan campuran banyak topik sekaligus.

### Kenapa dibutuhkan?
Model embedding (Phase 3, topik 9) menghasilkan **satu** vector dengan panjang tetap (1536 dimensi buat `text-embedding-3-small`) buat berapa pun panjang teks input-nya (selama masih di bawah limit token model). Kalau kita embed satu dokumen 10 halaman jadi satu vector doang, vector itu harus "meringkas" makna dari SEMUA isi 10 halaman itu sekaligus — hasilnya jadi representasi yang kabur/rata-rata, bukan representasi yang presisi buat satu bagian tertentu.

Selain itu, kalaupun dokumen itu berhasil di-retrieve karena "relevan secara umum", kita tetap harus kasih LLM bagian teks yang SPESIFIK relevan ke pertanyaan customer, bukan seluruh dokumen (yang boros token & context window, Phase 1). Chunking menyelesaikan kedua masalah ini sekaligus: tiap chunk jadi unit retrieval yang lebih presisi, dan cuma chunk yang relevan yang perlu ditempel ke prompt.

### Cara Kerja
Dua parameter kunci dalam chunking:
- **`chunk_size`** — panjang maksimum tiap chunk (di sini kita ukur pakai jumlah karakter, buat kesederhanaan; versi production sering pakai jumlah token biar konsisten dengan limit model embedding/LLM).
- **`overlap`** — berapa banyak karakter di akhir satu chunk yang "diulang" lagi di awal chunk berikutnya, supaya kalimat yang informasinya nyambung di perbatasan dua chunk gak keputus makna-nya jadi dua bagian yang gak nyambung.

Trade-off ukuran chunk:
- **`chunk_size` kecil** → tiap chunk lebih fokus/presisi (bagus buat retrieval spesifik), tapi butuh lebih banyak chunk per dokumen (lebih banyak panggilan embedding, dan konteks yang ditempel ke LLM jadi lebih terpotong-potong/kurang utuh).
- **`chunk_size` besar** → tiap chunk lebih "utuh" secara konteks, tapi embedding-nya jadi kurang presisi (kembali ke masalah "campur banyak topik") dan boros token kalau ditempel ke prompt.

Pendekatan chunking yang kita pakai di sini adalah **sliding window** berbasis karakter (potong tiap `chunk_size` karakter, geser sejauh `chunk_size - overlap`). ada juga pendekatan yang lebih canggih:
- **Recursive chunking** — coba potong di batas "alami" dulu (misal per paragraf, baru per kalimat, baru per kata) sebelum jatuh ke potong paksa per jumlah karakter, supaya gak motong tengah kalimat/kata.
- **Semantic chunking** — potong berdasarkan pergeseran makna (pakai embedding tiap kalimat buat deteksi "di titik mana topiknya berubah"), bukan berdasarkan panjang tetap.

Buat SupportPilot, sliding window sederhana ini biasanya sudah cukup buat artikel help-center yang terstruktur rapi (per-paragraf, gak terlalu panjang); recursive/semantic chunking lebih relevan buat dokumen yang jauh lebih panjang dan kompleks strukturnya.

### Contoh Kode — Python
Implementasi `chunk_text` (sliding window berbasis karakter), memakai `Chunk` yang sudah diperkenalkan di topik 13:

```python
def chunk_text(text: str, chunk_size: int = 500, overlap: int = 50) -> list[Chunk]:
    """
    Potong `text` jadi beberapa Chunk pakai sliding window:
    tiap chunk panjangnya maksimal `chunk_size` karakter, dan `overlap` karakter
    terakhir dari satu chunk diulang lagi di awal chunk berikutnya.
    """
    if overlap >= chunk_size:
        # kalau overlap >= chunk_size, posisi awal (`start`) gak akan pernah maju -> infinite loop
        raise ValueError("overlap harus lebih kecil dari chunk_size")

    chunks: list[Chunk] = []
    start = 0
    index = 0

    while start < len(text):
        potongan = text[start:start + chunk_size].strip()
        if potongan:
            # source dikosongkan dulu di sini; ingest_document (topik 13) yang mengisinya,
            # karena chunk_text sendiri gak tau teks ini berasal dari file mana
            chunks.append(Chunk(text=potongan, source="", chunk_index=index))
            index += 1
        start += chunk_size - overlap

    return chunks


# Contoh: potong satu artikel help-center SupportPilot yang lumayan panjang
artikel_integrasi_api = (
    "Untuk mengintegrasikan API SupportPilot, pertama-tama daftar dulu di dashboard developer "
    "buat dapetin API key. API key ini harus dikirim lewat header Authorization di tiap request. "
    "Setelah itu, endpoint utama yang perlu diketahui adalah /v1/tickets buat bikin tiket baru, "
    "dan /v1/tickets/{id} buat ambil detail satu tiket. Semua response dikembalikan dalam format "
    "JSON. Rate limit default adalah 100 request per menit per API key; kalau butuh limit lebih "
    "tinggi, hubungi tim support buat upgrade plan. Kesalahan umum yang sering terjadi adalah lupa "
    "menyertakan header Content-Type: application/json, yang bikin request POST gagal diproses."
)

potongan_potongan = chunk_text(artikel_integrasi_api, chunk_size=150, overlap=30)

for c in potongan_potongan:
    print(f"[chunk {c.chunk_index}] {c.text!r}")

print(f"Total chunk: {len(potongan_potongan)}")
```

### Trade-off & Pitfall
- **`overlap` yang terlalu besar relatif ke `chunk_size` bikin banyak duplikasi teks antar chunk** (boros storage & embedding calls) — dan kalau `overlap >= chunk_size`, chunking gak akan pernah maju (infinite loop), makanya kode di atas eksplisit menolak kasus ini.
- **Sliding window berbasis karakter bisa motong tengah kalimat/kata** — secara semantik ini kurang ideal (satu kalimat penting bisa kepotong jadi dua chunk yang gak nyambung maknanya), tapi implementasinya paling sederhana dan cukup buat banyak kasus.
- **`chunk_size` yang optimal itu spesifik ke jenis dokumen dan model embedding-nya** — gak ada angka "benar" universal; biasanya perlu dicoba beberapa nilai dan dievaluasi hasil retrieval-nya (topik 18).
- **Chunking yang buruk adalah salah satu penyebab retrieval salah paling umum** — kalau satu chunk mencampur dua topik berbeda (misal kebijakan refund NYAMBUNG ke cara reset password karena kebetulan bersebelahan di dokumen sumber), retrieval bisa nemuin chunk itu buat pertanyaan yang sebenarnya gak relevan.

### Kapan Dipakai
- Pakai sliding window sederhana (seperti di atas) buat dokumen yang terstruktur rapi dan gak terlalu panjang — cukup buat kebanyakan artikel help-center SupportPilot.
- Pakai **recursive chunking** kalau dokumennya punya struktur jelas (heading, paragraf, list) yang sebaiknya dihormati saat memotong, biar gak motong di tengah unit yang harusnya utuh.
- Pakai **semantic chunking** kalau dokumennya panjang dan topiknya bisa berubah-ubah di tengah teks tanpa penanda struktural yang jelas (butuh effort lebih, biasanya buat dokumen yang benar-benar kompleks).
- `chunk_size` kecil-menengah (ratusan karakter) biasanya jadi titik awal yang wajar buat artikel help-center; sesuaikan lagi berdasarkan hasil evaluasi retrieval (topik 18).

### Sering Ditanya Saat Interview
- **Kenapa gak bisa embed satu dokumen panjang jadi satu vector aja?** — hasilnya jadi representasi rata-rata dari banyak topik sekaligus (kabur), kurang presisi buat retrieval yang butuh nemuin bagian spesifik yang relevan.
- **Apa fungsi `overlap` dalam chunking?** — supaya kalimat/informasi yang nyambung tepat di perbatasan dua chunk gak keputus maknanya jadi dua chunk yang gak nyambung.
- **Apa risiko kalau `chunk_size` terlalu kecil vs terlalu besar?** — terlalu kecil bikin chunk kehilangan konteks yang lebih luas (dan lebih banyak panggilan embedding); terlalu besar bikin embedding-nya kembali kabur (campur banyak topik) dan boros token kalau ditempel ke prompt.
- **Apa beda recursive chunking dan semantic chunking?** — recursive chunking motong berdasarkan batas struktural (paragraf/kalimat/kata) sebelum jatuh ke potong paksa; semantic chunking motong berdasarkan deteksi pergeseran makna antar bagian teks (pakai embedding), bukan berdasarkan panjang tetap.

---

## 15. Retrieval

### Apa itu?
Retrieval adalah tahap "mencari" chunk-chunk mana yang paling relevan buat satu pertanyaan customer, dari seluruh chunk yang sudah tersimpan hasil ingestion (topik 13-14). Konsep-konsep kuncinya:

- **Top-K** — ambil K chunk teratas (paling mirip secara makna) dari hasil pencarian, bukan cuma satu, biar LLM punya beberapa kandidat konteks buat dipilih/digabungkan.
- **Similarity threshold** — batas minimum "kemiripan" (atau maksimum "jarak") supaya chunk yang sebenarnya gak relevan sama sekali gak ikut ditempel ke prompt, walau dia "paling mirip di antara yang ada".
- **Metadata filtering** — mempersempit pencarian pakai kondisi non-vector dulu (misal `WHERE source = 'artikel-billing.txt'`), baru urutkan sisanya pakai vector similarity.
- **Hybrid search** — gabungan pencarian keyword (full-text search PostgreSQL biasa) DAN vector similarity, buat menangkap kasus di mana keyword yang persis sama juga penting (misal kode error spesifik "ERR_4021" yang embedding-nya belum tentu "menangkap" kemiripan string yang persis sama).

### Kenapa dibutuhkan?
Setelah ingestion (topik 13) selesai, `article_chunks` bisa berisi ribuan baris chunk dari ratusan artikel. `retrieve_relevant_chunks` adalah fungsi yang menjembatani "pertanyaan customer" ke "chunk-chunk spesifik yang relevan" — ini pola yang sama persis dengan `search_knowledge_base` di Phase 3, tapi beroperasi di level chunk (lebih presisi), bukan di level satu artikel utuh.

Top-K dibutuhkan karena satu pertanyaan customer kadang butuh info dari lebih dari satu chunk (misal satu chunk soal langkah teknis, chunk lain soal syarat/batasannya) — ambil cuma 1 chunk (seperti topik 12) bisa aja melewatkan bagian penting yang justru ada di chunk lain. Similarity threshold & metadata filtering dibutuhkan karena Top-K doang gak menjamin hasilnya BENERAN relevan — kalau semua chunk yang ada memang gak relevan, Top-K tetap bakal mengembalikan "yang paling gak-jauh", padahal semuanya sebenarnya gak berguna buat pertanyaan itu.

### Cara Kerja
Alur `retrieve_relevant_chunks`, sama persis polanya dengan `search_knowledge_base` (Phase 3), tapi query ke tabel `article_chunks`:
```
Pertanyaan customer → generate_embedding() → query vector
    → SQL: ORDER BY embedding <=> query_vector LIMIT top_k
    → top_k chunk paling mirip secara makna, dari tabel article_chunks
```

### Contoh Kode — Python
```python
import psycopg2
import psycopg2.extras


def retrieve_relevant_chunks(conn, query: str, top_k: int = 5) -> list[dict]:
    """
    Sama seperti search_knowledge_base (Phase 3), tapi query ke tabel article_chunks
    (granularitas chunk, bukan satu artikel utuh).
    """
    query_embedding = generate_embedding(query)
    embedding_literal = "[" + ",".join(str(x) for x in query_embedding) + "]"

    with conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor) as cur:
        cur.execute(
            """
            SELECT
                id,
                source,
                chunk_index,
                content,
                embedding <=> %s::vector AS distance
            FROM article_chunks
            ORDER BY embedding <=> %s::vector
            LIMIT %s;
            """,
            (embedding_literal, embedding_literal, top_k),
        )
        rows = cur.fetchall()

    return [dict(row) for row in rows]


# Contoh pemakaian
chunk_relevan = retrieve_relevant_chunks(conn, "berapa lama waktu integrasi API SupportPilot", top_k=5)

for c in chunk_relevan:
    print(f"{c['source']} chunk#{c['chunk_index']} (distance={c['distance']:.4f})")
    print(f"  {c['content'][:80]}...")
```

Contoh nambahin metadata filtering (persempit ke satu sumber dokumen tertentu dulu) dan similarity threshold sederhana:
```python
def retrieve_relevant_chunks_terfilter(
    conn, query: str, source: str, max_distance: float = 0.3, top_k: int = 5
) -> list[dict]:
    """
    Variasi retrieve_relevant_chunks: cuma cari di dalam satu `source` tertentu
    (metadata filtering), dan buang chunk yang distance-nya di atas `max_distance`
    (similarity threshold) — supaya chunk yang beneran gak relevan gak ikut kebawa.
    """
    query_embedding = generate_embedding(query)
    embedding_literal = "[" + ",".join(str(x) for x in query_embedding) + "]"

    with conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor) as cur:
        cur.execute(
            """
            SELECT id, source, chunk_index, content, embedding <=> %s::vector AS distance
            FROM article_chunks
            WHERE source = %s AND embedding <=> %s::vector < %s
            ORDER BY embedding <=> %s::vector
            LIMIT %s;
            """,
            (embedding_literal, source, embedding_literal, max_distance, embedding_literal, top_k),
        )
        rows = cur.fetchall()

    return [dict(row) for row in rows]
```

### Trade-off & Pitfall
- **Top-K yang terlalu kecil bisa melewatkan chunk relevan yang kebetulan sedikit lebih "jauh" secara similarity** — tapi Top-K yang terlalu besar bikin banyak chunk gak relevan ikut ditempel ke prompt (boros token, dan bisa bikin LLM "terdistraksi" konteks yang gak berguna).
- **Similarity threshold itu perlu di-tuning pakai data nyata** (sama seperti disebut di Phase 3) — angka distance yang "cukup dekat" berbeda-beda tergantung karakteristik dokumen dan model embedding yang dipakai.
- **Vector similarity doang gak selalu menangkap kecocokan literal yang penting** (misal kode error spesifik, nomor tiket, nama produk) — kasus kayak gini butuh hybrid search (gabung full-text search PostgreSQL biasa) supaya kecocokan keyword persis tetap kepegang.
- **Metadata filtering yang salah (misal `source` yang typo) bisa bikin hasil kosong**, walau sebenarnya ada chunk relevan di sumber lain — filtering harus dipastikan sesuai konteks yang benar-benar dibutuhkan.

### Kapan Dipakai
- Pakai `retrieve_relevant_chunks` sebagai pengganti `search_knowledge_base` begitu dokumen sumbernya sudah di-chunk (topik 13-14) — chunk-level retrieval lebih presisi buat dokumen panjang.
- Pakai metadata filtering kalau ada konteks tambahan yang sudah pasti (misal customer lagi buka artikel tertentu, jadi retrieval bisa dipersempit ke `source` itu dulu).
- Pakai hybrid search (keyword + vector) kalau domainnya sering melibatkan istilah/kode yang harus cocok persis (kode error, SKU produk, nomor tiket).
- Top-K yang lebih besar dari kebutuhan akhir (misal ambil 10-20) cocok dipakai kalau tahap berikutnya adalah reranking (topik 16) — retrieve wide dulu, baru dipersempit lagi belakangan.

### Sering Ditanya Saat Interview
- **Apa itu Top-K dalam konteks retrieval?** — mengambil K hasil teratas (paling mirip secara makna) dari pencarian vector similarity, bukan cuma satu, supaya ada beberapa kandidat konteks buat LLM.
- **Kenapa retrieval butuh similarity threshold, gak cukup Top-K aja?** — Top-K selalu mengembalikan hasil "terdekat yang ada", walau semuanya sebenarnya gak relevan sama sekali; threshold mencegah chunk yang beneran gak nyambung ikut ditempel ke prompt.
- **Apa itu hybrid search, dan kenapa dibutuhkan?** — gabungan pencarian keyword (full-text search) dan vector similarity; dibutuhkan karena vector similarity doang kadang gak menangkap kecocokan literal yang penting (kode error, nomor spesifik).
- **Apa beda `retrieve_relevant_chunks` dengan `search_knowledge_base` di Phase 3?** — polanya identik (embedding query lalu ORDER BY jarak pgvector), bedanya cuma granularitas: yang satu di level chunk (tabel `article_chunks`), yang lain di level artikel utuh (tabel `knowledge_articles`).

---

## 16. Reranking

### Apa itu?
Reranking adalah tahap tambahan setelah retrieval: ambil kandidat chunk yang jumlahnya agak banyak (misal Top-20 dari `retrieve_relevant_chunks`), lalu urutkan ULANG kandidat itu pakai model yang lebih akurat (tapi lebih lambat) buat menilai relevansi, dan ambil cuma beberapa teratas dari hasil urutan baru itu (misal Top-3). Pola ini sering disebut **retrieve wide, rerank narrow**: retrieval awal cari banyak kandidat secara cepat (murah tapi kurang presisi), reranking mempersempitnya jadi sedikit tapi lebih presisi (lebih mahal tapi cuma dijalankan ke sedikit kandidat).

### Kenapa dibutuhkan?
Cosine similarity dari pgvector (topik 15) itu cepat karena dia membandingkan vector secara independen — tapi dia gak "membaca" query dan chunk secara bersamaan, cuma membandingkan posisi vector-nya di ruang embedding. Ini kadang menghasilkan urutan yang kurang presisi: chunk yang secara vector "lumayan dekat" belum tentu benar-benar paling relevan kalau dibaca konteksnya bareng-bareng dengan query aslinya.

Model reranking (misalnya cross-encoder) menyelesaikan ini dengan cara membaca **pasangan** (query, chunk) sekaligus dan menghasilkan skor relevansi yang lebih akurat — tapi ini jauh lebih lambat/mahal kalau dijalankan ke SEMUA chunk (makanya gak dipakai buat retrieval awal). Solusinya: retrieval awal (topik 15) mempersempit dari ribuan chunk jadi puluhan kandidat secara cepat, baru reranking mempersempit lagi dari puluhan itu jadi beberapa yang paling relevan secara lebih akurat.

### Cara Kerja
```
Query customer
    → retrieve_relevant_chunks(top_k besar, misal 20) — cepat, kurang presisi
    → rerank_chunks(top_k kecil, misal 3) — lambat tapi lebih presisi
    → chunk final yang ditempel ke prompt LLM
```

Cross-encoder (dari library `sentence-transformers`) bekerja beda dari embedding biasa (yang disebut bi-encoder): bi-encoder meng-embed query dan chunk secara **terpisah** lalu dibandingkan belakangan (itu yang bikin dia bisa di-precompute & cepat dicari lewat pgvector); cross-encoder membaca query dan chunk **bersamaan** dalam satu forward pass model, sehingga bisa menangkap interaksi antara keduanya secara lebih akurat — tapi harus dijalankan ulang buat tiap pasangan (query, chunk), gak bisa di-precompute.

### Contoh Kode — Python
```python
from sentence_transformers import CrossEncoder

# Model cross-encoder kecil yang khusus dilatih buat menilai relevansi query-dokumen
cross_encoder = CrossEncoder("cross-encoder/ms-marco-MiniLM-L-6-v2")


def rerank_chunks(query: str, chunks: list[dict], top_k: int = 3) -> list[dict]:
    """
    Rerank hasil retrieval (list of dict dari retrieve_relevant_chunks) pakai cross-encoder,
    yang menilai relevansi query-chunk lebih akurat dibanding cosine similarity biasa.
    Ambil top_k chunk dengan skor relevansi tertinggi.
    """
    pasangan_query_chunk = [(query, chunk["content"]) for chunk in chunks]
    skor_relevansi = cross_encoder.predict(pasangan_query_chunk)

    chunks_dengan_skor = [
        {**chunk, "rerank_score": float(skor)}
        for chunk, skor in zip(chunks, skor_relevansi)
    ]
    chunks_terurut = sorted(chunks_dengan_skor, key=lambda c: c["rerank_score"], reverse=True)

    return chunks_terurut[:top_k]


# Contoh pemakaian: retrieve wide (top_k=20), lalu rerank narrow (top_k=3)
kandidat = retrieve_relevant_chunks(conn, "berapa lama waktu integrasi API SupportPilot", top_k=20)
chunk_terbaik = rerank_chunks("berapa lama waktu integrasi API SupportPilot", kandidat, top_k=3)

for c in chunk_terbaik:
    print(f"{c['source']} chunk#{c['chunk_index']} (rerank_score={c['rerank_score']:.4f})")
```

### Trade-off & Pitfall
- **Cross-encoder jauh lebih lambat per pasangan dibanding cosine similarity di pgvector** — makanya cuma dijalankan ke kandidat hasil retrieval awal (puluhan), bukan ke SELURUH chunk yang ada (bisa ribuan/jutaan).
- **Reranking nambah satu tahap komputasi lagi ke pipeline** (load model, jalanin inference-nya) — buat kasus di mana retrieval awal sudah cukup akurat (dokumen sedikit, query sederhana), reranking bisa jadi overhead yang gak sepadan manfaatnya.
- **Model cross-encoder yang dipakai harus relevan ke domainnya** — model umum (dilatih di data web umum) belum tentu bisa menilai relevansi istilah spesifik SupportPilot seakurat model yang di-fine-tune khusus buat domain itu.
- **Kalau `top_k` di retrieval awal terlalu kecil, reranking gak bisa "menyelamatkan" chunk relevan yang sudah keburu gak lolos dari tahap retrieval** — reranking cuma bisa mengurutkan ULANG apa yang sudah lolos, bukan menemukan chunk baru yang gak ikut di-retrieve dari awal.

### Kapan Dipakai
- Pakai reranking kalau retrieval awal (vector similarity murni) sering menghasilkan urutan yang kurang presisi — misal chunk yang paling relevan gak selalu nangkring di posisi teratas Top-K.
- Pola retrieve wide (`top_k` besar, misal 15-30) lalu rerank narrow (`top_k` kecil, misal 3-5) adalah kombinasi yang umum: retrieval murah buat mempersempit dari banyak, reranking mahal-tapi-akurat buat mempersempit dari sedikit.
- Kalau latency sangat kritis (butuh respons sub-detik) dan retrieval awal sudah "cukup baik", reranking bisa dilewati demi kecepatan — ini trade-off akurasi vs latency yang perlu diputuskan sesuai kebutuhan produk.

### Sering Ditanya Saat Interview
- **Apa itu pola "retrieve wide, rerank narrow"?** — retrieval awal mengambil banyak kandidat secara cepat (murah, kurang presisi), lalu reranking mempersempitnya jadi sedikit hasil akhir pakai model yang lebih akurat (lebih mahal, tapi cuma dijalankan ke sedikit kandidat).
- **Apa beda bi-encoder (dipakai buat embedding) dan cross-encoder (dipakai buat reranking)?** — bi-encoder meng-embed query dan dokumen terpisah (bisa di-precompute, cepat dicari), cross-encoder membaca keduanya bersamaan dalam satu forward pass (lebih akurat, tapi gak bisa di-precompute, harus dihitung ulang tiap pasangan).
- **Kenapa cross-encoder gak dipakai langsung buat semua chunk sejak awal, tanpa retrieval dulu?** — terlalu lambat/mahal kalau dijalankan ke ribuan/jutaan chunk; cross-encoder cuma efisien dipakai ke sejumlah kecil kandidat yang sudah dipersempit lebih dulu oleh retrieval vector similarity.
- **Bisakah reranking menemukan chunk relevan yang gak ikut ter-retrieve di tahap awal?** — enggak, reranking cuma mengurutkan ulang kandidat yang SUDAH lolos dari retrieval awal; kalau `top_k` retrieval awal kekecilan dan chunk relevannya gak ikut lolos, reranking gak bisa menyelamatkannya.

---

## 17. RAG Failure Modes

### Apa itu?
RAG pipeline punya beberapa titik yang bisa gagal, dan tiap titik kegagalan punya gejala dan penyebab yang beda-beda. Lima jenis kegagalan yang paling umum:

- **Retrieval failure** — chunk yang di-retrieve salah/gak relevan sama sekali ke pertanyaan.
- **Context failure** — chunk yang di-retrieve sebenarnya relevan, tapi konteksnya gak lengkap (misal kepotong di tengah informasi penting).
- **Generation failure** — chunk yang diberikan sudah benar dan lengkap, tapi LLM tetap salah menjawab/mengabaikan konteksnya.
- **Chunking failure** — chunk yang dihasilkan dari dokumen sumber sudah "cacat" dari awal (mencampur topik, motong di tempat yang salah), sehingga baik retrieval maupun generation gak akan bisa menyelamatkan hasilnya.
- **Embedding failure** — model embedding gagal menangkap kemiripan makna yang seharusnya jelas (misal karena istilah domain-spesifik yang gak dikenali baik oleh model embedding umum).

### Kenapa dibutuhkan?
Kalau RAG pipeline menghasilkan jawaban yang salah, penyebabnya bisa di banyak tempat berbeda — dan cara memperbaikinya beda-beda tergantung di mana letak kegagalannya. Kalau masalahnya di chunking (chunk yang dihasilkan memang sudah cacat), memperbaiki prompt LLM (generation) gak akan menyelesaikan apa-apa; sebaliknya kalau masalahnya di generation (LLM ngabaikan konteks yang sudah benar), menambah lebih banyak chunk hasil retrieval juga gak akan membantu. Memahami kategori-kategori ini membantu men-debug RAG pipeline secara lebih terarah, bukan asal coba-coba ganti parameter.

### Cara Kerja
Posisi tiap failure mode dalam pipeline:
```
Dokumen sumber
    → [chunking failure bisa terjadi di sini]
    → Chunk + embedding
    → [embedding failure bisa terjadi di sini]
    → Query customer → retrieval
    → [retrieval failure bisa terjadi di sini]
    → Chunk yang di-retrieve
    → [context failure bisa terjadi di sini]
    → LLM generate jawaban
    → [generation failure bisa terjadi di sini]
    → Jawaban akhir
```

### Contoh Kode — Python
Contoh singkat masing-masing failure mode buat SupportPilot (bukan pipeline lengkap, cuma ilustrasi konkret tiap kasus):

**Retrieval failure** — pertanyaan pakai istilah yang beda dari isi chunk, hasil retrieve jadi gak relevan:
```python
# Chunk yang tersimpan isinya soal "kebijakan refund", tapi ditulis tanpa kata "uang kembali"
hasil = retrieve_relevant_chunks(conn, "duitku kapan balik ya kalau barangnya aku retur", top_k=3)
# kalau embedding gak cukup kuat menangkap "duit balik" ~ "refund", hasil retrieve bisa aja
# malah didominasi chunk soal topik lain yang gak relevan sama sekali
```

**Context failure** — chunk relevan, tapi kepotong sebelum informasi pentingnya (chunk_size kekecilan):
```python
chunk_terpotong = Chunk(
    text="Refund diproses dalam 3-5 hari kerja setelah barang diterima gudang, KECUALI",
    source="kebijakan-refund.txt",
    chunk_index=2,
)
# kalimat "KECUALI ..." (pengecualian penting) ada di chunk BERIKUTNYA yang gak ikut ke-retrieve
# akibatnya LLM menjawab seolah gak ada pengecualian, padahal sebenarnya ada
```

**Generation failure** — konteks yang diberikan sudah benar dan lengkap, tapi LLM tetap salah menjawab:
```python
konteks_benar = "Refund diproses dalam 3-5 hari kerja setelah barang diterima gudang."
prompt_tanpa_instruksi_tegas = f"{konteks_benar}\n\nCustomer: kapan refund saya cair?"
# tanpa instruksi eksplisit "jawab HANYA berdasarkan konteks ini", LLM kadang tetap
# menambahkan asumsi sendiri (misal ngasih estimasi tanggal pasti, padahal konteksnya cuma bilang "3-5 hari kerja")
```

**Chunking failure** — dua topik berbeda tercampur di satu chunk karena batas potongnya sembarangan:
```python
chunk_campur_topik = Chunk(
    text=(
        "...refund diproses 3-5 hari kerja. Untuk reset password, klik lupa password "
        "di halaman login lalu ikuti instruksi di email..."
    ),
    source="artikel-gabungan.txt",
    chunk_index=5,
)
# satu chunk ini "relevan sebagian" ke DUA pertanyaan berbeda (refund & reset password),
# tapi gak benar-benar fokus ke salah satunya -> retrieval jadi kurang presisi buat keduanya
```

**Embedding failure** — istilah domain-spesifik yang gak dikenali baik oleh model embedding umum:
```python
# "SLA" di konteks SupportPilot berarti durasi maksimum penyelesaian tiket,
# tapi model embedding umum belum tentu menangkap ini seakurat istilah umum sehari-hari
embedding_pertanyaan = generate_embedding("berapa lama SLA penyelesaian tiket saya")
embedding_artikel_durasi = generate_embedding("waktu maksimum penanganan tiket customer")
# secara makna keduanya identik buat orang SupportPilot, tapi similarity-nya bisa aja
# gak setinggi yang diharapkan kalau model embedding-nya gak familiar sama istilah "SLA"
```

### Trade-off & Pitfall
- **Gejala di permukaan (jawaban salah) bisa punya banyak kemungkinan akar penyebab** — jangan langsung asumsikan letak masalahnya tanpa mengecek tiap tahap pipeline satu-satu (chunk yang tersimpan, hasil retrieval, konteks yang ditempel ke prompt, baru jawaban akhirnya).
- **Menambah lebih banyak chunk (Top-K besar) gak menyelesaikan context failure atau chunking failure** — kalau chunk-nya sendiri sudah cacat/terpotong, menambah kuantitas chunk yang salah gak membantu.
- **Chunking failure paling sering "menyamar" jadi retrieval failure** — kelihatannya seperti "retrieval salah nemuin chunk", padahal akar masalahnya ada di chunk yang memang sudah tercampur topik dari awal, jauh sebelum tahap retrieval terjadi.
- **Embedding failure sulit dideteksi tanpa uji langsung** — butuh mencoba beberapa pasangan query-dokumen yang seharusnya mirip secara makna, dan cek apakah similarity-nya memang tinggi seperti yang diharapkan.

### Kapan Dipakai
- Pakai kerangka lima failure mode ini setiap kali RAG pipeline SupportPilot menghasilkan jawaban yang salah, buat mempersempit "di tahap mana" letak masalahnya sebelum mencoba memperbaiki apa pun.
- Kalau jawabannya salah tapi chunk yang di-retrieve sudah relevan dan lengkap — curigai generation failure (perbaiki instruksi prompt).
- Kalau chunk yang di-retrieve gak relevan sama sekali — cek dulu apakah chunk yang seharusnya relevan memang ADA di database (chunking/ingestion), baru curigai retrieval/embedding kalau memang sudah ada tapi gak ketemu.
- Evaluasi sistematis (topik 18) membantu mendeteksi pola-pola failure mode ini secara berulang, bukan cuma dari satu-dua kasus yang kebetulan ketemu.

### Sering Ditanya Saat Interview
- **Sebutkan lima failure mode utama dalam RAG pipeline.** — retrieval failure (chunk salah/gak relevan ke-retrieve), context failure (chunk relevan tapi gak lengkap), generation failure (LLM salah walau konteksnya benar), chunking failure (chunk sudah cacat dari awal), embedding failure (model gagal menangkap kemiripan makna yang seharusnya jelas).
- **Kenapa chunking failure sering disalahartikan sebagai retrieval failure?** — gejalanya sama-sama "chunk yang ke-retrieve kelihatan gak relevan", padahal akar masalahnya bisa jadi ada di chunk yang sudah tercampur topik sejak proses chunking, jauh sebelum retrieval berjalan.
- **Kalau chunk yang di-retrieve sudah benar tapi jawaban LLM tetap salah, di tahap mana masalahnya?** — generation failure — perlu perbaiki instruksi prompt (misal tegaskan "jawab HANYA berdasarkan konteks ini"), bukan mengubah retrieval/chunking.
- **Kenapa menambah Top-K gak selalu memperbaiki kualitas jawaban RAG?** — kalau masalahnya di chunking (chunk cacat) atau context (chunk kepotong), menambah jumlah chunk yang sama-sama bermasalah gak menyelesaikan akar masalahnya.

---

## 18. RAG Evaluation

### Apa itu?
RAG evaluation adalah cara mengukur seberapa baik RAG pipeline bekerja secara sistematis (bukan cuma "coba beberapa pertanyaan terus lihat kelihatannya oke atau enggak"), pakai beberapa metrik:

- **Retrieval precision/recall** — precision: dari chunk yang di-retrieve, berapa persen yang beneran relevan; recall: dari semua chunk yang seharusnya relevan, berapa persen yang berhasil ke-retrieve.
- **Context relevance** — seberapa relevan konteks yang ditempel ke prompt terhadap pertanyaan yang diajukan.
- **Faithfulness** — seberapa "setia" jawaban LLM terhadap konteks yang diberikan (apakah dia ngarang hal yang gak ada di konteks, atau murni menyimpulkan dari konteks itu).
- **Answer correctness** — seberapa benar jawaban akhirnya dibanding jawaban yang seharusnya (ground truth).
- **Latency** — berapa lama waktu total pipeline (retrieve + rerank + generate) buat menghasilkan satu jawaban.
- **Cost** — total biaya API (embedding + reranking + LLM call) buat menghasilkan satu jawaban.

### Kenapa dibutuhkan?
Tanpa evaluasi sistematis, perubahan ke pipeline (misal ganti `chunk_size`, ganti model reranking, ganti `top_k`) cuma bisa dinilai secara subjektif ("kelihatannya lebih bagus") — ini gak reliable buat keputusan yang mempengaruhi banyak customer sekaligus. Dengan metrik yang jelas dan test case yang konsisten, kita bisa membandingkan versi pipeline yang berbeda secara objektif: apakah perubahan `chunk_size` dari 500 ke 300 karakter benar-benar menaikkan retrieval hit rate, atau malah menurunkannya?

Evaluasi juga membantu mendeteksi failure mode (topik 17) secara berulang dan terukur, bukan cuma nemuin satu-dua kasus gagal secara kebetulan pas testing manual.

### Cara Kerja
Alur evaluasi: siapkan kumpulan test case (`{"query": ..., "expected_answer": ...}`), jalankan pipeline lengkap buat tiap test case, lalu bandingkan hasilnya dengan `expected_answer`:
```
test_cases (query + expected_answer)
    → untuk tiap test case: retrieve_relevant_chunks → rerank_chunks → LLM generate jawaban
    → bandingkan jawaban model vs expected_answer, dan cek apakah chunk relevan ke-retrieve
    → agregat jadi metrik: retrieval hit rate, answer correctness, latency rata-rata
```

Versi sederhana di sini pakai **keyword overlap** buat mengukur retrieval hit rate & answer correctness (bukan metrik canggih berbasis LLM-judge/embedding similarity) — cukup buat memberi sinyal kasar, dan gampang dipahami tanpa dependency tambahan.

### Contoh Kode — Python
```python
import time


def evaluate_rag_pipeline(test_cases: list[dict]) -> dict:
    """
    Jalankan RAG pipeline penuh (retrieve -> rerank -> LLM generate) buat tiap test case,
    lalu hitung metrik sederhana: retrieval hit rate, answer correctness (keyword overlap),
    dan latency rata-rata.

    test_cases: list of {"query": str, "expected_answer": str}

    Catatan: `conn` di sini memakai koneksi database yang sama yang sudah dibuka
    di awal (module-level), sama seperti contoh-contoh sebelumnya di phase ini.
    """
    total_kasus = len(test_cases)
    retrieval_hits = 0
    correctness_scores = []
    total_latency = 0.0

    for kasus in test_cases:
        query = kasus["query"]
        expected_answer = kasus["expected_answer"]
        kata_kunci = [kata for kata in expected_answer.lower().split() if kata]

        mulai = time.perf_counter()

        kandidat = retrieve_relevant_chunks(conn, query, top_k=10)
        chunk_terpilih = rerank_chunks(query, kandidat, top_k=3)

        # retrieval hit rate: apakah ADA chunk terpilih yang mengandung kata kunci dari expected_answer
        chunk_relevan_ditemukan = any(
            any(kata in chunk["content"].lower() for kata in kata_kunci)
            for chunk in chunk_terpilih
        )
        if chunk_relevan_ditemukan:
            retrieval_hits += 1

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

        selesai = time.perf_counter()
        total_latency += selesai - mulai

        # answer correctness sederhana: proporsi kata kunci expected_answer yang muncul di jawaban model
        if kata_kunci:
            kata_muncul = sum(1 for kata in kata_kunci if kata in jawaban_model.lower())
            skor_correctness = kata_muncul / len(kata_kunci)
        else:
            skor_correctness = 0.0
        correctness_scores.append(skor_correctness)

    return {
        "jumlah_test_case": total_kasus,
        "retrieval_hit_rate": retrieval_hits / total_kasus,
        "avg_answer_correctness": sum(correctness_scores) / total_kasus,
        "avg_latency_seconds": total_latency / total_kasus,
    }


# Contoh pemakaian
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

hasil_evaluasi = evaluate_rag_pipeline(test_cases)
print(hasil_evaluasi)
# contoh output: {'jumlah_test_case': 2, 'retrieval_hit_rate': 1.0,
#                 'avg_answer_correctness': 0.85, 'avg_latency_seconds': 1.42}
```

### Trade-off & Pitfall
- **Keyword overlap itu metrik kasar, bukan pengukuran makna yang sesungguhnya** — jawaban yang benar secara makna tapi ditulis pakai kata-kata berbeda dari `expected_answer` bisa dinilai "salah" secara keliru; metrik yang lebih canggih (embedding similarity, atau LLM-as-judge) lebih akurat tapi lebih mahal & kompleks buat dijalankan.
- **Test case yang sedikit/gak representatif menghasilkan metrik yang menyesatkan** — kalau cuma ada 2-3 test case, hasil evaluasi gampang "kebetulan bagus/jelek", gak cukup buat disimpulkan generalisasinya ke seluruh traffic customer.
- **Evaluasi ini manggil LLM beneran buat tiap test case** — biayanya bertambah linear sesuai jumlah test case; kalau evaluasi dijalankan tiap kali ada perubahan kecil ke pipeline, biaya ini perlu dipertimbangkan.
- **Faithfulness dan context relevance gak dihitung eksplisit di versi sederhana ini** — versi production biasanya menambahkan metrik ini juga (sering dibantu LLM-as-judge, yaitu minta LLM lain menilai "apakah jawaban ini setia ke konteks yang diberikan"), yang butuh setup lebih kompleks dibanding keyword overlap.

### Kapan Dipakai
- Jalankan `evaluate_rag_pipeline` setiap kali ada perubahan ke komponen pipeline (`chunk_size`, model reranking, prompt template, dll), buat memastikan perubahan itu beneran memperbaiki hasilnya, bukan cuma "kelihatannya lebih bagus" secara subjektif.
- Bangun test case dari pertanyaan customer nyata yang sudah pernah masuk (dengan jawaban yang sudah diverifikasi benar oleh manusia), supaya evaluasinya representatif ke traffic sesungguhnya.
- Kalau butuh keputusan yang lebih presisi (misal mau tau apakah masalahnya di retrieval atau generation, sesuai failure mode topik 17), tambahkan metrik retrieval-only (precision/recall di level chunk) terpisah dari metrik end-to-end (answer correctness).
- Latency & cost penting dipantau bareng-bareng dengan metrik kualitas — pipeline yang lebih akurat tapi terlalu lambat/mahal buat production tetap perlu dipertimbangkan ulang trade-off-nya.

### Sering Ditanya Saat Interview
- **Sebutkan beberapa metrik utama buat evaluasi RAG pipeline.** — retrieval precision/recall, context relevance, faithfulness, answer correctness, latency, dan cost.
- **Apa beda retrieval precision dan recall dalam konteks RAG?** — precision: dari chunk yang di-retrieve, berapa persen yang beneran relevan; recall: dari semua chunk yang seharusnya relevan, berapa persen yang berhasil ke-retrieve.
- **Apa itu faithfulness dalam evaluasi RAG, dan kenapa penting?** — mengukur seberapa setia jawaban LLM terhadap konteks yang diberikan (apakah dia ngarang hal yang gak ada di konteks); penting karena RAG yang "grounded" tapi tetap hallucinate berarti manfaat retrieval-nya gak kepakai maksimal.
- **Kenapa evaluasi RAG harus otomatis/sistematis, bukan cuma dicoba manual sekali-sekali?** — supaya perubahan ke pipeline (chunking, retrieval, reranking, prompt) bisa dibandingkan secara objektif dan konsisten, dan supaya failure mode yang berulang bisa terdeteksi, bukan cuma kebetulan ketemu di satu-dua kasus uji manual.

---

**Selanjutnya:** [Phase 05 — LLM Application Architecture](./phase-05-llm-application-architecture.md)
