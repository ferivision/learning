# Phase 12 — CI/CD & Deployment

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

## 92. CI/CD

### Apa itu?
CI/CD adalah praktik mengotomatiskan tahapan antara developer menulis kode sampai kode itu jadi image yang siap dideploy — **Continuous Integration (CI)** adalah bagian yang otomatis menjalankan test (dan build) setiap kali ada perubahan kode masuk, supaya bug ketahuan sedini mungkin sebelum sempat digabung ke branch utama; **Continuous Deployment/Delivery (CD)** adalah bagian yang otomatis membuat artifact yang siap dipakai (di OrderFlow: image Docker `orderflow-go`/`orderflow-node` dari Phase 11, di-push ke container registry) begitu CI lolos. **GitHub Actions** adalah salah satu tool buat menjalankan pipeline ini: satu file YAML `.yml` mendefinisikan satu **workflow**, dipicu oleh **trigger** (`on:`, misalnya `push` atau `pull_request`), berisi satu atau lebih **job** yang jalan di **runner** (mesin virtual sekali pakai), dan tiap job berisi urutan **step**.

### Kenapa dibutuhkan?
Tanpa CI/CD, tiap developer OrderFlow harus menjalankan `go test`/`npm test` secara manual sebelum push, gampang lupa atau di-skip pas buru-buru, dan proses build image + push ke registry (`registry.example.com/orderflow-go`, `registry.example.com/orderflow-node`) dilakukan manual dari laptop masing-masing — rawan beda hasil (versi Go/Node.js beda, environment variable lokal ikut kebawa) dibanding kalau dijalankan di environment yang bersih dan konsisten tiap kali. CI/CD memastikan **setiap** perubahan kode divalidasi test yang sama persis, dan **setiap** image yang di-push ke registry sudah pasti lolos test itu duluan — gak ada image yang sampai ke registry (apalagi ke Kubernetes, topik 89) tanpa lolos tahap validasi otomatis.

### Cara Kerja
```
Push ke branch "main" (orderflow-go atau orderflow-node):

  Trigger (on: push)
        |
        v
  Job "test"     : checkout kode -> setup toolchain -> jalankan test suite
        |            (kalau GAGAL, pipeline berhenti di sini -- job "build"
        |             dan "docker-push" gak pernah dijalankan sama sekali)
        v
  Job "build"    : checkout kode -> setup toolchain -> compile/lint
        |            (needs: test -- gak jalan kalau job "test" gagal)
        v
  Job "docker-push" : login ke registry -> docker build (pakai Dockerfile
        |              multi-stage dari Phase 11) -> docker push
        |              (needs: build, DAN if: hanya di branch "main" --
        |               pull_request cukup divalidasi test+build, gak
        |               perlu bikin image baru di registry)
        v
  Image baru muncul di registry.example.com/orderflow-go (atau -node)
  dengan tag <git-sha> DAN "latest" -- siap dipakai di manifest Deployment
  Kubernetes topik 93 lewat ganti field `image:`.
```

### Contoh Kode — Go
Workflow GitHub Actions untuk `orderflow-go` — tiga job berurutan lewat `needs:`: `test` (unit test dengan race detector), `build` (memastikan binary statis yang dipakai Dockerfile Phase 11 memang bisa di-compile), lalu `docker-push` (build image dari `Dockerfile` yang sama persis dengan Phase 11 dan push ke registry, HANYA di branch `main`):
```yaml
# .github/workflows/orderflow-go-ci.yml
name: orderflow-go CI/CD

on:
  push:
    branches: [main]
    paths:
      - "cmd/orderflow/**"
      - "go.mod"
      - "go.sum"
      - "Dockerfile"
  pull_request:
    branches: [main]

env:
  IMAGE_NAME: registry.example.com/orderflow-go

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.22"
      - name: Download dependencies
        run: go mod download
      - name: Run unit tests
        run: go test ./... -race -cover

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.22"
      - name: Compile static binary
        run: |
          CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
            go build -ldflags="-s -w" -o /tmp/orderflow-go ./cmd/orderflow

  docker-push:
    needs: build
    # Cuma jalan di branch main -- pull_request cukup divalidasi lewat
    # job test & build di atas, gak perlu bikin image baru di registry.
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Log in to container registry
        uses: docker/login-action@v3
        with:
          registry: registry.example.com
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}
      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.IMAGE_NAME }}:latest
```

