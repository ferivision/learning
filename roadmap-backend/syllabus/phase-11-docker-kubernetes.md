# Phase 11 — Docker & Kubernetes

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

## 88. Docker

### Apa itu?
Docker adalah tool buat mengemas aplikasi (OrderFlow, baik build Go `orderflow-go` maupun build Node.js `orderflow-node`) beserta seluruh dependency-nya (runtime, library, konfigurasi OS minimal) jadi satu **image** yang bisa dijalankan konsisten di mesin manapun — laptop developer, CI, atau server produksi. **Multi-stage build** adalah teknik menulis satu `Dockerfile` dengan beberapa tahap (`FROM ... AS <nama>`): tahap awal (`builder`) berisi toolchain berat buat compile/install dependency, tahap akhir cuma menyalin hasil jadi (binary Go atau `node_modules` + source Node.js) ke base image yang jauh lebih kecil — toolchain compiler gak ikut terbawa ke image final.

### Kenapa dibutuhkan?
Tanpa multi-stage build, image OrderFlow bakal ikut membawa seluruh Go compiler toolchain (ratusan MB) atau `devDependencies` Node.js (`nodemon`, testing library, dsb) ke produksi — image jadi besar (lambat di-pull, lambat di-deploy, terutama pas scaling cepat di topik 90), dan permukaan serangannya ikut membesar karena banyak tool yang gak dipakai runtime tapi tetap ada di dalam container. Base image minimal (`distroless`/`alpine`) dan **non-root user** juga penting buat keamanan: kalau container OrderFlow kena exploit (misalnya lewat dependency yang vulnerable), proses yang jalan sebagai user biasa (bukan `root`) membatasi kerusakan yang bisa dilakukan penyerang di dalam container itu — gak bisa modifikasi file sistem yang butuh privilege root, gak bisa install package tambahan kalau base image-nya memang gak ada package manager sama sekali (`distroless`).

### Cara Kerja
```
Multi-stage build orderflow-go:

  Stage "builder" (golang:1.22)          Stage final (gcr.io/distroless/static)
  +---------------------------+          +----------------------------------+
  | go.mod, go.sum, source     |          | HANYA binary orderflow-go yang   |
  | go build -o orderflow-go   | -COPY->  | disalin dari stage builder        |
  | (compiler, cache modul,     |          | + non-root user (nonroot:nonroot) |
  |  seluruh source ikut di sini)|          | + gak ada shell, gak ada          |
  +---------------------------+          |   package manager sama sekali     |
                                          +----------------------------------+
  Image builder: ~900MB                  Image final: ~15-20MB


Multi-stage build orderflow-node:

  Stage "deps" (node:20-alpine)           Stage final (node:20-alpine)
  +---------------------------+          +----------------------------------+
  | package.json, package-lock |          | node_modules (disalin APA ADANYA |
  | npm ci --omit=dev            | -COPY->  |  dari stage deps, gak install    |
  | (cuma dependency production, |          |  ulang)                          |
  |  bukan devDependencies)      |          | + source .js yang dibutuhkan     |
  |                               |          | + non-root user (node, bawaan    |
  |                               |          |   image alpine node)             |
  +---------------------------+          +----------------------------------+
  Image deps: ~180MB                     Image final: ~150-180MB
```

### Contoh Kode — Go
`Dockerfile` untuk `orderflow-go` — stage `builder` meng-compile binary statis, stage final berbasis `distroless/static` (gak ada shell/package manager sama sekali) dan menjalankan proses sebagai non-root user bawaan `distroless`:
```dockerfile
# Dockerfile (orderflow-go)

# ---- Stage 1: builder ----
FROM golang:1.22-bookworm AS builder
WORKDIR /src

# Copy go.mod/go.sum dulu supaya layer cache modul terpisah dari layer source --
# perubahan source code gak bikin "go mod download" diulang dari nol.
COPY go.mod go.sum ./
RUN go mod download

COPY . .

# CGO_ENABLED=0 menghasilkan binary statis yang gak butuh libc dari base image --
# wajib supaya bisa jalan di distroless/static yang gak punya libc sama sekali.
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags="-s -w" -o /out/orderflow-go ./cmd/orderflow

# ---- Stage 2: final ----
FROM gcr.io/distroless/static-debian12:nonroot AS final

# distroless:nonroot base image sudah menjalankan proses sebagai UID 65532
# (user "nonroot") secara default -- gak perlu USER/RUN adduser tambahan,
# dan image ini memang gak punya shell/package manager buat dieksploitasi.
WORKDIR /app
COPY --from=builder /out/orderflow-go /app/orderflow-go

EXPOSE 8080
ENTRYPOINT ["/app/orderflow-go"]
```
`.dockerignore` supaya file yang gak relevan (test data, git history) gak ikut ke build context:
```
.git
*_test.go
tmp/
*.md
```

