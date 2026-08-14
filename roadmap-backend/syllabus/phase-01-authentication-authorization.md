# Phase 01 — Authentication & Authorization

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

## 1. Authentication vs Authorization

### Apa itu?
Authentication (AuthN) adalah proses verifikasi identitas: "kamu ini siapa?". Authorization (AuthZ) adalah proses menentukan hak akses: "kamu boleh ngapain aja?". Dua konsep ini sering disingkat jadi satu kata "auth", padahal keduanya adalah layer terpisah yang dieksekusi berurutan — authentication selalu duluan, baru authorization.

### Kenapa dibutuhkan?
Tanpa authentication, sistem gak bisa tau siapa yang sedang request — semua request dianggap anonymous, atau lebih parah, dianggap punya identitas yang sama. Tanpa authorization, begitu user berhasil login, dia bisa akses/ubah resource siapa aja, termasuk milik user lain. Di OrderFlow: authentication mastiin request datang dari user yang valid (misal `user_id = 42`), authorization mastiin user 42 cuma bisa lihat/ubah order miliknya sendiri, bukan order milik user 99.

### Cara Kerja
```
Request masuk
  → Middleware Authentication: baca header Authorization, validasi JWT/session
     → Gagal → 401 Unauthorized
     → Berhasil → identitas (user_id, role) ditempel ke context request
  → Handler/Middleware Authorization: cek role/ownership terhadap resource yang diminta
     → Gagal → 403 Forbidden
     → Berhasil → lanjut ke business logic
```
Authorization butuh tau identitas user dulu, jadi urutannya gak bisa dibalik: gak mungkin cek "boleh ngapain" sebelum tau "ini siapa".

### Contoh Kode — Go
```go
package handler

import (
	"net/http"
	"strconv"

	"github.com/go-chi/chi/v5"
	"orderflow/internal/auth"
)

// DeleteOrder didaftarkan di router dengan auth.Middleware terpasang duluan,
// jadi di titik ini request sudah pasti authenticated.
func DeleteOrder(w http.ResponseWriter, r *http.Request) {
	// --- Authentication: identitas sudah divalidasi middleware, tinggal diambil ---
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

	// --- Authorization: admin boleh hapus order siapa aja, user biasa cuma order miliknya ---
	if claims.Role != "admin" && order.UserID != claims.UserID {
		http.Error(w, "forbidden", http.StatusForbidden)
		return
	}

	if err := deleteOrder(r.Context(), orderID); err != nil {
		http.Error(w, "failed to delete order", http.StatusInternalServerError)
		return
	}
	w.WriteHeader(http.StatusNoContent)
}
```

### Contoh Kode — Node.js
```javascript
const express = require('express');
const { authenticate } = require('../middleware/authenticate');
const { findOrderById, deleteOrderById } = require('../repositories/orderRepository');

const router = express.Router();

// authenticate = middleware authentication, jalan duluan buat semua route di bawah ini
router.delete('/orders/:id', authenticate, async (req, res) => {
  const orderId = Number(req.params.id);
  if (Number.isNaN(orderId)) {
    return res.status(400).json({ error: 'invalid order id' });
  }

  const order = await findOrderById(orderId);
  if (!order) {
    return res.status(404).json({ error: 'order not found' });
  }

  // --- Authorization: cek role atau ownership ---
  const isOwner = order.user_id === req.user.userId;
  const isAdmin = req.user.role === 'admin';
  if (!isOwner && !isAdmin) {
    return res.status(403).json({ error: 'forbidden' });
  }

  await deleteOrderById(orderId);
  return res.status(204).send();
});

module.exports = router;
```

### Trade-off & Pitfall
- Pitfall paling umum: cuma cek authentication ("user ini login gak?") terus lupa cek authorization ("dia boleh akses resource spesifik ini gak?"). Ini akar dari bug IDOR/BOLA yang dibahas detail di Phase 2.
- Jangan tulis ulang logic authentication di tiap handler — taruh di middleware biar konsisten dan gak ada endpoint yang kelupaan di-protect.
- Authorization yang di-hardcode per role gampang keliru begitu jumlah role/permission bertambah — biasanya berkembang jadi RBAC (topik 8) atau permission-based access control (topik 9).

### Kapan Dipakai
Authentication dibutuhkan di hampir semua API yang punya konsep "user" atau "client", kecuali endpoint yang memang didesain publik (misalnya health check). Authorization dibutuhkan begitu ada lebih dari satu level akses atau resource yang punya kepemilikan (ownership), seperti order, payment, atau data pribadi user.

### Sering Ditanya Saat Interview
- "Apa bedanya 401 dan 403, dan kenapa itu penting?" — 401 berarti belum/gagal authentication, 403 berarti sudah authenticated tapi authorization-nya gagal.
- "Gimana caramu mencegah user A akses data user B lewat endpoint yang sama?" — jawabannya soal authorization/ownership check, bukan cuma authentication.
- "Kalau authentication gagal di middleware, apa authorization masih perlu dijalankan?" — biasanya nggak, request langsung ditolak di layer authentication sebelum sempat masuk ke logic authorization.

---

## 2. Password Authentication

### Apa itu?
Password authentication adalah verifikasi identitas dengan mencocokkan password yang diinput user terhadap representasi password yang tersimpan di database. Representasi ini harus berupa hash satu arah, bukan password asli maupun hasil enkripsi yang bisa didekripsi balik.

### Kenapa dibutuhkan?
Database bisa bocor kapan saja — SQL injection, backup yang gak sengaja kebuka, insider threat. Kalau password disimpan plaintext atau dienkripsi (reversible), begitu database bocor, semua password user langsung ketahuan, termasuk password yang dipakai ulang di layanan lain. Hashing satu arah bikin attacker harus brute-force tiap password satu per satu, dan algoritma seperti bcrypt sengaja dibikin lambat supaya brute-force itu jadi mahal secara komputasi.

### Cara Kerja
```
Register:  password → bcrypt.GenerateFromPassword (salt otomatis) → password_hash → simpan di DB
Login:     password input → bcrypt.CompareHashAndPassword(hash, password) → match / no match
```
bcrypt menyisipkan salt random di dalam hash-nya sendiri (format `$2a$10$...`), jadi gak perlu kolom salt terpisah. "Cost factor" (misalnya 10-12) menentukan seberapa berat proses hashing-nya — makin tinggi, makin lambat di-brute-force, tapi makin berat juga buat server.

### Contoh Kode — Go
```go
package auth

import "golang.org/x/crypto/bcrypt"

// bcryptCost menentukan seberapa berat komputasi hashing.
// Semakin tinggi, semakin lambat brute-force, tapi semakin berat juga buat server.
const bcryptCost = bcrypt.DefaultCost // 10

func HashPassword(password string) (string, error) {
	hashed, err := bcrypt.GenerateFromPassword([]byte(password), bcryptCost)
	if err != nil {
		return "", err
	}
	return string(hashed), nil
}

func CheckPasswordHash(password, hash string) bool {
	err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
	return err == nil
}
```

