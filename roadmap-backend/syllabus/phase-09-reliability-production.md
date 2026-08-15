# Phase 09 — Reliability & Production

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

## 78. Health Checks (Liveness vs Readiness)

### Apa itu?
Health check adalah endpoint HTTP yang sengaja gak butuh auth (topik 1) supaya orchestrator (Kubernetes, load balancer di topik 67) bisa nanya "kamu sehat gak?" tanpa perlu token. Ada dua jenis yang sering ketuker: **Liveness** ("apakah process ini masih hidup dan gak deadlock?") dan **Readiness** ("apakah instance ini siap nerima traffic sekarang?"). Liveness cuma ngecek proses itu sendiri; Readiness ngecek dependency eksternal yang dibutuhkan buat kerja beneran — buat OrderFlow itu artinya koneksi ke Postgres (Phase 3) dan Redis (Phase 4).

### Kenapa dibutuhkan?
Tanpa health check, Kubernetes cuma tau proses OrderFlow "jalan" (masih ada PID-nya), padahal proses bisa jalan tapi macet total — misalnya semua goroutine kena deadlock nunggu lock (topik 70) yang gak pernah dilepas, tapi HTTP server-nya masih listen di port. Liveness check yang gagal berulang kali bikin Kubernetes restart pod itu (ngatasin deadlock/stuck state). Sebaliknya, kalau Postgres OrderFlow lagi kena masalah jaringan sementara, Readiness yang gagal bikin Load Balancer (topik 67) berhenti kirim traffic ke instance itu — TANPA restart proses, karena proses-nya sendiri sehat-sehat aja, cuma dependency-nya yang lagi bermasalah. Salah pasang keduanya (misalnya Readiness dipasang jadi Liveness) bikin Kubernetes restart semua pod OrderFlow serentak begitu Postgres down sebentar — padahal restart gak menyelesaikan apa-apa kalau masalahnya ada di Postgres, bukan di OrderFlow.

### Cara Kerja
```
Kubernetes / Load Balancer poll tiap N detik:

  GET /healthz/live   -->  LivenessHandler
                             |
                             +--> gak sentuh Postgres/Redis sama sekali
                             +--> selalu 200 OK selama HTTP server-nya
                                  masih bisa menjawab request

  GET /healthz/ready  -->  ReadinessHandler
                             |
                             +--> ctx dengan timeout pendek (2 detik)
                             +--> Postgres.Ping(ctx)  -----> gagal? status=unreachable
                             +--> Redis.Ping(ctx)     -----> gagal? status=unreachable
                             +--> salah satu gagal --> 503 Service Unavailable
                             +--> keduanya ok        --> 200 OK


Liveness GAGAL berkali-kali   -->  Kubernetes RESTART pod (proses dianggap stuck)
Readiness GAGAL               -->  Load Balancer STOP kirim traffic ke pod ini,
                                    TAPI pod TETAP HIDUP, gak di-restart --
                                    begitu Postgres/Redis pulih, Readiness lagi,
                                    traffic otomatis balik lagi.
```

### Contoh Kode — Go
`HealthChecker` membungkus koneksi Postgres (`*pgxpool.Pool`, Phase 3) dan Redis (`*redis.Client`, Phase 4) yang sudah ada, lalu mengekspos `LivenessHandler` dan `ReadinessHandler` sebagai method-nya:
```go
package health

import (
	"context"
	"encoding/json"
	"net/http"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/redis/go-redis/v9"
)

// HealthChecker menyimpan dependency yang perlu dicek Readiness -- Postgres
// dan Redis yang sama yang sudah dipakai OrderFlow sejak Phase 3 & 4.
type HealthChecker struct {
	db  *pgxpool.Pool
	rdb *redis.Client
}

func NewHealthChecker(db *pgxpool.Pool, rdb *redis.Client) *HealthChecker {
	return &HealthChecker{db: db, rdb: rdb}
}

// LivenessHandler sengaja gak menyentuh dependency eksternal apapun --
// kalau handler ini bisa dipanggil dan balikin response, proses OrderFlow
// dianggap "hidup". Kubernetes pakai ini buat memutuskan kapan restart pod.
func LivenessHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	w.Write([]byte("ok"))
}

type readinessStatus struct {
	Postgres string `json:"postgres"`
	Redis    string `json:"redis"`
}

// ReadinessHandler ngecek Postgres dan Redis dengan timeout pendek supaya
// health check sendiri gak ikut nge-hang kalau salah satu dependency lambat.
// Kalau salah satunya gak terjangkau, balikin 503 supaya Load Balancer
// (topik 67) berhenti kirim traffic ke instance ini.
func (h *HealthChecker) ReadinessHandler(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
	defer cancel()

	status := readinessStatus{Postgres: "ok", Redis: "ok"}
	ready := true

	if err := h.db.Ping(ctx); err != nil {
		status.Postgres = "unreachable"
		ready = false
	}

	if err := h.rdb.Ping(ctx).Err(); err != nil {
		status.Redis = "unreachable"
		ready = false
	}

	w.Header().Set("Content-Type", "application/json")
	if ready {
		w.WriteHeader(http.StatusOK)
	} else {
		w.WriteHeader(http.StatusServiceUnavailable)
	}
	json.NewEncoder(w).Encode(status)
}
```
Wiring-nya di `main`, dua route berbeda buat dua handler yang berbeda:
```go
package main

import (
	"net/http"

	"orderflow/internal/health"
)

func registerHealthRoutes(mux *http.ServeMux, checker *health.HealthChecker) {
	mux.HandleFunc("/healthz/live", health.LivenessHandler)
	mux.HandleFunc("/healthz/ready", checker.ReadinessHandler)
}
```

### Contoh Kode — Node.js
`createHealthHandlers` menerima pool Postgres (`pg`, Phase 3) dan client Redis (Phase 4) lewat closure, lalu mengembalikan dua handler yang masing-masing tetap punya signature `(req, res)` standar Express:
```javascript
// health.js
function createHealthHandlers(pgPool, redisClient) {
  // livenessHandler sengaja gak menyentuh Postgres/Redis -- kalau handler
  // ini bisa dijalankan dan balas response, proses dianggap "hidup".
  function livenessHandler(req, res) {
    res.status(200).send('ok');
  }

  // readinessHandler ngecek kedua dependency dengan timeout pendek. Salah
  // satu gagal -> 503, supaya load balancer berhenti kirim traffic ke
  // instance ini sampai dependency-nya pulih.
  async function readinessHandler(req, res) {
    const status = { postgres: 'ok', redis: 'ok' };
    let ready = true;

    try {
      await withTimeout(pgPool.query('SELECT 1'), 2000);
    } catch (err) {
      status.postgres = 'unreachable';
      ready = false;
    }

    try {
      await withTimeout(redisClient.ping(), 2000);
    } catch (err) {
      status.redis = 'unreachable';
      ready = false;
    }

    res.status(ready ? 200 : 503).json(status);
  }

  return { livenessHandler, readinessHandler };
}

// withTimeout membungkus promise supaya health check sendiri gak ikut
// nge-hang kalau salah satu dependency lambat merespons.
function withTimeout(promise, ms) {
  return Promise.race([
    promise,
    new Promise((_, reject) => setTimeout(() => reject(new Error('timeout')), ms)),
  ]);
}

module.exports = { createHealthHandlers };
```
Wiring-nya di file server utama:
```javascript
// server.js
const express = require('express');
const { createHealthHandlers } = require('./health');

function buildServer(pgPool, redisClient) {
  const app = express();
  const { livenessHandler, readinessHandler } = createHealthHandlers(pgPool, redisClient);

  app.get('/healthz/live', livenessHandler);
  app.get('/healthz/ready', readinessHandler);

  return app;
}

module.exports = { buildServer };
```

