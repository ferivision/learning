# Phase 03 — Embeddings

> Bagian dari [AI Engineering / Agent Roadmap](../README.md)

---

## 9. Embeddings

### Apa itu?
Embedding adalah representasi teks dalam bentuk **vector angka** (daftar float, misalnya 1536 angka) yang menangkap makna semantik dari teks itu. Ide dasarnya: teks yang artinya mirip bakal menghasilkan vector yang "posisinya" berdekatan di ruang vector (bayangin ruang berdimensi ribuan, bukan cuma 2D/3D kayak grafik biasa), sementara teks yang artinya jauh berbeda bakal menghasilkan vector yang berjauhan. Ini beda total dari sekadar "cocokin kata per kata" (keyword matching) — dua kalimat bisa gak punya satu kata pun yang sama persis, tapi tetap dianggap mirip secara makna kalau embedding-nya berdekatan.

Model yang menghasilkan embedding ini biasanya berbasis arsitektur **encoder** (lihat Phase 1, topik 2) — dia baca seluruh teks sekaligus secara bidirectional buat memahami makna utuhnya, terus meringkasnya jadi satu vector angka dengan panjang tetap, berapa pun panjang teks aslinya (selama masih di bawah limit token model embedding-nya).

### Kenapa dibutuhkan?
SupportPilot punya banyak `KnowledgeArticle` (artikel help-center) yang isinya macem-macem — cara reset password, kebijakan refund, cara lacak pengiriman, dll. Kalau customer nanya pakai kalimatnya sendiri (misal "gimana caranya ganti kata sandi ya"), pencarian keyword biasa (`LIKE '%password%'` atau full-text search PostgreSQL) gampang meleset karena customer gak selalu pakai kata yang persis sama kayak di judul/isi artikel ("kata sandi" vs "password", "lacak" vs "tracking").

Embedding menyelesaikan masalah ini dengan mengubah pencarian dari "cocokin kata" jadi "cocokin makna" — baik pertanyaan customer maupun tiap artikel help-center diubah ke vector, terus kita cari artikel yang vector-nya paling dekat dengan vector pertanyaan. Ini juga fondasi buat RAG (Phase 4): sebelum bisa "kasih LLM konteks dokumen yang relevan", kita harus bisa dulu **nemuin** dokumen mana yang relevan — dan embedding adalah cara paling umum buat itu.

### Cara Kerja
Alur dasarnya:
```
Teks mentah → Model Embedding → Vector angka (misal 1536 dimensi) → disimpan/dibandingkan
```

Beberapa istilah kunci:
- **Vector** — daftar angka float dengan panjang tetap (misal `[0.012, -0.034, 0.521, ...]` sepanjang 1536 elemen). Panjang ini disebut **dimensi** embedding, dan nilainya tetap sama buat semua output dari satu model embedding tertentu.
- **Semantic similarity** — kemiripan makna (bukan kemiripan tulisan/karakter). Ini yang diukur lewat jarak/similarity antar vector (dibahas detail di topik 10).
- **Embedding model** — model khusus (biasanya encoder-only) yang dilatih supaya teks dengan makna mirip menghasilkan vector yang berdekatan. Contoh yang umum dipakai:
  - **OpenAI `text-embedding-3-small`** — model hosted lewat API, gampang dipakai, gak perlu infrastruktur sendiri. Menghasilkan vector **1536 dimensi**. Ini yang kita pakai sebagai model utama di phase ini.
  - **`sentence-transformers`** (misal model `all-MiniLM-L6-v2`) — library open-source yang jalan lokal (gak perlu API call/koneksi internet), cocok kalau data sensitif gak boleh keluar ke API pihak ketiga, atau kalau mau hindari biaya per-panggilan API. Model ini biasanya menghasilkan vector dengan dimensi lebih kecil (misal 384), tergantung model spesifiknya.

Karena `generate_embedding` bakal kita pakai berulang kali di topik 10 dan 11 (buat artikel help-center maupun pertanyaan customer), kita bungkus jadi satu fungsi supaya konsisten model & parameternya di semua tempat.

