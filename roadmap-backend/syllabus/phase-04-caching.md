# Phase 04 — Caching

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

## 41. Why Cache?

### Apa itu?
Caching adalah teknik menyimpan hasil operasi yang mahal (paling umum: hasil query database) di storage yang jauh lebih cepat diakses -- biasanya in-memory store seperti Redis -- supaya request berikutnya yang butuh data yang sama gak perlu mengulang operasi mahal itu dari nol.

### Kenapa dibutuhkan?
Endpoint `GET /products/:id` di OrderFlow termasuk yang paling sering dipanggil -- tiap orang buka halaman produk, request itu jalan. Tanpa cache, tiap request selalu lewat `GetProductByID` (topik 25) balik ke Postgres, padahal sebagian besar kolomnya (`name`, `price`) nyaris gak pernah berubah dalam hitungan menit. Di traffic tinggi, pola ini bikin Postgres jadi bottleneck buat data yang sebenarnya jawabannya sering-sering sama persis.

### Cara Kerja
```
Tanpa cache (sekarang):
  Request 1 -> GetProductByID -> Postgres  (~5-15ms, tergantung load)
  Request 2 -> GetProductByID -> Postgres  (~5-15ms, query IDENTIK)
  Request 3 -> GetProductByID -> Postgres  (~5-15ms, query IDENTIK lagi)
  ...ribuan request/detik -> Postgres kena beban ribuan query identik/detik

Dengan cache (topik 42 dst):
  Request 1 -> cache MISS -> Postgres -> simpan hasilnya di cache (~5-15ms)
  Request 2 -> cache HIT  -> balik dari Redis langsung             (~0.5-1ms)
  Request 3 -> cache HIT  -> balik dari Redis langsung             (~0.5-1ms)
```

### Contoh Kode — Go
```go
package api

import (
	"encoding/json"
	"net/http"
	"strconv"

	"github.com/jackc/pgx/v5/pgxpool"

	"orderflow/db"
)

// GetProductHandler versi awal (belum ada cache): tiap request produk selalu
// lewat GetProductByID ke Postgres, walau datanya (name, price) nyaris gak
// pernah berubah dan endpoint ini termasuk yang paling sering dipanggil di
// OrderFlow.
func GetProductHandler(pool *pgxpool.Pool) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		id, err := strconv.ParseInt(r.PathValue("id"), 10, 64)
		if err != nil {
			http.Error(w, "invalid product id", http.StatusBadRequest)
			return
		}

		product, err := db.GetProductByID(r.Context(), pool, id)
		if err != nil {
			http.Error(w, "internal error", http.StatusInternalServerError)
			return
		}
		if product == nil {
			http.Error(w, "product not found", http.StatusNotFound)
			return
		}

		w.Header().Set("Content-Type", "application/json")
		json.NewEncoder(w).Encode(product)
	}
}
```

### Contoh Kode — Node.js
```javascript
const { getProductById } = require('./db');

// getProductHandler versi awal (belum ada cache): tiap request produk selalu
// lewat getProductById ke Postgres.
function getProductHandler(pool) {
  return async function (req, res) {
    const id = Number(req.params.id);
    if (!Number.isInteger(id)) {
      res.status(400).json({ error: 'invalid product id' });
      return;
    }

    const product = await getProductById(pool, id);
    if (!product) {
      res.status(404).json({ error: 'product not found' });
      return;
    }

    res.json(product);
  };
}

module.exports = { getProductHandler };
```

### Trade-off & Pitfall
- Cache bukan solusi gratis: dia menukar konsistensi instan dengan kecepatan -- ada jeda waktu (staleness window) di mana cache bisa gak sinkron sama data asli di Postgres, dan itu harus dikelola sadar lewat TTL (topik 44) dan invalidation (topik 43), bukan diabaikan.
- Caching paling worth it buat data yang **sering dibaca tapi jarang berubah** (read-heavy, write-light) -- product listing cocok, tapi data yang berubah tiap detik (misalnya live stock counter saat flash sale) mungkin gak cocok di-cache dengan TTL panjang.
- Menambah cache berarti menambah satu komponen infrastruktur baru yang bisa gagal (topik 46) -- kalau gak didesain fail-open, Redis yang down malah bisa ikut menjatuhkan endpoint yang tadinya cuma mengandalkan Postgres.

### Kapan Dipakai
Ketika ada endpoint read-heavy dengan data yang gak berubah tiap request (product detail, kategori, konfigurasi) dan Postgres mulai kelihatan kena beban dari query yang berulang persis sama.

