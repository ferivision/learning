# Phase 15 — Interview Thinking

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

Phase ini beda dari 14 phase sebelumnya: gak ada topik baru, gak ada kode. Isinya cuma enam "kerangka berpikir" buat menjawab pertanyaan interview backend yang bentuknya open-ended — tipe pertanyaan yang gak punya satu jawaban benar, tapi interviewer sebenarnya lagi menilai apakah kamu punya *urutan berpikir* yang sistematis, bukan menghafal daftar solusi. Semua worked example di bawah pakai konteks OrderFlow yang sama dengan phase 1–14, dan tiap kerangka secara eksplisit merujuk balik ke topik-topik yang sudah dibahas.

---

## Bagaimana Mencegah X?

### Kerangka Berpikir
Kalau interviewer nanya "gimana caramu mencegah [sesuatu yang gak boleh terjadi]?", jawaban yang cuma nyebut satu mekanisme (misalnya "pakai auth doang") kedengaran dangkal. Kerangka yang lebih meyakinkan adalah jalanin satu urutan lapisan pertahanan, dari yang paling dekat ke identitas sampai yang paling jauh (observability), dan jelasin kenapa satu lapisan aja gak cukup:

1. **Authentication** — pastikan dulu siapa yang ngirim request itu, sebelum mikirin apa pun yang lain.
2. **Authorization** — begitu tau identitasnya, cek apakah identitas itu MEMANG berhak melakukan aksi spesifik ini terhadap resource spesifik ini (bukan cuma "sudah login").
3. **Network** — batasi secara infrastruktur siapa yang bahkan bisa nyampe ke endpoint ini (misalnya endpoint admin cuma bisa diakses dari jaringan internal, bukan cuma dijaga di level aplikasi).
4. **Validation** — pastikan input yang masuk gak bisa dipakai buat memanipulasi logic (misalnya field `status` yang dikirim user gak boleh langsung dipercaya buat override alur bisnis).
5. **Encryption** — data sensitif yang terlibat (token, data pembayaran) gak boleh bisa dibaca kalau bocor di tengah jalan atau di storage.
6. **Rate Limiting** — batasi seberapa sering satu identitas/IP bisa mencoba aksi ini, biar percobaan brute-force atau abuse otomatis gak efektif.
7. **Monitoring** — kalau lima lapisan di atas somehow tetap kebobol atau ada percobaan mencurigakan, harus ada yang notice dan bisa diinvestigasi setelahnya.

Urutannya penting: authentication dan authorization itu pencegahan preventif di jalur request, network itu pencegahan di luar jalur aplikasi, validation & encryption itu pencegahan terhadap manipulasi data, rate limiting itu pencegahan terhadap abuse berulang, dan monitoring itu jaring pengaman terakhir kalau semua yang di atas gagal.

### Contoh Kasus Nyata
Worked example: mencegah pembatalan order yang gak sah di OrderFlow (`POST /orders/{id}/cancel`). Skenarionya, seorang customer iseng nyoba ganti ID di request cancel dari order miliknya sendiri ke ID order milik customer lain, atau nyoba ngehit endpoint itu berkali-kali buat lihat apakah ada celah race condition di status order.

Jalanin kerangkanya: (1) authentication — request harus bawa token valid; (2) authorization — order dengan ID itu harus benar-benar milik user yang login, DAN status order-nya masih di state yang boleh dibatalkan (misalnya gak bisa cancel order yang statusnya sudah "shipped"); (3) network — endpoint cancel gak butuh pembatasan network khusus karena memang dipakai customer biasa (beda dengan endpoint admin), tapi endpoint admin yang bisa cancel order siapa pun sebaiknya cuma bisa diakses dari jaringan internal; (4) validation — pastikan `order_id` di path valid dan bukan hasil injeksi; (5) encryption — gak terlalu relevan di aksi cancel spesifik ini (lebih relevan di data pembayaran), tapi tetap disebut biar kerangkanya lengkap; (6) rate limiting — batasi percobaan cancel berulang dalam waktu singkat dari satu user/IP; (7) monitoring — log setiap percobaan cancel yang ditolak karena authorization gagal, biar pola percobaan berulang ke banyak order ID kelihatan sebagai sinyal serangan enumeration.