### Contoh Kode — Python
Kita bikin `generate_embedding(text)` yang membungkus panggilan ke embeddings endpoint OpenAI, lalu coba di dua snippet artikel help-center SupportPilot yang beda topik:

```python
from openai import OpenAI

client = OpenAI()  # baca API key dari environment variable OPENAI_API_KEY


def generate_embedding(text: str) -> list[float]:
    """
    Ubah teks jadi vector angka yang merepresentasikan makna semantiknya.
    Model text-embedding-3-small menghasilkan vector 1536 dimensi.
    """
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=text,
    )
    # response.data adalah list (satu elemen per input teks); kita cuma kirim satu teks,
    # jadi ambil elemen pertama, lalu ambil field .embedding-nya
    return response.data[0].embedding


# Dua snippet artikel help-center SupportPilot dengan topik berbeda
artikel_reset_password = (
    "Cara reset password akun SupportPilot: buka halaman login, klik 'Lupa Password', "
    "lalu ikuti instruksi yang dikirim ke email terdaftar."
)
artikel_kebijakan_refund = (
    "Kebijakan refund SupportPilot: pengajuan refund diproses dalam 3-5 hari kerja "
    "setelah barang diterima kembali oleh gudang."
)

embedding_reset_password = generate_embedding(artikel_reset_password)
embedding_kebijakan_refund = generate_embedding(artikel_kebijakan_refund)

print(len(embedding_reset_password))  # 1536
print(len(embedding_kebijakan_refund))  # 1536
```

### Trade-off & Pitfall
- **API embedding tetap kena biaya per panggilan** (walau jauh lebih murah dibanding panggilan chat completion) — kalau punya ribuan artikel yang sering di-update, biaya ini perlu diperhitungkan, dan idealnya embedding artikel di-generate ulang cuma kalau isinya berubah, bukan tiap kali dibaca.
- **Model embedding beda = dimensi & "ruang vector" beda, gak bisa dicampur.** Vector dari `text-embedding-3-small` gak bisa dibandingkan langsung dengan vector dari `sentence-transformers` — keduanya punya dimensi berbeda dan "pemetaan makna ke angka" yang berbeda. Kalau ganti model embedding, semua embedding yang sudah tersimpan harus di-generate ulang.
- **`sentence-transformers` (lokal) adalah alternatif yang valid** kalau data sensitif gak boleh keluar ke API pihak ketiga, atau kalau volume panggilan sangat tinggi sehingga biaya API jadi mahal — trade-off-nya butuh compute sendiri (CPU/GPU) buat generate embedding, dan biasanya kualitas semantiknya sedikit di bawah model hosted besar untuk kasus umum.
- **Teks yang terlalu panjang tetap kena limit token model embedding** (mirip limit context window di Phase 1) — dokumen panjang biasanya perlu dipotong jadi beberapa chunk dulu sebelum di-embed (detail chunking ada di Phase 4).

### Kapan Dipakai
- Pakai embedding tiap kali butuh **mencari/mengelompokkan teks berdasarkan makna**, bukan berdasarkan kecocokan kata literal — contoh utama: pencarian artikel help-center, deteksi tiket duplikat, rekomendasi artikel terkait.
- Pilih **OpenAI embeddings API** kalau mau setup cepat, gak masalah kirim data ke API pihak ketiga, dan volume panggilannya wajar.
- Pilih **`sentence-transformers`** kalau butuh jalan lokal/self-hosted (data sensitif, kebutuhan privacy, atau ingin menghindari biaya per-panggilan API di volume sangat tinggi).
- Ini adalah building block dasar buat topik 10 (Vector Similarity) dan topik 11 (Vector Database) — dan fondasi utama buat RAG di Phase 4.

