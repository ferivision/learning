# Phase 13 — Performance

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

## 94. API Performance

### Apa itu?
API Performance tuning adalah proses "measure first, optimize later" -- ukur dulu lewat logs/metrics/tracing (Phase 10) atau profiler (Go: `pprof`; Node.js: `perf_hooks`/`--prof`) buat tahu persis bagian mana dari sebuah endpoint yang makan waktu paling banyak, baru optimasi bagian itu. Bottleneck-nya bisa macam-macam: query database, network roundtrip ke service lain, external API, CPU (misalnya hashing password, topik 1), memory, atau lock contention (`FOR UPDATE`, Phase 3 topik 34-35).

### Kenapa dibutuhkan?
Endpoint `POST /orders` di OrderFlow memanggil `CreateOrder`/`createOrder` (topik 29) yang melakukan beberapa query sekaligus di dalam satu transaction -- kalau endpoint ini lambat, ada banyak kandidat penyebab: query lambat, lock `FOR UPDATE` yang antre, koneksi database yang exhausted, atau bahkan cuma network latency biasa. Tanpa mengukur dulu, optimasi biasanya berdasarkan tebakan -- developer bisa habiskan waktu berhari-hari micro-optimize kode yang ternyata bukan bottleneck-nya, sementara masalah aslinya (misalnya index yang hilang, topik 95) gak pernah kesentuh.

### Cara Kerja
```
Measure (logs/metrics/tracing/profiler)
        |
        v
Identify bottleneck (fase mana yang paling lama: decode? query? encode?)
        |
        v
Optimize bagian itu SAJA (index, cache, query rewrite, dst)
        |
        v
Re-measure -- pastikan angkanya benar-benar membaik, ulangi loop kalau perlu
```

### Contoh Kode — Go
Handler `POST /orders` yang membungkus `CreateOrder` (topik 29) dengan instrumentasi timing per fase -- decode body, panggil `CreateOrder`, encode response -- supaya kalau endpoint ini lambat, langsung kelihatan fase mana yang jadi bottleneck tanpa perlu menebak:
```go
package api

import (
	"encoding/json"
	"log"
	"net/http"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"

	"orderflow/db"
)

type createOrderRequest struct {
	UserID int64          `json:"user_id"`
	Items  []db.OrderItem `json:"items"`
}

// CreateOrderHandler membungkus db.CreateOrder (topik 29) dengan instrumentasi
// timing per fase, supaya kalau endpoint ini lambat, kita tahu persis fase
// mana (decode, query, encode) yang jadi bottleneck.
func CreateOrderHandler(pool *pgxpool.Pool) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()

		var req createOrderRequest
		if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
			http.Error(w, "invalid request body", http.StatusBadRequest)
			return
		}
		decodeDone := time.Now()

		order, err := db.CreateOrder(r.Context(), pool, req.UserID, req.Items)
		createOrderDone := time.Now()
		if err != nil {
			http.Error(w, err.Error(), http.StatusUnprocessableEntity)
			return
		}

		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(order)
		encodeDone := time.Now()

		log.Printf(
			"POST /orders timing: decode=%v createOrder=%v encode=%v total=%v",
			decodeDone.Sub(start),
			createOrderDone.Sub(decodeDone),
			encodeDone.Sub(createOrderDone),
			encodeDone.Sub(start),
		)
	}
}
```
Untuk melihat lebih dalam dari sekadar timing per fase (misalnya CPU profile atau heap snapshot), `net/http/pprof` didaftarkan di server terpisah -- port beda dari traffic publik, supaya endpoint debug ini gak numpang di port yang sama dengan traffic user asli:
```go
// main.go -- server pprof terpisah, HANYA dibind ke localhost, gak pernah
// diekspos ke internet.
package main

import (
	"log"
	"net/http"
	_ "net/http/pprof"
)

func startPprofServer() {
	// http://localhost:6060/debug/pprof/profile?seconds=30 -- CPU profile 30 detik
	// http://localhost:6060/debug/pprof/heap                -- snapshot heap saat ini
	log.Println(http.ListenAndServe("localhost:6060", nil))
}
```