### Bagaimana Menjawabnya Saat Interview
Jangan langsung lompat ke detail teknis satu lapisan (misalnya langsung ngoceh soal JWT signature). Buka dengan bilang kamu mau jalanin beberapa lapisan pertahanan dari yang paling dekat ke request sampai yang paling jauh, lalu sebutin urutannya satu-satu sambil ngasih alasan kenapa lapisan itu perlu — itu nunjukin kamu ngerti "defense in depth", bukan cuma nyebut satu jurus. Kalau interviewer nanya "yang paling penting yang mana?", jawaban jujurnya biasanya authorization (topik yang paling sering dilewatkan padahal authentication-nya sudah benar), tapi tetap tekankan bahwa satu lapisan aja gak cukup — itu justru poin utama dari kerangka ini. Kalau ditanya lebih detail di satu lapisan, baru masuk ke level topik spesifik (misalnya IDOR check di authorization).

### Referensi Konsep Terkait
- Phase 1, topik 1 (Authentication vs Authorization) — dasar kenapa dua hal ini harus dipisahkan sebagai dua lapisan yang beda.
- Phase 1, topik 8 (RBAC) dan topik 9 (Least Privilege) — dasar buat authorization berbasis role dan prinsip "cuma kasih akses yang benar-benar dibutuhkan".
- Phase 2, topik 13 (API Authentication) dan topik 14 (API Authorization/IDOR-BOLA) — implementasi konkret dua lapisan pertama di level API OrderFlow.
- Phase 2, topik 15 (Rate Limiting) dan topik 24 (Encryption Fundamentals) — lapisan rate limiting dan encryption.
- Phase 7, topik 73 (API Gateway) — salah satu titik yang lazim buat menegakkan pembatasan network/routing sebelum request nyampe ke service.
- Phase 11, topik 91 (Kubernetes Security) — NetworkPolicy sebagai mekanisme pembatasan network di level infrastruktur cluster.
- Phase 10, topik 87 (Monitoring) — lapisan terakhir buat mendeteksi percobaan yang lolos atau mencurigakan.

---

## Bagaimana Scale X?

### Kerangka Berpikir
Pertanyaan "gimana scale X?" itu jebakan buat orang yang langsung jawab "tinggal tambah server" tanpa mikir dulu apa yang sebenarnya jadi batasan (bottleneck)-nya. Kerangka yang benar dimulai dari mengukur, bukan dari menambah resource:

1. **Identify bottleneck** — sebelum solve apa pun, tentuin dulu komponen mana yang sebenarnya jadi penghambat: CPU aplikasi? Koneksi database? Network I/O? Kalau salah identifikasi, solusi apa pun (termasuk nambah server) gak akan ngefek.
2. **Measure** — jangan nebak, ukur pakai data nyata (metrics, load test) biar keputusan berikutnya berbasis angka, bukan asumsi.
3. **Optimize** — sebelum nambah resource, coba dulu bikin kerjaan yang ada lebih efisien (query yang lebih cepat, hilangkan kerjaan yang gak perlu) — sering kali lebih murah dan lebih cepat daripada scaling infrastruktur.
4. **Cache** — kalau bottleneck-nya baca data yang sama berulang-ulang, taruh di layer yang lebih cepat diakses supaya gak selalu ngehit sumber yang lambat.
5. **Horizontal scaling** — kalau satu instance/proses udah dioptimasi habis dan masih gak cukup, tambah instance yang jalan paralel di belakang load balancer.
6. **Async processing** — kerjaan yang gak perlu langsung selesai sebelum response dikirim (misalnya kirim notifikasi, generate laporan) dipindah ke background lewat queue, biar request path utama tetap cepat.
7. **Database scaling** — kalau setelah semua di atas bottleneck-nya tetap di database, baru masuk ke opsi yang lebih berat: read replica buat baca, atau sharding buat nulis dalam skala sangat besar.