### Sering Ditanya Saat Interview
- **Apa itu embedding, dan kenapa beda dari keyword matching?** — representasi teks sebagai vector angka yang menangkap makna semantik; teks yang maknanya mirip menghasilkan vector yang berdekatan, walau kata-katanya beda persis — beda dari keyword matching yang cuma cocokin kata secara literal.
- **Kenapa embedding biasanya pakai model encoder, bukan decoder?** — encoder membaca seluruh teks secara bidirectional buat memahami makna utuhnya sekaligus, cocok buat "meringkas" makna jadi satu vector, sementara decoder didesain buat generate teks token demi token secara causal.
- **Apa yang terjadi kalau ganti model embedding di tengah jalan?** — semua embedding yang sudah tersimpan jadi gak valid buat dibandingkan dengan embedding baru, karena dimensi dan "ruang vector"-nya beda antar model; harus di-generate ulang semua.
- **Kapan sebaiknya pakai model embedding lokal (`sentence-transformers`) dibanding API hosted?** — kalau data sensitif gak boleh keluar ke pihak ketiga, atau volume panggilannya sangat tinggi sehingga biaya API jadi mahal; trade-off-nya butuh infrastruktur compute sendiri.

---

## 10. Vector Similarity

### Apa itu?
Vector similarity adalah cara mengukur **seberapa dekat/mirip** dua vector satu sama lain secara matematis — dan karena embedding merepresentasikan makna semantik (topik 9), mengukur kedekatan dua vector embedding artinya mengukur kedekatan makna dua teks aslinya. Ada beberapa cara ukur yang umum dipakai:

- **Cosine similarity** — mengukur sudut antara dua vector (bukan panjangnya). Nilainya antara -1 sampai 1: makin dekat ke 1, makin mirip arahnya (makin mirip maknanya); makin dekat ke 0, makin gak berhubungan; negatif berarti berlawanan makna. Ini metrik paling umum dipakai buat embedding teks.
- **Dot product** — perkalian elemen-per-elemen lalu dijumlahkan. Mirip cosine similarity, tapi juga dipengaruhi **panjang/magnitude** vector, bukan cuma arahnya — jadi kurang cocok kalau vector-vector-nya gak dinormalisasi ke panjang yang sama.
- **Euclidean distance** — jarak garis lurus antar dua titik vector di ruang berdimensi banyak (rumus Pythagoras yang diperluas). Ini mengukur **jarak**, bukan **kemiripan** — makin kecil nilainya, makin dekat/mirip (kebalikan dari cosine similarity yang makin besar makin mirip).

### Kenapa dibutuhkan?
Setelah teks diubah jadi vector (topik 9), kita butuh cara buat **membandingkan** vector-vector itu supaya bisa jawab pertanyaan konkret: "dari semua artikel help-center yang ada, mana yang paling relevan sama pertanyaan customer ini?" Tanpa metrik similarity yang jelas, kita cuma punya kumpulan angka tanpa cara buat menentukan mana yang "dekat" satu sama lain. Cosine similarity dipilih jadi metrik paling umum buat embedding teks karena dia fokus ke **arah** vector (representasi makna), bukan magnitude-nya — dua teks bisa aja punya "kekuatan sinyal" yang beda tapi makna yang sama persis.

### Cara Kerja
Rumus cosine similarity antara vector `A` dan `B`:
```
cosine_similarity(A, B) = dot_product(A, B) / (magnitude(A) × magnitude(B))
```
di mana:
- `dot_product(A, B)` = jumlah dari perkalian tiap elemen yang posisinya sama (`A[0]*B[0] + A[1]*B[1] + ...`).
- `magnitude(vector)` = akar dari jumlah kuadrat tiap elemen vector itu (panjang vector secara geometris, rumus Pythagoras diperluas ke banyak dimensi).

Pembagian dengan magnitude inilah yang bikin cosine similarity gak peduli sama "panjang" vector — dia menormalisasi dot product supaya hasilnya murni ngukur kemiripan **arah**, sehingga hasil akhirnya selalu berada di rentang -1 sampai 1 terlepas dari seberapa besar angka-angka di dalam vector aslinya.