### Trade-off & Pitfall
- Readiness yang terlalu "cerewet" (misalnya ikut ngecek payment provider eksternal di topik 79) bikin instance yang sebetulnya sehat ikut ditandai not-ready cuma karena dependency pihak ketiga lagi lambat — Readiness sebaiknya cuma ngecek dependency yang bener-bener bikin instance itu gak bisa kerja sama sekali (Postgres, Redis), bukan semua service downstream.
- Liveness yang ikut ngecek Postgres/Redis (ketuker sama Readiness) itu bahaya besar: begitu Postgres down, SEMUA pod OrderFlow bakal di-restart Kubernetes serentak — padahal restart gak menyelesaikan masalah Postgres-nya, malah bikin downtime tambahan pas pod lagi start ulang.
- Timeout di dalam Readiness (2 detik di contoh di atas) harus jauh lebih pendek dari interval polling-nya sendiri, supaya satu pemeriksaan yang lambat gak numpuk jadi banyak goroutine/promise yang nunggu di belakang layar.
- Endpoint health check harus dikecualikan dari middleware auth (Phase 1) dan rate limiting (Phase 2) — kalau kena rate limit dan dianggap "gagal", Kubernetes/Load Balancer bisa salah kesimpulan bahwa instance itu gak sehat.

### Kapan Dipakai
Wajib ada di semua service OrderFlow yang jalan di Kubernetes atau di belakang Load Balancer manapun (topik 67) — Liveness didaftarkan di `livenessProbe`, Readiness di `readinessProbe` pada manifest deployment. Untuk service kecil sekali yang jalan sebagai single process tanpa orchestrator, Readiness masih berguna dipasang di depan Load Balancer manapun, tapi Liveness jadi kurang relevan karena gak ada yang mau restart proses otomatis.

### Sering Ditanya Saat Interview
- "Apa beda konkret Liveness dan Readiness buat service seperti OrderFlow?" — Liveness cuma jawab "process-nya masih hidup", gak menyentuh dependency apapun, dan kegagalannya berujung restart pod; Readiness ngecek Postgres & Redis, dan kegagalannya cuma berujung "berhenti dikirimin traffic" tanpa restart.
- "Kalau Postgres OrderFlow down 30 detik, apa yang idealnya terjadi ke pod-pod-nya?" — Readiness gagal di semua pod (karena `Ping` ke Postgres gagal), Load Balancer stop kirim traffic ke semuanya, tapi Liveness tetap OK sehingga gak ada pod yang direstart; begitu Postgres pulih, Readiness lagi dan traffic otomatis balik tanpa butuh intervensi manual.
- "Kenapa Readiness butuh timeout sendiri yang lebih pendek dari interval probe-nya?" — supaya satu pengecekan yang lambat (misalnya Postgres nge-hang) gak bikin health check itu sendiri jadi nge-hang lebih lama dari interval polling berikutnya, yang bisa numpuk jadi banyak pemeriksaan pending sekaligus.

---

## 79. Circuit Breaker

### Apa itu?
Circuit Breaker adalah pola yang membungkus panggilan ke dependency eksternal yang gak reliable (buat OrderFlow: payment provider pihak ketiga yang dipanggil pas checkout) dengan sebuah state machine tiga status: **CLOSED** (normal, request diteruskan apa adanya), **OPEN** (dependency dianggap lagi bermasalah, request langsung ditolak tanpa nyoba manggil sama sekali), dan **HALF_OPEN** (state percobaan sesudah beberapa waktu, ngirim SATU request "tes" buat lihat apakah dependency-nya udah pulih).

### Kenapa dibutuhkan?
Retry dengan backoff (topik 51, Phase 5) menyelesaikan kegagalan yang sifatnya sesaat, tapi kalau payment provider beneran down atau super lambat buat waktu yang lama, retry berulang-ulang dari ribuan request checkout OrderFlow justru memperparah keadaan — tiap retry tetap makan satu koneksi TCP dan satu goroutine/thread yang nunggu response (atau timeout) dari provider yang gak akan jawab. Ini yang disebut cascading failure: dependency yang down bikin OrderFlow sendiri ikut kehabisan resource (connection pool penuh, goroutine numpuk) meski masalah aslinya ada di luar OrderFlow. Circuit Breaker memutus siklus ini: begitu kegagalan beruntun ngelewatin ambang batas, breaker "membuka" sirkuit dan langsung menolak request checkout berikutnya TANPA nyentuh network sama sekali — gagal cepat (fail fast) jauh lebih murah daripada gagal lambat (nunggu timeout tiap kali).

### Cara Kerja
```
                    request masuk
                          |
                          v
              +------------------------+
              |        CLOSED          |  <---------------------+
              |  request diteruskan    |                        |
              |  apa adanya ke payment |                        |
              |  provider              |                        |
              +------------------------+                        |
                gagal terus sampai                    request TES sukses
                failureCount >= threshold                        |
                          |                                      |
                          v                                      |
              +------------------------+          +------------------------+
              |         OPEN           |--waktu--> |       HALF_OPEN        |
              |  request DITOLAK       | resetTimeout  1 request TES        |
              |  langsung, gak manggil |  lewat    |  boleh lewat, sisanya  |
              |  provider sama sekali  |           |  masih ditolak         |
              +------------------------+           +------------------------+
                          ^                                      |
                          |                request TES gagal lagi
                          +--------------------------------------+

Transisi state:
  CLOSED    --(failureCount mencapai threshold)-->        OPEN
  OPEN      --(sudah lewat resetTimeout sejak dibuka)-->   HALF_OPEN
  HALF_OPEN --(request tes berhasil)-->                    CLOSED (failureCount direset)
  HALF_OPEN --(request tes gagal)-->                       OPEN (timer restart dari nol)
```