Urutan ini sengaja naik dari yang paling murah/cepat diterapkan (measure, optimize) ke yang paling mahal/kompleks (database scaling) — jangan lompat ke solusi paling mahal duluan.

### Contoh Kasus Nyata
Worked example: endpoint `GET /products` OrderFlow yang biasanya santai, tiba-tiba dihajar traffic berkali-kali lipat pas Black Friday, sampai response time melonjak dan sebagian request timeout.

Jalanin kerangkanya: (1) identify bottleneck — cek dulu apakah masalahnya CPU aplikasi kehabisan (banyak proses paralel), koneksi ke Postgres yang habis dari connection pool, atau memang query produknya sendiri yang lambat; (2) measure — lihat metrics p99 latency dan error rate endpoint ini (RED method), plus load test untuk simulasi traffic Black Friday sebelum hari-H; (3) optimize — kalau ternyata query produk kena N+1 atau kurang index yang pas, benerin itu duluan — kadang cukup ini aja response time udah turun drastis tanpa nambah infrastruktur; (4) cache — data produk gak sering berubah dalam hitungan detik, jadi cocok banget di-cache pakai pola cache-aside di Redis, drastis mengurangi beban ke Postgres; (5) horizontal scaling — tambah replica instance aplikasi OrderFlow di belakang load balancer buat menyerap lebih banyak concurrent request; (6) async processing — kalau ada side-effect di endpoint ini yang sebenarnya gak perlu sinkron (misalnya update analytics view count), pindahkan ke message queue; (7) database scaling — kalau setelah caching pun baca produk masih membebani Postgres primary, tambahkan read replica khusus buat query baca produk.

### Bagaimana Menjawabnya Saat Interview
Interviewer sering sengaja gak kasih detail lengkap soal di mana bottleneck-nya — itu justru bagian dari tes: apakah kamu langsung asumsi "pasti database" atau "pasti perlu Kubernetes autoscaling" tanpa nanya balik. Jawaban yang bagus dimulai dengan nanya balik atau eksplisit bilang "hal pertama yang aku lakuin adalah cari tau dulu bottleneck-nya di mana, karena solusinya beda tergantung itu CPU, database, atau network." Baru setelah itu jalanin kerangkanya secara berurutan, dan tegaskan bahwa optimize & cache biasanya dicoba dulu sebelum lompat ke horizontal scaling atau database scaling, karena itu solusi yang lebih murah dan lebih cepat diterapkan.

### Referensi Konsep Terkait
- Phase 10, topik 84 (Metrics) dan topik 86 (RED Method) — cara mengukur bottleneck berbasis data, bukan tebakan.
- Phase 13, topik 94 (API Performance), topik 95 (Database Performance), dan topik 97 (Load Testing) — praktik konkret identifikasi dan pengukuran bottleneck.
- Phase 3, topik 26 (Indexing), topik 28 (EXPLAIN ANALYZE), dan topik 35 (N+1 Query) — langkah optimize di level database sebelum scaling.
- Phase 4, topik 42 (Cache-Aside) dan topik 45 (Cache Stampede) — lapisan cache dan risikonya kalau cache-nya sendiri gak dirancang hati-hati saat traffic tinggi.
- Phase 7, topik 65 (Vertical vs Horizontal Scaling), topik 66 (Stateless Service), dan topik 67 (Load Balancer) — dasar horizontal scaling.
- Phase 5, topik 47 (Why Message Queue) — dasar memindahkan kerjaan ke async processing.
- Phase 3, topik 37 (Read Replica) dan topik 39 (Sharding) — opsi database scaling.