### Contoh Kode — Node.js
Workflow yang setara untuk `orderflow-node` — struktur job identik (`test` -> `build` -> `docker-push`), tapi toolchain-nya `setup-node` dan step test/build-nya pakai `npm`:
```yaml
# .github/workflows/orderflow-node-ci.yml
name: orderflow-node CI/CD

on:
  push:
    branches: [main]
    paths:
      - "src/**"
      - "package.json"
      - "package-lock.json"
      - "Dockerfile"
  pull_request:
    branches: [main]

env:
  IMAGE_NAME: registry.example.com/orderflow-node

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
      - name: Install dependencies
        run: npm ci
      - name: Run unit tests
        run: npm test -- --ci

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"
      - name: Install production dependencies
        run: npm ci --omit=dev
      - name: Lint
        run: npm run lint

  docker-push:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Log in to container registry
        uses: docker/login-action@v3
        with:
          registry: registry.example.com
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}
      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE_NAME }}:${{ github.sha }}
            ${{ env.IMAGE_NAME }}:latest
```

### Trade-off & Pitfall
- `needs:` bikin job berurutan (test -> build -> docker-push) yang lebih aman tapi lebih lambat dibanding menjalankan semuanya paralel — trade-off ini diterima karena gak masuk akal bikin image dari kode yang bahkan belum lolos test.
- Kredensial registry (`REGISTRY_USERNAME`/`REGISTRY_PASSWORD`) disimpan sebagai **GitHub Actions secret**, bukan hardcoded di YAML — kalau workflow file (yang publik/ter-review) sampai memuat kredensial asli, siapapun yang bisa baca repository bisa push image palsu mengatasnamakan `orderflow-go`/`orderflow-node`.
- Tag `latest` itu **mutable** (bisa berubah tiap push) dan gak boleh dipakai buat field `image:` di manifest Kubernetes produksi (topik 93) — pakai tag `${{ github.sha }}` yang immutable, supaya jelas persis versi kode mana yang lagi jalan dan gampang di-rollback ke SHA sebelumnya.
- CI cuma seketat test suite yang ditulis — job `test` hijau BUKAN jaminan gak ada bug sama sekali, cuma jaminan gak ada regresi di skenario yang sudah dites; test yang lemah/gak lengkap tetap bisa meloloskan image yang bermasalah ke registry.

### Kapan Dipakai
Wajib buat setiap service OrderFlow yang bakal dideploy ke produksi — `orderflow-go` dan `orderflow-node` sama-sama butuh pipeline ini supaya image yang sampai ke registry (dan akhirnya ke Kubernetes Deployment topik 93) selalu sudah lolos test yang sama. Untuk skrip sekali pakai atau proof-of-concept yang gak pernah dideploy ke lingkungan bersama, overhead menyiapkan workflow CI/CD biasanya gak sepadan.

### Sering Ditanya Saat Interview
- "Kenapa job `test`, `build`, dan `docker-push` dipisah, bukan digabung satu job saja?" — supaya pipeline bisa berhenti sedini mungkin (fail fast) kalau test gagal, gak perlu buang waktu lanjut compile atau build image; `needs:` juga bikin dependency-nya eksplisit dan gampang dibaca urutannya.
- "Kenapa job `docker-push` dibatasi `if: github.ref == 'refs/heads/main'`?" — supaya image baru cuma dibuat dari kode yang sudah masuk branch utama (sudah lolos review + merge), bukan dari tiap branch pull_request yang masih dalam proses — kalau enggak, registry bakal penuh image dari branch eksperimen yang gak jelas siap pakai atau enggak.
- "Kenapa image di-tag dengan `github.sha` selain `latest`?" — `latest` berubah-ubah tiap push jadi gak bisa dijadikan referensi versi yang pasti, sementara tag berbasis git SHA bersifat immutable — manifest Kubernetes (topik 93) bisa merujuk versi spesifik itu, dan kalau versi baru bermasalah, rollback tinggal ganti balik ke tag SHA sebelumnya yang jelas isinya.