### Contoh Kode — Node.js
Handler yang setara di Express, pakai modul bawaan `perf_hooks` (bukan library eksternal) buat mengukur waktu tiap fase:
```javascript
const { performance } = require('perf_hooks');
const { createOrder } = require('./db');

// createOrderHandler membungkus createOrder (topik 29) dengan instrumentasi
// timing per fase, setara dengan versi Go -- decode body ditangani middleware
// express.json() jadi yang diukur di sini: waktu tunggu createOrder dan waktu
// serialize response.
function createOrderHandler(pool) {
  return async function (req, res) {
    const start = performance.now();

    const { userId, items } = req.body;
    let order;
    try {
      order = await createOrder(pool, userId, items);
    } catch (err) {
      res.status(422).json({ error: err.message });
      return;
    }
    const createOrderDone = performance.now();

    res.json(order);
    const encodeDone = performance.now();

    console.log(
      `POST /orders timing: createOrder=${(createOrderDone - start).toFixed(2)}ms ` +
        `encode=${(encodeDone - createOrderDone).toFixed(2)}ms ` +
        `total=${(encodeDone - start).toFixed(2)}ms`
    );
  };
}

module.exports = { createOrderHandler };
```

### Trade-off & Pitfall
- Instrumentasi timing/logging di atas ada overhead-nya (walau kecil) -- kalau dibiarkan mencatat log detail di SETIAP request pada traffic tinggi tanpa sampling/rate-limit, log itu sendiri bisa jadi beban baru (disk I/O, biaya storage log).
- `pprof` (`/debug/pprof`) HARUS di server/port terpisah yang cuma bisa diakses secara internal -- kalau keekspos ke publik (Phase 2), siapapun bisa memicu CPU profile 30 detik berkali-kali dan bikin server kelebihan beban, atau membaca heap snapshot yang berpotensi bocorin data sensitif di memory.
- Timing wall-clock kasar seperti di atas cuma menunjukkan "fase mana yang lama", bukan "kenapa" -- untuk tahu detail (misalnya request menunggu row lock `FOR UPDATE` di `CreateOrder`, Phase 3 topik 34), tetap butuh profiler (`pprof`) atau distributed tracing (Phase 10).
- Mengukur di laptop lokal seringkali gak representatif -- latency ke Postgres/Redis di lokal (localhost, ~0ms network) beda jauh dengan production (network hop antar service/availability zone), jadi angka absolut dari local benchmark gak boleh dipakai sebagai patokan SLA produksi.

### Kapan Dipakai
Begitu metrik (Phase 10) menunjukkan P95/P99 latency `POST /orders` mulai naik, atau sebelum memutuskan rencana optimasi apapun -- "measure first" selalu mendahului "optimize", supaya effort gak salah sasaran ke bagian yang sebenarnya bukan bottleneck.

### Sering Ditanya Saat Interview
- "Kenapa harus 'measure first' sebelum optimasi, bukan langsung optimasi bagian yang 'kelihatannya' lambat?" -- intuisi soal bagian mana yang lambat seringkali salah; tanpa angka nyata, waktu development bisa habis micro-optimize kode yang ternyata cuma menyumbang sebagian kecil dari total latency, sementara bottleneck asli (misalnya query tanpa index, topik 95) gak pernah kesentuh.
- "Kenapa server `pprof` harus dipisah dari server API utama?" -- supaya endpoint debug yang berat (CPU profiling, heap dump) gak numpang di port yang sama dengan traffic user asli, dan supaya gampang dibatasi cuma bisa diakses secara internal (localhost/VPN), gak keekspos ke publik.
- "Apa beda profiling dengan monitoring/tracing (Phase 10)?" -- profiling ngezoom ke SATU instance yang lagi jalan buat lihat detail di mana waktu CPU/memory terpakai (biasanya dipicu manual saat debugging); monitoring/tracing berjalan terus-menerus di production, mengagregasi metrik dan span per-request lintas banyak instance/service buat mendeteksi tren/anomali dari waktu ke waktu.

---

## 95. Database Performance

### Apa itu?
`EXPLAIN ANALYZE` adalah perintah Postgres yang benar-benar MENJALANKAN sebuah query dan menunjukkan query plan yang sesungguhnya dipakai beserta angka eksekusi aktual (bukan cuma estimasi) -- jumlah baris yang benar-benar diperiksa, waktu tiap tahap, dan apakah planner memilih `Seq Scan` (baca semua baris) atau `Index Scan` (loncat langsung ke baris yang cocok). Dipakai berdampingan dengan `CREATE INDEX` buat membuktikan, dengan data, apakah index yang ditambahkan benar-benar dipakai dan benar-benar mempercepat query.

