# Phase 03 — Database

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

## 25. Database Fundamentals

### Apa itu?
Database fundamentals adalah pemahaman dasar soal gimana OrderFlow menyimpan datanya secara persisten di Postgres: tabel apa saja yang ada, kolom apa di tiap tabel, tipe data yang dipakai, dan relasi antar tabel lewat foreign key. Di OrderFlow ada lima tabel inti: `users`, `products`, `orders`, `order_items`, dan `payments`.

### Kenapa dibutuhkan?
Semua topik lain di phase ini — index, transaction, locking, replication — dibangun di atas skema dasar ini. Kalau skemanya berantakan dari awal (tipe data salah, foreign key gak ada, kolom yang harusnya `NOT NULL` malah nullable), masalah itu bakal nongol berulang-ulang di setiap layer di atasnya: query jadi lambat, data jadi inkonsisten, dan business logic (seperti `CreateOrder`) harus kerja ekstra buat kompensasi hal yang seharusnya dijamin database.

### Cara Kerja
```sql
CREATE TABLE users (
    id            BIGSERIAL PRIMARY KEY,
    email         TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    role          TEXT NOT NULL DEFAULT 'customer',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE products (
    id    BIGSERIAL PRIMARY KEY,
    name  TEXT NOT NULL,
    price NUMERIC(12, 2) NOT NULL,
    stock INTEGER NOT NULL DEFAULT 0
);

CREATE TABLE orders (
    id         BIGSERIAL PRIMARY KEY,
    user_id    BIGINT NOT NULL REFERENCES users (id),
    status     TEXT NOT NULL DEFAULT 'pending',
    total      NUMERIC(12, 2) NOT NULL,
    version    INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE order_items (
    order_id   BIGINT NOT NULL REFERENCES orders (id),
    product_id BIGINT NOT NULL REFERENCES products (id),
    qty        INTEGER NOT NULL,
    price      NUMERIC(12, 2) NOT NULL,
    PRIMARY KEY (order_id, product_id)
);

CREATE TABLE payments (
    id              BIGSERIAL PRIMARY KEY,
    order_id        BIGINT NOT NULL REFERENCES orders (id),
    status          TEXT NOT NULL DEFAULT 'pending',
    idempotency_key TEXT NOT NULL UNIQUE
);
```
`orders.version` dipakai buat optimistic locking (topik 33), dan `order_items` sengaja gak punya `id` sendiri — composite primary key `(order_id, product_id)` sudah cukup karena satu product cuma boleh muncul sekali per order.

### Contoh Kode — Go
```go
package db

import (
	"context"
	"embed"
	"errors"
	"fmt"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"
)

//go:embed schema.sql
var schemaFS embed.FS

// Product merepresentasikan baris di tabel products.
type Product struct {
	ID    int64
	Name  string
	Price float64
	Stock int
}

// Order merepresentasikan baris di tabel orders.
// Version dipakai untuk optimistic locking (topik 33).
type Order struct {
	ID      int64
	UserID  int64
	Status  string
	Total   float64
	Version int
}

// OrderItem merepresentasikan baris di tabel order_items.
type OrderItem struct {
	OrderID   int64
	ProductID int64
	Qty       int
	Price     float64
}

// RunMigrations menjalankan schema.sql sekali di awal -- biasanya lewat tool
// migrasi terpisah atau saat startup service, bukan berulang tiap request.
func RunMigrations(ctx context.Context, pool *pgxpool.Pool) error {
	schema, err := schemaFS.ReadFile("schema.sql")
	if err != nil {
		return fmt.Errorf("read schema.sql: %w", err)
	}
	if _, err := pool.Exec(ctx, string(schema)); err != nil {
		return fmt.Errorf("apply schema: %w", err)
	}
	return nil
}

// GetProductByID ambil satu product berdasarkan id -- operasi baca paling
// dasar yang dipakai hampir semua topik lain di phase ini.
func GetProductByID(ctx context.Context, db *pgxpool.Pool, id int64) (*Product, error) {
	var p Product
	err := db.QueryRow(ctx,
		`SELECT id, name, price, stock FROM products WHERE id = $1`,
		id,
	).Scan(&p.ID, &p.Name, &p.Price, &p.Stock)
	if err != nil {
		if errors.Is(err, pgx.ErrNoRows) {
			return nil, nil
		}
		return nil, fmt.Errorf("get product %d: %w", id, err)
	}
	return &p, nil
}
```

### Contoh Kode — Node.js
```javascript
const fs = require('fs');
const path = require('path');

// runMigrations menjalankan schema.sql sekali di awal.
async function runMigrations(pool) {
  const schema = fs.readFileSync(path.join(__dirname, 'schema.sql'), 'utf8');
  await pool.query(schema);
}

// getProductById ambil satu product berdasarkan id -- operasi baca paling
// dasar yang dipakai hampir semua topik lain di phase ini.
async function getProductById(pool, id) {
  const result = await pool.query(
    'SELECT id, name, price, stock FROM products WHERE id = $1',
    [id]
  );
  return result.rows[0] || null;
}

module.exports = { runMigrations, getProductById };
```

### Trade-off & Pitfall
- Pakai `NUMERIC(12, 2)` buat kolom uang (`price`, `total`), bukan `FLOAT`/`DOUBLE` — floating point binary gak bisa merepresentasikan pecahan desimal seperti 0.10 secara eksak, dan itu bahaya buat perhitungan uang yang harus presisi.
- `order_items` tanpa `id` sendiri (cukup composite PK) menghemat satu index, tapi kalau nanti butuh referensi ke satu baris item spesifik dari tabel lain, composite key lebih ribet dipakai sebagai foreign key dibanding single-column `id`.
- Lupa `NOT NULL` di kolom yang secara bisnis emang wajib ada (misalnya `orders.user_id`) bikin bug tersembunyi bisa lolos sampai jauh ke production, karena database gak menolak data yang jelas-jelas gak valid.

### Kapan Dipakai
Selalu — desain skema yang tepat di awal jauh lebih murah diperbaiki daripada migrasi besar-besaran setelah tabel `orders` OrderFlow sudah berisi jutaan baris data production.

### Sering Ditanya Saat Interview
- "Kenapa pakai `NUMERIC` bukan `FLOAT` buat kolom harga?" — `NUMERIC` menyimpan angka desimal secara eksak (arbitrary precision), sementara `FLOAT`/`DOUBLE` pakai representasi biner yang bisa menyebabkan rounding error kecil yang fatal untuk perhitungan uang.
- "Kenapa `order_items` gak punya primary key `id` sendiri?" — karena kombinasi `(order_id, product_id)` sudah unik secara alami (satu product cuma boleh muncul sekali per order) dan sudah cukup jadi primary key, tanpa perlu kolom tambahan.
- "Apa gunanya foreign key `orders.user_id REFERENCES users(id)`?" — database yang menjamin integritas referensial: gak mungkin ada order dengan `user_id` yang usernya gak ada, tanpa harus divalidasi manual tiap kali di application code.

---

## 26. Indexing

### Apa itu?
Index adalah struktur data tambahan (biasanya B-tree di Postgres) yang dibuat di atas satu atau lebih kolom tabel, supaya database bisa mencari baris yang cocok dengan kondisi `WHERE` tanpa harus membaca seluruh tabel baris demi baris (sequential scan).

### Kenapa dibutuhkan?
Tabel `orders` dan `products` di OrderFlow bakal terus tumbuh — dalam hitungan bulan bisa berisi jutaan baris. Query seperti "cari semua order milik user ini" atau "cari produk berdasarkan nama" yang sering dipanggil (tiap kali user buka halaman order history atau search produk) bakal makin lambat seiring data bertambah kalau Postgres harus scan seluruh tabel tiap kali.

### Cara Kerja
```sql
CREATE INDEX idx_orders_user_id ON orders (user_id);
CREATE INDEX idx_products_name  ON products (name);
```
```
Tanpa index:  WHERE user_id = 42  -> Postgres baca SEMUA baris orders, cek satu-satu
Dengan index: WHERE user_id = 42  -> Postgres cari langsung lewat B-tree index,
                                      lompat ke baris yang relevan
```

### Contoh Kode — Go
```go
package db

import (
	"context"

	"github.com/jackc/pgx/v5/pgxpool"
)

// AddIndexes menambahkan index yang dipakai kueri-kueri paling sering
// dijalankan OrderFlow: cari order milik user tertentu, dan cari produk
// berdasarkan nama.
func AddIndexes(ctx context.Context, pool *pgxpool.Pool) error {
	_, err := pool.Exec(ctx, `
		CREATE INDEX IF NOT EXISTS idx_orders_user_id ON orders (user_id);
		CREATE INDEX IF NOT EXISTS idx_products_name ON products (name);
	`)
	return err
}

// ListOrdersByUser memanfaatkan idx_orders_user_id, supaya Postgres gak
// perlu sequential scan seluruh tabel orders buat cari punya satu user.
func ListOrdersByUser(ctx context.Context, db *pgxpool.Pool, userID int64) ([]Order, error) {
	rows, err := db.Query(ctx,
		`SELECT id, user_id, status, total, version FROM orders WHERE user_id = $1`,
		userID)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var orders []Order
	for rows.Next() {
		var o Order
		if err := rows.Scan(&o.ID, &o.UserID, &o.Status, &o.Total, &o.Version); err != nil {
			return nil, err
		}
		orders = append(orders, o)
	}
	return orders, rows.Err()
}
```

### Contoh Kode — Node.js
```javascript
// addIndexes menambahkan index yang dipakai kueri-kueri paling sering
// dijalankan OrderFlow.
async function addIndexes(pool) {
  await pool.query(`
    CREATE INDEX IF NOT EXISTS idx_orders_user_id ON orders (user_id);
    CREATE INDEX IF NOT EXISTS idx_products_name ON products (name);
  `);
}

// listOrdersByUser memanfaatkan idx_orders_user_id.
async function listOrdersByUser(pool, userId) {
  const result = await pool.query(
    'SELECT id, user_id, status, total, version FROM orders WHERE user_id = $1',
    [userId]
  );
  return result.rows;
}

module.exports = { addIndexes, listOrdersByUser };
```

### Trade-off & Pitfall
- Index mempercepat `SELECT`, tapi memperlambat `INSERT`/`UPDATE`/`DELETE` sedikit, karena Postgres harus ikut update index tiap kali baris berubah — jangan asal index semua kolom "siapa tau kepake".
- Index butuh storage tambahan (bisa signifikan buat tabel besar) dan harus di-maintain — index yang gak pernah dipakai query manapun cuma jadi beban tanpa manfaat.
- `idx_products_name` pakai B-tree biasa cuma efektif buat pencarian exact-match atau prefix (`LIKE 'kabel%'`) — untuk pencarian substring bebas (`LIKE '%usb%'`) atau full-text search, B-tree gak banyak membantu, perlu index jenis lain (trigram, GIN).

### Kapan Dipakai
Tambahkan index di kolom yang sering dipakai buat filter (`WHERE`), join, atau sorting (`ORDER BY`) pada tabel yang datanya bakal besar — seperti `orders.user_id` yang dipanggil tiap user buka halaman order history-nya.

### Sering Ditanya Saat Interview
- "Gimana caramu tau satu kolom butuh index atau gak?" — lihat query pattern yang paling sering jalan (lewat query log atau `pg_stat_statements`), lalu cek `EXPLAIN ANALYZE`-nya (topik 28) apakah masih sequential scan pada tabel besar.
- "Kenapa gak semua kolom di-index saja biar aman?" — index bukan gratis: tiap write jadi lebih lambat karena harus update semua index terkait, dan storage jadi lebih besar; index yang gak kepake cuma beban.
- "Apa struktur data default index di Postgres?" — B-tree, cocok untuk equality (`=`) dan range query (`<`, `>`, `BETWEEN`); untuk kebutuhan lain (full-text search, geospatial) ada jenis index khusus seperti GIN atau GiST.

---

## 27. Composite Index

### Apa itu?
Composite index adalah index yang dibuat di atas lebih dari satu kolom sekaligus, dengan urutan kolom yang berpengaruh ke query mana saja yang bisa memanfaatkannya. Misalnya `CREATE INDEX ON orders (user_id, status)` beda dari dua index terpisah `(user_id)` dan `(status)`.

### Kenapa dibutuhkan?
Salah satu query paling sering dipanggil OrderFlow adalah "order aktif milik user ini" — filter dua kolom sekaligus (`user_id` dan `status`). Dengan cuma index single-column `idx_orders_user_id` (topik 26), Postgres masih harus filter manual satu-satu berdasarkan `status` setelah menemukan baris-baris `user_id` yang cocok. Composite index bikin kedua filter itu langsung ditangani index yang sama.