### Contoh Kode — Go
`CircuitBreaker` sebagai state machine yang thread-safe (dipanggil dari banyak goroutine handler sekaligus, seperti connection pool di topik 55-58), lalu `CallPaymentProviderWithCircuitBreaker` yang membungkus panggilan HTTP ke payment provider:
```go
package payment

import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"net/http"
	"sync"
	"time"
)

type PaymentRequest struct {
	OrderID int64
	Amount  float64
	Token   string
}

type PaymentResponse struct {
	TransactionID string
	Status        string
}

// ErrCircuitOpen dikembalikan saat breaker lagi OPEN -- caller (topik 81)
// bisa mengecek error ini secara spesifik buat menampilkan graceful
// degradation, bukan error generik "payment failed".
var ErrCircuitOpen = errors.New("circuit breaker open: payment provider skipped")

type circuitState int

const (
	stateClosed circuitState = iota
	stateOpen
	stateHalfOpen
)

// CircuitBreaker adalah state machine CLOSED -> OPEN -> HALF_OPEN yang
// dilindungi mutex karena dipanggil concurrent dari banyak goroutine handler
// checkout sekaligus.
type CircuitBreaker struct {
	mu               sync.Mutex
	state            circuitState
	failureCount     int
	failureThreshold int
	openedAt         time.Time
	resetTimeout     time.Duration
}

func NewCircuitBreaker(failureThreshold int, resetTimeout time.Duration) *CircuitBreaker {
	return &CircuitBreaker{
		state:            stateClosed,
		failureThreshold: failureThreshold,
		resetTimeout:     resetTimeout,
	}
}

// allow memutuskan apakah request boleh diteruskan, dan mengembalikan state
// yang berlaku SAAT keputusan ini diambil (dibutuhkan lagi di recordSuccess/
// recordFailure supaya transisi HALF_OPEN -> CLOSED atau HALF_OPEN -> OPEN
// akurat walau ada request lain yang datang di antaranya).
func (cb *CircuitBreaker) allow() (circuitState, error) {
	cb.mu.Lock()
	defer cb.mu.Unlock()

	switch cb.state {
	case stateOpen:
		if time.Since(cb.openedAt) < cb.resetTimeout {
			return stateOpen, ErrCircuitOpen
		}
		// resetTimeout sudah lewat -- transisi OPEN -> HALF_OPEN, izinkan
		// SATU request tes lewat.
		cb.state = stateHalfOpen
		return stateHalfOpen, nil
	case stateHalfOpen:
		// sudah ada satu request tes yang sedang berjalan; request lain yang
		// datang selagi masih HALF_OPEN tetap ditolak supaya cuma satu
		// "percobaan" yang aktif ke provider pada satu waktu.
		return stateHalfOpen, ErrCircuitOpen
	default:
		return stateClosed, nil
	}
}

// recordSuccess dipanggil setelah panggilan ke provider berhasil. Baik
// sukses dari CLOSED maupun dari request tes HALF_OPEN, hasilnya sama:
// breaker (kembali) CLOSED dan failureCount direset ke nol.
func (cb *CircuitBreaker) recordSuccess() {
	cb.mu.Lock()
	defer cb.mu.Unlock()
	cb.state = stateClosed
	cb.failureCount = 0
}

// recordFailure dipanggil setelah panggilan ke provider gagal. observedState
// adalah state yang dikembalikan allow() sebelumnya -- kalau kegagalan itu
// berasal dari request tes HALF_OPEN, breaker langsung balik OPEN lagi
// (timer resetTimeout dimulai dari nol). Kalau dari CLOSED, failureCount
// cuma nambah, dan baru pindah ke OPEN setelah nyampe threshold.
func (cb *CircuitBreaker) recordFailure(observedState circuitState) {
	cb.mu.Lock()
	defer cb.mu.Unlock()

	if observedState == stateHalfOpen {
		cb.state = stateOpen
		cb.openedAt = time.Now()
		return
	}

	cb.failureCount++
	if cb.failureCount >= cb.failureThreshold {
		cb.state = stateOpen
		cb.openedAt = time.Now()
	}
}

// paymentCircuit adalah satu breaker yang dipakai bersama oleh semua request
// checkout -- 5 kegagalan beruntun membuka sirkuit selama 30 detik.
var paymentCircuit = NewCircuitBreaker(5, 30*time.Second)

// CallPaymentProviderWithCircuitBreaker adalah satu-satunya cara OrderFlow
// manggil payment provider eksternal. Kalau breaker OPEN, fungsi ini gagal
// cepat (ErrCircuitOpen) tanpa pernah menyentuh network.
func CallPaymentProviderWithCircuitBreaker(ctx context.Context, req PaymentRequest) (*PaymentResponse, error) {
	state, err := paymentCircuit.allow()
	if err != nil {
		return nil, err
	}

	resp, err := callPaymentProviderHTTP(ctx, req)
	if err != nil {
		paymentCircuit.recordFailure(state)
		return nil, fmt.Errorf("call payment provider: %w", err)
	}

	paymentCircuit.recordSuccess()
	return resp, nil
}

type providerResponseBody struct {
	TransactionID string `json:"transaction_id"`
	Status        string `json:"status"`
}

// callPaymentProviderHTTP adalah panggilan HTTP mentah ke payment provider --
// dependency yang flaky itu sendiri, dibungkus timeout 3 detik supaya satu
// request yang nge-hang gak menahan goroutine selamanya.
func callPaymentProviderHTTP(ctx context.Context, req PaymentRequest) (*PaymentResponse, error) {
	body, err := json.Marshal(req)
	if err != nil {
		return nil, fmt.Errorf("encode payment request: %w", err)
	}

	httpReq, err := http.NewRequestWithContext(ctx, http.MethodPost,
		"https://payments.example.com/charge", bytesReader(body))
	if err != nil {
		return nil, err
	}
	httpReq.Header.Set("Content-Type", "application/json")

	client := &http.Client{Timeout: 3 * time.Second}
	resp, err := client.Do(httpReq)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	if resp.StatusCode >= http.StatusInternalServerError {
		return nil, fmt.Errorf("payment provider returned %d", resp.StatusCode)
	}

	var out providerResponseBody
	if err := json.NewDecoder(resp.Body).Decode(&out); err != nil {
		return nil, fmt.Errorf("decode payment response: %w", err)
	}
	return &PaymentResponse{TransactionID: out.TransactionID, Status: out.Status}, nil
}
```
Helper kecil buat contoh di atas (`bytesReader` cuma pembungkus tipis `bytes.NewReader`, dipisah biar potongan kode di atas fokus ke alur circuit breaker-nya):
```go
package payment

import (
	"bytes"
	"io"
)

func bytesReader(b []byte) io.Reader {
	return bytes.NewReader(b)
}
```

