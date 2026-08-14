# Backend Engineering Syllabus (roadmap-backend/) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Populate `roadmap-backend/` with the roadmap README and 15 phase syllabus files (103 topics total), all Go/Node.js code examples drawn from one consistent fictional project ("OrderFlow").

**Architecture:** This is a content-generation plan, not a code feature. Each "task" dispatches one subagent (via the Agent tool) that writes one complete markdown file directly to disk, then the orchestrating session runs a deterministic grep-based structural check (the "test") against that file before declaring the task done. Tasks run strictly in order because each phase's code examples build on function/struct signatures introduced in earlier phases.

**Tech Stack:** Markdown content only. Referenced (not built) stacks inside the content: Go (`net/http`/`chi`, `pgx`, `golang-jwt/jwt`, `golang.org/x/crypto/bcrypt`, `go-redis/redis`) and Node.js (Express, `jsonwebtoken`, `bcrypt`, `pg`, `ioredis`).

## Global Constraints

- Spec: `docs/superpowers/specs/2026-08-14-backend-syllabus-design.md` — every constraint below is copied from it verbatim in intent.
- Language: casual Bahasa Indonesia mixed with untranslated technical terms (goroutine, connection pool, dll). Do not force-translate technical jargon.
- Per-topic structure is mandatory for every topic except the explicitly-listed conceptual-only topics (see each task): `## <N>. <Topic Name>` followed by `### Apa itu?`, `### Kenapa dibutuhkan?`, `### Cara Kerja`, `### Contoh Kode — Go`, `### Contoh Kode — Node.js`, `### Trade-off & Pitfall`, `### Kapan Dipakai`, `### Sering Ditanya Saat Interview` (in that order).
- Conceptual-only topics (skip the two code sections, keep the other six): topic 68 (CAP Theorem), topics 76 and 77 (Basic Architecture, System Design Interview Framework), and all of Phase 15's six frameworks.
- Every phase file (`phase-01` … `phase-15`) opens with:
  ```markdown
  # Phase XX — <Phase Name>

  > Bagian dari [Backend Engineer Roadmap](../README.md)

  ---
  ```
- Every phase file except `phase-15` closes with:
  ```markdown
  ---

  **Selanjutnya:** [Phase XX+1 — <Next Phase Name>](./phase-XX+1-xxx.md)
  ```
- Code examples are real, valid, idiomatic Go and Node.js — never pseudocode.
- All code across all 15 files belongs to one fictional project, **OrderFlow** (an e-commerce order API), so later phases extend earlier phases' functions instead of introducing disconnected snippets.

### OrderFlow domain

Entities: `User` (id, email, password_hash, role), `Product` (id, name, price, stock), `Order` (id, user_id, status, total, version), `OrderItem` (order_id, product_id, qty, price), `Payment` (id, order_id, status, idempotency_key).

### OrderFlow API surface (fixed signatures — reuse these exact names across phases)

| Introduced in | Go | Node.js |
|---|---|---|
| Phase 1 | `HashPassword(password string) (string, error)` | `hashPassword(password)` |
| Phase 1 | `CheckPasswordHash(password, hash string) bool` | `checkPasswordHash(password, hash)` |
| Phase 1 | `GenerateAccessToken(userID int64, role string) (string, error)` | `generateAccessToken(userId, role)` |
| Phase 1 | `GenerateRefreshToken(userID int64) (string, error)` | `generateRefreshToken(userId)` |
| Phase 1 | `ValidateToken(tokenString string) (*Claims, error)` | `validateToken(token)` |
| Phase 2 | `RateLimitMiddleware(rdb *redis.Client, limit int, window time.Duration) func(http.Handler) http.Handler` | `rateLimitMiddleware(redisClient, limit, windowMs)` |
| Phase 2 | `IdempotencyMiddleware(rdb *redis.Client) func(http.Handler) http.Handler` | `idempotencyMiddleware(redisClient)` |
| Phase 3 | `GetProductByID(ctx context.Context, db *pgxpool.Pool, id int64) (*Product, error)` | `getProductById(pool, id)` |
| Phase 3 | `CreateOrder(ctx context.Context, db *pgxpool.Pool, userID int64, items []OrderItem) (*Order, error)` | `createOrder(pool, userId, items)` |
| Phase 4 | `GetProductCached(ctx context.Context, rdb *redis.Client, db *pgxpool.Pool, id int64) (*Product, error)` | `getProductCached(redisClient, pool, id)` |
| Phase 5 | `PublishOrderCreated(ctx context.Context, w *kafka.Writer, order *Order) error` | `publishOrderCreated(producer, order)` |
| Phase 5 | `ConsumeOrderEvents(ctx context.Context, r *kafka.Reader, handler func(OrderEvent) error) error` | `consumeOrderEvents(consumer, handler)` |
| Phase 5 | `IsEventProcessed(ctx context.Context, rdb *redis.Client, eventID string) (bool, error)` | `isEventProcessed(redisClient, eventId)` |
| Phase 6 | `ProcessOrdersWorkerPool(ctx context.Context, jobs <-chan Order, workers int) <-chan Result` | `processOrdersWorkerPool(jobsQueue, workerCount)` |
| Phase 6 | `GracefulShutdown(ctx context.Context, server *http.Server, pool *WorkerPool) error` | `gracefulShutdown(server, pool)` |
| Phase 7 | `AcquireDistributedLock(ctx context.Context, rdb *redis.Client, key string, ttl time.Duration) (bool, error)` | `acquireDistributedLock(redisClient, key, ttlMs)` |
| Phase 8 | `type OrderRepository interface { GetByID(ctx, id) (*Order, error); Create(ctx, order *Order) error }` | `class OrderRepository { getById(id) {}; create(order) {} }` |
| Phase 9 | `CallPaymentProviderWithCircuitBreaker(ctx context.Context, req PaymentRequest) (*PaymentResponse, error)` | `callPaymentProviderWithCircuitBreaker(req)` |
| Phase 9 | `LivenessHandler(w http.ResponseWriter, r *http.Request)`, `ReadinessHandler(w http.ResponseWriter, r *http.Request)` | `livenessHandler(req, res)`, `readinessHandler(req, res)` |
| Phase 10 | `RequestMetricsMiddleware(next http.Handler) http.Handler` | `requestMetricsMiddleware(req, res, next)` |
| Phase 14 | `SaveOrderWithOutbox(ctx context.Context, db *pgxpool.Pool, order *Order) error` | `saveOrderWithOutbox(pool, order)` |
| Phase 14 | `VerifyWebhookSignature(payload []byte, signature, secret string) bool` | `verifyWebhookSignature(payload, signature, secret)` |
| Phase 14 | `UploadProductImage(ctx context.Context, file io.Reader, filename string) (string, error)` | `uploadProductImage(fileBuffer, filename)` |

Each task's subagent prompt tells it which of these signatures already exist (so it can reference/reuse them in prose and code) and which ones it is responsible for introducing.

### Verification pattern (the "test" for every phase task)

Each task runs the same shape of check, parameterized by that phase's topic list and conceptual-skip count:

```bash
FILE="roadmap-backend/syllabus/phase-XX-name.md"
test -f "$FILE" && echo "file exists" || echo "MISSING FILE"
head -1 "$FILE"                                    # expect: # Phase XX — <Name>
grep -q '> Bagian dari \[Backend Engineer Roadmap\](../README.md)' "$FILE" && echo "byline OK" || echo "MISSING byline"
for t in "${TOPICS[@]}"; do grep -qF "## $t" "$FILE" || echo "MISSING TOPIC: $t"; done
echo "Apa itu?: $(grep -c '^### Apa itu?' "$FILE") / expect ${#TOPICS[@]}"
echo "Kenapa dibutuhkan?: $(grep -c '^### Kenapa dibutuhkan?' "$FILE") / expect ${#TOPICS[@]}"
echo "Cara Kerja: $(grep -c '^### Cara Kerja' "$FILE") / expect ${#TOPICS[@]}"
echo "Trade-off & Pitfall: $(grep -c '^### Trade-off & Pitfall' "$FILE") / expect ${#TOPICS[@]}"
echo "Kapan Dipakai: $(grep -c '^### Kapan Dipakai' "$FILE") / expect ${#TOPICS[@]}"
echo "Interview: $(grep -c '^### Sering Ditanya Saat Interview' "$FILE") / expect ${#TOPICS[@]}"
echo "Go code: $(grep -c '^### Contoh Kode — Go' "$FILE") / expect $(( ${#TOPICS[@]} - CONCEPTUAL_SKIP ))"
echo "Node code: $(grep -c '^### Contoh Kode — Node.js' "$FILE") / expect $(( ${#TOPICS[@]} - CONCEPTUAL_SKIP ))"
grep -qi 'TBD\|TODO\|lorem ipsum' "$FILE" && echo "PLACEHOLDER FOUND" || echo "no placeholders"
```