### Cara Kerja
```sql
CREATE INDEX idx_orders_user_status ON orders (user_id, status);
```
```
Composite index (user_id, status) efektif buat:
  WHERE user_id = 42                    -> pakai kolom pertama saja, tetap kepakai
  WHERE user_id = 42 AND status = 'active' -> kedua kolom match urutan index, paling optimal

TAPI gak efektif buat:
  WHERE status = 'active'               -> kolom pertama (user_id) gak difilter,
                                            index gak bisa dipakai buat "lompat" langsung
```

### Contoh Kode — Go
```go
package db

import (
	"context"

	"github.com/jackc/pgx/v5/pgxpool"
)

// AddCompositeIndex mempercepat kueri yang memfilter user_id DAN status
// sekaligus (misalnya "order aktif milik user ini"), yang gak optimal cuma
// pakai index single-column idx_orders_user_id.
func AddCompositeIndex(ctx context.Context, pool *pgxpool.Pool) error {
	_, err := pool.Exec(ctx,
		`CREATE INDEX IF NOT EXISTS idx_orders_user_status ON orders (user_id, status)`)
	return err
}

// ListActiveOrdersByUser memanfaatkan idx_orders_user_status: kedua kolom
// filter (user_id, status) match urutan kolom di index.
func ListActiveOrdersByUser(ctx context.Context, db *pgxpool.Pool, userID int64) ([]Order, error) {
	rows, err := db.Query(ctx,
		`SELECT id, user_id, status, total, version
		 FROM orders
		 WHERE user_id = $1 AND status = 'active'`,
		userID)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var orders []Order
	for rows.Next() {
		var o Order
		if err := rows.Scan(&o.ID, &o.UserID, &o.Status, &o.Total, &o.Version); err != nil {
			return nil, err
		}
		orders = append(orders, o)
	}
	return orders, rows.Err()
}
```

### Contoh Kode — Node.js
```javascript
// addCompositeIndex mempercepat kueri yang memfilter user_id DAN status sekaligus.
async function addCompositeIndex(pool) {
  await pool.query(
    'CREATE INDEX IF NOT EXISTS idx_orders_user_status ON orders (user_id, status)'
  );
}

// listActiveOrdersByUser memanfaatkan idx_orders_user_status.
async function listActiveOrdersByUser(pool, userId) {
  const result = await pool.query(
    `SELECT id, user_id, status, total, version
     FROM orders
     WHERE user_id = $1 AND status = 'active'`,
    [userId]
  );
  return result.rows;
}

module.exports = { addCompositeIndex, listActiveOrdersByUser };
```

### Trade-off & Pitfall
- Urutan kolom itu penting: `(user_id, status)` efisien buat query yang filter `user_id` saja atau `user_id` + `status`, tapi gak membantu query yang cuma filter `status` saja — kalau kedua pola query itu sama-sama sering, mungkin butuh index terpisah lagi.
- Composite index lebih besar dari index single-column, dan tetap kena overhead write yang sama (harus di-update tiap `INSERT`/`UPDATE`) — jangan bikin composite index buat kombinasi kolom yang jarang difilter bareng.
- Kalau sudah ada `idx_orders_user_status (user_id, status)`, index terpisah `idx_orders_user_id (user_id)` (topik 26) jadi agak redundan untuk sebagian besar query — bisa dipertimbangkan untuk dihapus salah satu supaya gak dobel maintenance cost.

### Kapan Dipakai
Ketika ada pola query yang konsisten memfilter beberapa kolom sekaligus (bukan cuma satu-satu), dan query itu cukup sering dipanggil untuk membenarkan overhead index tambahan — seperti "order aktif milik user" yang dipanggil tiap kali user buka dashboard-nya.

### Sering Ditanya Saat Interview
- "Kalau ada composite index `(user_id, status)`, apakah query `WHERE status = 'active'` saja bisa memanfaatkannya?" — secara umum tidak efisien, karena Postgres butuh kolom pertama index (`user_id`) untuk bisa langsung "lompat" ke bagian index yang relevan; query itu kemungkinan tetap sequential/full-index scan.
- "Kenapa urutan kolom di composite index penting?" — index B-tree composite disusun berjenjang berdasarkan urutan kolom yang dideklarasikan, jadi cuma prefix dari kolom itu (kolom pertama, atau kolom pertama+kedua, dst) yang bisa dipakai efisien oleh planner.
- "Kapan lebih baik pakai composite index dibanding dua index terpisah?" — kalau pola query paling umum memang memfilter kombinasi kolom itu bersamaan; dua index terpisah lebih fleksibel kalau kolom-kolom itu sering difilter sendiri-sendiri secara independen.

---

## 28. EXPLAIN ANALYZE

### Apa itu?
`EXPLAIN ANALYZE` adalah perintah Postgres yang benar-benar menjalankan sebuah query lalu menampilkan execution plan-nya secara detail: strategi apa yang dipakai planner (sequential scan, index scan, dst), berapa baris yang diproses tiap langkah, dan berapa lama tiap langkah makan waktu.

### Kenapa dibutuhkan?
Menambahkan index (topik 26, 27) itu gampang, tapi gak ada jaminan Postgres benar-benar memakainya untuk query tertentu — planner bisa saja memutuskan sequential scan tetap lebih murah tergantung ukuran tabel dan statistik data. `EXPLAIN ANALYZE` adalah satu-satunya cara pasti buat verifikasi apakah optimasi index yang dibuat beneran berpengaruh ke query real, bukan sekadar asumsi.

### Cara Kerja
```text
EXPLAIN ANALYZE SELECT id, user_id, status, total, version FROM orders WHERE user_id = 42;

                                                     QUERY PLAN
-----------------------------------------------------------------------------------------------
 Index Scan using idx_orders_user_id on orders  (cost=0.29..8.31 rows=3 width=48)
                                                 (actual time=0.021..0.024 rows=3 loops=1)
   Index Cond: (user_id = 42)
 Planning Time: 0.083 ms
 Execution Time: 0.041 ms
```
`Index Scan using idx_orders_user_id` artinya index-nya kepakai. Kalau baris pertama malah `Seq Scan on orders`, berarti Postgres milih baca seluruh tabel walau index-nya ada — biasanya karena tabelnya masih kecil (sequential scan justru lebih murah), atau statistik planner belum ter-update (perlu `ANALYZE orders;`).

### Contoh Kode — Go
```go
package db

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"
)

// ExplainOrdersByUser menjalankan EXPLAIN ANALYZE terhadap kueri "order
// milik user tertentu" (topik 26), buat lihat apakah Postgres benar-benar
// memakai idx_orders_user_id atau malah sequential scan.
func ExplainOrdersByUser(ctx context.Context, db *pgxpool.Pool, userID int64) ([]string, error) {
	rows, err := db.Query(ctx,
		`EXPLAIN ANALYZE SELECT id, user_id, status, total, version FROM orders WHERE user_id = $1`,
		userID)
	if err != nil {
		return nil, fmt.Errorf("explain analyze: %w", err)
	}
	defer rows.Close()

	var plan []string
	for rows.Next() {
		var line string
		if err := rows.Scan(&line); err != nil {
			return nil, err
		}
		plan = append(plan, line)
	}
	return plan, rows.Err()
}
```

### Contoh Kode — Node.js
```javascript
// explainOrdersByUser menjalankan EXPLAIN ANALYZE terhadap kueri "order
// milik user tertentu", buat lihat apakah Postgres benar-benar memakai
// idx_orders_user_id atau malah sequential scan.
async function explainOrdersByUser(pool, userId) {
  const result = await pool.query(
    'EXPLAIN ANALYZE SELECT id, user_id, status, total, version FROM orders WHERE user_id = $1',
    [userId]
  );
  return result.rows.map((row) => row['QUERY PLAN']);
}

module.exports = { explainOrdersByUser };
```

### Trade-off & Pitfall
- `EXPLAIN ANALYZE` (beda dari `EXPLAIN` biasa) benar-benar mengeksekusi query-nya — jangan pakai ini langsung di production buat query `DELETE`/`UPDATE` tanpa hati-hati, karena efeknya beneran terjadi ke data, bukan cuma simulasi. Untuk itu Postgres modern punya opsi `EXPLAIN (ANALYZE, ...)` yang bisa dibungkus transaction lalu di-rollback.
- Execution plan bisa berbeda antara environment development (data kecil) dan production (data besar/jutaan baris) — jangan cuma percaya hasil `EXPLAIN ANALYZE` dari database development yang datanya jauh lebih sedikit.
- Baca plan-nya dari dalam ke luar (node paling ke dalam dieksekusi duluan), bukan dari atas ke bawah — salah baca urutan eksekusi bisa bikin salah diagnosis bottleneck yang sebenarnya.

### Kapan Dipakai
Setiap kali curiga sebuah query lambat, atau sebagai verifikasi setelah menambahkan index baru — jangan cuma asumsi index dipakai, cek langsung lewat `EXPLAIN ANALYZE` sebelum dan sesudah perubahan.

### Sering Ditanya Saat Interview
- "Apa beda `EXPLAIN` dan `EXPLAIN ANALYZE`?" — `EXPLAIN` cuma menampilkan rencana eksekusi yang diestimasi planner tanpa benar-benar menjalankan query; `EXPLAIN ANALYZE` benar-benar mengeksekusinya dan menampilkan angka aktual (waktu, jumlah baris) selain estimasi.
- "Kenapa index sudah dibuat tapi `EXPLAIN ANALYZE` masih menunjukkan Seq Scan?" — bisa karena tabelnya masih kecil (sequential scan lebih murah untuk data sedikit), statistik planner belum ter-update (`ANALYZE` belum dijalankan), atau kondisi `WHERE`-nya gak match dengan kolom index.
- "Apa risiko menjalankan `EXPLAIN ANALYZE` untuk query `UPDATE`/`DELETE` di production?" — query itu benar-benar dieksekusi (bukan simulasi), jadi efeknya ke data beneran terjadi; sebaiknya dibungkus transaction yang di-rollback kalau cuma mau melihat plan-nya.

---

## 29. Transactions

### Apa itu?
Transaction adalah sekumpulan operasi database (`SELECT`, `INSERT`, `UPDATE`, dst) yang diperlakukan sebagai satu unit kerja: `BEGIN` untuk mulai, lalu diakhiri dengan `COMMIT` (semua perubahan disimpan permanen) atau `ROLLBACK` (semua perubahan dibatalkan, seolah gak pernah terjadi).

### Kenapa dibutuhkan?
`CreateOrder` di OrderFlow harus melakukan beberapa perubahan sekaligus: mengurangi `stock` tiap product yang dibeli, memasukkan baris baru ke `orders`, dan memasukkan baris-baris ke `order_items`. Kalau langkah kedua gagal (misalnya koneksi database putus) setelah langkah pertama (decrement stock) sudah jalan, tanpa transaction stock produk sudah berkurang padahal order-nya gak pernah benar-benar tercatat — data jadi rusak dan gak konsisten.

### Cara Kerja
```
tx := db.Begin()
  -> lock & cek stock tiap product (SELECT ... FOR UPDATE)
  -> UPDATE products SET stock = stock - qty
  -> INSERT INTO orders (...)
  -> INSERT INTO order_items (...)

kalau semua langkah sukses -> tx.Commit()   -> perubahan permanen
kalau ada satu langkah gagal -> tx.Rollback() -> SEMUA perubahan di atas dibatalkan,
                                                  seolah CreateOrder gak pernah dipanggil
```

### Contoh Kode — Go
```go
package db

import (
	"context"
	"errors"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"
)

var ErrInsufficientStock = errors.New("insufficient stock")

// CreateOrder membuat order baru beserta item-itemnya, dan mendekremen stock
// tiap product yang dibeli -- semua dalam satu transaction, supaya kalau ada
// bagian yang gagal (misalnya stock kurang), gak ada perubahan data yang
// tersimpan setengah jalan.
func CreateOrder(ctx context.Context, db *pgxpool.Pool, userID int64, items []OrderItem) (*Order, error) {
	tx, err := db.Begin(ctx)
	if err != nil {
		return nil, fmt.Errorf("begin tx: %w", err)
	}
	defer tx.Rollback(ctx) // no-op kalau tx.Commit sudah dipanggil duluan

	var total float64
	for _, item := range items {
		var stock int
		var price float64
		err := tx.QueryRow(ctx,
			`SELECT stock, price FROM products WHERE id = $1 FOR UPDATE`,
			item.ProductID,
		).Scan(&stock, &price)
		if err != nil {
			return nil, fmt.Errorf("get product %d: %w", item.ProductID, err)
		}
		if stock < item.Qty {
			return nil, fmt.Errorf("product %d: %w", item.ProductID, ErrInsufficientStock)
		}

		if _, err := tx.Exec(ctx,
			`UPDATE products SET stock = stock - $1 WHERE id = $2`,
			item.Qty, item.ProductID,
		); err != nil {
			return nil, fmt.Errorf("decrement stock %d: %w", item.ProductID, err)
		}
		total += price * float64(item.Qty)
	}

	var order Order
	err = tx.QueryRow(ctx,
		`INSERT INTO orders (user_id, status, total, version)
		 VALUES ($1, 'pending', $2, 1)
		 RETURNING id, user_id, status, total, version`,
		userID, total,
	).Scan(&order.ID, &order.UserID, &order.Status, &order.Total, &order.Version)
	if err != nil {
		return nil, fmt.Errorf("insert order: %w", err)
	}

	for _, item := range items {
		if _, err := tx.Exec(ctx,
			`INSERT INTO order_items (order_id, product_id, qty, price) VALUES ($1, $2, $3, $4)`,
			order.ID, item.ProductID, item.Qty, item.Price,
		); err != nil {
			return nil, fmt.Errorf("insert order item: %w", err)
		}
	}

	if err := tx.Commit(ctx); err != nil {
		return nil, fmt.Errorf("commit tx: %w", err)
	}
	return &order, nil
}
```