### Contoh Kode — Node.js
`Dockerfile` untuk `orderflow-node` — stage `deps` menginstall dependency production saja (`npm ci --omit=dev`) di layer cache-nya sendiri, stage final menyalin `node_modules` hasil install itu APA ADANYA (bukan install ulang) dan menjalankan sebagai non-root user `node` bawaan image `node:20-alpine`:
```dockerfile
# Dockerfile (orderflow-node)

# ---- Stage 1: deps ----
# Stage terpisah ini cuma install dependency production (npm ci --omit=dev)
# lewat cache layer tersendiri -- kalau cuma source code (src/) yang berubah,
# Docker gak perlu install ulang node_modules dari nol.
FROM node:20-alpine AS deps
WORKDIR /src

COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

# ---- Stage 2: final ----
FROM node:20-alpine AS final
ENV NODE_ENV=production
WORKDIR /app

# Salin HASIL install dari stage "deps" (bukan install ulang) -- npm/npm
# cache dari proses install gak ikut terbawa ke image final.
COPY --from=deps /src/node_modules ./node_modules
COPY package.json ./package.json
COPY src ./src

# Image node:20-alpine sudah menyediakan user "node" (UID 1000) bawaan --
# tinggal dipakai lewat USER, gak perlu bikin user baru manual.
USER node

EXPOSE 3000
CMD ["node", "src/server.js"]
```
`.dockerignore` yang setara untuk build Node.js:
```
.git
node_modules
*.test.js
*.md
```

### Trade-off & Pitfall
- `distroless` gak punya shell sama sekali — debugging langsung di dalam container produksi (`kubectl exec ... sh`) jadi gak mungkin; trade-off ini diterima demi permukaan serangan yang jauh lebih kecil, dan debugging dipindahkan ke observability (logs/tracing, Phase 10) alih-alih masuk ke container secara langsung.
- Multi-stage build yang lupa `.dockerignore` tetap bisa membengkak build context-nya (walau image final tetap kecil) — `node_modules` lokal atau `.git` yang ikut ter-upload ke Docker daemon bikin build lebih lambat meskipun gak masuk ke image akhir.
- Base image Node.js (`node:20-alpine`) masih membawa shell dan package manager `apk` — jauh lebih besar permukaan seranganya dibanding `distroless` yang dipakai `orderflow-go`; ini karena runtime Node.js butuh lebih banyak dependency sistem (native addon, dsb) yang lebih rumit di-strip habis dibanding binary Go yang sudah statis.
- Menjalankan container sebagai `root` (lupa `USER nonroot`/`USER node`) adalah kesalahan paling umum — kalaupun image-nya sendiri sudah minimal, proses yang jalan sebagai root di dalam container tetap membuka risiko lebih besar kalau ada container breakout vulnerability di level runtime container itu sendiri.

### Kapan Dipakai
Dipakai buat setiap service OrderFlow yang bakal dideploy — baik `orderflow-go` maupun `orderflow-node` wajib punya image yang immutable dan reproducible sebelum masuk ke Kubernetes (topik 89). Untuk skrip development lokal sekali pakai yang gak pernah di-deploy, overhead menulis multi-stage Dockerfile yang optimal biasanya gak sepadan — `docker-compose` dengan image dev biasa sudah cukup.

