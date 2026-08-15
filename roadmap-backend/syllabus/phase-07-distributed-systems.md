# Phase 07 — Distributed Systems

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

## 65. Vertical vs Horizontal Scaling

### Apa itu?
Vertical scaling (scale up) berarti nambah kapasitas satu mesin/instance yang sama — CPU, RAM, atau disk yang lebih besar. Horizontal scaling (scale out) berarti nambah jumlah instance yang menjalankan service yang sama, lalu request didistribusikan ke semua instance itu lewat load balancer (topik 67).

### Kenapa dibutuhkan?
Saat OrderFlow kena flash sale, traffic bisa naik puluhan kali lipat dalam hitungan menit. Vertical scaling punya batas fisik (mesin terbesar yang tersedia di cloud provider) dan biasanya butuh downtime buat resize instance. Horizontal scaling jauh lebih elastis — tinggal nambah instance baru di belakang load balancer — tapi menuntut service didesain stateless (topik 66) supaya request bisa mendarat di instance manapun tanpa masalah.

### Cara Kerja
```
Vertical scaling:
  [ API instance: 2 vCPU, 4GB ]  -->  [ API instance: 8 vCPU, 16GB ]
  (mesin yang sama, diperbesar)

Horizontal scaling:
  [ API instance ] [ API instance ] [ API instance ]
         \               |               /
          \--------- Load Balancer -----/
                        |
                     Client
  (banyak mesin identik, traffic dibagi rata)
```

### Contoh Kode — Go
```go
package main

import (
	"context"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

// main menjalankan satu instance OrderFlow API. Saat horizontal scaling
// nambah/ngurangin jumlah instance, tiap instance harus bisa start dan
// berhenti dengan aman tanpa memotong request yang sedang berjalan.
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/healthz", healthzHandler) // dipakai load balancer/orchestrator buat cek readiness
	mux.HandleFunc("/orders", ordersHandler)

	srv := &http.Server{
		Addr:    ":8080",
		Handler: mux,
	}

	go func() {
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatalf("server error: %v", err)
		}
	}()

	// Tangkap sinyal terminasi (dikirim orchestrator saat scale-in/deploy)
	// supaya instance sempat drain request yang sedang berjalan sebelum mati.
	stop := make(chan os.Signal, 1)
	signal.Notify(stop, syscall.SIGTERM, syscall.SIGINT)
	<-stop

	ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer cancel()
	if err := srv.Shutdown(ctx); err != nil {
		log.Printf("graceful shutdown gagal: %v", err)
	}
}

func healthzHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	w.Write([]byte("ok"))
}

func ordersHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
}
```

### Contoh Kode — Node.js
```javascript
const express = require('express');

const app = express();
app.get('/healthz', (req, res) => res.status(200).send('ok')); // dipakai load balancer/orchestrator buat cek readiness
app.get('/orders', (req, res) => res.status(200).json({ orders: [] }));

const server = app.listen(8080, () => {
  console.log('OrderFlow API listening on :8080');
});

// Tangkap SIGTERM (dikirim orchestrator saat scale-in/deploy) supaya instance
// sempat drain request yang sedang berjalan sebelum benar-benar mati.
process.on('SIGTERM', () => {
  server.close(() => {
    console.log('graceful shutdown selesai');
    process.exit(0);
  });

  setTimeout(() => {
    console.error('paksa keluar, graceful shutdown timeout');
    process.exit(1);
  }, 15000).unref();
});

module.exports = app;
```

### Trade-off & Pitfall
- Vertical scaling lebih simpel (gak perlu load balancer, gak perlu service stateless) tapi ada plafon kapasitas dan single point of failure — satu mesin down, seluruh service down.
- Horizontal scaling butuh investasi desain di awal (stateless service, shared storage buat session/cache/lock) yang gak dibutuhkan kalau cuma vertical scaling.
- Biaya horizontal scaling gak selalu linear — 10 instance kecil kadang lebih mahal atau lebih boros overhead (tiap instance butuh base memory buat runtime) dibanding satu instance besar untuk beban kerja yang sama.
- Auto-scaling horizontal butuh waktu (provisioning instance baru, health check jalan dulu) — kalau traffic spike-nya sangat mendadak, tetap perlu buffer kapasitas atau pre-scaling.

### Kapan Dipakai
Vertical scaling cocok untuk kebutuhan cepat dan sementara, atau workload yang memang sulit dipecah (database utama, misalnya). Horizontal scaling jadi pilihan default untuk API layer OrderFlow yang traffic-nya fluktuatif dan gak ada batas atas yang jelas, karena elastisitasnya jauh lebih baik untuk kebutuhan jangka panjang.

### Sering Ditanya Saat Interview
- "Kenapa horizontal scaling menuntut service stateless?" — karena load balancer bisa ngirim request ke instance manapun; kalau ada state yang cuma tersimpan di satu instance (misalnya di memory), request berikutnya yang mendarat di instance lain gak akan nemu state itu.
- "Apa downside murni nambah spec mesin (vertical scaling) buat OrderFlow?" — ada batas atas kapasitas mesin yang tersedia, biasanya butuh downtime restart pas resize, dan tetap single point of failure.
- "Gimana cara graceful shutdown membantu horizontal scaling?" — memastikan saat instance mau dimatikan (scale-in atau deploy baru), request yang sedang diproses tetap selesai dulu, gak langsung diputus di tengah jalan.

---

## 66. Stateless Service

### Apa itu?
Stateless service berarti instance API gak menyimpan data spesifik-user (session, cart, state request sebelumnya) di memory lokalnya sendiri. Semua informasi yang dibutuhkan buat memproses satu request harus datang dari request itu sendiri (misalnya JWT) atau dari storage bersama (database, Redis) yang bisa diakses instance manapun.

### Kenapa dibutuhkan?
Begitu OrderFlow di-scale horizontal jadi banyak instance di belakang load balancer, gak ada jaminan request dari user yang sama akan selalu mendarat di instance yang sama. Kalau ada state yang cuma hidup di memory satu instance, request berikutnya yang mendarat di instance lain gak akan nemu state itu — user kelihatan "logout" atau "keranjangnya hilang" secara acak.

### Cara Kerja
```
Request 1 (user X) -> Load Balancer -> Instance A
Request 2 (user X) -> Load Balancer -> Instance B  (bisa beda instance!)

Kalau state disimpan di memory Instance A saja:
  Instance B gak tau apa-apa soal request 1 -> data hilang/salah

Kalau state dibawa di JWT (topik 3, Phase 1) atau disimpan di storage bersama:
  Instance B baca ulang identitas dari JWT / query storage bersama -> konsisten
```

### Contoh Kode — Go
Versi **bermasalah** — cart disimpan di memory instance, gak bertahan lintas instance:
```go
package handler

import (
	"encoding/json"
	"net/http"
)

type CartItem struct {
	ProductID int64 `json:"product_id"`
	Qty       int   `json:"qty"`
}

// BUG: cart disimpan di map in-memory, per-instance. Begitu OrderFlow
// di-scale jadi beberapa instance di belakang load balancer, request user
// yang sama bisa mendarat di instance berbeda tiap kali, dan cart-nya
// "hilang" karena datanya cuma ada di instance pertama yang nampung.
var cartStoreBug = map[string][]CartItem{}

func AddToCartBug(w http.ResponseWriter, r *http.Request) {
	sessionID := r.Header.Get("X-Session-ID")
	var item CartItem
	if err := json.NewDecoder(r.Body).Decode(&item); err != nil {
		http.Error(w, "invalid body", http.StatusBadRequest)
		return
	}
	cartStoreBug[sessionID] = append(cartStoreBug[sessionID], item)
	w.WriteHeader(http.StatusOK)
}
```
Versi **fixed** — identitas dari JWT claims (`ValidateToken`, Phase 1), data cart di storage bersama:
```go
package handler

import (
	"context"
	"encoding/json"
	"net/http"

	"orderflow/internal/auth"
)

type CartItem struct {
	ProductID int64 `json:"product_id"`
	Qty       int   `json:"qty"`
}

type CartRepository interface {
	AddItem(ctx context.Context, userID int64, item CartItem) error
}

// AddToCart gak nyimpen apapun di memory instance ini. Identitas user
// datang dari JWT claims (ValidateToken, Phase 1) yang dikirim ulang di
// setiap request, dan cart-nya disimpan di storage bersama (CartRepository,
// backed Redis/DB) — jadi request bisa mendarat di instance manapun tanpa
// kehilangan data. Ini yang bikin API server OrderFlow stateless.
func AddToCart(cartRepo CartRepository) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		claims, ok := r.Context().Value(auth.ClaimsContextKey).(*auth.Claims)
		if !ok {
			http.Error(w, "unauthorized", http.StatusUnauthorized)
			return
		}

		var item CartItem
		if err := json.NewDecoder(r.Body).Decode(&item); err != nil {
			http.Error(w, "invalid body", http.StatusBadRequest)
			return
		}

		if err := cartRepo.AddItem(r.Context(), claims.UserID, item); err != nil {
			http.Error(w, "internal error", http.StatusInternalServerError)
			return
		}
		w.WriteHeader(http.StatusOK)
	}
}
```