### Contoh Kode — Node.js
```javascript
// createOrder membuat order baru beserta item-itemnya, dan mendekremen stock
// tiap product yang dibeli -- semua dalam satu transaction.
async function createOrder(pool, userId, items) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    let total = 0;
    for (const item of items) {
      const { rows } = await client.query(
        'SELECT stock, price FROM products WHERE id = $1 FOR UPDATE',
        [item.productId]
      );
      if (rows.length === 0) {
        throw new Error(`product ${item.productId} not found`);
      }
      const { stock, price } = rows[0];
      if (stock < item.qty) {
        throw new Error(`product ${item.productId}: insufficient stock`);
      }

      await client.query(
        'UPDATE products SET stock = stock - $1 WHERE id = $2',
        [item.qty, item.productId]
      );
      total += price * item.qty;
    }

    const orderResult = await client.query(
      `INSERT INTO orders (user_id, status, total, version)
       VALUES ($1, 'pending', $2, 1)
       RETURNING id, user_id, status, total, version`,
      [userId, total]
    );
    const order = orderResult.rows[0];

    for (const item of items) {
      await client.query(
        'INSERT INTO order_items (order_id, product_id, qty, price) VALUES ($1, $2, $3, $4)',
        [order.id, item.productId, item.qty, item.price]
      );
    }

    await client.query('COMMIT');
    return order;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

module.exports = { createOrder };
```

### Trade-off & Pitfall
- `defer tx.Rollback(ctx)` di Go itu pola aman: kalau `tx.Commit()` sudah berhasil dipanggil, `Rollback` setelahnya jadi no-op; kalau ada `return` error di tengah jalan, `Rollback` otomatis kepanggil lewat `defer`. Lupa pola ini bikin koneksi transaction "menggantung" (idle in transaction) kalau ada early return yang lupa rollback manual.
- Transaction yang terlalu lama terbuka (misalnya karena ada pemanggilan API eksternal di tengah-tengah transaction) menahan lock lebih lama dari perlu, dan bisa memicu masalah lain seperti deadlock (topik 32) — sebisa mungkin transaction cuma berisi operasi database, bukan I/O eksternal.
- Node.js versi ini pakai `pool.connect()` untuk dapat satu client dedicated selama transaction berlangsung — jangan pakai `pool.query()` langsung untuk multi-statement transaction, karena tiap panggilan `pool.query()` bisa jatuh ke koneksi fisik yang berbeda-beda dari pool.

### Kapan Dipakai
Setiap kali ada lebih dari satu perubahan data yang harus sama-sama berhasil atau sama-sama gagal (all-or-nothing) — seperti `CreateOrder` yang mengubah `products.stock` dan menulis ke `orders`/`order_items` sekaligus.

### Sering Ditanya Saat Interview
- "Kenapa `CreateOrder` harus jalan di dalam transaction?" — karena ada beberapa perubahan data (decrement stock, insert order, insert order items) yang harus konsisten sebagai satu unit; kalau salah satu gagal di tengah jalan tanpa transaction, data jadi rusak setengah-setengah.
- "Apa yang terjadi kalau lupa `COMMIT` atau `ROLLBACK` transaction?" — koneksi tetap menahan transaction terbuka (idle in transaction), yang bisa menahan lock lebih lama dari seharusnya dan menghabiskan slot koneksi di pool.
- "Kenapa Node.js versi `createOrder` butuh `pool.connect()`, bukan langsung `pool.query()` berkali-kali?" — semua statement dalam satu transaction harus dijalankan lewat koneksi fisik yang sama; `pool.query()` biasa gak menjamin itu karena tiap panggilan bisa diambil dari koneksi pool yang berbeda.

---

## 30. ACID

### Apa itu?
ACID adalah empat properti yang dijamin oleh transaction database relasional seperti Postgres: **A**tomicity (all-or-nothing), **C**onsistency (data selalu valid sesuai constraint), **I**solation (transaction yang jalan bersamaan gak saling mengganggu), dan **D**urability (data yang sudah commit gak akan hilang walau server crash).

### Kenapa dibutuhkan?
`CreateOrder` (topik 29) memanfaatkan keempat properti ini sekaligus tanpa perlu implementasi manual: Atomicity dari `BEGIN`/`COMMIT`/`ROLLBACK`, Consistency dari constraint seperti `CHECK (stock >= 0)`, Isolation dari `FOR UPDATE` (dibahas lebih detail di topik 31 dan 34), dan Durability dari write-ahead log (WAL) Postgres. Tanpa jaminan ini, tim OrderFlow harus menulis ulang logic verifikasi konsistensi data secara manual di setiap tempat yang menyentuh database.

### Cara Kerja
```
Atomicity   : CreateOrder yang gagal di tengah -> tx.Rollback() -> semua perubahan hilang
Consistency : constraint CHECK (stock >= 0) di kolom products.stock -> INSERT/UPDATE
              yang bikin stock negatif langsung DITOLAK database, apapun bug di application code-nya
Isolation   : dua CreateOrder yang jalan bersamaan terhadap product yang sama
              -> FOR UPDATE bikin salah satu nunggu sampai yang lain commit
Durability  : setelah tx.Commit() sukses, perubahan sudah tertulis di WAL --
              server crash setelah itu gak menghilangkan data yang sudah di-commit
```

### Contoh Kode — Go
```go
package db

import (
	"context"

	"github.com/jackc/pgx/v5/pgxpool"
)

// AddStockConstraint menjaga Consistency: database sendiri yang menolak
// stock negatif, walau ada bug di application code yang lolos validasi.
func AddStockConstraint(ctx context.Context, pool *pgxpool.Pool) error {
	_, err := pool.Exec(ctx,
		`ALTER TABLE products ADD CONSTRAINT chk_stock_non_negative CHECK (stock >= 0)`)
	return err
}

// SetSynchronousCommit mengatur Durability level koneksi: kalau "on" (default
// Postgres), commit gak di-ack ke client sampai WAL record-nya benar-benar
// tertulis ke disk, jadi data yang sudah commit gak akan hilang walau server
// crash sesaat setelahnya.
func SetSynchronousCommit(ctx context.Context, pool *pgxpool.Pool, on bool) error {
	value := "off"
	if on {
		value = "on"
	}
	_, err := pool.Exec(ctx, "SET synchronous_commit = "+value)
	return err
}
```

### Contoh Kode — Node.js
```javascript
// addStockConstraint menjaga Consistency: database sendiri yang menolak
// stock negatif, walau ada bug di application code yang lolos validasi.
async function addStockConstraint(pool) {
  await pool.query(
    'ALTER TABLE products ADD CONSTRAINT chk_stock_non_negative CHECK (stock >= 0)'
  );
}

// setSynchronousCommit mengatur Durability level koneksi.
async function setSynchronousCommit(pool, on) {
  const value = on ? 'on' : 'off';
  await pool.query(`SET synchronous_commit = ${value}`);
}

module.exports = { addStockConstraint, setSynchronousCommit };
```

### Trade-off & Pitfall
- `synchronous_commit = off` bikin commit lebih cepat (gak nunggu WAL flush ke disk), tapi ada window kecil di mana transaction yang sudah di-ack ke client bisa hilang kalau server crash sebelum WAL-nya benar-benar tertulis — trade-off antara latency dan Durability, biasanya cuma dipakai untuk data yang gak kritis (bukan buat `CreateOrder`).
- `CHECK` constraint itu safety net terakhir, bukan pengganti validasi di application layer — validasi yang benar seharusnya menolak request gak valid lebih awal (dengan pesan error yang jelas ke user), constraint database cuma jaring pengaman kalau ada bug yang lolos.
- Isolation gak berarti "gak ada interaksi sama sekali" antar transaction — level isolation yang dipilih (topik 31) menentukan seberapa ketat interaksi itu dibatasi, dan level yang lebih ketat biasanya berarti lebih banyak transaction yang harus saling menunggu.

### Kapan Dipakai
ACID relevan setiap kali OrderFlow mengandalkan Postgres (atau database relasional manapun) buat data yang butuh jaminan konsistensi kuat — data finansial seperti `orders`, `payments`, dan `products.stock` adalah contoh paling jelas kenapa ACID penting dibanding database yang cuma menjamin eventual consistency.

### Sering Ditanya Saat Interview
- "Jelaskan ACID dengan contoh dari `CreateOrder`." — Atomicity: kalau gagal di tengah, semua perubahan (decrement stock, insert order) di-rollback; Consistency: constraint `CHECK (stock >= 0)` menolak update yang bikin data invalid; Isolation: `FOR UPDATE` mencegah dua `CreateOrder` bersamaan salah hitung stock produk yang sama; Durability: setelah commit sukses, data gak hilang walau crash.
- "Apa risiko `synchronous_commit = off`?" — ada window kecil di mana transaction yang sudah dianggap sukses oleh client bisa hilang kalau database crash sebelum WAL-nya benar-benar tertulis ke disk.
- "Kenapa gak cukup validasi stock cuma di application code, harus ada `CHECK` constraint juga?" — application code bisa punya bug atau ada jalur lain yang menyentuh database langsung (migrasi manual, service lain); `CHECK` constraint adalah jaminan terakhir di level database yang gak bisa dilewati oleh bug application code.

---

## 31. Isolation Levels

### Apa itu?
Isolation level menentukan seberapa "terlihat" perubahan dari transaction lain yang sedang berjalan bersamaan terhadap satu transaction tertentu. Postgres punya empat level standar SQL: Read Uncommitted (diperlakukan sama seperti Read Committed di Postgres), Read Committed (default), Repeatable Read, dan Serializable.

### Kenapa dibutuhkan?
Kalau dua `CreateOrder` jalan bersamaan terhadap product yang sama, level isolation yang dipilih menentukan apakah keduanya bisa "saling melihat" perubahan stock satu sama lain di tengah jalan (yang bisa menyebabkan anomali seperti dua transaction sama-sama mikir stock masih cukup padahal gabungan qty-nya melebihi stock), atau dijamin sepenuhnya terisolasi seperti dijalankan satu-satu secara berurutan.

### Cara Kerja
```
Read Committed (default Postgres):
  tiap statement lihat data yang sudah di-commit SAAT statement itu dijalankan
  -> dua SELECT di transaction yang sama bisa lihat hasil berbeda kalau ada
     transaction lain commit di antaranya (non-repeatable read)

Repeatable Read:
  seluruh transaction lihat snapshot data sejak transaction itu MULAI
  -> SELECT yang sama diulang, hasilnya konsisten sepanjang transaction

Serializable:
  seperti Repeatable Read, TAPI Postgres juga mendeteksi konflik antar
  transaction yang kalau dijalankan bersamaan bisa menghasilkan hasil yang
  beda dari kalau dijalankan berurutan -> salah satu transaction akan gagal
  dengan serialization error (kode 40001), caller wajib retry
```