---

## 93. Deployment Strategies

### Apa itu?
Deployment strategy adalah cara mengganti versi lama sebuah service dengan versi baru di Kubernetes tanpa (atau minim) downtime. Tiga yang paling umum: **Rolling update** — Pod versi lama diganti Pod versi baru secara bertahap satu/beberapa demi satu, traffic terus mengalir ke Pod mana pun yang lagi hidup dan siap (lewat `readinessProbe`, topik 89); **Blue-Green** — versi baru ("green") disiapkan penuh berdampingan dengan versi lama ("blue") yang masih menerima 100% traffic, lalu traffic dialihkan **sekaligus** dari blue ke green begitu green terverifikasi siap, dengan blue tetap standby buat rollback instan; **Canary** — versi baru dirilis ke **sebagian kecil** traffic (misalnya 10%) berdampingan dengan versi lama yang tetap menangani sisanya, dipantau dulu sebelum diputuskan naik bertahap ke 100% atau di-rollback.

### Kenapa dibutuhkan?
Mengganti seluruh Pod sekaligus (misalnya `kubectl delete` semua Pod versi lama baru bikin yang baru) berarti ada jeda di mana OrderFlow gak punya Pod yang siap melayani traffic checkout sama sekali — downtime penuh. Rolling update menghindari downtime dengan mengganti Pod bertahap, tapi tetap mengekspos 100% Pod baru ke traffic dalam waktu relatif singkat begitu rollout selesai — kalau versi barunya ternyata ada bug yang cuma muncul di beban produksi nyata (bukan ketahuan di test, topik 92), semua traffic sudah kena. Blue-Green dan Canary menjawab kekhawatiran itu dengan cara berbeda: Blue-Green memungkinkan rollback instan (tinggal alihkan balik ke blue) karena versi lama masih hidup utuh, sementara Canary membatasi **blast radius** bug versi baru ke sebagian kecil user dulu sebelum dipercaya menangani semua traffic — penting buat jalur sensitif seperti checkout OrderFlow yang salah sedikit langsung berdampak ke transaksi nyata.

### Cara Kerja
```
Rolling update (maxSurge=1, maxUnavailable=0):

  v1 v1 v1        v1 v1 v1 v2      v1 v1 v2 v2      v1 v2 v2 v2      v2 v2 v2
  (3 replika)  -> (+1 Pod baru, -> (1 Pod lama   -> (2 Pod lama   -> (rollout
                   tunggu ready)     di-terminate)    di-terminate)     selesai)
  Traffic tetap dilayani penuh sepanjang proses -- gak pernah replika < 3.


Blue-Green:

  Service "orderflow-go-svc" --> selector app=orderflow-go, version=blue
                                          |
                                          v
                                  Deployment "blue" (v1, 100% traffic)
                                  Deployment "green" (v2, running, 0% traffic
                                                       -- diverifikasi dulu
                                                       lewat internal testing)

  Setelah green terverifikasi siap:
  Service selector diganti --> version=green
                                          |
                                          v
                                  Deployment "green" (v2) terima 100% traffic
                                  Deployment "blue" (v1) tetap standby -- kalau
                                  green bermasalah, selector tinggal dibalik
                                  lagi ke "blue" (rollback instan, gak perlu
                                  rebuild/redeploy).


Canary (weighted split via Ingress):

  Ingress utama (90%) --------> Service "orderflow-go-svc"    --> Deployment
                                                                   "orderflow-go" (v1, stabil)
  Ingress canary (10%,
   annotation canary-weight) -> Service "orderflow-go-canary-svc" --> Deployment
                                                                   "orderflow-go-canary" (v2)

  10% traffic nyata "mencicipi" v2 dulu; kalau metrik (error rate, latency,
  Phase 10) sehat, weight dinaikkan bertahap (10% -> 50% -> 100%) sampai v2
  jadi versi utama dan Deployment canary dihapus/dijadikan stabil yang baru.
```

