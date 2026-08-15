# Phase 08 — System Design

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

## 75. Software Architecture Patterns

### Apa itu?
Tiga hal yang saling terkait tapi beda level. **Clean/Hexagonal Architecture** adalah cara mengatur *layer* kode supaya business logic (use case) ada di tengah, gak bergantung langsung ke detail teknis seperti Postgres, Redis, atau framework HTTP — semua detail teknis itu ada di "pinggir" (adapter). **Repository Pattern** adalah penerapan konkretnya buat akses data: business logic cuma kenal sebuah *interface* (misalnya `OrderRepository`), gak pernah kenal `*pgxpool.Pool` atau SQL mentah secara langsung. **Dependency Injection (DI)** adalah cara "menyambungkan" implementasi konkret (misalnya `PostgresOrderRepository`) ke kode yang butuh interface itu — disuntikkan dari luar (biasanya lewat constructor) saat aplikasi start, bukan dibuat sendiri di tengah-tengah business logic.

### Kenapa dibutuhkan?
Sejak Phase 3, OrderFlow punya fungsi seperti `CreateOrder(ctx, db, userID, items)` dan `GetProductByID(ctx, db, id)` yang menerima `*pgxpool.Pool` langsung sebagai parameter. Ini gampang ditulis di awal, tapi begitu OrderFlow tumbuh, dua masalah muncul: (1) unit test buat HTTP handler yang manggil fungsi-fungsi ini jadi butuh database Postgres beneran, karena gak ada cara "mock" `*pgxpool.Pool`; (2) kalau suatu saat OrderFlow butuh ganti storage (misalnya pindah sebagian data ke storage lain, atau nambah layer caching di depan write), setiap tempat yang manggil `CreateOrder` langsung harus diubah satu-satu. Repository Pattern + DI menyelesaikan keduanya: business logic cukup bergantung ke `OrderRepository` (interface), yang gampang di-mock buat testing dan gampang diganti implementasinya tanpa menyentuh kode yang memakainya.

### Cara Kerja
```
Sebelum (tanpa Repository Pattern):

  OrderHandler --------------------> *pgxpool.Pool --------------------> Postgres
       |                                   ^
       '-- manggil CreateOrder(ctx, db, ...) langsung, "db" konkret pgxpool

  Masalah: OrderHandler TAU bahwa storage-nya Postgres. Unit test handler ini
  wajib nyalain Postgres beneran (atau minimal container test DB).


Sesudah (Repository Pattern + Dependency Injection):

  OrderHandler --> OrderRepository (interface) <-- PostgresOrderRepository --> Postgres
                        ^ GetByID(ctx, id)              (implementasi konkret,
                        ^ Create(ctx, order)              membungkus CreateOrder
                                                           & GetProductByID Phase 3)

  main() / wiring:
    pool := pgxpool.New(...)                       // 1. bikin dependency konkret
    repo := NewPostgresOrderRepository(pool)        // 2. bungkus jadi implementasi
    handler := NewOrderHandler(repo)                // 3. SUNTIKKAN ke handler (DI)

  OrderHandler cuma pegang tipe OrderRepository -- gak pernah tau ada
  pgxpool.Pool di baliknya. Testing tinggal suntikkan fake OrderRepository.
```