---

## Apa yang Terjadi Kalau X Gagal?

### Kerangka Berpikir
Pertanyaan tipe ini nguji apakah kamu mikirin failure mode dari awal desain, bukan cuma happy path. Kerangkanya adalah rantai respons berlapis, dari yang paling cepat merespons kegagalan sesaat sampai yang paling lambat tapi paling menyeluruh:

1. **Timeout** — jangan biarin satu request nunggu dependency yang lambat/hang tanpa batas; tentuin batas waktu wajar, biar resource gak ketahan selamanya.
2. **Retry** — kalau kegagalannya transient (timeout sesaat, 503), coba lagi dengan backoff, karena percobaan berikutnya sering kali berhasil.
3. **Circuit Breaker** — kalau kegagalannya berulang terus-menerus (bukan sesaat), berhenti nyoba dulu buat sementara waktu, biar gak terus-terusan membebani dependency yang lagi down dan biar request gagal cepat (fail fast) daripada nunggu timeout berkali-kali.
4. **Fallback** — begitu tau dependency itu down, pastikan HANYA fitur yang bergantung ke dependency itu yang terganggu (graceful degradation), fitur lain di sistem tetap jalan normal.
5. **Queue** — kalau operasinya bisa ditunda (gak butuh respons sinkron), taruh dulu di antrean, biar bisa diproses begitu dependency-nya pulih, daripada langsung gagal permanen.
6. **Idempotency** — begitu retry dan queue masuk ke skenario, pastikan operasi yang sama gak diproses dua kali efeknya kalau kebetulan diulang.
7. **Monitoring** — pantau semua kejadian di atas (berapa kali circuit breaker trip, berapa banyak yang masuk fallback) supaya ada visibilitas ke kondisi sistem.
8. **Alerting** — kalau kondisinya cukup parah/berkepanjangan, orang yang bertanggung jawab (on-call) harus diberitahu aktif, bukan cuma nunggu ada yang notice dashboard.

### Contoh Kasus Nyata
Worked example: payment provider eksternal OrderFlow tiba-tiba down di tengah jam sibuk checkout.

Jalanin kerangkanya: (1) timeout — panggilan ke payment provider dikasih batas waktu wajar, jangan sampai satu request checkout ketahan menit-menitan; (2) retry — kalau gagalnya karena timeout sesaat/503, coba ulang dengan exponential backoff, karena payment provider yang sekadar lambat sesaat sering pulih di percobaan kedua/ketiga; (3) circuit breaker — kalau kegagalan terus berulang (bukan cuma sesaat), circuit breaker trip ke status OPEN, biar request-request checkout berikutnya gagal cepat tanpa nunggu timeout satu-satu dan biar payment provider gak makin dibebani retry masif; (4) fallback — begitu circuit breaker OPEN, checkout dikasih pesan jelas "coba lagi nanti", TAPI browsing produk dan isi keranjang tetap jalan normal karena dua fitur itu gak menyentuh payment provider sama sekali; (5) queue — kalau OrderFlow punya alur pembayaran yang bisa async (misalnya generate invoice/notifikasi setelah bayar), bagian itu bisa antre dulu; (6) idempotency — kalau user retry checkout manual setelah gagal, `Idempotency-Key` yang sama harus mencegah dua transaksi charge terpisah untuk satu pembelian; (7) monitoring — pantau berapa lama circuit breaker OPEN dan berapa banyak checkout yang kena fallback; (8) alerting — kalau circuit breaker OPEN lebih dari beberapa menit di jam sibuk, halangi/page tim on-call, karena ini langsung berdampak ke revenue.