### Kenapa dibutuhkan?
Fitur riwayat pesanan OrderFlow menjalankan query yang memfilter `orders` berdasarkan `user_id` dan `status`, diurutkan `created_at` terbaru dulu. Selama tabel `orders` masih kecil, query itu cepat walau tanpa index sama sekali -- tapi begitu tabelnya bertumbuh sampai jutaan baris (kekhawatiran yang sama seperti di Phase 3), Postgres terpaksa memindai SEMUA baris satu per satu (`Seq Scan`) cuma buat nemuin puluhan baris punya satu user, dan itu bisa berubah dari hitungan milidetik jadi hitungan detik.

### Cara Kerja
```
Query lambat (tanpa index yang cocok):
  SELECT id, status, total, created_at FROM orders
  WHERE user_id = $1 AND status = $2
  ORDER BY created_at DESC

  -> Planner pilih Seq Scan (baca SEMUA baris orders, filter manual satu-satu)
  -> EXPLAIN ANALYZE:
       Seq Scan on orders (actual time=0.048..842.310 rows=42 loops=1)
         Filter: (user_id = $1) AND (status = $2)
         Rows Removed by Filter: 4999958
       Sort (actual time=842.315..842.402 rows=42 loops=1)   <- sort manual, index blm ada

Setelah tambah composite index:
  CREATE INDEX idx_orders_user_status_created
      ON orders (user_id, status, created_at DESC);

  -> Planner pilih Index Scan (loncat langsung ke baris yang match, sudah
     terurut created_at DESC -- gak perlu Sort tambahan)
  -> EXPLAIN ANALYZE:
       Index Scan using idx_orders_user_status_created on orders
         (actual time=0.021..0.312 rows=42 loops=1)
         Index Cond: (user_id = $1) AND (status = $2)
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

// OrderSummary subset kolom Order (topik 22) yang dipakai khusus buat
// halaman riwayat pesanan.
type OrderSummary struct {
	ID        int64
	Status    string
	Total     float64
	CreatedAt time.Time
}

// GetOrdersByUserSlow list order milik satu user, difilter status, diurutkan
// created_at terbaru dulu -- versi SEBELUM ada composite index di
// (user_id, status, created_at), rawan jadi Seq Scan begitu tabel orders
// membesar.
func GetOrdersByUserSlow(ctx context.Context, db *pgxpool.Pool, userID int64, status string) ([]OrderSummary, error) {
	rows, err := db.Query(ctx,
		`SELECT id, status, total, created_at FROM orders
		 WHERE user_id = $1 AND status = $2
		 ORDER BY created_at DESC`,
		userID, status,
	)
	if err != nil {
		return nil, fmt.Errorf("query orders by user %d: %w", userID, err)
	}
	defer rows.Close()

	var out []OrderSummary
	for rows.Next() {
		var o OrderSummary
		if err := rows.Scan(&o.ID, &o.Status, &o.Total, &o.CreatedAt); err != nil {
			return nil, fmt.Errorf("scan order summary: %w", err)
		}
		out = append(out, o)
	}
	return out, rows.Err()
}

// ExplainOrdersByUser menjalankan EXPLAIN ANALYZE atas query yang sama supaya
// query plan & waktu eksekusi ASLI (bukan estimasi) kelihatan -- dipakai
// sebelum/sesudah CREATE INDEX buat membuktikan index-nya benar-benar
// dipilih planner.
func ExplainOrdersByUser(ctx context.Context, db *pgxpool.Pool, userID int64, status string) (string, error) {
	rows, err := db.Query(ctx,
		`EXPLAIN ANALYZE SELECT id, status, total, created_at FROM orders
		 WHERE user_id = $1 AND status = $2
		 ORDER BY created_at DESC`,
		userID, status,
	)
	if err != nil {
		return "", fmt.Errorf("explain analyze: %w", err)
	}
	defer rows.Close()

	var plan string
	for rows.Next() {
		var line string
		if err := rows.Scan(&line); err != nil {
			return "", fmt.Errorf("scan explain line: %w", err)
		}
		plan += line + "\n"
	}
	return plan, rows.Err()
}
```
Migration yang menambahkan composite index -- urutan kolom PENTING: `user_id` dan `status` (equality filter) duluan, `created_at DESC` terakhir, supaya index bisa dipakai sekaligus buat filter DAN buat `ORDER BY` tanpa sort tambahan:
```sql
-- migration: index buat query riwayat pesanan di atas.
CREATE INDEX idx_orders_user_status_created
    ON orders (user_id, status, created_at DESC);
```