### Contoh Kode — Node.js
`CircuitBreaker` sebagai class dengan state machine yang sama persis (Node.js single-threaded, jadi gak butuh mutex, tapi transisi state-nya identik):
```javascript
// payment-circuit-breaker.js
const STATE_CLOSED = 'CLOSED';
const STATE_OPEN = 'OPEN';
const STATE_HALF_OPEN = 'HALF_OPEN';

class CircuitOpenError extends Error {
  constructor() {
    super('circuit breaker open: payment provider skipped');
    this.code = 'CIRCUIT_OPEN';
  }
}

class CircuitBreaker {
  constructor(failureThreshold, resetTimeoutMs) {
    this.state = STATE_CLOSED;
    this.failureThreshold = failureThreshold;
    this.resetTimeoutMs = resetTimeoutMs;
    this.failureCount = 0;
    this.openedAt = null;
  }

  // allow memutuskan apakah request boleh lewat, dan mengembalikan state
  // yang berlaku saat itu supaya recordSuccess/recordFailure tahu apakah
  // hasil ini berasal dari request tes HALF_OPEN atau dari CLOSED biasa.
  allow() {
    if (this.state === STATE_OPEN) {
      if (Date.now() - this.openedAt >= this.resetTimeoutMs) {
        this.state = STATE_HALF_OPEN;
        return STATE_HALF_OPEN;
      }
      throw new CircuitOpenError();
    }

    if (this.state === STATE_HALF_OPEN) {
      // sudah ada satu request tes berjalan -- tolak request lain sampai
      // hasil tes itu diketahui.
      throw new CircuitOpenError();
    }

    return STATE_CLOSED;
  }

  // recordSuccess: sukses dari CLOSED maupun dari request tes HALF_OPEN
  // sama-sama membawa breaker (kembali) ke CLOSED dengan failureCount reset.
  recordSuccess() {
    this.state = STATE_CLOSED;
    this.failureCount = 0;
  }

  // recordFailure: kegagalan request tes HALF_OPEN langsung balik ke OPEN
  // dengan timer baru; kegagalan dari CLOSED cuma nambah counter sampai
  // menyentuh threshold.
  recordFailure(observedState) {
    if (observedState === STATE_HALF_OPEN) {
      this.state = STATE_OPEN;
      this.openedAt = Date.now();
      return;
    }

    this.failureCount += 1;
    if (this.failureCount >= this.failureThreshold) {
      this.state = STATE_OPEN;
      this.openedAt = Date.now();
    }
  }
}

// paymentCircuit dipakai bersama oleh semua request checkout -- 5 kegagalan
// beruntun membuka sirkuit selama 30 detik, sama seperti versi Go-nya.
const paymentCircuit = new CircuitBreaker(5, 30000);

// callPaymentProviderWithCircuitBreaker adalah satu-satunya cara OrderFlow
// manggil payment provider eksternal. Kalau breaker OPEN, promise ini reject
// dengan CircuitOpenError tanpa pernah menyentuh network.
async function callPaymentProviderWithCircuitBreaker(req) {
  const state = paymentCircuit.allow(); // throw CircuitOpenError kalau OPEN

  try {
    const response = await fetch('https://payments.example.com/charge', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(req),
      signal: AbortSignal.timeout(3000),
    });

    if (response.status >= 500) {
      throw new Error(`payment provider returned ${response.status}`);
    }

    const data = await response.json();
    paymentCircuit.recordSuccess();
    return { transactionId: data.transaction_id, status: data.status };
  } catch (err) {
    paymentCircuit.recordFailure(state);
    throw err;
  }
}

module.exports = { callPaymentProviderWithCircuitBreaker, CircuitBreaker, CircuitOpenError };
```

### Trade-off & Pitfall
- Circuit Breaker gak menyelesaikan masalah dependency-nya sendiri — kalau payment provider beneran down, breaker cuma bikin OrderFlow gagal cepat dan gak ikut kolaps; provider-nya tetap harus dibetulkan di sisi mereka.
- Threshold (jumlah kegagalan) dan resetTimeout yang terlalu ketat bikin breaker gampang "trigger happy" — membuka sirkuit cuma karena satu-dua kegagalan sesaat yang sebetulnya sudah cukup ditangani retry biasa (topik 51); terlalu longgar bikin breaker telat melindungi sistem dari cascading failure.
- HALF_OPEN cuma mengizinkan SATU request tes lewat pada satu waktu — kalau implementasi salah dan mengizinkan banyak request tes bersamaan, semuanya bisa gagal serentak dan breaker jadi flapping (bolak-balik OPEN-HALF_OPEN-OPEN) tanpa pernah stabil di CLOSED.
- Circuit Breaker sebaiknya per-dependency, bukan satu breaker global buat semua panggilan eksternal — kalau payment provider dan, katakanlah, layanan pengiriman notifikasi berbagi satu breaker yang sama, payment provider yang down bisa ikut memblokir notifikasi yang sebenarnya sehat.

### Kapan Dipakai
Dipakai buat setiap panggilan ke dependency eksternal yang track record kegagalannya bisa "menular" ke sistem sendiri kalau dibiarkan retry terus-menerus — payment provider adalah contoh paling jelas di OrderFlow karena tiap checkout menunggunya secara synchronous. Untuk panggilan yang sudah asynchronous lewat message queue (topik 47-52, kegagalannya ditangani lewat retry + DLQ tanpa menahan request user), circuit breaker biasanya kurang krusial dibanding di jalur synchronous seperti checkout.

### Sering Ditanya Saat Interview
- "Kenapa gak cukup pakai retry with backoff aja tanpa circuit breaker?" — retry tetap menyentuh network setiap kali dicoba; kalau dependency-nya beneran down lama, retry dari ribuan request bakal terus menahan koneksi/goroutine menunggu timeout, memperparah tekanan ke sistem sendiri. Circuit breaker menghentikan percobaan itu sama sekali selama periode OPEN.
- "Apa yang terjadi kalau dua request checkout datang tepat saat breaker baru pindah ke HALF_OPEN?" — implementasi yang benar cuma mengizinkan SATU dari keduanya lewat sebagai request tes; request yang lain tetap ditolak dengan ErrCircuitOpen sampai hasil request tes itu diketahui, supaya provider yang baru pulih gak langsung dibanjiri ulang.
- "Kenapa kegagalan saat HALF_OPEN langsung membuka sirkuit lagi, bukan nambah failureCount seperti biasa?" — request tes di HALF_OPEN sudah mewakili "kesempatan kedua" buat provider; kalau kesempatan itu gagal, itu sinyal kuat provider-nya belum pulih, jadi lebih aman langsung balik OPEN dan restart timer daripada nunggu counter nyampe threshold lagi.

---

## 80. Backpressure

### Apa itu?
Backpressure adalah mekanisme sebuah komponen memberi sinyal ke komponen di hulu (upstream) buat memperlambat atau berhenti mengirim pekerjaan, begitu kapasitas pemrosesannya sudah penuh — alih-alih menerima semua pekerjaan yang masuk lalu menumpuknya di memory tanpa batas. Beda dengan rate limiting (Phase 2) yang membatasi traffic berdasarkan identitas client buat mencegah abuse, backpressure adalah proteksi diri sistem terhadap beban yang sifatnya legitimate tapi melebihi kapasitas — misalnya lonjakan checkout beneran saat flash sale, bukan serangan.