### Contoh Kode — Node.js
Versi **bermasalah**:
```javascript
// BUG: cart disimpan di memory Node process (Map), per-instance. Kalau
// OrderFlow di-scale ke beberapa instance di belakang load balancer,
// request user yang sama bisa mendarat di proses lain dan cart-nya
// keliatan kosong.
const cartStoreBug = new Map();

function addToCartBug(req, res) {
  const sessionId = req.header('X-Session-ID');
  const items = cartStoreBug.get(sessionId) || [];
  items.push(req.body);
  cartStoreBug.set(sessionId, items);
  return res.sendStatus(200);
}

module.exports = { addToCartBug };
```
Versi **fixed** — identitas dari JWT claims (`validateToken`, Phase 1), data cart di storage bersama:
```javascript
// addToCart gak nyimpen apapun di memory instance ini. Identitas user
// datang dari JWT claims (validateToken, Phase 1) yang sudah ditempel
// middleware authenticate ke req.user, dan cart-nya disimpan di storage
// bersama (cartRepository, backed Redis/DB) — aman dipanggil di instance
// manapun. Ini yang bikin API server OrderFlow stateless.
function addToCart(cartRepository) {
  return async (req, res) => {
    const { productId, qty } = req.body;
    await cartRepository.addItem(req.user.userId, { productId, qty });
    return res.sendStatus(200);
  };
}

module.exports = { addToCart };
```

### Trade-off & Pitfall
- Stateless gak berarti "gak ada state sama sekali" — state tetap ada, cuma dipindah ke tempat yang bisa diakses semua instance (database, Redis, atau dibawa di token itu sendiri seperti JWT).
- JWT yang membawa semua claims di dalam token (stateless secara penuh) gak bisa langsung di-revoke sebelum expired, kecuali ada mekanisme tambahan seperti blocklist di Redis — trade-off ini sudah dibahas di Phase 1 topik 3 & 6.
- In-memory cache lokal (misalnya buat mempercepat baca data yang jarang berubah) masih boleh dipakai, asal bukan satu-satunya sumber data — kalau instance itu mati atau request pindah ke instance lain, data harus tetap bisa diambil ulang dari sumber lain.
- Sticky session (load balancer selalu ngirim user yang sama ke instance yang sama) kadang dipakai sebagai workaround, tapi itu justru mengurangi manfaat horizontal scaling (distribusi beban jadi gak rata) dan bikin instance yang "dipegang" satu user jadi single point of failure buat user itu.

### Kapan Dipakai
Wajib diterapkan di seluruh API layer OrderFlow yang direncanakan discale horizontal — praktis semua service publik-facing. Layer yang secara sengaja stateful (misalnya database itu sendiri, atau in-memory job scheduler yang memang didesain single-instance) adalah pengecualian yang harus didesain secara eksplisit, bukan kebetulan.

### Sering Ditanya Saat Interview
- "Gimana caramu memastikan sebuah endpoint yang baru ditambahkan tetap stateless?" — pastikan handler gak menyimpan data ke variabel/struct level package atau memory lokal proses; semua yang perlu "diingat" antar request harus lewat storage bersama atau token.
- "JWT itu stateless, tapi refresh token-nya disimpan di Redis — apa itu berarti melanggar prinsip stateless?" — enggak, karena Redis itu shared storage yang bisa diakses instance manapun, bukan memory lokal satu instance; yang dihindari adalah state yang cuma hidup di satu proses.
- "Apa risiko sticky session sebagai pengganti desain stateless?" — mengurangi manfaat load balancing (distribusi beban gak merata) dan bikin satu instance jadi titik kegagalan buat semua user yang "lengket" di situ.

---

## 67. Load Balancer

### Apa itu?
Load balancer adalah komponen yang menerima traffic masuk dan mendistribusikannya ke beberapa instance backend yang identik, berdasarkan algoritma tertentu (round robin, least connections, weighted, dst), supaya beban gak menumpuk di satu instance saja.

### Kenapa dibutuhkan?
Horizontal scaling (topik 65) gak ada gunanya kalau semua traffic tetap diarahkan ke satu instance saja — load balancer adalah komponen yang bikin banyak instance itu benar-benar berguna, sekaligus jadi titik yang bisa mendeteksi instance mana yang sehat (lewat health check) dan berhenti mengirim traffic ke instance yang lagi bermasalah.

### Cara Kerja
```
Client -> Load Balancer
             |-- cek health check tiap backend (/healthz)
             |-- pilih backend sehat berikutnya (round robin / least conn / dst)
             |
             v
   [ backend 1 ] [ backend 2 ] [ backend 3 ]  <- instance OrderFlow API yang identik
```
Kalau salah satu backend gagal health check berturut-turut, load balancer berhenti mengirim traffic ke situ sampai backend itu sehat lagi.

### Contoh Kode — Go
```go
package main

import (
	"log"
	"net/http"
	"net/http/httputil"
	"net/url"
	"sync/atomic"
)

// roundRobinBalancer mendistribusikan request ke beberapa instance backend
// OrderFlow secara bergantian (round robin) — pola paling sederhana sebelum
// masuk ke algoritma yang lebih canggih (least connections, weighted, dst).
type roundRobinBalancer struct {
	backends []*httputil.ReverseProxy
	counter  uint64
}

func newRoundRobinBalancer(backendURLs []string) *roundRobinBalancer {
	backends := make([]*httputil.ReverseProxy, 0, len(backendURLs))
	for _, raw := range backendURLs {
		target, err := url.Parse(raw)
		if err != nil {
			log.Fatalf("invalid backend url %s: %v", raw, err)
		}
		backends = append(backends, httputil.NewSingleHostReverseProxy(target))
	}
	return &roundRobinBalancer{backends: backends}
}

func (b *roundRobinBalancer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	n := atomic.AddUint64(&b.counter, 1)
	backend := b.backends[n%uint64(len(b.backends))]
	backend.ServeHTTP(w, r)
}

func main() {
	balancer := newRoundRobinBalancer([]string{
		"http://orderflow-api-1:8080",
		"http://orderflow-api-2:8080",
		"http://orderflow-api-3:8080",
	})
	log.Fatal(http.ListenAndServe(":80", balancer))
}
```

### Contoh Kode — Node.js
```javascript
const http = require('http');
const httpProxy = require('http-proxy');

// backends berisi alamat tiap instance OrderFlow API yang identik.
const backends = [
  'http://orderflow-api-1:8080',
  'http://orderflow-api-2:8080',
  'http://orderflow-api-3:8080',
];

const proxy = httpProxy.createProxyServer({});
let counter = 0;

// roundRobinBalancer mendistribusikan request ke backend secara bergantian.
const server = http.createServer((req, res) => {
  const target = backends[counter % backends.length];
  counter += 1;
  proxy.web(req, res, { target }, (err) => {
    console.error('proxy error:', err.message);
    res.writeHead(502);
    res.end('bad gateway');
  });
});

server.listen(80, () => {
  console.log('load balancer listening on :80');
});
```