### Contoh Kode — Python
Implementasi `cosine_similarity` pakai `math` bawaan Python (gak butuh library eksternal seperti numpy):

```python
import math


def cosine_similarity(vec_a: list[float], vec_b: list[float]) -> float:
    """
    Hitung cosine similarity antara dua vector.
    Hasilnya antara -1 (berlawanan makna) sampai 1 (makna identik).
    """
    # dot product: jumlah perkalian elemen yang posisinya sama
    dot_product = sum(a * b for a, b in zip(vec_a, vec_b))

    # magnitude (panjang vector): akar dari jumlah kuadrat tiap elemen
    magnitude_a = math.sqrt(sum(a * a for a in vec_a))
    magnitude_b = math.sqrt(sum(b * b for b in vec_b))

    if magnitude_a == 0 or magnitude_b == 0:
        # vector nol (semua elemennya 0) gak punya "arah", similarity gak terdefinisi
        return 0.0

    return dot_product / (magnitude_a * magnitude_b)
```

Sekarang kita buktikan klaim di topik 9: dua artikel yang maknanya mirip (sama-sama soal reset password, tapi ditulis beda kalimat) harus punya similarity yang jauh lebih tinggi dibanding dua artikel yang topiknya beda:

```python
# artikel_reset_password & embedding_reset_password dari topik 9 di atas
artikel_reset_password_v2 = (
    "Kalau lupa password, klik tombol 'Lupa Password' di halaman login untuk reset "
    "lewat email."
)
embedding_reset_password_v2 = generate_embedding(artikel_reset_password_v2)

similarity_topik_sama = cosine_similarity(embedding_reset_password, embedding_reset_password_v2)
similarity_topik_beda = cosine_similarity(embedding_reset_password, embedding_kebijakan_refund)

print(f"Similarity (sama-sama soal reset password): {similarity_topik_sama:.4f}")
print(f"Similarity (reset password vs kebijakan refund): {similarity_topik_beda:.4f}")
# diharapkan: similarity_topik_sama jauh lebih tinggi dari similarity_topik_beda
# (walau dua kalimat reset password itu gak ada satu kata pun yang identik persis)
```

### Trade-off & Pitfall
- **Cosine similarity mengabaikan magnitude vector** — ini biasanya diinginkan buat embedding teks (fokus ke makna, bukan "kekuatan sinyal"), tapi bukan pilihan tepat buat semua jenis data vector (misal data numerik mentah yang magnitude-nya justru informatif).
- **Menghitung similarity satu-satu di application code (loop Python) gak scalable.** Kalau SupportPilot punya ribuan/jutaan artikel, membandingkan satu pertanyaan customer ke SEMUA artikel pakai loop `for` dan `cosine_similarity` satu-satu bakal lambat banget — ini alasan kenapa topik 11 (vector database) dibutuhkan buat skala production.
- **Threshold "cukup mirip" itu gak universal.** Similarity 0.8 mungkin "sangat mirip" buat satu domain, tapi cuma "agak mirip" buat domain lain — threshold biasanya perlu di-tuning berdasarkan data nyata, bukan angka baku yang berlaku di semua kasus.
- **Cosine similarity dan euclidean distance bisa menghasilkan urutan ranking yang beda**, terutama kalau vector-vector-nya belum dinormalisasi ke panjang yang sama — penting buat konsisten pakai satu metrik yang sama di seluruh sistem (jangan campur pemakaian keduanya buat kasus yang sama).

### Kapan Dipakai
- Pakai **cosine similarity** sebagai default buat membandingkan embedding teks — ini yang paling umum dan biasanya paling cocok buat kasus semantic search.
- Pakai **dot product** kalau embedding sudah dinormalisasi ke panjang 1 (unit vector) — dalam kondisi ini, dot product dan cosine similarity menghasilkan urutan ranking yang identik, tapi dot product sedikit lebih murah secara komputasi (skip langkah pembagian magnitude).
- Pakai **euclidean distance** kalau memang kebutuhannya benar-benar "jarak" di ruang vector (misal clustering data numerik), bukan kemiripan makna teks.
- Kalau jumlah data yang dibandingkan sudah banyak (ribuan+) dan butuh performa, jangan hitung similarity manual di application code — lanjut ke topik 11 (vector database).