### Kenapa dibutuhkan?
Tanpa backpressure, API server OrderFlow yang menerima 10.000 request checkout/detik padahal cuma sanggup memproses 2.000/detik akan tetap menerima semuanya — tiap request bikin goroutine baru (Phase 6) yang menunggu giliran manggil `CallPaymentProviderWithCircuitBreaker` (topik 79) atau `CreateOrder` (Phase 3), numpuk di memory sampai proses kehabisan resource dan crash total, yang jauh lebih buruk daripada menolak sebagian request lebih awal. Backpressure membatasi jumlah pekerjaan yang diproses bersamaan lewat semaphore/bounded queue: begitu batasnya penuh, request baru langsung ditolak (fail fast) dengan sinyal jelas (`503` + `Retry-After`) bukan diam-diam diantre tanpa batas sampai memory habis.

### Cara Kerja
```
Tanpa backpressure (unbounded):

  10.000 request/detik masuk
        |
        v
  semuanya diterima, tiap request bikin goroutine/promise baru
        |
        v
  goroutine menunggu giliran manggil dependency yang cuma sanggup 2.000/detik
        |
        v
  goroutine numpuk tanpa batas -> memory habis -> proses CRASH
  (request yang sudah lama menunggu pun ikut gagal, bukan cuma yang baru)


Dengan backpressure (bounded semaphore, kapasitas N):

  10.000 request/detik masuk
        |
        v
  coba ambil slot semaphore (kapasitas N, misal 2.000)
        |
        +--> slot TERSEDIA --> proses request seperti biasa,
        |                      lepas slot begitu selesai
        |
        +--> slot PENUH ----> TOLAK LANGSUNG, 503 + Retry-After
                               (gak pernah masuk antrean tak terbatas)

  Hasil: N request diproses dengan latency normal, sisanya ditolak cepat
  dan CLIENT tahu harus retry nanti -- proses OrderFlow sendiri gak pernah
  kehabisan memory karena jumlah pekerjaan aktif selalu dibatasi N.
```

### Contoh Kode — Go
Middleware `NewBackpressureMiddleware` memakai buffered channel sebagai semaphore — pola yang sama seperti worker pool di Phase 6, tapi di sini dipakai buat membatasi request HTTP yang diproses bersamaan:
```go
package middleware

import "net/http"

// NewBackpressureMiddleware membatasi jumlah request yang diproses
// bersamaan pakai buffered channel sebagai semaphore. Begitu semaphore
// penuh, request baru langsung ditolak 503 alih-alih diantre tanpa batas
// di memory -- fail fast, bukan menumpuk sampai proses kehabisan resource.
func NewBackpressureMiddleware(maxConcurrent int, next http.Handler) http.Handler {
	sem := make(chan struct{}, maxConcurrent)

	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		select {
		case sem <- struct{}{}:
			defer func() { <-sem }()
			next.ServeHTTP(w, r)
		default:
			w.Header().Set("Retry-After", "1")
			http.Error(w, "server busy, try again shortly", http.StatusServiceUnavailable)
		}
	})
}
```
Wiring-nya membungkus route checkout — endpoint lain (misalnya lihat produk) sengaja gak dibatasi seketat ini karena jauh lebih murah diproses:
```go
package main

import "net/http"

func buildMux(checkoutHandler http.HandlerFunc) *http.ServeMux {
	mux := http.NewServeMux()

	protectedCheckout := NewBackpressureMiddleware(200, checkoutHandler)
	mux.Handle("/orders", protectedCheckout)

	return mux
}
```

### Contoh Kode — Node.js
Node.js single-threaded, jadi backpressure di sini bukan soal goroutine, tapi soal jumlah request yang lagi "in-flight" nunggu operasi async (panggil payment provider, query Postgres) selesai:
```javascript
// backpressure-middleware.js
// createBackpressureMiddleware membatasi jumlah request yang lagi diproses
// bersamaan (in-flight). Begitu batasnya tercapai, request baru langsung
// ditolak 503 -- fail fast alih-alih diam-diam diantre di event loop tanpa
// batas sampai memory Node.js habis.
function createBackpressureMiddleware(maxConcurrent) {
  let inFlight = 0;

  return function backpressureMiddleware(req, res, next) {
    if (inFlight >= maxConcurrent) {
      res.set('Retry-After', '1');
      return res.status(503).json({ error: 'server busy, try again shortly' });
    }

    inFlight += 1;
    res.on('finish', () => {
      inFlight -= 1;
    });
    next();
  };
}

module.exports = { createBackpressureMiddleware };
```
Wiring-nya di route checkout:
```javascript
// server.js
const express = require('express');
const { createBackpressureMiddleware } = require('./backpressure-middleware');

function buildServer(checkoutHandler) {
  const app = express();
  const backpressure = createBackpressureMiddleware(200);

  app.post('/orders', backpressure, checkoutHandler);

  return app;
}

module.exports = { buildServer };
```

### Trade-off & Pitfall
- Kapasitas semaphore/limit yang terlalu kecil bikin OrderFlow menolak request yang sebetulnya masih sanggup diproses (under-utilizing resource yang ada); terlalu besar bikin backpressure gak sempat mencegah memory habis sebelum sistem keburu overload — angka ini perlu diukur dari load testing (topik 61, Phase 6), bukan ditebak.
- Backpressure di level satu instance gak melihat kondisi instance lain — kalau semua instance kompak menolak request bersamaan pas traffic lagi tinggi, itu sinyal kapasitas keseluruhan (jumlah instance, topik 65) yang kurang, bukan cuma masalah konfigurasi backpressure per instance.
- Client yang nerima 503 + `Retry-After` tapi retry-nya gak diberi jeda (langsung retry secepatnya) bisa memperparah keadaan, mirip thundering herd — retry di sisi client idealnya tetap pakai backoff (topik 51), bukan retry instan berulang-ulang.
- Membedakan endpoint yang perlu backpressure ketat (checkout, yang mahal karena manggil payment provider) dari endpoint murah (lihat produk) itu penting — membatasi semua endpoint dengan angka yang sama bisa menolak traffic murah yang sebenarnya gak mengancam kapasitas sistem sama sekali.

### Kapan Dipakai
Dipakai di jalur request yang mahal diproses atau bergantung ke dependency dengan kapasitas terbatas — checkout OrderFlow (manggil payment provider, topik 79) adalah kandidat utama. Untuk endpoint murah yang cuma baca dari cache (Phase 4) dan gak bergantung ke dependency lambat, backpressure biasanya kurang perlu karena kapasitasnya jauh lebih besar dari traffic normal.