### Contoh Kode — Node.js
```javascript
const { Pool } = require('pg');

// getOrdersByUserSlow setara dengan versi Go -- list order milik satu user,
// difilter status, diurutkan created_at terbaru dulu.
async function getOrdersByUserSlow(pool, userId, status) {
  const { rows } = await pool.query(
    `SELECT id, status, total, created_at FROM orders
     WHERE user_id = $1 AND status = $2
     ORDER BY created_at DESC`,
    [userId, status]
  );
  return rows;
}

// explainOrdersByUser menjalankan EXPLAIN ANALYZE atas query yang sama --
// tiap baris output plan dikembalikan sebagai array of string.
async function explainOrdersByUser(pool, userId, status) {
  const { rows } = await pool.query(
    `EXPLAIN ANALYZE SELECT id, status, total, created_at FROM orders
     WHERE user_id = $1 AND status = $2
     ORDER BY created_at DESC`,
    [userId, status]
  );
  return rows.map((row) => row['QUERY PLAN']);
}

module.exports = { getOrdersByUserSlow, explainOrdersByUser };
```

### Trade-off & Pitfall
- `EXPLAIN ANALYZE` benar-benar MENGEKSEKUSI query, beda dengan `EXPLAIN` biasa yang cuma nunjukin estimasi planner tanpa menjalankannya -- jalanin `EXPLAIN ANALYZE` di atas `UPDATE`/`DELETE`/`INSERT` beneran melakukan perubahan data itu; kalau cuma mau lihat plan-nya tanpa efek samping, bungkus dalam transaction lalu `ROLLBACK`.
- Urutan kolom di composite index mengikuti leftmost-prefix rule -- index `(user_id, status, created_at)` di atas gak akan kepake optimal buat query yang cuma memfilter `status` saja tanpa `user_id`, karena `status` bukan kolom pertama di index itu.
- Tiap index tambahan memperlambat `INSERT`/`UPDATE`/`DELETE` di tabel itu (index harus ikut di-update tiap kali) dan makan storage tambahan -- index cuma worth it buat query yang sudah terbukti lambat DAN sering dijalankan, bukan ditambahkan "just in case" ke semua kolom yang mungkin dipakai `WHERE`.
- Menambah index gak menyelesaikan masalah kalau bottleneck-nya sebenarnya di tempat lain -- connection pool yang terlalu kecil (banyak request antre nunggu koneksi bebas) atau beban baca yang memang sudah terlalu tinggi buat satu instance Postgres (butuh read replica) gak akan hilang cuma dengan menambah index.

### Kapan Dipakai
Setelah sebuah query terbukti lambat lewat pengukuran nyata (metrics/log lambat, topik 94) -- bukan menambah index preemptif ke semua kolom yang "kelihatannya" akan dipakai `WHERE`. Kalau index saja gak cukup (query sudah optimal tapi beban baca tetap sangat tinggi), baru pertimbangkan connection pool sizing atau read replica buat mendistribusikan beban baca.

### Sering Ditanya Saat Interview
- "Bedanya `EXPLAIN` dengan `EXPLAIN ANALYZE`?" -- `EXPLAIN` cuma nunjukin rencana & estimasi planner tanpa benar-benar menjalankan query; `EXPLAIN ANALYZE` benar-benar mengeksekusi query itu dan melaporkan angka aktual (actual time, rows aktual yang diperiksa).
- "Kenapa urutan kolom di composite index penting?" -- leftmost-prefix rule: index dipakai efisien kalau query menyaring kolom-kolomnya berurutan dari kiri; kalau query cuma menyaring kolom kedua/ketiga tanpa kolom pertama, index itu gak kepake optimal atau malah gak dipakai sama sekali.
- "Kenapa gak index semua kolom yang mungkin dipakai di WHERE saja biar aman?" -- tiap index nambah overhead nulis (setiap `INSERT`/`UPDATE`/`DELETE` juga harus meng-update index itu) dan makan storage tambahan; index yang jarang/gak pernah dipakai planner cuma jadi beban tanpa manfaat.
- "Kapan pindah dari 'tambah index' ke 'tambah read replica'?" -- kalau query individual sudah optimal (index sudah pas, `EXPLAIN ANALYZE` sudah bagus) tapi total volume baca tetap terlalu tinggi buat satu instance database, read replica mendistribusikan beban baca ke instance tambahan; connection pool sizing relevan kalau bottleneck-nya justru di jumlah koneksi yang tersedia, bukan di query itu sendiri.

---

## 96. Caching (Performance Angle)

### Apa itu?
Ini adalah sisi kuantitatif dari caching (Phase 4): mengukur, dengan angka nyata, seberapa besar selisih latency antara akses langsung ke Postgres lewat `GetProductByID`/`getProductById` (topik 25) dibanding lewat cache-aside `GetProductCached`/`getProductCached` (topik 42). Cocok dipakai kalau data sering dibaca DAN gak terlalu sering berubah -- seperti detail product yang jadi contoh di sini.