### Sering Ditanya Saat Interview
- "Kapan caching justru gak membantu?" — untuk data yang berubah sangat sering (hampir tiap request beda) atau data yang butuh strong consistency (misalnya saldo yang harus akurat detik itu juga tanpa toleransi delay), caching malah menambah kompleksitas tanpa manfaat nyata.
- "Apa risiko terbesar menambahkan cache ke sistem yang sudah jalan?" — stale data (data cache gak sinkron sama sumber asli) kalau invalidation-nya gak lengkap, dan single point of failure baru kalau cache layer-nya gak didesain fail-open.
- "Kenapa gak semua endpoint di-cache saja biar cepat semua?" — endpoint yang datanya sering berubah atau butuh data real-time (seperti status pembayaran) justru berisiko menyajikan data basi kalau di-cache; caching cuma cocok untuk data yang toleran terhadap delay singkat.

---

## 42. Cache-Aside

### Apa itu?
Cache-aside (disebut juga lazy loading) adalah pola caching paling umum: application code sendiri yang mengecek cache dulu -- kalau ketemu (cache hit), langsung dipakai; kalau gak ketemu (cache miss), baru query ke database, lalu isi cache-nya supaya request berikutnya kena hit.

### Kenapa dibutuhkan?
Untuk menutup masalah di topik 41 tanpa mengubah cara `GetProductByID` bekerja sama sekali. Cache-aside membungkus fungsi itu jadi `GetProductCached`/`getProductCached` -- caller (handler HTTP) cukup ganti pemanggilan fungsi, logic Postgres yang sudah ada di Phase 03 gak perlu disentuh sama sekali.

### Cara Kerja
```
GetProductCached(id):
  1. GET product:{id} dari Redis
  2a. Kalau ADA (hit)  -> unmarshal JSON -> return, SELESAI (gak sentuh Postgres)
  2b. Kalau GAK ADA (miss):
      -> GetProductByID(id) ke Postgres
      -> marshal hasilnya ke JSON
      -> SET product:{id} = JSON, dengan TTL (topik 44)
      -> return hasilnya
```

### Contoh Kode — Go
```go
package db

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/redis/go-redis/v9"
)

const productCacheTTL = 5 * time.Minute

func productCacheKey(id int64) string {
	return fmt.Sprintf("product:%d", id)
}

// GetProductCached ambil product lewat pola cache-aside: cek Redis dulu,
// kalau miss baru query Postgres lewat GetProductByID (topik 25) dan isi
// balik cache-nya buat request berikutnya.
func GetProductCached(ctx context.Context, rdb *redis.Client, db *pgxpool.Pool, id int64) (*Product, error) {
	key := productCacheKey(id)

	cached, err := rdb.Get(ctx, key).Result()
	if err == nil {
		var p Product
		if err := json.Unmarshal([]byte(cached), &p); err != nil {
			return nil, fmt.Errorf("unmarshal cached product %d: %w", id, err)
		}
		return &p, nil
	}
	if !errors.Is(err, redis.Nil) {
		return nil, fmt.Errorf("get cache %s: %w", key, err)
	}

	p, err := GetProductByID(ctx, db, id)
	if err != nil {
		return nil, err
	}
	if p == nil {
		return nil, nil
	}

	data, err := json.Marshal(p)
	if err != nil {
		return nil, fmt.Errorf("marshal product %d: %w", id, err)
	}
	if err := rdb.Set(ctx, key, data, productCacheTTL).Err(); err != nil {
		return nil, fmt.Errorf("set cache %s: %w", key, err)
	}
	return p, nil
}
```

### Contoh Kode — Node.js
```javascript
const { getProductById } = require('./db');

const PRODUCT_CACHE_TTL_SECONDS = 5 * 60;

function productCacheKey(id) {
  return `product:${id}`;
}

// getProductCached ambil product lewat pola cache-aside: cek Redis dulu,
// kalau miss baru query Postgres lewat getProductById dan isi balik
// cache-nya buat request berikutnya.
async function getProductCached(redisClient, pool, id) {
  const key = productCacheKey(id);

  const cached = await redisClient.get(key);
  if (cached) {
    return JSON.parse(cached);
  }

  const product = await getProductById(pool, id);
  if (!product) {
    return null;
  }

  await redisClient.set(key, JSON.stringify(product), 'EX', PRODUCT_CACHE_TTL_SECONDS);
  return product;
}

module.exports = { getProductCached, productCacheKey };
```