### Contoh Kode — Go
Repository Pattern: interface `OrderRepository` sebagai port, `PostgresOrderRepository` sebagai adapter yang membungkus `CreateOrder` dan `GetProductByID` dari Phase 3 apa adanya (gak menulis ulang logic locking atau hitung totalnya):
```go
package repository

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"

	orderdb "orderflow/internal/db"
)

// Order adalah domain model yang dipakai use case layer -- beda dari
// orderdb.Order (baris mentah tabel orders dari Phase 3), Order di sini
// juga membawa Items karena itu yang dibutuhkan use case "buat order baru"
// dan "lihat detail order".
type Order struct {
	ID      int64
	UserID  int64
	Status  string
	Total   float64
	Version int
	Items   []OrderItemDetail
}

// OrderItemDetail merepresentasikan satu item beserta detail product-nya.
type OrderItemDetail struct {
	ProductID   int64
	ProductName string
	Qty         int
	Price       float64
}

// OrderRepository adalah port (interface) di Clean/Hexagonal Architecture --
// use case layer cuma kenal interface ini, gak tau ataupun peduli apakah di
// baliknya Postgres, in-memory (buat unit test), atau storage lain.
type OrderRepository interface {
	GetByID(ctx context.Context, id int64) (*Order, error)
	Create(ctx context.Context, order *Order) error
}

// PostgresOrderRepository adalah adapter -- satu-satunya tempat di seluruh
// codebase yang tau OrderFlow pakai Postgres buat order.
type PostgresOrderRepository struct {
	db *pgxpool.Pool
}

// NewPostgresOrderRepository adalah constructor -- dipanggil sekali saat
// startup, bagian dari Dependency Injection (lihat Cara Kerja di atas).
func NewPostgresOrderRepository(db *pgxpool.Pool) *PostgresOrderRepository {
	return &PostgresOrderRepository{db: db}
}

// GetByID ambil order beserta detail tiap item-nya. Baris order & order_items
// diambil manual, lalu product tiap item di-lookup lewat GetProductByID
// (topik 25, Phase 3) -- reuse fungsi yang sudah ada apa adanya, gak nulis
// query product baru di sini.
func (r *PostgresOrderRepository) GetByID(ctx context.Context, id int64) (*Order, error) {
	var o Order
	err := r.db.QueryRow(ctx,
		`SELECT id, user_id, status, total, version FROM orders WHERE id = $1`,
		id,
	).Scan(&o.ID, &o.UserID, &o.Status, &o.Total, &o.Version)
	if err != nil {
		return nil, fmt.Errorf("get order %d: %w", id, err)
	}

	rows, err := r.db.Query(ctx,
		`SELECT product_id, qty, price FROM order_items WHERE order_id = $1`,
		id,
	)
	if err != nil {
		return nil, fmt.Errorf("get order items %d: %w", id, err)
	}
	defer rows.Close()

	for rows.Next() {
		var productID int64
		var qty int
		var price float64
		if err := rows.Scan(&productID, &qty, &price); err != nil {
			return nil, fmt.Errorf("scan order item: %w", err)
		}

		product, err := orderdb.GetProductByID(ctx, r.db, productID)
		if err != nil {
			return nil, fmt.Errorf("get product %d: %w", productID, err)
		}
		name := "unknown"
		if product != nil {
			name = product.Name
		}
		o.Items = append(o.Items, OrderItemDetail{
			ProductID:   productID,
			ProductName: name,
			Qty:         qty,
			Price:       price,
		})
	}
	return &o, rows.Err()
}

// Create membungkus CreateOrder (topik 29, Phase 3) apa adanya -- semua
// logic SELECT ... FOR UPDATE dan hitung total tetap di satu tempat.
// Repository ini cuma mapping dari domain Order ke parameter yang
// CreateOrder butuhkan, lalu menyalin hasilnya balik ke order milik caller.
func (r *PostgresOrderRepository) Create(ctx context.Context, order *Order) error {
	items := make([]orderdb.OrderItem, len(order.Items))
	for i, item := range order.Items {
		items[i] = orderdb.OrderItem{
			ProductID: item.ProductID,
			Qty:       item.Qty,
		}
	}

	created, err := orderdb.CreateOrder(ctx, r.db, order.UserID, items)
	if err != nil {
		return fmt.Errorf("create order: %w", err)
	}

	order.ID = created.ID
	order.Status = created.Status
	order.Total = created.Total
	order.Version = created.Version
	return nil
}
```
Dependency Injection: `main` yang menyambungkan implementasi konkret ke handler yang cuma kenal interface-nya:
```go
package main

import (
	"context"
	"encoding/json"
	"net/http"

	"github.com/jackc/pgx/v5/pgxpool"

	"orderflow/internal/repository"
)

// OrderHandler cuma kenal repository.OrderRepository (interface), gak
// pernah kenal PostgresOrderRepository ataupun *pgxpool.Pool sama sekali --
// ini yang bikin handler bisa di-unit-test pakai fake OrderRepository tanpa
// nyalain database beneran.
type OrderHandler struct {
	orders repository.OrderRepository
}

func NewOrderHandler(orders repository.OrderRepository) *OrderHandler {
	return &OrderHandler{orders: orders}
}

func (h *OrderHandler) GetOrder(w http.ResponseWriter, r *http.Request, orderID int64) {
	order, err := h.orders.GetByID(r.Context(), orderID)
	if err != nil {
		http.Error(w, "internal error", http.StatusInternalServerError)
		return
	}
	if order == nil {
		http.Error(w, "not found", http.StatusNotFound)
		return
	}
	json.NewEncoder(w).Encode(order)
}

// main melakukan Dependency Injection secara manual: bikin concrete
// implementation (PostgresOrderRepository) sekali di titik paling luar
// aplikasi, lalu "suntikkan" ke handler lewat constructor. Kalau nanti
// OrderFlow ganti storage buat order, cuma baris di fungsi ini yang
// berubah -- OrderHandler dan tipe OrderRepository sama sekali gak disentuh.
func main() {
	pool, err := pgxpool.New(context.Background(), "postgres://localhost/orderflow")
	if err != nil {
		panic(err)
	}
	defer pool.Close()

	orderRepo := repository.NewPostgresOrderRepository(pool)
	handler := NewOrderHandler(orderRepo)

	mux := http.NewServeMux()
	mux.HandleFunc("/orders/", func(w http.ResponseWriter, r *http.Request) {
		handler.GetOrder(w, r, 42) // parsing id dari URL disederhanakan
	})
	http.ListenAndServe(":8080", mux)
}
```