### Trade-off & Pitfall
- Round robin sederhana gak memperhitungkan beban aktual tiap backend — kalau satu request jauh lebih berat dari yang lain, instance yang kebetulan dapat request berat itu bisa lebih terbebani walau jumlah request-nya sama rata. Least connections lebih adil untuk beban yang gak seragam.
- Load balancer sendiri bisa jadi single point of failure kalau cuma ada satu instance — di production biasanya dipasang lebih dari satu load balancer (atau pakai managed load balancer dari cloud provider) dengan DNS/VIP failover.
- Health check yang terlalu longgar (interval jarang, threshold gagal terlalu tinggi) bikin instance yang sudah bermasalah tetap dapat traffic lebih lama; terlalu ketat bisa false-positive men-drop instance yang sebenarnya cuma lambat sesaat (misalnya lagi garbage collection).
- Load balancer L7 (paham HTTP, bisa routing berdasarkan path/header) beda dengan L4 (cuma paham TCP/IP) — L7 lebih fleksibel tapi overhead-nya sedikit lebih besar.

### Kapan Dipakai
Wajib dipasang begitu OrderFlow API di-deploy lebih dari satu instance — tanpa load balancer, horizontal scaling gak ada gunanya karena traffic tetap harus diarahkan manual ke satu alamat. Di production biasanya dipakai managed load balancer (cloud provider) atau reverse proxy khusus (nginx, Envoy) daripada implementasi custom seperti contoh di atas, yang lebih cocok buat memahami konsepnya.

### Sering Ditanya Saat Interview
- "Apa beda load balancing L4 dan L7?" — L4 beroperasi di level TCP/UDP (cuma liat IP & port), L7 beroperasi di level HTTP (bisa liat path, header, cookie) sehingga bisa routing lebih pintar tapi overhead pemrosesannya lebih besar.
- "Kenapa health check penting buat load balancer?" — supaya traffic gak terus dikirim ke instance yang sudah gak sehat/crash, load balancer perlu tau instance mana yang masih bisa melayani request.
- "Apa risiko kalau load balancer cuma satu instance?" — jadi single point of failure; kalau load balancer itu down, semua traffic ke seluruh backend ikut terputus walau backend-nya sendiri sehat.

---

## 68. CAP Theorem

### Apa itu?
CAP Theorem menyatakan bahwa sebuah sistem terdistribusi cuma bisa menjamin dua dari tiga properti berikut secara bersamaan saat terjadi network partition: **Consistency** (semua node melihat data yang sama persis di waktu yang sama), **Availability** (setiap request selalu dapat response, walau ada node yang gak bisa dihubungi), dan **Partition Tolerance** (sistem tetap jalan walau ada gangguan komunikasi antar node).

### Kenapa dibutuhkan?
OrderFlow yang di-deploy di banyak region/availability zone otomatis rentan network partition (koneksi antar data center putus sesaat) — itu fakta infrastruktur yang gak bisa dihindari di sistem terdistribusi manapun. Karena Partition Tolerance praktis wajib dipilih (kalau gak, sistem cuma bisa jalan di satu node, bukan terdistribusi lagi namanya), pilihan sebenarnya ada di antara Consistency dan Availability saat partition terjadi — dan itu keputusan arsitektur yang harus disadari, bukan kebetulan.

### Cara Kerja
```
Normal (tanpa partition): semua node bisa saling komunikasi, C dan A bisa dua-duanya terjaga.

Saat network partition terjadi (node A gak bisa ngobrol ke node B):

Pilih CP (Consistency + Partition Tolerance):
  Node yang gak bisa konfirmasi ke node lain -> TOLAK request (gak tersedia sementara)
  -> data tetap konsisten, tapi availability turun

Pilih AP (Availability + Partition Tolerance):
  Setiap node tetap layani request pakai data yang dia punya saat itu
  -> availability terjaga, tapi node yang partition bisa punya data beda (stale)
     sampai partition-nya sembuh dan data disinkronkan ulang (topik 69)
```

### Trade-off & Pitfall
- Memilih CP berarti sebagian user bisa dapat error/unavailable saat partition, demi memastikan gak ada yang baca data yang salah/stale — cocok untuk data finansial seperti saldo pembayaran OrderFlow.
- Memilih AP berarti sistem tetap responsif walau ada partition, tapi harus siap menangani data yang sempat gak konsisten antar node dan proses rekonsiliasi setelahnya (eventual consistency, topik 69) — cocok untuk data yang toleran stale sebentar, seperti jumlah stock yang ditampilkan di listing produk.
- CAP Theorem sering disalahpahami sebagai "harus pilih 2 dari 3 selamanya" — padahal trade-off-nya cuma relevan **saat partition terjadi**; di kondisi normal, sistem bisa menjaga C dan A dua-duanya.
- Keputusan CP vs AP gak harus seragam untuk seluruh sistem OrderFlow — data pembayaran bisa didesain condong ke CP, sementara data katalog produk condong ke AP, dalam satu arsitektur yang sama.

### Kapan Dipakai
Relevan setiap kali mendesain sistem yang datanya direplikasi/di-partition lintas node atau region — misalnya memutuskan strategi replikasi database OrderFlow, atau memilih behavior sistem saat data center di satu region gak bisa dihubungi sementara.

### Sering Ditanya Saat Interview
- "Kenapa Partition Tolerance biasanya dianggap wajib, bukan pilihan?" — karena network partition adalah fakta yang gak bisa sepenuhnya dihindari di sistem terdistribusi manapun; sistem yang gak toleran partition sama sekali berarti cuma jalan di satu node, bukan sistem terdistribusi.
- "Beri contoh bagian OrderFlow yang lebih cocok CP, dan bagian yang lebih cocok AP." — proses pembayaran/saldo lebih cocok CP (gak boleh baca saldo yang salah), sementara jumlah stock yang ditampilkan di halaman listing produk lebih cocok AP (boleh sedikit stale demi tetap responsif).
- "Apa hubungan CAP Theorem dengan eventual consistency?" — eventual consistency adalah salah satu cara mengimplementasikan pilihan AP: sistem tetap available walau data antar node sempat berbeda, dengan jaminan bahwa data itu akan konvergen/konsisten lagi setelah beberapa saat.

---

## 69. Eventual Consistency

### Apa itu?
Eventual consistency adalah model di mana update ke satu bagian sistem gak langsung terlihat di semua bagian lain secara instan, tapi dijamin akan "menyusul" dan konsisten setelah beberapa saat, tanpa update baru yang masuk.

### Kenapa dibutuhkan?
Data source of truth (misalnya jumlah stock di database order utama) butuh strong consistency, tapi read-model yang dipakai buat menampilkan listing produk ke user gak harus selalu real-time detik itu juga — kalau harus selalu strong consistent, setiap baca listing produk butuh query langsung ke database utama yang sama, yang gampang jadi bottleneck saat traffic tinggi. Eventual consistency membolehkan update itu disebarkan secara asynchronous (lewat message queue, Phase 5) ke cache/read-model, sehingga baca jadi cepat dan skalabel, dengan trade-off ada window waktu singkat di mana datanya "sedikit telat".

### Cara Kerja
```
1. Order dibuat -> disimpan ke database utama (source of truth, strong consistency)
2. Handler publish event "order.placed" ke message queue (async, gak nunggu selesai)
3. Consumer terpisah membaca event itu, update stock count di cache/read-model
4. Selama consumer belum selesai memproses (biasanya cuma hitungan milidetik-detik),
   listing produk yang baca dari cache masih nunjukin stock lama
5. Setelah consumer selesai -> cache sudah konsisten dengan database utama
```