### Trade-off & Pitfall
- Request pertama ke sebuah product (atau request pertama setelah cache expired) selalu kena cache miss -- latensinya sama seperti gak ada cache sama sekali (cold start), plus sedikit overhead nulis ke Redis.
- Cache-aside gak otomatis update kalau data di Postgres berubah lewat jalur lain (misalnya `CreateOrder` mendekremen `stock`) -- tanpa invalidation eksplisit (topik 43), cache bisa nyimpen data basi sampai TTL-nya habis sendiri.
- Kalau `p == nil` (product gak ketemu di Postgres), versi ini sengaja gak menyimpan "negative cache" -- artinya request berulang buat product yang gak ada akan selalu tembus ke Postgres; ini trade-off sadar supaya implementasinya tetap sederhana di fase ini.

### Kapan Dipakai
Untuk data read-heavy yang gak butuh akurasi real-time mutlak, di mana caller siap menerima jeda singkat antara data berubah di database dan cache ter-update -- seperti detail product yang tampil di halaman katalog.

### Sering Ditanya Saat Interview
- "Jelaskan alur cache-aside dari `GetProductCached`." — cek Redis dulu pakai key `product:{id}`; kalau hit, unmarshal dan return langsung; kalau miss, query Postgres lewat `GetProductByID`, lalu simpan hasilnya ke Redis dengan TTL sebelum di-return ke caller.
- "Apa beda cache-aside dengan write-through?" — cache-aside mengisi cache secara lazy cuma pas ada cache miss saat baca; write-through mengisi/update cache langsung pas data ditulis ke database, jadi cache selalu lebih up-to-date tapi nulis-nya jadi lebih lambat (dua tempat harus ditulis).
- "Kenapa `errors.Is(err, redis.Nil)` dicek secara eksplisit?" — `redis.Nil` adalah cara go-redis menandakan "key gak ada" (bukan error koneksi beneran); kalau gak dibedakan dari error Redis lainnya, cache miss yang normal bisa salah dianggap sebagai kegagalan sistem.

---

## 43. Cache Invalidation

### Apa itu?
Cache invalidation adalah proses menghapus atau mengganti entry cache yang sudah gak valid lagi, begitu data sumbernya (Postgres) berubah -- supaya pembacaan berikutnya gak dapat versi lama yang udah kadaluarsa.

### Kenapa dibutuhkan?
`GetProductCached` (topik 42) cuma mengandalkan TTL buat "menyegarkan" data. Masalahnya, `stock` sebuah product berubah tiap kali ada order baru lewat `CreateOrder` (topik 29) -- kalau cuma andalin TTL, user bisa lihat stock yang salah (misalnya masih kelihatan tersedia padahal barusan habis dibeli orang lain) selama beberapa menit sampai TTL-nya berakhir sendiri.

### Cara Kerja
```
CreateOrderInvalidateCache(userID, items):
  1. CreateOrder(userID, items)   -- transaction: decrement stock, insert order & items
  2. Kalau sukses, buat tiap item yang dibeli:
       DEL product:{productID}    -- buang cache lama dari Redis
  3. Return order

Setelah langkah 2, GetProductCached buat product itu pasti MISS di request
berikutnya -> otomatis ambil ulang dari Postgres -> stock yang ditampilkan
sudah yang terbaru, gak perlu nunggu TTL habis.
```

### Contoh Kode — Go
```go
package db

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/redis/go-redis/v9"
)

// InvalidateProductCache hapus cache key satu product -- dipanggil setelah
// ada perubahan data product (misalnya stock berkurang lewat CreateOrder),
// supaya read berikutnya lewat GetProductCached wajib ambil data terbaru
// dari Postgres, bukan versi lama yang sudah kadaluarsa dari cache.
func InvalidateProductCache(ctx context.Context, rdb *redis.Client, id int64) error {
	if err := rdb.Del(ctx, productCacheKey(id)).Err(); err != nil {
		return fmt.Errorf("invalidate cache product %d: %w", id, err)
	}
	return nil
}

// CreateOrderInvalidateCache membungkus CreateOrder (topik 29): setelah order
// berhasil dibuat dan stock produk-produk yang dibeli sudah didekremen di
// Postgres, tiap product cache key-nya diinvalidasi supaya GetProductCached
// berikutnya gak balikin stock lama yang udah gak akurat.
func CreateOrderInvalidateCache(ctx context.Context, rdb *redis.Client, db *pgxpool.Pool, userID int64, items []OrderItem) (*Order, error) {
	order, err := CreateOrder(ctx, db, userID, items)
	if err != nil {
		return nil, err
	}

	for _, item := range items {
		if err := InvalidateProductCache(ctx, rdb, item.ProductID); err != nil {
			// Order-nya sendiri sudah sukses -- kegagalan invalidate cache gak
			// boleh bikin caller mikir order-nya gagal, cukup dicatat/di-retry
			// di luar (misalnya lewat message queue, Phase 05).
			return order, fmt.Errorf("order %d created but cache invalidation failed: %w", order.ID, err)
		}
	}
	return order, nil
}
```