### Contoh Kode — Node.js
Repository Pattern: `class OrderRepository` yang membungkus `createOrder` dan `getProductById` dari Phase 3 apa adanya:
```javascript
// order-repository.js
// createOrder & getProductById diimport dari modul Phase 3 -- kelas ini
// gak menulis ulang logic locking/hitung total, cuma membungkusnya di balik
// interface yang konsisten (getById / create) supaya use case layer gak
// perlu tau detail SQL ataupun bahwa penyimpanannya Postgres.
const { createOrder, getProductById } = require('./db'); // dari phase 03

class OrderRepository {
  constructor(pool) {
    this.pool = pool;
  }

  // getById ambil order beserta detail tiap item-nya. Baris order_items
  // diambil manual, lalu product tiap item di-lookup lewat getProductById
  // (topik 25) -- reuse fungsi Phase 3 apa adanya.
  async getById(id) {
    const orderResult = await this.pool.query(
      'SELECT id, user_id, status, total, version FROM orders WHERE id = $1',
      [id]
    );
    if (orderResult.rows.length === 0) {
      return null;
    }
    const order = orderResult.rows[0];

    const itemsResult = await this.pool.query(
      'SELECT product_id, qty, price FROM order_items WHERE order_id = $1',
      [id]
    );

    const items = [];
    for (const row of itemsResult.rows) {
      const product = await getProductById(this.pool, row.product_id);
      items.push({
        productId: row.product_id,
        productName: product ? product.name : 'unknown',
        qty: row.qty,
        price: row.price,
      });
    }

    return { ...order, items };
  }

  // create membungkus createOrder (topik 29) apa adanya -- semua logic
  // SELECT ... FOR UPDATE dan hitung total tetap di satu tempat (Phase 3).
  async create(order) {
    const items = order.items.map((item) => ({
      productId: item.productId,
      qty: item.qty,
    }));
    return createOrder(this.pool, order.userId, items);
  }
}

module.exports = { OrderRepository };
```
Dependency Injection: handler cuma kenal instance `OrderRepository` yang di-suntikkan lewat constructor, wiring-nya ada di satu tempat saja:
```javascript
// order-handler.js
// OrderHandler cuma kenal instance OrderRepository yang di-suntikkan lewat
// constructor -- gak pernah require('./db') ataupun pg langsung, jadi bisa
// di-unit-test pakai fake repository tanpa database beneran.
class OrderHandler {
  constructor(orderRepository) {
    this.orderRepository = orderRepository;
  }

  async getOrder(req, res) {
    const order = await this.orderRepository.getById(Number(req.params.id));
    if (!order) {
      return res.sendStatus(404);
    }
    return res.json(order);
  }
}

module.exports = { OrderHandler };
```
```javascript
// main.js
// Dependency Injection manual: bikin concrete implementation (OrderRepository
// dengan pool Postgres beneran) sekali di titik paling luar aplikasi, lalu
// suntikkan ke handler lewat constructor. Kalau nanti OrderFlow ganti
// storage buat order, cuma baris di file ini yang berubah.
const { Pool } = require('pg');
const { OrderRepository } = require('./order-repository');
const { OrderHandler } = require('./order-handler');

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const orderRepository = new OrderRepository(pool);
const orderHandler = new OrderHandler(orderRepository);

module.exports = { orderHandler };
```