### Sering Ditanya Saat Interview
- "Apa beda backpressure dengan rate limiting?" — rate limiting membatasi traffic per client/API key buat mencegah abuse (Phase 2), diterapkan di edge berdasarkan identitas pengirim; backpressure membatasi total pekerjaan yang diproses bersamaan oleh sistem sendiri, gak peduli siapa pengirimnya, buat melindungi kapasitas internal.
- "Kenapa lebih baik menolak request lebih awal (fail fast) daripada mengantrekannya tanpa batas?" — request yang diantre tanpa batas tetap memakan memory (goroutine/promise yang menunggu) dan pada akhirnya, kalau antreannya keburu penuh sistem, semua request termasuk yang sudah lama menunggu ikut gagal bersamaan saat proses crash — menolak sebagian request lebih awal jauh lebih terkendali dan lebih murah.
- "Gimana cara menentukan angka maxConcurrent yang tepat?" — diukur lewat load testing (topik 61) terhadap kapasitas nyata dependency di belakangnya (misalnya berapa request/detik yang sanggup ditangani connection pool Postgres atau payment provider sebelum latency naik drastis), bukan angka yang ditebak sembarangan.

---

## 81. Graceful Degradation

### Apa itu?
Graceful Degradation adalah strategi mendesain sistem supaya kegagalan satu dependency cuma mematikan fitur yang bergantung ke dependency itu, sementara fitur lain yang gak bergantung ke sana tetap jalan normal — dibanding satu dependency yang down bikin seluruh aplikasi ikut gak bisa dipakai sama sekali.

### Kenapa dibutuhkan?
Payment provider adalah dependency eksternal yang paling gampang bermasalah di OrderFlow (topik 79), tapi payment provider cuma dibutuhkan di SATU titik: proses checkout (`CallPaymentProviderWithCircuitBreaker`). Kalau circuit breaker-nya lagi OPEN karena payment provider down, gak ada alasan user jadi gak bisa buka halaman produk (`GetProductByID`, Phase 3) atau lihat isi keranjang — dua fitur itu sama sekali gak menyentuh payment provider. Tanpa graceful degradation, tim yang panik sering bikin kesalahan sebaliknya: memasang health check global yang menandai SELURUH service "unhealthy" begitu satu dependency non-esensial down, yang bikin Readiness (topik 78) gagal dan Load Balancer berhenti kirim traffic sama sekali — padahal 90% fitur OrderFlow (browsing, keranjang) gak butuh payment provider sama sekali.

### Cara Kerja
```
Payment provider DOWN, circuit breaker (topik 79) berstatus OPEN:

  GET /products/{id}   ------> ProductHandler ------> Postgres/Redis (Phase 3-4)
                                (gak import payment sama sekali)      |
                                                                 tetap 200 OK

  GET /cart            ------> CartHandler   ------> Redis/session store
                                (gak import payment sama sekali)      |
                                                                 tetap 200 OK

  POST /orders (checkout) ---> CheckoutHandler ---> CallPaymentProviderWithCircuitBreaker
                                       |                        |
                                       |                  ErrCircuitOpen
                                       v                        |
                          errors.Is(err, ErrCircuitOpen)? -------
                                       |
                                       v
                     503 + pesan jelas: "checkout gak bisa diproses
                     sekarang, coba lagi beberapa saat -- produk &
                     keranjang tetap bisa dipakai"

  Fitur yang TERDAMPAK : cuma checkout (satu-satunya pemanggil payment provider)
  Fitur yang GAK TERDAMPAK : browsing produk, isi keranjang, riwayat order
```

### Contoh Kode — Go
`CheckoutHandler` adalah satu-satunya handler yang bergantung ke `payment` (topik 79); handler lain (produk, keranjang) sengaja gak di-import di sini sama sekali sebagai bukti mereka gak terdampak:
```go
package handler

import (
	"encoding/json"
	"errors"
	"net/http"

	"orderflow/internal/payment"
)

type CheckoutRequest struct {
	OrderID int64   `json:"order_id"`
	Amount  float64 `json:"amount"`
	Token   string  `json:"token"`
}

// CheckoutHandler adalah satu-satunya handler yang manggil payment provider.
// Kalau circuit breaker lagi OPEN, checkout ditolak lebih awal dengan pesan
// yang jelas ke user -- browsing produk dan isi keranjang (handler lain,
// gak menyentuh package payment sama sekali) tetap jalan normal.
func CheckoutHandler(w http.ResponseWriter, r *http.Request) {
	var req CheckoutRequest
	if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
		http.Error(w, "invalid request body", http.StatusBadRequest)
		return
	}

	resp, err := payment.CallPaymentProviderWithCircuitBreaker(r.Context(), payment.PaymentRequest{
		OrderID: req.OrderID,
		Amount:  req.Amount,
		Token:   req.Token,
	})
	if err != nil {
		if errors.Is(err, payment.ErrCircuitOpen) {
			w.Header().Set("Retry-After", "30")
			http.Error(w,
				"checkout lagi gak bisa diproses, coba lagi beberapa saat -- produk & keranjang tetap bisa dipakai",
				http.StatusServiceUnavailable)
			return
		}
		http.Error(w, "payment failed", http.StatusBadGateway)
		return
	}

	json.NewEncoder(w).Encode(resp)
}
```

### Contoh Kode — Node.js
`checkoutHandler` adalah satu-satunya handler yang mengimpor `payment-circuit-breaker`; handler produk/keranjang di file lain sengaja gak menyentuhnya:
```javascript
// checkout-handler.js
const { callPaymentProviderWithCircuitBreaker } = require('./payment-circuit-breaker');

// checkoutHandler adalah satu-satunya handler yang manggil payment provider.
// Kalau circuit breaker OPEN, checkout ditolak lebih awal dengan pesan
// jelas -- endpoint produk & keranjang (gak require payment-circuit-breaker
// sama sekali) tetap jalan normal.
async function checkoutHandler(req, res) {
  try {
    const result = await callPaymentProviderWithCircuitBreaker({
      orderId: req.body.orderId,
      amount: req.body.amount,
      token: req.body.token,
    });
    res.json(result);
  } catch (err) {
    if (err.code === 'CIRCUIT_OPEN') {
      res.set('Retry-After', '30');
      return res.status(503).json({
        error: 'checkout lagi gak bisa diproses, coba lagi beberapa saat -- produk & keranjang tetap bisa dipakai',
      });
    }
    return res.status(502).json({ error: 'payment failed' });
  }
}

module.exports = { checkoutHandler };
```
Sebagai pembanding, `product-handler.js` gak pernah `require('./payment-circuit-breaker')` — inilah yang membuktikan fitur ini gak ikut terdampak:
```javascript
// product-handler.js
const { getProductById } = require('./db'); // dari Phase 3, bukan payment

async function productHandler(req, res) {
  const product = await getProductById(req.pool, Number(req.params.id));
  if (!product) {
    return res.sendStatus(404);
  }
  return res.json(product);
}

module.exports = { productHandler };
```

