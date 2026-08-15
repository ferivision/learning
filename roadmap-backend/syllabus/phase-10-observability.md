# Phase 10 — Observability

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

## 83. Logs

### Apa itu?
Structured logging adalah cara nulis log sebagai data terstruktur (key-value/JSON) bukan sekadar string bebas ("user 42 checkout gagal") — tiap baris log punya field yang konsisten (`level`, `msg`, `user_id`, `error`, dst) supaya gampang di-`grep`, di-filter, dan di-agregasi oleh log aggregator (Loki, Elasticsearch, Datadog). Correlation ID adalah satu identifier (biasanya di-generate di awal request) yang ditempelkan ke SETIAP baris log yang dihasilkan selama request itu berjalan, termasuk yang lewat beberapa fungsi atau bahkan beberapa service.

### Kenapa dibutuhkan?
Tanpa structured logging, log OrderFlow cuma kumpulan kalimat bebas yang gak konsisten formatnya antar developer — satu orang nulis `"user 42 gagal checkout"`, orang lain nulis `"checkout failed for user=42"`, dan log aggregator gak bisa nge-query "tampilkan semua error checkout untuk user 42" secara reliable karena harus tebak-tebak pola teksnya. Tanpa correlation ID, begitu satu request `CreateOrder` (Phase 3, topik 29) gagal di production, engineer yang investigasi cuma punya timestamp kira-kira dan harus menebak baris log mana saja yang berasal dari request yang sama di antara ribuan baris log dari request lain yang jalan bersamaan di waktu yang berdekatan — apalagi kalau `CreateOrder` sendiri menghasilkan beberapa baris log (mulai, gagal ambil stock, commit) yang tersebar.

### Cara Kerja
```
Request masuk --> CorrelationIDMiddleware
                     |
                     +--> ada header X-Correlation-ID dari caller?
                     |      ya  -> pakai itu (request ini bagian dari
                     |             trace yang sudah dimulai service lain)
                     |      gak -> generate ID baru
                     |
                     +--> simpan cid di context.Context (Go) /
                     |    AsyncLocalStorage (Node.js)
                     |
                     v
              handler --> CreateOrderLogged(ctx, ...)
                             |
                             +--> log "order.create.start"   {correlation_id: cid, ...}
                             +--> panggil CreateOrder (Phase 3, topik 29) asli
                             +--> log "order.create.success" {correlation_id: cid, ...}
                                  atau
                             +--> log "order.create.failed"  {correlation_id: cid, error: ...}

Semua baris log di atas punya correlation_id yang SAMA -->
  `grep correlation_id=abc123` di log aggregator langsung menampilkan
  seluruh cerita satu request, dari awal sampai gagal/sukses.
```

### Contoh Kode — Go
`CorrelationIDMiddleware` generate (atau meneruskan) correlation ID lewat `context.Context`, sebelum request menyentuh handler manapun:
```go
package httpmw

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"net/http"
)

type correlationIDKey struct{}

// CorrelationIDMiddleware mengambil correlation ID dari header
// X-Correlation-ID kalau caller (API gateway, service lain) sudah
// mengirimkannya, atau generate yang baru kalau belum ada -- lalu
// menyimpannya di context supaya semua log sepanjang request ini bisa
// disatukan lewat satu ID yang sama.
func CorrelationIDMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		cid := r.Header.Get("X-Correlation-ID")
		if cid == "" {
			cid = newCorrelationID()
		}
		w.Header().Set("X-Correlation-ID", cid)

		ctx := context.WithValue(r.Context(), correlationIDKey{}, cid)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

func newCorrelationID() string {
	b := make([]byte, 8)
	_, _ = rand.Read(b)
	return hex.EncodeToString(b)
}

// CorrelationIDFromContext dipakai di layer manapun (handler, CreateOrderLogged
// di bawah) yang perlu menyertakan correlation ID yang sama di log-nya.
func CorrelationIDFromContext(ctx context.Context) string {
	cid, _ := ctx.Value(correlationIDKey{}).(string)
	return cid
}
```
`CreateOrderLogged` membungkus `CreateOrder` (Phase 3, topik 29) dengan structured logging lewat `log/slog` — `CreateOrder` sendiri sama sekali gak diubah:
```go
package db

import (
	"context"
	"log/slog"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"

	"orderflow/httpmw"
)

// CreateOrderLogged membungkus CreateOrder (Phase 3, topik 29) dengan
// structured logging (JSON lewat slog) yang menyertakan correlation ID dari
// context, plus durasi eksekusi -- CreateOrder sendiri gak disentuh sama
// sekali, cuma dibungkus di layer ini.
func CreateOrderLogged(ctx context.Context, db *pgxpool.Pool, userID int64, items []OrderItem) (*Order, error) {
	cid := httpmw.CorrelationIDFromContext(ctx)
	start := time.Now()

	slog.InfoContext(ctx, "order.create.start",
		"correlation_id", cid,
		"user_id", userID,
		"item_count", len(items),
	)

	order, err := CreateOrder(ctx, db, userID, items)
	if err != nil {
		slog.ErrorContext(ctx, "order.create.failed",
			"correlation_id", cid,
			"user_id", userID,
			"error", err.Error(),
			"duration_ms", time.Since(start).Milliseconds(),
		)
		return nil, err
	}

	slog.InfoContext(ctx, "order.create.success",
		"correlation_id", cid,
		"order_id", order.ID,
		"duration_ms", time.Since(start).Milliseconds(),
	)
	return order, nil
}
```