### Trade-off & Pitfall
- Repository Pattern nambah satu layer indirection (interface + implementasi) buat setiap entity — untuk aplikasi kecil dengan satu jenis storage yang gak akan pernah ganti, ini bisa kerasa seperti boilerplate berlebihan dibanding manggil `CreateOrder(ctx, db, ...)` langsung.
- Interface `OrderRepository` yang cuma punya satu implementasi (`PostgresOrderRepository`) sepanjang umur project itu wajar dan BUKAN tanda "gagal abstraksi" — manfaat utamanya bukan "suatu saat ganti Postgres", tapi testability (bisa disuntik fake repository) dan batas layer yang jelas.
- Kalau interface repository didesain terlalu generik (misalnya `Save(interface{}) error` buat semua entity), business logic jadi kehilangan tipe yang kuat dan malah butuh type assertion di mana-mana — lebih baik satu interface spesifik per entity seperti `OrderRepository`, `ProductRepository`, dst.
- Dependency Injection manual (seperti contoh di atas) cukup buat ukuran OrderFlow saat ini; begitu graph dependency-nya membesar (puluhan repository/service saling bergantung), tim biasanya pindah ke DI container/framework (contoh di Go: `wire`, `fx`) supaya wiring-nya gak jadi satu fungsi `main` yang panjang sekali.

### Kapan Dipakai
Dipakai begitu business logic OrderFlow mulai punya aturan yang cukup kompleks untuk di-unit-test terpisah dari database (misalnya validasi stock, hitung diskon, aturan pembatalan order), atau begitu ada kebutuhan nyata buat mock storage saat testing. Untuk script kecil sekali pakai atau prototipe yang gak butuh test unit terpisah, manggil fungsi database langsung (seperti `CreateOrder`) tanpa lapisan repository masih sah-sah saja.

### Sering Ditanya Saat Interview
- "Apa beda Repository Pattern dengan cuma nulis fungsi `CreateOrder(db, ...)` biasa?" — `CreateOrder` biasa menerima `*pgxpool.Pool` konkret sebagai parameter, jadi pemanggilnya otomatis terikat ke Postgres; Repository Pattern membungkusnya di balik interface (`OrderRepository`) sehingga pemanggil cuma bergantung ke kontrak abstrak, bukan ke Postgres secara langsung.
- "Kenapa Dependency Injection bikin kode lebih gampang di-test?" — karena dependency (seperti `OrderRepository`) disuntikkan dari luar lewat constructor, test bisa mengganti dependency asli dengan fake/mock yang perilakunya dikontrol penuh, tanpa perlu nyalain Postgres beneran atau mengubah kode yang diuji.
- "Apa risiko terbesar kalau Clean/Hexagonal Architecture diterapkan berlebihan di project kecil?" — over-engineering: terlalu banyak layer dan interface buat logic yang sebetulnya sederhana bikin kode lebih susah dinavigasi (satu perubahan kecil harus menyentuh banyak file), padahal manfaat abstraksinya belum tentu kepakai kalau requirement-nya gak pernah berubah.

---

## 76. Basic Architecture

### Apa itu?
Basic Architecture adalah gambaran arsitektur "standar" yang muncul di hampir semua sistem backend production skala menengah: client memanggil lewat CDN/WAF, request masuk ke Load Balancer, diteruskan ke banyak API server yang stateless, API server itu baca/tulis lewat cache (Redis) dan database (Postgres), dan pekerjaan berat/gak perlu synchronous dilempar ke Message Queue supaya diproses belakangan oleh Worker.

### Kenapa dibutuhkan?
Ini adalah "peta dasar" yang dipakai buat menjelaskan hampir semua keputusan arsitektur di phase-phase sebelumnya sekaligus: kenapa API server OrderFlow harus stateless (topik 65-66) begitu ada Load Balancer (topik 67), kenapa `GetProductCached` (topik 42, Phase 4) menaruh Redis di depan Postgres, kenapa `PublishOrderCreated` (topik 48, Phase 5) melempar pekerjaan ke Kafka bukan diproses langsung di request handler. Tanpa gambaran utuh ini, tiap komponen kelihatan seperti potongan lepas — padahal semuanya saling terhubung jadi satu request flow.