### Contoh Kode — Node.js
```javascript
const { createOrder } = require('./db');
const { productCacheKey } = require('./cache');

// invalidateProductCache hapus cache key satu product -- dipanggil setelah
// ada perubahan data product.
async function invalidateProductCache(redisClient, id) {
  await redisClient.del(productCacheKey(id));
}

// createOrderInvalidateCache membungkus createOrder (topik 29): setelah order
// berhasil dibuat, tiap product cache key yang stock-nya berubah diinvalidasi.
async function createOrderInvalidateCache(redisClient, pool, userId, items) {
  const order = await createOrder(pool, userId, items);

  for (const item of items) {
    try {
      await invalidateProductCache(redisClient, item.productId);
    } catch (err) {
      // Order-nya sendiri sudah sukses -- kegagalan invalidate cache gak
      // boleh bikin caller mikir order-nya gagal.
      console.error(`order ${order.id} created but cache invalidation failed`, err);
    }
  }
  return order;
}

module.exports = { invalidateProductCache, createOrderInvalidateCache };
```

### Trade-off & Pitfall
- Invalidation harus dijalankan **setelah** transaction `CreateOrder` benar-benar commit, bukan sebelumnya -- kalau invalidate duluan lalu transaction-nya gagal/rollback, cache jadi kosong (miss) padahal data di Postgres gak berubah sama sekali, dan kalau invalidate sebelum commit lalu ada read di antaranya, request itu bisa langsung reload data lama yang belum ter-commit sebagai cache baru.
- Invalidation cuma bekerja kalau **semua** jalur yang mengubah data ikut memanggilnya -- kalau ada tempat lain yang update `products.stock` langsung (migrasi manual, admin panel terpisah) tanpa lewat `CreateOrderInvalidateCache`, cache-nya diam-diam jadi basi tanpa ada yang sadar.
- Hapus cache (`DEL`) lebih aman daripada langsung nulis ulang value baru ke cache (`SET` dengan data baru) di titik invalidasi -- kalau ada race condition dua request nulis cache bersamaan dengan data yang berbeda urutannya, `DEL` cuma bikin cache miss (aman, tinggal reload), sementara `SET` langsung berisiko cache ke-overwrite data yang salah urutan.

### Kapan Dipakai
Setiap kali ada operasi yang mengubah data yang juga di-cache di tempat lain -- terutama operasi yang sering dipanggil dan datanya langsung berdampak ke yang ditampilkan ke user, seperti `CreateOrder` yang mengubah `stock` product.

### Sering Ditanya Saat Interview
- "Kenapa `CreateOrderInvalidateCache` invalidate cache SETELAH `CreateOrder` sukses, bukan sebelum atau di tengah?" — supaya invalidation cuma terjadi kalau perubahan datanya beneran ter-commit; kalau invalidate duluan lalu transaction gagal, cache jadi kosong padahal data gak berubah, dan kalau invalidate di tengah transaction yang belum commit, request lain bisa reload data yang belum final.
- "Apa yang terjadi kalau ada tim lain nambah fitur update stock manual tanpa lewat `CreateOrderInvalidateCache`?" — cache-nya gak ikut ter-invalidate, jadi user bisa lihat stock lama sampai TTL (topik 44) habis sendiri -- ini kenapa TTL tetap penting sebagai safety net, gak boleh cuma mengandalkan invalidation manual.
- "Kenapa pakai `DEL` buat invalidate, bukan langsung `SET` value baru ke cache?" — `DEL` lebih aman dari race condition: kalau ada penulisan cache yang tumpang tindih urutannya, hasil paling buruk cuma cache miss (lalu reload dari database), bukan cache ke-overwrite dengan data yang salah/basi.

---

## 44. TTL

### Apa itu?
TTL (Time-To-Live) adalah durasi hidup sebuah cache key sebelum otomatis dihapus/expired oleh Redis sendiri, tanpa perlu ada yang eksplisit memanggil `DEL`.