Before the subagent runs, this fails (`MISSING FILE`). After it runs, every count must match and no `MISSING`/`PLACEHOLDER FOUND` lines may appear. If any check fails, re-dispatch the subagent for that phase with the specific gap called out (don't hand-patch — regenerate so the file stays coherent), then re-run the check.

---

### Task 0: Scaffold `roadmap-backend/` and write the README

**Files:**
- Create: `roadmap-backend/README.md`
- Create: `roadmap-backend/syllabus/` (directory, via the README commit)

**Interfaces:**
- Consumes: nothing.
- Produces: nothing (README has no code surface); establishes the directory Task 1 writes into.

- [ ] **Step 1: Write the failing check**

Run:
```bash
test -f roadmap-backend/README.md && echo "exists" || echo "MISSING"
```
Expected: `MISSING`

- [ ] **Step 2: Create the directory and the README**

```bash
mkdir -p roadmap-backend/syllabus
```

Write `roadmap-backend/README.md` with exactly this content (verbatim, no edits — this is the user-authored roadmap index):

```markdown
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
```

- [ ] **Step 3: Run the check again**

Run: `test -f roadmap-backend/README.md && echo "exists" || echo "MISSING"` and `ls roadmap-backend/syllabus`
Expected: `exists`, and the `syllabus` directory listed empty.

- [ ] **Step 4: Commit**

```bash
git add roadmap-backend/README.md
git commit -m "docs: add backend engineering roadmap README"
```

---

### Task 1: Phase 1 — Authentication & Authorization

**Files:**
- Create: `roadmap-backend/syllabus/phase-01-authentication-authorization.md`

**Interfaces:**
- Consumes: nothing (first phase).
- Produces: `HashPassword`/`hashPassword`, `CheckPasswordHash`/`checkPasswordHash`, `GenerateAccessToken`/`generateAccessToken`, `GenerateRefreshToken`/`generateRefreshToken`, `ValidateToken`/`validateToken` (see Global Constraints API surface table).

- [ ] **Step 1: Write the failing check**

```bash
FILE="roadmap-backend/syllabus/phase-01-authentication-authorization.md"
test -f "$FILE" && echo "exists" || echo "MISSING"
```
Expected: `MISSING`

- [ ] **Step 2: Dispatch the writing subagent**

Use the Agent tool (general-purpose, foreground) with this exact prompt:

```
Write roadmap-backend/syllabus/phase-01-authentication-authorization.md and save it with the Write tool. Do not paste the content back — just write the file.

CONTEXT — this is one phase file in a 15-phase backend engineering syllabus. Language: casual Bahasa Indonesia mixed with untranslated technical terms (goroutine, connection pool, dll — don't force-translate jargon).

SHARED FICTIONAL PROJECT (OrderFlow): all Go/Node.js code examples in this and every other phase belong to one e-commerce order API called OrderFlow. Entities: User (id, email, password_hash, role), Product (id, name, price, stock), Order (id, user_id, status, total, version), OrderItem (order_id, product_id, qty, price), Payment (id, order_id, status, idempotency_key). Go stack: net/http or chi router, pgx (Postgres), golang-jwt/jwt, golang.org/x/crypto/bcrypt, go-redis/redis. Node.js stack: Express, jsonwebtoken, bcrypt, pg, ioredis.

THIS PHASE introduces the OrderFlow auth layer. You are responsible for introducing these exact function signatures (use them verbatim so later phases can reuse them):
- Go: HashPassword(password string) (string, error) ; CheckPasswordHash(password, hash string) bool ; GenerateAccessToken(userID int64, role string) (string, error) ; GenerateRefreshToken(userID int64) (string, error) ; ValidateToken(tokenString string) (*Claims, error)
- Node.js: hashPassword(password) ; checkPasswordHash(password, hash) ; generateAccessToken(userId, role) ; generateRefreshToken(userId) ; validateToken(token)

FILE STRUCTURE — open with exactly:
# Phase 01 — Authentication & Authorization

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

Then one section per topic below, in order, each following EXACTLY this structure (all 8 subsections, in this order):
## <N>. <Topic Name>
### Apa itu?
### Kenapa dibutuhkan?
### Cara Kerja
### Contoh Kode — Go
### Contoh Kode — Node.js
### Trade-off & Pitfall
### Kapan Dipakai
### Sering Ditanya Saat Interview

TOPICS FOR THIS PHASE (all require both code sections — none are conceptual-only in this phase):
1. Authentication vs Authorization
2. Password Authentication (use HashPassword/hashPassword and CheckPasswordHash/checkPasswordHash here)
3. JWT (use GenerateAccessToken/generateAccessToken and ValidateToken/validateToken here)
4. Access Token vs Refresh Token (use GenerateRefreshToken/generateRefreshToken here, extend the JWT code from topic 3 rather than starting over)
5. API Key vs Access Token
6. Session vs JWT
7. OAuth 2.0
8. RBAC (Role-Based Access Control) (build on the role claim already carried by GenerateAccessToken)
9. Least Privilege

Code must be real, valid, idiomatic Go and Node.js — never pseudocode. Each topic's Go and Node.js examples should be doing the same thing (mirrored behavior, idiomatic per language).

End the file with:
---

**Selanjutnya:** [Phase 02 — API Security & Web Security](./phase-02-api-web-security.md)

Do not skip any topic. Do not write TBD/TODO placeholders — every subsection needs real content.
```

- [ ] **Step 3: Run the check again**

```bash
FILE="roadmap-backend/syllabus/phase-01-authentication-authorization.md"
TOPICS=(
  "1. Authentication vs Authorization" "2. Password Authentication" "3. JWT"
  "4. Access Token vs Refresh Token" "5. API Key vs Access Token" "6. Session vs JWT"
  "7. OAuth 2.0" "8. RBAC" "9. Least Privilege"
)
test -f "$FILE" && echo "exists" || echo "MISSING"
head -1 "$FILE"
grep -q '> Bagian dari \[Backend Engineer Roadmap\](../README.md)' "$FILE" && echo "byline OK" || echo "MISSING byline"
for t in "${TOPICS[@]}"; do grep -qF "## $t" "$FILE" || echo "MISSING TOPIC: $t"; done
for h in 'Apa itu?' 'Kenapa dibutuhkan?' 'Cara Kerja' 'Trade-off & Pitfall' 'Kapan Dipakai' 'Sering Ditanya Saat Interview'; do
  echo "$h: $(grep -c "^### $h" "$FILE") / expect 9"
done
echo "Go code: $(grep -c '^### Contoh Kode — Go' "$FILE") / expect 9"
echo "Node code: $(grep -c '^### Contoh Kode — Node.js' "$FILE") / expect 9"
grep -q '\*\*Selanjutnya:\*\*' "$FILE" && echo "footer OK" || echo "MISSING footer"
grep -qi 'TBD\|TODO\|lorem ipsum' "$FILE" && echo "PLACEHOLDER FOUND" || echo "no placeholders"
```
Expected: `exists`, byline OK, no `MISSING TOPIC` lines, all six subsection counts = 9, Go code = 9, Node code = 9, footer OK, "no placeholders". If any line fails, re-dispatch Step 2 calling out the specific gap, then re-run this check.

- [ ] **Step 4: Commit**

```bash
git add roadmap-backend/syllabus/phase-01-authentication-authorization.md
git commit -m "docs: add phase 01 syllabus (authentication & authorization)"
```

---

### Task 2: Phase 2 — API Security & Web Security

**Files:**
- Create: `roadmap-backend/syllabus/phase-02-api-web-security.md`

**Interfaces:**
- Consumes: `ValidateToken`/`validateToken` (Task 1) for the API Authentication topic.
- Produces: `RateLimitMiddleware`/`rateLimitMiddleware`, `IdempotencyMiddleware`/`idempotencyMiddleware`.

- [ ] **Step 1: Write the failing check**

```bash
test -f roadmap-backend/syllabus/phase-02-api-web-security.md && echo "exists" || echo "MISSING"
```
Expected: `MISSING`

- [ ] **Step 2: Dispatch the writing subagent**

```
Write roadmap-backend/syllabus/phase-02-api-web-security.md with the Write tool. Do not paste the content back.

CONTEXT & PROJECT: same OrderFlow project as before (entities: User, Product, Order, OrderItem, Payment; Go stack net/http|chi + pgx + golang-jwt/jwt + bcrypt + go-redis; Node.js stack Express + jsonwebtoken + bcrypt + pg + ioredis). Language: casual Bahasa Indonesia mixed with untranslated technical terms.

ALREADY EXISTS from Phase 1 (reuse/reference by name, don't redefine): Go ValidateToken(tokenString string) (*Claims, error), Node.js validateToken(token) — an auth middleware that calls this already protects OrderFlow's routes.

THIS PHASE introduces: Go RateLimitMiddleware(rdb *redis.Client, limit int, window time.Duration) func(http.Handler) http.Handler and IdempotencyMiddleware(rdb *redis.Client) func(http.Handler) http.Handler ; Node.js rateLimitMiddleware(redisClient, limit, windowMs) and idempotencyMiddleware(redisClient).

FILE STRUCTURE — open with exactly:
# Phase 02 — API Security & Web Security

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

Then one section per topic, each with all 8 subsections in order (## <N>. <Name> / ### Apa itu? / ### Kenapa dibutuhkan? / ### Cara Kerja / ### Contoh Kode — Go / ### Contoh Kode — Node.js / ### Trade-off & Pitfall / ### Kapan Dipakai / ### Sering Ditanya Saat Interview). ALL topics in this phase require both code sections.

TOPICS:
10. HTTP Fundamentals
11. REST API Design (use OrderFlow's /orders, /products, /users endpoints as the example)
12. API Documentation (OpenAPI/Swagger)
13. API Authentication (reference ValidateToken/validateToken from Phase 1)
14. API Authorization (IDOR/BOLA) (show the bug: checking auth but not resource ownership on GET /orders/:id)
15. Rate Limiting (introduce RateLimitMiddleware/rateLimitMiddleware here)
16. Idempotency (introduce IdempotencyMiddleware/idempotencyMiddleware here, applied to POST /payments)
17. Retry
18. Timeout
19. CORS
20. CSRF
21. XSS
22. SQL Injection (show a vulnerable OrderFlow query and the parameterized fix)
23. SSRF
24. Encryption Fundamentals

End the file with:
---

**Selanjutnya:** [Phase 03 — Database](./phase-03-database.md)

Real, valid, idiomatic Go and Node.js code only — no pseudocode, no TBD/TODO, no skipped topics.
```

- [ ] **Step 3: Run the check again**

```bash
FILE="roadmap-backend/syllabus/phase-02-api-web-security.md"
TOPICS=(
  "10. HTTP Fundamentals" "11. REST API Design" "12. API Documentation"
  "13. API Authentication" "14. API Authorization" "15. Rate Limiting"
  "16. Idempotency" "17. Retry" "18. Timeout" "19. CORS" "20. CSRF" "21. XSS"
  "22. SQL Injection" "23. SSRF" "24. Encryption Fundamentals"
)
test -f "$FILE" && echo "exists" || echo "MISSING"
grep -q '> Bagian dari \[Backend Engineer Roadmap\](../README.md)' "$FILE" && echo "byline OK" || echo "MISSING byline"
for t in "${TOPICS[@]}"; do grep -qF "## $t" "$FILE" || echo "MISSING TOPIC: $t"; done
for h in 'Apa itu?' 'Kenapa dibutuhkan?' 'Cara Kerja' 'Trade-off & Pitfall' 'Kapan Dipakai' 'Sering Ditanya Saat Interview'; do
  echo "$h: $(grep -c "^### $h" "$FILE") / expect 15"
done
echo "Go code: $(grep -c '^### Contoh Kode — Go' "$FILE") / expect 15"
echo "Node code: $(grep -c '^### Contoh Kode — Node.js' "$FILE") / expect 15"
grep -q '\*\*Selanjutnya:\*\*' "$FILE" && echo "footer OK" || echo "MISSING footer"
grep -qi 'TBD\|TODO\|lorem ipsum' "$FILE" && echo "PLACEHOLDER FOUND" || echo "no placeholders"
```
Expected: all counts = 15, no MISSING/PLACEHOLDER lines. Re-dispatch Step 2 with the gap named if anything fails.

- [ ] **Step 4: Commit**

```bash
git add roadmap-backend/syllabus/phase-02-api-web-security.md
git commit -m "docs: add phase 02 syllabus (API security & web security)"
```

---

### Task 3: Phase 3 — Database

**Files:**
- Create: `roadmap-backend/syllabus/phase-03-database.md`

**Interfaces:**
- Consumes: OrderFlow entities (User, Product, Order, OrderItem) from Global Constraints.
- Produces: `GetProductByID`/`getProductById`, `CreateOrder`/`createOrder`.

- [ ] **Step 1: Write the failing check**

```bash
test -f roadmap-backend/syllabus/phase-03-database.md && echo "exists" || echo "MISSING"
```
Expected: `MISSING`

- [ ] **Step 2: Dispatch the writing subagent**

```
Write roadmap-backend/syllabus/phase-03-database.md with the Write tool. Do not paste the content back.

CONTEXT & PROJECT: OrderFlow e-commerce order API. Entities: User (id, email, password_hash, role), Product (id, name, price, stock), Order (id, user_id, status, total, version), OrderItem (order_id, product_id, qty, price), Payment (id, order_id, status, idempotency_key). Go stack: pgx (Postgres). Node.js stack: pg. Language: casual Bahasa Indonesia mixed with untranslated technical terms.

THIS PHASE introduces the OrderFlow persistence layer: Go GetProductByID(ctx context.Context, db *pgxpool.Pool, id int64) (*Product, error) and CreateOrder(ctx context.Context, db *pgxpool.Pool, userID int64, items []OrderItem) (*Order, error) (CreateOrder must run inside a transaction that locks/decrements Product.stock — use this for the Transactions, Deadlock, Optimistic/Pessimistic Locking, and N+1 topics) ; Node.js getProductById(pool, id) and createOrder(pool, userId, items) with the same behavior.

FILE STRUCTURE — open with exactly:
# Phase 03 — Database

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

One section per topic, all 8 subsections in order (## <N>. <Name> / ### Apa itu? / ### Kenapa dibutuhkan? / ### Cara Kerja / ### Contoh Kode — Go / ### Contoh Kode — Node.js / ### Trade-off & Pitfall / ### Kapan Dipakai / ### Sering Ditanya Saat Interview). ALL topics in this phase require both code sections.

TOPICS:
25. Database Fundamentals (define the OrderFlow schema: users, products, orders, order_items, payments tables with SQL CREATE TABLE)
26. Indexing (index on orders.user_id and products.name)
27. Composite Index (index on orders(user_id, status))
28. EXPLAIN ANALYZE (run against a query on orders)
29. Transactions (show CreateOrder/createOrder's transaction wrapping stock decrement + order insert)
30. ACID
31. Isolation Levels
32. Deadlock (two concurrent CreateOrder calls locking Product rows out of order)
33. Optimistic Locking (use Order.version)
34. Pessimistic Locking (SELECT ... FOR UPDATE on Product.stock inside CreateOrder)
35. N+1 Query (fetching an order's items one-by-one vs a single JOIN)
36. Connection Pooling (pgx pool / pg Pool config)
37. Read Replica
38. Database Replication
39. Sharding
40. Database Security

End the file with:
---

**Selanjutnya:** [Phase 04 — Caching](./phase-04-caching.md)

Real, valid, idiomatic Go and Node.js code only — no pseudocode, no TBD/TODO, no skipped topics.
```

- [ ] **Step 3: Run the check again**

```bash
FILE="roadmap-backend/syllabus/phase-03-database.md"
TOPICS=(
  "25. Database Fundamentals" "26. Indexing" "27. Composite Index" "28. EXPLAIN ANALYZE"
  "29. Transactions" "30. ACID" "31. Isolation Levels" "32. Deadlock" "33. Optimistic Locking"
  "34. Pessimistic Locking" "35. N+1 Query" "36. Connection Pooling" "37. Read Replica"
  "38. Database Replication" "39. Sharding" "40. Database Security"
)
test -f "$FILE" && echo "exists" || echo "MISSING"
grep -q '> Bagian dari \[Backend Engineer Roadmap\](../README.md)' "$FILE" && echo "byline OK" || echo "MISSING byline"
for t in "${TOPICS[@]}"; do grep -qF "## $t" "$FILE" || echo "MISSING TOPIC: $t"; done
for h in 'Apa itu?' 'Kenapa dibutuhkan?' 'Cara Kerja' 'Trade-off & Pitfall' 'Kapan Dipakai' 'Sering Ditanya Saat Interview'; do
  echo "$h: $(grep -c "^### $h" "$FILE") / expect 16"
done
echo "Go code: $(grep -c '^### Contoh Kode — Go' "$FILE") / expect 16"
echo "Node code: $(grep -c '^### Contoh Kode — Node.js' "$FILE") / expect 16"
grep -q '\*\*Selanjutnya:\*\*' "$FILE" && echo "footer OK" || echo "MISSING footer"
grep -qi 'TBD\|TODO\|lorem ipsum' "$FILE" && echo "PLACEHOLDER FOUND" || echo "no placeholders"
```
Expected: all counts = 16, no MISSING/PLACEHOLDER lines.

- [ ] **Step 4: Commit**

```bash
git add roadmap-backend/syllabus/phase-03-database.md
git commit -m "docs: add phase 03 syllabus (database)"
```

---

### Task 4: Phase 4 — Caching

**Files:**
- Create: `roadmap-backend/syllabus/phase-04-caching.md`

**Interfaces:**
- Consumes: `GetProductByID`/`getProductById` (Task 3).
- Produces: `GetProductCached`/`getProductCached`.

- [ ] **Step 1: Write the failing check**

```bash
test -f roadmap-backend/syllabus/phase-04-caching.md && echo "exists" || echo "MISSING"
```
Expected: `MISSING`

- [ ] **Step 2: Dispatch the writing subagent**

```
Write roadmap-backend/syllabus/phase-04-caching.md with the Write tool. Do not paste the content back.

CONTEXT & PROJECT: OrderFlow e-commerce order API. Go stack: go-redis/redis + pgx. Node.js stack: ioredis + pg. Language: casual Bahasa Indonesia mixed with untranslated technical terms.

ALREADY EXISTS from Phase 3 (reuse by name): Go GetProductByID(ctx, db, id) (*Product, error), Node.js getProductById(pool, id).

THIS PHASE introduces: Go GetProductCached(ctx context.Context, rdb *redis.Client, db *pgxpool.Pool, id int64) (*Product, error) — a cache-aside wrapper around GetProductByID ; Node.js getProductCached(redisClient, pool, id) wrapping getProductById the same way.

FILE STRUCTURE — open with exactly:
# Phase 04 — Caching

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

One section per topic, all 8 subsections in order. ALL topics require both code sections.

TOPICS:
41. Why Cache?
42. Cache-Aside (introduce GetProductCached/getProductCached here, wrapping GetProductByID/getProductById)
43. Cache Invalidation (invalidate the product cache key when stock changes in CreateOrder)
44. TTL
45. Cache Stampede (product cache key expiring under concurrent load)
46. Redis

End the file with:
---

**Selanjutnya:** [Phase 05 — Message Queue & Async Processing](./phase-05-message-queue.md)

Real, valid, idiomatic Go and Node.js code only — no pseudocode, no TBD/TODO, no skipped topics.
```

- [ ] **Step 3: Run the check again**

```bash
FILE="roadmap-backend/syllabus/phase-04-caching.md"
TOPICS=("41. Why Cache?" "42. Cache-Aside" "43. Cache Invalidation" "44. TTL" "45. Cache Stampede" "46. Redis")
test -f "$FILE" && echo "exists" || echo "MISSING"
grep -q '> Bagian dari \[Backend Engineer Roadmap\](../README.md)' "$FILE" && echo "byline OK" || echo "MISSING byline"
for t in "${TOPICS[@]}"; do grep -qF "## $t" "$FILE" || echo "MISSING TOPIC: $t"; done
for h in 'Apa itu?' 'Kenapa dibutuhkan?' 'Cara Kerja' 'Trade-off & Pitfall' 'Kapan Dipakai' 'Sering Ditanya Saat Interview'; do
  echo "$h: $(grep -c "^### $h" "$FILE") / expect 6"
done
echo "Go code: $(grep -c '^### Contoh Kode — Go' "$FILE") / expect 6"
echo "Node code: $(grep -c '^### Contoh Kode — Node.js' "$FILE") / expect 6"
grep -q '\*\*Selanjutnya:\*\*' "$FILE" && echo "footer OK" || echo "MISSING footer"
grep -qi 'TBD\|TODO\|lorem ipsum' "$FILE" && echo "PLACEHOLDER FOUND" || echo "no placeholders"
```
Expected: all counts = 6, no MISSING/PLACEHOLDER lines.

- [ ] **Step 4: Commit**

```bash
git add roadmap-backend/syllabus/phase-04-caching.md
git commit -m "docs: add phase 04 syllabus (caching)"
```

---

### Task 5: Phase 5 — Message Queue & Async Processing

**Files:**
- Create: `roadmap-backend/syllabus/phase-05-message-queue.md`

**Interfaces:**
- Consumes: `CreateOrder`/`createOrder` (Task 3).
- Produces: `PublishOrderCreated`/`publishOrderCreated`, `ConsumeOrderEvents`/`consumeOrderEvents`, `IsEventProcessed`/`isEventProcessed`.

- [ ] **Step 1: Write the failing check**

```bash
test -f roadmap-backend/syllabus/phase-05-message-queue.md && echo "exists" || echo "MISSING"
```
Expected: `MISSING`

- [ ] **Step 2: Dispatch the writing subagent**

```
Write roadmap-backend/syllabus/phase-05-message-queue.md with the Write tool. Do not paste the content back.

CONTEXT & PROJECT: OrderFlow e-commerce order API. Go stack: segmentio/kafka-go (or a RabbitMQ client where the topic calls for it) + go-redis/redis. Node.js stack: kafkajs (or amqplib for RabbitMQ topics) + ioredis. Language: casual Bahasa Indonesia mixed with untranslated technical terms.

ALREADY EXISTS from Phase 3 (reuse by name): Go CreateOrder(ctx, db, userID, items) (*Order, error), Node.js createOrder(pool, userId, items) — an OrderCreated event should be published right after a successful CreateOrder call.

THIS PHASE introduces: Go PublishOrderCreated(ctx context.Context, w *kafka.Writer, order *Order) error and ConsumeOrderEvents(ctx context.Context, r *kafka.Reader, handler func(OrderEvent) error) error and IsEventProcessed(ctx context.Context, rdb *redis.Client, eventID string) (bool, error) ; Node.js publishOrderCreated(producer, order), consumeOrderEvents(consumer, handler), isEventProcessed(redisClient, eventId).

FILE STRUCTURE — open with exactly:
# Phase 05 — Message Queue & Async Processing

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

One section per topic, all 8 subsections in order. ALL topics require both code sections.

TOPICS:
47. Why Message Queue?
48. Producer / Consumer (introduce PublishOrderCreated/publishOrderCreated and ConsumeOrderEvents/consumeOrderEvents here)
49. At-Least-Once Delivery
50. Idempotent Consumer (introduce IsEventProcessed/isEventProcessed here, used inside the OrderCreated consumer to skip already-processed events)
51. Retry (Queue)
52. Dead Letter Queue
53. Kafka Basics
54. RabbitMQ Basics

End the file with:
---

**Selanjutnya:** [Phase 06 — Go Concurrency, Testing & Backend Performance](./phase-06-concurrency-testing-performance.md)

Real, valid, idiomatic Go and Node.js code only — no pseudocode, no TBD/TODO, no skipped topics.
```

- [ ] **Step 3: Run the check again**

```bash
FILE="roadmap-backend/syllabus/phase-05-message-queue.md"
TOPICS=("47. Why Message Queue?" "48. Producer / Consumer" "49. At-Least-Once Delivery" "50. Idempotent Consumer" "51. Retry (Queue)" "52. Dead Letter Queue" "53. Kafka Basics" "54. RabbitMQ Basics")
test -f "$FILE" && echo "exists" || echo "MISSING"
grep -q '> Bagian dari \[Backend Engineer Roadmap\](../README.md)' "$FILE" && echo "byline OK" || echo "MISSING byline"
for t in "${TOPICS[@]}"; do grep -qF "## $t" "$FILE" || echo "MISSING TOPIC: $t"; done
for h in 'Apa itu?' 'Kenapa dibutuhkan?' 'Cara Kerja' 'Trade-off & Pitfall' 'Kapan Dipakai' 'Sering Ditanya Saat Interview'; do
  echo "$h: $(grep -c "^### $h" "$FILE") / expect 8"
done
echo "Go code: $(grep -c '^### Contoh Kode — Go' "$FILE") / expect 8"
echo "Node code: $(grep -c '^### Contoh Kode — Node.js' "$FILE") / expect 8"
grep -q '\*\*Selanjutnya:\*\*' "$FILE" && echo "footer OK" || echo "MISSING footer"
grep -qi 'TBD\|TODO\|lorem ipsum' "$FILE" && echo "PLACEHOLDER FOUND" || echo "no placeholders"
```
Expected: all counts = 8, no MISSING/PLACEHOLDER lines.

- [ ] **Step 4: Commit**

```bash
git add roadmap-backend/syllabus/phase-05-message-queue.md
git commit -m "docs: add phase 05 syllabus (message queue & async processing)"
```

---

### Task 6: Phase 6 — Go Concurrency, Testing & Backend Performance

**Files:**
- Create: `roadmap-backend/syllabus/phase-06-concurrency-testing-performance.md`

**Interfaces:**
- Consumes: `PublishOrderCreated`/`publishOrderCreated` and `ConsumeOrderEvents`/`consumeOrderEvents` (Task 5) as the workload the worker pool processes.
- Produces: `ProcessOrdersWorkerPool`/`processOrdersWorkerPool`, `GracefulShutdown`/`gracefulShutdown`.

- [ ] **Step 1: Write the failing check**

```bash
test -f roadmap-backend/syllabus/phase-06-concurrency-testing-performance.md && echo "exists" || echo "MISSING"
```
Expected: `MISSING`

- [ ] **Step 2: Dispatch the writing subagent**

```
Write roadmap-backend/syllabus/phase-06-concurrency-testing-performance.md with the Write tool. Do not paste the content back.

CONTEXT & PROJECT: OrderFlow e-commerce order API. Language: casual Bahasa Indonesia mixed with untranslated technical terms. For Node.js topics that don't have a direct goroutine/channel equivalent, explain the closest real concept (Worker Threads, the event loop, EventEmitter/async queues, AbortController) rather than forcing a fake 1:1 mapping — say explicitly where the analogy breaks down.

ALREADY EXISTS from Phase 5 (reuse by name): Go ConsumeOrderEvents(ctx, r, handler) error, Node.js consumeOrderEvents(consumer, handler) — the OrderCreated events these consume are the workload processed by this phase's worker pool.

THIS PHASE introduces: Go ProcessOrdersWorkerPool(ctx context.Context, jobs <-chan Order, workers int) <-chan Result and GracefulShutdown(ctx context.Context, server *http.Server, pool *WorkerPool) error ; Node.js processOrdersWorkerPool(jobsQueue, workerCount) and gracefulShutdown(server, pool).

FILE STRUCTURE — open with exactly:
# Phase 06 — Go Concurrency, Testing & Backend Performance

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

One section per topic, all 8 subsections in order. ALL topics require both code sections (for Node.js, the code shows the closest real equivalent, clearly labeled as such, not a literal translation).

TOPICS:
55. Goroutine (Node.js: explain the Worker Threads / event loop equivalent)
56. Channel (Node.js: explain the EventEmitter/async queue equivalent)
57. Mutex
58. Race Condition
59. Deadlock in Go
60. Worker Pool (introduce ProcessOrdersWorkerPool/processOrdersWorkerPool here, consuming OrderCreated events via ConsumeOrderEvents/consumeOrderEvents)
61. Context (Node.js: AbortController/AbortSignal)
62. Graceful Shutdown (introduce GracefulShutdown/gracefulShutdown here)
63. Testing Fundamentals (Unit vs Integration vs E2E)
64. Table-Driven Tests & Mocking (Go: table-driven test example testing ProcessOrdersWorkerPool's job handling; Node.js: Jest/Vitest test table + mocking equivalent)

End the file with:
---

**Selanjutnya:** [Phase 07 — Distributed Systems](./phase-07-distributed-systems.md)

Real, valid, idiomatic Go and Node.js code only — no pseudocode, no TBD/TODO, no skipped topics.
```

- [ ] **Step 3: Run the check again**

```bash
FILE="roadmap-backend/syllabus/phase-06-concurrency-testing-performance.md"
TOPICS=("55. Goroutine" "56. Channel" "57. Mutex" "58. Race Condition" "59. Deadlock in Go" "60. Worker Pool" "61. Context" "62. Graceful Shutdown" "63. Testing Fundamentals" "64. Table-Driven Tests")
test -f "$FILE" && echo "exists" || echo "MISSING"
grep -q '> Bagian dari \[Backend Engineer Roadmap\](../README.md)' "$FILE" && echo "byline OK" || echo "MISSING byline"
for t in "${TOPICS[@]}"; do grep -qF "## $t" "$FILE" || echo "MISSING TOPIC: $t"; done
for h in 'Apa itu?' 'Kenapa dibutuhkan?' 'Cara Kerja' 'Trade-off & Pitfall' 'Kapan Dipakai' 'Sering Ditanya Saat Interview'; do
  echo "$h: $(grep -c "^### $h" "$FILE") / expect 10"
done
echo "Go code: $(grep -c '^### Contoh Kode — Go' "$FILE") / expect 10"
echo "Node code: $(grep -c '^### Contoh Kode — Node.js' "$FILE") / expect 10"
grep -q '\*\*Selanjutnya:\*\*' "$FILE" && echo "footer OK" || echo "MISSING footer"
grep -qi 'TBD\|TODO\|lorem ipsum' "$FILE" && echo "PLACEHOLDER FOUND" || echo "no placeholders"
```
Expected: all counts = 10, no MISSING/PLACEHOLDER lines.

- [ ] **Step 4: Commit**

```bash
git add roadmap-backend/syllabus/phase-06-concurrency-testing-performance.md
git commit -m "docs: add phase 06 syllabus (concurrency, testing & performance)"
```

---

### Task 7: Phase 7 — Distributed Systems

**Files:**
- Create: `roadmap-backend/syllabus/phase-07-distributed-systems.md`

**Interfaces:**
- Consumes: `RateLimitMiddleware`/`rateLimitMiddleware` and `IdempotencyMiddleware`/`idempotencyMiddleware` (Task 2, as prior Redis-backed примерs to build on for Distributed Lock).
- Produces: `AcquireDistributedLock`/`acquireDistributedLock`.

- [ ] **Step 1: Write the failing check**

```bash
test -f roadmap-backend/syllabus/phase-07-distributed-systems.md && echo "exists" || echo "MISSING"
```
Expected: `MISSING`

- [ ] **Step 2: Dispatch the writing subagent**

```
Write roadmap-backend/syllabus/phase-07-distributed-systems.md with the Write tool. Do not paste the content back.

CONTEXT & PROJECT: OrderFlow e-commerce order API. Go stack: go-redis/redis. Node.js stack: ioredis. Language: casual Bahasa Indonesia mixed with untranslated technical terms.

ALREADY EXISTS from Phase 2 (reuse by name as the pattern for Redis-backed coordination): Go RateLimitMiddleware(rdb, limit, window), Node.js rateLimitMiddleware(redisClient, limit, windowMs) — both already use Redis as shared state, same idea this phase extends into a lock.

THIS PHASE introduces: Go AcquireDistributedLock(ctx context.Context, rdb *redis.Client, key string, ttl time.Duration) (bool, error) (SET NX with TTL) ; Node.js acquireDistributedLock(redisClient, key, ttlMs) — used to make sure only one OrderFlow instance runs a scheduled stock-reconciliation job at a time.

Skip the two code sections ONLY for topic 68 (CAP Theorem) — it's purely conceptual. Every other topic needs both code sections.

FILE STRUCTURE — open with exactly:
# Phase 07 — Distributed Systems

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

One section per topic, 8 subsections in order for code topics, 6 subsections (skip the two Contoh Kode sections) for topic 68.

TOPICS:
65. Vertical vs Horizontal Scaling
66. Stateless Service (show how OrderFlow's JWT-based auth from Phase 1 already makes the API server stateless)
67. Load Balancer
68. CAP Theorem — CONCEPTUAL ONLY, skip both code sections
69. Eventual Consistency
70. Distributed Lock (introduce AcquireDistributedLock/acquireDistributedLock here)
71. Distributed Transaction
72. Service-to-Service Communication (REST vs gRPC vs GraphQL)
73. API Gateway
74. Service Discovery

End the file with:
---

**Selanjutnya:** [Phase 08 — System Design](./phase-08-system-design.md)

Real, valid, idiomatic Go and Node.js code only where required — no pseudocode, no TBD/TODO, no skipped topics.
```

- [ ] **Step 3: Run the check again**

```bash
FILE="roadmap-backend/syllabus/phase-07-distributed-systems.md"
TOPICS=("65. Vertical vs Horizontal Scaling" "66. Stateless Service" "67. Load Balancer" "68. CAP Theorem" "69. Eventual Consistency" "70. Distributed Lock" "71. Distributed Transaction" "72. Service-to-Service Communication" "73. API Gateway" "74. Service Discovery")
test -f "$FILE" && echo "exists" || echo "MISSING"
grep -q '> Bagian dari \[Backend Engineer Roadmap\](../README.md)' "$FILE" && echo "byline OK" || echo "MISSING byline"
for t in "${TOPICS[@]}"; do grep -qF "## $t" "$FILE" || echo "MISSING TOPIC: $t"; done
for h in 'Apa itu?' 'Kenapa dibutuhkan?' 'Cara Kerja' 'Trade-off & Pitfall' 'Kapan Dipakai' 'Sering Ditanya Saat Interview'; do
  echo "$h: $(grep -c "^### $h" "$FILE") / expect 10"
done
echo "Go code: $(grep -c '^### Contoh Kode — Go' "$FILE") / expect 9 (10 topics minus CAP Theorem)"
echo "Node code: $(grep -c '^### Contoh Kode — Node.js' "$FILE") / expect 9"
grep -q '\*\*Selanjutnya:\*\*' "$FILE" && echo "footer OK" || echo "MISSING footer"
grep -qi 'TBD\|TODO\|lorem ipsum' "$FILE" && echo "PLACEHOLDER FOUND" || echo "no placeholders"
```
Expected: subsection counts = 10, code counts = 9, no MISSING/PLACEHOLDER lines.

- [ ] **Step 4: Commit**

```bash
git add roadmap-backend/syllabus/phase-07-distributed-systems.md
git commit -m "docs: add phase 07 syllabus (distributed systems)"
```

---

### Task 8: Phase 8 — System Design

**Files:**
- Create: `roadmap-backend/syllabus/phase-08-system-design.md`

**Interfaces:**
- Consumes: whole OrderFlow architecture built across Tasks 1–7 (for the System Design Interview Framework worked example).
- Produces: `OrderRepository` interface / `OrderRepository` class (Go interface, Node.js class).

- [ ] **Step 1: Write the failing check**

```bash
test -f roadmap-backend/syllabus/phase-08-system-design.md && echo "exists" || echo "MISSING"
```
Expected: `MISSING`

- [ ] **Step 2: Dispatch the writing subagent**

```
Write roadmap-backend/syllabus/phase-08-system-design.md with the Write tool. Do not paste the content back.

CONTEXT & PROJECT: OrderFlow e-commerce order API (User, Product, Order, OrderItem, Payment entities; auth from Phase 1, DB layer from Phase 3, caching from Phase 4, queue from Phase 5, worker pool from Phase 6, distributed lock/gateway from Phase 7). Language: casual Bahasa Indonesia mixed with untranslated technical terms.

THIS PHASE introduces: Go `type OrderRepository interface { GetByID(ctx context.Context, id int64) (*Order, error); Create(ctx context.Context, order *Order) error }` with a Postgres implementation wrapping CreateOrder/GetProductByID from Phase 3 ; Node.js `class OrderRepository { async getById(id) {...}; async create(order) {...} }` doing the same.

Skip the two code sections for topics 76 and 77 — they're conceptual/diagram-only. Topic 75 (Software Architecture Patterns) DOES need code (the OrderRepository example above).

FILE STRUCTURE — open with exactly:
# Phase 08 — System Design

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

One section per topic. Topic 75 gets all 8 subsections; topics 76 and 77 get only the 6 non-code subsections (### Apa itu? / ### Kenapa dibutuhkan? / ### Cara Kerja / ### Trade-off & Pitfall / ### Kapan Dipakai / ### Sering Ditanya Saat Interview — skip the two Contoh Kode headers entirely for these two).

TOPICS:
75. Software Architecture Patterns (Clean/Hexagonal Architecture, Repository Pattern, Dependency Injection) — introduce OrderRepository here
76. Basic Architecture — CONCEPTUAL ONLY, skip both code sections. Use OrderFlow's full architecture (client → LB → API → Redis/Postgres → queue → worker) as the worked diagram.
77. System Design Interview Framework (7 steps) — CONCEPTUAL ONLY, skip both code sections. Walk through "design OrderFlow" as the worked example for all 7 steps.

End the file with:
---

**Selanjutnya:** [Phase 09 — Reliability & Production](./phase-09-reliability-production.md)

Real, valid, idiomatic Go and Node.js code only where required — no pseudocode, no TBD/TODO, no skipped topics.
```

- [ ] **Step 3: Run the check again**

```bash
FILE="roadmap-backend/syllabus/phase-08-system-design.md"
TOPICS=("75. Software Architecture Patterns" "76. Basic Architecture" "77. System Design Interview Framework")
test -f "$FILE" && echo "exists" || echo "MISSING"
grep -q '> Bagian dari \[Backend Engineer Roadmap\](../README.md)' "$FILE" && echo "byline OK" || echo "MISSING byline"
for t in "${TOPICS[@]}"; do grep -qF "## $t" "$FILE" || echo "MISSING TOPIC: $t"; done
for h in 'Apa itu?' 'Kenapa dibutuhkan?' 'Cara Kerja' 'Trade-off & Pitfall' 'Kapan Dipakai' 'Sering Ditanya Saat Interview'; do
  echo "$h: $(grep -c "^### $h" "$FILE") / expect 3"
done
echo "Go code: $(grep -c '^### Contoh Kode — Go' "$FILE") / expect 1 (only topic 75)"
echo "Node code: $(grep -c '^### Contoh Kode — Node.js' "$FILE") / expect 1"
grep -q '\*\*Selanjutnya:\*\*' "$FILE" && echo "footer OK" || echo "MISSING footer"
grep -qi 'TBD\|TODO\|lorem ipsum' "$FILE" && echo "PLACEHOLDER FOUND" || echo "no placeholders"
```
Expected: subsection counts = 3, code counts = 1, no MISSING/PLACEHOLDER lines.

- [ ] **Step 4: Commit**

```bash
git add roadmap-backend/syllabus/phase-08-system-design.md
git commit -m "docs: add phase 08 syllabus (system design)"
```

---

### Task 9: Phase 9 — Reliability & Production

**Files:**
- Create: `roadmap-backend/syllabus/phase-09-reliability-production.md`

**Interfaces:**
- Consumes: nothing required, but should call an external "payment provider" as OrderFlow's flaky dependency.
- Produces: `CallPaymentProviderWithCircuitBreaker`/`callPaymentProviderWithCircuitBreaker`, `LivenessHandler`/`livenessHandler`, `ReadinessHandler`/`readinessHandler`.

- [ ] **Step 1: Write the failing check**

```bash
test -f roadmap-backend/syllabus/phase-09-reliability-production.md && echo "exists" || echo "MISSING"
```
Expected: `MISSING`

- [ ] **Step 2: Dispatch the writing subagent**

```
Write roadmap-backend/syllabus/phase-09-reliability-production.md with the Write tool. Do not paste the content back.

CONTEXT & PROJECT: OrderFlow e-commerce order API. OrderFlow calls an external payment provider to charge orders — this is the flaky external dependency for this phase's examples. Language: casual Bahasa Indonesia mixed with untranslated technical terms.

THIS PHASE introduces: Go CallPaymentProviderWithCircuitBreaker(ctx context.Context, req PaymentRequest) (*PaymentResponse, error) and LivenessHandler(w http.ResponseWriter, r *http.Request), ReadinessHandler(w http.ResponseWriter, r *http.Request) (readiness checks the Postgres and Redis connections) ; Node.js callPaymentProviderWithCircuitBreaker(req), livenessHandler(req, res), readinessHandler(req, res).

FILE STRUCTURE — open with exactly:
# Phase 09 — Reliability & Production

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

One section per topic, all 8 subsections in order. ALL topics require both code sections.

TOPICS:
78. Health Checks (Liveness vs Readiness) (introduce LivenessHandler/livenessHandler and ReadinessHandler/readinessHandler here)
79. Circuit Breaker (introduce CallPaymentProviderWithCircuitBreaker/callPaymentProviderWithCircuitBreaker here)
80. Backpressure
81. Graceful Degradation (if the payment provider circuit is open, OrderFlow still lets users browse products/cart, just blocks checkout)
82. Backup & Disaster Recovery (RPO/RTO) (show a simple pg_dump-based backup script and restore command)

End the file with:
---

**Selanjutnya:** [Phase 10 — Observability](./phase-10-observability.md)

Real, valid, idiomatic Go and Node.js code only — no pseudocode, no TBD/TODO, no skipped topics.
```

- [ ] **Step 3: Run the check again**

```bash
FILE="roadmap-backend/syllabus/phase-09-reliability-production.md"
TOPICS=("78. Health Checks" "79. Circuit Breaker" "80. Backpressure" "81. Graceful Degradation" "82. Backup & Disaster Recovery")
test -f "$FILE" && echo "exists" || echo "MISSING"
grep -q '> Bagian dari \[Backend Engineer Roadmap\](../README.md)' "$FILE" && echo "byline OK" || echo "MISSING byline"
for t in "${TOPICS[@]}"; do grep -qF "## $t" "$FILE" || echo "MISSING TOPIC: $t"; done
for h in 'Apa itu?' 'Kenapa dibutuhkan?' 'Cara Kerja' 'Trade-off & Pitfall' 'Kapan Dipakai' 'Sering Ditanya Saat Interview'; do
  echo "$h: $(grep -c "^### $h" "$FILE") / expect 5"
done
echo "Go code: $(grep -c '^### Contoh Kode — Go' "$FILE") / expect 5"
echo "Node code: $(grep -c '^### Contoh Kode — Node.js' "$FILE") / expect 5"
grep -q '\*\*Selanjutnya:\*\*' "$FILE" && echo "footer OK" || echo "MISSING footer"
grep -qi 'TBD\|TODO\|lorem ipsum' "$FILE" && echo "PLACEHOLDER FOUND" || echo "no placeholders"
```
Expected: all counts = 5, no MISSING/PLACEHOLDER lines.

- [ ] **Step 4: Commit**

```bash
git add roadmap-backend/syllabus/phase-09-reliability-production.md
git commit -m "docs: add phase 09 syllabus (reliability & production)"
```

---

### Task 10: Phase 10 — Observability

**Files:**
- Create: `roadmap-backend/syllabus/phase-10-observability.md`

**Interfaces:**
- Consumes: `CreateOrder`/`createOrder` (Task 3) and `CallPaymentProviderWithCircuitBreaker`/`callPaymentProviderWithCircuitBreaker` (Task 9) as the instrumented code paths.
- Produces: `RequestMetricsMiddleware`/`requestMetricsMiddleware`.

- [ ] **Step 1: Write the failing check**

```bash
test -f roadmap-backend/syllabus/phase-10-observability.md && echo "exists" || echo "MISSING"
```
Expected: `MISSING`

- [ ] **Step 2: Dispatch the writing subagent**

```
Write roadmap-backend/syllabus/phase-10-observability.md with the Write tool. Do not paste the content back.

CONTEXT & PROJECT: OrderFlow e-commerce order API. Go stack: standard log/slog for structured logging, Prometheus client_golang for metrics, OpenTelemetry Go SDK for tracing. Node.js stack: pino for structured logging, prom-client for metrics, OpenTelemetry JS SDK for tracing. Language: casual Bahasa Indonesia mixed with untranslated technical terms.

ALREADY EXISTS (reuse by name as the instrumented paths): Go CreateOrder(ctx, db, userID, items) (*Order, error) and CallPaymentProviderWithCircuitBreaker(ctx, req) (*PaymentResponse, error) ; Node.js createOrder(pool, userId, items) and callPaymentProviderWithCircuitBreaker(req).

THIS PHASE introduces: Go RequestMetricsMiddleware(next http.Handler) http.Handler (records RED metrics per request) ; Node.js requestMetricsMiddleware(req, res, next).

FILE STRUCTURE — open with exactly:
# Phase 10 — Observability

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

One section per topic, all 8 subsections in order. ALL topics require both code sections — for topic 87 (Monitoring), the code sections should be instrumentation snippets (Prometheus counter/histogram in Go, prom-client equivalent in Node.js) rather than full infra setup, per the roadmap's own note that infra setup isn't needed here.

TOPICS:
83. Logs (structured logging + correlation ID, applied to CreateOrder/createOrder)
84. Metrics (introduce RequestMetricsMiddleware/requestMetricsMiddleware here)
85. Tracing (trace a request through CreateOrder → CallPaymentProviderWithCircuitBreaker)
86. RED Method
87. Monitoring (Prometheus, Grafana, OpenTelemetry — instrumentation code only, not full infra config)

End the file with:
---

**Selanjutnya:** [Phase 11 — Docker & Kubernetes](./phase-11-docker-kubernetes.md)

Real, valid, idiomatic Go and Node.js code only — no pseudocode, no TBD/TODO, no skipped topics.
```

- [ ] **Step 3: Run the check again**

```bash
FILE="roadmap-backend/syllabus/phase-10-observability.md"
TOPICS=("83. Logs" "84. Metrics" "85. Tracing" "86. RED Method" "87. Monitoring")
test -f "$FILE" && echo "exists" || echo "MISSING"
grep -q '> Bagian dari \[Backend Engineer Roadmap\](../README.md)' "$FILE" && echo "byline OK" || echo "MISSING byline"
for t in "${TOPICS[@]}"; do grep -qF "## $t" "$FILE" || echo "MISSING TOPIC: $t"; done
for h in 'Apa itu?' 'Kenapa dibutuhkan?' 'Cara Kerja' 'Trade-off & Pitfall' 'Kapan Dipakai' 'Sering Ditanya Saat Interview'; do
  echo "$h: $(grep -c "^### $h" "$FILE") / expect 5"
done
echo "Go code: $(grep -c '^### Contoh Kode — Go' "$FILE") / expect 5"
echo "Node code: $(grep -c '^### Contoh Kode — Node.js' "$FILE") / expect 5"
grep -q '\*\*Selanjutnya:\*\*' "$FILE" && echo "footer OK" || echo "MISSING footer"
grep -qi 'TBD\|TODO\|lorem ipsum' "$FILE" && echo "PLACEHOLDER FOUND" || echo "no placeholders"
```
Expected: all counts = 5, no MISSING/PLACEHOLDER lines.

- [ ] **Step 4: Commit**

```bash
git add roadmap-backend/syllabus/phase-10-observability.md
git commit -m "docs: add phase 10 syllabus (observability)"
```

---

### Task 11: Phase 11 — Docker & Kubernetes

**Files:**
- Create: `roadmap-backend/syllabus/phase-11-docker-kubernetes.md`

**Interfaces:**
- Consumes: OrderFlow as a whole (both Go and Node.js implementations get containerized/deployed).
- Produces: nothing new in the API-surface table (this phase produces Dockerfiles/YAML manifests, not functions).

- [ ] **Step 1: Write the failing check**

```bash
test -f roadmap-backend/syllabus/phase-11-docker-kubernetes.md && echo "exists" || echo "MISSING"
```
Expected: `MISSING`

- [ ] **Step 2: Dispatch the writing subagent**

```
Write roadmap-backend/syllabus/phase-11-docker-kubernetes.md with the Write tool. Do not paste the content back.

CONTEXT & PROJECT: OrderFlow e-commerce order API, with a Go build (binary orderflow-go) and a Node.js build (orderflow-node). Language: casual Bahasa Indonesia mixed with untranslated technical terms.

For this phase, "Contoh Kode — Go" and "Contoh Kode — Node.js" hold that topic's Dockerfile/YAML AS APPLIED to the Go build vs the Node.js build respectively (e.g. different base image, build steps, container port, image name) — both should be complete, valid manifests, not identical copies.

FILE STRUCTURE — open with exactly:
# Phase 11 — Docker & Kubernetes

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

One section per topic, all 8 subsections in order. ALL topics require both code sections.

TOPICS:
88. Docker (multi-stage Dockerfile for orderflow-go and a separate one for orderflow-node; cover non-root user, minimal base image)
89. Kubernetes (Pod, Deployment, Service, ConfigMap, Secret, Ingress, Namespace — YAML for deploying orderflow-go vs orderflow-node)
90. Kubernetes Scaling (HorizontalPodAutoscaler YAML for both deployments)
91. Kubernetes Security (RBAC, Service Account, Network Policy YAML for both deployments)

End the file with:
---

**Selanjutnya:** [Phase 12 — CI/CD & Deployment](./phase-12-cicd-deployment.md)

Real, valid YAML/Dockerfile syntax only — no pseudocode, no TBD/TODO, no skipped topics.
```

- [ ] **Step 3: Run the check again**

```bash
FILE="roadmap-backend/syllabus/phase-11-docker-kubernetes.md"
TOPICS=("88. Docker" "89. Kubernetes" "90. Kubernetes Scaling" "91. Kubernetes Security")
test -f "$FILE" && echo "exists" || echo "MISSING"
grep -q '> Bagian dari \[Backend Engineer Roadmap\](../README.md)' "$FILE" && echo "byline OK" || echo "MISSING byline"
for t in "${TOPICS[@]}"; do grep -qF "## $t" "$FILE" || echo "MISSING TOPIC: $t"; done
for h in 'Apa itu?' 'Kenapa dibutuhkan?' 'Cara Kerja' 'Trade-off & Pitfall' 'Kapan Dipakai' 'Sering Ditanya Saat Interview'; do
  echo "$h: $(grep -c "^### $h" "$FILE") / expect 4"
done
echo "Go code: $(grep -c '^### Contoh Kode — Go' "$FILE") / expect 4"
echo "Node code: $(grep -c '^### Contoh Kode — Node.js' "$FILE") / expect 4"
grep -q '\*\*Selanjutnya:\*\*' "$FILE" && echo "footer OK" || echo "MISSING footer"
grep -qi 'TBD\|TODO\|lorem ipsum' "$FILE" && echo "PLACEHOLDER FOUND" || echo "no placeholders"
```
Expected: all counts = 4, no MISSING/PLACEHOLDER lines.

- [ ] **Step 4: Commit**

```bash
git add roadmap-backend/syllabus/phase-11-docker-kubernetes.md
git commit -m "docs: add phase 11 syllabus (docker & kubernetes)"
```

---

### Task 12: Phase 12 — CI/CD & Deployment

**Files:**
- Create: `roadmap-backend/syllabus/phase-12-cicd-deployment.md`

**Interfaces:**
- Consumes: Dockerfiles for `orderflow-go`/`orderflow-node` (Task 11).
- Produces: nothing new in the API-surface table.

- [ ] **Step 1: Write the failing check**

```bash
test -f roadmap-backend/syllabus/phase-12-cicd-deployment.md && echo "exists" || echo "MISSING"
```
Expected: `MISSING`

- [ ] **Step 2: Dispatch the writing subagent**

```
Write roadmap-backend/syllabus/phase-12-cicd-deployment.md with the Write tool. Do not paste the content back.

CONTEXT & PROJECT: OrderFlow e-commerce order API, with orderflow-go and orderflow-node Dockerfiles already defined in Phase 11. Language: casual Bahasa Indonesia mixed with untranslated technical terms.

FILE STRUCTURE — open with exactly:
# Phase 12 — CI/CD & Deployment

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

One section per topic, all 8 subsections in order. ALL topics require both code sections.

TOPICS:
92. CI/CD (a GitHub Actions workflow YAML for orderflow-go: test → build → docker push; a separate one for orderflow-node with the same stages)
93. Deployment Strategies (Rolling, Blue-Green, Canary — explain each, then show a Kubernetes Deployment YAML snippet configuring a rolling update strategy, and a brief canary example using two Deployments + a weighted Service/Ingress split, for both orderflow-go and orderflow-node)

End the file with:
---

**Selanjutnya:** [Phase 13 — Performance](./phase-13-performance.md)

Real, valid YAML syntax only — no pseudocode, no TBD/TODO, no skipped topics.
```

- [ ] **Step 3: Run the check again**

```bash
FILE="roadmap-backend/syllabus/phase-12-cicd-deployment.md"
TOPICS=("92. CI/CD" "93. Deployment Strategies")
test -f "$FILE" && echo "exists" || echo "MISSING"
grep -q '> Bagian dari \[Backend Engineer Roadmap\](../README.md)' "$FILE" && echo "byline OK" || echo "MISSING byline"
for t in "${TOPICS[@]}"; do grep -qF "## $t" "$FILE" || echo "MISSING TOPIC: $t"; done
for h in 'Apa itu?' 'Kenapa dibutuhkan?' 'Cara Kerja' 'Trade-off & Pitfall' 'Kapan Dipakai' 'Sering Ditanya Saat Interview'; do
  echo "$h: $(grep -c "^### $h" "$FILE") / expect 2"
done
echo "Go code: $(grep -c '^### Contoh Kode — Go' "$FILE") / expect 2"
echo "Node code: $(grep -c '^### Contoh Kode — Node.js' "$FILE") / expect 2"
grep -q '\*\*Selanjutnya:\*\*' "$FILE" && echo "footer OK" || echo "MISSING footer"
grep -qi 'TBD\|TODO\|lorem ipsum' "$FILE" && echo "PLACEHOLDER FOUND" || echo "no placeholders"
```
Expected: all counts = 2, no MISSING/PLACEHOLDER lines.

- [ ] **Step 4: Commit**

```bash
git add roadmap-backend/syllabus/phase-12-cicd-deployment.md
git commit -m "docs: add phase 12 syllabus (ci/cd & deployment)"
```

---

### Task 13: Phase 13 — Performance

**Files:**
- Create: `roadmap-backend/syllabus/phase-13-performance.md`

**Interfaces:**
- Consumes: `GetProductCached`/`getProductCached` (Task 4), `CreateOrder`/`createOrder` (Task 3).
- Produces: nothing new in the API-surface table.

- [ ] **Step 1: Write the failing check**

```bash
test -f roadmap-backend/syllabus/phase-13-performance.md && echo "exists" || echo "MISSING"
```
Expected: `MISSING`

- [ ] **Step 2: Dispatch the writing subagent**

```
Write roadmap-backend/syllabus/phase-13-performance.md with the Write tool. Do not paste the content back.

CONTEXT & PROJECT: OrderFlow e-commerce order API. Language: casual Bahasa Indonesia mixed with untranslated technical terms.

ALREADY EXISTS (reuse by name for the perf-tuning examples): Go GetProductByID(ctx, db, id), GetProductCached(ctx, rdb, db, id), CreateOrder(ctx, db, userID, items) ; Node.js getProductById(pool, id), getProductCached(redisClient, pool, id), createOrder(pool, userId, items).

FILE STRUCTURE — open with exactly:
# Phase 13 — Performance

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

One section per topic, all 8 subsections in order. ALL topics require both code sections.

TOPICS:
94. API Performance (profile the POST /orders endpoint that calls CreateOrder/createOrder)
95. Database Performance (apply EXPLAIN ANALYZE + indexing from Phase 3 to a slow orders query)
96. Caching (Performance Angle) (measure GetProductByID vs GetProductCached / getProductById vs getProductCached latency)
97. Load Testing (a k6 script load-testing GET /products/:id and POST /orders)

End the file with:
---

**Selanjutnya:** [Phase 14 — Advanced Backend Concepts](./phase-14-advanced-concepts.md)

Real, valid, idiomatic Go and Node.js code (plus a real k6 script for topic 97) — no pseudocode, no TBD/TODO, no skipped topics.
```

- [ ] **Step 3: Run the check again**

```bash
FILE="roadmap-backend/syllabus/phase-13-performance.md"
TOPICS=("94. API Performance" "95. Database Performance" "96. Caching" "97. Load Testing")
test -f "$FILE" && echo "exists" || echo "MISSING"
grep -q '> Bagian dari \[Backend Engineer Roadmap\](../README.md)' "$FILE" && echo "byline OK" || echo "MISSING byline"
for t in "${TOPICS[@]}"; do grep -qF "## $t" "$FILE" || echo "MISSING TOPIC: $t"; done
for h in 'Apa itu?' 'Kenapa dibutuhkan?' 'Cara Kerja' 'Trade-off & Pitfall' 'Kapan Dipakai' 'Sering Ditanya Saat Interview'; do
  echo "$h: $(grep -c "^### $h" "$FILE") / expect 4"
done
echo "Go code: $(grep -c '^### Contoh Kode — Go' "$FILE") / expect 4"
echo "Node code: $(grep -c '^### Contoh Kode — Node.js' "$FILE") / expect 4"
grep -q '\*\*Selanjutnya:\*\*' "$FILE" && echo "footer OK" || echo "MISSING footer"
grep -qi 'TBD\|TODO\|lorem ipsum' "$FILE" && echo "PLACEHOLDER FOUND" || echo "no placeholders"
```
Expected: all counts = 4, no MISSING/PLACEHOLDER lines.

- [ ] **Step 4: Commit**

```bash
git add roadmap-backend/syllabus/phase-13-performance.md
git commit -m "docs: add phase 13 syllabus (performance)"
```

---

### Task 14: Phase 14 — Advanced Backend Concepts

**Files:**
- Create: `roadmap-backend/syllabus/phase-14-advanced-concepts.md`

**Interfaces:**
- Consumes: `CreateOrder`/`createOrder` (Task 3), `PublishOrderCreated`/`publishOrderCreated` (Task 5), `CallPaymentProviderWithCircuitBreaker`/`callPaymentProviderWithCircuitBreaker` (Task 9), `IsEventProcessed`/`isEventProcessed` (Task 5).
- Produces: `SaveOrderWithOutbox`/`saveOrderWithOutbox`, `VerifyWebhookSignature`/`verifyWebhookSignature`, `UploadProductImage`/`uploadProductImage`.

- [ ] **Step 1: Write the failing check**

```bash
test -f roadmap-backend/syllabus/phase-14-advanced-concepts.md && echo "exists" || echo "MISSING"
```
Expected: `MISSING`

- [ ] **Step 2: Dispatch the writing subagent**

```
Write roadmap-backend/syllabus/phase-14-advanced-concepts.md with the Write tool. Do not paste the content back.

CONTEXT & PROJECT: OrderFlow e-commerce order API. Language: casual Bahasa Indonesia mixed with untranslated technical terms.

ALREADY EXISTS (reuse by name): Go CreateOrder(ctx, db, userID, items) (*Order, error), PublishOrderCreated(ctx, w, order) error, CallPaymentProviderWithCircuitBreaker(ctx, req) (*PaymentResponse, error), IsEventProcessed(ctx, rdb, eventID) (bool, error) ; Node.js createOrder(pool, userId, items), publishOrderCreated(producer, order), callPaymentProviderWithCircuitBreaker(req), isEventProcessed(redisClient, eventId).

THIS PHASE introduces: Go SaveOrderWithOutbox(ctx context.Context, db *pgxpool.Pool, order *Order) error (inserts the order and an outbox event row in one transaction) and VerifyWebhookSignature(payload []byte, signature, secret string) bool and UploadProductImage(ctx context.Context, file io.Reader, filename string) (string, error) ; Node.js saveOrderWithOutbox(pool, order), verifyWebhookSignature(payload, signature, secret), uploadProductImage(fileBuffer, filename).

FILE STRUCTURE — open with exactly:
# Phase 14 — Advanced Backend Concepts

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

One section per topic, all 8 subsections in order. ALL topics require both code sections.

TOPICS:
98. Event-Driven Architecture (Order → OrderCreated event via PublishOrderCreated/publishOrderCreated → Payment/Notification consumers)
99. Saga Pattern (Create Order → Reserve Inventory → Charge Payment via CallPaymentProviderWithCircuitBreaker/callPaymentProviderWithCircuitBreaker → Ship, with compensating actions on failure)
100. Outbox Pattern (introduce SaveOrderWithOutbox/saveOrderWithOutbox here)
101. Distributed Idempotency (tie together IsEventProcessed/isEventProcessed from Phase 5 with Payment.idempotency_key)
102. Webhooks (introduce VerifyWebhookSignature/verifyWebhookSignature here, verifying a payment provider webhook)
103. File Upload (introduce UploadProductImage/uploadProductImage here, validating a product image upload)

End the file with:
---

**Selanjutnya:** [Phase 15 — Interview Thinking](./phase-15-interview-thinking.md)

Real, valid, idiomatic Go and Node.js code only — no pseudocode, no TBD/TODO, no skipped topics.
```

- [ ] **Step 3: Run the check again**

```bash
FILE="roadmap-backend/syllabus/phase-14-advanced-concepts.md"
TOPICS=("98. Event-Driven Architecture" "99. Saga Pattern" "100. Outbox Pattern" "101. Distributed Idempotency" "102. Webhooks" "103. File Upload")
test -f "$FILE" && echo "exists" || echo "MISSING"
grep -q '> Bagian dari \[Backend Engineer Roadmap\](../README.md)' "$FILE" && echo "byline OK" || echo "MISSING byline"
for t in "${TOPICS[@]}"; do grep -qF "## $t" "$FILE" || echo "MISSING TOPIC: $t"; done
for h in 'Apa itu?' 'Kenapa dibutuhkan?' 'Cara Kerja' 'Trade-off & Pitfall' 'Kapan Dipakai' 'Sering Ditanya Saat Interview'; do
  echo "$h: $(grep -c "^### $h" "$FILE") / expect 6"
done
echo "Go code: $(grep -c '^### Contoh Kode — Go' "$FILE") / expect 6"
echo "Node code: $(grep -c '^### Contoh Kode — Node.js' "$FILE") / expect 6"
grep -q '\*\*Selanjutnya:\*\*' "$FILE" && echo "footer OK" || echo "MISSING footer"
grep -qi 'TBD\|TODO\|lorem ipsum' "$FILE" && echo "PLACEHOLDER FOUND" || echo "no placeholders"
```
Expected: all counts = 6, no MISSING/PLACEHOLDER lines.

- [ ] **Step 4: Commit**

```bash
git add roadmap-backend/syllabus/phase-14-advanced-concepts.md
git commit -m "docs: add phase 14 syllabus (advanced backend concepts)"
```

---

### Task 15: Phase 15 — Interview Thinking

**Files:**
- Create: `roadmap-backend/syllabus/phase-15-interview-thinking.md`

**Interfaces:**
- Consumes: every concept from Phases 1–14, as worked examples grounded in OrderFlow scenarios (e.g. "how to prevent duplicate processing" references the idempotency key from Phase 2/5 and distributed idempotency from Phase 14).
- Produces: nothing (final phase, no code).

- [ ] **Step 1: Write the failing check**

```bash
test -f roadmap-backend/syllabus/phase-15-interview-thinking.md && echo "exists" || echo "MISSING"
```
Expected: `MISSING`

- [ ] **Step 2: Dispatch the writing subagent**

```
Write roadmap-backend/syllabus/phase-15-interview-thinking.md with the Write tool. Do not paste the content back.

CONTEXT & PROJECT: OrderFlow e-commerce order API — this final phase is purely conceptual, no code at all. Language: casual Bahasa Indonesia mixed with untranslated technical terms.

FILE STRUCTURE — open with exactly:
# Phase 15 — Interview Thinking

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

Then one `## <Framework Title>` section per framework below (no numbering — these aren't part of the 1-103 topic list). Each section needs exactly these 4 subsections (no code sections, no Kapan Dipakai/Trade-off split — use this framework-specific structure instead):
### Kerangka Berpikir
(the step-by-step thinking framework itself)
### Contoh Kasus Nyata
(a concrete OrderFlow scenario, e.g. for duplicate processing: "user double-clicks 'Bayar' on an order")
### Bagaimana Menjawabnya Saat Interview
(how to walk an interviewer through the answer out loud)
### Referensi Konsep Terkait
(which earlier phases/topics this framework draws on, by phase number and topic name)

FRAMEWORKS (use these exact titles as ## headings):
## Bagaimana Mencegah X?
Framework: Authentication → Authorization → Network → Validation → Encryption → Rate Limiting → Monitoring. Worked example: preventing unauthorized order cancellation.

## Bagaimana Scale X?
Framework: Identify bottleneck → Measure → Optimize → Cache → Horizontal scaling → Async processing → Database scaling. Worked example: OrderFlow's /products endpoint under Black Friday traffic.

## Apa yang Terjadi Kalau X Gagal?
Framework: Timeout → Retry → Circuit Breaker → Fallback → Queue → Idempotency → Monitoring → Alerting. Worked example: the payment provider going down mid-checkout.

## Bagaimana Improve API yang Lambat?
Framework: Logs → Metrics → Tracing → Find bottleneck → DB/Cache/Network/CPU → Optimize. Worked example: a slow GET /orders/:id endpoint.

## Bagaimana Cegah Duplicate Processing?
Framework: Idempotency Key → Unique Constraint → Processed Event → Database Transaction. Worked example: user double-clicking "Bayar" on checkout, or a webhook firing twice.

## Bagaimana Cegah Unauthorized Access?
Framework: Authentication → Authorization → Least Privilege → Network Restriction → Monitoring. Worked example: preventing a customer from viewing another customer's order via IDOR.

Do NOT include any code blocks or "Contoh Kode" headings anywhere in this file — this phase is explicitly conceptual only.

This is the last phase — do NOT add a "Selanjutnya" footer link.
```

- [ ] **Step 3: Run the check again**

```bash
FILE="roadmap-backend/syllabus/phase-15-interview-thinking.md"
TITLES=("## Bagaimana Mencegah X?" "## Bagaimana Scale X?" "## Apa yang Terjadi Kalau X Gagal?" "## Bagaimana Improve API yang Lambat?" "## Bagaimana Cegah Duplicate Processing?" "## Bagaimana Cegah Unauthorized Access?")
test -f "$FILE" && echo "exists" || echo "MISSING"
grep -q '> Bagian dari \[Backend Engineer Roadmap\](../README.md)' "$FILE" && echo "byline OK" || echo "MISSING byline"
for t in "${TITLES[@]}"; do grep -qF "$t" "$FILE" || echo "MISSING: $t"; done
echo "Kerangka Berpikir: $(grep -c '^### Kerangka Berpikir' "$FILE") / expect 6"
echo "Contoh Kasus Nyata: $(grep -c '^### Contoh Kasus Nyata' "$FILE") / expect 6"
echo "Bagaimana Menjawabnya: $(grep -c '^### Bagaimana Menjawabnya Saat Interview' "$FILE") / expect 6"
echo "Referensi Konsep: $(grep -c '^### Referensi Konsep Terkait' "$FILE") / expect 6"
grep -q '### Contoh Kode' "$FILE" && echo "UNEXPECTED CODE SECTION FOUND" || echo "no code sections (correct)"
grep -q '\*\*Selanjutnya:\*\*' "$FILE" && echo "UNEXPECTED footer found" || echo "no footer (correct, last phase)"
grep -qi 'TBD\|TODO\|lorem ipsum' "$FILE" && echo "PLACEHOLDER FOUND" || echo "no placeholders"
```
Expected: all four subsection counts = 6, no MISSING titles, "no code sections (correct)", "no footer (correct, last phase)", "no placeholders".

- [ ] **Step 4: Commit**

```bash
git add roadmap-backend/syllabus/phase-15-interview-thinking.md
git commit -m "docs: add phase 15 syllabus (interview thinking)"
```

---

## Self-Review Notes

- **Spec coverage:** Task 0 covers the README requirement; Tasks 1–15 cover all 15 phase files and all 103 numbered topics plus Phase 15's 6 unnumbered frameworks, verified against the spec's Topic Reference section. Conceptual-only exceptions (CAP Theorem, Basic Architecture, System Design Interview Framework, all of Phase 15) match the spec exactly.
- **Placeholder scan:** every task step contains literal, runnable check commands and literal subagent prompts — no "TBD"/"similar to Task N" shortcuts.
- **Type consistency:** the OrderFlow API surface table in Global Constraints is the single source of truth for every function signature referenced across Tasks 1–14; each task's prompt copies the exact signature rather than re-deriving it.
- **Scope:** one cohesive deliverable (roadmap-backend/ fully populated); no unrelated repo changes.