### Contoh Kode — Node.js
`correlationIdMiddleware` pakai `AsyncLocalStorage` bawaan Node.js supaya correlation ID gak perlu diteruskan manual sebagai parameter ke tiap fungsi:
```javascript
// correlation-id.js
const { AsyncLocalStorage } = require('node:async_hooks');
const crypto = require('node:crypto');

const correlationStorage = new AsyncLocalStorage();

// correlationIdMiddleware generate correlation ID baru (atau pakai yang sudah
// dikirim caller lewat header x-correlation-id), lalu menyimpannya di
// AsyncLocalStorage supaya kode di bawahnya (termasuk createOrderLogged)
// bisa mengambilnya tanpa perlu diteruskan manual sebagai parameter.
function correlationIdMiddleware(req, res, next) {
  const cid = req.headers['x-correlation-id'] || crypto.randomUUID();
  res.setHeader('x-correlation-id', cid);
  correlationStorage.run({ correlationId: cid }, () => next());
}

function currentCorrelationId() {
  return correlationStorage.getStore()?.correlationId;
}

module.exports = { correlationIdMiddleware, currentCorrelationId };
```
`createOrderLogged` membungkus `createOrder` (Phase 3, topik 29) dengan structured logging lewat `pino`:
```javascript
// order-logged.js
const pino = require('pino');
const { createOrder } = require('./order-create'); // createOrder asli (Phase 3, topik 29)
const { currentCorrelationId } = require('./correlation-id');

const logger = pino();

// createOrderLogged membungkus createOrder (Phase 3, topik 29) dengan
// structured logging (JSON lewat pino) yang menyertakan correlation ID --
// createOrder sendiri gak diubah sama sekali.
async function createOrderLogged(pool, userId, items) {
  const correlationId = currentCorrelationId();
  const start = Date.now();

  logger.info({ correlationId, userId, itemCount: items.length }, 'order.create.start');

  try {
    const order = await createOrder(pool, userId, items);
    logger.info(
      { correlationId, orderId: order.id, durationMs: Date.now() - start },
      'order.create.success'
    );
    return order;
  } catch (err) {
    logger.error(
      { correlationId, userId, err, durationMs: Date.now() - start },
      'order.create.failed'
    );
    throw err;
  }
}

module.exports = { createOrderLogged };
```

### Trade-off & Pitfall
- Structured logging yang berlebihan (log setiap baris query SQL di dalam `CreateOrder`, misalnya) bikin volume log meledak dan biaya log aggregator (yang biasanya charge per GB ingested) ikut meledak — log yang berguna adalah log di titik keputusan (mulai, sukses, gagal), bukan tiap baris eksekusi.
- Correlation ID yang di-generate ulang di tengah jalan (misalnya tiap fungsi generate ID sendiri alih-alih meneruskan yang sudah ada dari context/`AsyncLocalStorage`) menghilangkan seluruh manfaatnya — baris log dari request yang sama jadi punya ID berbeda-beda dan gak bisa dikorelasikan lagi.
- Field log yang isinya data sensitif (password, token pembayaran, nomor kartu) gampang gak sengaja ke-log kalau developer log seluruh `struct`/`object` request tanpa filter — ini pelanggaran serius terhadap data pengguna dan sering luput dari review karena "cuma logging".
- `AsyncLocalStorage` (Node.js) punya overhead performa kecil tapi nyata dibanding meneruskan value secara eksplisit sebagai parameter — untuk hot path dengan traffic sangat tinggi, trade-off ini perlu diukur, bukan diasumsikan gratis.

### Kapan Dipakai
Structured logging + correlation ID dipasang sejak awal, di SEMUA endpoint OrderFlow, bukan cuma yang "penting" seperti checkout — masalah sering muncul justru di endpoint yang jarang dipantau (misalnya update profil) dan tanpa correlation ID, investigasi jadi jauh lebih lambat dibanding kalau infrastrukturnya sudah siap dari awal.

### Sering Ditanya Saat Interview
- "Kenapa gak cukup pakai `console.log`/`fmt.Println` biasa buat debugging production?" — log bebas format gak konsisten dan gak bisa di-query secara reliable oleh log aggregator; structured logging (JSON dengan field tetap) bisa di-filter dan di-agregasi (misalnya "hitung semua error `order.create.failed` per jam") tanpa harus parsing teks manual.
- "Correlation ID beda dengan trace ID (topik 85) gak?" — konsepnya mirip (identifier yang menyatukan satu request), tapi correlation ID biasanya cuma dipakai buat mengorelasikan LOG, sedangkan trace ID adalah bagian dari sistem tracing (span, parent-child) yang juga mengukur durasi dan hubungan antar operasi — di praktiknya keduanya sering disamakan atau bahkan trace ID dipakai juga sebagai correlation ID.
- "Kalau `CreateOrder` dipanggil dari dua tempat berbeda (HTTP handler dan consumer message queue, Phase 5), gimana correlation ID-nya?" — masing-masing entry point punya middleware/wrapper-nya sendiri yang generate atau meneruskan correlation ID sesuai sumbernya (header HTTP vs field di message queue), tapi begitu masuk ke context, `CreateOrderLogged` gak perlu tahu request ini asalnya dari mana.

---

## 84. Metrics

### Apa itu?
Metrics adalah data numerik yang diagregasi sepanjang waktu — counter (jumlah kumulatif, misalnya total request), histogram (distribusi nilai, misalnya durasi request), dan gauge (nilai yang naik-turun, misalnya jumlah koneksi aktif) — berbeda dari log yang berupa catatan diskrit per event. `RequestMetricsMiddleware`/`requestMetricsMiddleware` adalah middleware yang mencatat metrics dasar (RED, dibahas detail di topik 86) untuk SETIAP request HTTP yang masuk ke OrderFlow.

### Kenapa dibutuhkan?
Log (topik 83) bagus buat investigasi SATU request spesifik, tapi gak praktis buat menjawab pertanyaan agregat seperti "berapa persen request OrderFlow yang error dalam 5 menit terakhir?" atau "berapa p99 latency endpoint checkout minggu ini?" — menjawabnya dengan `grep` lewat jutaan baris log terlalu lambat dan mahal. Metrics menyimpan angka yang sudah teragregasi (lewat library seperti `client_golang`/`prom-client`) dan di-scrape Prometheus secara periodik, sehingga pertanyaan semacam itu bisa dijawab dalam hitungan detik lewat PromQL, dan dipakai buat alerting real-time ("p99 latency di atas 2 detik selama 5 menit berturut-turut -> page on-call").