### Bagaimana Menjawabnya Saat Interview
Kerangka ini paling meyakinkan kalau dijelaskan sebagai "garis waktu kegagalan" — mulai dari detik-detik pertama (timeout, retry) sampai kondisi berkepanjangan (circuit breaker, fallback, alerting). Tekankan bahwa retry dan circuit breaker itu saling melengkapi, bukan pilih salah satu: retry menangani kegagalan sesaat, circuit breaker menangani kegagalan yang berkepanjangan supaya sistem gak terus-menerus buang waktu retry ke dependency yang jelas-jelas lagi down. Sebutkan juga eksplisit kenapa idempotency wajib disebut di sini: begitu ada retry (baik otomatis maupun manual dari user), ada risiko operasi ke-proses dua kali, jadi idempotency itu bukan topik terpisah, tapi konsekuensi langsung dari adanya retry.

### Referensi Konsep Terkait
- Phase 2, topik 18 (Timeout) dan topik 17 (Retry) — dua lapisan pertama dalam menangani kegagalan transient.
- Phase 9, topik 79 (Circuit Breaker) dan topik 81 (Graceful Degradation) — mekanisme fail-fast dan fallback per-fitur, keduanya dibahas dengan skenario payment provider yang sama persis.
- Phase 9, topik 78 (Health Checks) — kaitannya dengan readiness dan kenapa satu dependency non-esensial down gak boleh mematikan seluruh service.
- Phase 5, topik 47 (Why Message Queue) dan topik 52 (Dead Letter Queue) — opsi menunda pemrosesan lewat antrean, dan apa yang terjadi kalau tetap gagal setelah semua retry habis.
- Phase 2, topik 16 (Idempotency) dan Phase 14, topik 101 (Distributed Idempotency) — mencegah efek ganda begitu retry masuk ke gambaran.
- Phase 10, topik 87 (Monitoring) — termasuk penjelasan eksplisit soal alerting berbasis threshold metrics.

---

## Bagaimana Improve API yang Lambat?

### Kerangka Berpikir
Ini kerangka investigasi, bukan kerangka desain — bedanya dengan "Bagaimana Scale X?" adalah di sini kamu debug satu endpoint yang udah lambat di production, bukan mengantisipasi traffic yang belum terjadi. Urutannya dari observability yang paling murah diakses ke yang paling detail:

1. **Logs** — mulai dari sini karena paling gampang dan cepat: lihat request spesifik yang lambat, apakah ada error/warning yang nyangkut.
2. **Metrics** — lihat gambaran agregat: apakah lambatnya cuma request tertentu (outlier) atau memang p99 latency endpoint ini secara umum udah naik.
3. **Tracing** — begitu tau endpoint-nya lambat secara umum, tracing nunjukin DI BAGIAN MANA dari request itu waktu paling banyak kehabisan (query database? panggilan ke service lain? serialization?).
4. **Find bottleneck** — dari tracing, sempitin ke satu-dua titik spesifik yang paling makan waktu.
5. **DB/Cache/Network/CPU** — setelah nemu titiknya, klasifikasikan itu masalah di lapisan mana: query database yang gak optimal, cache miss yang seharusnya hit, panggilan network ke service lain yang lambat, atau CPU-bound (misalnya serialization JSON yang berat).
6. **Optimize** — baru di titik ini masuk ke solusi konkret sesuai lapisan yang teridentifikasi (tambah index, benerin N+1, cache hasil query, dsb) — solusinya SANGAT tergantung hasil langkah sebelumnya, jangan optimasi sebelum tau apa yang sebenarnya lambat.

### Contoh Kasus Nyata
Worked example: `GET /orders/{id}` di OrderFlow yang biasanya cepat, tapi belakangan user komplain kadang lemot beberapa detik.