### Contoh Kode — Go
```go
package handler

import (
	"context"
	"encoding/json"
	"net/http"

	"orderflow/internal/queue"
)

type OrderPlacedEvent struct {
	OrderID   int64 `json:"order_id"`
	ProductID int64 `json:"product_id"`
	Qty       int   `json:"qty"`
}

// CreateOrder nyimpen order ke database utama (source of truth, strong
// consistency) lalu publish event ke queue supaya stock count di
// read-model (cache produk yang dipakai listing) di-update belakangan
// secara asynchronous — itu sebabnya listing produk bisa kelihatan stock
// yang sedikit "telat" beberapa saat setelah order dibuat.
func CreateOrder(publisher queue.Publisher) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		var req struct {
			ProductID int64 `json:"product_id"`
			Qty       int   `json:"qty"`
		}
		if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
			http.Error(w, "invalid body", http.StatusBadRequest)
			return
		}

		orderID, err := saveOrder(r.Context(), req.ProductID, req.Qty)
		if err != nil {
			http.Error(w, "internal error", http.StatusInternalServerError)
			return
		}

		event, _ := json.Marshal(OrderPlacedEvent{OrderID: orderID, ProductID: req.ProductID, Qty: req.Qty})
		if err := publisher.Publish(r.Context(), "order.placed", event); err != nil {
			// order sudah tersimpan di source of truth; kegagalan publish
			// event gak boleh menggagalkan response ke user, tapi harus
			// dicatat/di-retry supaya read-model gak keburu ketinggalan lama.
			logPublishFailure(err)
		}

		w.WriteHeader(http.StatusCreated)
		json.NewEncoder(w).Encode(map[string]int64{"order_id": orderID})
	}
}

func saveOrder(ctx context.Context, productID int64, qty int) (int64, error) {
	// simpan ke database utama (strong consistency)
	return 1, nil
}

func logPublishFailure(err error) {}
```

### Contoh Kode — Node.js
```javascript
// stockCacheConsumer men-subscribe event "order.placed" dan meng-update
// stock count di cache produk (read-model yang dipakai listing) secara
// async. Ada window waktu singkat di mana listing produk masih nunjukin
// stock lama sebelum consumer ini selesai memproses event — itu yang
// disebut eventual consistency.
function stockCacheConsumer(queueClient, productCache) {
  queueClient.subscribe('order.placed', async (message) => {
    const event = JSON.parse(message.content.toString());
    const current = await productCache.getStock(event.productId);
    await productCache.setStock(event.productId, current - event.qty);
  });
}

module.exports = { stockCacheConsumer };
```

### Trade-off & Pitfall
- Window ketidakkonsistenan biasanya cuma milidetik sampai beberapa detik (tergantung antrian consumer), tapi kalau consumer down atau lag, window itu bisa membesar — perlu monitoring lag queue (topik 51, Phase 5) supaya "eventual" gak jadi "gak pernah".
- Jangan pakai eventual consistency buat data yang butuh kebenaran instan dan gak toleran salah baca — misalnya validasi stock final sebelum charge pembayaran tetap harus baca dari source of truth langsung, bukan dari cache yang eventually consistent.
- User bisa bingung kalau order yang baru saja dibuat belum langsung muncul di halaman "riwayat order" yang baca dari read-model berbeda — perlu UX yang mengakomodasi ini (misalnya optimistic UI di frontend) atau pastikan endpoint yang critical tetap baca dari source of truth.
- Kegagalan publish event (network error ke message broker) harus ditangani — kalau dibiarkan silent, read-model bisa permanen gak sinkron, bukan cuma "eventually" konsisten.

### Kapan Dipakai
Cocok untuk data read-heavy yang toleran stale beberapa saat — listing produk, dashboard analytics, notifikasi. Untuk data yang butuh kebenaran mutlak di detik itu juga (validasi saldo sebelum charge, cek stock final sebelum konfirmasi order), tetap baca langsung dari source of truth dengan strong consistency.

### Sering Ditanya Saat Interview
- "Apa beda strong consistency dan eventual consistency?" — strong consistency menjamin semua baca selalu dapat data terbaru begitu ada update selesai; eventual consistency membolehkan ada jeda waktu di mana data yang dibaca masih versi lama, dengan jaminan akan konvergen ke data terbaru setelah beberapa saat.
- "Gimana caramu tau seberapa 'telat' sebuah sistem eventual consistency di production?" — monitor consumer lag di message queue (topik 51) — semakin besar lag, semakin lama window ketidakkonsistenannya.
- "Kenapa gak semua bagian OrderFlow pakai eventual consistency biar semuanya cepat?" — karena ada data yang kesalahannya mahal kalau dibaca stale (saldo, stock final saat checkout) — di situ correctness lebih penting daripada kecepatan baca.

---

## 70. Distributed Lock

### Apa itu?
Distributed lock adalah mekanisme memastikan cuma satu proses/instance yang boleh menjalankan operasi tertentu di waktu yang sama, walau operasi itu dipicu dari banyak instance sekaligus — diimplementasikan lewat storage bersama (Redis) yang semua instance bisa akses, bukan lock lokal in-memory yang cuma berlaku di satu proses.

### Kenapa dibutuhkan?
OrderFlow punya job terjadwal buat stock-reconciliation (mencocokkan stock di database dengan data dari warehouse) yang harus jalan tiap beberapa menit. Karena OrderFlow di-deploy di banyak instance/pod (horizontal scaling, topik 65) dan tiap instance punya scheduler-nya sendiri, tanpa distributed lock, job yang sama bisa jalan bersamaan di beberapa instance sekaligus — menyebabkan double-processing dan race condition di data stock.

### Cara Kerja
```
Instance A               Instance B
    |                         |
    |-- SET lock:job token NX EX ttl -->|   (cuma satu yang berhasil, karena NX)
    |     berhasil (bukan ada)          |     gagal (key sudah ada)
    |                                   |
    v                                   v
  jalankan job                    skip, job lagi dipegang instance lain
    |
    v
  selesai -> DEL lock:job (HANYA kalau value-nya masih token milik instance A --
             mencegah instance A gak sengaja menghapus lock instance lain kalau
             lock-nya sendiri sudah expired duluan dan diambil ulang instance C)
```

### Contoh Kode — Go
```go
package lock

import (
	"context"
	"errors"
	"sync"
	"time"

	"github.com/google/uuid"
	"github.com/redis/go-redis/v9"
)

// ErrLockNotHeld dibalikin kalau instance ini mencoba release lock yang
// bukan (atau sudah bukan) miliknya.
var ErrLockNotHeld = errors.New("lock not held by this instance")

// releaseScript menghapus key HANYA kalau value-nya masih sama dengan token
// yang kita simpan waktu acquire — mencegah instance ini gak sengaja
// menghapus lock yang sebenarnya sudah dipegang instance lain (karena lock
// milik instance ini sendiri sudah expired duluan dan diambil ulang oleh
// instance lain).
var releaseScript = redis.NewScript(`
if redis.call("GET", KEYS[1]) == ARGV[1] then
	return redis.call("DEL", KEYS[1])
else
	return 0
end
`)

var (
	mu     sync.Mutex
	tokens = map[string]string{} // key -> token yang instance ini pegang
)

// AcquireDistributedLock mencoba mengambil lock terdistribusi lewat Redis
// pakai SET key value NX EX ttl — cuma satu instance yang bisa berhasil SET
// selama key itu belum expired/dihapus. Dipakai supaya cuma satu instance
// OrderFlow yang menjalankan job stock-reconciliation di waktu yang sama,
// walau scheduler-nya jalan di semua instance/pod sekaligus.
func AcquireDistributedLock(ctx context.Context, rdb *redis.Client, key string, ttl time.Duration) (bool, error) {
	token := uuid.NewString()
	ok, err := rdb.SetNX(ctx, key, token, ttl).Result()
	if err != nil {
		return false, err
	}
	if !ok {
		return false, nil
	}

	mu.Lock()
	tokens[key] = token
	mu.Unlock()
	return true, nil
}

// ReleaseDistributedLock melepas lock, tapi HANYA kalau token yang disimpan
// di Redis masih sama dengan token yang kita pegang. Kalau lock sudah
// expired dan diambil instance lain, ini TIDAK akan menghapus lock milik
// instance lain itu.
func ReleaseDistributedLock(ctx context.Context, rdb *redis.Client, key string) error {
	mu.Lock()
	token, held := tokens[key]
	mu.Unlock()
	if !held {
		return ErrLockNotHeld
	}

	res, err := releaseScript.Run(ctx, rdb, []string{key}, token).Int()
	if err != nil {
		return err
	}

	mu.Lock()
	delete(tokens, key)
	mu.Unlock()

	if res == 0 {
		return ErrLockNotHeld
	}
	return nil
}
```
Pemakaian — job stock-reconciliation cuma jalan di satu instance:
```go
func RunStockReconciliationJob(ctx context.Context, rdb *redis.Client) {
	const lockKey = "lock:stock-reconciliation"

	ok, err := AcquireDistributedLock(ctx, rdb, lockKey, 5*time.Minute)
	if err != nil {
		log.Printf("gagal cek lock: %v", err)
		return
	}
	if !ok {
		log.Println("instance lain lagi menjalankan stock reconciliation, skip")
		return
	}
	defer func() {
		if err := ReleaseDistributedLock(ctx, rdb, lockKey); err != nil {
			log.Printf("gagal release lock: %v", err)
		}
	}()

	reconcileStock(ctx)
}
```