### Cara Kerja
```
Setiap request masuk --> RequestMetricsMiddleware
                            |
                            +--> catat waktu mulai
                            +--> teruskan ke handler asli (next.ServeHTTP / next())
                            +--> handler selesai, status code diketahui
                            +--> requestsTotal.Inc()      label: method, path, status
                            +--> requestDuration.Observe() label: method, path

Prometheus server --> GET /metrics (tiap ~15 detik) --> scrape nilai counter
                                                          & histogram di atas
                                                          --> simpan sebagai
                                                          time-series
```

### Contoh Kode — Go
```go
package httpmw

import (
	"net/http"
	"strconv"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
)

var (
	requestsTotal = promauto.NewCounterVec(prometheus.CounterOpts{
		Name: "orderflow_http_requests_total",
		Help: "Total HTTP request yang diterima OrderFlow, per method/path/status.",
	}, []string{"method", "path", "status"})

	requestDuration = promauto.NewHistogramVec(prometheus.HistogramOpts{
		Name:    "orderflow_http_request_duration_seconds",
		Help:    "Durasi HTTP request OrderFlow dalam detik, per method/path.",
		Buckets: prometheus.DefBuckets,
	}, []string{"method", "path"})
)

// statusRecorder membungkus http.ResponseWriter supaya status code yang
// benar-benar dikirim ke client bisa dibaca setelah handler selesai --
// http.ResponseWriter bawaan gak punya cara buat itu.
type statusRecorder struct {
	http.ResponseWriter
	status int
}

func (r *statusRecorder) WriteHeader(status int) {
	r.status = status
	r.ResponseWriter.WriteHeader(status)
}

// RequestMetricsMiddleware mencatat RED metrics (topik 86) untuk SETIAP
// request yang masuk ke OrderFlow -- Rate dari requestsTotal, Errors dari
// label status pada requestsTotal, dan Duration dari requestDuration.
func RequestMetricsMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		start := time.Now()
		rec := &statusRecorder{ResponseWriter: w, status: http.StatusOK}

		next.ServeHTTP(rec, r)

		duration := time.Since(start).Seconds()
		status := strconv.Itoa(rec.status)
		requestsTotal.WithLabelValues(r.Method, r.URL.Path, status).Inc()
		requestDuration.WithLabelValues(r.Method, r.URL.Path).Observe(duration)
	})
}
```

### Contoh Kode — Node.js
```javascript
// request-metrics-middleware.js
const client = require('prom-client');

const requestsTotal = new client.Counter({
  name: 'orderflow_http_requests_total',
  help: 'Total HTTP request yang diterima OrderFlow, per method/path/status.',
  labelNames: ['method', 'path', 'status'],
});

const requestDuration = new client.Histogram({
  name: 'orderflow_http_request_duration_seconds',
  help: 'Durasi HTTP request OrderFlow dalam detik, per method/path.',
  labelNames: ['method', 'path'],
  buckets: client.linearBuckets(0.01, 0.05, 20),
});

// requestMetricsMiddleware mencatat RED metrics (topik 86) untuk SETIAP
// request yang masuk -- dipasang paling awal di middleware chain Express
// supaya menangkap semua request, termasuk yang berakhir dengan error.
function requestMetricsMiddleware(req, res, next) {
  const start = process.hrtime.bigint();

  res.on('finish', () => {
    const durationSeconds = Number(process.hrtime.bigint() - start) / 1e9;
    const path = req.route ? req.route.path : req.path;

    requestsTotal.inc({ method: req.method, path, status: res.statusCode });
    requestDuration.observe({ method: req.method, path }, durationSeconds);
  });

  next();
}

module.exports = { requestMetricsMiddleware, requestsTotal, requestDuration };
```

### Trade-off & Pitfall
- Label kardinalitas tinggi (misalnya pakai `user_id` atau full query string sebagai label, bukan `path` yang sudah dinormalisasi lewat routing) bikin Prometheus menyimpan satu time-series terpisah per kombinasi label unik — dengan jutaan user, ini bisa meledakkan memory Prometheus sampai crash; label harus dibatasi ke nilai yang jumlahnya terbatas (method, path pattern, status code).
- `RequestMetricsMiddleware` yang dipasang SETELAH middleware lain yang bisa gagal duluan (misalnya auth middleware yang langsung `return` di error tanpa lanjut ke `next`) gak akan pernah mencatat request yang gagal di situ — urutan middleware menentukan cakupan metrics.
- Histogram buckets yang gak sesuai skala latency asli (misalnya default bucket yang berhenti di 10 detik padahal kebanyakan request OrderFlow di bawah 100ms) bikin resolusi p99 di bucket bawah jadi kasar — buckets harus disesuaikan dengan distribusi latency yang benar-benar terjadi.
- Metrics cuma memberi angka AGREGAT — begitu sesuatu terlihat aneh dari dashboard (misalnya error rate naik), Log (topik 83) dan Tracing (topik 85) tetap dibutuhkan buat tahu request SPESIFIK mana yang gagal dan kenapa.

### Kapan Dipakai
`RequestMetricsMiddleware` dipasang di paling awal middleware chain OrderFlow, sebelum middleware lain yang mungkin gagal duluan, supaya SEMUA request tercatat tanpa terkecuali — termasuk yang ditolak auth (topik 1) atau rate limiting (Phase 2), karena justru itu data yang paling sering dibutuhkan buat mendeteksi serangan atau bug di client.