Jalanin kerangkanya: (1) logs — cek log request yang lambat itu, apakah ada retry ke service lain atau warning yang muncul; (2) metrics — lihat dashboard p99 latency endpoint `GET /orders/{id}` seminggu terakhir, ternyata memang naik bertahap, bukan cuma outlier sesekali; (3) tracing — trace salah satu request yang lambat, kelihatan sebagian besar waktu abis di satu span: query buat ambil order items; (4) find bottleneck — span itu ternyata query yang ambil item-item order plus detail produk masing-masing, dipanggil satu-satu per item (bukan sekali query buat semua item); (5) klasifikasi — ini masalah di lapisan database, spesifiknya pola N+1 query; (6) optimize — ganti jadi satu query pakai `JOIN` atau `WHERE id IN (...)`, dan sambil di situ pastikan kolom yang dipakai buat filter/join memang punya index yang pas.

### Bagaimana Menjawabnya Saat Interview
Poin yang paling penting buat ditekankan: kamu gak langsung nebak "pasti kurang index" atau "pasti perlu cache" begitu denger "API lambat" — kamu jalanin investigasi berlapis dari observability yang murah (logs, metrics) ke yang detail (tracing) SEBELUM menyentuh solusi. Banyak kandidat lompat langsung ke solusi (nambah index, nambah cache) tanpa nunjukin cara mereka nemuin masalahnya duluan — itu yang bikin jawaban kedengaran hafalan, bukan proses berpikir. Kalau interviewer kasih detail tambahan di tengah jalan (misalnya "ternyata query-nya udah ada index"), tunjukin kamu bisa nyesuain arah investigasi (lanjut cek cache atau network) daripada ngotot ke satu hipotesis.

### Referensi Konsep Terkait
- Phase 10, topik 83 (Logs), topik 84 (Metrics), dan topik 85 (Tracing) — tiga pilar observability yang jadi tiga langkah pertama kerangka ini persis sesuai urutan topiknya.
- Phase 10, topik 86 (RED Method) — cara membaca metrics agregat (rate, error, duration) buat tau endpoint mana yang bermasalah.
- Phase 3, topik 35 (N+1 Query), topik 26 (Indexing), dan topik 28 (EXPLAIN ANALYZE) — alat dan pola paling umum buat kasus bottleneck di lapisan database.
- Phase 4, topik 42 (Cache-Aside) — opsi optimize kalau bottleneck-nya ternyata data yang sering dibaca ulang tapi gak di-cache.
- Phase 13, topik 94 (API Performance) dan topik 95 (Database Performance) — praktik lanjutan buat improve performa API dan database secara lebih menyeluruh.

---

## Bagaimana Cegah Duplicate Processing?

### Kerangka Berpikir
Duplicate processing biasanya bukan karena satu penyebab tunggal — ada beberapa titik berbeda di mana duplikasi bisa masuk, jadi kerangkanya adalah beberapa lapisan pertahanan yang saling menguatkan, bukan satu mekanisme tunggal:

1. **Idempotency Key** — client (atau consumer) kasih identifier unik ke satu "niat" operasi, dipakai ulang persis kalau operasi yang sama di-retry, biar server bisa tau ini request yang sama, bukan request baru.
2. **Unique Constraint** — di level database, pastikan ada constraint yang secara fisik mencegah dua baris dengan key yang sama tersimpan, sebagai pengaman terakhir kalau ada race condition di antara pengecekan aplikasi.
3. **Processed Event** — khusus buat event/message dari queue atau webhook, simpan catatan event ID yang sudah diproses, biar kalau event yang sama datang lagi (karena at-least-once delivery), consumer tau buat skip, bukan proses ulang.
4. **Database Transaction** — bungkus pengecekan dan penyimpanan hasil dalam satu transaction, biar gak ada jendela waktu antara "cek sudah pernah diproses belum" dan "simpan hasilnya" yang bisa diselip proses lain.

Empat lapisan ini menjawab skenario berbeda: idempotency key untuk request yang di-retry sengaja oleh client, unique constraint untuk race condition di level database, processed event untuk delivery ganda dari message broker, dan transaction untuk memastikan pengecekan-dan-penyimpanan itu atomik.