### Contoh Kode — Go
```go
package db

import (
	"context"
	"errors"
	"fmt"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgconn"
	"github.com/jackc/pgx/v5/pgxpool"
)

// CreateOrderSerializable sama seperti CreateOrder (topik 29), tapi eksplisit
// pakai isolation level SERIALIZABLE -- mencegah anomali seperti write skew
// kalau banyak CreateOrder jalan bersamaan terhadap product yang sama.
func CreateOrderSerializable(ctx context.Context, db *pgxpool.Pool, userID int64, items []OrderItem) (*Order, error) {
	tx, err := db.BeginTx(ctx, pgx.TxOptions{IsoLevel: pgx.Serializable})
	if err != nil {
		return nil, fmt.Errorf("begin tx: %w", err)
	}
	defer tx.Rollback(ctx)

	var total float64
	for _, item := range items {
		var stock int
		var price float64
		if err := tx.QueryRow(ctx,
			`SELECT stock, price FROM products WHERE id = $1`,
			item.ProductID,
		).Scan(&stock, &price); err != nil {
			return nil, fmt.Errorf("get product %d: %w", item.ProductID, err)
		}
		if stock < item.Qty {
			return nil, fmt.Errorf("product %d: %w", item.ProductID, ErrInsufficientStock)
		}
		if _, err := tx.Exec(ctx,
			`UPDATE products SET stock = stock - $1 WHERE id = $2`,
			item.Qty, item.ProductID,
		); err != nil {
			return nil, fmt.Errorf("decrement stock %d: %w", item.ProductID, err)
		}
		total += price * float64(item.Qty)
	}

	var order Order
	err = tx.QueryRow(ctx,
		`INSERT INTO orders (user_id, status, total, version)
		 VALUES ($1, 'pending', $2, 1)
		 RETURNING id, user_id, status, total, version`,
		userID, total,
	).Scan(&order.ID, &order.UserID, &order.Status, &order.Total, &order.Version)
	if err != nil {
		return nil, fmt.Errorf("insert order: %w", err)
	}

	if err := tx.Commit(ctx); err != nil {
		var pgErr *pgconn.PgError
		if errors.As(err, &pgErr) && pgErr.Code == "40001" {
			return nil, fmt.Errorf("serialization conflict, retry: %w", err)
		}
		return nil, fmt.Errorf("commit tx: %w", err)
	}
	return &order, nil
}
```

### Contoh Kode — Node.js
```javascript
// createOrderSerializable sama seperti createOrder (topik 29), tapi eksplisit
// pakai isolation level SERIALIZABLE.
async function createOrderSerializable(pool, userId, items) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE');

    let total = 0;
    for (const item of items) {
      const { rows } = await client.query(
        'SELECT stock, price FROM products WHERE id = $1',
        [item.productId]
      );
      if (rows.length === 0) {
        throw new Error(`product ${item.productId} not found`);
      }
      const { stock, price } = rows[0];
      if (stock < item.qty) {
        throw new Error(`product ${item.productId}: insufficient stock`);
      }
      await client.query(
        'UPDATE products SET stock = stock - $1 WHERE id = $2',
        [item.qty, item.productId]
      );
      total += price * item.qty;
    }

    const orderResult = await client.query(
      `INSERT INTO orders (user_id, status, total, version)
       VALUES ($1, 'pending', $2, 1)
       RETURNING id, user_id, status, total, version`,
      [userId, total]
    );
    const order = orderResult.rows[0];

    try {
      await client.query('COMMIT');
    } catch (err) {
      // kode 40001 = serialization_failure, caller wajib retry
      if (err.code === '40001') {
        throw new Error('serialization conflict, retry');
      }
      throw err;
    }
    return order;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

module.exports = { createOrderSerializable };
```

### Trade-off & Pitfall
- `SERIALIZABLE` paling aman secara konsistensi, tapi transaction bisa gagal dengan serialization error walau gak ada error logis apapun di query-nya sendiri — caller (handler HTTP) wajib punya retry logic, kalau gak, user bakal lihat error yang seharusnya bisa sukses kalau dicoba ulang.
- Level isolation yang lebih ketat (Repeatable Read, Serializable) menjaga konsistensi lebih kuat, tapi berpotensi lebih banyak transaction gagal/nunggu satu sama lain dibanding Read Committed — ada trade-off langsung antara correctness dan throughput.
- Default Postgres (Read Committed) sudah cukup untuk banyak kasus karena kombinasi dengan `FOR UPDATE` (topik 34) sudah mencegah race condition yang paling umum; `SERIALIZABLE` biasanya baru dibutuhkan untuk kasus yang lebih kompleks (melibatkan banyak tabel/kondisi sekaligus).

### Kapan Dipakai
Level Read Committed (default) + `FOR UPDATE` di tempat yang tepat sudah cukup untuk kebanyakan operasi OrderFlow seperti `CreateOrder`. Naikkan ke `SERIALIZABLE` untuk operasi yang anomali-nya sulit dicegah cuma dengan row-level locking biasa, dan pastikan caller siap retry kalau kena serialization error.

### Sering Ditanya Saat Interview
- "Apa beda Read Committed dan Repeatable Read?" — Read Committed melihat data ter-commit terbaru di setiap statement baru (bisa beda antar statement dalam satu transaction), Repeatable Read mengunci snapshot data sejak transaction dimulai, jadi konsisten sepanjang transaction itu.
- "Kenapa `SERIALIZABLE` bisa membuat transaction gagal padahal query-nya benar?" — Postgres mendeteksi kondisi di mana eksekusi paralel transaction-transaction itu bisa menghasilkan hasil berbeda dibanding kalau dijalankan berurutan, lalu sengaja meng-abort salah satunya (serialization error 40001) supaya caller retry, daripada diam-diam menghasilkan data yang salah.
- "Apa konsekuensi memilih isolation level yang lebih ketat dari yang dibutuhkan?" — lebih banyak transaction yang saling menunggu atau gagal karena konflik, yang menurunkan throughput sistem walau data jadi lebih terjamin konsisten.

---

## 32. Deadlock

### Apa itu?
Deadlock terjadi ketika dua (atau lebih) transaction saling menunggu row yang dikunci satu sama lain, dan gak ada satupun yang bisa lanjut — transaction A menunggu row yang dikunci transaction B, sementara transaction B menunggu row yang dikunci transaction A. Postgres mendeteksi kondisi ini dan mem-abort salah satu transaction secara paksa (error `40P01`).

### Kenapa dibutuhkan?
`CreateOrder` mengunci row `products` lewat `FOR UPDATE` (topik 29, 34) untuk tiap item yang dibeli. Kalau urutan locking-nya gak konsisten — misalnya order A mengunci product 1 lalu product 2, sementara order B (yang jalan bersamaan) mengunci product 2 lalu product 1 — keduanya bisa saling menunggu, dan salah satunya bakal di-abort Postgres. Ini bug yang gampang lolos dari testing biasa karena cuma muncul di kondisi konkurensi tertentu.

### Cara Kerja
```
Order A: items = [product 1, product 2]     Order B: items = [product 2, product 1]

Waktu   Order A                              Order B
t1      lock product 1 (FOR UPDATE)  OK
t2                                           lock product 2 (FOR UPDATE)  OK
t3      lock product 2 -> MENUNGGU (dikunci B)
t4                                           lock product 1 -> MENUNGGU (dikunci A)
t5      DEADLOCK -- Postgres deteksi, abort salah satu transaction (error 40P01)
```
Fix standar: pastikan semua transaction mengunci row dalam **urutan yang konsisten** (misalnya selalu urut dari `product_id` terkecil), jadi gak mungkin ada dua transaction saling menunggu.

### Contoh Kode — Go
```go
package db

import (
	"context"
	"fmt"
	"sort"

	"github.com/jackc/pgx/v5/pgxpool"
)

// CreateOrderUnsafe mengunci row product persis sesuai urutan items yang
// dikirim client -- kalau dua request datang bersamaan dengan urutan produk
// terbalik (order A: [1, 2], order B: [2, 1]), keduanya bisa saling menunggu
// row yang sudah dikunci satu sama lain -> deadlock (Postgres error 40P01).
func CreateOrderUnsafe(ctx context.Context, db *pgxpool.Pool, userID int64, items []OrderItem) (*Order, error) {
	tx, err := db.Begin(ctx)
	if err != nil {
		return nil, fmt.Errorf("begin tx: %w", err)
	}
	defer tx.Rollback(ctx)

	var total float64
	for _, item := range items {
		var stock int
		var price float64
		if err := tx.QueryRow(ctx,
			`SELECT stock, price FROM products WHERE id = $1 FOR UPDATE`,
			item.ProductID,
		).Scan(&stock, &price); err != nil {
			return nil, fmt.Errorf("lock product %d: %w", item.ProductID, err)
		}
		if stock < item.Qty {
			return nil, fmt.Errorf("product %d: %w", item.ProductID, ErrInsufficientStock)
		}
		if _, err := tx.Exec(ctx,
			`UPDATE products SET stock = stock - $1 WHERE id = $2`,
			item.Qty, item.ProductID,
		); err != nil {
			return nil, fmt.Errorf("decrement stock %d: %w", item.ProductID, err)
		}
		total += price * float64(item.Qty)
	}

	var order Order
	if err := tx.QueryRow(ctx,
		`INSERT INTO orders (user_id, status, total, version)
		 VALUES ($1, 'pending', $2, 1)
		 RETURNING id, user_id, status, total, version`,
		userID, total,
	).Scan(&order.ID, &order.UserID, &order.Status, &order.Total, &order.Version); err != nil {
		return nil, fmt.Errorf("insert order: %w", err)
	}

	if err := tx.Commit(ctx); err != nil {
		return nil, fmt.Errorf("commit tx: %w", err)
	}
	return &order, nil
}

// CreateOrderSafeLockOrder FIX: mengurutkan items berdasarkan ProductID
// sebelum mengunci row-nya -- semua transaction yang bersaing sekarang
// selalu mengunci row product dengan urutan yang sama, jadi gak mungkin
// saling menunggu (consistent lock ordering, fix standar untuk deadlock).
func CreateOrderSafeLockOrder(ctx context.Context, db *pgxpool.Pool, userID int64, items []OrderItem) (*Order, error) {
	sorted := make([]OrderItem, len(items))
	copy(sorted, items)
	sort.Slice(sorted, func(i, j int) bool {
		return sorted[i].ProductID < sorted[j].ProductID
	})
	return CreateOrder(ctx, db, userID, sorted)
}
```

### Contoh Kode — Node.js
```javascript
// createOrderUnsafe mengunci row product persis sesuai urutan items yang
// dikirim client -- rawan deadlock kalau dua request datang bersamaan dengan
// urutan produk terbalik.
async function createOrderUnsafe(pool, userId, items) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    let total = 0;
    for (const item of items) {
      const { rows } = await client.query(
        'SELECT stock, price FROM products WHERE id = $1 FOR UPDATE',
        [item.productId]
      );
      if (rows.length === 0) {
        throw new Error(`product ${item.productId} not found`);
      }
      const { stock, price } = rows[0];
      if (stock < item.qty) {
        throw new Error(`product ${item.productId}: insufficient stock`);
      }
      await client.query(
        'UPDATE products SET stock = stock - $1 WHERE id = $2',
        [item.qty, item.productId]
      );
      total += price * item.qty;
    }

    const orderResult = await client.query(
      `INSERT INTO orders (user_id, status, total, version)
       VALUES ($1, 'pending', $2, 1)
       RETURNING id, user_id, status, total, version`,
      [userId, total]
    );
    await client.query('COMMIT');
    return orderResult.rows[0];
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

// createOrderSafeLockOrder FIX: mengurutkan items berdasarkan productId
// sebelum mengunci row-nya, supaya semua transaction yang bersaing mengunci
// row dengan urutan yang sama (consistent lock ordering).
async function createOrderSafeLockOrder(pool, userId, items) {
  const sorted = [...items].sort((a, b) => a.productId - b.productId);
  return createOrder(pool, userId, sorted);
}

module.exports = { createOrderUnsafe, createOrderSafeLockOrder };
```

### Trade-off & Pitfall
- Consistent lock ordering (sort dulu sebelum lock) itu solusi paling umum dan murah, tapi harus diterapkan di **semua** tempat yang mengunci row product yang sama — kalau ada satu jalur kode lain (misalnya proses admin bulk-update stock) yang mengunci tanpa urutan konsisten, deadlock tetap bisa terjadi.
- Deadlock yang berhasil dideteksi Postgres selalu bikin salah satu transaction gagal (bukan keduanya) — application code tetap harus siap menangani error itu (misalnya retry), sorting cuma mengurangi kemungkinannya terjadi, bukan menghilangkan semua kemungkinan error transaction sama sekali.
- `deadlock_timeout` Postgres (default 1 detik) menentukan seberapa lama Postgres menunggu sebelum benar-benar memeriksa siklus deadlock — nilai yang terlalu tinggi bikin transaction yang stuck terasa lambat sebelum akhirnya di-abort.

### Kapan Dipakai
Waspadai risiko deadlock setiap kali ada lebih dari satu row yang dikunci dalam satu transaction (seperti `CreateOrder` yang mengunci banyak `products` sekaligus) dan ada kemungkinan banyak transaction serupa jalan bersamaan — terapkan consistent lock ordering sebagai kebiasaan default, bukan cuma reaksi setelah insiden.

### Sering Ditanya Saat Interview
- "Gimana caramu mencegah deadlock di `CreateOrder`?" — mengurutkan item berdasarkan `product_id` sebelum mengunci row-nya (consistent lock ordering), supaya semua transaction yang bersaing terhadap product yang sama selalu mengunci dengan urutan yang sama.
- "Apa yang dilakukan Postgres kalau mendeteksi deadlock?" — Postgres mendeteksi siklus tunggu antar transaction, lalu memilih salah satu transaction untuk di-abort paksa (error `40P01`), supaya transaction lainnya bisa lanjut.
- "Apakah consistent lock ordering menghilangkan semua kemungkinan error deadlock?" — mengurangi drastis kemungkinannya untuk kasus yang sudah dipikirkan, tapi kalau ada jalur kode lain yang mengunci row yang sama tanpa mengikuti urutan yang sama, deadlock tetap mungkin terjadi.