### Contoh Kode — Node.js
```javascript
const { randomUUID } = require('crypto');

const heldTokens = new Map(); // key -> token yang instance ini pegang

// RELEASE_SCRIPT menghapus key HANYA kalau value-nya masih sama dengan
// token yang kita simpan waktu acquire — mencegah instance ini gak
// sengaja menghapus lock yang sebenarnya sudah dipegang instance lain.
const RELEASE_SCRIPT = `
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
else
  return 0
end
`;

// acquireDistributedLock mencoba mengambil lock terdistribusi lewat Redis
// pakai SET key value PX ttlMs NX — cuma satu instance yang bisa berhasil
// SET selama key itu belum expired/dihapus. Dipakai supaya cuma satu
// instance OrderFlow yang menjalankan job stock-reconciliation di waktu
// yang sama.
async function acquireDistributedLock(redisClient, key, ttlMs) {
  const token = randomUUID();
  const result = await redisClient.set(key, token, 'PX', ttlMs, 'NX');
  if (result !== 'OK') {
    return false;
  }
  heldTokens.set(key, token);
  return true;
}

// releaseDistributedLock melepas lock, tapi HANYA kalau token yang disimpan
// di Redis masih sama dengan token yang kita pegang. Kalau lock sudah
// expired dan diambil instance lain, ini TIDAK akan menghapus lock milik
// instance lain itu.
async function releaseDistributedLock(redisClient, key) {
  const token = heldTokens.get(key);
  if (!token) {
    return false;
  }

  const result = await redisClient.eval(RELEASE_SCRIPT, 1, key, token);
  heldTokens.delete(key);
  return result === 1;
}

module.exports = { acquireDistributedLock, releaseDistributedLock };
```
Pemakaian — job stock-reconciliation cuma jalan di satu instance:
```javascript
const { acquireDistributedLock, releaseDistributedLock } = require('./distributedLock');

async function runStockReconciliationJob(redisClient) {
  const lockKey = 'lock:stock-reconciliation';
  const acquired = await acquireDistributedLock(redisClient, lockKey, 5 * 60 * 1000);
  if (!acquired) {
    console.log('instance lain lagi menjalankan stock reconciliation, skip');
    return;
  }

  try {
    await reconcileStock();
  } finally {
    await releaseDistributedLock(redisClient, lockKey);
  }
}

module.exports = { runStockReconciliationJob };
```

### Trade-off & Pitfall
- Kesalahan paling umum: release lock cuma dengan `DEL key` tanpa cek dulu apakah value-nya masih milik kita. Kalau job berjalan lebih lama dari TTL, lock bisa keburu expired dan diambil instance lain — `DEL` polos dari instance pertama justru menghapus lock yang sekarang dipegang instance kedua. Solusinya CAS (compare-and-delete) lewat token unik seperti contoh di atas, bukan `DEL` langsung.
- TTL harus disetel lebih panjang dari estimasi waktu job selesai (dengan buffer), tapi gak terlalu panjang — kalau instance yang pegang lock crash sebelum sempat release, lock itu bakal "nyangkut" sampai TTL habis; TTL adalah mekanisme recovery-nya.
- Implementasi single-Redis-instance seperti di atas tetap punya satu titik kegagalan (kalau Redis itu down/failover, lock bisa hilang atau diambil dua instance sekaligus untuk sesaat) — buat kebutuhan yang lebih kritikal, algoritma Redlock (quorum dari beberapa Redis instance independen) mengurangi risiko ini, dengan kompleksitas operasional lebih tinggi.
- Distributed lock cocok buat mencegah kerja duplikat (seperti job scheduler), tapi jangan dipakai sebagai pengganti transaksi database yang butuh atomicity kuat pada level data — itu ranahnya database transaction/distributed transaction (topik 71).

### Kapan Dipakai
Dipakai saat OrderFlow perlu memastikan cuma satu eksekusi yang jalan di satu waktu untuk operasi yang di-trigger dari banyak instance — job terjadwal (stock reconciliation, generate laporan harian), atau proses yang gak aman dijalankan concurrent walau secara logic idempotent (menghindari kerja ganda yang sia-sia meski gak merusak data).

### Sering Ditanya Saat Interview
- "Kenapa gak boleh langsung `DEL` lock waktu release?" — kalau lock sudah expired duluan (job lebih lama dari TTL) dan diambil instance lain, `DEL` polos bakal menghapus lock milik instance lain itu, bukan lock kita — harus cek dulu token-nya masih milik kita (CAS) sebelum hapus.
- "Kenapa acquire lock pakai SET NX dengan TTL, bukan cuma NX tanpa TTL?" — TTL adalah mekanisme recovery kalau instance yang pegang lock crash sebelum sempat release; tanpa TTL, lock bisa nyangkut selamanya dan gak ada instance lain yang bisa jalanin job itu lagi.
- "Apa keterbatasan distributed lock berbasis satu Redis instance, dan gimana Redlock mengatasinya?" — satu Redis instance adalah single point of failure (bisa gagal/failover di waktu yang gak tepat); Redlock mengharuskan lock berhasil di-acquire di mayoritas (quorum) dari beberapa Redis instance independen, sehingga kegagalan satu instance gak langsung merusak jaminan lock.

---

## 71. Distributed Transaction

### Apa itu?
Distributed transaction adalah operasi yang melibatkan perubahan data di lebih dari satu service/database berbeda, tapi tetap harus terlihat "semua berhasil atau semua gagal" secara keseluruhan — walau gak bisa dibungkus satu database transaction biasa karena masing-masing service punya database sendiri.

### Kenapa dibutuhkan?
Proses `POST /orders` di OrderFlow menyentuh setidaknya tiga hal: order service (bikin order), inventory service (reservasi stock), dan payment service (charge pembayaran) — masing-masing punya database sendiri di arsitektur microservices. Kalau payment gagal setelah stock sudah direservasi, stock itu harus dilepas lagi, dan order yang sudah dibuat harus dibatalkan — tanpa mekanisme yang jelas, sistem bisa berakhir dengan stock yang "nyangkut" ke-reserve padahal order-nya gagal.

### Cara Kerja
```
Saga pattern (orchestrator-based):
  1. CreateOrder    -> sukses
  2. ReserveStock   -> sukses
  3. ChargePayment  -> GAGAL
       -> compensate step 2: ReleaseStock
       -> compensate step 1: CancelOrder

Setiap step yang sudah sukses harus punya "compensating action" (aksi pembalik)
yang dijalankan mundur kalau ada step setelahnya yang gagal -- bukan rollback
otomatis seperti transaksi database biasa, karena tiap service database-nya terpisah.
```