### Sering Ditanya Saat Interview
- "Kenapa pakai multi-stage build, bukan cukup satu `FROM` aja?" — supaya toolchain berat (Go compiler, `devDependencies` Node.js) yang cuma dibutuhkan saat build gak ikut terbawa ke image final; image jadi lebih kecil (lebih cepat di-pull/deploy) dan permukaan serangannya lebih kecil karena tool yang gak dipakai runtime gak ada di dalamnya.
- "Kenapa `orderflow-go` bisa pakai `distroless/static` yang bahkan gak ada libc, sementara `orderflow-node` gak bisa?" — binary Go dikompilasi jadi statis (`CGO_ENABLED=0`) sehingga gak butuh libc atau dependency sistem apapun saat runtime; Node.js sendiri adalah runtime interpreter yang butuh binary `node` beserta dependency sistemnya, jadi base image-nya perlu minimal punya runtime Node.js utuh (walau tetap bisa pakai varian alpine yang minim).
- "Kenapa container harus jalan sebagai non-root user?" — supaya kalau ada vulnerability yang berhasil dieksploitasi di dalam container, kerusakan yang bisa dilakukan penyerang terbatas ke privilege user biasa, bukan privilege root yang bisa memodifikasi file sistem atau, dalam skenario container breakout, berpotensi mempengaruhi host lebih jauh.

---

## 89. Kubernetes

### Apa itu?
Kubernetes adalah orchestrator yang menjalankan dan mengelola container (image dari topik 88) di sekumpulan mesin (cluster). Resource intinya: **Pod** (unit terkecil, membungkus satu atau lebih container yang jalan bersama), **Deployment** (mengelola sejumlah replika Pod, menangani rolling update, dan otomatis membuat ulang Pod yang mati), **Service** (endpoint stabil dengan satu nama/IP buat menjangkau sekumpulan Pod yang IP-nya berubah-ubah), **ConfigMap** (konfigurasi non-rahasia seperti `LOG_LEVEL`), **Secret** (data sensitif seperti connection string database, disimpan ter-encode base64 dan bisa dienkripsi lebih lanjut di level cluster), **Ingress** (routing HTTP dari luar cluster ke Service berdasarkan host/path), dan **Namespace** (partisi logis dalam satu cluster buat memisahkan resource, misalnya `orderflow-go` dan `orderflow-node` dipisah biar gak saling tabrak nama).

### Kenapa dibutuhkan?
Menjalankan container secara manual (`docker run`) gak sanggup menjawab pertanyaan produksi: kalau satu instance OrderFlow crash, siapa yang otomatis menjalankannya lagi? Kalau traffic naik, siapa yang menyebar request ke banyak instance sekaligus? Kalau connection string Postgres (Phase 3) perlu diganti, gimana caranya tanpa menyalakan ulang satu-satu secara manual di setiap mesin? Deployment menjawab pertanyaan pertama-kedua (self-healing + rolling update), Service menjawab load balancing antar Pod, ConfigMap/Secret memisahkan konfigurasi dari image (image yang sama bisa dipakai di staging maupun produksi cukup dengan ConfigMap/Secret yang beda), dan Ingress menjawab bagaimana traffic dari internet sampai ke Service yang tepat berdasarkan hostname (`api.orderflow.example.com` vs `legacy.orderflow.example.com` kalau `orderflow-go` dan `orderflow-node` dijalankan berdampingan selama migrasi).

### Cara Kerja
```
Traffic dari internet:

  Ingress (host-based routing)
    |
    +-- host: go.orderflow.example.com   --> Service "orderflow-go-svc"
    |                                          |
    |                                          +--> Pod (orderflow-go) replica 1
    |                                          +--> Pod (orderflow-go) replica 2
    |                                          +--> Pod (orderflow-go) replica 3
    |
    +-- host: node.orderflow.example.com --> Service "orderflow-node-svc"
                                               |
                                               +--> Pod (orderflow-node) replica 1
                                               +--> Pod (orderflow-node) replica 2

  Deployment "orderflow-go"    : menjaga 3 Pod di atas tetap hidup,
                                 baca config dari ConfigMap + Secret
  Deployment "orderflow-node"  : menjaga 2 Pod di atas tetap hidup,
                                 baca config dari ConfigMap + Secret

  Kedua Deployment tinggal di Namespace "orderflow" yang sama, tapi
  dipisahkan lewat label/nama resource supaya gak saling tabrak.
```