### Sering Ditanya Saat Interview
- **Apa itu cosine similarity, dan kenapa jadi metrik default buat embedding teks?** — mengukur sudut/arah antara dua vector, hasilnya -1 sampai 1; jadi default karena fokus ke kemiripan arah (makna), gak terpengaruh magnitude vector.
- **Apa beda cosine similarity dan euclidean distance?** — cosine similarity mengukur kemiripan sudut/arah (makin besar makin mirip), euclidean distance mengukur jarak garis lurus antar titik (makin kecil makin mirip) — keduanya "berlawanan arah" secara interpretasi nilai.
- **Kapan dot product setara dengan cosine similarity?** — kalau kedua vector sudah dinormalisasi ke panjang (magnitude) 1; dalam kondisi itu, rankingnya identik dan dot product lebih murah secara komputasi.
- **Kenapa menghitung similarity satu-satu di application code gak scalable buat production?** — karena butuh membandingkan satu vector ke semua vector lain satu-satu (linear scan), yang jadi lambat kalau datanya banyak — solusinya pakai index approximate nearest neighbor di vector database (topik 11).

---

## 11. Vector Database (pgvector)

### Apa itu?
Vector database adalah database yang dioptimalkan buat menyimpan vector (embedding) dan mencari vector-vector yang paling mirip dengan sebuah vector query, secara **cepat** walau datanya jutaan baris — pakai struktur index khusus (approximate nearest neighbor / ANN), bukan cuma linear scan bandingin satu-satu kayak di topik 10. **pgvector** adalah extension resmi buat PostgreSQL yang menambahkan tipe data `vector` beserta operator jarak dan index ANN — jadi database yang lu udah kenal (PostgreSQL) bisa langsung dipakai jadi vector database, tanpa perlu adopsi database terpisah yang baru.

### Kenapa dibutuhkan?
Di topik 10 udah kelihatan masalahnya: menghitung `cosine_similarity` satu-satu di application code (loop Python) buat semua artikel itu gak scalable — tiap kali ada pertanyaan customer baru, kita harus loop ke SEMUA baris `KnowledgeArticle` dan hitung similarity-nya satu-satu (linear scan, kompleksitas O(n)). Kalau artikelnya cuma puluhan, ini masih kerasa cepat; begitu jumlahnya ribuan atau jutaan, ini jadi lambat dan gak praktis buat dipakai di request customer secara real-time.

Vector database (lewat index ANN) menyelesaikan ini dengan struktur data khusus yang bisa mempersempit ruang pencarian jauh lebih cepat dari linear scan — **approximate** artinya dia gak menjamin hasil 100% paling akurat secara matematis (beda tipis kemungkinan ada), tapi trade-off ini sepadan dengan lonjakan kecepatan yang didapat, dan di praktiknya akurasinya tetap sangat tinggi. pgvector jadi pilihan natural buat SupportPilot karena kita udah pakai PostgreSQL buat `Customer`, `Order`, `Ticket`, dll — gak perlu operasikan database terpisah cuma buat fitur pencarian semantik ini.

### Cara Kerja
Tiga bagian yang perlu dipahami:

- **Vector column** — kolom di tabel PostgreSQL dengan tipe data `vector(N)`, di mana `N` adalah dimensi embedding-nya (harus konsisten dengan model embedding yang dipakai — kita pakai `text-embedding-3-small` di topik 9 yang menghasilkan 1536 dimensi, jadi kolomnya `vector(1536)`).
- **Distance operator** — pgvector menyediakan operator SQL khusus buat menghitung jarak/similarity langsung di query: `<->` (euclidean distance), `<#>` (negative inner product), dan `<=>` (cosine distance, yaitu `1 - cosine_similarity`). Karena topik 10 kita pakai cosine similarity sebagai metrik default, di sini kita pakai `<=>` (cosine distance) secara konsisten.
- **ANN index** — index khusus (misalnya `ivfflat` atau `hnsw`) yang dibangun di atas vector column, supaya query `ORDER BY ... LIMIT k` gak perlu scan semua baris satu-satu — mirip konsepnya kayak B-tree index mempercepat `WHERE`/`ORDER BY` di kolom biasa, tapi ANN index ini didesain khusus buat ruang vector berdimensi banyak.

Skema tabel `knowledge_articles` beserta index ANN-nya:
```sql
-- Aktifkan extension pgvector di database (sekali per database)
CREATE EXTENSION IF NOT EXISTS vector;

-- Tabel artikel help-center SupportPilot, dengan kolom embedding bertipe vector
CREATE TABLE knowledge_articles (
    id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    -- dimensi 1536 harus sama persis dengan output model embedding yang dipakai
    -- (text-embedding-3-small dari topik 9)
    embedding VECTOR(1536) NOT NULL
);

-- Index ANN (ivfflat) buat mempercepat pencarian nearest-neighbor pakai cosine distance.
-- "lists" adalah jumlah cluster yang dipakai ivfflat buat mempersempit pencarian —
-- rule of thumb umum: lists ≈ sqrt(jumlah baris), disesuaikan lagi seiring data bertambah.
CREATE INDEX knowledge_articles_embedding_idx
ON knowledge_articles
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

Alur pencarian di `search_knowledge_base`:
```
Pertanyaan customer → generate_embedding() → query vector
    → SQL: ORDER BY embedding <=> query_vector LIMIT top_k
    → top_k artikel paling mirip secara makna
```

### Contoh Kode — Python
`search_knowledge_base` menerima koneksi database (`conn`, psycopg2 connection biasa — reader udah familiar sama pola ini dari kerjaan PostgreSQL sehari-hari), pertanyaan customer dalam bentuk teks, dan `top_k` (berapa banyak artikel teratas yang mau diambil):

```python
import psycopg2
import psycopg2.extras


def search_knowledge_base(conn, query: str, top_k: int = 5) -> list[dict]:
    """
    Cari top_k artikel help-center SupportPilot yang paling mirip secara makna
    dengan `query` (pertanyaan customer), pakai pgvector cosine distance (<=>).
    """
    # Ubah pertanyaan customer jadi vector, pakai fungsi yang sama dari topik 9
    # (harus model embedding yang SAMA dengan yang dipakai buat generate embedding artikel)
    query_embedding = generate_embedding(query)

    # pgvector menerima literal vector dalam bentuk string '[0.1,0.2,...]';
    # kita bikin string itu manual dari list[float] hasil generate_embedding
    embedding_literal = "[" + ",".join(str(x) for x in query_embedding) + "]"

    # RealDictCursor supaya tiap baris hasil query langsung berbentuk dict,
    # bukan tuple biasa — jadi gampang dikonversi ke list[dict] di akhir fungsi
    with conn.cursor(cursor_factory=psycopg2.extras.RealDictCursor) as cur:
        cur.execute(
            """
            SELECT
                id,
                title,
                content,
                embedding <=> %s::vector AS distance
            FROM knowledge_articles
            ORDER BY embedding <=> %s::vector
            LIMIT %s;
            """,
            (embedding_literal, embedding_literal, top_k),
        )
        rows = cur.fetchall()

    # rows berisi RealDictRow (mirip dict); konversi eksplisit ke dict biasa
    # supaya tipe return-nya konsisten dan gampang di-serialize (misal ke JSON)
    return [dict(row) for row in rows]
```

Contoh pemakaian:
```python
import psycopg2

conn = psycopg2.connect(
    dbname="supportpilot",
    user="supportpilot_app",
    password="...",
    host="localhost",
)

hasil = search_knowledge_base(conn, "gimana caranya ganti kata sandi ya", top_k=3)