### Kenapa dibutuhkan?
Waktu cache-aside pertama kali diperkenalkan di Phase 4, asumsinya "cache lebih cepat dari database" diterima sebagai teori. Tapi menambahkan Redis berarti menambah satu komponen infrastruktur baru yang bisa gagal (Phase 4 topik 46) dan kompleksitas invalidation (topik 43) -- kompleksitas itu cuma sepadan kalau memang terbukti dengan angka bahwa cache HIT jauh lebih cepat dari roundtrip ke Postgres. Tanpa mengukur, klaim "sudah lebih cepat karena ada cache" cuma asumsi, bukan fakta.

### Cara Kerja
```
N request berturut-turut ke product yang SAMA:

Path A -- GetProductByID/getProductById (SELALU ke Postgres):
  request 1..N -> Postgres setiap kali -> latency relatif konsisten
                                            (network + query time)

Path B -- GetProductCached/getProductCached (cache-aside, topik 42):
  request 1   -> cache MISS -> Postgres + tulis ke Redis  (paling lambat)
  request 2..N -> cache HIT -> cuma roundtrip Redis        (jauh lebih cepat)

Kumpulkan durasi tiap request di kedua path -> urutkan ascending -> hitung
P50 (median) dan P95 (95% request lebih cepat dari angka ini) -> bandingkan.
```

### Contoh Kode — Go
Benchmark memakai `testing.B` bawaan Go, dijalankan lewat `go test -bench=BenchmarkGetProduct -benchtime=1000x`:
```go
package db_test

import (
	"context"
	"testing"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/redis/go-redis/v9"

	"orderflow/db"
)

var (
	benchPool *pgxpool.Pool
	benchRDB  *redis.Client
)

// BenchmarkGetProductByID mengukur latency akses langsung ke Postgres lewat
// GetProductByID (topik 25) -- tiap iterasi selalu roundtrip ke database,
// gak ada cache sama sekali.
func BenchmarkGetProductByID(b *testing.B) {
	ctx := context.Background()
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		if _, err := db.GetProductByID(ctx, benchPool, 1); err != nil {
			b.Fatal(err)
		}
	}
}

// BenchmarkGetProductCached mengukur latency lewat GetProductCached (topik
// 42) -- iterasi pertama MISS (roundtrip Postgres + isi cache), sisanya HIT
// (cuma roundtrip Redis), jadi rata-rata b.N besar mendekati latency HIT.
func BenchmarkGetProductCached(b *testing.B) {
	ctx := context.Background()
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		if _, err := db.GetProductCached(ctx, benchRDB, benchPool, 1); err != nil {
			b.Fatal(err)
		}
	}
}
```
Selain `go test -bench`, pengukuran manual berikut menghitung P50/P95 eksplisit, supaya bisa ditampilkan berdampingan dalam satu laporan:
```go
package db_test

import (
	"sort"
	"time"
)

// measureLatencies menjalankan fn sebanyak n kali secara berurutan dan
// mengembalikan durasi tiap panggilan, terurut ascending.
func measureLatencies(n int, fn func() error) ([]time.Duration, error) {
	durations := make([]time.Duration, 0, n)
	for i := 0; i < n; i++ {
		start := time.Now()
		if err := fn(); err != nil {
			return nil, err
		}
		durations = append(durations, time.Since(start))
	}
	sort.Slice(durations, func(i, j int) bool { return durations[i] < durations[j] })
	return durations, nil
}

// percentile mengambil nilai P-persentil dari slice durasi yang SUDAH terurut
// ascending (dihasilkan oleh measureLatencies di atas).
func percentile(sorted []time.Duration, p float64) time.Duration {
	if len(sorted) == 0 {
		return 0
	}
	idx := int(float64(len(sorted)-1) * p)
	return sorted[idx]
}
```