### Sering Ditanya Saat Interview
- "Apa beda Counter, Histogram, dan Gauge?" — Counter cuma naik (reset ke 0 kalau proses restart), cocok buat total request/error; Histogram mencatat distribusi nilai dalam bucket, cocok buat latency yang butuh persentil (p50/p95/p99); Gauge bisa naik-turun, cocok buat nilai sesaat seperti jumlah koneksi aktif atau ukuran antrian.
- "Kenapa `status` code dimasukkan sebagai label di `requestsTotal`, bukan bikin counter terpisah per status?" — supaya PromQL bisa menghitung error rate dengan satu query (`sum(rate(orderflow_http_requests_total{status=~"5.."}[5m])) / sum(rate(orderflow_http_requests_total[5m]))`) tanpa perlu menjumlahkan banyak metric name berbeda secara manual.
- "Kenapa `path` yang dipakai sebagai label harus pattern (`/orders/:id`), bukan path asli (`/orders/42`)?" — path asli per user/resource jumlahnya gak terbatas (kardinalitas tinggi, lihat Trade-off), sedangkan pattern-nya tetap sama untuk semua request ke endpoint yang sama, jadi jumlah time-series tetap terkendali.

---

## 85. Tracing

### Apa itu?
Distributed tracing merekam perjalanan SATU request lewat serangkaian operasi (yang bisa lintas fungsi, lintas goroutine, bahkan lintas service) sebagai satu **trace**, yang terdiri dari beberapa **span** — tiap span punya durasi, atribut, dan status sendiri, tapi semuanya berbagi satu trace ID yang sama supaya bisa divisualisasikan sebagai satu alur waktu utuh (misalnya di Jaeger atau Grafana Tempo).

### Kenapa dibutuhkan?
Metrics (topik 84) bisa bilang "p99 latency checkout naik jadi 3 detik", tapi gak bisa jawab "3 detik itu dihabiskan di mana?" — apakah di `CreateOrder` (Phase 3, topik 29) yang lambat karena lock contention, atau di `CallPaymentProviderWithCircuitBreaker` (Phase 9, topik 79) yang nunggu payment provider eksternal yang lelet? Tanpa tracing, engineer harus menebak berdasarkan asumsi atau nambahin log manual satu-satu; dengan tracing, satu trace langsung menunjukkan span mana yang paling lama, dalam urutan yang persis sama seperti eksekusi aslinya.

### Cara Kerja
```
Trace ID: 7f3a...  (satu ID yang sama untuk SEMUA span di bawah)

[checkout] ------------------------------------------------- 420ms
   |
   +-- [CreateOrder] --------------------------- 80ms
   |      (lock produk, decrement stock, insert order+items)
   |
   +-- [CallPaymentProviderWithCircuitBreaker] --------- 330ms
          |
          +-- circuit breaker: allow() -- 0.1ms (CLOSED)
          +-- HTTP call ke payment provider -- 329ms  <-- paling lambat!

Span "checkout" adalah ROOT SPAN (parent), dua span di bawahnya adalah
CHILD SPAN -- context (Go) / active span (Node.js) yang membawa trace ID
yang sama diteruskan dari root ke masing-masing child, supaya trace
visualization tahu span mana anak dari span mana.
```

### Contoh Kode — Go
`CheckoutHandler` membungkus `CreateOrder` (Phase 3, topik 29) dan `CallPaymentProviderWithCircuitBreaker` (Phase 9, topik 79) dengan span terpisah di bawah satu root span `checkout`:
```go
package handler

import (
	"context"
	"net/http"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/codes"
	"go.opentelemetry.io/otel/trace"

	"orderflow/db"
	"orderflow/internal/payment"
)

var tracer = otel.Tracer("orderflow/checkout")

// CheckoutHandler membungkus CreateOrder dan
// CallPaymentProviderWithCircuitBreaker di dalam root span "checkout" --
// keduanya jadi child span dengan trace ID yang sama, jadi satu request
// checkout kelihatan sebagai satu trace utuh di Jaeger/Tempo.
func CheckoutHandler(w http.ResponseWriter, r *http.Request) {
	ctx, span := tracer.Start(r.Context(), "checkout")
	defer span.End()

	userID, items := parseCheckoutRequest(r) // parsing body, di luar cakupan topik ini

	order, err := createOrderTraced(ctx, userID, items)
	if err != nil {
		span.RecordError(err)
		span.SetStatus(codes.Error, err.Error())
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}

	if _, err := chargeTraced(ctx, order); err != nil {
		span.RecordError(err)
		span.SetStatus(codes.Error, err.Error())
		http.Error(w, err.Error(), http.StatusPaymentRequired)
		return
	}

	w.WriteHeader(http.StatusCreated)
}

// createOrderTraced membungkus db.CreateOrder (Phase 3, topik 29) dengan
// child span -- ctx yang membawa parent span "checkout" diteruskan supaya
// trace ID tetap sama, cuma menambah satu span baru buat operasi ini.
func createOrderTraced(ctx context.Context, userID int64, items []db.OrderItem) (*db.Order, error) {
	ctx, span := tracer.Start(ctx, "CreateOrder", trace.WithAttributes(
		attribute.Int64("user.id", userID),
		attribute.Int("item.count", len(items)),
	))
	defer span.End()

	order, err := db.CreateOrder(ctx, dbPool, userID, items)
	if err != nil {
		span.RecordError(err)
		span.SetStatus(codes.Error, err.Error())
		return nil, err
	}
	span.SetAttributes(attribute.Int64("order.id", order.ID))
	return order, nil
}

// chargeTraced membungkus payment.CallPaymentProviderWithCircuitBreaker
// (Phase 9, topik 79) dengan child span terpisah -- kalau breaker lagi OPEN,
// span ini bakal langsung kelihatan gagal cepat tanpa network call sama sekali.
func chargeTraced(ctx context.Context, order *db.Order) (*payment.PaymentResponse, error) {
	ctx, span := tracer.Start(ctx, "CallPaymentProviderWithCircuitBreaker", trace.WithAttributes(
		attribute.Int64("order.id", order.ID),
	))
	defer span.End()

	resp, err := payment.CallPaymentProviderWithCircuitBreaker(ctx, payment.PaymentRequest{
		OrderID: order.ID,
		Amount:  order.Total,
	})
	if err != nil {
		span.RecordError(err)
		span.SetStatus(codes.Error, err.Error())
		return nil, err
	}
	return resp, nil
}
```

