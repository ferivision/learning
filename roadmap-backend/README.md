# Backend Engineer — Master Learning Roadmap

> Roadmap ini disusun berdasarkan fase (phase), bukan random topic list. Ikuti urutannya, karena tiap phase membangun fondasi buat phase berikutnya.
>
> Setiap topik sebaiknya dipelajari dengan menjawab 6 pertanyaan ini (lihat bagian [How to Study Each Topic](#how-to-study-each-topic)):
> 1. Apa itu?
> 2. Kenapa dibutuhkan?
> 3. Gimana cara kerjanya?
> 4. Masalah apa yang diselesaikan?
> 5. Apa trade-off-nya?
> 6. Kapan gue pakai ini?

---

## Daftar Isi
- [Phase 1 — Authentication & Authorization](#phase-1--authentication--authorization)
- [Phase 2 — API Security & Web Security](#phase-2--api-security--web-security)
- [Phase 3 — Database](#phase-3--database)
- [Phase 4 — Caching](#phase-4--caching)
- [Phase 5 — Message Queue & Async Processing](#phase-5--message-queue--async-processing)
- [Phase 6 — Go Concurrency, Testing & Backend Performance](#phase-6--go-concurrency-testing--backend-performance)
- [Phase 7 — Distributed Systems](#phase-7--distributed-systems)
- [Phase 8 — System Design](#phase-8--system-design)
- [Phase 9 — Reliability & Production](#phase-9--reliability--production)
- [Phase 10 — Observability](#phase-10--observability)
- [Phase 11 — Docker & Kubernetes](#phase-11--docker--kubernetes)
- [Phase 12 — CI/CD & Deployment](#phase-12--cicd--deployment)
- [Phase 13 — Performance](#phase-13--performance)
- [Phase 14 — Advanced Backend Concepts](#phase-14--advanced-backend-concepts)
- [Phase 15 — Interview Thinking](#phase-15--interview-thinking)
- [Recommended Study Order](#recommended-study-order)
- [How to Study Each Topic](#how-to-study-each-topic)
- [Practical Project](#practical-project-to-connect-everything)
- [Resources](#resources)
- [Final Mental Model](#final-mental-model)

---

## Phase 1 — Authentication & Authorization

### 1. Authentication vs Authorization
Authentication = Who are you?
Authorization = What are you allowed to do?

```
Login
 ↓
Authentication
 ↓
User = 123

DELETE /users/456
 ↓
Authorization
 ↓
Can User 123 delete User 456?
```

**Remember:** Authentication → Identity. Authorization → Permission.

### 2. Password Authentication
Understand: password hashing, hashing vs encryption, salt, bcrypt, Argon2, password validation, password reset, account lockout, brute-force protection.

```
Password → Hash → Database        ✅
Password → Encryption → Database  ❌
```

### 3. JWT
JWT = JSON Web Token. Structure: `Header.Payload.Signature`.

Common claims: `sub` (user ID), `iat` (issued at), `exp` (expiration), `role`.

```
Login → Validate credentials → Generate JWT → Client
 → Authorization: Bearer <JWT> → API verifies signature → Authenticated
```

Know: signature, expiration, claims, stateless auth, JWT validation, JWT revocation problem, jangan taruh data sensitif di payload, selalu pakai HTTPS.

### 4. Access Token vs Refresh Token
- **Access Token**: short-lived, akses resource, dikirim ke API, punya expiration.
- **Refresh Token**: long-lived, dipakai buat dapetin access token baru, disimpan aman, bisa di-rotate/revoke.

```
Login → Access Token + Refresh Token
 → Access Token expires → Refresh Token → New Access Token → User stays logged in
```

### 5. API Key vs Access Token
- **API Key** → identitas aplikasi/service, biasanya statis, service-to-service.
- **Access Token** → identitas user/session, biasanya short-lived, sering berupa JWT.

### 6. Session vs JWT
| | Pros | Cons |
|---|---|---|
| Session | Easy revocation, server controls state | Butuh session storage, perlu shared store di distributed system |
| JWT | Stateless, mudah dipakai lintas service | Revocation lebih sulit, token tetap valid sampai expired |

### 7. OAuth 2.0
Konsep: Resource Owner, Client, Authorization Server, Resource Server, Access Token, Refresh Token, Authorization Code Flow, PKCE.

```
User → Authorization Server → Authorization Code → Client → Access Token → Resource Server
```

**Jangan bingung:** OAuth 2.0 = authorization framework, JWT = token format.

### 8. RBAC (Role-Based Access Control)
```
Admin  → CREATE, READ, UPDATE, DELETE
Editor → CREATE, READ, UPDATE
Viewer → READ
```
Flow: Authentication → Identify user → Get role → Check permission → Allow/Deny.

### 9. Least Privilege
Berikan tiap user/service hanya permission yang benar-benar dibutuhkan. Kalau credential bocor → limited permission → limited damage.

---

## Phase 2 — API Security & Web Security

### 10. HTTP Fundamentals
Methods: GET, POST, PUT, PATCH, DELETE — pahami safe, idempotent, dan cacheable methods.

Status codes penting:
```
200 OK        201 Created     204 No Content
400 Bad Request   401 Unauthenticated   403 Forbidden
404 Not Found     409 Conflict          422 Unprocessable Entity   429 Too Many Requests
500 Internal Server Error  502 Bad Gateway  503 Service Unavailable  504 Gateway Timeout
```
**401** → Who are you? **403** → I know who you are, but you're not allowed.

### 11. REST API Design
Konsep: resource-oriented API, HTTP methods, status codes, request/response structure, versioning, pagination, filtering, sorting, search, validation.

```
GET /users          GET /users/123
POST /users         PATCH /users/123
DELETE /users/123
```

### 12. API Documentation (OpenAPI/Swagger) — *new*
API tanpa dokumentasi susah dipakai orang lain (termasuk future-you).
- OpenAPI spec (dulu Swagger) → format standar buat describe endpoint, request/response schema, auth requirement.
- Manfaat: auto-generate client SDK, interactive docs (Swagger UI), contract yang jadi source of truth antara frontend-backend.
- Practice: tulis spec sebelum/bersamaan dengan implementasi (design-first), bukan cuma nempel setelah selesai.

### 13. API Authentication
Mekanisme umum: API Key, JWT, OAuth 2.0, Session, mTLS. Pahami kapan masing-masing cocok dipakai.

### 14. API Authorization
```
GET /orders/123
```
Backend yang cuma cek "apakah user authenticated?" itu belum cukup. Harus cek **"apakah order 123 milik user ini?"** — kalau tidak, ini jadi celah **IDOR/BOLA** (Insecure Direct Object Reference / Broken Object Level Authorization).

### 15. Rate Limiting
Algoritma: Fixed Window, Sliding Window, Token Bucket, Leaky Bucket. Storage umum: Redis.
Use case: login, OTP, public API, expensive endpoint, brute-force prevention.

### 16. Idempotency
Operasi idempotent = diulang berkali-kali, hasilnya tetap sama.
```
POST /payments
Idempotency-Key: abc123
```
Kalau request di-retry dengan key yang sama → server kenali request sebelumnya → gak bikin payment duplikat. Penting untuk payment, order, distributed systems.

### 17. Retry
Gunakan: exponential backoff, max retry count, jitter. Hati-hati retry buat operasi yang **non-idempotent**.

### 18. Timeout
Tanpa timeout: service hang → connection menumpuk → sistem gak sehat.
Jenis: connection timeout, read timeout, write timeout, request timeout, DB query timeout.

### 19. CORS
Mengatur origin browser mana yang boleh akses resource. **CORS itu proteksi di sisi browser, bukan proteksi API dari non-browser client.**

### 20. CSRF
Attacker menipu browser yang sudah authenticated buat bikin request tanpa sepengetahuan user. Penting kalau auth pakai cookie.
Proteksi: CSRF token, SameSite cookies, validasi Origin/Referer.

### 21. XSS
Attacker inject JavaScript jahat ke halaman. Tipe: Stored, Reflected, DOM-based.
Proteksi: output encoding, input validation, Content Security Policy, secure framework default.

### 22. SQL Injection
```
"SELECT * FROM users WHERE email = '" + email + "'"   ❌ Dangerous
Parameterized query / prepared statement                ✅ Safe
```
Raw SQL itu sendiri gak bahaya — yang bahaya itu string concatenation.

### 23. SSRF
Attacker menipu backend supaya request ke resource internal (internal service, cloud metadata, admin endpoint).
Proteksi: URL allowlist, block private IP range, restrict protocol, network segmentation, validasi redirect.

### 24. Encryption Fundamentals — *new*
- **Symmetric encryption** (AES) → satu key buat encrypt & decrypt, cepat, dipakai buat data at-rest.
- **Asymmetric encryption** (RSA, ECC) → public/private key pair, dipakai buat exchange key & signature.
- **TLS handshake** (garis besar): client hello → server certificate → key exchange → encrypted session dimulai.
- **Encryption in transit** (TLS/HTTPS) vs **encryption at rest** (disk/DB encryption) — dua hal yang beda, keduanya perlu ada.
- Ini fondasi buat paham kenapa "selalu pakai HTTPS" (poin 3) dan "encryption" di DB security (poin 40) itu penting.

---

## Phase 3 — Database

### 25. Database Fundamentals
Tables, rows, primary key, foreign key, constraints, unique constraint, normalization, denormalization.

### 26. Indexing
```sql
CREATE INDEX idx_users_email ON users(email);
```
Tanpa index → sequential scan. Dengan index → index lookup. Trade-off: faster read, tapi lebih banyak storage & write lebih lambat. **Jangan index semua kolom.**

### 27. Composite Index
```sql
CREATE INDEX idx_users_status_created ON users(status, created_at);
```
Urutan kolom penting. Pahami: leftmost prefix, selectivity, query pattern, covering index.

### 28. EXPLAIN ANALYZE
```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@email.com';
```
Perhatikan: sequential scan vs index scan, execution time, rows, cost, actual vs estimated rows.
```
Slow Query → EXPLAIN ANALYZE → Understand plan → Index/rewrite query → Test again
```

### 29. Transactions
```sql
BEGIN
 UPDATE A
 UPDATE B
COMMIT   -- atau ROLLBACK kalau gagal
```

### 30. ACID
**A**tomicity (all or nothing), **C**onsistency (DB tetap valid), **I**solation (transaction konkuren gak saling ganggu), **D**urability (data yang commit tetap ada walau ada failure).

### 31. Isolation Levels
Read Uncommitted → Read Committed → Repeatable Read → Serializable.
Anomali: Dirty Read, Non-repeatable Read, Phantom Read.
**Higher isolation → more consistency, less concurrency.**

### 32. Deadlock
```
Transaction A → lock Row 1 → wait Row 2
Transaction B → lock Row 2 → wait Row 1
```
Prevention: consistent lock ordering, short transaction, proper index, retry.

### 33. Optimistic Locking
```sql
UPDATE users SET name='John', version=6 WHERE id=123 AND version=5;
```
Kalau affected rows = 0 → ada orang lain yang udah ubah record itu.

### 34. Pessimistic Locking
```sql
SELECT * FROM users WHERE id = 123 FOR UPDATE;
```
**Optimistic** → detect conflict. **Pessimistic** → prevent conflict pakai lock.

### 35. N+1 Query
```
Get 100 users → 1 query users → 100 queries orders → 101 queries total
```
Solusi: JOIN, preload, eager loading, batch query, DataLoader.

### 36. Connection Pooling
Reuse koneksi DB daripada buka-tutup tiap request. Perhatikan: max connections, min/idle connections, connection lifetime, idle timeout.

### 37. Read Replica
```
             ┌→ Primary DB (write)
API → Router ┤
             └→ Read Replica (read)
```
Trade-off: replication lag.

### 38. Database Replication
Primary → Replica. Pahami: synchronous vs asynchronous replication, replication lag, failover, read/write routing.

### 39. Sharding
```
Users 1–1M   → DB 1
Users 1M–2M  → DB 2
```
Pahami: shard key, consistent hashing, hot shard, cross-shard query, rebalancing. **Jangan buru-buru sharding kalau belum perlu.**

### 40. Database Security
```
Network → Authentication → Authorization → Encryption → Secrets → Monitoring
```

---

## Phase 4 — Caching

### 41. Why Cache?
Cache mengurangi: DB load, latency, biaya komputasi mahal.

### 42. Cache-Aside
```
Request → Check Cache → Hit → Return
                      → Miss → DB → Store in Cache → Return
```

### 43. Cache Invalidation
```
Update DB → Delete cache
```
*"There are only two hard things in Computer Science: cache invalidation and naming things."*

### 44. TTL (Time To Live)
Setelah expired → cache miss → fetch DB → cache lagi.

### 45. Cache Stampede
```
Cache expires → 1,000 request bersamaan → 1,000 DB query → DB overload
```
Solusi: lock, request coalescing, randomized TTL, background refresh, stale-while-revalidate.

### 46. Redis
Struktur data: String, Hash, List, Set, Sorted Set, TTL, Pub/Sub, distributed lock basic.
Use case: cache, session, rate limiting, distributed lock, queue sederhana, counter.

---

## Phase 5 — Message Queue & Async Processing

### 47. Why Message Queue?
```
Tanpa queue: API → Heavy Processing → Response (lambat)
Dengan queue: API → Queue → Worker → Heavy Processing (API respon cepat)
```

### 48. Producer / Consumer
```
API → Producer → Kafka/RabbitMQ/SQS → Consumer → Worker
```

### 49. At-Least-Once Delivery
Message bisa terkirim lebih dari sekali (misal consumer crash sebelum ACK). Karena itu **consumer harus idempotent**.

### 50. Idempotent Consumer
Sebelum proses, cek: "Apakah payment 123 udah pernah diproses?" Kalau ya → skip.

### 51. Retry (Queue)
```
Message → Consumer → Failure → Retry → Failure → Retry
```
Gunakan exponential backoff, retry limit, Dead Letter Queue.

### 52. Dead Letter Queue
Message yang gagal terus-menerus dipindah ke DLQ, biar bisa diinvestigasi tanpa nge-block message normal lainnya.

### 53. Kafka Basics
Topic, Partition, Producer, Consumer, Consumer Group, Offset.
```
Topic: orders → Partition 0, 1, 2
Consumer Group: Consumer A→P0, B→P1, C→P2
```
Benefit: high throughput, horizontal scaling, event streaming.

### 54. RabbitMQ Basics
```
Producer → Exchange (Direct/Topic/Fanout/Headers) → Queue → Consumer
```
Cocok buat traditional task/message processing.

---

## Phase 6 — Go Concurrency, Testing & Backend Performance

### 55. Goroutine
```go
go process()
```
Jangan bikin goroutine tanpa batas.

### 56. Channel
Komunikasi antar goroutine. Pahami buffered vs unbuffered channel, send, receive, close.

### 57. Mutex
```go
mutex.Lock()
counter++
mutex.Unlock()
```

### 58. Race Condition
Terjadi kalau operasi konkuren akses shared data tanpa sinkronisasi. Solusi: mutex, atomic operation, channel, proper ownership.

### 59. Deadlock in Go
Sama konsepnya kayak deadlock di DB, tapi di level goroutine. Prevention: consistent lock ordering, avoid unnecessary lock, context/timeout.

### 60. Worker Pool
```
Jobs → Queue → Worker 1, 2, 3, 4
```
Daripada bikin 1000 goroutine buat 1000 job, batasi jumlah worker. Manfaat: controlled concurrency, prevent resource exhaustion.

### 61. Context
`context.Context` handle cancellation, timeout, request-scoped values.
```
HTTP Request → Context → DB Query → External API
```
Kalau request dibatalkan → context cancelled → downstream work ikut dibatalkan.

### 62. Graceful Shutdown
```
SIGTERM → Stop accepting new request → Finish existing request
        → Stop workers → Close DB connection → Exit
```
Penting banget di Kubernetes.

### 63. Testing Fundamentals (Unit vs Integration vs E2E) — *new*
- **Unit test** → test satu fungsi/unit kecil secara isolasi (biasanya pakai mock dependency).
- **Integration test** → test interaksi antar komponen (misal: service + real/test DB).
- **E2E test** → test full flow dari luar, mensimulasikan user beneran.
- **Test pyramid**: banyak unit test di bawah, integration test secukupnya di tengah, E2E test sedikit di puncak (karena paling lambat & paling mahal maintain).
- **Contract testing** → penting kalau kerja dengan microservices, memastikan API contract antar service gak berubah tanpa sepengetahuan consumer-nya.

### 64. Table-Driven Tests & Mocking (Go) — *new*
```go
tests := []struct{
    name string
    input int
    want  int
}{
    {"positive", 2, 4},
    {"zero", 0, 0},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        got := double(tt.input)
        if got != tt.want { t.Errorf("got %d, want %d", got, tt.want) }
    })
}
```
- Table-driven test = pattern umum di Go buat test banyak kasus tanpa duplikasi kode.
- Mocking dependency (DB, external API) pakai interface + fake implementation, biar unit test gak bergantung ke resource asli.

---

## Phase 7 — Distributed Systems

### 65. Vertical vs Horizontal Scaling
Vertical → 1 server, upgrade CPU/RAM. Horizontal → tambah lebih banyak server. Horizontal scaling umumnya lebih disukai buat stateless API.

### 66. Stateless Service
Server gak bergantung ke local memory buat state user/session. State disimpan di DB/Redis/external storage, jadi request bisa masuk ke server manapun.

### 67. Load Balancer
```
              ┌→ Server A
Client → LB ──┼→ Server B
              └→ Server C
```
Pahami: round robin, least connections, health check, sticky session.

### 68. CAP Theorem
Saat network partition terjadi, sistem distributed harus pilih antara **Consistency** dan **Availability** (Partition Tolerance biasanya wajib ada).

### 69. Eventual Consistency
Data bisa beda sementara antar replica, tapi lama-lama konsisten. Berguna buat read replica, distributed system, event-driven architecture.

### 70. Distributed Lock
Dipakai kalau beberapa server harus koordinasi supaya cuma satu yang eksekusi critical task. Implementasi: Redis, Database, ZooKeeper.

### 71. Distributed Transaction
Hindari kalau bisa. Pertimbangkan: Saga pattern, event-driven architecture, compensation, idempotency.

### 72. Service-to-Service Communication (REST vs gRPC vs GraphQL) — *expanded*
- **REST** → simple, gampang di-debug, paling umum.
- **gRPC** → cepat (protobuf + HTTP/2), strong contract, cocok buat komunikasi internal antar service.
- **GraphQL** → client yang tentuin data apa yang mau diambil dalam satu request (menghindari over-fetching/under-fetching), cocok buat API yang dikonsumsi banyak client dengan kebutuhan data berbeda-beda. Trade-off: caching lebih rumit dibanding REST, dan butuh effort lebih di schema design.
- **Message Queue** → async, decoupled, retryable.

### 73. API Gateway — *new*
Single entry point di depan banyak microservices.
```
Client → API Gateway → Service A / Service B / Service C
```
Tanggung jawab umum: routing, authentication, rate limiting, request/response transformation, aggregation. Manfaat: client gak perlu tau alamat tiap service, cross-cutting concern (auth, logging) bisa dipusatkan.

### 74. Service Discovery
```
Order Service → service discovery → Payment Service
```
Di Kubernetes, Kubernetes Service udah menyediakan service discovery yang stabil.

---

## Phase 8 — System Design

### 75. Software Architecture Patterns — *new*
- **Layered / Clean Architecture / Hexagonal Architecture** → pisahkan business logic dari framework/DB/HTTP detail, biar gampang di-test dan di-maintain.
- **Repository pattern** → abstraksi akses data, business logic gak perlu tau detail query/DB.
- **Dependency Injection** → dependency (DB, HTTP client, dll) di-inject dari luar, bukan di-hardcode di dalam fungsi — memudahkan testing (bisa di-mock).
- Kenapa penting: tanpa ini, kode sulit di-unit-test dan sulit ganti implementasi (misal ganti DB atau cache provider) tanpa nulis ulang business logic.

### 76. Basic Architecture
```
Client → CDN/WAF → Load Balancer → API Servers → Redis → Database
                                        ↓
                                  Message Queue → Worker → DB/External Service
```

### 77. System Design Interview Framework
**Step 1 — Requirements**: apa fungsi sistem, siapa usernya, fitur utama, expected traffic, data size, latency & availability requirement.

**Step 2 — API Design**: definisikan endpoint, pikirkan auth, pagination, idempotency, error handling.

**Step 3 — Data Model**: entities, primary/foreign key, index, relationship, query pattern.

**Step 4 — Architecture**: gambar komponen (API, cache, DB, queue, worker).

**Step 5 — Scaling**: apa yang jadi bottleneck? API → horizontal scale. DB → index/replica. Cache → Redis. Heavy work → queue. External API → retry/timeout/circuit breaker.

**Step 6 — Reliability**: timeout, retry, circuit breaker, idempotency, health check, graceful degradation, failover, backup, DR.

**Step 7 — Security**: network → auth → authorization → encryption → secrets → monitoring.

---

## Phase 9 — Reliability & Production

### 78. Health Checks
**Liveness** → apakah aplikasi hidup? **Readiness** → apakah aplikasi siap terima traffic (DB connection ok, dependency sehat)?

### 79. Circuit Breaker
```
API → Payment Service → repeated failure → Circuit OPEN → stop sending request sementara
```
States: CLOSED → OPEN → HALF-OPEN.

### 80. Backpressure
```
Producer 10,000 msg/sec  vs  Consumer 1,000 msg/sec  → queue membengkak
```
Solusi: limit concurrency, queue, rate limiting, load shedding.

### 81. Graceful Degradation
Kalau service non-critical down, fungsi utama tetap jalan.
```
Recommendation Service DOWN → Checkout STILL WORKS
```

### 82. Backup & Disaster Recovery
**RPO** (Recovery Point Objective) → berapa banyak data yang boleh hilang? **RTO** (Recovery Time Objective) → berapa lama sistem boleh down?

---

## Phase 10 — Observability

### 83. Logs
Menjawab: **apa yang terjadi?** Gunakan structured logging, log level, correlation ID.

### 84. Metrics
Menjawab: **seberapa banyak/sering?** Contoh: request rate, error rate, latency, CPU, memory, DB connections, queue depth.

### 85. Tracing
Menjawab: **di mana request menghabiskan waktu?**
```
API 20ms → Auth 10ms → DB 300ms → External API 500ms
```

### 86. RED Method
**R**ate, **E**rrors, **D**uration — 3 metric inti buat monitor service.

### 87. Monitoring
Tools/konsep: Prometheus, Grafana, Datadog, OpenTelemetry, alerting.

---

## Phase 11 — Docker & Kubernetes

### 88. Docker
Image, container, Dockerfile, registry, volume, network, multi-stage build.
Security: jangan run as root, jangan hardcode secret, minimal base image, scan image, update dependency.

### 89. Kubernetes
Core concept: Pod, Deployment, Service, ConfigMap, Secret, Ingress, Namespace.
```
Deployment → Pods → Container
Service → stable network endpoint → Pods
```

### 90. Kubernetes Scaling
Horizontal Pod Autoscaler: traffic naik → CPU naik → pod nambah. Pahami: requests/limits, HPA, readiness/liveness probe, rolling deployment.

### 91. Kubernetes Security
RBAC, Service Account, Secret, Network Policy, namespace isolation, non-root container.

---

## Phase 12 — CI/CD & Deployment

### 92. CI/CD
```
Git Push → Test → Build → Security Scan → Docker Image → Deploy → Monitor
```

### 93. Deployment Strategies
- **Rolling**: ganti instance lama secara bertahap.
- **Blue-Green**: switch traffic dari environment lama (blue) ke baru (green) setelah divalidasi.
- **Canary**: kirim sebagian kecil traffic ke versi baru dulu, monitor, baru naikkan persentasenya.

---

## Phase 13 — Performance

### 94. API Performance
```
Measure first (logs/metrics/tracing) → identify bottleneck → optimize
```
Bottleneck potensial: DB, network, external API, CPU, memory, lock contention.

### 95. Database Performance
```
Slow query → EXPLAIN ANALYZE → index → query optimization → connection pool → read replica
```

### 96. Caching (Performance Angle)
Cocok dipakai kalau data sering dibaca **dan** gak terlalu sering berubah.

### 97. Load Testing
Metric penting: requests/sec, latency, P95, P99, throughput, error rate, saturation.
```
P50 → typical    P95 → 95% request lebih cepat dari ini    P99 → 99% request lebih cepat dari ini
```

---

## Phase 14 — Advanced Backend Concepts

### 98. Event-Driven Architecture
```
Order Service → OrderCreated Event → Message Broker → Payment/Notification/Analytics Service
```
Benefit: decoupling, async, scalability. Trade-off: eventual consistency, debugging lebih kompleks, duplicate event, ordering issue.

### 99. Saga Pattern
```
Create Order → Reserve Inventory → Charge Payment → Ship
```
Kalau payment gagal → Cancel Order + Release Inventory (compensating action).

### 100. Outbox Pattern
Masalah: update DB sukses tapi publish event gagal.
```
Transaction: Update business data + Insert event ke Outbox table
Worker: baca Outbox → publish event
```
Membuat update DB + event creation jadi atomic.

### 101. Distributed Idempotency
Penting untuk payment, order, queue, retry, webhook. Gunakan idempotency key, unique constraint, processed-event table.

### 102. Webhooks
```
Payment Provider → Webhook → API → Verify Signature → Process Event → Return 2xx
```
Pahami: sender, receiver, signature verification, retry, idempotency, duplicate event, timeout.

### 103. File Upload
Security checklist: file size limit, MIME type validation, extension validation, virus scan, random filename, object storage, jangan eksekusi file yang diupload, signed URL.

---

## Phase 15 — Interview Thinking

**"Bagaimana cara mencegah X?"** → Authentication → Authorization → Network → Validation → Encryption → Rate Limiting → Monitoring

**"Bagaimana cara scale X?"** → Identify bottleneck → Measure → Optimize → Cache → Horizontal scaling → Async processing → Database scaling

**"Apa yang terjadi kalau X gagal?"** → Timeout → Retry → Circuit Breaker → Fallback → Queue → Idempotency → Monitoring → Alerting

**"Bagaimana cara improve API yang lambat?"** → Logs → Metrics → Tracing → Find bottleneck → DB/Cache/Network/CPU → Optimize

**"Bagaimana cara cegah duplicate processing?"** → Idempotency Key → Unique Constraint → Processed Event → Database Transaction

**"Bagaimana cara cegah unauthorized access?"** → Authentication → Authorization → Least Privilege → Network Restriction → Monitoring

---

## Recommended Study Order

Jangan belajar 103 topik secara random. Ikuti level ini, nomor mengacu ke penomoran topik di atas (satu sistem penomoran, gak ada versi kedua lagi).

### Level 1 — Must Know
1, 3, 4, 5, 8, 9, 11, 10, 22, 12, 26, 28, 29, 30, 31, 36, 35, 46, 42, 47, 16, 63

*(Authentication vs Authorization, JWT, Access/Refresh Token, API Key, RBAC, Least Privilege, REST API, HTTP Status Codes, SQL Injection, API Documentation, Indexing, EXPLAIN ANALYZE, Transactions, ACID, Isolation Level, Connection Pooling, N+1 Query, Redis, Cache-Aside, Message Queue, Idempotency, Testing Fundamentals)*

### Level 2 — Strong Backend
32, 33, 34, 37, 38, 15, 17, 18, 79, 60, 55, 56, 57, 61, 62, 53, 49, 52, 69, 67, 66, 64, 24

*(Deadlock, Optimistic/Pessimistic Locking, Read Replica, Replication, Rate Limiting, Retry+Backoff, Timeout, Circuit Breaker, Worker Pool, Goroutine, Channel, Mutex, Context, Graceful Shutdown, Kafka, At-Least-Once Delivery, DLQ, Eventual Consistency, Load Balancer, Stateless Service, Table-Driven Test & Mocking, Encryption Fundamentals)*

### Level 3 — System Design
65, 68, 70, 74, 98, 99, 100, 71, 82, 75, 72, 73, 77

*(Horizontal Scaling, CAP Theorem, Distributed Lock, Service Discovery, Event-Driven Architecture, Saga Pattern, Outbox Pattern, Distributed Transactions, Backup/DR, Software Architecture Patterns, REST vs gRPC vs GraphQL, API Gateway, System Design Framework)*

### Level 4 — Production
88, 89, 91, 90, 93, 92, 40, 83, 84, 85, 87

*(Docker, Kubernetes, Kubernetes RBAC, HPA & Deployment Strategy, CI/CD, Secrets Management, Observability: Logs/Metrics/Tracing/Monitoring)*

### Level 5 — Advanced
102, 101, 103, 80, 39

*(Webhooks, Distributed Idempotency, File Upload Security, Backpressure, Sharding & Consistent Hashing)*

---

## How to Study Each Topic

Jangan cuma hafalin definisi. Jawab 6 pertanyaan ini untuk tiap topik:

1. **Apa itu?**
2. **Kenapa dibutuhkan?**
3. **Gimana cara kerjanya?**
4. **Masalah apa yang diselesaikan?**
5. **Apa trade-off-nya?**
6. **Kapan gue pakai ini?**

Contoh — Redis:
- **Apa?** → In-memory data store.
- **Kenapa?** → Mengurangi latency dan beban DB.
- **Gimana?** → Simpan data yang sering diakses di memory.
- **Masalah?** → DB read yang mahal/sering.
- **Trade-off?** → Infrastruktur tambahan + cache invalidation + stale data.
- **Kapan?** → Data yang sering diakses dan sedikit staleness masih bisa ditoleransi.

---

## Practical Project to Connect Everything

Bangun satu backend Go production-style:

```
                    Client
                       ↓
                  Load Balancer
                       ↓
                  Go API Server
                  /           \
                 ↓             ↓
              Redis          PostgreSQL
                 ↑
                 |
              Cache

Go API → Message Queue → Worker Pool → Background Processing
```

Implementasikan:
- **Authentication**: login, password hashing, JWT, access/refresh token
- **Authorization**: RBAC, permission, resource ownership
- **Database**: PostgreSQL, index, transaction, optimistic locking, connection pool
- **Cache**: Redis, cache-aside, TTL, rate limiting
- **Async**: message queue, worker, retry, DLQ, idempotent consumer
- **Testing**: unit test tiap layer, integration test untuk DB/queue, table-driven test
- **Security**: input validation, SQL injection prevention, rate limiting, secrets management, TLS
- **Production**: Docker, Kubernetes, health check, graceful shutdown, logging, metrics, tracing

---

## Resources

**Security**
- OWASP Top 10 — https://owasp.org/www-project-top-ten/
- PortSwigger Web Security Academy — https://portswigger.net/web-security
- OWASP Web Security Testing Guide — https://owasp.org/www-project-web-security-testing-guide/

**Go**
- Go documentation — https://go.dev/doc/
- Go blog — https://go.dev/blog/

**PostgreSQL**
- PostgreSQL Documentation — https://www.postgresql.org/docs/
- Fokus: Indexes, EXPLAIN, Transactions, Isolation, Locks, Performance

**Redis**
- Redis Documentation — https://redis.io/docs/
- Fokus: Data structures, TTL, Caching, Distributed locks, Pub/Sub

**Kubernetes**
- Kubernetes Documentation — https://kubernetes.io/docs/
- Fokus: Pod, Deployment, Service, Ingress, ConfigMap, Secret, RBAC, HPA, Probes

**API Design & Docs**
- OpenAPI Specification — https://swagger.io/specification/

---

## Final Mental Model

```
REQUEST
   ↓
AUTHENTICATION
   ↓
AUTHORIZATION
   ↓
VALIDATION
   ↓
BUSINESS LOGIC
   ↓
CACHE / DATABASE
   ↓
ASYNC PROCESSING
   ↓
RESPONSE
```

Untuk production:
```
             SECURITY
                ↓
Client → API → Service → DB
          ↓       ↓       ↓
       Metrics  Queue   Backup
          ↓       ↓
       Logging  Worker
          ↓
       Tracing
```

Untuk setiap komponen, selalu tanyakan:
- Bagaimana cara scale-nya?
- Bagaimana cara gagalnya?
- Bagaimana cara mengamankannya?
- Bagaimana cara monitoring-nya?
- Bagaimana cara recovery kalau gagal?
- Apa trade-off-nya?