### Trade-off & Pitfall
- Graceful degradation butuh dependency graph yang jelas antar fitur — kalau `CheckoutHandler` "diam-diam" juga dipanggil dari flow lain (misalnya endpoint admin buat retry order gagal), degradasi checkout ikut mematikan flow itu juga tanpa disadari; batas fitur yang terdampak harus dipetakan eksplisit, bukan diasumsikan.
- Pesan error yang ditampilkan ke user harus jujur soal apa yang bisa dan gak bisa dipakai ("checkout gak bisa, tapi browsing tetap bisa") — pesan generik seperti "terjadi kesalahan" bikin user gak tahu harus ngapain, padahal sebagian besar fitur sebenarnya masih normal.
- Graceful degradation bukan pengganti Circuit Breaker (topik 79) atau Health Check (topik 78) — ia adalah lapisan tambahan di atas keduanya: circuit breaker yang mendeteksi dependency bermasalah, graceful degradation yang memutuskan bagian mana dari user experience yang boleh tetap jalan meski dependency itu OPEN.
- Fitur yang "boleh gagal diam-diam" (misalnya rekomendasi produk yang gagal dimuat) beda dengan fitur yang harus ditolak eksplisit dengan pesan jelas (checkout) — treat keduanya sama rata (semua ditampilkan sebagai error keras, atau semua disembunyikan diam-diam) sama-sama pengalaman user yang buruk.

### Kapan Dipakai
Dipakai di setiap sistem yang punya lebih dari satu fitur dengan dependency eksternal yang berbeda-beda — begitu ada fitur yang bergantung ke dependency yang diketahui gak selalu reliable (payment provider, layanan pengiriman notifikasi, rekomendasi produk pihak ketiga), fitur itu perlu dipetakan supaya kegagalannya gak "menular" ke fitur lain yang sebenarnya independen.

### Sering Ditanya Saat Interview
- "Kalau payment provider down, kenapa OrderFlow gak sekalian menampilkan halaman maintenance penuh biar konsisten?" — karena browsing produk dan keranjang gak bergantung ke payment provider sama sekali; menutup seluruh aplikasi cuma karena satu dependency yang sebetulnya cuma dipakai di satu fitur adalah downtime yang gak perlu dan merugikan bisnis (user yang lagi browsing tetap bisa dilayani).
- "Gimana cara memastikan graceful degradation ini bener-bener diterapkan, bukan cuma niat baik di kepala developer?" — dengan menjaga dependency graph tetap eksplisit di level kode (seperti `product-handler.js` yang secara sengaja gak pernah `require` modul payment) dan diuji lewat test/chaos testing yang mematikan payment provider secara sengaja lalu memverifikasi endpoint produk & keranjang tetap balikin 200.
- "Apa hubungan graceful degradation dengan Circuit Breaker?" — Circuit Breaker menyediakan SINYAL yang bisa dibedakan (`ErrCircuitOpen` vs error lain) tentang kapan sebuah dependency dianggap bermasalah; graceful degradation memakai sinyal itu buat memutuskan bagian mana dari aplikasi yang harus berhenti (checkout) dan bagian mana yang tetap jalan (browsing, keranjang).

---

## 82. Backup & Disaster Recovery (RPO/RTO)

### Apa itu?
Backup & Disaster Recovery adalah rencana buat memulihkan data OrderFlow (order, product, user di Postgres, Phase 3) kalau terjadi kejadian fatal — corrupt data, human error (`DROP TABLE` gak sengaja), atau seluruh region cloud provider down. Dua metrik yang mengukur seberapa baik rencana ini: **RPO (Recovery Point Objective)** — seberapa banyak data yang boleh hilang, diukur dalam waktu (misalnya "boleh kehilangan maksimal 1 jam data terakhir" berarti backup harus diambil minimal tiap 1 jam); dan **RTO (Recovery Time Objective)** — seberapa lama sistem boleh down sebelum pulih total (misalnya "maksimal 30 menit dari insiden sampai OrderFlow bisa terima order lagi").

### Kenapa dibutuhkan?
Semua topik reliability sebelumnya di phase ini (health check, circuit breaker, backpressure, graceful degradation) mengasumsikan data di Postgres tetap utuh — mereka melindungi dari kegagalan sementara, bukan dari hilangnya data secara permanen. Kalau Postgres OrderFlow kena corrupt disk atau ada yang gak sengaja jalanin migration destruktif ke production, gak ada circuit breaker atau health check yang bisa mengembalikan data order yang sudah hilang — satu-satunya yang bisa menyelamatkan adalah backup yang sudah diambil sebelumnya. Tanpa RPO/RTO yang didefinisikan eksplisit, tim gak punya cara ngukur "seberapa buruk kerusakan yang bisa diterima" — apakah kehilangan 1 hari data order itu bencana besar (buat OrderFlow, iya — itu berarti ribuan order hilang) atau bisa diterima, dan berapa lama bisnis boleh berhenti terima order sebelum kerugian jadi gak tertahankan.

### Cara Kerja
```
Timeline normal operasional:

  00:00   01:00   02:00   03:00   04:00   05:00 <- INSIDEN (Postgres corrupt)
    |       |       |       |       |       X
  backup  backup  backup  backup  backup
 (pg_dump tiap 1 jam, disimpan di storage terpisah dari Postgres)

  RPO = 1 jam --> data yang hilang paling banyak adalah data antara
                  backup 04:00 dan insiden di 05:00 (order yang masuk
                  di jam itu, kalau belum sempat di-backup lagi)

Proses recovery setelah insiden:

  05:00 insiden terdeteksi
    |
    v
  05:05 provision Postgres instance baru (kosong)
    |
    v
  05:10 pg_restore dari backup terakhir (04:00) ke instance baru
    |
    v
  05:25 verifikasi data (row count, spot check order terbaru yang berhasil di-restore)
    |
    v
  05:30 arahkan OrderFlow ke instance baru, terima traffic lagi

  RTO = 30 menit --> dari insiden (05:00) sampai OrderFlow bisa terima
                     order lagi (05:30)
```