### Kenapa dibutuhkan?
Invalidation (topik 43) cuma bisa jalan kalau semua jalur perubahan data ikut memanggilnya -- realitanya, hampir gak pernah ada jaminan 100% semua jalur tercover (bug, jalur baru yang lupa diupdate, migrasi manual). TTL jadi jaring pengaman: walaupun ada satu jalur invalidation yang kelewat, cache basi itu paling lama cuma bertahan selama TTL-nya, gak selamanya.

### Cara Kerja
```
TTL pendek (misal 30 detik):
  + staleness window kecil, cepat "sembuh sendiri" kalau invalidation kelewat
  - cache hit ratio lebih rendah, lebih sering balik ke Postgres

TTL panjang (misal 1 jam):
  + cache hit ratio tinggi, beban Postgres jauh berkurang
  - kalau ada invalidation yang kelewat, data basi bisa nyangkut sampai 1 jam

TTL yang tepat: seberapa toleran bisnis terhadap data yang sedikit basi,
dikombinasikan dengan seberapa andal invalidation (topik 43) di jalur itu.
```

### Contoh Kode — Go
```go
package db

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/redis/go-redis/v9"
)

// GetProductCachedWithTTL sama seperti GetProductCached (topik 42), tapi TTL
// cache-nya bisa dikonfigurasi per pemanggilan -- product yang harganya lagi
// sering berubah (misalnya lagi flash sale) bisa dikasih TTL pendek, sementara
// product biasa bisa pakai TTL yang lebih panjang.
func GetProductCachedWithTTL(ctx context.Context, rdb *redis.Client, db *pgxpool.Pool, id int64, ttl time.Duration) (*Product, error) {
	key := productCacheKey(id)

	cached, err := rdb.Get(ctx, key).Result()
	if err == nil {
		var p Product
		if err := json.Unmarshal([]byte(cached), &p); err != nil {
			return nil, fmt.Errorf("unmarshal cached product %d: %w", id, err)
		}
		return &p, nil
	}
	if !errors.Is(err, redis.Nil) {
		return nil, fmt.Errorf("get cache %s: %w", key, err)
	}

	p, err := GetProductByID(ctx, db, id)
	if err != nil {
		return nil, err
	}
	if p == nil {
		return nil, nil
	}

	data, err := json.Marshal(p)
	if err != nil {
		return nil, fmt.Errorf("marshal product %d: %w", id, err)
	}
	if err := rdb.Set(ctx, key, data, ttl).Err(); err != nil {
		return nil, fmt.Errorf("set cache %s: %w", key, err)
	}
	return p, nil
}
```

### Contoh Kode — Node.js
```javascript
const { getProductById } = require('./db');
const { productCacheKey } = require('./cache');

// getProductCachedWithTTL sama seperti getProductCached (topik 42), tapi TTL
// cache-nya bisa dikonfigurasi per pemanggilan lewat parameter ttlSeconds.
async function getProductCachedWithTTL(redisClient, pool, id, ttlSeconds) {
  const key = productCacheKey(id);

  const cached = await redisClient.get(key);
  if (cached) {
    return JSON.parse(cached);
  }

  const product = await getProductById(pool, id);
  if (!product) {
    return null;
  }

  await redisClient.set(key, JSON.stringify(product), 'EX', ttlSeconds);
  return product;
}

module.exports = { getProductCachedWithTTL };
```

### Trade-off & Pitfall
- TTL yang terlalu pendek nyaris meniadakan manfaat caching -- kalau TTL-nya lebih pendek dari rata-rata jeda antar request buat product yang sama, cache-nya nyaris selalu miss dan Postgres tetap kena beban seperti gak ada cache.
- TTL yang terlalu panjang bikin dampak dari invalidation yang kelewat (topik 43) jadi jauh lebih parah -- data basi bisa nyangkut berjam-jam kalau gak ada satupun jalur yang menginvalidasi cache-nya.
- Jangan pakai satu TTL global buat semua jenis data -- product yang lagi flash sale (harga/stock berubah cepat) butuh TTL jauh lebih pendek dibanding product biasa yang datanya nyaris statis berminggu-minggu.

### Kapan Dipakai
Selalu -- TTL harus tetap dipasang di setiap cache key walaupun sudah ada invalidation eksplisit (topik 43), sebagai defense-in-depth kalau-kalau ada jalur invalidation yang lolos gak tertangani.