### Contoh Kode — Node.js
`checkoutHandler` melakukan hal yang sama persis lewat OpenTelemetry JS SDK — `tracer.startActiveSpan` otomatis mengambil span aktif sebagai parent:
```javascript
// checkout-handler.js
const { trace, SpanStatusCode } = require('@opentelemetry/api');
const { createOrder } = require('./order-create');
const { callPaymentProviderWithCircuitBreaker } = require('./payment-circuit-breaker');

const tracer = trace.getTracer('orderflow-checkout');

// checkoutHandler membungkus createOrder (Phase 3, topik 29) dan
// callPaymentProviderWithCircuitBreaker (Phase 9, topik 79) di dalam root
// span "checkout" -- keduanya jadi child span dengan trace ID yang sama.
async function checkoutHandler(req, res) {
  await tracer.startActiveSpan('checkout', async (span) => {
    try {
      const { userId, items } = req.body;

      const order = await createOrderTraced(userId, items);
      await chargeTraced(order);

      res.status(201).json(order);
    } catch (err) {
      span.recordException(err);
      span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
      res.status(500).json({ error: err.message });
    } finally {
      span.end();
    }
  });
}

// createOrderTraced membungkus createOrder (Phase 3, topik 29) dengan child
// span -- OpenTelemetry JS otomatis mengambil span "checkout" yang aktif
// sebagai parent, jadi trace ID-nya tetap sama.
async function createOrderTraced(userId, items) {
  return tracer.startActiveSpan('CreateOrder', async (span) => {
    span.setAttribute('user.id', userId);
    span.setAttribute('item.count', items.length);
    try {
      const order = await createOrder(pool, userId, items);
      span.setAttribute('order.id', order.id);
      return order;
    } catch (err) {
      span.recordException(err);
      span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
      throw err;
    } finally {
      span.end();
    }
  });
}

// chargeTraced membungkus callPaymentProviderWithCircuitBreaker (Phase 9,
// topik 79) dengan child span terpisah -- kalau breaker lagi OPEN, span ini
// bakal langsung selesai tanpa network call sama sekali.
async function chargeTraced(order) {
  return tracer.startActiveSpan('CallPaymentProviderWithCircuitBreaker', async (span) => {
    span.setAttribute('order.id', order.id);
    try {
      return await callPaymentProviderWithCircuitBreaker({
        orderId: order.id,
        amount: order.total,
      });
    } catch (err) {
      span.recordException(err);
      span.setStatus({ code: SpanStatusCode.ERROR, message: err.message });
      throw err;
    } finally {
      span.end();
    }
  });
}

module.exports = { checkoutHandler };
```

### Trade-off & Pitfall
- Trace ID/context yang gak diteruskan dengan benar (misalnya `ctx` baru dibuat di tengah jalan alih-alih meneruskan `ctx` yang dikembalikan `tracer.Start`, atau lupa `startActiveSpan` callback) memutus rantai parent-child — span yang seharusnya jadi anak malah muncul sebagai trace terpisah, dan investigasi jadi sama susahnya seperti gak ada tracing sama sekali.
- Span yang terlalu granular (bikin span baru buat setiap pemanggilan fungsi kecil, termasuk yang cuma butuh microdetik) menambah overhead dan bikin visualisasi trace penuh noise — span sebaiknya dipasang di boundary yang berarti secara operasional (panggilan ke database, ke service eksternal, ke queue), bukan di setiap baris kode.
- Lupa `span.End()` (Go, biasanya lewat `defer`) atau lupa `span.end()` di `finally` (Node.js) bikin span itu gak pernah selesai/ke-report — kalau ini terjadi di banyak request, exporter bisa kehabisan memory menyimpan span yang gak pernah di-flush.
- Tracing menambah latency kecil per request (overhead pembuatan span, serialization, network call ke collector) — untuk hot path dengan traffic sangat tinggi, sampling (cuma trace sebagian kecil request, bukan 100%) biasanya dipakai supaya overhead tetap terkendali.

### Kapan Dipakai
Tracing paling berguna di titik-titik yang melibatkan beberapa operasi berurutan lintas dependency — checkout OrderFlow adalah contoh sempurna karena melibatkan `CreateOrder` (database) dan `CallPaymentProviderWithCircuitBreaker` (network eksternal) yang durasinya bisa sangat berbeda-beda tiap request; buat endpoint sederhana yang cuma satu query database, metrics (topik 84) biasanya sudah cukup tanpa perlu tracing detail.

### Sering Ditanya Saat Interview
- "Apa beda span dan trace?" — trace adalah keseluruhan perjalanan satu request (diidentifikasi satu trace ID), sedangkan span adalah satu unit kerja di dalam trace itu (punya durasi dan atribut sendiri); satu trace bisa terdiri dari banyak span yang membentuk struktur parent-child.
- "Kalau `CallPaymentProviderWithCircuitBreaker` gagal karena `ErrCircuitOpen` (topik 79), apa yang terlihat di trace?" — span "CallPaymentProviderWithCircuitBreaker" bakal selesai HAMPIR SEKETIKA (durasi sangat pendek) dengan status error dan atribut error `ErrCircuitOpen` tercatat lewat `span.RecordError`/`span.recordException` — perbedaan durasi yang mencolok ini justru jadi sinyal jelas bahwa request gagal cepat di circuit breaker, bukan gagal lambat karena network timeout.
- "Kenapa gak cukup pakai Log (topik 83) dengan correlation ID buat tracing manual?" — bisa, tapi correlation ID di log gak otomatis punya struktur parent-child dan durasi per operasi seperti span; tracing tool (Jaeger/Tempo) memvisualisasikan durasi tiap span secara proporsional dalam satu timeline, yang jauh lebih cepat dibaca daripada menyusun ulang urutan dari baris-baris log manual.

---

## 86. RED Method