Dipakai saat registrasi User baru di OrderFlow:
```go
func RegisterUser(ctx context.Context, db *pgxpool.Pool, email, password string) (int64, error) {
	hash, err := auth.HashPassword(password)
	if err != nil {
		return 0, fmt.Errorf("hash password: %w", err)
	}

	var userID int64
	err = db.QueryRow(ctx,
		`INSERT INTO users (email, password_hash, role) VALUES ($1, $2, 'customer') RETURNING id`,
		email, hash,
	).Scan(&userID)
	if err != nil {
		return 0, fmt.Errorf("insert user: %w", err)
	}
	return userID, nil
}
```

### Contoh Kode — Node.js
```javascript
const bcrypt = require('bcrypt');

const SALT_ROUNDS = 10; // setara bcrypt.DefaultCost di Go

async function hashPassword(password) {
  return bcrypt.hash(password, SALT_ROUNDS);
}

async function checkPasswordHash(password, hash) {
  return bcrypt.compare(password, hash);
}

module.exports = { hashPassword, checkPasswordHash };
```

Dipakai saat registrasi User baru di OrderFlow:
```javascript
const { hashPassword } = require('./auth');
const pool = require('../db/pool'); // instance pg.Pool

async function registerUser(email, password) {
  const passwordHash = await hashPassword(password);
  const result = await pool.query(
    `INSERT INTO users (email, password_hash, role) VALUES ($1, $2, 'customer') RETURNING id`,
    [email, passwordHash]
  );
  return result.rows[0].id;
}

module.exports = { registerUser };
```

### Trade-off & Pitfall
- Jangan pakai MD5/SHA-256 polos buat password — algoritma itu didesain cepat, justru memudahkan brute-force. Pakai algoritma yang sengaja lambat: bcrypt, scrypt, atau Argon2 (pemenang Password Hashing Competition, direkomendasikan buat sistem baru).
- Cost factor bcrypt adalah trade-off langsung antara security dan latency login/register — makin tinggi cost, makin lama waktu hashing (bisa ratusan ms), jadi jangan asal set tinggi tanpa load testing.
- bcrypt punya batas panjang input 72 byte — password yang lebih panjang efektif terpotong. Kalau butuh dukung passphrase panjang, pertimbangkan Argon2, atau pre-hash SHA-256 sebelum masuk bcrypt.
- Selalu bandingkan pakai fungsi resmi (`CompareHashAndPassword`/`bcrypt.compare`) yang constant-time, jangan compare hash string manual pakai `==` — itu rentan timing attack.

### Kapan Dipakai
Dipakai di hampir semua sistem yang punya login berbasis credential sendiri (bukan lewat OAuth/SSO provider). Kalau perusahaan sudah punya identity provider terpusat, password authentication biasanya didelegasikan ke OAuth 2.0/OIDC (topik 7) supaya OrderFlow gak perlu menyimpan password user sama sekali.

### Sering Ditanya Saat Interview
- "Kenapa gak boleh pakai enkripsi buat simpan password?" — enkripsi itu reversible (ada key buat decrypt), kalau key bocor semua password ter-expose; hashing itu satu arah.
- "Apa itu salt dan kenapa penting?" — nilai random yang digabung ke password sebelum di-hash, supaya dua user dengan password sama tetap punya hash berbeda, dan mencegah rainbow table attack.
- "Kenapa bcrypt/Argon2 lebih baik dari SHA-256 buat password?" — karena didesain lambat dan computationally expensive sehingga brute-force jadi mahal, sementara SHA-256 didesain cepat (bagus buat checksum, buruk buat password).

---

## 3. JWT

### Apa itu?
JWT (JSON Web Token) adalah format token yang membawa klaim (claims) dalam bentuk JSON dan ditandatangani secara digital, terdiri dari tiga bagian dipisahkan titik: `Header.Payload.Signature`. Header berisi algoritma signing, payload berisi claims (data), signature memastikan token gak diubah-ubah oleh pihak lain.

### Kenapa dibutuhkan?
Server perlu cara "mengingat" bahwa user tertentu sudah login, tanpa harus query session ke database di setiap request (stateless). JWT menyimpan identitas & role user langsung di dalam token, ditandatangani pakai secret/private key server, sehingga server cukup verifikasi signature-nya buat percaya isi token — tanpa hit database sama sekali di request biasa.

### Cara Kerja
```
Login → validasi email/password → generate JWT (claims: user_id, role, exp)
      → sign pakai secret key (HS256) atau private key (RS256)
      → kirim JWT ke client

Request berikutnya → header Authorization: Bearer <JWT>
      → server verify signature JWT pakai secret/public key
      → valid & belum expired → ambil claims → user dianggap authenticated
```
Claims umum: `sub` (subject/user id), `iat` (issued at), `exp` (expiration), plus custom claim seperti `role`.

### Contoh Kode — Go
```go
package auth

import (
	"context"
	"errors"
	"fmt"
	"net/http"
	"os"
	"strings"
	"time"

	"github.com/golang-jwt/jwt/v5"
)

var jwtSecret = []byte(os.Getenv("JWT_SECRET")) // jangan pernah hardcode di source code

const accessTokenTTL = 15 * time.Minute

// Claims membawa data user yang dibutuhkan tanpa perlu query DB lagi di tiap request.
type Claims struct {
	UserID int64  `json:"user_id"`
	Role   string `json:"role"`
	jwt.RegisteredClaims
}

func GenerateAccessToken(userID int64, role string) (string, error) {
	now := time.Now()
	claims := Claims{
		UserID: userID,
		Role:   role,
		RegisteredClaims: jwt.RegisteredClaims{
			Subject:   fmt.Sprintf("%d", userID),
			IssuedAt:  jwt.NewNumericDate(now),
			ExpiresAt: jwt.NewNumericDate(now.Add(accessTokenTTL)),
		},
	}

	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString(jwtSecret)
}

func ValidateToken(tokenString string) (*Claims, error) {
	claims := &Claims{}
	token, err := jwt.ParseWithClaims(tokenString, claims, func(t *jwt.Token) (interface{}, error) {
		if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
			return nil, fmt.Errorf("unexpected signing method: %v", t.Header["alg"])
		}
		return jwtSecret, nil
	})
	if err != nil {
		return nil, err
	}
	if !token.Valid {
		return nil, errors.New("invalid token")
	}
	return claims, nil
}

type contextKey string

const ClaimsContextKey contextKey = "claims"

// Middleware memvalidasi Bearer token dan menempel claims-nya ke context request.
func Middleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		authHeader := r.Header.Get("Authorization")
		tokenString := strings.TrimPrefix(authHeader, "Bearer ")
		if tokenString == "" || tokenString == authHeader {
			http.Error(w, "missing bearer token", http.StatusUnauthorized)
			return
		}

		claims, err := ValidateToken(tokenString)
		if err != nil {
			http.Error(w, "invalid or expired token", http.StatusUnauthorized)
			return
		}

		ctx := context.WithValue(r.Context(), ClaimsContextKey, claims)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}
```

### Contoh Kode — Node.js
```javascript
const jwt = require('jsonwebtoken');

const JWT_SECRET = process.env.JWT_SECRET; // jangan pernah hardcode di source code
const ACCESS_TOKEN_TTL = '15m';

function generateAccessToken(userId, role) {
  return jwt.sign(
    { user_id: userId, role },
    JWT_SECRET,
    { subject: String(userId), expiresIn: ACCESS_TOKEN_TTL }
  );
}

function validateToken(token) {
  // jwt.verify melempar error kalau signature invalid atau token expired
  return jwt.verify(token, JWT_SECRET);
}

module.exports = { generateAccessToken, validateToken };
```