### Cara Kerja
```
                              Client
                                |
                          CDN / WAF
                                |
                    Load Balancer (topik 67)
                                |
        -----------------------+-----------------------
        |              |               |               |
   API Server      API Server      API Server      API Server   (stateless,
                                                                  topik 65-66,
                                                                  N instance)
        \______________|_______________|_______________/
                                |
              +-----------------+-----------------+
              |                                    |
        Redis (topik 42-46)              Postgres primary + replica
        cache-aside GetProductCached      (topik 22-40), CreateOrder
              |                            SELECT ... FOR UPDATE
              +-----------------+-----------------+
                                |
                 publish OrderEvent "order.created"
                       (topik 48, Phase 5)
                                |
                        Kafka topic order.created
                                |
                 Consumer group (idempotent + retry + DLQ,
                                  topik 48-52)
                                |
                    Worker Pool (topik 55-62)
                                |
        +-----------------+----+----+-----------------+
        |                 |         |                 |
   Kirim email       Generate    Notify         Payment gateway /
   konfirmasi        invoice     warehouse       service eksternal
```
Alur satu request "checkout" dari ujung ke ujung: client kirim `POST /orders` dengan JWT (Phase 1) di header → Load Balancer meneruskan ke salah satu API server (topik 65-67) → API server validasi token (`ValidateToken`, topik 3) → panggil `CreateOrder` yang mengunci row product lewat `SELECT ... FOR UPDATE` (topik 29) di Postgres → setelah commit, `PublishOrderCreated` mengirim event ke Kafka (topik 48) → API server langsung balas response ke client tanpa nunggu proses lanjutannya → consumer group membaca event itu secara asynchronous → worker pool (topik 60) memproses tiap event: kirim email, generate invoice, notify warehouse — semuanya di luar jalur request/response yang dilihat user.

### Trade-off & Pitfall
- Diagram ini adalah "arsitektur matang", bukan titik awal — membangun Kafka, worker pool, dan read replica sejak hari pertama buat OrderFlow versi MVP dengan traffic kecil itu over-engineering; komponen-komponen ini ditambahkan satu per satu begitu masalah konkretnya benar-benar muncul (baca lambat → cache, tulis berat/lambat → queue, traffic naik → horizontal scale).
- Menaruh cache (Redis) hanya di depan operasi baca itu gampang; menjaga cache tetap konsisten dengan Postgres begitu ada operasi tulis (invalidasi, topik 43) adalah bagian yang paling sering jadi sumber bug — data stale yang gak sengaja ke-serve ke user.
- Memisahkan pekerjaan ke message queue membuat proses "order dibuat" (fast, synchronous) dan "notifikasi terkirim" (lambat, asynchronous) jadi dua hal yang gak lagi atomik — trade-off ini sudah eksplisit dibahas sebagai eventual consistency di topik 69.
- Diagram di atas menyederhanakan "Postgres" jadi satu kotak, padahal di baliknya bisa ada primary + replica (topik 33-34) atau bahkan sharding (topik 40) — kompleksitas storage itu sendiri bisa jadi satu diagram terpisah kalau perlu dibahas lebih detail saat interview.

### Kapan Dipakai
Dipakai sebagai starting point tiap kali diminta "gambar arsitektur backend buat sistem X" di interview, atau sebagai checklist waktu mendesain service baru di OrderFlow: sudah ada load balancer di depan API? API-nya stateless? Ada cache buat data yang sering dibaca jarang berubah? Pekerjaan berat sudah dipisah ke queue? Kalau salah satu kotak di diagram ini belum ada tapi masalahnya sudah muncul (misalnya database mulai keteteran baca), itu sinyal komponen itu perlu ditambahkan.