---

## 33. Optimistic Locking

### Apa itu?
Optimistic locking adalah strategi menangani concurrent update dengan asumsi konflik itu jarang terjadi: setiap baris punya kolom `version` yang bertambah tiap kali di-update, dan sebuah update cuma berhasil kalau `version` yang dikirim caller masih sama dengan `version` di database saat itu. Kalau sudah beda (ada proses lain yang update duluan), update itu ditolak, bukan langsung dikunci di awal seperti pessimistic locking.

### Kenapa dibutuhkan?
Update status order (misalnya dari `pending` ke `shipped`) di OrderFlow bisa dipicu dari beberapa jalur berbeda secara bersamaan — customer service yang update manual, webhook dari partner logistik, dan job otomatis yang mengecek pembayaran. Tanpa mekanisme apapun, update terakhir yang jalan bisa menimpa perubahan proses lain secara diam-diam (lost update), tanpa ada yang tau ada konflik.

### Cara Kerja
```
Client A baca order: {id: 7, status: "pending", version: 3}
Client B baca order: {id: 7, status: "pending", version: 3}   (versi yang sama)

Client A: UPDATE orders SET status='shipped', version=4 WHERE id=7 AND version=3
          -> match, version di DB masih 3 -> berhasil, version sekarang jadi 4

Client B: UPDATE orders SET status='cancelled', version=4 WHERE id=7 AND version=3
          -> TIDAK match, version di DB sudah 4 (bukan 3 lagi) -> 0 baris ter-update
          -> Client B tau updatenya gagal karena konflik, harus re-fetch data terbaru
             lalu putuskan mau retry atau kasih tau user
```

### Contoh Kode — Go
```go
package db

import (
	"context"
	"errors"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"
)

var ErrVersionConflict = errors.New("order was modified by another process")

// UpdateOrderStatus optimistic locking: update cuma berhasil kalau
// expectedVersion masih sama dengan Order.version di database saat ini.
// Kalau ada proses lain yang sudah update duluan, version di DB udah beda,
// kondisi WHERE gak match, RowsAffected() == 0 -> caller tau harus
// re-fetch order terbaru lalu retry.
func UpdateOrderStatus(ctx context.Context, db *pgxpool.Pool, orderID int64, expectedVersion int, newStatus string) error {
	tag, err := db.Exec(ctx,
		`UPDATE orders
		 SET status = $1, version = version + 1
		 WHERE id = $2 AND version = $3`,
		newStatus, orderID, expectedVersion)
	if err != nil {
		return fmt.Errorf("update order status: %w", err)
	}
	if tag.RowsAffected() == 0 {
		return ErrVersionConflict
	}
	return nil
}
```

### Contoh Kode — Node.js
```javascript
// updateOrderStatus optimistic locking: update cuma berhasil kalau
// expectedVersion masih sama dengan version di database saat ini.
async function updateOrderStatus(pool, orderId, expectedVersion, newStatus) {
  const result = await pool.query(
    `UPDATE orders
     SET status = $1, version = version + 1
     WHERE id = $2 AND version = $3`,
    [newStatus, orderId, expectedVersion]
  );
  if (result.rowCount === 0) {
    throw new Error('order was modified by another process');
  }
}

module.exports = { updateOrderStatus };
```

### Trade-off & Pitfall
- Optimistic locking gak menahan lock sama sekali sampai update dijalankan, jadi throughput-nya bagus untuk kasus konflik jarang — tapi kalau konflik sering terjadi (banyak client update baris yang sama secara bersamaan), banyak request bakal gagal dan harus retry berulang-ulang, yang justru lebih boros daripada pessimistic locking.
- `ErrVersionConflict` harus ditangani eksplisit oleh caller (biasanya dengan re-fetch data terbaru dan menawarkan retry ke user atau proses otomatis) — kalau cuma dibiarkan jadi error generic 500, user gak akan tau bahwa itu sebenarnya konflik yang bisa diselesaikan dengan mencoba lagi.
- Optimistic locking cuma melindungi kolom yang benar-benar dicek `version`-nya di kondisi `WHERE` — kalau ada kode lain yang update `orders.status` langsung tanpa ikut mengecek `version`, mekanisme ini gampang "dilewati" secara gak sengaja.

### Kapan Dipakai
Cocok untuk data yang jarang di-update bersamaan tapi tetap butuh deteksi konflik, seperti update status order dari beberapa sumber berbeda — beda dengan `products.stock` yang selalu berpotensi tinggi konflik saat flash sale (lebih cocok pessimistic locking, topik 34).

### Sering Ditanya Saat Interview
- "Apa beda optimistic locking dan pessimistic locking?" — optimistic locking gak mengunci apapun di awal, cuma mendeteksi konflik lewat kolom version saat update; pessimistic locking langsung mengunci row sejak awal transaction (`SELECT ... FOR UPDATE`), jadi transaction lain harus menunggu.
- "Kenapa `orders` pakai optimistic locking, sementara `products.stock` pakai pessimistic locking?" — status order jarang di-update bersamaan oleh banyak proses sekaligus dan gak butuh lock ketat; stock produk sangat rawan konflik tinggi (banyak `CreateOrder` bersaing memperebutkan stock yang sama), jadi butuh lock eksplisit dari awal.
- "Apa yang harus dilakukan aplikasi kalau `UpdateOrderStatus` gagal karena `ErrVersionConflict`?" — ambil ulang data order terbaru dari database, evaluasi apakah perubahan yang diinginkan masih relevan, lalu retry update dengan version yang baru, atau beri tahu user bahwa datanya sudah berubah.

---

## 34. Pessimistic Locking

### Apa itu?
Pessimistic locking adalah strategi mengunci row database sejak awal transaction lewat `SELECT ... FOR UPDATE`, sebelum ada perubahan apapun dibuat — transaction lain yang mencoba mengunci row yang sama harus menunggu sampai transaction pertama selesai (`COMMIT`/`ROLLBACK`), daripada baru terdeteksi konflik belakangan seperti optimistic locking.

### Kenapa dibutuhkan?
`CreateOrder` harus mengecek `stock` produk lalu menguranginya — dua operasi terpisah yang, tanpa lock, rawan race condition: dua request `CreateOrder` bersamaan sama-sama membaca `stock = 5`, sama-sama mikir stok cukup buat beli 5 unit, sama-sama commit, padahal gabungan qty-nya 10 dan stock cuma 5 (overselling). `FOR UPDATE` mengunci baris `products` itu sejak dibaca, jadi request kedua wajib menunggu request pertama commit dulu, dan bakal melihat stock yang sudah ter-update.

### Cara Kerja
```
BUG (tanpa FOR UPDATE), stock produk 42 = 5, dua request bersamaan beli 5 unit:

Request A: SELECT stock FROM products WHERE id=42        -> baca stock=5
Request B: SELECT stock FROM products WHERE id=42        -> baca stock=5 (bersamaan, gak dikunci)
Request A: stock(5) >= qty(5) -> lolos -> UPDATE stock = stock - 5 -> commit, stock jadi 0
Request B: stock(5) >= qty(5) -> lolos (padahal harusnya udah 0!) -> UPDATE stock = stock - 5
           -> commit, stock jadi -5 -> OVERSELLING, dua order sukses padahal stock cuma cukup untuk satu

FIX (dengan FOR UPDATE):
Request A: SELECT stock FROM products WHERE id=42 FOR UPDATE  -> dapat lock, baca stock=5
Request B: SELECT stock FROM products WHERE id=42 FOR UPDATE  -> MENUNGGU (row dikunci A)
Request A: UPDATE stock = stock - 5 -> commit -> lock dilepas, stock jadi 0
Request B: lanjut, baca ulang stock -> SEKARANG 0 -> stock(0) < qty(5) -> ditolak, ErrInsufficientStock
```

### Contoh Kode — Go
```go
package db

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"
)

// CreateOrderNoLock BUG: baca stock TANPA FOR UPDATE, lalu decrement di
// statement terpisah -- dua request bersamaan bisa sama-sama lolos cek
// stock sebelum salah satunya sempat commit, menyebabkan overselling.
func CreateOrderNoLock(ctx context.Context, db *pgxpool.Pool, userID int64, items []OrderItem) (*Order, error) {
	tx, err := db.Begin(ctx)
	if err != nil {
		return nil, fmt.Errorf("begin tx: %w", err)
	}
	defer tx.Rollback(ctx)

	var total float64
	for _, item := range items {
		var stock int
		var price float64
		// BUG: gak ada FOR UPDATE, row products gak dikunci saat dibaca.
		if err := tx.QueryRow(ctx,
			`SELECT stock, price FROM products WHERE id = $1`,
			item.ProductID,
		).Scan(&stock, &price); err != nil {
			return nil, fmt.Errorf("get product %d: %w", item.ProductID, err)
		}
		if stock < item.Qty {
			return nil, fmt.Errorf("product %d: %w", item.ProductID, ErrInsufficientStock)
		}
		if _, err := tx.Exec(ctx,
			`UPDATE products SET stock = stock - $1 WHERE id = $2`,
			item.Qty, item.ProductID,
		); err != nil {
			return nil, fmt.Errorf("decrement stock %d: %w", item.ProductID, err)
		}
		total += price * float64(item.Qty)
	}

	var order Order
	if err := tx.QueryRow(ctx,
		`INSERT INTO orders (user_id, status, total, version)
		 VALUES ($1, 'pending', $2, 1)
		 RETURNING id, user_id, status, total, version`,
		userID, total,
	).Scan(&order.ID, &order.UserID, &order.Status, &order.Total, &order.Version); err != nil {
		return nil, fmt.Errorf("insert order: %w", err)
	}

	if err := tx.Commit(ctx); err != nil {
		return nil, fmt.Errorf("commit tx: %w", err)
	}
	return &order, nil
}

// CreateOrder FIX: SELECT ... FOR UPDATE mengunci row products sejak
// dibaca, jadi request lain yang mengincar product yang sama wajib
// menunggu transaction ini selesai sebelum bisa baca ulang stock terbaru.
// (Ini fungsi CreateOrder yang sama seperti topik 29.)
func CreateOrder(ctx context.Context, db *pgxpool.Pool, userID int64, items []OrderItem) (*Order, error) {
	tx, err := db.Begin(ctx)
	if err != nil {
		return nil, fmt.Errorf("begin tx: %w", err)
	}
	defer tx.Rollback(ctx)

	var total float64
	for _, item := range items {
		var stock int
		var price float64
		if err := tx.QueryRow(ctx,
			`SELECT stock, price FROM products WHERE id = $1 FOR UPDATE`,
			item.ProductID,
		).Scan(&stock, &price); err != nil {
			return nil, fmt.Errorf("get product %d: %w", item.ProductID, err)
		}
		if stock < item.Qty {
			return nil, fmt.Errorf("product %d: %w", item.ProductID, ErrInsufficientStock)
		}
		if _, err := tx.Exec(ctx,
			`UPDATE products SET stock = stock - $1 WHERE id = $2`,
			item.Qty, item.ProductID,
		); err != nil {
			return nil, fmt.Errorf("decrement stock %d: %w", item.ProductID, err)
		}
		total += price * float64(item.Qty)
	}

	var order Order
	if err := tx.QueryRow(ctx,
		`INSERT INTO orders (user_id, status, total, version)
		 VALUES ($1, 'pending', $2, 1)
		 RETURNING id, user_id, status, total, version`,
		userID, total,
	).Scan(&order.ID, &order.UserID, &order.Status, &order.Total, &order.Version); err != nil {
		return nil, fmt.Errorf("insert order: %w", err)
	}

	if err := tx.Commit(ctx); err != nil {
		return nil, fmt.Errorf("commit tx: %w", err)
	}
	return &order, nil
}
```