### Contoh Kode — Go
`Deployment` `orderflow-go` (topik 89) dikonfigurasi eksplisit pakai `strategy.rollingUpdate` — `maxUnavailable: 0` memastikan replika yang benar-benar siap gak pernah turun di bawah 3 selama rollout, `maxSurge: 1` mengizinkan maksimal 1 Pod ekstra sementara di atas 3 replika normal selagi Pod baru sedang di-bootstrap:
```yaml
# orderflow-go-deployment-rolling.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orderflow-go
  namespace: orderflow
  labels:
    app: orderflow-go
    track: stable
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: orderflow-go
      track: stable
  template:
    metadata:
      labels:
        app: orderflow-go
        track: stable
    spec:
      serviceAccountName: orderflow-go-sa
      containers:
        - name: orderflow-go
          image: registry.example.com/orderflow-go:1.5.0
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: orderflow-go-config
            - secretRef:
                name: orderflow-go-secret
          readinessProbe:
            httpGet:
              path: /healthz/ready
              port: 8080
            periodSeconds: 5
```
Contoh Canary untuk `orderflow-go` — Deployment kedua `orderflow-go-canary` (label `track: canary`) dengan versi baru, Service terpisah yang menyeleksi label itu, dan Ingress canary dengan annotation `nginx.ingress.kubernetes.io/canary-weight` yang mengarahkan 10% traffic dari host `go.orderflow.example.com` (host yang sama dengan `orderflow-go-ingress` di topik 89) ke Service canary ini, sisanya tetap ke `orderflow-go-svc` stabil. **Penting**: selector `orderflow-go-svc` (topik 89) harus diperbarui supaya mensyaratkan `track: stable` juga, bukan cuma `app: orderflow-go` — kalau enggak, selector itu inclusive-match dan bakal IKUT menyeleksi Pod canary (yang sama-sama berlabel `app: orderflow-go`) sebagai endpoint, sehingga Pod canary menerima traffic dari dua jalur sekaligus (10% lewat Ingress canary, ditambah sebagian traffic "stabil" lewat Service ini) dan weighted-split 90/10 yang dikira berlaku jadi gak akurat:
```yaml
# orderflow-go-svc.yaml -- update dari topik 89: selector ditambah
# `track: stable` supaya TIDAK ikut menyeleksi Pod canary di bawah.
apiVersion: v1
kind: Service
metadata:
  name: orderflow-go-svc
  namespace: orderflow
spec:
  selector:
    app: orderflow-go
    track: stable
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
---
# orderflow-go-canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orderflow-go-canary
  namespace: orderflow
  labels:
    app: orderflow-go
    track: canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: orderflow-go
      track: canary
  template:
    metadata:
      labels:
        app: orderflow-go
        track: canary
    spec:
      serviceAccountName: orderflow-go-sa
      containers:
        - name: orderflow-go
          image: registry.example.com/orderflow-go:1.6.0-rc1
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: orderflow-go-config
            - secretRef:
                name: orderflow-go-secret
---
# orderflow-go-canary-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: orderflow-go-canary-svc
  namespace: orderflow
spec:
  selector:
    app: orderflow-go
    track: canary
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
---
# orderflow-go-canary-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: orderflow-go-canary-ingress
  namespace: orderflow
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
    - host: go.orderflow.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: orderflow-go-canary-svc
                port:
                  number: 80
```