### Contoh Kode — Go
Manifest lengkap untuk `orderflow-go`: `Namespace`, `ConfigMap`, `Secret`, `Deployment` (2 container port terpisah, image `orderflow-go`), dan `Service`:
```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: orderflow
---
# orderflow-go-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: orderflow-go-config
  namespace: orderflow
data:
  LOG_LEVEL: "info"
  HTTP_PORT: "8080"
---
# orderflow-go-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: orderflow-go-secret
  namespace: orderflow
type: Opaque
stringData:
  DATABASE_URL: "postgres://orderflow:changeme@postgres:5432/orderflow"
  REDIS_URL: "redis://redis:6379/0"
---
# orderflow-go-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orderflow-go
  namespace: orderflow
  labels:
    app: orderflow-go
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orderflow-go
  template:
    metadata:
      labels:
        app: orderflow-go
    spec:
      containers:
        - name: orderflow-go
          image: registry.example.com/orderflow-go:1.4.0
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: orderflow-go-config
            - secretRef:
                name: orderflow-go-secret
          livenessProbe:
            httpGet:
              path: /healthz/live
              port: 8080
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /healthz/ready
              port: 8080
            periodSeconds: 5
---
# orderflow-go-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: orderflow-go-svc
  namespace: orderflow
spec:
  selector:
    app: orderflow-go
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
---
# orderflow-go-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: orderflow-go-ingress
  namespace: orderflow
spec:
  rules:
    - host: go.orderflow.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: orderflow-go-svc
                port:
                  number: 80
```

### Contoh Kode — Node.js
Manifest yang setara untuk `orderflow-node` — image, port container, dan hostname Ingress-nya berbeda dari versi Go:
```yaml
# orderflow-node-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: orderflow-node-config
  namespace: orderflow
data:
  LOG_LEVEL: "info"
  HTTP_PORT: "3000"
---
# orderflow-node-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: orderflow-node-secret
  namespace: orderflow
type: Opaque
stringData:
  DATABASE_URL: "postgres://orderflow:changeme@postgres:5432/orderflow"
  REDIS_URL: "redis://redis:6379/0"
---
# orderflow-node-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orderflow-node
  namespace: orderflow
  labels:
    app: orderflow-node
spec:
  replicas: 2
  selector:
    matchLabels:
      app: orderflow-node
  template:
    metadata:
      labels:
        app: orderflow-node
    spec:
      containers:
        - name: orderflow-node
          image: registry.example.com/orderflow-node:1.4.0
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: orderflow-node-config
            - secretRef:
                name: orderflow-node-secret
          livenessProbe:
            httpGet:
              path: /healthz/live
              port: 3000
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /healthz/ready
              port: 3000
            periodSeconds: 5
---
# orderflow-node-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: orderflow-node-svc
  namespace: orderflow
spec:
  selector:
    app: orderflow-node
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
---
# orderflow-node-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: orderflow-node-ingress
  namespace: orderflow
spec:
  rules:
    - host: node.orderflow.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: orderflow-node-svc
                port:
                  number: 80
```

### Trade-off & Pitfall
- Menyimpan data sensitif di `Secret` cuma meng-encode base64, BUKAN mengenkripsinya — siapapun yang punya akses baca ke Secret tersebut (lewat RBAC yang longgar, topik 91) bisa dengan mudah men-decode-nya; enkripsi-at-rest untuk `etcd` (tempat Kubernetes menyimpan semua object termasuk Secret) harus diaktifkan terpisah di level cluster.
- `livenessProbe` dan `readinessProbe` di manifest ini memanggil endpoint yang sama persis dengan yang dibangun di Phase 9 (topik 78) — kalau path atau port-nya salah ketik di manifest, Kubernetes bisa salah kesimpulan Pod gak pernah "siap" walau aplikasinya sebenarnya sehat, atau sebaliknya restart Pod yang sebenarnya baik-baik saja.
- Deployment tanpa `resources.requests`/`resources.limits` (sengaja disederhanakan di contoh di atas) bisa membuat satu Pod yang bocor memory menghabiskan seluruh resource node dan mempengaruhi Pod lain di node yang sama — di produksi nyata, requests/limits CPU & memory wajib diset eksplisit.
- Memisahkan `orderflow-go` dan `orderflow-node` jadi Deployment & Service terpisah (bukan satu Deployment gabungan) memudahkan scaling independen (topik 90) dan rollout independen, tapi berarti dua kali lipat manifest yang harus dijaga tetap konsisten satu sama lain.

### Kapan Dipakai
Dipakai begitu OrderFlow perlu dijalankan dengan lebih dari satu instance secara reliable, butuh self-healing otomatis, dan butuh cara terstruktur memisahkan konfigurasi dari image (multi-environment: staging vs produksi). Untuk proof-of-concept atau demo internal yang dijalankan seorang diri di satu mesin, `docker run`/`docker-compose` biasa tanpa Kubernetes sudah cukup dan jauh lebih sederhana.