### Contoh Kode — Node.js
```javascript
// createOrderNoLock BUG: baca stock TANPA FOR UPDATE -- dua request
// bersamaan bisa sama-sama lolos cek stock sebelum salah satunya sempat
// commit, menyebabkan overselling.
async function createOrderNoLock(pool, userId, items) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    let total = 0;
    for (const item of items) {
      // BUG: gak ada FOR UPDATE, row products gak dikunci saat dibaca.
      const { rows } = await client.query(
        'SELECT stock, price FROM products WHERE id = $1',
        [item.productId]
      );
      if (rows.length === 0) {
        throw new Error(`product ${item.productId} not found`);
      }
      const { stock, price } = rows[0];
      if (stock < item.qty) {
        throw new Error(`product ${item.productId}: insufficient stock`);
      }
      await client.query(
        'UPDATE products SET stock = stock - $1 WHERE id = $2',
        [item.qty, item.productId]
      );
      total += price * item.qty;
    }

    const orderResult = await client.query(
      `INSERT INTO orders (user_id, status, total, version)
       VALUES ($1, 'pending', $2, 1)
       RETURNING id, user_id, status, total, version`,
      [userId, total]
    );
    await client.query('COMMIT');
    return orderResult.rows[0];
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

// createOrder FIX: SELECT ... FOR UPDATE mengunci row products sejak
// dibaca. (Ini fungsi createOrder yang sama seperti topik 29.)
async function createOrder(pool, userId, items) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    let total = 0;
    for (const item of items) {
      const { rows } = await client.query(
        'SELECT stock, price FROM products WHERE id = $1 FOR UPDATE',
        [item.productId]
      );
      if (rows.length === 0) {
        throw new Error(`product ${item.productId} not found`);
      }
      const { stock, price } = rows[0];
      if (stock < item.qty) {
        throw new Error(`product ${item.productId}: insufficient stock`);
      }
      await client.query(
        'UPDATE products SET stock = stock - $1 WHERE id = $2',
        [item.qty, item.productId]
      );
      total += price * item.qty;
    }

    const orderResult = await client.query(
      `INSERT INTO orders (user_id, status, total, version)
       VALUES ($1, 'pending', $2, 1)
       RETURNING id, user_id, status, total, version`,
      [userId, total]
    );
    await client.query('COMMIT');
    return orderResult.rows[0];
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

module.exports = { createOrderNoLock, createOrder };
```

### Trade-off & Pitfall
- `FOR UPDATE` mengunci row sepanjang transaction berjalan — kalau transaction-nya lama (misalnya ada logic lambat lain di antara `SELECT ... FOR UPDATE` dan `COMMIT`), request lain yang mengincar product yang sama harus menunggu lebih lama, menurunkan throughput saat flash sale rame-ramenya.
- Pessimistic locking cuma efektif kalau **semua** jalur kode yang mengubah `stock` konsisten pakai `FOR UPDATE` — kalau ada satu endpoint lain (misalnya admin panel restock) yang update stock tanpa lock, race condition tetap bisa terjadi lewat jalur itu.
- Dibanding optimistic locking (topik 33), pessimistic locking lebih boros kalau konflik jarang terjadi (row terkunci padahal gak ada yang benar-benar rebutan), tapi jauh lebih aman untuk kasus dengan concurrency tinggi seperti stock produk populer.

### Kapan Dipakai
Wajib untuk data dengan risiko concurrent write tinggi yang harus benar-benar akurat, seperti `products.stock` saat `CreateOrder` — beda dengan update status order (topik 33) yang cukup pakai optimistic locking karena konfliknya jarang terjadi.

### Sering Ditanya Saat Interview
- "Kenapa `SELECT stock` biasa aja gak cukup, harus `FOR UPDATE`?" — tanpa `FOR UPDATE`, dua transaction bisa sama-sama membaca stock yang sama sebelum salah satunya sempat commit perubahan, sehingga keduanya lolos pengecekan padahal gabungan permintaannya melebihi stock yang tersedia (overselling).
- "Apa downside `FOR UPDATE` dibanding gak pakai lock sama sekali?" — request lain yang butuh row yang sama harus menunggu sampai transaction yang memegang lock selesai, yang bisa jadi bottleneck kalau banyak request bersaing terhadap product populer yang sama secara bersamaan.
- "Kalau flash sale bikin banyak `CreateOrder` mengincar satu product yang sama, apa yang terjadi ke request-request itu?" — mereka akan antre menunggu lock `FOR UPDATE` dilepas satu per satu (serial terhadap product itu), bukan diproses paralel — ini yang bikin desain locking di sekitar hot row jadi penting untuk performa saat traffic tinggi.

---

## 35. N+1 Query

### Apa itu?
N+1 query adalah anti-pattern di mana kode menjalankan 1 query untuk ambil daftar data (misalnya daftar `order_items`), lalu menjalankan 1 query tambahan untuk masing-masing baris hasil itu (misalnya ambil detail product tiap item) — total jadi 1 + N query, padahal seringkali bisa digabung jadi 1 query saja pakai `JOIN`.

### Kenapa dibutuhkan?
Menampilkan ringkasan satu order butuh nama tiap product yang dibeli, bukan cuma `product_id`-nya. Kalau ditulis naif — loop `order_items` lalu panggil `GetProductByID` di dalam loop itu — order dengan 20 item bakal memicu 21 query database buat satu request HTTP, yang jauh lebih lambat (dan lebih membebani database) dibanding 1 query `JOIN` yang mengambil semuanya sekaligus.

### Cara Kerja
```
BUG (N+1):
  SELECT product_id, qty, price FROM order_items WHERE order_id = 7   -- 1 query, misal 20 baris
  for setiap baris:
    SELECT id, name, price, stock FROM products WHERE id = <product_id>  -- 20 query tambahan!
  total: 1 + 20 = 21 query buat satu order

FIX (JOIN):
  SELECT oi.product_id, p.name, oi.qty, oi.price
  FROM order_items oi
  JOIN products p ON p.id = oi.product_id
  WHERE oi.order_id = 7
  -- 1 query, gak peduli berapa banyak item di order itu
```

### Contoh Kode — Go
```go
package db

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"
)

// OrderItemWithProduct gabungan order_item + detail product-nya, dipakai
// buat nampilin ringkasan order ke user.
type OrderItemWithProduct struct {
	ProductID   int64
	ProductName string
	Qty         int
	Price       float64
}

// GetOrderItemsNPlusOne BUG KLASIK: 1 query ambil order_items, lalu 1 query
// tambahan PER item buat ambil nama product-nya (panggil GetProductByID di
// dalam loop) -> total 1 + N query untuk order dengan N item.
func GetOrderItemsNPlusOne(ctx context.Context, db *pgxpool.Pool, orderID int64) ([]OrderItemWithProduct, error) {
	rows, err := db.Query(ctx,
		`SELECT product_id, qty, price FROM order_items WHERE order_id = $1`,
		orderID)
	if err != nil {
		return nil, fmt.Errorf("get order items: %w", err)
	}
	defer rows.Close()

	var items []OrderItemWithProduct
	for rows.Next() {
		var productID int64
		var qty int
		var price float64
		if err := rows.Scan(&productID, &qty, &price); err != nil {
			return nil, err
		}

		// query tambahan per item -- ini yang bikin N+1
		product, err := GetProductByID(ctx, db, productID)
		if err != nil {
			return nil, fmt.Errorf("get product %d: %w", productID, err)
		}
		name := "unknown"
		if product != nil {
			name = product.Name
		}
		items = append(items, OrderItemWithProduct{
			ProductID:   productID,
			ProductName: name,
			Qty:         qty,
			Price:       price,
		})
	}
	return items, rows.Err()
}

// GetOrderItemsJoin FIX: satu query JOIN, gak peduli berapa banyak item di
// dalam order, tetap 1 round-trip ke database.
func GetOrderItemsJoin(ctx context.Context, db *pgxpool.Pool, orderID int64) ([]OrderItemWithProduct, error) {
	rows, err := db.Query(ctx,
		`SELECT oi.product_id, p.name, oi.qty, oi.price
		 FROM order_items oi
		 JOIN products p ON p.id = oi.product_id
		 WHERE oi.order_id = $1`,
		orderID)
	if err != nil {
		return nil, fmt.Errorf("get order items: %w", err)
	}
	defer rows.Close()

	var items []OrderItemWithProduct
	for rows.Next() {
		var item OrderItemWithProduct
		if err := rows.Scan(&item.ProductID, &item.ProductName, &item.Qty, &item.Price); err != nil {
			return nil, err
		}
		items = append(items, item)
	}
	return items, rows.Err()
}
```

### Contoh Kode — Node.js
```javascript
// getOrderItemsNPlusOne BUG KLASIK: 1 query ambil order_items, lalu 1 query
// tambahan PER item buat ambil nama product-nya -> total 1 + N query.
async function getOrderItemsNPlusOne(pool, orderId) {
  const { rows } = await pool.query(
    'SELECT product_id, qty, price FROM order_items WHERE order_id = $1',
    [orderId]
  );

  const items = [];
  for (const row of rows) {
    // query tambahan per item -- ini yang bikin N+1
    const product = await getProductById(pool, row.product_id);
    items.push({
      productId: row.product_id,
      productName: product ? product.name : 'unknown',
      qty: row.qty,
      price: row.price,
    });
  }
  return items;
}

// getOrderItemsJoin FIX: satu query JOIN, tetap 1 round-trip ke database
// berapapun banyak item di dalam order.
async function getOrderItemsJoin(pool, orderId) {
  const { rows } = await pool.query(
    `SELECT oi.product_id AS "productId", p.name AS "productName", oi.qty, oi.price
     FROM order_items oi
     JOIN products p ON p.id = oi.product_id
     WHERE oi.order_id = $1`,
    [orderId]
  );
  return rows;
}

module.exports = { getOrderItemsNPlusOne, getOrderItemsJoin };
```

### Trade-off & Pitfall
- `JOIN` menyelesaikan N+1 untuk kasus yang datanya berasal dari relasi langsung seperti ini, tapi untuk kasus yang lebih kompleks (misalnya ambil data dari service lain lewat HTTP, bukan tabel di database yang sama) solusinya beda — biasanya pakai batching (satu request "ambil banyak ID sekaligus") daripada `JOIN`.
- `JOIN` yang menggabungkan tabel besar bisa menghasilkan row set yang lebih besar dari yang dibutuhkan kalau relasinya one-to-many dan ada kolom lain yang ikut di-duplikasi per baris — perlu diperhatikan terutama kalau order-nya bisa punya ratusan item.
- N+1 sering gak kelihatan di testing dengan data kecil (1-2 item per order) karena bedanya cuma beberapa milidetik — masalahnya baru kelihatan jelas di production dengan data besar atau saat load tinggi, jadi penting dicek lewat code review atau query logging, bukan cuma manual testing.

### Kapan Dipakai
Selalu waspada N+1 setiap kali ada loop yang di dalamnya memanggil fungsi lain yang query ke database (atau ke service lain) — kalau data yang dibutuhkan sebenarnya berasal dari relasi tabel yang sama, hampir selalu lebih baik digabung jadi satu query lewat `JOIN`.

### Sering Ditanya Saat Interview
- "Apa itu N+1 query problem dan gimana cara mendeteksinya?" — pola di mana 1 query awal diikuti N query tambahan (satu per baris hasil query awal); bisa dideteksi lewat query logging/APM yang menunjukkan jumlah query per request, atau code review yang curiga terhadap query di dalam loop.
- "Kenapa `JOIN` lebih baik daripada loop yang manggil `GetProductByID` satu-satu?" — `JOIN` menggabungkan pengambilan data jadi satu round-trip ke database, jauh lebih efisien secara network latency dan beban database dibanding N round-trip terpisah.
- "Apakah N+1 selalu bisa diselesaikan dengan `JOIN`?" — kalau data N-nya berasal dari database yang sama, hampir selalu iya; kalau berasal dari service eksternal lewat HTTP, solusinya biasanya batching request (kirim semua ID sekaligus dalam satu request) daripada `JOIN` biasa.

---

## 36. Connection Pooling

### Apa itu?
Connection pooling adalah teknik menjaga sekumpulan koneksi database yang sudah terbuka dan dipakai bergantian oleh banyak request, alih-alih membuka koneksi TCP baru ke Postgres setiap kali ada request masuk dan menutupnya setelah selesai.

### Kenapa dibutuhkan?
Membuka koneksi baru ke Postgres itu gak murah — ada overhead TCP handshake, autentikasi, dan alokasi resource di sisi Postgres untuk tiap koneksi baru. Kalau OrderFlow membuka koneksi baru untuk setiap request HTTP (yang bisa ribuan per detik saat traffic tinggi), overhead itu menumpuk dan Postgres bisa kehabisan slot koneksi (`max_connections`) jauh sebelum CPU/memory-nya benar-benar penuh.

### Cara Kerja
```
Tanpa pooling:  request masuk -> buka koneksi baru -> query -> tutup koneksi
                (overhead handshake TCP + auth di setiap request)

Dengan pooling: startup service -> buka N koneksi sekaligus, simpan di pool
                request masuk -> pinjam 1 koneksi dari pool -> query
                                -> kembalikan koneksi ke pool (bukan ditutup)
                request berikutnya bisa langsung pakai koneksi yang sama
```