### Contoh Kode — Go
```go
package saga

import (
	"context"
	"fmt"
)

type CreateOrderRequest struct {
	UserID    int64
	ProductID int64
	Qty       int
	Amount    float64
}

type OrderService interface {
	CreateOrder(ctx context.Context, req CreateOrderRequest) (int64, error)
	CancelOrder(ctx context.Context, orderID int64) error
	MarkPaid(ctx context.Context, orderID int64) error
}

type InventoryService interface {
	ReserveStock(ctx context.Context, productID int64, qty int) error
	ReleaseStock(ctx context.Context, productID int64, qty int) error
}

type PaymentService interface {
	Charge(ctx context.Context, userID int64, amount float64) error
}

// CreateOrderSaga menjalankan proses order sebagai serangkaian step lintas
// service (order, inventory, payment) yang gak bisa dibungkus satu database
// transaction karena masing-masing punya database sendiri. Kalau salah satu
// step gagal, saga menjalankan compensating action buat step-step
// sebelumnya yang sudah sukses — bukan rollback otomatis seperti transaksi
// database biasa.
type CreateOrderSaga struct {
	OrderService     OrderService
	InventoryService InventoryService
	PaymentService   PaymentService
}

func (s *CreateOrderSaga) Run(ctx context.Context, req CreateOrderRequest) (int64, error) {
	orderID, err := s.OrderService.CreateOrder(ctx, req)
	if err != nil {
		return 0, fmt.Errorf("create order gagal: %w", err)
	}

	if err := s.InventoryService.ReserveStock(ctx, req.ProductID, req.Qty); err != nil {
		// compensate: order yang sudah terlanjur dibuat harus dibatalkan
		s.OrderService.CancelOrder(ctx, orderID)
		return 0, fmt.Errorf("reserve stock gagal, order dibatalkan: %w", err)
	}

	if err := s.PaymentService.Charge(ctx, req.UserID, req.Amount); err != nil {
		// compensate: lepas reservasi stock, lalu batalkan order
		s.InventoryService.ReleaseStock(ctx, req.ProductID, req.Qty)
		s.OrderService.CancelOrder(ctx, orderID)
		return 0, fmt.Errorf("payment gagal, stock & order dibatalkan: %w", err)
	}

	s.OrderService.MarkPaid(ctx, orderID)
	return orderID, nil
}
```

### Contoh Kode — Node.js
```javascript
// createOrderSaga menjalankan proses order sebagai serangkaian step lintas
// service (order, inventory, payment) yang masing-masing punya database
// sendiri. Kalau salah satu step gagal, saga menjalankan compensating
// action buat step-step sebelumnya yang sudah sukses.
async function createOrderSaga({ orderService, inventoryService, paymentService }, req) {
  const orderId = await orderService.createOrder(req);

  try {
    await inventoryService.reserveStock(req.productId, req.qty);
  } catch (err) {
    await orderService.cancelOrder(orderId);
    throw new Error(`reserve stock gagal, order dibatalkan: ${err.message}`);
  }

  try {
    await paymentService.charge(req.userId, req.amount);
  } catch (err) {
    await inventoryService.releaseStock(req.productId, req.qty);
    await orderService.cancelOrder(orderId);
    throw new Error(`payment gagal, stock & order dibatalkan: ${err.message}`);
  }

  await orderService.markPaid(orderId);
  return orderId;
}

module.exports = { createOrderSaga };
```

### Trade-off & Pitfall
- Saga gak punya isolasi seketat transaksi database (ACID) — ada window waktu di mana order sudah "ada" tapi belum lunas, atau stock sudah direservasi tapi order-nya nanti dibatalkan; service lain yang baca data di window itu harus siap dengan state transisi ini.
- Compensating action harus idempotent dan bisa gagal juga — kalau `ReleaseStock` gagal saat compensating, butuh retry/dead letter queue (Phase 5) supaya stock gak permanen "nyangkut" ke-reserve.
- Alternatif two-phase commit (2PC) menjamin atomicity lebih ketat, tapi butuh semua service lock resource-nya sampai koordinator memutuskan commit/abort — ini bikin sistem kurang available (kalau koordinator down, semua node yang lagi nunggu jadi ke-block) dan gak umum dipakai lintas microservices modern; saga lebih disukai karena lebih available walau lebih kompleks secara logic compensating.
- Orchestrator-based saga (seperti contoh di atas, satu service yang mengatur urutan step) lebih gampang di-trace tapi bikin service itu tau terlalu banyak soal service lain; choreography-based saga (tiap service dengar event dan bereaksi sendiri, lewat message queue) lebih terdesentralisasi tapi lebih susah dilacak alurnya.

### Kapan Dipakai
Dipakai setiap kali satu proses bisnis OrderFlow menyentuh lebih dari satu service dengan database terpisah dan butuh jaminan "semua berhasil atau semua di-compensate" — checkout order (order + inventory + payment) adalah contoh paling jelas. Untuk operasi yang cuma menyentuh satu database, transaksi database biasa (BEGIN/COMMIT/ROLLBACK) sudah cukup dan jauh lebih sederhana.

### Sering Ditanya Saat Interview
- "Kenapa gak pakai transaksi database biasa buat proses checkout yang melibatkan order, inventory, dan payment service?" — karena masing-masing service punya database sendiri di arsitektur microservices; gak ada satu koneksi database yang bisa membungkus ketiganya dalam satu BEGIN/COMMIT.
- "Apa itu compensating transaction?" — aksi pembalik yang meniadakan efek dari step yang sudah sukses sebelumnya, dijalankan kalau ada step setelahnya yang gagal — misalnya `ReleaseStock` buat membalikkan efek `ReserveStock`.
- "Apa beda orchestration-based saga dan choreography-based saga?" — orchestration punya satu koordinator yang secara eksplisit memanggil tiap service secara berurutan dan tau seluruh alurnya; choreography gak punya koordinator terpusat, tiap service bereaksi terhadap event dari service lain, alurnya jadi lebih terdesentralisasi tapi lebih susah dilacak.

---

## 72. Service-to-Service Communication (REST vs gRPC vs GraphQL)

### Apa itu?
Ini soal bagaimana service-service internal OrderFlow (order service, inventory service, payment service, dst) saling memanggil satu sama lain. Tiga pendekatan paling umum: REST (HTTP + JSON, resource-oriented), gRPC (HTTP/2 + Protocol Buffers biner, kontrak service didefinisikan lewat file `.proto`), dan GraphQL (satu endpoint, client yang menentukan field apa saja yang mau diambil lewat query).

### Kenapa dibutuhkan?
Komunikasi service-to-service (internal, antar microservice) punya karakteristik beda dengan komunikasi client-to-server (eksternal, dari browser/mobile app) — biasanya frekuensinya jauh lebih tinggi, latency-nya lebih kritikal, dan yang manggil maupun yang dipanggil sama-sama backend yang bisa dikontrol kontraknya. Pilihan protokol yang salah bisa berarti overhead serialisasi yang gak perlu (JSON di REST lebih besar & lebih lambat di-parse dibanding protobuf biner di gRPC) atau over-fetching data (REST kadang balikin field yang gak dipakai; GraphQL mengatasi ini dengan field selection).

### Cara Kerja
```
REST:    Order Service --HTTP GET /stock/42 (JSON)--> Inventory Service
gRPC:    Order Service --HTTP/2 GetStock(productId=42) (protobuf biner)--> Inventory Service
GraphQL: Client --query { order(id: 1) { id status items { name } } }--> GraphQL Gateway
                                                          --> resolver manggil service terkait
```
gRPC pakai skema `.proto` yang men-generate client & server stub di kedua sisi, jadi kontraknya strict-typed dan divalidasi saat compile time, beda dengan REST/JSON yang skemanya biasanya cuma dokumentasi (OpenAPI, topik 12) tanpa enforcement compile-time.

### Contoh Kode — Go
```go
package client

import (
	"context"
	"time"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"

	inventorypb "orderflow/proto/inventory"
)

// NewInventoryClient bikin koneksi gRPC ke Inventory Service — dipakai
// Order Service buat cek stock secara internal (service-to-service), bukan
// lewat REST publik yang overhead-nya lebih besar (JSON text-based) dibanding
// gRPC yang pakai protobuf biner di atas HTTP/2.
func NewInventoryClient(addr string) (inventorypb.InventoryServiceClient, error) {
	conn, err := grpc.NewClient(addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		return nil, err
	}
	return inventorypb.NewInventoryServiceClient(conn), nil
}

// CheckStock manggil Inventory Service lewat gRPC, dengan timeout ketat
// karena ini dipanggil sinkron di dalam request path pembuatan order.
func CheckStock(client inventorypb.InventoryServiceClient, productID int64) (int32, error) {
	ctx, cancel := context.WithTimeout(context.Background(), 500*time.Millisecond)
	defer cancel()

	resp, err := client.GetStock(ctx, &inventorypb.GetStockRequest{ProductId: productID})
	if err != nil {
		return 0, err
	}
	return resp.Quantity, nil
}
```