Middleware pemakaian (Express):
```javascript
const { validateToken } = require('./auth');

function authenticate(req, res, next) {
  const authHeader = req.headers.authorization || '';
  const [scheme, token] = authHeader.split(' ');

  if (scheme !== 'Bearer' || !token) {
    return res.status(401).json({ error: 'missing bearer token' });
  }

  try {
    const claims = validateToken(token);
    req.user = { userId: Number(claims.sub), role: claims.role };
    next();
  } catch (err) {
    return res.status(401).json({ error: 'invalid or expired token' });
  }
}

module.exports = { authenticate };
```

### Trade-off & Pitfall
- JWT itu stateless — sekali di-issue, dia valid sampai expired, gak ada cara langsung "cabut paksa" satu token tanpa infrastruktur tambahan (blocklist di Redis, versioning, dsb). Beda dengan session yang bisa langsung dihapus dari server.
- Jangan taruh data sensitif (password, nomor kartu, dll) di payload — payload JWT cuma di-encode base64, bukan dienkripsi, jadi siapapun yang pegang token bisa baca isinya.
- Selalu set `exp` yang pendek untuk access token, karena makin panjang umur token, makin lama window of exposure kalau token itu bocor.
- Wajib pakai HTTPS — kalau token disadap di jaringan (man-in-the-middle) dan expiry-nya panjang, attacker bisa langsung impersonate user.
- Validasi algoritma signing di server (`alg` di header), jangan percaya begitu saja — ada serangan klasik "alg confusion" di mana attacker mengubah `alg` jadi `none` atau ganti RS256 jadi HS256 memakai public key sebagai secret.

### Kapan Dipakai
Cocok untuk API stateless yang harus scale horizontal tanpa shared session store, terutama di arsitektur microservices di mana banyak service perlu memverifikasi identitas user tanpa selalu hit satu auth service pusat.

### Sering Ditanya Saat Interview
- "Kalau JWT stateless, gimana cara logout/revoke token sebelum expired?" — biasanya pakai token blocklist di Redis (simpan `jti` yang di-revoke), kombinasi short-lived access token + refresh token whitelist (topik 4), atau token versioning per user.
- "Apa yang terjadi kalau secret key JWT bocor?" — attacker bisa forge token apapun dengan identitas dan role apapun; makanya secret harus disimpan di secret manager, bukan hardcode di kode/repo.
- "Kenapa JWT payload gak boleh berisi data sensitif?" — karena payload cuma di-base64 encode, siapapun bisa decode dan baca isinya tanpa perlu secret key sama sekali.

---

## 4. Access Token vs Refresh Token

### Apa itu?
Access token adalah token berumur pendek yang dikirim di setiap request buat akses resource API. Refresh token adalah token berumur panjang yang cuma dipakai buat satu hal: menukar dirinya jadi access token baru, tanpa user harus login ulang pakai password.

### Kenapa dibutuhkan?
Kalau access token dibuat berumur panjang (misal 7 hari) supaya user gak perlu login mulu, risikonya kalau token itu bocor, attacker punya akses penuh selama 7 hari. Kalau dibuat pendek (15 menit) tanpa refresh token, user harus login ulang tiap 15 menit — pengalaman yang buruk. Kombinasi keduanya menyelesaikan dua masalah sekaligus: access token pendek membatasi window of exposure, refresh token panjang menjaga user tetap login secara transparan.

### Cara Kerja
```
Login → server generate Access Token (15 menit) + Refresh Token (7 hari)
     → keduanya dikirim ke client (access token di body/header, refresh token di httpOnly cookie)

Access Token expired → client kirim Refresh Token ke POST /auth/refresh
     → server validasi signature + cek apakah jti-nya masih ada di whitelist (Redis)
     → server generate Access Token baru
     → user tetap logged in tanpa input password lagi
```
Refresh token disimpan server-side (whitelist di Redis, key-nya `jti`) supaya bisa di-revoke kapan saja — beda dari access token yang murni stateless.

### Contoh Kode — Go
Melanjutkan `auth.go` dari topik 3, menambahkan `GenerateRefreshToken`:
```go
import (
	"crypto/rand"
	"encoding/hex"
)

const refreshTokenTTL = 7 * 24 * time.Hour

// generateJTI bikin ID unik buat refresh token, dipakai sebagai key whitelist di Redis
// supaya refresh token bisa di-revoke kapan saja (misal saat user logout).
func generateJTI() (string, error) {
	b := make([]byte, 16)
	if _, err := rand.Read(b); err != nil {
		return "", err
	}
	return hex.EncodeToString(b), nil
}

func GenerateRefreshToken(userID int64) (string, error) {
	jti, err := generateJTI()
	if err != nil {
		return "", fmt.Errorf("generate jti: %w", err)
	}

	now := time.Now()
	claims := jwt.RegisteredClaims{
		Subject:   fmt.Sprintf("%d", userID),
		ID:        jti,
		IssuedAt:  jwt.NewNumericDate(now),
		ExpiresAt: jwt.NewNumericDate(now.Add(refreshTokenTTL)),
	}

	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	signed, err := token.SignedString(jwtSecret)
	if err != nil {
		return "", fmt.Errorf("sign refresh token: %w", err)
	}
	return signed, nil
}
```

Login handler yang generate keduanya dan simpan refresh token ke whitelist Redis:
```go
func LoginHandler(rdb *redis.Client) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		var req struct {
			Email    string `json:"email"`
			Password string `json:"password"`
		}
		if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
			http.Error(w, "invalid body", http.StatusBadRequest)
			return
		}

		user, err := findUserByEmail(r.Context(), req.Email)
		if err != nil || !CheckPasswordHash(req.Password, user.PasswordHash) {
			http.Error(w, "invalid credentials", http.StatusUnauthorized)
			return
		}

		accessToken, err := GenerateAccessToken(user.ID, user.Role)
		if err != nil {
			http.Error(w, "failed to generate token", http.StatusInternalServerError)
			return
		}
		refreshToken, err := GenerateRefreshToken(user.ID)
		if err != nil {
			http.Error(w, "failed to generate token", http.StatusInternalServerError)
			return
		}

		// ambil jti dari refresh token yang baru dibuat, simpan ke whitelist Redis
		refreshClaims, _ := ValidateToken(refreshToken)
		rdb.Set(r.Context(), "refresh:"+refreshClaims.ID, user.ID, refreshTokenTTL)

		json.NewEncoder(w).Encode(map[string]string{
			"access_token":  accessToken,
			"refresh_token": refreshToken,
		})
	}
}

// RefreshHandler menukar refresh token yang masih valid jadi access token baru.
func RefreshHandler(rdb *redis.Client) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		var req struct {
			RefreshToken string `json:"refresh_token"`
		}
		if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
			http.Error(w, "invalid body", http.StatusBadRequest)
			return
		}

		claims, err := ValidateToken(req.RefreshToken)
		if err != nil {
			http.Error(w, "invalid or expired refresh token", http.StatusUnauthorized)
			return
		}

		exists, err := rdb.Exists(r.Context(), "refresh:"+claims.ID).Result()
		if err != nil || exists == 0 {
			http.Error(w, "refresh token revoked", http.StatusUnauthorized)
			return
		}

		userID, _ := strconv.ParseInt(claims.Subject, 10, 64)
		user, err := findUserByID(r.Context(), userID)
		if err != nil {
			http.Error(w, "user not found", http.StatusUnauthorized)
			return
		}

		newAccessToken, err := GenerateAccessToken(user.ID, user.Role)
		if err != nil {
			http.Error(w, "failed to generate token", http.StatusInternalServerError)
			return
		}
		json.NewEncoder(w).Encode(map[string]string{"access_token": newAccessToken})
	}
}
```