### Contoh Kode — Node.js
```javascript
const { performance } = require('perf_hooks');
const { getProductById } = require('./db');
const { getProductCached } = require('./cache');

// measureLatencies menjalankan fn sebanyak n kali secara berurutan dan
// mengembalikan array durasi (ms) terurut ascending -- setara dengan
// pendekatan Go di atas.
async function measureLatencies(n, fn) {
  const durations = [];
  for (let i = 0; i < n; i++) {
    const start = performance.now();
    await fn();
    durations.push(performance.now() - start);
  }
  durations.sort((a, b) => a - b);
  return durations;
}

// percentile mengambil nilai P-persentil dari array durasi yang SUDAH
// terurut ascending (dihasilkan oleh measureLatencies di atas).
function percentile(sorted, p) {
  const idx = Math.floor((sorted.length - 1) * p);
  return sorted[idx];
}

// compareProductLatency membandingkan latency getProductById (selalu ke
// Postgres) vs getProductCached (cache-aside lewat Redis, topik 42) dengan
// n panggilan berurutan terhadap product yang sama.
async function compareProductLatency(pool, redisClient, productId, n = 1000) {
  const dbLatencies = await measureLatencies(n, () => getProductById(pool, productId));
  const cacheLatencies = await measureLatencies(n, () => getProductCached(redisClient, pool, productId));

  return {
    db: { p50: percentile(dbLatencies, 0.5), p95: percentile(dbLatencies, 0.95) },
    cache: { p50: percentile(cacheLatencies, 0.5), p95: percentile(cacheLatencies, 0.95) },
  };
}

module.exports = { measureLatencies, percentile, compareProductLatency };
```

### Trade-off & Pitfall
- Benchmark di atas mengetes SATU product_id yang sama berulang-ulang -- itu skenario cache paling optimistik (selalu HIT setelah iterasi pertama). Traffic produksi asli tersebar ke ribuan product_id berbeda, jadi hit-rate riilnya bakal lebih rendah daripada angka benchmark ideal ini.
- `go test -bench` menjalankan tiap `Benchmark*` berurutan dalam satu proses -- kalau `BenchmarkGetProductCached` kebetulan jalan setelah cache sudah "dipanaskan" (warm) oleh test lain, hasilnya bisa bias dibanding kondisi cache benar-benar kosong seperti di awal deployment produksi.
- `measureLatencies` berjalan sekuensial (satu panggilan tunggu selesai sebelum lanjut ke berikutnya) -- ini gak mensimulasikan banyak request CONCURRENT beneran; buat itu, load testing dengan banyak virtual user paralel (topik 97) lebih representatif.
- Untuk kasus MISS, cache-aside justru sedikit lebih lambat dari langsung ke Postgres tanpa cache (dua roundtrip: cek Redis dulu, baru Postgres) -- worth it cuma kalau hit-rate cukup tinggi buat menutup overhead itu di rata-rata keseluruhan.

### Kapan Dipakai
Sebelum dan sesudah menambahkan cache, buat memvalidasi klaim "cache bikin lebih cepat" dengan angka nyata -- bukan cuma asumsi teori. Diulang secara berkala juga penting karena pola akses data (dan karena itu hit-rate) bisa berubah seiring waktu.

### Sering Ditanya Saat Interview
- "Kenapa iterasi pertama `BenchmarkGetProductCached` lebih lambat dari sisanya?" -- MISS pertama harus roundtrip ke Postgres DAN menulis hasilnya ke Redis dulu; sisanya HIT dan cuma roundtrip Redis -- mirip cold-start.
- "Kenapa pakai P95 bukan cuma rata-rata (average) buat membandingkan performa?" -- rata-rata bisa menyembunyikan outlier; P95 menunjukkan pengalaman "hampir terburuk" yang dirasakan sebagian user, jauh lebih relevan buat SLA dibanding rata-rata yang bisa terlihat bagus padahal ada ekor lambat.
- "Apa risiko benchmark ini kalau cuma dites terhadap satu product_id yang sama berulang-ulang?" -- hasilnya optimistik banget karena key itu pasti selalu HIT setelah iterasi pertama; produksi asli punya ribuan product_id dengan distribusi akses gak merata, yang bikin hit-rate riil lebih rendah dari benchmark ideal ini.

---

## 97. Load Testing

### Apa itu?
Load testing mensimulasikan banyak Virtual User (VU) memukul endpoint sungguhan secara paralel dan terus-menerus selama periode tertentu, buat mengukur metrik seperti requests/sec (throughput), latency (P50/P95/P99), error rate, dan titik saturasi -- sebelum traffic produksi asli yang "mengetesnya" duluan. **k6** adalah salah satu tool populer buat ini: skrip test-nya ditulis dalam JavaScript, dijalankan oleh binary `k6`, bukan oleh Node.js.

### Kenapa dibutuhkan?
`CreateOrder`/`createOrder` (topik 29) mengunci row `products` lewat `FOR UPDATE` (Phase 3 topik 34-35) -- request yang bersaing memperebutkan product yang sama jadi ter-serialize (antre satu-satu). Di traffic normal ini gak kerasa, tapi begitu ada lonjakan traffic nyata (misalnya flash sale di OrderFlow), lock contention itu bisa bikin latency `POST /orders` melonjak drastis. Tanpa load testing, potensi masalah ini baru ketahuan saat traffic produksi asli yang mengalaminya -- sudah terlambat, karena user sungguhan yang kena dampaknya.