### Contoh Kode — Node.js
```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');

const packageDefinition = protoLoader.loadSync('inventory.proto', {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
});
const inventoryProto = grpc.loadPackageDefinition(packageDefinition).inventory;

// createInventoryClient bikin koneksi gRPC ke Inventory Service, dipanggil
// Order Service secara internal (service-to-service) — lebih efisien
// dibanding REST buat komunikasi internal yang frequent dan latency-sensitive.
function createInventoryClient(addr) {
  return new inventoryProto.InventoryService(addr, grpc.credentials.createInsecure());
}

function checkStock(client, productId) {
  return new Promise((resolve, reject) => {
    const deadline = new Date(Date.now() + 500);
    client.getStock({ product_id: productId }, { deadline }, (err, response) => {
      if (err) return reject(err);
      return resolve(response.quantity);
    });
  });
}

module.exports = { createInventoryClient, checkStock };
```

### Trade-off & Pitfall
- gRPC butuh setup lebih rumit (compile `.proto` jadi stub, tooling di kedua sisi harus konsisten versinya) dibanding REST yang cukup HTTP client biasa — worth it untuk komunikasi internal frequent, tapi overkill buat integrasi eksternal/partner yang kliennya gak selalu bisa pakai tooling gRPC.
- GraphQL bagus buat client eksternal yang butuh fleksibilitas milih field (menghindari over-fetching/under-fetching), tapi kurang cocok buat komunikasi internal service-to-service yang polanya biasanya sudah tetap dan gak butuh fleksibilitas query dinamis.
- REST tetap paling universal — didukung semua bahasa/tooling tanpa setup tambahan, gampang di-debug pakai curl/browser — trade-off-nya payload JSON lebih besar dan skema gak di-enforce compile-time seperti gRPC.
- gRPC pakai HTTP/2 yang butuh dukungan infrastruktur (load balancer, proxy) yang paham HTTP/2 dengan benar — beberapa load balancer/proxy lama cuma jago di HTTP/1.1 dan butuh konfigurasi khusus buat gRPC.

### Kapan Dipakai
gRPC cocok untuk komunikasi internal antar service OrderFlow yang frequent dan latency-sensitive (Order Service ke Inventory Service). REST cocok untuk API publik yang dikonsumsi banyak jenis client berbeda dan butuh kemudahan debugging/dokumentasi luas. GraphQL cocok kalau ada kebutuhan spesifik client (misalnya mobile app yang mau meminimalkan jumlah round-trip dan cuma ambil field yang dibutuhkan) — bukan pengganti default REST/gRPC untuk semua kasus.

### Sering Ditanya Saat Interview
- "Kenapa gRPC lebih cepat dibanding REST/JSON buat komunikasi internal?" — protobuf itu format biner yang lebih ringkas dan lebih cepat di-serialize/deserialize dibanding JSON teks, dan gRPC jalan di atas HTTP/2 yang mendukung multiplexing (banyak request dalam satu koneksi).
- "Kapan GraphQL lebih masuk akal daripada REST?" — ketika client (terutama mobile, yang sensitif ke jumlah round-trip dan ukuran payload) butuh mengambil data dari banyak resource berbeda dalam satu request, dengan kontrol field mana saja yang diambil.
- "Apa risiko utama pakai gRPC buat API publik yang dikonsumsi partner eksternal?" — partner harus punya tooling buat compile `.proto` dan generate client stub di bahasa mereka, yang lebih ribet dibanding REST yang cukup HTTP client biasa; banyak partner eksternal lebih familiar dan lebih gampang integrasi lewat REST.

---

## 73. API Gateway

### Apa itu?
API Gateway adalah satu titik masuk (single entry point) buat semua request eksternal ke arsitektur microservices OrderFlow, yang meneruskan tiap request ke service internal yang sesuai, sekaligus jadi tempat yang tepat buat menangani concern lintas-service seperti authentication, rate limiting, dan logging secara terpusat.

### Kenapa dibutuhkan?
Tanpa API Gateway, client eksternal (mobile app, partner) harus tau alamat internal tiap microservice (order service, product service, payment service) satu-satu, dan tiap service harus mengimplementasikan sendiri authentication, rate limiting, dst — duplikasi logic yang gampang jadi gak konsisten (ingat topik 13: satu endpoint yang kelupaan di-protect auth adalah sumber breach paling umum). API Gateway menyentralisasi semua itu di satu tempat.

### Cara Kerja
```
Client --> API Gateway (satu alamat publik)
             |-- validasi JWT (ValidateToken, Phase 1) sekali di sini
             |-- rate limiting (topik 15, Phase 2) sekali di sini
             |-- routing berdasarkan path prefix:
             |     /orders/*   -> Order Service (internal, gak exposed langsung ke publik)
             |     /products/* -> Product Service
             |     /payments/* -> Payment Service
```

### Contoh Kode — Go
```go
package main

import (
	"log"
	"net/http"
	"net/http/httputil"
	"net/url"

	"orderflow/internal/auth"
)

// NewGateway jadi satu pintu masuk buat semua request eksternal, meneruskan
// ke microservice internal yang sesuai berdasarkan path prefix. Auth
// (ValidateToken, Phase 1) divalidasi sekali di sini, bukan diulang-ulang
// di tiap service di belakangnya.
func NewGateway(orderServiceURL, productServiceURL, paymentServiceURL string) http.Handler {
	mux := http.NewServeMux()

	mux.Handle("/orders/", auth.Middleware(reverseProxyTo(orderServiceURL)))
	mux.Handle("/products/", reverseProxyTo(productServiceURL)) // publik, gak butuh auth
	mux.Handle("/payments/", auth.Middleware(reverseProxyTo(paymentServiceURL)))

	return mux
}

func reverseProxyTo(rawURL string) http.Handler {
	target, err := url.Parse(rawURL)
	if err != nil {
		log.Fatalf("invalid service url %s: %v", rawURL, err)
	}
	return httputil.NewSingleHostReverseProxy(target)
}

func main() {
	gateway := NewGateway(
		"http://order-service.internal:8080",
		"http://product-service.internal:8080",
		"http://payment-service.internal:8080",
	)
	log.Fatal(http.ListenAndServe(":443", gateway))
}
```

### Contoh Kode — Node.js
```javascript
const express = require('express');
const { createProxyMiddleware } = require('http-proxy-middleware');
const { authenticate } = require('./middleware/authenticate');

// createGateway jadi satu pintu masuk buat semua request eksternal,
// meneruskan ke microservice internal berdasarkan path prefix. Auth
// (validateToken, Phase 1) divalidasi sekali di sini, bukan diulang-ulang
// di tiap service.
function createGateway({ orderServiceUrl, productServiceUrl, paymentServiceUrl }) {
  const app = express();

  app.use('/orders', authenticate, createProxyMiddleware({ target: orderServiceUrl, changeOrigin: true }));
  app.use('/products', createProxyMiddleware({ target: productServiceUrl, changeOrigin: true })); // publik
  app.use('/payments', authenticate, createProxyMiddleware({ target: paymentServiceUrl, changeOrigin: true }));

  return app;
}

module.exports = { createGateway };
```

### Trade-off & Pitfall
- API Gateway bisa jadi single point of failure kalau cuma satu instance — perlu di-deploy dengan redundansi (banyak instance di belakang load balancer, topik 67) seperti service lain.
- Menaruh terlalu banyak business logic di gateway (bukan cuma cross-cutting concern seperti auth/rate limit) bikin gateway jadi "god object" yang susah dipelihara — logic spesifik domain tetap harus tinggal di service masing-masing.
- Gateway nambah satu hop network tambahan buat setiap request — biasanya overhead-nya kecil dibanding manfaat sentralisasi, tapi tetap perlu diperhatikan untuk endpoint yang sangat latency-sensitive.
- Kalau gateway down, seluruh akses eksternal ke semua service ikut terputus walau service di baliknya sehat — availability gateway jadi kritikal buat keseluruhan sistem.