### Apa itu?
RED Method adalah kerangka minimal buat memantau kesehatan service yang menghadapi request (request-driven service, seperti HTTP API OrderFlow): **R**ate (berapa banyak request per detik), **E**rrors (berapa persen dari request itu yang gagal), dan **D**uration (berapa lama tiap request diproses, biasanya dilihat lewat persentil seperti p50/p95/p99). Tiga angka ini, dipantau bersamaan, biasanya sudah cukup buat mendeteksi mayoritas masalah production tanpa perlu dashboard yang rumit.

### Kenapa dibutuhkan?
Tanpa kerangka yang jelas, tim OrderFlow bisa kebanjiran metrics acak tanpa tahu mana yang paling penting dipantau duluan saat insiden — RED memberi urutan investigasi yang jelas: kalau Rate turun drastis, kemungkinan ada masalah upstream (client gak bisa connect); kalau Errors naik, ada bug atau dependency yang down; kalau Duration naik, ada bottleneck (lock contention di `CreateOrder`, atau payment provider yang lambat). `RequestMetricsMiddleware`/`requestMetricsMiddleware` (topik 84) sudah mencatat ketiga angka ini secara generik per HTTP request, tapi RED sebagai METODE juga berlaku untuk operasi non-HTTP seperti panggilan ke payment provider — yang justru gagal-nya "menular" ke keseluruhan checkout kalau gak dipantau terpisah dari metrics HTTP generik.

### Cara Kerja
```
RED generik per HTTP request (topik 84):
  Rate     = rate(orderflow_http_requests_total[5m])
  Errors   = rate(orderflow_http_requests_total{status=~"5.."}[5m])
             / rate(orderflow_http_requests_total[5m])
  Duration = histogram_quantile(0.99,
               rate(orderflow_http_request_duration_seconds_bucket[5m]))

RED khusus payment provider (topik ini):
  satu request checkout bisa gagal di CreateOrder TANPA pernah menyentuh
  payment provider -- kalau Errors payment provider digabung ke Errors
  HTTP generik, tim gak bisa bedakan "checkout gagal karena stock habis"
  vs "checkout gagal karena payment provider down", padahal keduanya
  butuh respons berbeda (yang pertama bukan insiden, yang kedua mungkin ya).

  Rate     = rate(orderflow_payment_calls_total[5m])
  Errors   = rate(orderflow_payment_calls_total{outcome="error"}[5m])
             / rate(orderflow_payment_calls_total[5m])
  Duration = histogram_quantile(0.99,
               rate(orderflow_payment_call_duration_seconds_bucket[5m]))
```

### Contoh Kode — Go
`CallPaymentProviderWithREDMetrics` membungkus `CallPaymentProviderWithCircuitBreaker` (Phase 9, topik 79) dengan RED metrics yang terpisah dari `RequestMetricsMiddleware` (topik 84), supaya kegagalan payment provider gak tercampur dengan kegagalan HTTP generik:
```go
package payment

import (
	"context"
	"errors"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
)

// Metrics RED khusus payment provider -- terpisah dari RequestMetricsMiddleware
// (topik 84) yang RED-nya per HTTP request, karena satu request checkout bisa
// gagal TANPA pernah menyentuh payment provider sama sekali.
var (
	paymentCallsTotal = promauto.NewCounterVec(prometheus.CounterOpts{
		Name: "orderflow_payment_calls_total",
		Help: "Rate: total panggilan ke payment provider, per outcome.",
	}, []string{"outcome"}) // outcome: success | error | circuit_open

	paymentCallDuration = promauto.NewHistogram(prometheus.HistogramOpts{
		Name:    "orderflow_payment_call_duration_seconds",
		Help:    "Duration: durasi tiap panggilan ke payment provider.",
		Buckets: prometheus.DefBuckets,
	})
)

// CallPaymentProviderWithREDMetrics membungkus
// CallPaymentProviderWithCircuitBreaker (Phase 9, topik 79) supaya Rate,
// Errors, dan Duration panggilan ke payment provider tercatat terpisah dari
// metrics HTTP generik (topik 84).
func CallPaymentProviderWithREDMetrics(ctx context.Context, req PaymentRequest) (*PaymentResponse, error) {
	start := time.Now()

	resp, err := CallPaymentProviderWithCircuitBreaker(ctx, req)

	paymentCallDuration.Observe(time.Since(start).Seconds())
	switch {
	case err == nil:
		paymentCallsTotal.WithLabelValues("success").Inc()
	case errors.Is(err, ErrCircuitOpen):
		paymentCallsTotal.WithLabelValues("circuit_open").Inc()
	default:
		paymentCallsTotal.WithLabelValues("error").Inc()
	}

	return resp, err
}
```

### Contoh Kode — Node.js
```javascript
// payment-red-metrics.js
const client = require('prom-client');
const { callPaymentProviderWithCircuitBreaker, CircuitOpenError } = require('./payment-circuit-breaker');

const paymentCallsTotal = new client.Counter({
  name: 'orderflow_payment_calls_total',
  help: 'Rate: total panggilan ke payment provider, per outcome.',
  labelNames: ['outcome'], // success | error | circuit_open
});

const paymentCallDuration = new client.Histogram({
  name: 'orderflow_payment_call_duration_seconds',
  help: 'Duration: durasi tiap panggilan ke payment provider.',
});

// callPaymentProviderWithRedMetrics membungkus
// callPaymentProviderWithCircuitBreaker (Phase 9, topik 79) supaya Rate,
// Errors, dan Duration panggilan ke payment provider tercatat terpisah dari
// requestMetricsMiddleware (topik 84) yang RED-nya per HTTP request.
async function callPaymentProviderWithRedMetrics(req) {
  const end = paymentCallDuration.startTimer();

  try {
    const result = await callPaymentProviderWithCircuitBreaker(req);
    paymentCallsTotal.inc({ outcome: 'success' });
    return result;
  } catch (err) {
    paymentCallsTotal.inc({ outcome: err instanceof CircuitOpenError ? 'circuit_open' : 'error' });
    throw err;
  } finally {
    end();
  }
}

module.exports = { callPaymentProviderWithRedMetrics };
```