### Sering Ditanya Saat Interview
- "Kalau sudah ada cache invalidation yang benar, masih perlu TTL gak?" — perlu, sebagai safety net; invalidation manual gampang kelewat kalau ada jalur baru yang lupa diupdate, TTL membatasi seberapa lama dampak dari kelalaian itu bisa bertahan.
- "Gimana cara menentukan angka TTL yang tepat?" — lihat seberapa sering data itu berubah dan seberapa toleran bisnis terhadap staleness-nya; data yang jarang berubah dan gak kritis (deskripsi produk) bisa TTL panjang, data yang lebih volatile (stock saat flash sale) butuh TTL pendek atau malah lebih mengandalkan invalidation eksplisit.
- "Apa yang terjadi kalau TTL disetel 0 atau gak disetel sama sekali?" — key-nya gak pernah expired sendiri di Redis, jadi satu-satunya cara cache-nya bersih adalah invalidation eksplisit -- kalau ada jalur yang lupa invalidate, data basi itu bisa nyangkut selamanya.

---

## 45. Cache Stampede

### Apa itu?
Cache stampede (disebut juga dogpile effect atau thundering herd) adalah kondisi ketika sebuah cache key yang sangat sering diakses (hot key) expired, dan banyak request datang bersamaan persis di momen itu -- semuanya sama-sama kena cache miss, dan semuanya sama-sama menghajar Postgres di waktu yang sama buat data yang identik.

### Kenapa dibutuhkan?
Product-product terlaris di OrderFlow bisa menerima ribuan request/detik ke endpoint yang sama. Kalau cache key product itu (topik 42, 44) expired pas lagi rame-ramenya, semua request yang harusnya cukup satu query ke Postgres malah jadi ribuan query identik dalam sekejap -- tepat pas Postgres paling gak butuh lonjakan beban itu.

### Cara Kerja
```
Tanpa proteksi stampede:
  t0    product:99 expired
  t0+ε  1000 request datang bersamaan, SEMUANYA cache miss
        -> 1000 query GetProductByID ke Postgres, buat data yang SAMA PERSIS

Dengan singleflight/dedup:
  t0    product:99 expired
  t0+ε  1000 request datang bersamaan, SEMUANYA cache miss
        -> cuma request PERTAMA yang benar-benar query ke Postgres
        -> 999 request lain nunggu hasil query yang sama, gak query sendiri-sendiri
```

### Contoh Kode — Go
```go
package db

import (
	"context"
	"fmt"
	"strconv"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/redis/go-redis/v9"
	"golang.org/x/sync/singleflight"
)

var productLoadGroup singleflight.Group

// GetProductCachedSingleflight membungkus GetProductCached (topik 42) dengan
// singleflight: kalau cache key sebuah product lagi expired dan banyak
// request datang bersamaan buat product yang sama, cuma SATU goroutine yang
// benar-benar query ke Postgres dan isi ulang cache -- goroutine lain cukup
// nunggu hasil goroutine pertama, gak ikutan menghajar Postgres bareng-bareng
// (cache stampede / dogpile effect).
func GetProductCachedSingleflight(ctx context.Context, rdb *redis.Client, db *pgxpool.Pool, id int64) (*Product, error) {
	key := "product-load:" + strconv.FormatInt(id, 10)

	result, err, _ := productLoadGroup.Do(key, func() (interface{}, error) {
		return GetProductCached(ctx, rdb, db, id)
	})
	if err != nil {
		return nil, fmt.Errorf("load product %d: %w", id, err)
	}
	if result == nil {
		return nil, nil
	}
	return result.(*Product), nil
}
```

### Contoh Kode — Node.js
```javascript
const { getProductCached } = require('./cache');

const inFlightLoads = new Map();

// getProductCachedSingleflight membungkus getProductCached (topik 42) dengan
// dedup di level proses: kalau ada beberapa request bersamaan yang minta
// product sama pas cache-nya lagi expired, cuma satu Promise query yang
// benar-benar jalan -- request lain ikut nebeng Promise yang sama, gak
// masing-masing bikin query baru ke Postgres (cache stampede).
async function getProductCachedSingleflight(redisClient, pool, id) {
  const key = `product-load:${id}`;

  if (inFlightLoads.has(key)) {
    return inFlightLoads.get(key);
  }

  const promise = getProductCached(redisClient, pool, id).finally(() => {
    inFlightLoads.delete(key);
  });
  inFlightLoads.set(key, promise);

  return promise;
}

module.exports = { getProductCachedSingleflight };
```