### Contoh Kode — Node.js
Melanjutkan `auth.js` dari topik 3, menambahkan `generateRefreshToken`:
```javascript
const crypto = require('crypto');

const REFRESH_TOKEN_TTL = '7d';
const REFRESH_TOKEN_TTL_SECONDS = 7 * 24 * 60 * 60;

function generateRefreshToken(userId) {
  const jti = crypto.randomBytes(16).toString('hex');
  return jwt.sign(
    {},
    JWT_SECRET,
    { subject: String(userId), jwtid: jti, expiresIn: REFRESH_TOKEN_TTL }
  );
}

module.exports = { generateAccessToken, generateRefreshToken, validateToken };
```

Login & refresh handler:
```javascript
const jwt = require('jsonwebtoken');
const redis = require('../redisClient'); // instance ioredis
const { findUserByEmail, findUserById } = require('../repositories/userRepository');
const { generateAccessToken, generateRefreshToken, validateToken, checkPasswordHash } = require('./auth');

async function login(req, res) {
  const { email, password } = req.body;
  const user = await findUserByEmail(email);
  if (!user || !(await checkPasswordHash(password, user.password_hash))) {
    return res.status(401).json({ error: 'invalid credentials' });
  }

  const accessToken = generateAccessToken(user.id, user.role);
  const refreshToken = generateRefreshToken(user.id);

  // ambil jti dari refresh token yang baru dibuat, simpan ke whitelist Redis
  const { jti } = jwt.decode(refreshToken);
  await redis.set(`refresh:${jti}`, user.id, 'EX', REFRESH_TOKEN_TTL_SECONDS);

  return res.json({ access_token: accessToken, refresh_token: refreshToken });
}

async function refresh(req, res) {
  const { refresh_token: refreshToken } = req.body;
  let claims;
  try {
    claims = validateToken(refreshToken);
  } catch (err) {
    return res.status(401).json({ error: 'invalid or expired refresh token' });
  }

  const stillValid = await redis.get(`refresh:${claims.jti}`);
  if (!stillValid) {
    return res.status(401).json({ error: 'refresh token revoked' });
  }

  const user = await findUserById(Number(claims.sub));
  if (!user) {
    return res.status(401).json({ error: 'user not found' });
  }

  const newAccessToken = generateAccessToken(user.id, user.role);
  return res.json({ access_token: newAccessToken });
}

module.exports = { login, refresh };
```

### Trade-off & Pitfall
- Refresh token yang gak di-rotate (dipakai berkali-kali tanpa pernah diganti) meningkatkan risiko: sekali bocor, attacker bisa terus generate access token baru sampai refresh token expired. Refresh token rotation — setiap dipakai langsung diganti yang baru, yang lama di-invalidate — mengurangi risiko ini.
- Simpan refresh token di httpOnly, secure cookie (bukan localStorage) buat web app, supaya gak gampang dicuri lewat XSS.
- Refresh token butuh state di server (whitelist), jadi sistem gak 100% stateless lagi — ini trade-off yang harus diterima demi kemampuan revoke.
- Kalau sistem mendeteksi refresh token yang sama dipakai dua kali setelah rotation (reuse detection), itu sinyal kuat token itu bocor, dan sistem harus langsung revoke semua token milik user tersebut.

### Kapan Dipakai
Dipakai di hampir semua sistem login modern yang mengutamakan UX (user gak perlu re-login tiap beberapa menit) sekaligus keamanan (access token tetap short-lived). Untuk service-to-service call yang gak melibatkan sesi user, biasanya cukup pakai API key (topik 5) atau client credentials grant OAuth, bukan pola access/refresh token ini.

### Sering Ditanya Saat Interview
- "Kenapa gak bikin access token aja yang umurnya panjang, biar simpel?" — makin panjang umur token, makin besar window of exposure kalau bocor, dan JWT yang stateless susah di-revoke sebelum expired.
- "Di mana sebaiknya refresh token disimpan di sisi client?" — httpOnly secure cookie, supaya JavaScript (dan XSS) gak bisa baca isinya.
- "Apa itu refresh token rotation dan kenapa penting?" — setiap refresh token dipakai, langsung diganti baru; kalau ada percobaan pakai refresh token lama yang sudah di-rotate, itu tanda pencurian token dan semua token user harus di-revoke.

---

## 5. API Key vs Access Token

### Apa itu?
API key adalah credential statis yang merepresentasikan identitas aplikasi/service (bukan user individu), biasanya berupa string acak panjang yang dikirim lewat header seperti `X-API-Key`. Access token merepresentasikan identitas user/sesi tertentu, biasanya short-lived dan berbentuk JWT seperti yang sudah dibahas di topik 3.

### Kenapa dibutuhkan?
OrderFlow gak cuma diakses user lewat browser/mobile app — ada juga integrasi service-to-service, misalnya partner logistik yang narik status order lewat API, atau internal service `inventory-service` yang update stock. Buat kasus ini gak ada "user yang login", jadi access token yang terikat ke sesi user kurang pas. API key memberi identitas ke aplikasi/partner itu sendiri, dengan siklus hidup yang berbeda (biasanya gak expired otomatis, tapi bisa di-rotate manual).

### Cara Kerja
```
Partner request → header X-API-Key: <key>
  → server hash key yang diterima → cocokkan ke kolom key_hash di tabel api_keys
  → ketemu & masih aktif → identitas = partner/service tertentu → lanjut proses
```
API key sebaiknya disimpan di DB dalam bentuk hash (mirip password), bukan plaintext, supaya kalau DB bocor, key asli tetap aman.

### Contoh Kode — Go
```go
package auth

import (
	"context"
	"crypto/sha256"
	"encoding/hex"
	"net/http"
)

func hashAPIKey(key string) string {
	sum := sha256.Sum256([]byte(key))
	return hex.EncodeToString(sum[:])
}

type apiKeyContextKey string

const APIKeyClientContextKey apiKeyContextKey = "api_key_client"

type Client struct {
	ID   int64
	Name string
}

type APIKeyStore interface {
	FindActiveByKeyHash(ctx context.Context, keyHash string) (Client, error)
}

// APIKeyMiddleware dipakai buat endpoint service-to-service, misalnya
// partner logistik yang narik status order OrderFlow.
func APIKeyMiddleware(store APIKeyStore) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			key := r.Header.Get("X-API-Key")
			if key == "" {
				http.Error(w, "missing api key", http.StatusUnauthorized)
				return
			}

			client, err := store.FindActiveByKeyHash(r.Context(), hashAPIKey(key))
			if err != nil {
				http.Error(w, "invalid api key", http.StatusUnauthorized)
				return
			}

			ctx := context.WithValue(r.Context(), APIKeyClientContextKey, client)
			next.ServeHTTP(w, r.WithContext(ctx))
		})
	}
}
```