### Sering Ditanya Saat Interview
- "Apa beda Pod dan Deployment?" — Pod adalah unit terkecil yang benar-benar menjalankan container, tapi Pod yang dibuat manual gak akan otomatis dibuat ulang kalau mati; Deployment adalah lapisan di atasnya yang menjaga jumlah replika Pod tertentu tetap hidup dan menangani rolling update saat image berganti versi.
- "Kenapa `orderflow-go` dan `orderflow-node` dipisah jadi Deployment & Service sendiri-sendiri, bukan digabung satu Deployment?" — supaya keduanya bisa di-scale (topik 90) dan di-rollout secara independen — menaikkan replika `orderflow-node` gak perlu ikut menyentuh `orderflow-go` sama sekali, dan sebaliknya.
- "Kenapa Secret di Kubernetes dianggap belum cukup aman hanya dengan base64 encoding?" — base64 adalah encoding, bukan enkripsi — gampang di-decode siapapun yang bisa membaca object Secret-nya; keamanan sebenarnya bergantung pada RBAC yang ketat (topik 91) dan enkripsi-at-rest `etcd` yang dikonfigurasi terpisah di level cluster.

---

## 90. Kubernetes Scaling

### Apa itu?
HorizontalPodAutoscaler (HPA) adalah resource Kubernetes yang otomatis menambah atau mengurangi jumlah replika Pod pada sebuah Deployment (topik 89), berdasarkan metrik yang diamati — paling umum rata-rata pemakaian CPU/memory di seluruh Pod, tapi bisa juga metrik custom (misalnya panjang antrean, topik 47-52 Phase 5). Ini beda dengan vertical scaling (memperbesar resource satu Pod) — HPA melakukan horizontal scaling, menambah JUMLAH Pod yang identik.

### Kenapa dibutuhkan?
Traffic checkout OrderFlow gak konstan — flash sale bisa mendatangkan traffic 10x lipat dibanding jam normal (skenario yang sama seperti yang dibahas backpressure di topik 80), sementara jam tengah malam traffic-nya jauh lebih sepi. Menetapkan `replicas` Deployment secara statis berarti harus dipilih salah satu dari dua kondisi buruk: dipatok tinggi terus (boros resource & biaya di jam sepi) atau dipatok rendah (Pod kewalahan dan mulai menolak request lewat backpressure saat flash sale). HPA menyesuaikan jumlah replika secara otomatis mengikuti beban aktual, sehingga OrderFlow cuma memakai (dan membayar) resource sebanyak yang benar-benar dibutuhkan pada waktu tertentu.

### Cara Kerja
```
HPA memantau tiap N detik (default 15 detik):

  rata-rata CPU utilization seluruh Pod Deployment
        |
        v
  target CPU (misal 70%) vs CPU aktual sekarang
        |
        +-- CPU aktual JAUH DI ATAS target -----> tambah replika (scale up)
        |                                          (hingga maxReplicas)
        |
        +-- CPU aktual JAUH DI BAWAH target -----> kurangi replika (scale down)
        |                                          (hingga minReplicas)
        |
        +-- CPU aktual sekitar target -----------> jumlah replika tetap

Contoh nyata orderflow-go (traffic checkout naik saat flash sale):

  minReplicas=3, maxReplicas=20, target CPU=70%

  traffic normal   : CPU rata-rata 40%  -> tetap 3 replika
  flash sale mulai : CPU rata-rata 90%  -> HPA naikkan bertahap ke ~7-8 replika
  flash sale reda   : CPU turun ke 30%  -> HPA turunkan bertahap kembali ke 3 replika
```

### Contoh Kode — Go
`HorizontalPodAutoscaler` untuk Deployment `orderflow-go` (topik 89) — target CPU 70%, boleh naik sampai 20 replika karena checkout adalah jalur yang paling sensitif terhadap lonjakan traffic:
```yaml
# orderflow-go-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: orderflow-go-hpa
  namespace: orderflow
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: orderflow-go
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleDown:
      # Tunggu 5 menit stabil sebelum mulai scale down, supaya HPA gak
      # "flapping" (naik-turun replika berulang) kalau CPU cuma sesaat turun.
      stabilizationWindowSeconds: 300
    scaleUp:
      stabilizationWindowSeconds: 0
```