### Trade-off & Pitfall
- `singleflight` (Go) dan `Map` in-flight (Node.js) cuma dedup di dalam **satu proses/instance**; kalau OrderFlow jalan di banyak instance sekaligus di belakang load balancer, tiap instance tetap bisa kirim satu query ke Postgres di saat yang sama -- buat proteksi lintas instance, butuh distributed lock di Redis sendiri (misalnya `SET key value NX EX ttl`).
- Request yang "nebeng" hasil singleflight/in-flight promise ikut menunggu selama request pertama selesai -- kalau query pertama lambat/gagal, semua yang nebeng ikut lambat/gagal bareng, jadi query yang dibungkus sebaiknya punya timeout yang jelas.
- Solusi lain yang sering dipakai bareng (bukan pengganti) adalah "TTL jitter" -- kasih variasi kecil random ke TTL tiap key (topik 44), supaya banyak key gak expired persis di detik yang sama dan memicu stampede bareng-bareng ke banyak key sekaligus.

### Kapan Dipakai
Untuk cache key yang traffic-nya sangat tinggi (hot key) dan biaya query fallback-nya (ke Postgres) cukup mahal -- product terlaris atau data yang ditampilkan di halaman utama adalah kandidat paling jelas.

### Sering Ditanya Saat Interview
- "Apa itu cache stampede dan kapan itu terjadi?" — kondisi ketika satu cache key yang sangat populer expired, dan banyak request bersamaan sama-sama cache miss lalu sama-sama query ke database di waktu yang sama, menciptakan lonjakan beban mendadak.
- "Kenapa `singleflight` di Go gak cukup buat OrderFlow yang jalan di banyak instance/pod?" — `singleflight` cuma mendedup pemanggilan dalam satu proses; instance lain yang jalan paralel gak tau ada singleflight itu, jadi tetap bisa mengirim query duplikat ke Postgres di waktu yang sama. Perlu distributed lock di level Redis buat proteksi lintas instance.
- "Selain singleflight/lock, cara apa lagi mencegah banyak key expired bersamaan?" — menambahkan jitter (variasi acak kecil) ke TTL tiap key, supaya key-key populer gak semuanya expired persis di detik yang sama, menyebarkan potensi lonjakan beban ke waktu yang berbeda-beda.

---

## 46. Redis

### Apa itu?
Redis adalah in-memory data structure store yang dipakai OrderFlow sebagai layer cache (topik 41-45), diakses lewat client `go-redis` di Go dan `ioredis` di Node.js. Selain string biasa (dipakai buat cache product di atas), Redis juga punya struktur data lain (hash, list, set, sorted set) yang berguna di luar konteks caching murni.

### Kenapa dibutuhkan?
Semua topik cache-aside, invalidation, TTL, dan anti-stampede di atas jalan di atas infrastruktur Redis yang nyata -- yang berarti Redis sendiri punya karakteristik operasional yang harus dipahami: bagaimana koneksinya dikonfigurasi, bagaimana cek kesehatannya, dan yang paling penting -- apa yang terjadi kalau Redis-nya sendiri down. Kalau gak didesain dengan benar, cache yang tujuannya mempercepat sistem malah bisa jadi titik kegagalan baru yang menjatuhkan OrderFlow.

### Cara Kerja
```
Client setup:
  - DialTimeout / ReadTimeout / WriteTimeout dikonfigurasi pendek,
    supaya kalau Redis lambat/gak respons, request gak nyangkut lama

Health check:
  - PING ke Redis, terpisah dari health check Postgres,
    supaya operator tau layer mana yang bermasalah

Fail-open (paling penting):
  - Kalau GetProductCached error karena Redis down,
    JANGAN gagalkan request -- fallback ke GetProductByID langsung ke Postgres.
    Cache itu OPTIMASI, bukan dependency keras. Redis down = lebih lambat,
    BUKAN = OrderFlow ikut down.
```

### Contoh Kode — Go
```go
package db

import (
	"context"
	"fmt"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/redis/go-redis/v9"
)

// NewRedisClient bikin koneksi ke Redis yang dipakai sebagai layer cache
// OrderFlow (GetProductCached dkk, topik 42-45) -- dikonfigurasi dengan
// timeout yang pendek supaya request gak nyangkut lama kalau Redis-nya
// lagi bermasalah.
func NewRedisClient(addr, password string, dbIndex int) *redis.Client {
	return redis.NewClient(&redis.Options{
		Addr:         addr,
		Password:     password,
		DB:           dbIndex,
		DialTimeout:  2 * time.Second,
		ReadTimeout:  1 * time.Second,
		WriteTimeout: 1 * time.Second,
	})
}

// PingRedis cek konektivitas Redis -- dipakai health check endpoint OrderFlow
// buat tau apakah layer cache-nya sehat, terpisah dari status Postgres.
func PingRedis(ctx context.Context, rdb *redis.Client) error {
	if err := rdb.Ping(ctx).Err(); err != nil {
		return fmt.Errorf("ping redis: %w", err)
	}
	return nil
}

// GetProductCachedFailOpen sama seperti GetProductCached (topik 42), tapi
// kalau Redis lagi down/error, gak bikin request gagal total -- langsung
// fallback ke Postgres lewat GetProductByID. Cache itu optimasi, bukan
// dependency keras; Redis mati seharusnya bikin lebih lambat, bukan bikin
// OrderFlow ikutan down.
func GetProductCachedFailOpen(ctx context.Context, rdb *redis.Client, db *pgxpool.Pool, id int64) (*Product, error) {
	product, err := GetProductCached(ctx, rdb, db, id)
	if err != nil {
		return GetProductByID(ctx, db, id)
	}
	return product, nil
}
```