### Contoh Kode — Go
```go
package db

import (
	"context"
	"fmt"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
)

// NewPool bikin connection pool pgx yang dipakai bareng-bareng oleh seluruh
// goroutine handler HTTP OrderFlow -- bukan bikin koneksi baru tiap request.
func NewPool(ctx context.Context, dsn string) (*pgxpool.Pool, error) {
	cfg, err := pgxpool.ParseConfig(dsn)
	if err != nil {
		return nil, fmt.Errorf("parse dsn: %w", err)
	}

	cfg.MaxConns = 20                      // batas atas koneksi ke Postgres
	cfg.MinConns = 5                       // koneksi idle yang selalu siap dipakai
	cfg.MaxConnLifetime = time.Hour        // recycle koneksi lama secara berkala
	cfg.MaxConnIdleTime = 10 * time.Minute // tutup koneksi idle yang gak kepake

	pool, err := pgxpool.NewWithConfig(ctx, cfg)
	if err != nil {
		return nil, fmt.Errorf("create pool: %w", err)
	}
	if err := pool.Ping(ctx); err != nil {
		return nil, fmt.Errorf("ping pool: %w", err)
	}
	return pool, nil
}
```

### Contoh Kode — Node.js
```javascript
const { Pool } = require('pg');

// createPool bikin connection pool pg yang dipakai bareng-bareng oleh
// seluruh request handler OrderFlow -- bukan bikin koneksi baru tiap request.
function createPool(connectionString) {
  return new Pool({
    connectionString,
    max: 20,                            // batas atas koneksi ke Postgres
    idleTimeoutMillis: 10 * 60 * 1000,   // tutup koneksi idle yang gak kepake
    connectionTimeoutMillis: 5000,       // gagal cepat kalau pool penuh & gak dapat slot
  });
}

module.exports = { createPool };
```

### Trade-off & Pitfall
- `MaxConns`/`max` yang terlalu besar bisa bikin Postgres kehabisan `max_connections` kalau ada banyak instance service yang masing-masing punya pool sendiri (misalnya 10 instance x `max: 20` = 200 koneksi) — perlu dihitung total across semua instance, bukan cuma per instance.
- Pool yang terlalu kecil bikin request harus antre menunggu koneksi kosong saat traffic tinggi, walau Postgres-nya sendiri masih punya kapasitas — `connectionTimeoutMillis`/timeout yang wajar penting supaya request gagal cepat dengan error yang jelas, bukan menggantung lama.
- Koneksi yang gak pernah di-recycle (`MaxConnLifetime`) bisa jadi masalah kalau ada perubahan konfigurasi di sisi Postgres/load balancer yang butuh koneksi baru untuk diterapkan — koneksi lama yang bertahan terlalu lama gak ikut menerima perubahan itu.

### Kapan Dipakai
Selalu — connection pooling itu default yang wajar untuk hampir semua service backend yang bicara ke database relasional, termasuk OrderFlow. Ukuran pool-nya yang perlu disesuaikan dengan kapasitas Postgres dan jumlah instance service yang berjalan.

### Sering Ditanya Saat Interview
- "Kenapa gak buka koneksi baru saja tiap request, lebih simpel?" — overhead TCP handshake dan autentikasi per koneksi baru signifikan dibanding query itu sendiri, dan Postgres bisa cepat kehabisan slot koneksi (`max_connections`) kalau volume request tinggi.
- "Gimana caramu menentukan ukuran pool yang tepat?" — pertimbangkan `max_connections` Postgres dibagi jumlah instance service yang berjalan bersamaan, plus headroom untuk koneksi lain (migrasi, monitoring); jangan cuma pasang angka besar tanpa hitungan.
- "Apa yang terjadi kalau semua koneksi di pool sedang dipakai dan ada request baru masuk?" — request itu menunggu (antre) sampai ada koneksi yang dikembalikan ke pool, dibatasi oleh connection timeout; kalau timeout habis, request gagal dengan error yang jelas daripada menggantung selamanya.

---

## 37. Read Replica

### Apa itu?
Read replica adalah salinan database yang terus-menerus disinkronkan dari database utama (primary) lewat replication (topik 38), dan cuma dipakai untuk query baca (`SELECT`) — semua operasi tulis (`INSERT`/`UPDATE`/`DELETE`) tetap harus lewat primary.

### Kenapa dibutuhkan?
Katalog produk OrderFlow jauh lebih sering dibaca (user browsing, search) daripada ditulis (admin update produk sesekali). Kalau semua query baca dan tulis dibebankan ke satu database yang sama, primary bisa kewalahan saat traffic baca tinggi, padahal traffic itu sebenarnya bisa dialihkan ke replica supaya primary fokus menangani write yang lebih sensitif seperti `CreateOrder`.

### Cara Kerja
```
Write (INSERT/UPDATE/DELETE)  -> SELALU ke Primary
                                  Primary replikasi perubahan ke Replica (async)

Read (SELECT) yang gak butuh data paling baru-baru amat -> boleh ke Replica
Read yang butuh data paling up-to-date (misalnya baca ulang order yang baru
dibuat sendiri) -> tetap ke Primary, karena replikasi ke Replica ada jeda
(replication lag, topik 38)
```

### Contoh Kode — Go
```go
package db

import (
	"context"

	"github.com/jackc/pgx/v5/pgxpool"
)

// Pools memisahkan pool koneksi ke primary (writer) dan replica (reader).
// Query yang cuma baca (misal ListProducts) diarahkan ke replica supaya gak
// membebani primary yang harus menangani semua write (termasuk CreateOrder).
type Pools struct {
	Primary *pgxpool.Pool
	Replica *pgxpool.Pool
}

// ListProductsFromReplica sengaja query ke replica -- replikasi Postgres
// bersifat async, jadi ada kemungkinan kecil data yang dibaca sedikit lebih
// lama (replication lag) dibanding yang baru saja ditulis ke primary.
func (p *Pools) ListProductsFromReplica(ctx context.Context) ([]Product, error) {
	rows, err := p.Replica.Query(ctx, `SELECT id, name, price, stock FROM products`)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var products []Product
	for rows.Next() {
		var pr Product
		if err := rows.Scan(&pr.ID, &pr.Name, &pr.Price, &pr.Stock); err != nil {
			return nil, err
		}
		products = append(products, pr)
	}
	return products, rows.Err()
}

// CreateOrder tetap harus lewat Primary -- write gak boleh diarahkan ke replica.
func (p *Pools) CreateOrder(ctx context.Context, userID int64, items []OrderItem) (*Order, error) {
	return CreateOrder(ctx, p.Primary, userID, items)
}
```

### Contoh Kode — Node.js
```javascript
const { Pool } = require('pg');

// createPools memisahkan pool koneksi ke primary (writer) dan replica (reader).
function createPools(primaryUrl, replicaUrl) {
  return {
    primary: new Pool({ connectionString: primaryUrl }),
    replica: new Pool({ connectionString: replicaUrl }),
  };
}

// listProductsFromReplica sengaja query ke replica -- ada kemungkinan kecil
// replication lag dibanding data yang baru ditulis ke primary.
async function listProductsFromReplica(pools) {
  const { rows } = await pools.replica.query('SELECT id, name, price, stock FROM products');
  return rows;
}

// createOrderViaPrimary tetap harus lewat primary -- write gak boleh
// diarahkan ke replica.
async function createOrderViaPrimary(pools, userId, items) {
  return createOrder(pools.primary, userId, items);
}

module.exports = { createPools, listProductsFromReplica, createOrderViaPrimary };
```

### Trade-off & Pitfall
- Replication lag berarti replica bisa menampilkan data yang sedikit "basi" dibanding primary — jangan pakai replica untuk kasus yang butuh baca data yang baru saja ditulis oleh proses yang sama (misalnya langsung baca ulang order setelah `CreateOrder` sukses, di request yang sama).
- Menambah replica menambah kapasitas baca, tapi gak membantu kalau bottleneck-nya justru di write (`CreateOrder`, `UpdateOrderStatus`) — semua write tetap lewat satu primary yang sama, replica gak mengurangi beban itu sama sekali.
- Salah routing (misalnya query baca yang sebetulnya butuh data terbaru malah ke replica) adalah bug yang sering gak kelihatan di testing dengan traffic rendah, karena lag-nya biasanya cuma milidetik sampai beberapa detik — baru kelihatan jadi masalah nyata saat replica-nya sedang lag lebih lama dari biasanya.

### Kapan Dipakai
Ketika beban baca jauh lebih besar dari beban tulis dan sebagian besar query baca itu toleran terhadap data yang sedikit basi (katalog produk, laporan, dashboard analytics) — bukan untuk data yang harus selalu real-time konsisten dengan penulisannya.

### Sering Ditanya Saat Interview
- "Kapan sebuah query baca sebaiknya tetap ke primary, bukan ke replica?" — kalau query itu butuh melihat data yang baru saja ditulis di request/proses yang sama (read-after-write consistency), karena replica mungkin belum menerima perubahan itu akibat replication lag.
- "Apakah read replica membantu mengurangi beban write ke primary?" — tidak, replica cuma menambah kapasitas untuk query baca; semua operasi tulis tetap sepenuhnya dibebankan ke primary.
- "Apa risiko utama pakai read replica?" — replication lag: replica bisa menampilkan data yang belum ter-update dari perubahan terbaru di primary, sehingga user bisa melihat data yang sedikit basi kalau routing query-nya salah.

---

## 38. Database Replication

### Apa itu?
Database replication adalah proses menyalin perubahan data dari satu database (primary) ke satu atau lebih database lain (replica) secara terus-menerus, biasanya lewat streaming write-ahead log (WAL) di Postgres — replica terus "mengejar" perubahan yang terjadi di primary.

### Kenapa dibutuhkan?
Selain buat read replica (topik 37), replikasi juga jadi dasar disaster recovery: kalau primary OrderFlow tiba-tiba mati (hardware failure, corruption), salah satu replica bisa langsung dipromosikan jadi primary baru, jauh lebih cepat daripada restore dari backup yang mungkin ketinggalan beberapa jam.

### Cara Kerja
```
Primary                                    Replica
  |-- WAL record (perubahan data) -------->|
  |                                         |-- terima & apply WAL record
  |-- WAL record berikutnya --------------->|    ke datanya sendiri
  |                                         |-- data replica sekarang
  |                                         |   mencerminkan primary,
  |                                         |   tapi dengan jeda kecil
  |                                         |   (replication lag)

pg_last_xact_replay_timestamp() di replica -> kapan WAL terakhir di-apply
now() - pg_last_xact_replay_timestamp()     -> replication lag saat ini
```

### Contoh Kode — Go
```go
package db

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"
)

// ReplicationLagSeconds ngecek seberapa jauh replica ketinggalan dari
// primary, dengan query ke fungsi bawaan Postgres di sisi replica.
func ReplicationLagSeconds(ctx context.Context, replica *pgxpool.Pool) (float64, error) {
	var lagSeconds float64
	err := replica.QueryRow(ctx, `
		SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp()))
	`).Scan(&lagSeconds)
	if err != nil {
		return 0, fmt.Errorf("check replication lag: %w", err)
	}
	return lagSeconds, nil
}
```

### Contoh Kode — Node.js
```javascript
// replicationLagSeconds ngecek seberapa jauh replica ketinggalan dari primary.
async function replicationLagSeconds(replicaPool) {
  const result = await replicaPool.query(
    'SELECT EXTRACT(EPOCH FROM (now() - pg_last_xact_replay_timestamp())) AS lag'
  );
  return Number(result.rows[0].lag);
}

module.exports = { replicationLagSeconds };
```

### Trade-off & Pitfall
- Streaming replication Postgres secara default bersifat async — primary gak menunggu replica selesai apply WAL sebelum commit dianggap sukses, jadi ada window kecil di mana data yang sudah commit di primary belum ada di replica kalau primary tiba-tiba mati.
- Synchronous replication (primary menunggu minimal satu replica konfirmasi sebelum commit) menghilangkan risiko itu, tapi menambah latency tiap commit dan bikin availability write tergantung juga pada replica-nya hidup.
- Monitoring replication lag itu wajib, bukan opsional — lag yang terus membesar biasanya tanda replica kekurangan resource (CPU/disk I/O/network) buat mengejar primary, dan kalau dibiarkan bisa membuat replica makin gak berguna untuk read replica maupun failover.

### Kapan Dipakai
Wajib untuk setup production yang butuh high availability (failover cepat kalau primary mati) dan/atau read replica untuk scaling baca — dengan monitoring lag yang aktif supaya masalah replikasi ketahuan sebelum berdampak ke user.

### Sering Ditanya Saat Interview
- "Apa beda synchronous dan asynchronous replication?" — asynchronous: primary commit tanpa menunggu replica, ada risiko kecil data hilang kalau primary mati sebelum replica sempat menerima; synchronous: primary menunggu minimal satu replica konfirmasi dulu, lebih aman tapi menambah latency dan availability write bergantung ke replica.
- "Gimana caramu tau replica sedang lag jauh dari primary?" — bandingkan `now()` dengan `pg_last_xact_replay_timestamp()` di sisi replica; nilai itu menunjukkan seberapa lama sejak WAL terakhir yang diterima benar-benar di-apply.
- "Kenapa replikasi penting selain buat scaling baca?" — jadi dasar disaster recovery/high availability: kalau primary mati, salah satu replica bisa dipromosikan jadi primary baru jauh lebih cepat daripada restore dari backup.