### Contoh Kode — Node.js
```javascript
const crypto = require('crypto');

function hashApiKey(key) {
  return crypto.createHash('sha256').update(key).digest('hex');
}

// apiKeyAuth dipakai buat endpoint service-to-service — bedanya dengan
// authenticate (JWT) ada di identitas yang dibawa: aplikasi/partner, bukan user.
function apiKeyAuth(apiKeyStore) {
  return async (req, res, next) => {
    const key = req.header('X-API-Key');
    if (!key) {
      return res.status(401).json({ error: 'missing api key' });
    }

    const client = await apiKeyStore.findActiveByKeyHash(hashApiKey(key));
    if (!client) {
      return res.status(401).json({ error: 'invalid api key' });
    }

    req.client = client;
    next();
  };
}

module.exports = { apiKeyAuth, hashApiKey };
```

### Trade-off & Pitfall
- API key biasanya gak punya expiration bawaan seperti JWT — kalau bocor, dia valid selamanya sampai di-revoke manual. Harus ada mekanisme rotation & revocation yang jelas.
- API key merepresentasikan aplikasi, bukan user, jadi gak cocok buat fine-grained authorization per-user (misalnya "user mana yang boleh refund order ini"). Untuk itu access token/JWT lebih pas karena bisa bawa claim `user_id` & `role`.
- Simpan API key dalam bentuk hash di DB (sama seperti password), dan jangan pernah log key mentah di access log/error log.
- Rate limiting dan monitoring per-API-key penting, karena satu partner yang bermasalah gak boleh mengganggu partner lain.

### Kapan Dipakai
API key cocok buat integrasi partner eksternal dan service-to-service internal yang gak melibatkan identitas user — lebih butuh "siapa aplikasinya" daripada "siapa usernya". Access token/JWT cocok buat request yang datang atas nama user tertentu dan butuh authorization berbasis role/ownership user itu.

### Sering Ditanya Saat Interview
- "Kalau partner API OrderFlow butuh akses ke data user tertentu, apakah cukup API key aja?" — biasanya nggak, butuh kombinasi API key (identitas aplikasi) dengan mekanisme scoped ke resource tertentu, misalnya OAuth client credentials + scope, atau access token atas nama user lewat authorization code flow.
- "Gimana cara revoke API key yang bocor tanpa ganggu partner lain?" — tiap partner/aplikasi punya key sendiri di tabel `api_keys`, jadi revoke cukup disable/hapus row spesifik itu.
- "Kenapa API key gak dianggap seaman access token buat autentikasi user?" — API key statis dan long-lived, gak membawa scope/expiry granular per user seperti JWT.

---

## 6. Session vs JWT

### Apa itu?
Session-based auth: setelah login, server bikin session record (biasanya di Redis/DB) dan kirim session ID ke client lewat cookie. Client cukup kirim balik session ID itu, server lookup datanya di storage. JWT-based auth: seluruh data user (claims) ditandatangani dan disimpan di token itu sendiri, gak butuh lookup ke storage buat verifikasi (stateless).

### Kenapa dibutuhkan?
Ini soal trade-off arsitektur: kalau OrderFlow butuh revocation instan dan kontrol penuh atas siapa yang masih login (misal fitur "logout dari semua device"), session lebih gampang. Kalau OrderFlow butuh scale horizontal ke banyak instance tanpa shared state, dan dipanggil dari banyak service berbeda, JWT lebih murah karena verifikasi gak perlu network call ke storage.

### Cara Kerja
Session:
```
Login → server create session record di Redis (session_id → user_id, role, expiry)
     → server set cookie: session_id=<id> (httpOnly, secure)
Request berikutnya → server ambil session_id dari cookie → lookup ke Redis → dapat data user
Logout → server hapus session record di Redis → cookie langsung gak valid lagi
```
JWT:
```
Login → server generate JWT berisi claims, sign, kirim ke client
Request berikutnya → server verify signature JWT → langsung dapat claims, tanpa network call
Logout → gak ada storage yang dihapus; JWT tetap valid sampai expired (kecuali ada blocklist tambahan)
```

### Contoh Kode — Go
Session-based, pakai go-redis sebagai session store:
```go
package handler

import (
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"net/http"
	"time"

	"github.com/redis/go-redis/v9"
)

const sessionTTL = 24 * time.Hour

func generateSessionID() string {
	b := make([]byte, 16)
	rand.Read(b)
	return hex.EncodeToString(b)
}

func LoginSessionHandler(rdb *redis.Client) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		user := authenticateCredentials(r) // validasi email/password

		sessionID := generateSessionID()
		sessionData := fmt.Sprintf(`{"user_id":%d,"role":"%s"}`, user.ID, user.Role)
		rdb.Set(r.Context(), "session:"+sessionID, sessionData, sessionTTL)

		http.SetCookie(w, &http.Cookie{
			Name:     "session_id",
			Value:    sessionID,
			HttpOnly: true,
			Secure:   true,
			SameSite: http.SameSiteLaxMode,
			MaxAge:   int(sessionTTL.Seconds()),
		})
		w.WriteHeader(http.StatusOK)
	}
}

func SessionMiddleware(rdb *redis.Client) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			cookie, err := r.Cookie("session_id")
			if err != nil {
				http.Error(w, "unauthorized", http.StatusUnauthorized)
				return
			}
			if _, err := rdb.Get(r.Context(), "session:"+cookie.Value).Result(); err != nil {
				http.Error(w, "session expired or invalid", http.StatusUnauthorized)
				return
			}
			next.ServeHTTP(w, r)
		})
	}
}
```
JWT-based sudah dibahas lengkap di topik 3 & 4 (`GenerateAccessToken`, `ValidateToken`, `Middleware`) — rujuk ke sana untuk versi statelessnya.

### Contoh Kode — Node.js
Session-based, pakai Express + `express-session` + `connect-redis`:
```javascript
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const redisClient = require('../redisClient');

app.use(session({
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: { httpOnly: true, secure: true, sameSite: 'lax', maxAge: 24 * 60 * 60 * 1000 },
}));

app.post('/login', async (req, res) => {
  const user = await authenticateCredentials(req.body);
  req.session.userId = user.id;
  req.session.role = user.role;
  res.sendStatus(200);
});

function requireSession(req, res, next) {
  if (!req.session.userId) {
    return res.status(401).json({ error: 'unauthorized' });
  }
  next();
}

module.exports = { requireSession };
```
JWT-based sudah dibahas lengkap di topik 3 & 4 (`generateAccessToken`, `validateToken`, middleware `authenticate`) — rujuk ke sana untuk versi statelessnya.