### Contoh Kode — Node.js
Manifest rolling update yang setara untuk `orderflow-node` — `maxUnavailable: 0` juga dipakai supaya kapasitas gak pernah turun di bawah 2 replika, dan `maxSurge: 1` diset eksplisit (bukan mengandalkan default `25%`) supaya konsisten dengan `orderflow-go`: maksimal 1 Pod ekstra sementara di atas 2 replika normal selagi Pod baru sedang di-bootstrap:
```yaml
# orderflow-node-deployment-rolling.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orderflow-node
  namespace: orderflow
  labels:
    app: orderflow-node
    track: stable
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: orderflow-node
      track: stable
  template:
    metadata:
      labels:
        app: orderflow-node
        track: stable
    spec:
      serviceAccountName: orderflow-node-sa
      containers:
        - name: orderflow-node
          image: registry.example.com/orderflow-node:1.5.0
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: orderflow-node-config
            - secretRef:
                name: orderflow-node-secret
          readinessProbe:
            httpGet:
              path: /healthz/ready
              port: 3000
            periodSeconds: 5
```
Canary yang setara untuk `orderflow-node` — struktur identik, target port dan host Ingress-nya mengikuti `orderflow-node-svc`/`orderflow-node-ingress` (topik 89). Sama seperti `orderflow-go`, selector `orderflow-node-svc` (topik 89) juga harus diperbarui supaya mensyaratkan `track: stable`, kalau enggak Service stabil ini bakal ikut menyeleksi Pod canary yang sama-sama berlabel `app: orderflow-node`:
```yaml
# orderflow-node-svc.yaml -- update dari topik 89: selector ditambah
# `track: stable` supaya TIDAK ikut menyeleksi Pod canary di bawah.
apiVersion: v1
kind: Service
metadata:
  name: orderflow-node-svc
  namespace: orderflow
spec:
  selector:
    app: orderflow-node
    track: stable
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
---
# orderflow-node-canary-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orderflow-node-canary
  namespace: orderflow
  labels:
    app: orderflow-node
    track: canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: orderflow-node
      track: canary
  template:
    metadata:
      labels:
        app: orderflow-node
        track: canary
    spec:
      serviceAccountName: orderflow-node-sa
      containers:
        - name: orderflow-node
          image: registry.example.com/orderflow-node:1.6.0-rc1
          ports:
            - containerPort: 3000
          envFrom:
            - configMapRef:
                name: orderflow-node-config
            - secretRef:
                name: orderflow-node-secret
---
# orderflow-node-canary-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: orderflow-node-canary-svc
  namespace: orderflow
spec:
  selector:
    app: orderflow-node
    track: canary
  ports:
    - port: 80
      targetPort: 3000
  type: ClusterIP
---
# orderflow-node-canary-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: orderflow-node-canary-ingress
  namespace: orderflow
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
    - host: node.orderflow.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: orderflow-node-canary-svc
                port:
                  number: 80
```