---

## 39. Sharding

### Apa itu?
Sharding adalah teknik membagi data ke beberapa database terpisah (shard) berdasarkan suatu kunci partisi (misalnya `user_id`), di mana masing-masing shard cuma menyimpan sebagian dari keseluruhan data — beda dengan replication yang menyalin SEMUA data ke tiap node.

### Kenapa dibutuhkan?
Kalau OrderFlow tumbuh sampai tabel `orders` berisi miliaran baris, satu database — walau sudah punya index dan read replica — akan mencapai batas kapasitas fisik (storage, write throughput) di satu mesin. Sharding membagi beban itu ke banyak database terpisah, masing-masing cuma menangani sebagian user, sehingga total kapasitas sistem bisa terus tumbuh horizontal.

### Cara Kerja
```
shard_index = hash(user_id) % jumlah_shard

user_id=101 -> hash -> shard 2   -> semua order milik user 101 ada di shard 2
user_id=205 -> hash -> shard 0   -> semua order milik user 205 ada di shard 0

CreateOrder(userID, items):
  shard := pilih shard berdasarkan shard_index(userID)
  jalankan CreateOrder seperti biasa, tapi ke pool koneksi shard itu
```
Karena partisinya berdasarkan `user_id`, semua data satu user (order, order_items-nya) konsisten selalu ada di shard yang sama — query "order milik user ini" gak perlu nyari ke banyak shard sekaligus.

### Contoh Kode — Go
```go
package db

import (
	"context"
	"fmt"
	"hash/fnv"

	"github.com/jackc/pgx/v5/pgxpool"
)

// ShardedPools memegang pool koneksi ke tiap shard database OrderFlow. Data
// user, order, dan order_items dipartisi berdasarkan user_id, supaya satu
// database gak perlu menampung seluruh order semua user.
type ShardedPools struct {
	Shards []*pgxpool.Pool
}

// shardIndex menentukan shard mana yang menyimpan data milik userID, pakai
// hash yang stabil supaya user yang sama selalu jatuh ke shard yang sama.
func (s *ShardedPools) shardIndex(userID int64) int {
	h := fnv.New32a()
	fmt.Fprintf(h, "%d", userID)
	return int(h.Sum32()) % len(s.Shards)
}

// CreateOrder mengarahkan order milik userID ke shard yang tepat.
func (s *ShardedPools) CreateOrder(ctx context.Context, userID int64, items []OrderItem) (*Order, error) {
	shard := s.Shards[s.shardIndex(userID)]
	return CreateOrder(ctx, shard, userID, items)
}
```

### Contoh Kode — Node.js
```javascript
const crypto = require('crypto');

// shardIndex menentukan shard mana yang menyimpan data milik userId, pakai
// hash yang stabil supaya user yang sama selalu jatuh ke shard yang sama.
function shardIndex(userId, shardCount) {
  const hash = crypto.createHash('md5').update(String(userId)).digest('hex');
  const n = parseInt(hash.slice(0, 8), 16);
  return n % shardCount;
}

// createOrderSharded mengarahkan order milik userId ke shard yang tepat.
async function createOrderSharded(shardPools, userId, items) {
  const shard = shardPools[shardIndex(userId, shardPools.length)];
  return createOrder(shard, userId, items);
}

module.exports = { shardIndex, createOrderSharded };
```

### Trade-off & Pitfall
- Sharding menambah kompleksitas operasional besar-besaran: migrasi schema harus dijalankan ke semua shard, query analitik lintas user jadi harus menggabungkan hasil dari banyak shard (fan-out), dan `JOIN` antar data yang kebetulan ada di shard berbeda gak bisa dilakukan langsung di level database.
- Mengubah jumlah shard di kemudian hari (resharding) itu operasi berat — data harus dipindah-pindah sesuai hash baru, biasanya butuh downtime atau migrasi bertahap yang hati-hati.
- Kalau kunci partisi dipilih gak tepat (misalnya ada satu user dengan volume order jauh lebih besar dari user lain), satu shard bisa jadi "hot shard" yang jauh lebih terbebani dibanding shard lain — partitioning key yang baik harus mendistribusikan beban secara merata.

### Kapan Dipakai
Baru dipertimbangkan setelah opsi yang lebih murah (index, read replica, connection pooling yang lebih baik) sudah gak cukup lagi menangani skala data/traffic — sharding menyelesaikan masalah kapasitas skala besar, tapi dengan cost kompleksitas yang signifikan, jadi bukan langkah pertama.

### Sering Ditanya Saat Interview
- "Apa beda sharding dan replication?" — replication menyalin SEMUA data ke tiap node (untuk availability/scaling baca), sharding membagi data jadi partisi-partisi terpisah yang masing-masing cuma menyimpan sebagian data (untuk scaling kapasitas write/storage).
- "Kenapa `user_id` jadi pilihan wajar untuk shard key OrderFlow?" — sebagian besar query OrderFlow (order history, cart, dst) sudah natural difilter per user, jadi mempartisi berdasarkan `user_id` bikin sebagian besar query tetap cuma perlu menyentuh satu shard.
- "Apa risiko terbesar dari sharding dibanding scaling vertikal (upgrade mesin) atau read replica?" — kompleksitas operasional: query lintas shard jadi lebih sulit, resharding di kemudian hari mahal, dan ada risiko hot shard kalau distribusi data gak merata.

---

## 40. Database Security

### Apa itu?
Database security mencakup semua praktik yang melindungi data OrderFlow dari akses gak sah dan kebocoran: koneksi terenkripsi (SSL/TLS), role database dengan privilege terbatas (least privilege), kredensial yang gak hardcoded, dan query yang selalu di-parameterize untuk mencegah SQL injection.

### Kenapa dibutuhkan?
Database OrderFlow menyimpan data paling sensitif di seluruh sistem — `password_hash`, data pembayaran, riwayat order user. Satu celah kecil (misalnya query yang dibangun dari string concatenation langsung dari input user, atau kredensial database yang bocor ke repo publik) bisa berujung ke kebocoran seluruh data pelanggan, jauh lebih fatal dibanding celah di layer lain.

### Cara Kerja
```
BUG: query dibangun lewat string concatenation dari input user
  name := "kabel'; DROP TABLE products; --"
  query := "SELECT * FROM products WHERE name ILIKE '%" + name + "%'"
  -> attacker bisa menyisipkan SQL arbitrary lewat input `name`

FIX: parameterized query -- input user SELALU dikirim terpisah dari SQL,
     driver (pgx/pg) yang menangani escaping-nya, bukan string manipulation manual
  query := "SELECT * FROM products WHERE name ILIKE $1"
  params := ["%" + name + "%"]
  -> `name` diperlakukan sebagai DATA, bukan bagian dari perintah SQL
```

### Contoh Kode — Go
```go
package db

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"
)

// NewAppPool konek ke Postgres pakai role terbatas (bukan superuser), lewat
// SSL, dan credential dari env var/secret manager -- bukan hardcoded di kode.
func NewAppPool(ctx context.Context, host, dbName, appUser, appPassword string) (*pgxpool.Pool, error) {
	dsn := fmt.Sprintf(
		"postgres://%s:%s@%s/%s?sslmode=require",
		appUser, appPassword, host, dbName,
	)
	pool, err := pgxpool.New(ctx, dsn)
	if err != nil {
		return nil, fmt.Errorf("create pool: %w", err)
	}
	return pool, nil
}

// FindProductsByNameUnsafe BUG: bikin query dengan string concatenation
// langsung dari input user -- attacker bisa kirim name seperti
// `x'; DROP TABLE products; --` buat inject SQL arbitrary.
func FindProductsByNameUnsafe(ctx context.Context, db *pgxpool.Pool, name string) ([]Product, error) {
	query := fmt.Sprintf(`SELECT id, name, price, stock FROM products WHERE name ILIKE '%%%s%%'`, name)
	rows, err := db.Query(ctx, query)
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var products []Product
	for rows.Next() {
		var p Product
		if err := rows.Scan(&p.ID, &p.Name, &p.Price, &p.Stock); err != nil {
			return nil, err
		}
		products = append(products, p)
	}
	return products, rows.Err()
}

// FindProductsByName FIX: pakai parameterized query ($1), bukan string
// concatenation -- mencegah SQL injection walau `name` datang dari input user.
func FindProductsByName(ctx context.Context, db *pgxpool.Pool, name string) ([]Product, error) {
	rows, err := db.Query(ctx,
		`SELECT id, name, price, stock FROM products WHERE name ILIKE $1`,
		"%"+name+"%")
	if err != nil {
		return nil, err
	}
	defer rows.Close()

	var products []Product
	for rows.Next() {
		var p Product
		if err := rows.Scan(&p.ID, &p.Name, &p.Price, &p.Stock); err != nil {
			return nil, err
		}
		products = append(products, p)
	}
	return products, rows.Err()
}
```

### Contoh Kode — Node.js
```javascript
const { Pool } = require('pg');

// createAppPool konek ke Postgres pakai role terbatas (bukan superuser),
// lewat SSL, dan credential dari env var/secret manager -- bukan hardcoded.
function createAppPool() {
  return new Pool({
    host: process.env.DB_HOST,
    database: process.env.DB_NAME,
    user: process.env.DB_APP_USER,
    password: process.env.DB_APP_PASSWORD,
    ssl: { rejectUnauthorized: true },
  });
}

// findProductsByNameUnsafe BUG: bikin query dengan string concatenation
// langsung dari input user -- attacker bisa inject SQL arbitrary.
async function findProductsByNameUnsafe(pool, name) {
  const query = `SELECT id, name, price, stock FROM products WHERE name ILIKE '%${name}%'`;
  const result = await pool.query(query);
  return result.rows;
}

// findProductsByName FIX: pakai parameterized query ($1), bukan string
// concatenation -- mencegah SQL injection walau `name` datang dari input user.
async function findProductsByName(pool, name) {
  const result = await pool.query(
    'SELECT id, name, price, stock FROM products WHERE name ILIKE $1',
    [`%${name}%`]
  );
  return result.rows;
}

module.exports = { createAppPool, findProductsByNameUnsafe, findProductsByName };
```

### Trade-off & Pitfall
- Role database aplikasi (`appUser`) sebaiknya cuma punya privilege yang benar-benar dibutuhkan (`SELECT`/`INSERT`/`UPDATE`/`DELETE` di tabel tertentu), bukan superuser — kalau credential-nya bocor, blast radius-nya jadi terbatas, bukan akses penuh ke seluruh database termasuk kemampuan `DROP TABLE` atau ubah role lain.
- `sslmode=require` mencegah data lewat jaringan dalam bentuk plaintext, tapi gak memverifikasi identitas server (rawan man-in-the-middle) — untuk keamanan lebih tinggi, `sslmode=verify-full` yang benar-benar memvalidasi certificate server.
- Parameterized query menyelesaikan SQL injection dari value, tapi gak melindungi kalau ada bagian query yang dibangun dinamis dari nama kolom/tabel berdasarkan input user (identifier gak bisa di-parameterize seperti value) — bagian itu tetap butuh whitelist manual.

### Kapan Dipakai
Selalu, tanpa pengecualian — parameterized query dan role dengan least privilege bukan optimasi opsional, itu baseline minimum untuk aplikasi yang menyentuh data production, apalagi data sesensitif yang disimpan OrderFlow.

### Sering Ditanya Saat Interview
- "Kenapa `FindProductsByNameUnsafe` rawan SQL injection sementara `FindProductsByName` aman?" — versi unsafe membangun string SQL langsung dari input user lewat concatenation, jadi attacker bisa menyisipkan syntax SQL tambahan; versi aman mengirim input sebagai parameter terpisah ($1), yang selalu diperlakukan driver sebagai data, bukan bagian dari perintah SQL.
- "Kenapa aplikasi sebaiknya gak konek ke database pakai role superuser?" — supaya blast radius kebocoran credential terbatas: role dengan privilege minimal cuma bisa melakukan operasi yang memang dibutuhkan aplikasi, gak bisa mengubah struktur tabel atau role lain kalau disalahgunakan.
- "Apa yang gak dilindungi oleh parameterized query?" — bagian query yang berupa identifier dinamis (nama kolom/tabel) yang dibangun dari input user, karena parameterization cuma berlaku untuk value, bukan identifier; kasus itu perlu validasi/whitelist manual di application code.

---

**Selanjutnya:** [Phase 04 — Caching](./phase-04-caching.md)