### Contoh Kode — Node.js
`HorizontalPodAutoscaler` untuk Deployment `orderflow-node` — target CPU lebih rendah (60%) karena event loop Node.js yang single-threaded lebih cepat terasa lambat di CPU tinggi dibanding goroutine Go, dan `maxReplicas` lebih kecil karena traffic yang ditangani `orderflow-node` di skenario ini lebih ringan:
```yaml
# orderflow-node-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: orderflow-node-hpa
  namespace: orderflow
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: orderflow-node
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
    scaleUp:
      stabilizationWindowSeconds: 0
```

### Trade-off & Pitfall
- HPA berbasis CPU/memory butuh `resources.requests` sudah diset di Deployment (topik 89) — tanpa itu, Kubernetes gak punya baseline buat menghitung persentase utilization, dan HPA gak akan berfungsi sama sekali walau manifest-nya sudah benar.
- Scale up butuh waktu (Pod baru harus di-schedule, image di-pull kalau belum ada di node, container start, lolos readinessProbe topik 78 dulu baru mulai terima traffic) — HPA bukan solusi instan buat lonjakan traffic yang sangat mendadak; kombinasi dengan backpressure (topik 80) tetap dibutuhkan buat menahan periode singkat sebelum Pod baru siap.
- `stabilizationWindowSeconds` yang terlalu pendek di scale down bikin HPA gampang flapping (naik-turun replika berulang) kalau traffic-nya naik-turun sesaat; terlalu panjang bikin resource yang gak lagi dibutuhkan tetap dibayar lebih lama dari seharusnya.
- Scaling berbasis CPU gak selalu merepresentasikan beban sebenarnya — untuk beban yang lebih terkait panjang antrean job (Phase 5) daripada CPU, metrik custom (via Prometheus Adapter, misalnya panjang queue) jauh lebih akurat dibanding CPU utilization semata.

### Kapan Dipakai
Dipakai buat Deployment yang traffic-nya berfluktuasi signifikan dan gak bisa diprediksi dengan angka replika tetap — checkout OrderFlow adalah kandidat jelas karena lonjakan flash sale. Untuk service internal dengan traffic yang benar-benar stabil dan bisa diprediksi (misalnya cron job internal), replika tetap tanpa HPA lebih sederhana dan gak menambah kompleksitas operasional yang gak perlu.

### Sering Ditanya Saat Interview
- "Kenapa HPA butuh `resources.requests` sudah diset di Deployment?" — HPA menghitung utilization sebagai persentase dari `requests`, bukan dari kapasitas node secara absolut; tanpa `requests`, Kubernetes gak tahu "100%" itu berapa, jadi HPA gak bisa menghitung target sama sekali.
- "Kenapa scale down butuh stabilization window yang lebih panjang dibanding scale up?" — supaya HPA gak buru-buru mengurangi replika cuma karena penurunan traffic sesaat, yang kalau traffic naik lagi sesaat kemudian bakal memicu scale up lagi — flapping seperti ini boros resource (constant Pod create/destroy) dan bisa bikin sebagian request kena backpressure (topik 80) di masa transisi.
- "Apa keterbatasan HPA saat menghadapi lonjakan traffic yang sangat tiba-tiba (spike)?" — Pod baru butuh waktu untuk di-schedule, start, dan lolos readiness probe sebelum bisa menerima traffic, jadi ada jeda antara traffic naik dan kapasitas benar-benar bertambah; selama jeda itu, backpressure (topik 80) dan circuit breaker (topik 79) yang menahan sistem supaya gak kolaps duluan.

---

## 91. Kubernetes Security

### Apa itu?
Sekumpulan mekanisme Kubernetes buat membatasi apa yang boleh dilakukan sebuah Pod dan siapa yang boleh mengaksesnya: **RBAC (Role-Based Access Control)** membatasi operasi apa (get, list, create, delete) yang boleh dilakukan terhadap resource Kubernetes (Pod, Secret, dll) oleh identitas tertentu; **ServiceAccount** adalah identitas yang dipakai Pod (bukan manusia) saat memanggil Kubernetes API dari dalam cluster; **Role** & **RoleBinding** mendefinisikan izin itu dan mengikatnya ke ServiceAccount tertentu dalam satu namespace; **NetworkPolicy** membatasi koneksi jaringan mana yang boleh masuk/keluar dari sekumpulan Pod, di level jaringan cluster itu sendiri (bukan cuma di level aplikasi).