### Trade-off & Pitfall
- Rolling update dengan `maxUnavailable: 0` menjamin kapasitas gak pernah turun, tapi butuh resource ekstra sementara (`maxSurge`) selama rollout, dan tetap mengekspos 100% traffic ke versi baru begitu rollout selesai — kalau bug versi baru cuma muncul di beban produksi nyata, semua traffic sudah lanjut ke versi bermasalah itu sebelum ada kesempatan mundur.
- Manifest rolling update `orderflow-go`/`orderflow-node` di atas menambahkan label `track: stable` ke `spec.selector.matchLabels`, padahal Deployment aslinya (topik 89, Phase 11) cuma punya `matchLabels: {app: orderflow-go}`/`{app: orderflow-node}` — `spec.selector` pada `Deployment` bersifat IMMUTABLE begitu Deployment sudah dibuat, jadi menerapkan manifest ini dengan `kubectl apply` ke Deployment yang sudah ada bakal gagal dengan error `field is immutable`; kalau baru mau mengadopsi pola canary (`track: stable`) di atas Deployment existing, Deployment lama itu harus dihapus dan dibuat ulang (atau di-deploy dengan nama baru dulu lalu migrasi Service ke sana) — bukan sekadar `apply`. Ini gotcha nyata yang sering baru ketahuan pas migrasi ke canary di cluster yang sudah production, bukan cuma soal teori.
- Blue-Green butuh 2x kapasitas resource selama masa transisi (blue dan green sama-sama hidup penuh) — lebih mahal dibanding rolling update, dan kalau OrderFlow melakukan migrasi skema database (Phase 3) yang breaking, versi blue dan green harus tetap kompatibel dengan skema yang sama selama keduanya hidup berdampingan.
- Canary butuh dukungan weighted routing (annotation `nginx.ingress.kubernetes.io/canary-weight` di atas, atau service mesh seperti Istio) — gak semua Ingress controller/cluster mendukungnya; kalau kontrolnya salah dikonfigurasi (misal lupa annotation `canary: "true"`), traffic bisa gak ke-split sama sekali dan seluruhnya tetap ke Deployment stabil, membuat canary terlihat "aman" padahal sebenarnya gak pernah benar-benar dites.
- Selector Service stabil (`orderflow-go-svc`/`orderflow-node-svc`, topik 89) itu inclusive-match — kalau cuma `app: orderflow-go`/`app: orderflow-node` tanpa disempitkan dengan `track: stable`, Service itu bakal IKUT menyeleksi Pod canary (yang sama-sama berlabel `app: orderflow-go`/`app: orderflow-node`, ditambah `track: canary`) sebagai endpoint-nya. Akibatnya Pod canary menerima traffic dari dua jalur sekaligus (10% lewat Ingress canary + sebagian traffic "stabil" yang harusnya cuma ke versi lama), dan weighted-split 90/10 yang dikira berlaku sebenarnya gak akurat — ini gotcha umum di canary deployment yang gampang kelewat kalau cuma fokus ke Ingress-nya dan lupa ngecek selector Service.
- Deployment `orderflow-go-canary`/`orderflow-node-canary` berbagi `ConfigMap`/`Secret` yang sama dengan versi stabil (topik 91) — kalau versi canary butuh konfigurasi berbeda (misalnya feature flag baru), harus disiapkan `ConfigMap` terpisah, kalau enggak canary jalan dengan environment yang identik dengan stabil dan gak benar-benar menguji perbedaan konfigurasinya.

### Kapan Dipakai
Rolling update adalah default yang dipakai buat sebagian besar rilis rutin OrderFlow — cukup aman buat perubahan kecil-menengah dan gak butuh infrastruktur tambahan. Blue-Green dipakai buat rilis besar yang butuh kemampuan rollback instan tanpa syarat (misalnya migrasi versi major yang riskan) dan tim bersedia membayar biaya 2x resource sementara. Canary dipakai buat fitur yang risikonya tinggi kalau langsung kena semua user sekaligus — checkout OrderFlow adalah kandidat jelas, karena bug kecil di jalur itu langsung berdampak ke transaksi uang nyata, jadi lebih aman "dicicipi" 10% traffic dulu sebelum dipercaya menangani semuanya.

### Sering Ditanya Saat Interview
- "Apa beda rolling update dengan canary, padahal sama-sama gradual?" — rolling update otomatis mengganti SEMUA Pod ke versi baru sampai selesai tanpa berhenti di persentase tertentu; canary sengaja BERHENTI di persentase kecil (misalnya 10%) untuk periode observasi, dan keputusan naik ke 100% atau rollback dilakukan secara sadar berdasarkan metrik, bukan otomatis lanjut begitu saja.
- "Kenapa canary butuh dua `Deployment` terpisah, bukan cukup satu dengan dua versi image?" — satu `Deployment` cuma bisa menjalankan SATU `image:` dalam satu waktu (semua Pod-nya identik); dua Deployment dengan label berbeda (`track: canary`) memungkinkan dua versi image berjalan berdampingan, masing-masing diseleksi Service-nya sendiri sehingga traffic-nya bisa di-split secara independen.
- "Kenapa Blue-Green makin jarang dipakai murni di Kubernetes dibanding kombinasi rolling update + canary?" — Kubernetes gak punya resource native khusus buat Blue-Green (beda dengan `strategy.rollingUpdate` yang built-in di `Deployment`); Blue-Green harus disusun manual lewat dua set Deployment + switch label Service, dan biaya 2x resource selama transisi seringkali gak sepadan dibanding rolling update (murah, built-in) dikombinasikan dengan canary (blast radius kecil) buat kasus yang butuh kehati-hatian ekstra.

---

**Selanjutnya:** [Phase 13 — Performance](./phase-13-performance.md)