### Contoh Kode — Node.js
```javascript
const Redis = require('ioredis');
const { getProductCached } = require('./cache');
const { getProductById } = require('./db');

// createRedisClient bikin koneksi ke Redis yang dipakai sebagai layer cache
// OrderFlow, dengan retry strategy yang wajar supaya gak langsung nyerah
// kalau koneksi sempat putus sebentar.
function createRedisClient(host, port) {
  return new Redis({
    host,
    port,
    connectTimeout: 2000,
    retryStrategy(times) {
      return Math.min(times * 100, 2000);
    },
  });
}

// pingRedis cek konektivitas Redis -- dipakai health check endpoint OrderFlow.
async function pingRedis(redisClient) {
  const reply = await redisClient.ping();
  return reply === 'PONG';
}

// getProductCachedFailOpen sama seperti getProductCached (topik 42), tapi
// kalau Redis lagi down/error, langsung fallback ke Postgres lewat
// getProductById daripada bikin request gagal total.
async function getProductCachedFailOpen(redisClient, pool, id) {
  try {
    return await getProductCached(redisClient, pool, id);
  } catch (err) {
    return getProductById(pool, id);
  }
}

module.exports = { createRedisClient, pingRedis, getProductCachedFailOpen };
```

### Trade-off & Pitfall
- Redis itu single-threaded buat eksekusi command (walau I/O-nya bisa multi-threaded di versi baru) -- ini bikin tiap command atomic secara alami, tapi juga berarti satu command yang lambat (misalnya `KEYS *` di dataset besar) bisa memblokir semua command lain yang antre di belakangnya.
- Fail-open (topik ini) penting, tapi jangan sampai bikin error Redis "hilang" tanpa jejak -- error dari `GetProductCached` yang di-fallback tetap harus di-log/di-monitor, supaya tim tau Redis lagi bermasalah walau user-facing request tetap sukses.
- Redis punya opsi persistence (RDB snapshot berkala, atau AOF yang mencatat tiap write) -- tapi untuk pure cache layer seperti di OrderFlow ini, kehilangan seluruh isi Redis saat restart itu gak masalah (data aslinya tetap aman di Postgres), jadi persistence yang terlalu agresif cuma nambah overhead tanpa manfaat nyata untuk use case ini.

### Kapan Dipakai
Redis cocok dipakai kapan pun OrderFlow butuh storage bersama yang cepat dan dipakai lintas instance aplikasi -- bukan cuma buat cache product seperti di topik 42-45, tapi juga use case lain seperti session store, rate limiting counter, atau distributed lock (disebut singkat di topik 45).

### Sering Ditanya Saat Interview
- "Kenapa desain caching di OrderFlow harus fail-open kalau Redis down?" — karena cache itu optimasi kecepatan, bukan sumber data utama; kalau Redis down bikin seluruh request gagal, cache malah jadi single point of failure baru yang lebih parah dampaknya dibanding Postgres yang jadi lambat sedikit.
- "Apa risiko dari sifat single-threaded Redis?" — satu command yang mahal (misalnya scan seluruh keyspace) bisa memblokir semua command lain yang menunggu di antrean, walau tiap command individualnya sangat cepat.
- "Apakah data di Redis perlu di-backup/persist kalau cuma dipakai sebagai cache?" — untuk pure cache seperti product cache di OrderFlow, tidak wajib -- kalau Redis restart dan kehilangan semua data, itu setara "semua key expired sekaligus", cache-aside (topik 42) otomatis mengisi ulang dari Postgres seperti biasa, gak ada data yang benar-benar hilang.

---

**Selanjutnya:** [Phase 05 — Message Queue & Async Processing](./phase-05-message-queue.md)