### Trade-off & Pitfall
- RED cuma cocok buat service request-driven (HTTP API, RPC) — buat komponen yang sifatnya queue-driven (consumer message queue, Phase 5) atau batch job, kerangka yang lebih relevan biasanya USE Method (Utilization, Saturation, Errors) yang fokus ke resource, bukan request.
- Menggabungkan Errors dari operasi yang penyebab kegagalannya sangat berbeda (misalnya stock habis vs payment provider down) ke satu angka bikin alert jadi kurang actionable — alert "error rate 5%" gak bilang apa-apa soal HARUS action apa; RED per-dependency (seperti contoh di atas) memberi sinyal yang lebih jelas soal ke mana harus melihat duluan.
- Duration yang cuma dilihat dari rata-rata (bukan persentil p95/p99) menyembunyikan masalah — rata-rata bisa kelihatan normal padahal 1% request butuh waktu 10x lebih lama dari biasanya (yang biasanya paling merepotkan user yang kena "ekor" distribusi itu).
- RED gak menjelaskan PENYEBAB masalah, cuma menunjukkan ADA masalah dan di komponen mana — begitu RED menunjukkan Duration payment provider naik, Tracing (topik 85) tetap dibutuhkan buat tahu apakah itu network latency, atau payment provider-nya sendiri yang lambat memproses.

### Kapan Dipakai
RED dipasang sebagai baseline monitoring untuk SETIAP request-driven boundary yang penting di OrderFlow — bukan cuma HTTP layer secara umum (topik 84), tapi juga tiap dependency eksternal kritis (payment provider di topik ini) yang polanya beda dan bisa gagal secara independen dari layer HTTP di atasnya.

### Sering Ditanya Saat Interview
- "Kenapa RED payment provider dipisah dari RED HTTP request, padahal keduanya bisa sama-sama disebut 'metrics checkout'?" — karena kegagalan di satu titik gak berarti kegagalan di titik lain (checkout bisa gagal di `CreateOrder` tanpa payment provider terlibat sama sekali); memisahkan RED per dependency membuat root cause lebih cepat ditemukan dibanding satu metric besar yang mencampur semua kemungkinan penyebab.
- "Apa hubungan RED dengan RequestMetricsMiddleware/requestMetricsMiddleware (topik 84)?" — `RequestMetricsMiddleware` adalah IMPLEMENTASI RED untuk satu boundary spesifik (HTTP request); RED sendiri adalah METODE yang bisa diterapkan berulang kali ke boundary lain (payment provider, database, message queue) dengan pola instrumentasi yang serupa (counter buat Rate/Errors, histogram buat Duration).
- "Kalau cuma bisa pasang satu jenis monitoring buat OrderFlow, kenapa pilih RED?" — RED menjawab pertanyaan yang paling sering ditanya saat insiden ("apakah user terdampak, seberapa banyak, dan seberapa parah") dengan tiga angka yang murah dihitung dan gampang dipahami siapa pun di tim, termasuk yang gak paham detail internal sistemnya.

---

## 87. Monitoring

### Apa itu?
Monitoring di sini merujuk ke ekosistem tooling yang menyatukan Logs (topik 83), Metrics (topik 84/86), dan Tracing (topik 85) jadi satu gambaran observability yang bisa dipantau tim — biasanya lewat kombinasi Prometheus (penyimpanan & query metrics), Grafana (dashboard & alerting), dan OpenTelemetry (standar instrumentasi yang menghasilkan metrics dan trace, seperti yang sudah dipakai di topik 84 dan 85). Cakupan topik ini SENGAJA dibatasi ke kode instrumentasi (counter/histogram yang ditempel ke business logic) — bukan konfigurasi infra Prometheus/Grafana (scrape config, dashboard JSON, alert rules), karena itu di luar cakupan syllabus backend engineering ini.

### Kenapa dibutuhkan?
Topik 84 dan 86 sudah mencatat metrics teknis (HTTP request, panggilan payment provider), tapi tim produk/bisnis OrderFlow sering butuh angka yang bisa langsung dipahami tanpa perlu ngerti HTTP status code atau circuit breaker — misalnya "berapa order yang berhasil dibuat hari ini" dan "berapa rata-rata nilai tiap order". Monitoring yang baik menyatukan metric teknis (buat engineer, dashboard latency/error rate) dan metric bisnis (buat product manager, dashboard jumlah order) di platform yang sama (Grafana), supaya satu insiden bisa langsung dikorelasikan dampaknya ke bisnis — misalnya "error rate naik jam 14:00, dan `orders_created_total` per menit turun drastis di jam yang sama" langsung menunjukkan insiden itu benar-benar berdampak ke revenue, bukan cuma noise teknis.

### Cara Kerja
```
Business metric (topik ini) BERBEDA dari RED generik (topik 84/86):
  RED HTTP        -> "berapa cepat & seberapa sering API merespons"
  RED payment     -> "seberapa sehat panggilan ke payment provider"
  Business metric -> "berapa banyak DAN seberapa besar order yang terjadi"
                      -- angka yang langsung dipahami tim produk tanpa
                      perlu tahu apapun soal HTTP/payment provider

CreateOrder (Phase 3) sukses
        |
        v
CreateOrderMonitored
        |
        +--> ordersCreatedTotal.Inc()          -- Grafana panel "Orders/min"
        +--> orderValue.Observe(order.Total)   -- Grafana panel "Order value
                                                    distribution"

Grafana dashboard menampilkan panel teknis (RED topik 84/86) dan panel
bisnis (topik ini) BERDAMPINGAN, di-scrape dari endpoint /metrics yang sama
oleh Prometheus server yang sama -- OpenTelemetry (topik 85) melengkapi
gambaran ini dengan trace detail kalau salah satu panel menunjukkan anomali.
```