### Trade-off & Pitfall
| | Session | JWT |
|---|---|---|
| Revocation | Instan (hapus record di storage) | Susah, butuh blocklist tambahan |
| Scalability | Butuh shared storage (Redis) antar instance | Stateless, gampang scale horizontal |
| Payload size | Kecil (cuma session ID di cookie) | Lebih besar (seluruh claims ikut terkirim tiap request) |
| Cross-service | Semua service harus akses session store yang sama | Gampang diverifikasi lintas service selama share secret/public key |
| Ketergantungan infra | Redis/DB down = semua session bermasalah | Auth service down gak langsung ganggu verifikasi token yang sudah ada |

Pitfall umum: pakai JWT tapi expect bisa "logout" seketika seperti session — padahal JWT tetap valid sampai expired kecuali ada mekanisme tambahan (blocklist, atau short TTL + refresh token whitelist seperti topik 4).

### Kapan Dipakai
Session cocok untuk aplikasi monolith/sedikit service dengan kebutuhan revocation ketat (banking, admin panel internal). JWT cocok untuk arsitektur microservices/API publik yang butuh verifikasi cepat lintas service tanpa selalu bergantung ke satu storage pusat — dan bisa dikombinasikan dengan refresh token whitelist (topik 4) untuk tetap dapat revocation yang reasonable.

### Sering Ditanya Saat Interview
- "Kenapa JWT dibilang stateless padahal refresh token-nya disimpan di Redis?" — access token-nya sendiri stateless (verifikasi cukup pakai signature), tapi refresh token butuh state tambahan supaya bisa di-revoke; ini hybrid approach yang umum dipakai di produksi.
- "Gimana kalau butuh scale session-based auth ke banyak instance server?" — pindahkan session store dari in-memory ke shared storage seperti Redis, supaya semua instance bisa akses data session yang sama.
- "Apa risiko terbesar JWT dibanding session?" — kesulitan revocation instan; token yang sudah di-issue tetap valid sampai expired walau di sisi server user itu sudah dianggap "logout".

---

## 7. OAuth 2.0

### Apa itu?
OAuth 2.0 adalah authorization framework yang memungkinkan satu aplikasi (client) mendapat akses terbatas ke resource milik user di aplikasi lain (resource server), tanpa harus tau password user tersebut. Contoh sehari-hari: "Login with Google" di OrderFlow — OrderFlow gak pernah lihat password Google user, cukup dapat token yang membuktikan Google sudah verifikasi identitas user itu.

### Kenapa dibutuhkan?
Membangun & maintain sistem authentication sendiri itu mahal dan berisiko (harus handle password reset, MFA, account lockout, dst). Dengan OAuth 2.0 — biasanya dikombinasikan dengan OpenID Connect buat identity — OrderFlow bisa mendelegasikan proses login ke identity provider yang sudah teruji (Google, GitHub, dsb), plus user gak perlu bikin password baru khusus buat OrderFlow.

### Cara Kerja
Authorization Code Flow, paling umum dipakai untuk web app:
```
1. User klik "Login with Google" di OrderFlow
2. OrderFlow redirect user ke Google Authorization Server (client_id, redirect_uri, scope, state)
3. User login & approve di halaman Google
4. Google redirect balik ke OrderFlow dengan authorization code
5. OrderFlow (backend) menukar code + client_secret → access token (+ id_token) ke Google
6. OrderFlow pakai id_token buat identifikasi user, atau access token buat panggil Google API
```
Istilah penting: Resource Owner (user), Client (OrderFlow), Authorization Server (Google), Resource Server (Google API), Authorization Code, Access Token, PKCE (Proof Key for Code Exchange — proteksi tambahan wajib untuk mobile/SPA yang gak bisa simpan `client_secret` dengan aman).

### Contoh Kode — Go
```go
package auth

import (
	"context"
	"encoding/json"
	"net/http"
	"os"

	"golang.org/x/oauth2"
	"golang.org/x/oauth2/google"
)

var googleOAuthConfig = &oauth2.Config{
	ClientID:     os.Getenv("GOOGLE_CLIENT_ID"),
	ClientSecret: os.Getenv("GOOGLE_CLIENT_SECRET"),
	RedirectURL:  "https://orderflow.example.com/auth/google/callback",
	Scopes:       []string{"openid", "email", "profile"},
	Endpoint:     google.Endpoint,
}

// Step 1: redirect user ke Google Authorization Server.
func GoogleLoginHandler(w http.ResponseWriter, r *http.Request) {
	state := generateRandomState() // simpan di cookie/session, dicocokkan lagi saat callback
	url := googleOAuthConfig.AuthCodeURL(state)
	http.Redirect(w, r, url, http.StatusTemporaryRedirect)
}

// Step 2: handle callback, tukar authorization code jadi token.
func GoogleCallbackHandler(w http.ResponseWriter, r *http.Request) {
	code := r.URL.Query().Get("code")

	token, err := googleOAuthConfig.Exchange(context.Background(), code)
	if err != nil {
		http.Error(w, "failed to exchange token", http.StatusUnauthorized)
		return
	}

	client := googleOAuthConfig.Client(context.Background(), token)
	resp, err := client.Get("https://www.googleapis.com/oauth2/v2/userinfo")
	if err != nil {
		http.Error(w, "failed to fetch user info", http.StatusInternalServerError)
		return
	}
	defer resp.Body.Close()

	var googleUser struct {
		Email string `json:"email"`
	}
	if err := json.NewDecoder(resp.Body).Decode(&googleUser); err != nil {
		http.Error(w, "failed to parse user info", http.StatusInternalServerError)
		return
	}

	// cari atau buat User OrderFlow berdasarkan email dari Google,
	// lalu generate access token OrderFlow sendiri seperti biasa
	user, err := findOrCreateUserByEmail(r.Context(), googleUser.Email)
	if err != nil {
		http.Error(w, "failed to resolve user", http.StatusInternalServerError)
		return
	}
	accessToken, err := GenerateAccessToken(user.ID, user.Role)
	if err != nil {
		http.Error(w, "failed to generate token", http.StatusInternalServerError)
		return
	}

	json.NewEncoder(w).Encode(map[string]string{"access_token": accessToken})
}
```

### Contoh Kode — Node.js
```javascript
const axios = require('axios');
const crypto = require('crypto');
const { generateAccessToken } = require('./auth');
const { findOrCreateUserByEmail } = require('../repositories/userRepository');

const GOOGLE_AUTH_URL = 'https://accounts.google.com/o/oauth2/v2/auth';
const GOOGLE_TOKEN_URL = 'https://oauth2.googleapis.com/token';
const REDIRECT_URI = 'https://orderflow.example.com/auth/google/callback';

// Step 1: redirect user ke Google Authorization Server.
function googleLogin(req, res) {
  const state = crypto.randomBytes(16).toString('hex');
  req.session.oauthState = state; // dicocokkan lagi saat callback, mencegah CSRF

  const params = new URLSearchParams({
    client_id: process.env.GOOGLE_CLIENT_ID,
    redirect_uri: REDIRECT_URI,
    response_type: 'code',
    scope: 'openid email profile',
    state,
  });
  res.redirect(`${GOOGLE_AUTH_URL}?${params.toString()}`);
}

// Step 2: handle callback, tukar authorization code jadi token.
async function googleCallback(req, res) {
  const { code, state } = req.query;
  if (state !== req.session.oauthState) {
    return res.status(400).json({ error: 'invalid state' });
  }

  const { data: tokenData } = await axios.post(GOOGLE_TOKEN_URL, {
    code,
    client_id: process.env.GOOGLE_CLIENT_ID,
    client_secret: process.env.GOOGLE_CLIENT_SECRET,
    redirect_uri: REDIRECT_URI,
    grant_type: 'authorization_code',
  });

  const { data: googleUser } = await axios.get(
    'https://www.googleapis.com/oauth2/v2/userinfo',
    { headers: { Authorization: `Bearer ${tokenData.access_token}` } }
  );

  const user = await findOrCreateUserByEmail(googleUser.email);
  const accessToken = generateAccessToken(user.id, user.role);

  return res.json({ access_token: accessToken });
}

module.exports = { googleLogin, googleCallback };
```