### Sering Ditanya Saat Interview
- "Kenapa Redis diletakkan di antara API server dan Postgres, bukan sebaliknya?" — API server selalu cek Redis dulu (cache-aside, topik 42) sebelum ke Postgres, supaya query yang sering diulang (misalnya detail product yang sama dibuka ribuan kali) gak harus selalu membebani database; Postgres cuma disentuh saat cache miss atau saat menulis data.
- "Kapan sebuah sistem mulai butuh message queue, dibanding cukup manggil fungsi selanjutnya secara synchronous?" — begitu ada pekerjaan yang (a) gak perlu selesai sebelum response dibalas ke user (kirim email, generate invoice) atau (b) perlu didengar oleh lebih dari satu consumer independen di masa depan; kalau semua langkah wajib selesai sebelum response dan cuma ada satu consumer, synchronous call masih lebih sederhana.
- "Apa yang bikin API server di diagram ini bisa di-scale horizontal dengan aman?" — API server-nya stateless (topik 65-66): semua data yang perlu "diingat" antar request datang dari JWT atau dari storage bersama (Redis/Postgres), bukan dari memory lokal satu instance, jadi Load Balancer bebas mengirim request ke instance manapun.

---

## 77. System Design Interview Framework

### Apa itu?
Kerangka 7 langkah buat menjawab soal system design di interview secara terstruktur, bukan langsung lompat ke detail teknis: (1) Requirements, (2) API Design, (3) Data Model, (4) Architecture, (5) Scaling, (6) Reliability, (7) Security. Framework ini juga yang dipakai buat menyusun urutan topik di seluruh roadmap ini — Phase 1-2 soal Security/API, Phase 3-4 soal Data Model/Architecture, Phase 5-7 soal Scaling, Phase 9 soal Reliability.

### Kenapa dibutuhkan?
Pertanyaan interview seperti "desain OrderFlow, sistem order untuk e-commerce" itu sengaja ambigu — kalau langsung menjawab dengan diagram teknis tanpa klarifikasi, kandidat gampang salah asumsi skala (ribuan vs jutaan user) atau salah prioritas (fokus ke high availability padahal interviewer sebenarnya paling peduli soal konsistensi stock). Framework 7 langkah ini memaksa klarifikasi dan estimasi dulu sebelum desain, dan memastikan gak ada aspek penting (security, reliability) yang kelewat cuma karena kehabisan waktu di detail arsitektur.

### Cara Kerja
Jalankan ketujuh langkah ini secara berurutan dengan "desain OrderFlow" sebagai contoh:

**Step 1 — Requirements**: Tanyakan dulu ke interviewer sebelum desain apapun. Untuk OrderFlow: apa fungsi utamanya (buat order, cek stock, bayar)? Siapa usernya (customer + admin gudang)? Fitur wajib vs nice-to-have (checkout wajib, rekomendasi produk nice-to-have)? Expected traffic (berapa order/detik saat flash sale)? Data size (berapa juta produk, berapa order/hari)? Latency & availability requirement (checkout harus di bawah berapa detik, boleh downtime berapa lama)?

**Step 2 — API Design**: Definisikan endpoint utamanya — `POST /orders` (checkout, idempotent lewat idempotency key supaya klik dobel gak bikin dua order), `GET /orders/{id}` (lihat detail order, wajib auth — topik 1 & 8 Phase 1), `GET /products/{id}` (lihat produk, public). Pikirkan juga pagination buat `GET /orders` (daftar order milik user) dan error handling yang konsisten (bentuk error response yang sama di semua endpoint, topik-topik Phase 2).

**Step 3 — Data Model**: Entity utamanya `User`, `Product`, `Order`, `OrderItem`, `Payment` (persis skema Phase 3, topik 25). Primary/foreign key: `orders.user_id REFERENCES users(id)`, `order_items` pakai composite key `(order_id, product_id)`. Index yang dibutuhkan: `idx_orders_user_id` buat query "order milik user ini" (topik 26). Query pattern yang paling sering: baca detail product (jauh lebih sering dari nulis), baca order history per user.

**Step 4 — Architecture**: Gambar komponennya persis seperti diagram topik 76 — Client → Load Balancer → API Servers (stateless) → Redis + Postgres → Message Queue → Worker. Ini titik di mana keputusan-keputusan dari Phase 3-7 dipetakan jadi satu gambar utuh.