### Contoh Kasus Nyata
Worked example ada dua skenario yang sering ditanya bareng karena akar masalahnya sama (duplicate processing) tapi sumbernya beda:

**Skenario 1 — user double-click "Bayar":** user klik tombol bayar, koneksi lemot, user (gak sabar) klik lagi sebelum response pertama balik. Dua request `POST /payments` terkirim buat satu niat pembayaran yang sama. Kalau gak dicegah, dua-duanya bisa diproses jadi dua transaksi charge terpisah. Frontend generate satu `Idempotency-Key` (UUID) begitu tombol pertama kali diklik, dan dua request itu (kalau memang double-click) bakal bawa key yang sama — server tinggal tolak/balikin hasil yang sama untuk key yang udah pernah diproses, dijaga sebagai lapisan terakhir oleh unique constraint di kolom `idempotency_key`.

**Skenario 2 — webhook payment provider firing twice:** payment provider ngirim webhook notifikasi "payment success", tapi karena provider itu sendiri gak yakin webhook pertamanya sukses diterima (network putus di tengah, gak dapat 2xx tepat waktu), dia ngirim ulang webhook yang sama. OrderFlow (sebagai consumer/receiver) harus nyimpen event ID webhook yang udah diproses sebelumnya, biar webhook kedua yang notifikasi kejadian sama gak memicu OrderFlow update status order dan kirim notifikasi ke user dua kali.

### Bagaimana Menjawabnya Saat Interview
Hal pertama yang bikin jawaban kedengaran kuat: eksplisit bedakan dua sumber duplikasi ini — client yang retry (butuh idempotency key yang di-generate CLIENT) vs sistem eksternal yang re-deliver (butuh processed event tracking di CONSUMER) — karena banyak kandidat cuma nyebut satu dan mikir itu udah cukup buat semua kasus. Lalu tekankan bahwa unique constraint di database itu bukan opsional/nice-to-have, tapi pengaman WAJIB — pengecekan di level aplikasi (SELECT dulu baru INSERT) selalu punya celah race condition kalau dua request masuk nyaris bersamaan, dan cuma constraint di level database yang benar-benar atomik menutup celah itu. Kalau ditanya "kenapa gak cukup cuma cek di aplikasi aja", itu jawabannya persis di titik ini.

### Referensi Konsep Terkait
- Phase 2, topik 16 (Idempotency) — mekanisme `Idempotency-Key` di level API, dibahas persis dengan skenario double-click tombol bayar.
- Phase 5, topik 49 (At-Least-Once Delivery) dan topik 50 (Idempotent Consumer) — kenapa message/event bisa terkirim lebih dari sekali, dan pola `processed_event` buat menanganinya di sisi consumer.
- Phase 14, topik 101 (Distributed Idempotency) dan topik 102 (Webhooks) — kombinasi unique constraint (`ON CONFLICT DO NOTHING`) sebagai pengaman terakhir terhadap race condition, dan skenario webhook payment provider yang re-deliver notifikasi yang persis sama.
- Phase 3, topik 29 (Transactions) dan topik 30 (ACID) — dasar kenapa pengecekan-dan-penyimpanan harus dibungkus atomik dalam satu transaction.

---

## Bagaimana Cegah Unauthorized Access?

### Kerangka Berpikir
Ini kerangka yang lebih sempit dari "Bagaimana Mencegah X?" — fokusnya spesifik ke akses yang gak sah ke DATA/RESOURCE, bukan pencegahan umum terhadap segala jenis serangan:

1. **Authentication** — pastikan identitas requester valid dulu, langkah paling dasar.
2. **Authorization** — begitu identitas jelas, cek apakah identitas itu berhak mengakses OBJECT SPESIFIK yang diminta (bukan cuma "sudah login" secara umum) — ini titik yang paling sering dilewatkan.
3. **Least Privilege** — bahkan identitas yang authorized pun idealnya cuma dikasih akses seminim yang dia butuhkan buat kerjaannya, bukan akses penuh "just in case".
4. **Network Restriction** — batasi secara infrastruktur siapa yang bisa bahkan nyampe ke endpoint/resource itu, sebagai lapisan tambahan di luar kontrol aplikasi.
5. **Monitoring** — pantau dan log percobaan akses yang ditolak, biar pola percobaan yang mencurigakan (misalnya satu user nyoba banyak ID berurutan) kelihatan dan bisa direspons.

### Contoh Kasus Nyata
Worked example: mencegah IDOR — seorang customer OrderFlow yang login, lalu iseng ganti angka ID di URL `GET /orders/{id}` dari order miliknya sendiri jadi ID order customer lain, buat lihat apakah dia bisa baca detail order (termasuk alamat pengiriman dan info pembayaran) yang bukan miliknya.

Jalanin kerangkanya: (1) authentication — endpoint ini sudah dijaga middleware auth, jadi customer itu memang berhasil lolos tahap "siapa kamu"; (2) authorization — di sinilah bug-nya kalau IDOR terjadi: handler cuma cek token valid, tapi lupa cek `order.user_id == claims.user_id`, jadi order siapa pun langsung dibalikin begitu ID-nya valid; fix-nya adalah nambahin object-level authorization check itu, dan balikin 404 (bukan 403) buat order yang bukan miliknya, biar attacker gak bisa mastiin ID itu valid tapi cuma bukan miliknya; (3) least privilege — kalaupun ada role admin di OrderFlow yang memang boleh baca semua order, role customer biasa gak boleh dikasih izin lebih dari "baca order miliknya sendiri"; (4) network restriction — kurang relevan buat endpoint customer yang memang publik-facing, tapi kalau ada endpoint internal admin yang bisa baca semua order tanpa filter, endpoint itu sebaiknya cuma bisa diakses dari jaringan internal; (5) monitoring — log setiap percobaan akses order yang ditolak karena bukan pemiliknya, biar satu user yang nyoba banyak ID berurutan (pola enumeration) kelihatan sebagai sinyal serangan, bukan cuma error 404 yang diam-diam ke-drop.

### Bagaimana Menjawabnya Saat Interview
Poin yang paling penting buat ditekankan di awal: authentication dan authorization itu DUA hal yang beda, dan bug unauthorized access yang paling umum (IDOR/BOLA) itu terjadi justru ketika authentication-nya sudah benar tapi authorization object-level-nya yang dilewatkan — endpoint kelihatan "aman" karena ada middleware auth, padahal middleware itu cuma jawab "siapa kamu", bukan "apakah resource ini punyamu". Kalau interviewer tanya detail lanjutan soal 403 vs 404, jelaskan trade-off information disclosure-nya: 403 mengonfirmasi resource itu ada (memudahkan ID enumeration), 404 lebih aman karena gak membedakan "gak ada" dari "ada tapi bukan milikmu".

### Referensi Konsep Terkait
- Phase 1, topik 1 (Authentication vs Authorization) — dasar kenapa dua konsep ini harus dipisahkan.
- Phase 1, topik 8 (RBAC) dan topik 9 (Least Privilege) — dasar pembatasan akses berbasis role dan prinsip akses seminim mungkin.
- Phase 2, topik 13 (API Authentication) dan topik 14 (API Authorization/IDOR-BOLA) — implementasi konkret authentication dan object-level authorization di OrderFlow, termasuk contoh bug dan fix persis dengan skenario `GET /orders/{id}` di atas.
- Phase 3, topik 40 (Database Security) — lapisan least privilege di level akses database, bukan cuma di level API.
- Phase 11, topik 91 (Kubernetes Security) — NetworkPolicy sebagai mekanisme network restriction di level infrastruktur.
- Phase 10, topik 87 (Monitoring) — lapisan terakhir buat mendeteksi pola percobaan akses yang mencurigakan.