### Kenapa dibutuhkan?
Secara default, Kubernetes cenderung longgar: ServiceAccount default sebuah Pod, kalau gak dibatasi eksplisit lewat RBAC, kadang punya izin lebih dari yang dibutuhkan aplikasi tersebut, dan secara default semua Pod dalam satu cluster bisa saling terhubung tanpa batasan jaringan sama sekali. Kalau container `orderflow-node` (yang lebih sering mengonsumsi dependency pihak ketiga npm dengan permukaan supply-chain attack lebih besar) berhasil dieksploitasi, tanpa RBAC yang ketat penyerang yang sudah masuk ke dalam Pod bisa memakai ServiceAccount Pod tersebut buat membaca Secret lain di namespace yang sama (termasuk Secret milik `orderflow-go`, topik 89) — padahal `orderflow-node` semestinya cuma butuh baca ConfigMap/Secret miliknya sendiri. Tanpa NetworkPolicy, Pod yang sama juga bisa mencoba terhubung langsung ke Postgres (Phase 3) walau semestinya cuma `orderflow-go` dan `orderflow-node` yang boleh, bukan Pod lain di cluster yang gak berkepentingan.

### Cara Kerja
```
Tanpa RBAC/NetworkPolicy yang ketat (default longgar):

  Pod orderflow-node (ter-compromise) ---> bisa akses API server dengan
                                            ServiceAccount default yang
                                            terlalu permisif
                                            |
                                            +--> baca Secret orderflow-go
                                            +--> koneksi bebas ke Postgres,
                                                 Redis, Pod lain di cluster


Dengan RBAC + NetworkPolicy diterapkan:

  ServiceAccount "orderflow-node-sa"
        |
        v (RoleBinding)
  Role "orderflow-node-role" -- izin TERBATAS:
        get/list ConfigMap & Secret MILIK orderflow-node SAJA
        (gak ada izin ke Secret orderflow-go, gak ada izin create/delete Pod)

  NetworkPolicy "orderflow-node-netpol" -- traffic keluar/masuk TERBATAS:
        ingress: HANYA dari Ingress controller
        egress : HANYA ke Postgres (port 5432) & Redis (port 6379)
                 (gak bisa konek ke Pod lain di cluster yang gak relevan)
```

### Contoh Kode — Go
`ServiceAccount`, `Role`, `RoleBinding`, dan `NetworkPolicy` untuk Deployment `orderflow-go` (topik 89) — izin RBAC dibatasi hanya baca ConfigMap/Secret miliknya sendiri, dan traffic keluar dibatasi hanya ke Postgres & Redis:
```yaml
# orderflow-go-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orderflow-go-sa
  namespace: orderflow
---
# orderflow-go-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: orderflow-go-role
  namespace: orderflow
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["orderflow-go-config"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["orderflow-go-secret"]
    verbs: ["get"]
---
# orderflow-go-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: orderflow-go-rolebinding
  namespace: orderflow
subjects:
  - kind: ServiceAccount
    name: orderflow-go-sa
    namespace: orderflow
roleRef:
  kind: Role
  name: orderflow-go-role
  apiGroup: rbac.authorization.k8s.io
---
# orderflow-go-networkpolicy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: orderflow-go-netpol
  namespace: orderflow
spec:
  podSelector:
    matchLabels:
      app: orderflow-go
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    # DNS resolution wajib diizinkan eksplisit -- tanpa ini, egress ke
    # Postgres/Redis di atas gak akan bisa resolve hostname-nya sama sekali.
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
```

### Contoh Kode — Node.js
Manifest yang setara untuk `orderflow-node` — nama resource dan `podSelector` merujuk ke Deployment `orderflow-node`, tapi struktur izin RBAC & pembatasan egress-nya identik secara prinsip:
```yaml
# orderflow-node-serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: orderflow-node-sa
  namespace: orderflow
---
# orderflow-node-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: orderflow-node-role
  namespace: orderflow
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["orderflow-node-config"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["orderflow-node-secret"]
    verbs: ["get"]
---
# orderflow-node-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: orderflow-node-rolebinding
  namespace: orderflow
subjects:
  - kind: ServiceAccount
    name: orderflow-node-sa
    namespace: orderflow
roleRef:
  kind: Role
  name: orderflow-node-role
  apiGroup: rbac.authorization.k8s.io
---
# orderflow-node-networkpolicy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: orderflow-node-netpol
  namespace: orderflow
spec:
  podSelector:
    matchLabels:
      app: orderflow-node
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 3000
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
```