### Contoh Kode — Go
Instrumentasi business metric lewat Prometheus counter & histogram, membungkus `CreateOrder` (Phase 3, topik 29) tanpa mengubahnya:
```go
package db

import (
	"context"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
)

// Business metric -- bukan RED teknis (topik 84/86), tapi angka yang
// langsung dipahami tim produk/bisnis tanpa perlu ngerti HTTP status code
// atau payment provider sama sekali.
var (
	ordersCreatedTotal = promauto.NewCounter(prometheus.CounterOpts{
		Name: "orderflow_orders_created_total",
		Help: "Total order yang berhasil dibuat (CreateOrder sukses).",
	})

	orderValue = promauto.NewHistogram(prometheus.HistogramOpts{
		Name:    "orderflow_order_value_total",
		Help:    "Distribusi nilai (total harga) tiap order yang berhasil dibuat.",
		Buckets: []float64{10_000, 50_000, 100_000, 500_000, 1_000_000, 5_000_000},
	})
)

// CreateOrderMonitored membungkus CreateOrder (Phase 3, topik 29) supaya
// tiap order sukses tercatat sebagai business metric -- dashboard Grafana
// yang menampilkan metric ini gak butuh tahu apapun soal circuit breaker
// atau HTTP status code, cukup "berapa order dan berapa nilainya".
func CreateOrderMonitored(ctx context.Context, db *pgxpool.Pool, userID int64, items []OrderItem) (*Order, error) {
	order, err := CreateOrder(ctx, db, userID, items)
	if err != nil {
		return nil, err
	}

	ordersCreatedTotal.Inc()
	orderValue.Observe(order.Total)
	return order, nil
}
```

### Contoh Kode — Node.js
```javascript
// order-monitoring.js
const client = require('prom-client');
const { createOrder } = require('./order-create');

const ordersCreatedTotal = new client.Counter({
  name: 'orderflow_orders_created_total',
  help: 'Total order yang berhasil dibuat (createOrder sukses).',
});

const orderValue = new client.Histogram({
  name: 'orderflow_order_value_total',
  help: 'Distribusi nilai (total harga) tiap order yang berhasil dibuat.',
  buckets: [10_000, 50_000, 100_000, 500_000, 1_000_000, 5_000_000],
});

// createOrderMonitored membungkus createOrder (Phase 3, topik 29) supaya
// tiap order sukses tercatat sebagai business metric yang sama persis
// dengan versi Go-nya -- dashboard Grafana bisa dipakai bareng dari kedua
// service tanpa perlu tahu bahasa implementasinya.
async function createOrderMonitored(pool, userId, items) {
  const order = await createOrder(pool, userId, items);
  ordersCreatedTotal.inc();
  orderValue.observe(order.total);
  return order;
}

module.exports = { createOrderMonitored };
```

### Trade-off & Pitfall
- Business metric yang ditempel LANGSUNG di dalam `CreateOrder` (bukan lewat wrapper terpisah seperti `CreateOrderMonitored`) mencampur concern teknis (transaction, locking) dengan concern observability — kalau nanti `CreateOrder` dipanggil dari path lain yang gak butuh dicatat sebagai business metric (misalnya job migrasi data lama), wrapper terpisah bisa dilewati, sedangkan kode yang menyatu gak bisa.
- Dashboard Grafana yang berisi puluhan panel tanpa hierarki jelas (teknis dan bisnis bercampur tanpa pemisahan) bikin on-call bingung panel mana yang harus dilihat duluan saat insiden — dashboard yang baik biasanya dipisah per audiens (dashboard teknis buat engineer, dashboard bisnis buat product) meski datanya berasal dari Prometheus yang sama.
- Alert yang dipasang langsung di atas business metric (misalnya "alert kalau `orders_created_total` per menit turun di bawah X") gampang false-positive di jam-jam yang secara alami sepi (tengah malam) kalau threshold-nya statis — alert semacam ini biasanya butuh baseline yang menyesuaikan pola waktu, bukan angka tetap.
- Endpoint `/metrics` yang mengekspos counter dan histogram di atas HARUS tetap dilindungi dari akses publik langsung (network policy/firewall, bukan auth topik 1, karena Prometheus scraper gak bisa login) — endpoint ini gampang jadi bocoran informasi bisnis (jumlah order, nilai transaksi) kalau kebuka ke internet.

### Kapan Dipakai
Business metric seperti `orders_created_total` dan `order_value_total` dipasang begitu ada stakeholder non-engineering (product, bisnis) yang butuh visibilitas real-time ke angka yang mereka pahami — biasanya sejak OrderFlow mulai dipakai serius oleh user nyata, bukan cuma internal testing, karena di situ korelasi antara insiden teknis dan dampak bisnis mulai benar-benar penting.

### Sering Ditanya Saat Interview
- "Kenapa cakupan topik Monitoring di sini cuma instrumentasi kode, bukan setup Prometheus/Grafana penuh?" — instrumentasi (menempelkan counter/histogram ke business logic seperti `CreateOrder`) adalah bagian yang jadi tanggung jawab backend engineer; konfigurasi scrape/dashboard/alert rule biasanya masuk domain platform/SRE dan gak spesifik ke logic aplikasi seperti syllabus ini.
- "Apa hubungan OpenTelemetry dengan Prometheus di sini?" — OpenTelemetry (topik 85) adalah standar instrumentasi yang bisa menghasilkan metrics DAN trace lewat SDK yang sama, sedangkan Prometheus (topik 84/86/87 ini) adalah salah satu backend penyimpanan & query buat metrics — keduanya saling melengkapi: OpenTelemetry bisa dikonfigurasi buat mengekspor metrics ke format yang Prometheus scrape, atau trace ke Jaeger/Tempo.
- "Kenapa business metric seperti `orders_created_total` gak cukup dilihat dari query SQL langsung ke database?" — query SQL langsung ke database production buat monitoring dashboard membebani database yang sama yang dipakai `CreateOrder` buat transaction real-time (Phase 3), dan gak punya riwayat time-series otomatis; metric yang di-scrape Prometheus terpisah dari beban database dan sudah punya riwayat waktu yang siap divisualisasikan tanpa query tambahan.

---

**Selanjutnya:** [Phase 11 — Docker & Kubernetes](./phase-11-docker-kubernetes.md)