### Kapan Dipakai
Dipakai begitu OrderFlow dipecah jadi lebih dari satu microservice yang diakses client eksternal — API Gateway jadi titik sentralisasi auth, rate limiting, dan routing, sekaligus menyembunyikan struktur internal microservices dari client publik.

### Sering Ditanya Saat Interview
- "Apa manfaat utama API Gateway di arsitektur microservices?" — menyentralisasi concern lintas-service (auth, rate limiting, logging, routing) di satu tempat, jadi gak perlu diimplementasikan ulang di tiap service, dan menyembunyikan struktur internal microservices dari client eksternal.
- "Apa risiko kalau API Gateway down?" — seluruh akses eksternal ke semua service di baliknya ikut terputus, walau service-service itu sendiri sehat — makanya gateway sendiri butuh redundansi (banyak instance + load balancer).
- "Kenapa business logic spesifik domain sebaiknya gak ditaruh di API Gateway?" — supaya gateway tetap fokus jadi cross-cutting layer yang generic, gak jadi bottleneck perubahan setiap kali ada perubahan business logic di satu domain tertentu.

---

## 74. Service Discovery

### Apa itu?
Service discovery adalah mekanisme yang memungkinkan satu service menemukan alamat (IP/port) instance service lain yang sedang aktif secara dinamis, tanpa hardcode alamat di config — penting karena di arsitektur microservices yang di-scale horizontal, instance bisa datang dan pergi kapan saja (deploy baru, auto-scaling, crash & restart).

### Kenapa dibutuhkan?
Order Service perlu manggil Inventory Service (topik 72), tapi alamat instance Inventory Service bisa berubah-ubah — instance baru muncul saat auto-scaling, instance lama mati saat deploy. Kalau alamatnya di-hardcode di config Order Service, setiap kali topologi berubah, config itu harus diupdate manual — gak scalable dan rawan human error. Service discovery (misalnya lewat Consul) menyediakan registry terpusat yang selalu up-to-date soal instance mana yang sehat dan di mana alamatnya.

### Cara Kerja
```
1. Instance Inventory Service baru start
     -> register diri ke Consul (alamat + health check endpoint)
2. Consul secara berkala cek health check tiap instance yang terdaftar
3. Order Service butuh manggil Inventory Service:
     -> query Consul: "instance sehat mana saja buat 'inventory-service'?"
     -> Consul balikin daftar alamat instance yang lolos health check
     -> Order Service pilih salah satu (atau load balance sendiri lintas hasil itu)
4. Instance yang crash otomatis gak lagi dibalikin Consul begitu health check-nya gagal
```

### Contoh Kode — Go
```go
package discovery

import (
	"fmt"

	consulapi "github.com/hashicorp/consul/api"
)

// RegisterService mendaftarkan instance OrderFlow ini ke Consul saat startup,
// lengkap dengan health check — supaya service lain bisa menemukan alamat
// instance yang masih hidup secara dinamis, tanpa hardcode IP/port di config.
func RegisterService(client *consulapi.Client, serviceName, serviceID, address string, port int) error {
	registration := &consulapi.AgentServiceRegistration{
		ID:      serviceID,
		Name:    serviceName,
		Address: address,
		Port:    port,
		Check: &consulapi.AgentServiceCheck{
			HTTP:                           fmt.Sprintf("http://%s:%d/healthz", address, port),
			Interval:                       "10s",
			Timeout:                        "2s",
			DeregisterCriticalServiceAfter: "1m",
		},
	}
	return client.Agent().ServiceRegister(registration)
}

// DiscoverHealthyInstances nyari semua instance sehat dari service tertentu
// — dipakai misalnya Order Service buat nemuin alamat Inventory Service
// yang sedang aktif, sebelum manggil lewat gRPC/REST (topik 72).
func DiscoverHealthyInstances(client *consulapi.Client, serviceName string) ([]string, error) {
	entries, _, err := client.Health().Service(serviceName, "", true, nil)
	if err != nil {
		return nil, err
	}

	addrs := make([]string, 0, len(entries))
	for _, entry := range entries {
		addrs = append(addrs, fmt.Sprintf("%s:%d", entry.Service.Address, entry.Service.Port))
	}
	return addrs, nil
}
```

### Contoh Kode — Node.js
```javascript
const Consul = require('consul');

const consul = new Consul({ host: 'consul.internal', port: 8500 });

// registerService mendaftarkan instance OrderFlow ini ke Consul saat
// startup, lengkap dengan health check — supaya service lain bisa
// menemukan alamat instance yang masih hidup secara dinamis.
async function registerService(serviceName, serviceId, address, port) {
  await consul.agent.service.register({
    id: serviceId,
    name: serviceName,
    address,
    port,
    check: {
      http: `http://${address}:${port}/healthz`,
      interval: '10s',
      timeout: '2s',
      deregistercriticalserviceafter: '1m',
    },
  });
}

// discoverHealthyInstances nyari semua instance sehat dari service tertentu
// — dipakai misalnya Order Service buat nemuin alamat Inventory Service
// yang sedang aktif sebelum manggil lewat gRPC/REST (topik 72).
async function discoverHealthyInstances(serviceName) {
  const results = await consul.health.service({ service: serviceName, passing: true });
  return results.map((entry) => `${entry.Service.Address}:${entry.Service.Port}`);
}

module.exports = { registerService, discoverHealthyInstances };
```

### Trade-off & Pitfall
- Service discovery menambah dependency infrastruktur baru (Consul, etcd, atau setara) yang harus dijaga availability-nya — kalau registry-nya down, service jadi gak bisa saling menemukan sama sekali, jadi registry ini sendiri butuh redundansi.
- Ada dua model umum: client-side discovery (client query registry lalu pilih instance sendiri, seperti contoh di atas) dan server-side discovery (client cukup manggil satu alamat stabil, load balancer/gateway di baliknya yang urus lookup ke registry) — client-side lebih fleksibel tapi bikin logic discovery tersebar di banyak service; server-side lebih sentralisasi tapi nambah hop.
- Health check yang gak akurat (misalnya cuma cek proses hidup, bukan cek endpoint aplikasi benar-benar bisa melayani request) bisa bikin registry tetap mengarahkan traffic ke instance yang sebenarnya bermasalah.
- Di lingkungan container orchestration seperti Kubernetes, sebagian kebutuhan service discovery sudah otomatis disediakan lewat DNS internal (Service resource) — pakai Consul terpisah kadang jadi redundan kecuali butuh fitur lebih (multi-datacenter, health check custom) yang gak disediakan orchestrator-nya.

### Kapan Dipakai
Dibutuhkan begitu OrderFlow punya lebih dari satu microservice yang saling memanggil dan tiap service di-scale secara independen dengan jumlah instance yang berubah-ubah. Untuk deployment yang sudah jalan di atas orchestrator seperti Kubernetes, evaluasi dulu apakah DNS-based discovery bawaan orchestrator itu sudah cukup sebelum menambah komponen service discovery terpisah.

### Sering Ditanya Saat Interview
- "Apa beda client-side dan server-side service discovery?" — client-side berarti pemanggil sendiri yang query registry dan memilih instance; server-side berarti ada komponen perantara (load balancer/gateway) yang urus lookup itu, client cukup manggil satu alamat stabil.
- "Kenapa hardcode alamat IP microservice di config gak scalable?" — karena instance di arsitektur microservices modern datang dan pergi terus (auto-scaling, deploy, crash), config yang hardcode harus diupdate manual tiap kali topologi berubah, rawan human error dan downtime saat lupa update.
- "Gimana service discovery menangani instance yang crash?" — instance yang crash bakal gagal health check yang dilakukan registry secara berkala, sehingga otomatis dikeluarkan dari daftar instance sehat dan gak lagi dikirimi traffic oleh service lain.

---

**Selanjutnya:** [Phase 08 — System Design](./phase-08-system-design.md)