### Contoh Kode — Go
`RunBackup` menjalankan `pg_dump` dalam custom format (`-Fc`, lebih ringkas dan mendukung restore selektif dibanding plain SQL) dengan timestamp di nama file; `RestoreBackup` menjalankan `pg_restore` buat proses disaster recovery:
```go
package backup

import (
	"fmt"
	"os"
	"os/exec"
	"time"
)

// RunBackup menjalankan pg_dump dan menyimpan hasilnya sebagai file custom
// format (-Fc) yang terkompresi, dengan nama file berisi timestamp supaya
// jelas RPO dari backup mana yang tersedia kalau dibutuhkan buat restore.
func RunBackup(dbURL, backupDir string) (string, error) {
	timestamp := time.Now().UTC().Format("20060102T150405Z")
	outFile := fmt.Sprintf("%s/orderflow-%s.dump", backupDir, timestamp)

	cmd := exec.Command("pg_dump", dbURL, "-Fc", "-f", outFile)
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	if err := cmd.Run(); err != nil {
		return "", fmt.Errorf("pg_dump failed: %w", err)
	}
	return outFile, nil
}

// RestoreBackup menjalankan pg_restore ke database tujuan saat disaster
// recovery. --clean --if-exists menghapus object lama dulu (kalau ada)
// sebelum restore, supaya gak bentrok sama skema yang mungkin sudah ada
// di instance baru.
func RestoreBackup(dumpFile, targetDBURL string) error {
	cmd := exec.Command("pg_restore", "--clean", "--if-exists", "-d", targetDBURL, dumpFile)
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	if err := cmd.Run(); err != nil {
		return fmt.Errorf("pg_restore failed: %w", err)
	}
	return nil
}
```
Dijalankan sebagai scheduled job (misalnya lewat cron atau Kubernetes CronJob) tiap jam, sesuai RPO 1 jam yang sudah disepakati:
```go
package main

import (
	"fmt"
	"os"

	"orderflow/internal/backup"
)

func main() {
	dbURL := os.Getenv("DATABASE_URL")
	backupDir := "/var/backups/orderflow"

	outFile, err := backup.RunBackup(dbURL, backupDir)
	if err != nil {
		fmt.Fprintln(os.Stderr, "backup failed:", err)
		os.Exit(1)
	}
	fmt.Println("backup written to", outFile)
}
```

### Contoh Kode — Node.js
Versi yang sama pakai `child_process.execFile` buat menjalankan `pg_dump`/`pg_restore` (tetap manggil binary Postgres asli, bukan reimplementasi logic dump/restore-nya):
```javascript
// backup.js
const { execFile } = require('child_process');
const { promisify } = require('util');

const execFileAsync = promisify(execFile);

// runBackup menjalankan pg_dump dan menyimpan hasilnya sebagai custom-format
// dump dengan timestamp di nama file, supaya jelas RPO dari tiap backup yang
// tersimpan.
async function runBackup(dbUrl, backupDir) {
  const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
  const outFile = `${backupDir}/orderflow-${timestamp}.dump`;
  await execFileAsync('pg_dump', [dbUrl, '-Fc', '-f', outFile]);
  return outFile;
}

// restoreBackup menjalankan pg_restore ke database tujuan saat disaster
// recovery. --clean --if-exists menghapus object lama (kalau ada) dulu
// sebelum restore.
async function restoreBackup(dumpFile, targetDbUrl) {
  await execFileAsync('pg_restore', ['--clean', '--if-exists', '-d', targetDbUrl, dumpFile]);
}

module.exports = { runBackup, restoreBackup };
```
Dijalankan sebagai scheduled job tiap jam:
```javascript
// run-backup-job.js
const { runBackup } = require('./backup');

async function main() {
  const dbUrl = process.env.DATABASE_URL;
  const backupDir = '/var/backups/orderflow';

  try {
    const outFile = await runBackup(dbUrl, backupDir);
    console.log('backup written to', outFile);
  } catch (err) {
    console.error('backup failed:', err);
    process.exitCode = 1;
  }
}

main();
```
Contoh perintah manual buat restore ke instance baru saat disaster recovery beneran terjadi:
```bash
pg_restore --clean --if-exists -d "$TARGET_DATABASE_URL" orderflow-20260101T040000Z.dump
```

### Trade-off & Pitfall
- Backup yang disimpan di disk/server yang SAMA dengan Postgres produksi gak melindungi dari kegagalan disk atau region itu sendiri — backup wajib disalin ke storage terpisah secara fisik/region (misalnya object storage di region lain), kalau tidak, RPO/RTO yang direncanakan jadi omong kosong saat insiden beneran terjadi di level infrastruktur.
- RPO yang lebih ketat (backup lebih sering, misalnya tiap 5 menit lewat continuous archiving/WAL shipping, bukan cuma `pg_dump` tiap jam) berarti lebih banyak resource (storage, I/O) yang dipakai buat backup — trade-off ini harus disesuaikan dengan seberapa mahal kehilangan data buat bisnis, bukan diset "seketat mungkin" secara default.
- `pg_dump` custom-format cocok buat database berukuran kecil-menengah; begitu ukuran database OrderFlow membesar signifikan, waktu yang dibutuhkan `pg_dump` maupun `pg_restore` ikut membesar, yang bisa membuat RTO yang sudah disepakati (misalnya 30 menit) gak lagi realistis — solusi di skala besar biasanya pindah ke physical backup/replication tools yang lebih cepat direstore.
- Backup yang gak pernah dicoba di-restore adalah backup yang gak bisa dipercaya — banyak tim baru sadar backup-nya corrupt atau gak lengkap justru pas insiden beneran terjadi; restore drill (mencoba restore ke instance test secara berkala) adalah bagian yang sama pentingnya dengan proses backup itu sendiri.

### Kapan Dipakai
Wajib ada sejak Postgres OrderFlow menyimpan data yang gak bisa direkonstruksi ulang dari sumber lain (order, pembayaran, user) — bukan sesuatu yang ditunda sampai "nanti kalau sudah production beneran". RPO/RTO yang spesifik (bukan cuma "kami backup rutin") perlu disepakati di awal bersama pihak bisnis, karena angka itu yang menentukan seberapa sering backup harus jalan dan seberapa cepat proses restore harus disiapkan.

### Sering Ditanya Saat Interview
- "Apa beda RPO dan RTO?" — RPO ngukur seberapa banyak DATA yang boleh hilang (ditentukan seberapa sering backup diambil); RTO ngukur seberapa lama WAKTU sistem boleh down sampai pulih total (ditentukan seberapa cepat proses restore & verifikasi bisa dijalankan).
- "Kalau RPO OrderFlow disepakati 1 jam, apa artinya buat frekuensi backup?" — backup (pg_dump atau mekanisme lain seperti WAL archiving) harus diambil setidaknya tiap 1 jam, karena kalau insiden terjadi tepat sebelum backup berikutnya, data yang hilang paling banyak adalah data dari jam terakhir sejak backup sebelumnya.
- "Kenapa backup wajib disimpan di storage/region yang terpisah dari database produksinya?" — supaya kejadian yang menghancurkan database produksi (disk corrupt, region down) gak ikut menghancurkan backup-nya sekaligus; kalau keduanya di tempat yang sama, satu insiden bisa menghilangkan data DAN cara buat memulihkannya secara bersamaan.

---

**Selanjutnya:** [Phase 10 — Observability](./phase-10-observability.md)