for artikel in hasil:
    print(f"{artikel['title']} (distance={artikel['distance']:.4f})")
    # distance makin kecil = makin mirip (karena ini cosine DISTANCE, bukan similarity)
```

### Trade-off & Pitfall
- **ANN index itu approximate, bukan exact.** `ivfflat`/`hnsw` bisa sesekali "melewatkan" hasil yang secara matematis sedikit lebih mirip, demi kecepatan — trade-off ini hampir selalu sepadan di skala production, tapi jangan asumsikan hasilnya 100% identik dengan linear scan brute-force.
- **Operator distance harus konsisten sama index op class-nya.** Index di atas dibikin pakai `vector_cosine_ops` (buat operator `<=>`) — kalau query malah pakai `<->` (euclidean) atau `<#>` (inner product), PostgreSQL gak akan bisa memanfaatkan index itu secara optimal (bisa fallback ke sequential scan yang lambat).
- **Dimensi vector di kolom tabel HARUS sama persis dengan output model embedding yang dipakai.** Kalau model embedding diganti (misal dari `text-embedding-3-small` 1536 dimensi ke model lain dengan dimensi beda), insert/query bakal error karena mismatch dimensi — kolom `vector(1536)` gak bisa nerima vector 384 dimensi begitu aja.
- **`ivfflat` butuh data yang cukup dan `ANALYZE` sebelum index-nya efektif** — index ini membangun cluster berdasarkan distribusi data yang ADA saat index dibuat; kalau dibuat saat tabel masih kosong/sedikit lalu banyak data baru masuk belakangan, kualitas index-nya bisa menurun sampai di-`REINDEX` ulang.
- **Jangan bandingkan vector dari dua model embedding berbeda** — sama seperti disebut di topik 9, satu tabel `knowledge_articles` harus konsisten pakai satu model embedding buat semua barisnya.

### Kapan Dipakai
- Pakai pgvector (atau vector database lain) begitu jumlah data yang mau dicari-mirip-miripkan udah cukup besar (ratusan ke atas) sehingga linear scan di application code (topik 10) mulai kerasa lambat, atau begitu fitur ini butuh dipakai di request customer secara real-time.
- Karena SupportPilot udah pakai PostgreSQL, pgvector adalah pilihan paling natural — gak perlu operasikan database terpisah (misal vector database khusus) cuma buat fitur semantic search ini, kecuali kebutuhan skalanya benar-benar ekstrem (puluhan juta+ vector dengan traffic sangat tinggi).
- Ini adalah komponen inti dari **retrieval** step di RAG (Phase 4) — `search_knowledge_base` di sini nantinya jadi building block buat "cari dokumen relevan" sebelum dikasih ke LLM sebagai context.

### Sering Ditanya Saat Interview
- **Kenapa butuh vector database, gak cukup hitung similarity di application code?** — di skala kecil cukup, tapi begitu datanya banyak, linear scan (bandingin satu-satu) jadi lambat; vector database pakai index ANN yang mempersempit pencarian jauh lebih cepat tanpa harus scan semua baris.
- **Kenapa pgvector cocok buat tim yang udah pakai PostgreSQL?** — karena cukup nambah extension ke database yang sudah ada, gak perlu operasikan sistem database terpisah cuma buat fitur vector search.
- **Apa itu index ANN (approximate nearest neighbor), dan apa trade-off-nya?** — index yang mempercepat pencarian vector-vector terdekat, tapi hasilnya "approximate" (gak 100% dijamin paling akurat secara matematis) — trade-off kecepatan vs presisi sempurna yang biasanya sepadan buat kebutuhan production.
- **Kenapa dimensi vector di kolom tabel harus cocok sama model embedding-nya?** — karena tipe `vector(N)` di pgvector fixed-size; kalau model embedding diganti dan dimensinya beda, insert/query akan gagal karena mismatch ukuran vector.

---

**Selanjutnya:** [Phase 04 — RAG](./phase-04-rag.md)