### Trade-off & Pitfall
- `resourceNames` di dalam `Role` (membatasi ke ConfigMap/Secret spesifik, bukan semua ConfigMap/Secret di namespace) membuat RBAC lebih ketat tapi juga lebih verbose — tiap kali ada ConfigMap/Secret baru buat service yang sama, `Role`-nya harus diupdate eksplisit; ini trade-off yang sepadan dibanding izin longgar `resources: ["secrets"]` tanpa `resourceNames` yang berarti "boleh baca SEMUA Secret di namespace ini".
- NetworkPolicy defaultnya "deny all" begitu ada SATU NetworkPolicy yang menyeleksi sebuah Pod (lewat `podSelector`) — lupa mengizinkan egress ke DNS (port 53/UDP) adalah pitfall paling umum, karena tanpa itu Pod-nya gak bisa resolve hostname Postgres/Redis sama sekali walau rule egress ke IP-nya sendiri sudah benar.
- RBAC yang terlalu ketat (misalnya lupa mengizinkan `watch` selain `get`/`list`) bisa membuat aplikasi yang mengandalkan mekanisme watch ConfigMap/Secret (auto-reload config tanpa restart Pod) gagal diam-diam tanpa error yang jelas — perlu diuji end-to-end, bukan cuma diasumsikan izinnya sudah cukup dari membaca manifest saja.
- NetworkPolicy hanya efektif kalau CNI plugin yang dipakai cluster memang mendukungnya (misalnya Calico, Cilium) — beberapa CNI plugin dasar mengabaikan resource `NetworkPolicy` sepenuhnya tanpa error, sehingga manifest-nya "valid" tapi gak benar-benar diterapkan; ini wajib diverifikasi di cluster yang dipakai, bukan diasumsikan otomatis berfungsi di semua distribusi Kubernetes.

### Kapan Dipakai
Wajib diterapkan di cluster produksi apapun yang menjalankan lebih dari satu service dalam satu namespace atau cluster yang sama — RBAC dan NetworkPolicy adalah pertahanan berlapis (defense in depth) yang membatasi dampak kalau salah satu service (misalnya `orderflow-node` lewat dependency npm yang vulnerable) berhasil dieksploitasi, supaya kompromi itu gak otomatis menyebar ke service lain (`orderflow-go`) atau ke Secret/data yang gak seharusnya terjangkau. Untuk cluster development lokal single-tenant yang gak pernah menyimpan data sensitif sungguhan, RBAC/NetworkPolicy custom seketat ini biasanya berlebihan dan cukup pakai default cluster.

### Sering Ditanya Saat Interview
- "Kenapa `orderflow-go` dan `orderflow-node` masing-masing butuh ServiceAccount & Role sendiri, bukan berbagi satu ServiceAccount?" — supaya kalau salah satu (misalnya `orderflow-node`) ter-compromise, ServiceAccount yang dipegang penyerang cuma punya izin terbatas ke resource milik `orderflow-node` sendiri, gak otomatis bisa membaca Secret milik `orderflow-go` — prinsip least privilege, membatasi blast radius satu kompromi.
- "Apa yang terjadi kalau NetworkPolicy dipasang tapi lupa mengizinkan egress ke DNS?" — Pod-nya gak akan bisa resolve hostname (misalnya nama Service Postgres/Redis) sama sekali, karena begitu ada NetworkPolicy yang menyeleksi sebuah Pod, semua traffic yang gak eksplisit diizinkan otomatis ditolak, termasuk query DNS yang sebelumnya berjalan tanpa perlu diizinkan secara eksplisit.
- "Kenapa RBAC dan NetworkPolicy dianggap pelengkap, bukan pengganti, keamanan level aplikasi (Phase 1 & 2)?" — RBAC/NetworkPolicy melindungi di level infrastruktur Kubernetes (siapa boleh akses API server, Pod mana boleh konek ke mana), sementara autentikasi/otorisasi user (Phase 1) dan proteksi API dari abuse (Phase 2) melindungi di level request HTTP aplikasi itu sendiri — keduanya dibutuhkan sekaligus sebagai lapisan pertahanan yang berbeda, bukan salah satu menggantikan yang lain.

---

**Selanjutnya:** [Phase 12 — CI/CD & Deployment](./phase-12-cicd-deployment.md)