### Trade-off & Pitfall
- OAuth 2.0 itu authorization framework, bukan protokol authentication murni — untuk identity verification yang proper (tau siapa user-nya), butuh OpenID Connect (OIDC) di atas OAuth 2.0, yang menambahkan `id_token` (JWT berisi identitas).
- Parameter `state` wajib dipakai dan divalidasi buat mencegah CSRF di flow ini — jangan pernah di-skip.
- Untuk SPA/mobile app yang gak bisa simpan `client_secret` dengan aman, wajib pakai PKCE, bukan authorization code flow polos.
- OrderFlow tetap perlu maintain mapping antara identitas eksternal (Google user) dan User internal OrderFlow — biasanya lewat email atau provider user ID, dan perlu handle kasus email yang sama dari provider berbeda.

### Kapan Dipakai
Dipakai kalau OrderFlow mau menyediakan "Login with Google/GitHub" biar user gak perlu bikin akun & password baru, atau mengizinkan aplikasi pihak ketiga mengakses data OrderFlow atas nama user (misal aplikasi akuntansi yang menarik data order milik user, dengan izin eksplisit dari user itu).

### Sering Ditanya Saat Interview
- "Apa beda OAuth 2.0 dengan OpenID Connect?" — OAuth 2.0 fokus ke authorization (akses resource), OIDC adalah layer tambahan di atas OAuth 2.0 yang menstandarkan authentication (identitas user) lewat `id_token`.
- "Kenapa authorization code ditukar di backend, bukan langsung dari browser?" — supaya `client_secret` gak pernah terekspos ke browser/client side.
- "Apa itu PKCE dan kenapa dibutuhkan?" — mekanisme tambahan (`code_verifier` & `code_challenge`) buat mengamankan authorization code flow di client yang gak bisa simpan secret dengan aman, seperti SPA dan mobile app.

---

## 8. RBAC (Role-Based Access Control)

### Apa itu?
RBAC adalah model authorization di mana permission diberikan lewat role, bukan langsung ke masing-masing user. User di-assign satu atau lebih role (misal `admin`, `customer`), dan tiap role punya set permission yang sudah ditentukan.

### Kenapa dibutuhkan?
Tanpa RBAC, cek permission jadi hardcode per `user_id` ("kalau `user_id == 7`, boleh akses admin panel") yang gak scalable. Dengan RBAC, cukup tentukan role apa yang boleh ngapain sekali di awal, lalu tinggal assign role ke user. Nambah admin baru tinggal ubah kolom `role` di tabel `users`, gak perlu ubah kode.

### Cara Kerja
```
Login → GenerateAccessToken(userID, role) → role ikut dibawa di JWT claims
Request → middleware authentication ambil claims.Role dari token
        → middleware/handler authorization cek: apakah role ini boleh akses endpoint ini?
```
Karena `role` sudah ditempel di JWT claims sejak topik 3 (`GenerateAccessToken(userID int64, role string)`), RBAC di OrderFlow tinggal memanfaatkan claim itu — gak perlu query DB lagi buat tau role user.

### Contoh Kode — Go
```go
package auth

import "net/http"

// RequireRole membuat middleware yang cuma meloloskan request kalau
// role di claims ada di daftar allowedRoles.
func RequireRole(allowedRoles ...string) func(http.Handler) http.Handler {
	allowed := make(map[string]bool, len(allowedRoles))
	for _, role := range allowedRoles {
		allowed[role] = true
	}

	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			claims, ok := r.Context().Value(ClaimsContextKey).(*Claims)
			if !ok {
				http.Error(w, "unauthorized", http.StatusUnauthorized)
				return
			}
			if !allowed[claims.Role] {
				http.Error(w, "forbidden: insufficient role", http.StatusForbidden)
				return
			}
			next.ServeHTTP(w, r)
		})
	}
}
```
Pemakaian di router (chi):
```go
r.With(auth.Middleware, auth.RequireRole("admin")).
	Delete("/products/{id}", handler.DeleteProduct)

r.With(auth.Middleware, auth.RequireRole("admin", "warehouse_staff")).
	Patch("/products/{id}/stock", handler.UpdateStock)
```

### Contoh Kode — Node.js
```javascript
// requireRole membuat middleware yang cuma meloloskan request kalau
// role di req.user ada di daftar allowedRoles.
function requireRole(...allowedRoles) {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ error: 'unauthorized' });
    }
    if (!allowedRoles.includes(req.user.role)) {
      return res.status(403).json({ error: 'forbidden: insufficient role' });
    }
    next();
  };
}

module.exports = { requireRole };
```
Pemakaian di router (Express):
```javascript
const { authenticate } = require('../middleware/authenticate');
const { requireRole } = require('../middleware/requireRole');

router.delete('/products/:id', authenticate, requireRole('admin'), deleteProduct);
router.patch(
  '/products/:id/stock',
  authenticate,
  requireRole('admin', 'warehouse_staff'),
  updateStock
);
```

### Trade-off & Pitfall
- RBAC sederhana (satu role per user) gampang jadi gak cukup granular kalau kebutuhan makin kompleks — misalnya "customer_service boleh lihat semua order tapi cuma boleh refund order di bawah Rp500rb". Kasus begini butuh model lebih detail: permission-based access control (topik 9) atau ABAC (Attribute-Based Access Control).
- Karena role ditempel di JWT saat login, kalau role user diubah di DB (misal demote dari admin ke customer), perubahan itu baru efektif setelah access token lama expired — trade-off stateless JWT yang sama seperti di topik 3/6. Solusinya: TTL access token pendek, atau invalidasi via refresh token whitelist.
- Jangan cuma cek role di frontend/client — role check wajib dilakukan di backend, karena request API bisa dikirim langsung tanpa lewat UI.
- Hindari role yang terlalu granular sampai jadi ribuan kombinasi (role explosion) — kalau sudah begitu, biasanya waktunya pindah ke model permission-based.

### Kapan Dipakai
Cocok dipakai kalau jumlah role dalam sistem terbatas dan jelas (misal `customer`, `admin`, `warehouse_staff` di OrderFlow), dan kebutuhan authorization-nya cukup direpresentasikan lewat role tanpa perlu kondisi tambahan yang rumit.