### Cara Kerja
```
k6 script:
  export const options = { stages: [...], thresholds: {...} }
  export default function () { ... }   -- 1 iterasi = 1 perilaku virtual user (VU)

k6 runtime:
  VU 1..N dijalankan paralel sesuai stage (ramp-up -> steady -> ramp-down)
  -> tiap request dicatat: duration, status, error
  -> agregat di akhir:
       http_req_duration (P50/P95/P99)  -> latency
       http_reqs                        -> throughput (RPS)
       http_req_failed                  -> error rate
       vus_max                          -> titik saturasi yang dites

  thresholds (options.thresholds) -- kalau P95 lewat batas atau error rate
  kelewat tinggi, k6 exit code != 0 -> bisa dipasang sebagai GATE otomatis di
  pipeline CI/CD (Phase 12) sebelum image baru benar-benar dideploy.
```

### Contoh Kode — Go
Load generator ringan pakai goroutine + `sync.WaitGroup`, dipakai buat smoke-test cepat tanpa perlu install k6 -- bukan pengganti k6 (topik ini masih butuh k6 buat laporan performa yang serius, lihat Trade-off), cuma pelengkap:
```go
package main

import (
	"bytes"
	"encoding/json"
	"fmt"
	"net/http"
	"os"
	"sort"
	"sync"
	"time"
)

// loadTest memukul satu endpoint sebanyak `requests` kali dengan `concurrency`
// worker paralel, lalu mengembalikan durasi tiap request yang berhasil,
// terurut ascending.
func loadTest(client *http.Client, method, url string, body []byte, requests, concurrency int) []time.Duration {
	var (
		mu        sync.Mutex
		durations = make([]time.Duration, 0, requests)
		wg        sync.WaitGroup
		sem       = make(chan struct{}, concurrency)
	)

	for i := 0; i < requests; i++ {
		wg.Add(1)
		sem <- struct{}{}
		go func() {
			defer wg.Done()
			defer func() { <-sem }()

			start := time.Now()
			req, err := http.NewRequest(method, url, bytes.NewReader(body))
			if err != nil {
				return
			}
			req.Header.Set("Content-Type", "application/json")
			resp, err := client.Do(req)
			if err != nil {
				return
			}
			resp.Body.Close()

			mu.Lock()
			durations = append(durations, time.Since(start))
			mu.Unlock()
		}()
	}
	wg.Wait()

	sort.Slice(durations, func(i, j int) bool { return durations[i] < durations[j] })
	return durations
}

// percentile mengambil nilai P-persentil dari slice durasi yang SUDAH
// terurut ascending (dihasilkan oleh loadTest di atas).
func percentile(sorted []time.Duration, p float64) time.Duration {
	if len(sorted) == 0 {
		return 0
	}
	idx := int(float64(len(sorted)-1) * p)
	return sorted[idx]
}

func main() {
	client := &http.Client{Timeout: 10 * time.Second}
	base := "http://localhost:8080"

	getDurations := loadTest(client, http.MethodGet, base+"/products/1", nil, 500, 50)
	fmt.Fprintf(os.Stdout, "GET /products/:id  p50=%v p95=%v p99=%v\n",
		percentile(getDurations, 0.5), percentile(getDurations, 0.95), percentile(getDurations, 0.99))

	orderBody, _ := json.Marshal(map[string]any{
		"user_id": 1,
		"items":   []map[string]any{{"product_id": 1, "qty": 1, "price": 10000}},
	})
	postDurations := loadTest(client, http.MethodPost, base+"/orders", orderBody, 500, 50)
	fmt.Fprintf(os.Stdout, "POST /orders       p50=%v p95=%v p99=%v\n",
		percentile(postDurations, 0.5), percentile(postDurations, 0.95), percentile(postDurations, 0.99))
}
```