**Step 5 — Scaling**: Identifikasi bottleneck satu per satu. API server kehabisan kapasitas → horizontal scaling (topik 65), gampang karena sudah stateless. Postgres kewalahan baca → tambah index (topik 26) atau read replica (topik 33). Baca yang sama diulang-ulang → Redis cache (topik 42-46). Proses berat (kirim email massal, generate invoice) → message queue + worker pool (topik 47-62), jangan diproses synchronous di request handler. Kalau traffic tulis (`CreateOrder`) sendiri yang jadi bottleneck di satu tabel `orders`, opsi lanjutannya sharding berdasarkan `user_id` (topik 40, Phase 3).

**Step 6 — Reliability**: Pastikan tiap panggilan ke service lain (payment gateway, service eksternal) punya timeout & retry dengan backoff (topik 51), dibungkus circuit breaker biar gak ikut down kalau service itu lambat. `CreateOrder` harus idempotent terhadap retry (topik 50) supaya double-submit gak bikin order dobel. Health check (liveness/readiness, Phase 9) di tiap instance API supaya Load Balancer gak ngirim traffic ke instance yang lagi gak sehat. Ada strategi graceful degradation (misalnya rekomendasi produk boleh gagal diam-diam, tapi checkout gak boleh) dan backup/disaster recovery buat Postgres.

**Step 7 — Security**: Urutkan dari luar ke dalam — network (WAF/rate limiting di depan, Phase 2), authentication (JWT, topik 3 Phase 1), authorization (RBAC, topik 8 Phase 1, pastikan user A gak bisa lihat order milik user B), encryption (TLS in transit, encrypt data sensitif at rest), secrets management (kredensial database/API key gak nge-hardcode di kode), dan monitoring/audit log buat mendeteksi aktivitas mencurigakan.

### Trade-off & Pitfall
- Menghabiskan terlalu banyak waktu di Step 1 (klarifikasi) tanpa pernah sampai gambar arsitektur konkret sama buruknya dengan langsung lompat ke Step 4 tanpa klarifikasi sama sekali — framework ini butuh alokasi waktu yang seimbang, biasanya klarifikasi + estimasi cuma 10-15% dari total waktu interview.
- Step 5-7 (Scaling, Reliability, Security) sering kelewat karena waktu habis di Step 4 (menggambar diagram detail) — kandidat yang kuat menyisakan waktu khusus buat ketiga step terakhir ini, karena itu yang sering membedakan jawaban "sekadar bisa gambar arsitektur" dengan jawaban level senior.
- Framework 7 langkah ini adalah kerangka berpikir, bukan checklist kaku yang harus diikuti runtut sekali jalan — di percakapan interview asli, wajar bolak-balik (misalnya nemu masalah scaling pas lagi bahas data model, lalu balik lagi bahas index).

### Kapan Dipakai
Dipakai di setiap sesi system design interview, dan juga berguna di dunia nyata sebagai checklist waktu proposal desain untuk fitur/service baru di OrderFlow — memastikan requirement, API, data model, arsitektur, scaling, reliability, dan security semuanya sudah dipikirkan sebelum mulai implementasi, bukan ditambal belakangan satu-satu.

### Sering Ditanya Saat Interview
- "Kenapa harus klarifikasi requirement dulu sebelum mulai gambar arsitektur?" — soal system design sengaja dibuat ambigu; tanpa klarifikasi, kandidat bisa mendesain buat skala atau prioritas yang salah (misalnya optimasi buat 10 request/detik padahal interviewer maunya desain buat flash sale 10.000 request/detik), yang bikin seluruh desain berikutnya jadi gak relevan.
- "Kalau waktu interview terbatas, step mana yang paling penting buat gak dilewatkan?" — Step 1 (Requirements) dan Step 4 (Architecture) adalah fondasi yang harus ada; tapi kandidat senior biasanya tetap menyisihkan waktu buat menyentuh Step 5-7 (Scaling, Reliability, Security) walau singkat, karena melewatkannya sama sekali sering dibaca sebagai kurang pengalaman production.
- "Gimana urutan mikirin security yang benar buat sistem seperti OrderFlow?" — dari luar ke dalam: network protection dulu (WAF, rate limiting), baru authentication (memastikan identitas user), baru authorization (memastikan user itu berhak akses resource spesifik yang diminta), baru encryption & secrets, ditutup monitoring buat mendeteksi kalau ada yang lolos dari semua lapisan sebelumnya.

---

**Selanjutnya:** [Phase 09 — Reliability & Production](./phase-09-reliability-production.md)