### Sering Ditanya Saat Interview
- "Kenapa cukup taruh role di JWT, gak perlu query DB tiap request?" — karena tujuannya stateless: role sudah divalidasi & ditandatangani saat login, jadi server tinggal percaya isi token tanpa network call tambahan.
- "Apa risiko kalau role di JWT gak sinkron sama role terbaru di database?" — user yang di-demote/di-suspend tetap bisa pakai privilege lama sampai token-nya expired; makanya access token TTL harus pendek.
- "Kapan RBAC gak cukup dan butuh model lain?" — kalau authorization butuh mempertimbangkan atribut lain di luar role, misalnya kepemilikan resource, waktu, lokasi, atau kondisi bisnis spesifik — itu wilayah ABAC atau permission-based access control.

---

## 9. Least Privilege

### Apa itu?
Least privilege adalah prinsip keamanan: setiap user, service, atau proses cuma diberi permission minimum yang benar-benar dia butuhkan buat menjalankan tugasnya, gak lebih. Ini prinsip desain, bukan satu teknologi spesifik, dan diterapkan di banyak layer: role user, service account, kredensial database, sampai IAM policy cloud.

### Kenapa dibutuhkan?
Kalau satu service atau credential dikompromikan (bocor, di-hack), dampaknya harus terbatas sesuai permission yang dia punya. Di OrderFlow: kalau kredensial database yang dipakai `reporting service` (read-only) bocor, attacker cuma bisa baca data — gak bisa hapus/ubah order orang. Bandingkan kalau semua service pakai satu kredensial superuser yang sama: satu kebocoran kecil bisa jadi bencana besar.

### Cara Kerja
```
Tentukan: siapa/apa yang butuh akses → akses apa persisnya yang dibutuhkan → berikan hanya itu
Review berkala: apakah masih dibutuhkan? kalau ada permission yang gak kepake, cabut.
```
Penerapan konkret di OrderFlow:
- DB user berbeda untuk service berbeda: `orderflow_api` (read-write ke tabel yang relevan) vs `orderflow_reporting` (read-only, cuma ke view tertentu).
- Permission granular di atas RBAC untuk aksi sensitif, bukan cuma "admin boleh semua" (misalnya permission spesifik `orders:refund`).
- API key partner cuma dikasih scope yang dia butuhkan (misal partner logistik cuma boleh baca status order, gak boleh baca data payment).

### Contoh Kode — Go
Permission granular di atas role — least privilege diterapkan dengan gak asal kasih "admin boleh semua", tapi eksplisit per aksi:
```go
package auth

import "net/http"

var rolePermissions = map[string][]string{
	"admin":            {"orders:read", "orders:refund", "products:write"},
	"customer_service": {"orders:read", "orders:refund"}, // gak dapat products:write
	"customer":         {"orders:read"},                  // cuma order miliknya sendiri
}

func hasPermission(role, permission string) bool {
	for _, p := range rolePermissions[role] {
		if p == permission {
			return true
		}
	}
	return false
}

func RequirePermission(permission string) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			claims, ok := r.Context().Value(ClaimsContextKey).(*Claims)
			if !ok {
				http.Error(w, "unauthorized", http.StatusUnauthorized)
				return
			}
			if !hasPermission(claims.Role, permission) {
				http.Error(w, "forbidden: missing permission "+permission, http.StatusForbidden)
				return
			}
			next.ServeHTTP(w, r)
		})
	}
}
```
Pemakaian:
```go
r.With(auth.Middleware, auth.RequirePermission("orders:refund")).
	Post("/orders/{id}/refund", handler.RefundOrder)
```
Least privilege di level koneksi database — reporting service pakai role Postgres yang terpisah dari API utama:
```go
// Reporting service cuma butuh baca data, jadi pakai role Postgres read-only,
// bukan role yang sama dengan API utama yang punya akses write.
reportingPool, err := pgxpool.New(ctx, "postgres://orderflow_reporting:***@db:5432/orderflow?sslmode=require")
```

### Contoh Kode — Node.js
```javascript
// rolePermissions mendefinisikan permission spesifik per role — least privilege
// diterapkan dengan gak asal kasih "admin boleh semua", tapi eksplisit per aksi.
const rolePermissions = {
  admin: ['orders:read', 'orders:refund', 'products:write'],
  customer_service: ['orders:read', 'orders:refund'], // gak dapat products:write
  customer: ['orders:read'], // cuma order miliknya sendiri
};

function hasPermission(role, permission) {
  return (rolePermissions[role] || []).includes(permission);
}

function requirePermission(permission) {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ error: 'unauthorized' });
    }
    if (!hasPermission(req.user.role, permission)) {
      return res.status(403).json({ error: `forbidden: missing permission ${permission}` });
    }
    next();
  };
}

module.exports = { requirePermission };
```
Pemakaian:
```javascript
router.post(
  '/orders/:id/refund',
  authenticate,
  requirePermission('orders:refund'),
  refundOrder
);
```
Least privilege di level koneksi database:
```javascript
// Reporting service cuma butuh baca data, jadi pakai user Postgres read-only,
// bukan credential yang sama dengan API utama yang punya akses write.
const { Pool } = require('pg');
const reportingPool = new Pool({
  connectionString: 'postgres://orderflow_reporting:***@db:5432/orderflow',
  ssl: true,
});
```

### Trade-off & Pitfall
- Least privilege yang terlalu ketat tanpa proses yang jelas bisa bikin developer/operator kesulitan kerja normal (terlalu banyak "access denied" buat hal wajar) — butuh proses request-access yang cepat buat kasus legitimate.
- Permission-based access control (seperti contoh di atas) butuh maintenance lebih dari RBAC sederhana — setiap fitur baru berarti perlu mikir permission apa yang relevan.
- Kredensial/permission yang gak pernah direview ulang cenderung membengkak seiring waktu (permission creep) — orang pindah tim/role tapi izin lama gak pernah dicabut. Perlu audit berkala.
- Menerapkan least privilege di banyak layer (DB user, API permission, cloud IAM) berarti lebih banyak "identitas" yang harus dikelola dan diaudit, dibanding satu credential serba bisa yang lebih simpel tapi jauh lebih berisiko.

### Kapan Dipakai
Selalu, sebagai prinsip default di semua desain sistem — bukan fitur opsional. Terutama krusial untuk service account/API key partner eksternal, kredensial database antar service, dan permission granular untuk aksi sensitif (refund, delete, akses data pribadi) yang gak boleh cuma bergantung pada role umum seperti "admin".

### Sering Ditanya Saat Interview
- "Kalau credential service bocor, bagaimana least privilege membatasi dampaknya?" — dampak kebocoran terbatas sesuai permission yang dimiliki credential itu; kredensial read-only gak bisa dipakai buat mengubah/menghapus data walau sudah bocor.
- "Apa bedanya prinsip least privilege dengan RBAC?" — RBAC adalah salah satu cara mengimplementasikan authorization; least privilege adalah prinsip yang mengatur seberapa granular & minimal permission yang seharusnya diberikan, baik lewat RBAC, ABAC, atau mekanisme lain.
- "Gimana cara audit apakah prinsip least privilege sudah diterapkan dengan baik?" — review berkala permission yang ter-assign vs yang benar-benar dipakai (access log), cabut permission yang gak pernah dipakai, dan hindari shared/broad credential.

---

**Selanjutnya:** [Phase 02 — API Security & Web Security](./phase-02-api-web-security.md)