### Contoh Kode — Node.js
Skrip k6 asli (dijalankan lewat `k6 run loadtest.js`, BUKAN lewat `node`) yang meng-load-test `GET /products/:id` dan `POST /orders` sekaligus -- 3 stage (ramp-up 30 detik ke 50 VU, steady 1 menit, ramp-down 10 detik), dengan threshold yang men-gate P95 latency dan error rate:
```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

// options -- ramp-up ke 50 VU dalam 30s, steady 50 VU selama 1 menit,
// ramp-down ke 0 dalam 10s. thresholds nge-gate: k6 exit code != 0 kalau P95
// latency > 300ms atau error rate > 1% -- bisa dipasang sebagai gate di
// pipeline CI/CD (Phase 12) sebelum image baru di-deploy.
export const options = {
  stages: [
    { duration: '30s', target: 50 },
    { duration: '1m', target: 50 },
    { duration: '10s', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<300'],
    http_req_failed: ['rate<0.01'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';

// default function -- 1 iterasi = perilaku 1 virtual user (VU): baca detail
// product (GET /products/:id, topik 42/96), lalu bikin order (POST /orders,
// yang manggil CreateOrder/createOrder, topik 29/94) buat product itu.
// product_id diacak (1-20) supaya beban FOR UPDATE (Phase 3 topik 34) gak
// numpuk di satu product saja -- lebih representatif dari traffic asli.
export default function () {
  const productId = Math.floor(Math.random() * 20) + 1;

  const getRes = http.get(`${BASE_URL}/products/${productId}`);
  check(getRes, {
    'GET /products/:id status 200': (r) => r.status === 200,
  });

  const payload = JSON.stringify({
    user_id: 1,
    items: [{ product_id: productId, qty: 1, price: 10000 }],
  });
  const postRes = http.post(`${BASE_URL}/orders`, payload, {
    headers: { 'Content-Type': 'application/json' },
  });
  check(postRes, {
    'POST /orders status 200 or 201': (r) => r.status === 200 || r.status === 201,
  });

  sleep(1);
}
```

### Trade-off & Pitfall
- k6 mengulang `default function` per VU secepat mungkin, cuma dibatasi `sleep(1)` di akhir tiap iterasi di atas -- lupa `sleep` bikin tiap VU menghantam endpoint jauh lebih agresif dibanding pola traffic user asli, hasilnya jadi gak representatif.
- JANGAN jalankan load test di atas langsung ke database PRODUKSI beneran -- `POST /orders` yang diuji betulan memanggil `CreateOrder` dan mengurangi `stock` produk sungguhan; selalu jalankan di environment staging terpisah dengan data dummy.
- Load generator Go di atas gak punya fitur bawaan k6 seperti stage ramp-up/ramp-down, threshold gate otomatis, atau laporan P50/P95/P99 lengkap -- cukup buat smoke-test cepat, tapi buat laporan performa yang serius tetap pakai k6 (atau tool sejenis: Gatling, Locust).
- Kalau skrip cuma menguji SATU `product_id` yang fixed, lock contention `FOR UPDATE` (Phase 3 topik 34) bikin hasil P95/P99 terlihat jauh lebih buruk daripada traffic asli yang tersebar ke banyak product berbeda -- skrip k6 di atas sengaja memakai `product_id` acak (1-20) supaya lebih representatif.

### Kapan Dipakai
Sebelum event traffic tinggi yang diprediksi (flash sale, promo besar) di OrderFlow, dan sebagai bagian rutin sebelum rilis besar untuk memastikan perubahan baru gak menurunkan kapasitas yang sudah divalidasi sebelumnya.

### Sering Ditanya Saat Interview
- "Apa beda P95 dan P99, dan kenapa dua-duanya penting?" -- P95 berarti 95% request lebih cepat dari angka itu (5% lebih lambat); P99 lebih ketat (cuma 1% yang lebih lambat) dan menyoroti ekor terburuk yang sering diabaikan rata-rata -- penting karena user yang kena ekor lambat itu tetap user nyata dengan pengalaman buruk.
- "Kenapa thresholds di k6 penting buat CI/CD (Phase 12)?" -- threshold bikin k6 keluar dengan exit code error kalau kriteria (P95 latency, error rate) gak terpenuhi, sehingga bisa dijadikan gate otomatis di pipeline -- rilis yang menurunkan performa ketahuan sebelum sampai ke production, bukan setelah user complain.
- "Kenapa jumlah VU (Virtual User) di k6 gak sama persis dengan RPS (requests per second)?" -- satu VU adalah satu 'thread' simulasi yang mengulang skenario terus-menerus; berapa banyak request/detik yang benar-benar dihasilkan tergantung berapa lama tiap iterasi (termasuk `sleep`) -- makanya k6 melaporkan RPS aktual (`http_reqs`) terpisah dari jumlah VU yang dikonfigurasi.

---

**Selanjutnya:** [Phase 14 — Advanced Backend Concepts](./phase-14-advanced-concepts.md)
