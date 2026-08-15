# Phase 06 — Go Concurrency, Testing & Backend Performance

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

## 55. Goroutine

### Apa itu?
Goroutine adalah unit eksekusi yang dikelola langsung oleh Go runtime, dijalankan cukup dengan awalan `go` di depan pemanggilan fungsi. Beda dari OS thread yang berat (butuh alokasi stack tetap beberapa MB dan context switch lewat kernel), goroutine punya stack awal cuma ~2KB yang bisa tumbuh/menyusut otomatis, dan dijadwalkan oleh Go runtime sendiri (bukan OS) lewat model M:N — ribuan goroutine dipetakan ke segelintir OS thread (`GOMAXPROCS`). Di OrderFlow, goroutine adalah fondasi buat memproses banyak order secara konkuren tanpa harus bikin satu OS thread per order.

### Kenapa dibutuhkan?
Kalau `ConsumeOrderEvents` (phase 05) memproses event `OrderCreated` satu-satu secara sekuensial di satu goroutine, throughput consumer terbatas oleh kecepatan handler-nya — event ke-100 harus nunggu 99 event sebelumnya selesai diproses dulu. Goroutine memungkinkan banyak order diproses "bersamaan" (concurrently), memanfaatkan waktu tunggu I/O (query database, panggil service lain) yang kalau dijalankan sekuensial cuma buang-buang waktu CPU nganggur.

### Cara Kerja
```
Model M:N goroutine (disederhanakan):

  Goroutine:  G1  G2  G3  G4  G5  G6  G7  G8  ... (ribuan, murah dibikin)
                \  |  |  |  /  |  |  /
                 \ |  |  |/    |  |/
  OS Thread:      M1        M2        (GOMAXPROCS, biasanya = jumlah CPU core)
                    \        /
  OS/Kernel:         CPU core(s)

Go runtime scheduler memindah-mindahkan goroutine antar OS thread (M1, M2)
secara otomatis -- termasuk saat goroutine itu blocking di I/O, digantikan
goroutine lain di thread yang sama supaya CPU gak nganggur.
```

### Contoh Kode — Go
```go
package worker

import (
	"fmt"
	"sync"
)

// Order dan Result dipakai bersama di seluruh phase ini (lihat juga topik 60).
type Order struct {
	ID     int64
	UserID int64
	Total  float64
}

type Result struct {
	OrderID int64
	Err     error
}

// processOrder mensimulasikan pemrosesan satu order (misalnya validasi
// akhir dan pembaruan status) -- dipisah jadi fungsi sendiri supaya bisa
// diuji terisolasi lewat table-driven test (topik 64).
func processOrder(order Order) Result {
	if order.Total <= 0 {
		return Result{OrderID: order.ID, Err: fmt.Errorf("order %d: total must be positive, got %.2f", order.ID, order.Total)}
	}
	return Result{OrderID: order.ID}
}

// ProcessOrdersConcurrently memproses setiap order di goroutine-nya
// sendiri-sendiri -- versi paling sederhana sebelum dibatasi jadi worker
// pool (topik 60). Cocok untuk jumlah order kecil, TAPI resikonya dibahas
// di trade-off: kalau jumlah order-nya besar, ini bisa membuat goroutine
// tak terbatas.
func ProcessOrdersConcurrently(orders []Order) []Result {
	results := make([]Result, len(orders))
	var wg sync.WaitGroup

	for i, order := range orders {
		wg.Add(1)
		go func(idx int, o Order) {
			defer wg.Done()
			results[idx] = processOrder(o)
		}(i, order)
	}

	wg.Wait()
	return results
}
```

### Contoh Kode — Node.js
Node.js gak punya goroutine karena modelnya beda secara fundamental: satu **event loop** single-threaded menjalankan semua JavaScript, dan "konkurensi" untuk operasi I/O (query database, HTTP call) dicapai lewat callback/Promise non-blocking, BUKAN lewat banyak alur eksekusi paralel seperti goroutine. Kalau butuh eksekusi paralel sungguhan (misalnya kerja CPU-bound berat), Node.js punya `worker_threads` — tapi ini jauh lebih berat dibanding goroutine (tiap Worker adalah instance V8 terpisah dengan memory overhead signifikan, gak bisa dibikin ribuan sekaligus seperti goroutine).
```javascript
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');

// processOrdersConcurrently pakai worker_threads untuk kerja CPU-bound berat
// (bukan I/O biasa -- untuk I/O, async/await di event loop utama sudah
// cukup dan lebih murah, lihat catatan di trade-off).
function processOrdersConcurrently(orders) {
  return Promise.all(orders.map((order) => runInWorker(order)));
}

function runInWorker(order) {
  return new Promise((resolve, reject) => {
    const worker = new Worker(__filename, { workerData: order });
    worker.once('message', resolve);
    worker.once('error', reject);
  });
}

if (!isMainThread) {
  const order = workerData;
  const result =
    order.total <= 0
      ? { orderId: order.id, err: `order ${order.id}: total must be positive, got ${order.total}` }
      : { orderId: order.id, err: null };
  parentPort.postMessage(result);
}

module.exports = { processOrdersConcurrently };
```

### Trade-off & Pitfall
- Goroutine murah dibikin (~2KB stack awal), tapi TIDAK gratis — membuat satu goroutine per order tanpa batas (seperti `ProcessOrdersConcurrently` di atas) bisa menyebabkan jutaan goroutine hidup bersamaan kalau volume order tiba-tiba melonjak, membebani memory dan scheduler. Ini kenapa topik 60 (Worker Pool) membatasi jumlahnya.
- Goroutine yang lupa di-`wg.Wait()`-kan atau lupa punya jalan keluar (misalnya nunggu channel yang gak pernah dikirimi apa-apa) jadi goroutine leak — goroutine itu tetap hidup selamanya, gak pernah di-garbage-collect selama masih ada referensi/blocking-nya.
- `worker_threads` di Node.js jauh lebih berat daripada goroutine (startup time dalam hitungan milidetik, overhead memory per instance V8) — dipakai cuma kalau memang butuh paralelisme CPU sungguhan, bukan sebagai pengganti umum untuk semua "concurrency" seperti goroutine di Go.

### Kapan Dipakai
Setiap kali ada pekerjaan independen yang bisa dijalankan konkuren tanpa saling menunggu — seperti memproses banyak order sekaligus. Untuk volume tinggi, goroutine sebaiknya dibatasi lewat worker pool (topik 60), bukan dibuat tanpa batas per item pekerjaan.

### Sering Ditanya Saat Interview
- "Apa beda goroutine dengan OS thread?" — goroutine dikelola Go runtime sendiri (bukan kernel), stack awalnya kecil dan bisa tumbuh dinamis, dan ribuan goroutine bisa dipetakan ke segelintir OS thread lewat model M:N scheduler; OS thread jauh lebih berat (stack tetap beberapa MB, context switch lewat kernel).
- "Kenapa Node.js gak punya goroutine?" — karena modelnya beda secara fundamental: Node.js menjalankan JavaScript di satu event loop single-threaded, konkurensi I/O dicapai lewat non-blocking callback/Promise, bukan lewat banyak alur eksekusi yang dijadwalkan runtime seperti goroutine; paralelisme sungguhan baru didapat lewat `worker_threads` yang jauh lebih berat.
- "Apa resiko bikin satu goroutine per item pekerjaan tanpa batas?" — kalau volume pekerjaannya tiba-tiba melonjak, jumlah goroutine yang hidup bersamaan ikut melonjak tak terbatas, membebani memory dan scheduler — makanya butuh worker pool (topik 60) yang membatasi jumlah goroutine aktif.

---

## 56. Channel

### Apa itu?
Channel adalah conduit typed di Go buat mengirim dan menerima nilai antar goroutine, sekaligus alat sinkronisasi. Filosofinya: "Do not communicate by sharing memory; share memory by communicating" — daripada beberapa goroutine mengakses variable yang sama (butuh mutex, topik 57), mereka saling kirim nilai lewat channel. Ada dua jenis: unbuffered (`make(chan T)`, pengirim blocking sampai ada penerima siap) dan buffered (`make(chan T, n)`, pengirim cuma blocking kalau buffer penuh).

### Kenapa dibutuhkan?
`ProcessOrdersWorkerPool` (topik 60) butuh cara buat mengalirkan job (`Order`) dari satu sumber (hasil `ConsumeOrderEvents`, phase 05) ke banyak worker goroutine, dan mengalirkan balik hasilnya (`Result`) tanpa race condition (topik 58). Channel menyediakan mekanisme ini secara built-in: aman dipakai concurrent dari banyak goroutine sekaligus, tanpa perlu lock manual di level pemakainya.

### Cara Kerja
```
Unbuffered channel (synchronous handoff):
  Goroutine A: ch <- order          (BLOCKING sampai ada yang nerima)
  Goroutine B:     order := <-ch    (BLOCKING sampai ada yang ngirim)
  Keduanya "bertemu" tepat saat handoff terjadi.

Buffered channel, kapasitas 3:
  ch <- order1  -> masuk buffer (gak blocking, buffer: [o1])
  ch <- order2  -> masuk buffer (gak blocking, buffer: [o1, o2])
  ch <- order3  -> masuk buffer (gak blocking, buffer: [o1, o2, o3], PENUH)
  ch <- order4  -> BLOCKING sampai ada yang narik satu dari buffer

close(ch):
  - Cuma PENGIRIM yang boleh close, gak pernah penerima.
  - Kirim ke channel yang sudah closed -> PANIC.
  - Terima dari channel yang sudah closed -> langsung return zero value,
    gak blocking (dicek lewat `v, ok := <-ch`, ok == false kalau closed & kosong).
```

### Contoh Kode — Go
```go
package worker

import "context"

// FanOutOrders adalah contoh sederhana pola channel: satu goroutine
// (producer) mengalirkan order ke channel jobs -- pondasi sebelum jadi
// worker pool penuh di topik 60. Order sudah didefinisikan di topik 55.
func FanOutOrders(ctx context.Context, orders []Order) <-chan Order {
	jobs := make(chan Order, len(orders)) // buffered supaya producer gak blocking
	go func() {
		defer close(jobs) // WAJIB: cuma producer yang close, menandakan "gak ada job lagi"
		for _, order := range orders {
			select {
			case jobs <- order:
			case <-ctx.Done():
				return
			}
		}
	}()
	return jobs
}

// DrainJobs membaca semua job dari channel sampai channel-nya ditutup --
// pola `for order := range jobs` otomatis berhenti begitu channel closed
// dan buffer-nya sudah kosong.
func DrainJobs(jobs <-chan Order) []Order {
	var drained []Order
	for order := range jobs {
		drained = append(drained, order)
	}
	return drained
}
```

### Contoh Kode — Node.js
Node.js gak punya primitive channel bawaan. Padanan realistisnya ada dua: `EventEmitter` untuk pola pub/sub fire-and-forget (gak ada backpressure/blocking bawaan), atau queue berbasis Promise buatan sendiri kalau butuh semantic "tunggu sampai ada item" seperti channel. Bedanya penting: channel Go (unbuffered) BISA blocking si pengirim sampai ada penerima; `EventEmitter.emit()` TIDAK PERNAH blocking dan kalau gak ada listener, event-nya hilang begitu saja (gak ada "buffer" otomatis).
```javascript
const { EventEmitter } = require('events');

// fanOutOrders pakai EventEmitter -- ANALOGINYA TERBATAS: emit() gak
// blocking (beda dari unbuffered channel Go) dan kalau listener belum
// terpasang saat emit() dipanggil, event itu hilang, gak diantre otomatis.
function fanOutOrders(orders) {
  const emitter = new EventEmitter();
  process.nextTick(() => {
    for (const order of orders) {
      emitter.emit('job', order);
    }
    emitter.emit('done');
  });
  return emitter;
}

// AsyncQueue adalah queue berbasis Promise -- padanan yang lebih dekat ke
// channel Go karena pull() BENAR-BENAR menunggu (blocking secara async)
// sampai ada item atau queue ditutup, mirip `<-ch` di Go. Dipakai lagi di
// topik 60 dan 62 sebagai jobsQueue.
class AsyncQueue {
  constructor() {
    this._items = [];
    this._waiters = [];
    this._closed = false;
  }

  push(item) {
    if (this._closed) throw new Error('cannot push to a closed queue');
    if (this._waiters.length > 0) this._waiters.shift()({ value: item, done: false });
    else this._items.push(item);
  }

  pull() {
    if (this._items.length > 0) return Promise.resolve({ value: this._items.shift(), done: false });
    if (this._closed) return Promise.resolve({ value: undefined, done: true });
    return new Promise((resolve) => this._waiters.push(resolve));
  }

  close() {
    this._closed = true;
    while (this._waiters.length > 0) this._waiters.shift()({ value: undefined, done: true });
  }
}

module.exports = { fanOutOrders, AsyncQueue };
```

### Trade-off & Pitfall
- Buffered channel yang terlalu besar bisa menyembunyikan masalah backpressure — producer terasa "lancar" terus menerus ngirim job padahal consumer sebenarnya udah gak sanggup ngejar, sampai akhirnya buffer penuh dan producer baru berhenti mendadak.
- Lupa `close()` channel di sisi producer bikin `for range jobs` di consumer nunggu selamanya (goroutine leak) — sebaliknya, close channel dari sisi PENERIMA (bukan pengirim) atau close dua kali menyebabkan panic runtime.
- `EventEmitter` di Node.js gak punya backpressure bawaan sama sekali — kalau listener-nya lambat memproses tiap event, `emit()` berikutnya tetap jalan terus tanpa menunggu, beda jauh dari channel Go yang otomatis blocking pengirim kalau buffer penuh.

### Kapan Dipakai
Setiap kali goroutine perlu saling kirim data atau sinyal (misalnya job ke worker pool, sinyal selesai) — channel adalah cara idiomatic Go buat sinkronisasi antar goroutine tanpa lock manual.

### Sering Ditanya Saat Interview
- "Kapan pakai buffered vs unbuffered channel?" — unbuffered kalau butuh jaminan "pengirim tau penerima udah benar-benar menerima" (rendezvous point); buffered kalau cuma butuh antrean sementara dan pengirim gak perlu nunggu penerima siap, dengan kapasitas terbatas sebagai bentuk backpressure sederhana.
- "Siapa yang boleh close channel?" — cuma goroutine PENGIRIM (producer), gak pernah penerima; close menandakan "gak akan ada nilai baru lagi", dan mengirim ke channel yang sudah closed menyebabkan panic.
- "Kenapa Node.js gak punya channel seperti Go?" — karena model concurrency-nya beda: Node.js single-threaded dengan event loop, jadi gak butuh primitive sinkronisasi antar-alur-eksekusi-paralel seperti channel; kalau butuh semantic serupa (menunggu sampai ada item), harus dibikin manual pakai queue berbasis Promise.

---

## 57. Mutex

### Apa itu?
Mutex (mutual exclusion) adalah primitive sinkronisasi yang memastikan cuma satu goroutine yang boleh mengakses sebuah critical section (biasanya shared state) di satu waktu. Di Go, `sync.Mutex` punya dua method: `Lock()` dan `Unlock()` — goroutine yang panggil `Lock()` akan blocking kalau mutex-nya lagi dipegang goroutine lain, sampai goroutine itu `Unlock()`.

### Kenapa dibutuhkan?
Kalau banyak worker di `ProcessOrdersWorkerPool` (topik 60) sama-sama meng-increment sebuah counter in-memory (misalnya "total order berhasil diproses") tanpa sinkronisasi, hasilnya gak bisa diprediksi — ini persis race condition yang dibahas di topik 58. Mutex memastikan operasi baca-modifikasi-tulis pada shared state itu jadi atomik dari sudut pandang goroutine lain.

### Cara Kerja
```
Tanpa mutex (BERBAHAYA, race condition):
  Goroutine A: baca counter (0) -> tambah 1 -> tulis counter (1)
  Goroutine B: baca counter (0) -> tambah 1 -> tulis counter (1)
  Kedua goroutine baca nilai yang SAMA (0) sebelum sempat saling tulis ->
  counter akhir = 1, PADAHAL seharusnya 2 (dua increment).

Dengan mutex:
  Goroutine A: Lock() -> baca(0) -> tambah 1 -> tulis(1) -> Unlock()
  Goroutine B:          [BLOCKING nunggu Lock()]... -> Lock() -> baca(1) -> tambah 1 -> tulis(2) -> Unlock()
  Cuma satu goroutine yang mengeksekusi critical section di satu waktu ->
  counter akhir = 2, SESUAI ekspektasi.
```

### Contoh Kode — Go
```go
package worker

import "sync"

// ProcessedCounter melacak jumlah order yang berhasil diproses worker pool
// -- diakses dari banyak goroutine worker sekaligus, jadi WAJIB dilindungi
// mutex.
type ProcessedCounter struct {
	mu    sync.Mutex
	count int
}

// Increment menaikkan counter secara aman walau dipanggil dari banyak
// goroutine secara bersamaan -- Lock/Unlock memastikan cuma satu goroutine
// yang mengeksekusi c.count++ di satu waktu.
func (c *ProcessedCounter) Increment() {
	c.mu.Lock()
	defer c.mu.Unlock() // defer memastikan Unlock tetap terpanggil walau ada panic
	c.count++
}

// Value membaca nilai counter saat ini -- pembacaan JUGA harus lewat lock,
// bukan cuma penulisan, supaya gak baca nilai yang lagi ditulis goroutine lain.
func (c *ProcessedCounter) Value() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.count
}
```

### Contoh Kode — Node.js
Untuk kode async biasa (single-threaded event loop), Node.js TIDAK butuh mutex seperti Go — JavaScript sinkron di antara dua titik `await` gak pernah diinterupsi goroutine/thread lain, jadi operasi seperti `counter++` (tanpa `await` di tengahnya) otomatis atomik. TAPI kalau pakai `worker_threads` dengan `SharedArrayBuffer` (memory yang BENAR-BENAR dibagi antar thread), race condition sungguhan bisa terjadi lagi, dan padanan terdekat mutex adalah `Atomics`.
```javascript
// ProcessedCounter versi async biasa -- AMAN tanpa lock karena JavaScript
// single-threaded: di antara `this.count++` gak ada titik `await` yang bisa
// diselipi kode lain, jadi operasi ini atomik secara alami di event loop.
class ProcessedCounter {
  constructor() {
    this.count = 0;
  }
  increment() {
    this.count++; // aman TANPA lock, selama gak ada `await` di tengah operasi ini
  }
  value() {
    return this.count;
  }
}

// incrementSharedCounter dipakai kalau counter dibagi antar worker_threads
// SUNGGUHAN (bukan cuma antar async function di thread yang sama) -- di
// sinilah race condition beneran bisa muncul di Node.js, dan Atomics.add()
// adalah padanan terdekat operasi atomik seperti mutex.
function incrementSharedCounter(sharedInt32Array) {
  // Atomics.add melakukan baca-tambah-tulis sebagai satu operasi atomik di
  // level memory, mencegah race antar worker_threads yang berbagi buffer
  // yang sama -- ini BUKAN mutex (gak ada Lock/Unlock, gak ada blocking),
  // tapi menyelesaikan masalah yang sama untuk operasi sederhana seperti ini.
  Atomics.add(sharedInt32Array, 0, 1);
}

module.exports = { ProcessedCounter, incrementSharedCounter };
```

### Trade-off & Pitfall
- Lupa `Unlock()` (misalnya lupa pakai `defer` dan ada early return di antara `Lock()` dan `Unlock()`) bikin semua goroutine lain yang butuh mutex yang sama blocking SELAMANYA — ini salah satu penyebab paling umum deadlock (topik 59).
- Critical section yang kelamaan dipegang (misalnya melakukan I/O di dalam `Lock()`/`Unlock()`) memperbesar contention — goroutine lain jadi antre lebih lama, mengurangi manfaat concurrency yang tadinya mau didapat.
- Di Node.js, godaan untuk "menambahkan lock" pada kode async biasa itu SIA-SIA dan bahkan bisa jadi sumber bug baru (misalnya lock yang gak pernah dilepas kalau ada exception) — masalah race condition di kode async Node.js biasanya bukan soal "butuh mutex", tapi soal urutan `await` (dibahas di topik 58).

### Kapan Dipakai
Setiap kali ada shared mutable state (seperti counter, map, atau struct) yang diakses dari lebih dari satu goroutine secara bersamaan, dan operasinya bukan cuma baca (kalau cuma baca-baca aja tanpa pernah ditulis lagi setelah inisialisasi, gak butuh mutex).

### Sering Ditanya Saat Interview
- "Kenapa `Value()` juga perlu di-lock, padahal cuma membaca?" — karena tanpa lock, pembacaan bisa terjadi tepat di tengah-tengah penulisan oleh goroutine lain, menghasilkan nilai yang gak konsisten; lock memastikan baca dan tulis saling eksklusif.
- "Kenapa Node.js versi async biasa gak butuh mutex buat `counter++`?" — karena JavaScript menjalankan kode secara single-threaded, dan operasi tanpa `await` di tengahnya gak pernah diinterupsi oleh task lain, sehingga otomatis atomik dari sudut pandang event loop; ini beda dari Go di mana goroutine benar-benar berjalan concurrent (bisa paralel di multi-core).
- "Kapan Node.js justru BISA punya race condition sungguhan yang butuh Atomics?" — kalau memory-nya benar-benar dibagi antar `worker_threads` lewat `SharedArrayBuffer`; di situ, beberapa OS thread sungguhan bisa mengakses memory yang sama secara paralel, dan `Atomics` dipakai buat operasi read-modify-write yang atomik.

---

## 58. Race Condition

### Apa itu?
Race condition terjadi ketika hasil program tergantung pada urutan/timing eksekusi dua atau lebih goroutine yang mengakses shared data yang sama, dan minimal salah satunya menulis, tanpa sinkronisasi yang tepat. Go punya tool bawaan buat mendeteksinya: `go run -race` atau `go test -race`, yang menginstrumentasi akses memory saat runtime dan melaporkan race yang sebenarnya terjadi (bukan cuma potensi race secara statis).

### Kenapa dibutuhkan?
Race condition adalah kelas bug yang PALING gampang ditulis gak sengaja di kode concurrent — kelihatan benar saat ditest sekali-dua kali (sering kali gak muncul konsisten), tapi bisa menghasilkan data korup atau hasil yang salah secara acak di production, terutama di bawah beban tinggi. `ProcessedCounter` (topik 57) yang lupa dikasih mutex adalah salah satu contoh paling umum.

### Cara Kerja
```
go run -race main.go, dengan 2 goroutine sama-sama increment counter TANPA mutex:

  Goroutine A                          Goroutine B
  ----------                           ----------
  tmp := counter   (baca: 0)
                                        tmp := counter   (baca: 0)   <- BACA SEBELUM A SEMPAT NULIS
  counter = tmp+1  (tulis: 1)
                                        counter = tmp+1  (tulis: 1)  <- HARUSNYA 2, JADI 1 (LOST UPDATE)

-race detector akan mencetak:
  WARNING: DATA RACE
  Write at 0x00c0000... by goroutine 7:
    worker.(*UnsafeCounter).Increment()
  Previous read at 0x00c0000... by goroutine 6:
    worker.(*UnsafeCounter).Increment()
```

### Contoh Kode — Go
```go
package worker

import "sync"

// UnsafeCounter ANTI-CONTOH: counter tanpa mutex yang diakses dari banyak
// goroutine sekaligus -- ini PERSIS jenis kode yang akan ditangkap
// `go test -race`. JANGAN dipakai di kode sungguhan; ini sengaja ditulis
// untuk demonstrasi race condition.
type UnsafeCounter struct {
	count int
}

func (c *UnsafeCounter) Increment() {
	c.count++ // BUKAN operasi atomik -- baca, tambah, tulis, tiga langkah terpisah
}

// DemonstrateRaceCondition menjalankan 100 goroutine yang sama-sama
// Increment() UnsafeCounter -- jalankan dengan `go run -race` untuk melihat
// race detector menangkapnya. Hasil akhir count SERINGKALI < 100 (lost
// update), meskipun kadang kebetulan pas 100 -- itulah kenapa race condition
// berbahaya: gak konsisten, sulit dideteksi cuma dari observasi hasil.
func DemonstrateRaceCondition() int {
	c := &UnsafeCounter{}
	var wg sync.WaitGroup
	for i := 0; i < 100; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			c.Increment()
		}()
	}
	wg.Wait()
	return c.count // ANTI-CONTOH: expected 100, actual sering < 100
}

// FixedCounter adalah perbaikannya -- sama seperti ProcessedCounter (topik
// 57), pakai sync.Mutex supaya Increment jadi atomik dari sudut pandang
// goroutine lain. Jalankan `go test -race` pada versi ini: TIDAK ada
// warning, dan hasil SELALU tepat 100.
type FixedCounter struct {
	mu    sync.Mutex
	count int
}

func (c *FixedCounter) Increment() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.count++
}

func (c *FixedCounter) Value() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.count
}
```

### Contoh Kode — Node.js
JavaScript single-threaded gak punya race condition di level operasi primitif sesederhana `counter++` (topik 57) — tapi race condition LOGIS tetap bisa terjadi di kode async, dalam bentuk "check-then-act" yang terinterupsi oleh `await`: antara mengecek suatu kondisi dan bertindak berdasarkan kondisi itu, event loop bisa saja menjalankan kode LAIN dulu (termasuk pemanggilan yang sama dari request lain) sebelum tindakannya selesai.
```javascript
// ANTI-CONTOH: reserveStockUnsafe kelihatan aman (cek dulu baru kurangi),
// tapi antara `await getStock(productId)` dan `await setStock(...)` ada
// TITIK await -- di titik itu, event loop BEBAS menjalankan
// reserveStockUnsafe() lain (misalnya dari request customer berbeda yang
// berebut stock yang sama) sebelum yang pertama sempat menulis stock barunya.
async function reserveStockUnsafe(db, productId, qty) {
  const stock = await db.getStock(productId); // titik await #1 -- interupsi bisa terjadi di sini
  if (stock < qty) {
    throw new Error('insufficient stock');
  }
  await db.setStock(productId, stock - qty); // titik await #2 -- pakai `stock` yang mungkin sudah basi
}

// Dua request bersamaan, stock awal = 5, masing-masing minta qty = 5:
//   Request A: getStock() -> 5 (OK, 5 >= 5)
//   Request B: getStock() -> 5 (OK, 5 >= 5)   <- interleaved SEBELUM A setStock()
//   Request A: setStock(5 - 5 = 0)
//   Request B: setStock(5 - 5 = 0)            <- HARUSNYA GAGAL, tapi lolos -- overselling!

// FIX: pindahkan check-and-act ke satu statement atomik di level database
// (bukan baca-lalu-tulis dari application code), misalnya conditional
// UPDATE yang gagal kalau stock gak cukup.
async function reserveStockSafe(db, productId, qty) {
  // UPDATE ... WHERE stock >= qty mengembalikan jumlah row yang ke-update;
  // 0 berarti stock gak cukup -- SATU round-trip atomik, gak ada window
  // untuk interleaving seperti reserveStockUnsafe.
  const updatedRows = await db.query(
    'UPDATE products SET stock = stock - $1 WHERE id = $2 AND stock >= $1',
    [qty, productId]
  );
  if (updatedRows === 0) {
    throw new Error('insufficient stock');
  }
}

module.exports = { reserveStockUnsafe, reserveStockSafe };
```

### Trade-off & Pitfall
- Race condition sering gak konsisten muncul — kode yang "kelihatan jalan" pas testing manual bisa tetap punya race yang cuma muncul di bawah beban production yang tinggi; jangan pernah menyimpulkan "aman" cuma dari observasi hasil tanpa `go test -race`.
- Di Node.js, "race condition" lebih sering muncul sebagai bug logika check-then-act yang terpisah oleh `await`, bukan corrupted memory seperti di Go — fix-nya biasanya memindahkan operasi ke satu round-trip database yang atomik (seperti conditional `UPDATE`), bukan menambahkan lock di application code.
- `go test -race` cuma mendeteksi race yang BENAR-BENAR TEREKSEKUSI selama test berjalan — kalau code path racy-nya kebetulan gak kepicu oleh test case yang ada, race detector gak akan melaporkan apa-apa walau bug-nya tetap ada.

### Kapan Dipakai
"Kapan dipakai" gak relevan buat topik ini secara langsung (race condition adalah bug, bukan tools) — tapi `go test -race`/`go run -race` WAJIB dijalankan di CI untuk semua kode Go yang concurrent, dan review kode async Node.js harus selalu curiga terhadap check-then-act yang dipisah oleh `await`.

### Sering Ditanya Saat Interview
- "Apa itu race condition dan kenapa berbahaya?" — hasil program yang tergantung timing eksekusi goroutine yang mengakses shared data tanpa sinkronisasi; berbahaya karena seringkali gak konsisten muncul, bisa lolos testing manual tapi menyebabkan data korup di production di bawah beban tinggi.
- "Kenapa Node.js single-threaded masih bisa punya race condition?" — karena walau JavaScript-nya sendiri single-threaded, operasi async (seperti query database) menciptakan titik interupsi (`await`) di mana kode lain bisa berjalan duluan; check-then-act yang dipisah oleh titik interupsi ini punya window yang sama berbahayanya dengan race condition di Go, walau mekanismenya beda.
- "Gimana cara Go mendeteksi race condition?" — pakai `-race` flag (`go run -race`, `go test -race`) yang menginstrumentasi akses memory saat runtime dan melaporkan kalau ada read/write ke lokasi memory yang sama dari goroutine berbeda tanpa sinkronisasi yang valid.

---

## 59. Deadlock in Go

### Apa itu?
Deadlock terjadi ketika sekumpulan goroutine saling menunggu satu sama lain sehingga TIDAK ADA yang bisa lanjut jalan, selamanya. Go runtime bisa mendeteksi kasus paling ekstrem-nya: kalau SEMUA goroutine di program (termasuk goroutine utama) sama-sama blocking menunggu sesuatu yang gak akan pernah terjadi, runtime akan panic dengan pesan `fatal error: all goroutines are asleep - deadlock!` dan program berhenti. Tapi deadlock PARSIAL (cuma sebagian goroutine yang macet, ada goroutine lain yang masih jalan) TIDAK terdeteksi otomatis — program tetap "hidup" tapi sebagian pekerjaannya macet selamanya.

### Kenapa dibutuhkan?
Worker pool (topik 60) yang melibatkan banyak goroutine, channel, dan kadang beberapa mutex sekaligus rawan deadlock kalau urutan locking-nya gak konsisten, atau kalau ada channel yang dikirimi tanpa ada yang pernah menerimanya. Memahami pola-pola penyebab deadlock penting supaya bisa dihindari dari desain, bukan cuma ditambal setelah production hang.

### Cara Kerja
```
Pola #1 -- unbuffered channel tanpa penerima:
  ch := make(chan int)
  ch <- 1        // BLOCKING selamanya, gak ada goroutine lain yang <-ch
  fmt.Println("never reached")
  -> fatal error: all goroutines are asleep - deadlock!  (goroutine utama satu-satunya, dan dia yang blocking)

Pola #2 -- circular lock ordering (klasik "dining philosophers"):
  Goroutine A: Lock(mutexX) -> ... -> Lock(mutexY)   (urutan: X lalu Y)
  Goroutine B: Lock(mutexY) -> ... -> Lock(mutexX)   (urutan: Y lalu X -- KEBALIKANNYA)

  Timeline yang menyebabkan deadlock:
  A: Lock(mutexX) berhasil
  B: Lock(mutexY) berhasil
  A: Lock(mutexY) -> BLOCKING (dipegang B)
  B: Lock(mutexX) -> BLOCKING (dipegang A)
  -> A menunggu B, B menunggu A, SELAMANYA (deadlock parsial, goroutine
     lain di program masih bisa jalan normal -- makanya TIDAK terdeteksi
     otomatis oleh runtime).
```

### Contoh Kode — Go
```go
package worker

import "sync"

// Account dipakai untuk mendemonstrasikan deadlock lewat circular lock
// ordering (pola #2) -- dua mutex (masing-masing milik satu Account) yang
// dikunci dalam urutan berbeda oleh goroutine berbeda.
type Account struct {
	mu      sync.Mutex
	ID      int
	Balance float64
}

// TransferBetweenAccountsUnsafe ANTI-CONTOH: mengunci dua mutex (akun asal
// dan akun tujuan) TANPA urutan yang konsisten -- urutan locking di sini
// tergantung urutan parameter from/to yang dipanggil. Kalau dipanggil dari
// dua goroutine dengan arah yang saling bertukar (A->B dan B->A bersamaan),
// ini bisa deadlock persis seperti pola #2 di atas.
func TransferBetweenAccountsUnsafe(from, to *Account, amount float64) {
	from.mu.Lock() // ANTI-CONTOH: lock "from" duluan, urutan tergantung urutan parameter
	defer from.mu.Unlock()
	to.mu.Lock() // kalau goroutine lain lock ke arah sebaliknya -> DEADLOCK
	defer to.mu.Unlock()

	from.Balance -= amount
	to.Balance += amount
}

// TransferBetweenAccountsSafe FIX: selalu lock mutex dalam urutan yang
// KONSISTEN (berdasarkan ID akun, bukan urutan parameter from/to) --
// menghilangkan kemungkinan circular wait sama sekali, berapa pun urutan
// from/to yang dipanggil oleh pemanggil yang berbeda-beda.
func TransferBetweenAccountsSafe(from, to *Account, amount float64) {
	first, second := from, to
	if first.ID > second.ID {
		first, second = second, first
	}
	first.mu.Lock()
	defer first.mu.Unlock()
	second.mu.Lock()
	defer second.mu.Unlock()

	from.Balance -= amount
	to.Balance += amount
}
```

### Contoh Kode — Node.js
Karena Node.js single-threaded, deadlock dalam pengertian Go (OS/goroutine sungguhan saling blocking, CPU menganggur menunggu lock) TIDAK bisa terjadi — event loop selalu bisa lanjut mengerjakan task/timer/event LAIN, gak akan pernah "membeku total" seperti proses Go yang semua goroutine-nya asleep. Tapi ada pola analog yang punya efek serupa: sebuah rantai `Promise` yang saling menunggu secara sirkular hasilnya SAMA-SAMA gak pernah resolve — cuma bedanya, ini bukan "seluruh runtime hang", cuma alur logika tertentu yang macet selamanya, dan Node.js TIDAK punya deteksi otomatis seperti pesan `fatal error` di Go.
```javascript
const { EventEmitter } = require('events');

// createDeadlockAnalog ANTI-CONTOH: dua fungsi saling menunggu event dari
// satu sama lain lewat EventEmitter -- stepA gak akan pernah selesai
// karena nunggu event dari stepB, dan stepB gak akan pernah emit event itu
// karena dia sendiri lagi nunggu stepA selesai duluan. Ini analog circular
// wait di Go (pola #2), meski gak bikin seluruh proses Node.js hang -- cuma
// alur ini yang macet selamanya.
function createDeadlockAnalog() {
  const bus = new EventEmitter();

  async function stepA() {
    await new Promise((resolve) => bus.once('stepB-done', resolve)); // nunggu B selamanya
    return 'stepA done';
  }

  async function stepB() {
    await new Promise((resolve) => bus.once('stepA-done', resolve)); // nunggu A selamanya
    bus.emit('stepB-done');
    return 'stepB done';
  }

  return Promise.all([stepA(), stepB()]); // Promise ini TIDAK PERNAH resolve
}

// createFixedFlow FIX: hilangkan circular dependency-nya -- stepB menunggu
// PROMISE stepA secara langsung (dependency satu arah), BUKAN saling
// menunggu event lewat EventEmitter. Ini juga sengaja menghindari jebakan
// EventEmitter dari topik 56: emit() gak menunggu listener terpasang dulu,
// jadi kalau stepA emit SEBELUM stepB sempat mendaftarkan listener-nya,
// event itu hilang dan stepB tetap menggantung -- makanya fix yang benar
// bukan "atur ulang urutan pendaftaran listener", tapi hilangkan pola
// saling-tunggu-event-nya sama sekali.
function createFixedFlow() {
  async function stepA() {
    return 'stepA done';
  }

  async function stepB(stepAPromise) {
    await stepAPromise; // bergantung LANGSUNG ke hasil stepA, bukan sirkular
    return 'stepB done';
  }

  const a = stepA();
  return Promise.all([a, stepB(a)]); // resolve dengan normal, gak ada siklus tunggu
}

module.exports = { createDeadlockAnalog, createFixedFlow };
```

### Trade-off & Pitfall
- Deadlock TOTAL (semua goroutine asleep) gampang ketahuan karena program langsung crash dengan pesan jelas; deadlock PARSIAL (cuma sebagian goroutine macet) jauh lebih berbahaya karena program kelihatan "masih hidup" (goroutine lain tetap jalan, health check masih 200 OK) padahal sebagian fungsionalitas diam-diam berhenti selamanya.
- Urutan locking yang konsisten (seperti `TransferBetweenAccountsSafe`) menghilangkan circular wait, tapi menambah kompleksitas kode dan sedikit overhead (perbandingan sebelum locking) — untuk sistem dengan banyak sekali jenis lock, pertimbangkan juga desain ulang supaya gak butuh lock lebih dari satu sekaligus.
- Analog "deadlock" di Node.js (Promise yang saling menunggu secara sirkular) TIDAK dideteksi otomatis oleh runtime sama sekali — gak ada pesan error, prosesnya cuma "menggantung" di alur itu selamanya; harus ketahuan lewat monitoring/timeout manual (`Promise.race` dengan timeout) atau memang dicegah dari desain (topologi dependency yang gak sirkular).

### Kapan Dipakai
"Kapan dipakai" gak relevan (ini bug pattern yang harus DIHINDARI, bukan dipakai) — tapi kesadaran akan pola-pola penyebabnya (unbuffered channel tanpa penerima, circular lock ordering) WAJIB dipakai sebagai checklist review setiap kali menulis kode concurrent yang melibatkan lebih dari satu channel/mutex.

### Sering Ditanya Saat Interview
- "Apa syarat classic deadlock (empat kondisi Coffman)?" — mutual exclusion (resource cuma bisa dipegang satu pihak), hold-and-wait (memegang satu resource sambil menunggu resource lain), no preemption (resource gak bisa direbut paksa), dan circular wait (rantai tunggu yang membentuk siklus); menghilangkan salah satu saja (biasanya circular wait, lewat urutan locking konsisten) sudah cukup mencegah deadlock.
- "Kenapa Go bisa mendeteksi deadlock total tapi gak deadlock parsial?" — runtime Go tau persis kalau SEMUA goroutine yang ada (termasuk main) lagi blocking dan gak ada satupun yang bisa dijadwalkan lagi — itu sinyal jelas gak akan pernah ada progress lagi. Deadlock parsial (goroutine lain masih jalan normal) gak punya sinyal seperti itu dari sudut pandang runtime, karena program secara keseluruhan masih "hidup".
- "Kenapa Node.js gak bisa mengalami deadlock seperti Go?" — karena event loop-nya single-threaded dan selalu bisa lanjut mengerjakan task/timer lain yang gak terkait, program secara keseluruhan gak akan pernah "membeku total"; tapi alur logika tertentu (misalnya Promise yang saling menunggu secara sirkular) tetap bisa macet selamanya tanpa deteksi otomatis dari runtime.

---

## 60. Worker Pool

### Apa itu?
Worker pool adalah pola menjalankan sejumlah TETAP goroutine ("worker") yang sama-sama mengambil job dari satu channel yang dibagi bersama, alih-alih membuat satu goroutine baru per job seperti `ProcessOrdersConcurrently` di topik 55. Di OrderFlow, `ProcessOrdersWorkerPool` adalah implementasi pola ini: menerima channel `jobs` (diisi dari event `OrderCreated` yang dibaca `ConsumeOrderEvents`, phase 05) dan jumlah `workers`, lalu mengembalikan channel `Result` yang diisi begitu tiap job selesai diproses.

### Kenapa dibutuhkan?
Event `OrderCreated` yang mengalir lewat `ConsumeOrderEvents` bisa datang dalam volume yang gak terduga — kalau tiap event langsung diproses di goroutine barunya sendiri (topik 55), lonjakan traffic bisa memicu jutaan goroutine hidup bersamaan, menghabiskan memory dan membebani scheduler. Worker pool membatasi concurrency ke angka yang terkontrol (`workers`), sehingga throughput pemrosesan bisa diprediksi dan resource pemakaiannya terbatas, apapun volume event yang masuk.

### Cara Kerja
```
ConsumeOrderEvents (phase 05) membaca topic "order.created"
        |
        v
   channel jobs (chan Order)
        |
   +----+----+----+
   v    v    v    v
Worker1 Worker2 Worker3 ... WorkerN   (jumlah TETAP = `workers`)
   |    |    |    |
   +----+----+----+
        |
        v
   channel results (chan Result)  <- dikembalikan ProcessOrdersWorkerPool

Tiap worker menjalankan loop: ambil satu Order dari jobs, proses
(processOrder), kirim Result ke results, ulangi -- sampai channel jobs
ditutup (dari GracefulShutdown, topik 62) ATAU context dibatalkan.
```

### Contoh Kode — Go
```go
package worker

import (
	"context"
	"sync"
)

// ProcessOrdersWorkerPool menjalankan `workers` goroutine yang sama-sama
// mengambil job dari channel `jobs` (biasanya diisi dari event OrderCreated
// lewat ConsumeOrderEvents, phase 05) dan mengirim hasilnya ke channel yang
// dikembalikan. Jumlah goroutine DIBATASI ke `workers`, bukan satu per
// Order (topik 55), supaya lonjakan event OrderCreated gak menyebabkan
// goroutine tak terbatas. Order, Result, dan processOrder sudah
// didefinisikan di topik 55, dipakai lagi di sini.
func ProcessOrdersWorkerPool(ctx context.Context, jobs <-chan Order, workers int) <-chan Result {
	results := make(chan Result)
	var wg sync.WaitGroup

	for i := 0; i < workers; i++ {
		wg.Add(1)
		go func(workerID int) {
			defer wg.Done()
			for {
				select {
				case order, ok := <-jobs:
					if !ok {
						return // channel jobs ditutup (graceful shutdown, topik 62) -> worker berhenti
					}
					select {
					case results <- processOrder(order):
					case <-ctx.Done():
						return
					}
				case <-ctx.Done():
					return
				}
			}
		}(i)
	}

	// Goroutine terpisah ini menunggu SEMUA worker selesai (wg.Wait()) baru
	// menutup channel results -- supaya consumer channel results tau kapan
	// harus berhenti membaca (lewat `for range results`), dan supaya close
	// cuma dipanggil SEKALI walau ada banyak worker.
	go func() {
		wg.Wait()
		close(results)
	}()

	return results
}
```

### Contoh Kode — Node.js
```javascript
// processOrder mensimulasikan pemrosesan satu order -- dipisah dari
// processOrdersWorkerPool supaya bisa diuji sendiri (topik 64).
function processOrder(order) {
  if (order.total <= 0) {
    return { orderId: order.id, err: new Error(`order ${order.id}: total must be positive, got ${order.total}`) };
  }
  return { orderId: order.id, err: null };
}

// processOrdersWorkerPool menjalankan `workerCount` loop async konkuren
// yang menarik job dari jobsQueue (AsyncQueue, topik 56) -- KARENA Node.js
// single-threaded, ini BUKAN paralelisme sungguhan seperti goroutine
// (topik 55): tiap "worker" di sini cuma alur async yang mengambil giliran
// di event loop yang sama. Cocok untuk job yang I/O-bound (misalnya update
// status ke database), TAPI gak memberi keuntungan CPU paralel untuk job
// yang CPU-bound berat (butuh worker_threads untuk itu).
async function processOrdersWorkerPool(jobsQueue, workerCount, onResult) {
  const workers = Array.from({ length: workerCount }, (_, workerId) => runWorker(jobsQueue, workerId, onResult));
  await Promise.all(workers);
}

async function runWorker(jobsQueue, workerId, onResult) {
  for (;;) {
    const { value: order, done } = await jobsQueue.pull();
    if (done) return; // jobsQueue ditutup (graceful shutdown, topik 62) -> worker berhenti
    onResult(processOrder(order));
  }
}

module.exports = { processOrdersWorkerPool, processOrder };
```

### Trade-off & Pitfall
- Jumlah `workers` yang terlalu kecil bikin job menumpuk di channel `jobs` (latency naik walau CPU/network sebenarnya masih sanggup); terlalu besar bisa membebani resource downstream yang dipanggil tiap worker (misalnya koneksi database ikut menyentuh connection pool limit) — angka ini harus di-tuning berdasarkan karakteristik beban sungguhan, bukan ditebak.
- Kalau salah satu job menyebabkan `panic` di dalam worker Go dan gak ada `recover()`, panic itu bisa mematikan seluruh proses (worker lain, bahkan HTTP server), bukan cuma job yang bermasalah — worker pool produksi biasanya butuh `recover()` per job supaya satu job yang buruk gak menjatuhkan semuanya.
- `processOrdersWorkerPool` di Node.js membagi concurrency lewat event loop yang sama (bukan OS thread terpisah) — kalau `processOrder` ternyata melakukan kerja CPU-bound berat (bukan cuma I/O), semua "worker" akan saling memperlambat satu sama lain karena berebut satu thread JavaScript yang sama; solusinya butuh `worker_threads` (topik 55), bukan sekadar menambah `workerCount`.

### Kapan Dipakai
Setiap kali volume job yang masuk gak terduga/berpotensi melonjak (seperti event `OrderCreated` dari `ConsumeOrderEvents`) dan concurrency-nya perlu dibatasi ke angka yang bisa diprediksi resource usage-nya — worker pool adalah pola standar untuk ini di Go maupun Node.js.

### Sering Ditanya Saat Interview
- "Kenapa gak spawn goroutine per event OrderCreated aja, kayak topik 55?" — kalau volume event melonjak tiba-tiba (misalnya flash sale), jumlah goroutine yang hidup bersamaan ikut melonjak tak terbatas, berpotensi menghabiskan memory; worker pool membatasi concurrency ke angka `workers` yang tetap, apapun volume event yang masuk.
- "Kenapa channel `results` ditutup lewat goroutine terpisah yang nge-`wg.Wait()`, bukan langsung di akhir loop worker?" — kalau tiap worker langsung `close(results)` sendiri-sendiri setelah selesai, worker kedua yang mau close akan panic (double close); dengan `wg.Wait()` di goroutine terpisah, close cuma dipanggil SEKALI, tepat setelah SEMUA worker benar-benar selesai.
- "Apa yang membedakan worker pool Go dan Node.js di soal true parallelism?" — worker Go beneran berjalan paralel di banyak CPU core (lewat OS thread yang dikelola scheduler Go); "worker" Node.js versi `processOrdersWorkerPool` di atas cuma konkuren di satu thread JavaScript yang sama (interleaved lewat event loop), jadi cocok untuk job I/O-bound tapi TIDAK memberi keuntungan CPU paralel untuk job yang CPU-bound berat.

---

## 61. Context

### Apa itu?
`context.Context` di Go adalah value yang dioper eksplisit sebagai parameter pertama fungsi, membawa sinyal pembatalan (cancellation), deadline/timeout, dan kadang value request-scoped (misalnya request ID) — dipropagasi turun ke semua fungsi yang dipanggil, termasuk lintas goroutine. `ConsumeOrderEvents` (phase 05) dan `ProcessOrdersWorkerPool` (topik 60) sama-sama menerima `ctx` sebagai parameter pertama, supaya pembatalan dari luar (misalnya graceful shutdown, topik 62) bisa "menjalar" ke semua goroutine yang lagi bekerja.

### Kenapa dibutuhkan?
Tanpa context, gak ada cara standar buat bilang ke sebuah goroutine yang lagi bekerja "berhenti, kita udah gak butuh hasilmu lagi" — misalnya kalau request HTTP-nya sudah di-cancel client, atau proses aplikasi lagi mau shutdown. `context.Context` menyediakan mekanisme itu secara seragam: fungsi apapun yang menerima `ctx` bisa `select` pada `ctx.Done()` buat tau kapan harus berhenti, tanpa perlu channel pembatalan custom yang beda-beda tiap fungsi.

### Cara Kerja
```
context.WithTimeout(parent, 3*time.Second):
  ctx, cancel := context.WithTimeout(parentCtx, 3*time.Second)
  defer cancel()  <- WAJIB dipanggil walau timeout belum habis, supaya
                     resource internal context (timer) dilepas lebih awal

  select {
  case result := <-doWork(ctx):
      // pekerjaan selesai SEBELUM timeout
  case <-ctx.Done():
      // timeout habis DULUAN -- ctx.Err() == context.DeadlineExceeded
  }

Propagasi lewat parent-child:
  context.Background()
      -> context.WithTimeout(3s)          [child A]
          -> context.WithCancel()          [child B, turunan dari A]
  Kalau A dibatalkan/timeout, B ikut ke-cancel otomatis (Done() tertutup) --
  tapi cancel B TIDAK mempengaruhi A (satu arah: parent -> child).
```

### Contoh Kode — Go
```go
package worker

import (
	"context"
	"fmt"
	"time"
)

// ProcessOrderWithTimeout membatasi waktu pemrosesan satu order -- kalau
// processOrder (topik 55/60) belum selesai sebelum timeout habis, fungsi
// ini berhenti menunggu dan mengembalikan error, TANPA menunggu selamanya.
func ProcessOrderWithTimeout(ctx context.Context, order Order) (Result, error) {
	ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
	defer cancel() // WAJIB, supaya timer internal context dilepas begitu fungsi ini selesai

	done := make(chan Result, 1) // buffered supaya goroutine di bawah gak leak kalau ctx keburu Done
	go func() {
		done <- processOrder(order) // simulasi kerja yang bisa jadi lambat (query database, dst)
	}()

	select {
	case result := <-done:
		return result, nil
	case <-ctx.Done():
		return Result{}, fmt.Errorf("process order %d: %w", order.ID, ctx.Err())
	}
}
```

### Contoh Kode — Node.js
Node.js gak punya `context.Context` bawaan, tapi `AbortController`/`AbortSignal` (standar Web API yang didukung Node.js) memenuhi kebutuhan yang mirip: sinyal pembatalan yang bisa dioper ke fungsi async dan "disebarkan" ke operasi turunannya (`fetch`, banyak API bawaan Node lain sudah menerima `signal`).
```javascript
// processOrderWithTimeout meniru ProcessOrderWithTimeout versi Go, pakai
// AbortController sebagai pengganti context.Context.
async function processOrderWithTimeout(order, timeoutMs) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(new Error(`process order ${order.id}: timeout`)), timeoutMs);

  try {
    return await processOrderAsync(order, controller.signal);
  } finally {
    clearTimeout(timer); // WAJIB, supaya timer gak nyala percuma kalau sudah selesai duluan
  }
}

function processOrderAsync(order, signal) {
  if (signal.aborted) {
    return Promise.reject(signal.reason);
  }
  return new Promise((resolve, reject) => {
    const onAbort = () => reject(signal.reason);
    signal.addEventListener('abort', onAbort, { once: true });
    // simulasi kerja async (query database, dst)
    setImmediate(() => {
      signal.removeEventListener('abort', onAbort);
      resolve({ orderId: order.id, err: order.total <= 0 ? new Error('invalid total') : null });
    });
  });
}

module.exports = { processOrderWithTimeout };
```

### Trade-off & Pitfall
- `context.Context` di Go terintegrasi langsung dengan `select`/channel sebagai bagian dari bahasa (`ctx.Done()` adalah channel biasa) — `AbortSignal` di Node.js berbasis event (`addEventListener('abort', ...)`), jadi tiap fungsi async custom harus SECARA MANUAL mendengarkan event abort dan membatalkan pekerjaannya sendiri; gak ada penjalaran otomatis ke operasi lain seperti propagasi context antar fungsi Go yang semuanya menerima `ctx`.
- Lupa `cancel()` (Go) atau lupa `clearTimeout()` (Node.js) menyebabkan resource (timer internal) gak pernah dilepas sampai garbage collector kebetulan membersihkannya — `go vet` bisa mendeteksi context yang lupa di-cancel di sebagian kasus, tapi gak semua.
- Menyimpan value non-request-scoped (misalnya dependency seperti database pool) di `context.WithValue` adalah anti-pattern umum — context value seharusnya cuma untuk data yang benar-benar melekat ke satu request/operasi (request ID, trace ID), bukan sebagai pengganti dependency injection.

### Kapan Dipakai
Setiap fungsi yang melakukan operasi yang BISA dibatalkan atau punya batas waktu wajar (query database, HTTP call ke service lain, job di worker pool) sebaiknya menerima `ctx`/`signal` sebagai parameter, supaya pemanggilnya (termasuk graceful shutdown, topik 62) punya cara menghentikannya lebih awal kalau perlu.

### Sering Ditanya Saat Interview
- "Kenapa `context.Context` selalu jadi parameter PERTAMA di Go, dan kenapa gak disimpan di struct?" — konvensi ini membuatnya eksplisit di setiap signature fungsi bahwa fungsi itu bisa dibatalkan/punya deadline, dan menghindari context yang "basi" tersimpan lama di struct (context seharusnya berumur sependek operasi yang direpresentasikannya, bukan disimpan permanen).
- "Apa beda `context.WithCancel`, `WithTimeout`, dan `WithDeadline`?" — `WithCancel` memberi fungsi `cancel()` yang dipanggil manual kapan saja; `WithTimeout` otomatis cancel setelah durasi tertentu berlalu; `WithDeadline` otomatis cancel pada waktu absolut tertentu (`WithTimeout` sebenarnya cuma `WithDeadline` dengan `time.Now().Add(d)`).
- "Kenapa `AbortSignal` di Node.js dibilang analoginya gak 1:1 dengan `context.Context`?" — karena `context.Context` di Go terpropagasi otomatis lewat parameter fungsi dan berintegrasi langsung dengan `select`, sementara `AbortSignal` berbasis event yang harus didengarkan manual oleh tiap fungsi async yang ingin mendukungnya — gak ada mekanisme bahasa yang memaksa semua turunannya ikut mendengarkan sinyal abort yang sama secara otomatis.

---

## 62. Graceful Shutdown

### Apa itu?
Graceful shutdown adalah proses mematikan aplikasi dengan cara yang RAPI ketika menerima sinyal berhenti (`SIGTERM` biasanya dari orchestrator seperti Kubernetes, atau `SIGINT` dari Ctrl+C) — berhenti menerima request/job BARU, tapi memberi waktu request/job yang SEDANG berjalan untuk selesai dulu, baru benar-benar exit. Di OrderFlow, `GracefulShutdown` mengoordinasikan mematikan HTTP server DAN `WorkerPool` (topik 60) dalam urutan yang aman.

### Kenapa dibutuhkan?
Kalau proses langsung dimatikan (`kill -9` atau exit paksa) saat ada order yang lagi diproses worker pool, order itu bisa berhenti di tengah jalan — misalnya status-nya kepalang ke-update sebagian, atau event `OrderCreated` yang lagi diproses `ConsumeOrderEvents` (phase 05) belum sempat commit offset (walau itu berarti event-nya akan di-retry lagi lewat at-least-once, phase 05 topik 49 — tetap lebih baik dihindari kalau bisa). Graceful shutdown memberi waktu buat semua pekerjaan yang sedang berjalan selesai dengan bersih dulu.

### Cara Kerja
```
1. Orchestrator (Kubernetes, dst) kirim SIGTERM ke proses OrderFlow.
2. GracefulShutdown menangkap sinyal itu (signal.Notify).
3. HTTP server berhenti MENERIMA koneksi baru (server.Shutdown), tapi
   request yang lagi diproses dibiarkan selesai dulu (sampai batas waktu).
4. WorkerPool.jobs channel ditutup -- gak ada job baru masuk, tapi worker
   yang lagi memproses job yang sudah terlanjur diambil dibiarkan selesai.
5. Tunggu server.Shutdown() DAN pool.Stop() sama-sama selesai, ATAU sampai
   batas waktu keseluruhan habis (misalnya 15 detik) -- kalau lewat batas
   waktu, paksa berhenti (jangan tunggu selamanya).
6. Proses exit dengan bersih.
```

### Contoh Kode — Go
```go
package worker

import (
	"context"
	"fmt"
	"net/http"
	"os"
	"os/signal"
	"sync"
	"syscall"
	"time"
)

// WorkerPool membungkus channel jobs dan mekanisme berhenti yang rapi --
// dibangun di atas ProcessOrdersWorkerPool (topik 60).
type WorkerPool struct {
	jobs    chan Order
	done    chan struct{}
	cancel  context.CancelFunc
	closeMu sync.Mutex
	closed  bool
}

// NewWorkerPool membuat WorkerPool baru dan langsung menjalankan
// ProcessOrdersWorkerPool di baliknya, menyalurkan hasil (Result) lewat
// callback onResult (misalnya buat update status order di database).
func NewWorkerPool(workers int, onResult func(Result)) *WorkerPool {
	ctx, cancel := context.WithCancel(context.Background())
	jobs := make(chan Order)
	results := ProcessOrdersWorkerPool(ctx, jobs, workers)

	done := make(chan struct{})
	go func() {
		for result := range results {
			onResult(result)
		}
		close(done)
	}()

	return &WorkerPool{jobs: jobs, done: done, cancel: cancel}
}

// Jobs mengembalikan channel jobs supaya bisa diisi dari luar (misalnya
// oleh handler ConsumeOrderEvents, phase 05).
func (p *WorkerPool) Jobs() chan<- Order {
	return p.jobs
}

// Stop menutup channel jobs (gak ada job baru masuk) dan menunggu semua
// worker & result-drainer selesai memproses sisa job, atau ctx habis waktu
// duluan (dipaksa berhenti lewat cancel()).
func (p *WorkerPool) Stop(ctx context.Context) error {
	p.closeMu.Lock()
	if !p.closed {
		p.closed = true
		close(p.jobs)
	}
	p.closeMu.Unlock()

	select {
	case <-p.done:
		return nil
	case <-ctx.Done():
		p.cancel() // paksa semua worker berhenti walau masih ada job tersisa
		return fmt.Errorf("worker pool stop: %w", ctx.Err())
	}
}

// GracefulShutdown menunggu sinyal SIGTERM/SIGINT, mematikan HTTP server
// (berhenti menerima koneksi baru, tunggu yang sedang berjalan selesai),
// lalu mematikan WorkerPool dengan cara yang sama -- dengan batas waktu
// keseluruhan supaya proses gak menggantung selamanya kalau ada yang macet.
func GracefulShutdown(ctx context.Context, server *http.Server, pool *WorkerPool) error {
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)
	<-sigCh // blocking sampai ada sinyal shutdown

	shutdownCtx, cancel := context.WithTimeout(ctx, 15*time.Second)
	defer cancel()

	if err := server.Shutdown(shutdownCtx); err != nil {
		return fmt.Errorf("shutdown http server: %w", err)
	}
	if err := pool.Stop(shutdownCtx); err != nil {
		return fmt.Errorf("stop worker pool: %w", err)
	}
	return nil
}
```

### Contoh Kode — Node.js
```javascript
// WorkerPool membungkus AsyncQueue jobs (topik 56) dan Promise
// processOrdersWorkerPool yang sedang berjalan (topik 60), supaya bisa
// dimatikan dengan rapi lewat gracefulShutdown.
class WorkerPool {
  constructor(workerCount, onResult) {
    this.jobsQueue = new AsyncQueue();
    this._closed = false;
    this.donePromise = processOrdersWorkerPool(this.jobsQueue, workerCount, onResult);
  }

  // stop menutup jobsQueue (gak ada job baru masuk) dan menunggu semua
  // worker selesai memproses sisa job, atau timeoutMs habis duluan.
  async stop(timeoutMs) {
    if (!this._closed) {
      this._closed = true;
      this.jobsQueue.close();
    }
    let timer;
    const timeout = new Promise((_, reject) => {
      timer = setTimeout(() => reject(new Error('worker pool stop timed out')), timeoutMs);
    });
    try {
      return await Promise.race([this.donePromise, timeout]);
    } finally {
      clearTimeout(timer);
    }
  }
}

// gracefulShutdown menunggu sinyal SIGTERM/SIGINT, berhenti menerima
// koneksi HTTP baru (server.close menunggu koneksi yang berjalan selesai),
// lalu mematikan worker pool -- meniru urutan yang sama dengan
// GracefulShutdown versi Go.
function gracefulShutdown(server, pool) {
  return new Promise((resolve, reject) => {
    const shutdown = async () => {
      try {
        await new Promise((res, rej) => server.close((err) => (err ? rej(err) : res())));
        await pool.stop(15000);
        resolve();
      } catch (err) {
        reject(err);
      }
    };
    process.once('SIGTERM', shutdown);
    process.once('SIGINT', shutdown);
  });
}

module.exports = { WorkerPool, gracefulShutdown };
```

### Trade-off & Pitfall
- Batas waktu shutdown (15 detik di contoh ini) adalah trade-off antara "beri cukup waktu buat request/job selesai dengan bersih" dan "jangan bikin orchestrator (Kubernetes) menunggu terlalu lama sebelum akhirnya kill paksa" — Kubernetes sendiri punya `terminationGracePeriodSeconds` yang harus lebih besar dari timeout ini, kalau gak, proses akan di-kill paksa sebelum sempat shutdown dengan rapi.
- `server.Shutdown()`/`server.close()` cuma menghentikan koneksi HTTP BARU — kalau ada request yang sudah lama berjalan (misalnya long-polling atau streaming response), graceful shutdown bisa "menggantung" menunggu request itu sampai timeout keseluruhan habis, baru akhirnya dipaksa berhenti.
- Urutan mematikan HTTP server DULU baru worker pool itu penting: kalau worker pool dimatikan duluan sementara HTTP server masih menerima request baru yang butuh mengantre job ke worker pool, request itu akan gagal karena gak ada tempat menaruh job-nya.

### Kapan Dipakai
Wajib ada di semua service yang dijalankan di lingkungan orchestrated (Kubernetes, ECS, dst) yang secara rutin mematikan dan menyalakan ulang instance (deployment baru, autoscaling, node maintenance) — tanpa graceful shutdown, setiap deployment berpotensi memutus request/job yang sedang berjalan secara tiba-tiba.

### Sering Ditanya Saat Interview
- "Kenapa graceful shutdown butuh timeout, bukan menunggu semua pekerjaan selesai tanpa batas?" — supaya proses gak menggantung selamanya kalau ada job/request yang macet atau butuh waktu sangat lama; lebih baik dipaksa berhenti setelah batas waktu wajar daripada orchestrator akhirnya kill paksa tanpa kesempatan cleanup sama sekali.
- "Kenapa HTTP server dimatikan SEBELUM worker pool, bukan sebaliknya atau bersamaan?" — supaya gak ada request baru yang masuk dan butuh menaruh job ke worker pool yang sudah berhenti menerima job baru; urutan ini memastikan semua job yang perlu diproses sudah masuk ke pool sebelum pool itu sendiri mulai proses shutdown-nya.
- "Apa yang terjadi kalau `SIGTERM` diabaikan/gak ditangkap sama sekali?" — orchestrator (Kubernetes) akan menunggu selama `terminationGracePeriodSeconds`, lalu mengirim `SIGKILL` yang TIDAK bisa ditangkap/di-cleanup sama sekali — proses langsung mati, memutus semua request/job yang sedang berjalan tanpa kesempatan menyelesaikannya dengan rapi.

---

## 63. Testing Fundamentals (Unit vs Integration vs E2E)

### Apa itu?
Tiga level testing yang berbeda cakupannya: **unit test** menguji satu fungsi/unit kode terisolasi (tanpa dependency eksternal seperti database asli — dependency-nya di-mock/stub, dibahas di topik 64), **integration test** menguji interaksi antara komponen dengan dependency SUNGGUHAN (misalnya `CreateOrder`, phase 03, dites terhadap Postgres beneran, biasanya lewat test container), dan **end-to-end (E2E) test** menguji seluruh alur dari luar sistem (misalnya HTTP request ke endpoint `/orders` sampai response-nya), mensimulasikan cara user/client sungguhan memakai sistem.

### Kenapa dibutuhkan?
Ketiga level ini melengkapi satu sama lain, bukan saling menggantikan: unit test cepat dijalankan (milidetik) dan cocok buat menguji logika murni seperti `processOrder` (topik 55/60) atau `backoffDuration` (phase 05) secara mendetail dengan banyak variasi; tapi unit test gak akan menangkap masalah seperti query SQL yang salah syntax atau konfigurasi Kafka yang keliru — itu baru ketahuan lewat integration/E2E test yang jalan lebih lambat tapi lebih dekat ke kondisi production sungguhan.

### Cara Kerja
```
Testing pyramid (banyak-sedikit dari bawah ke atas):

         /\
        /E2E\        <- paling sedikit, paling lambat, paling dekat production
       /------\          (HTTP request penuh: POST /orders -> response)
      /Integr. \     <- sedang, pakai dependency sungguhan (test container Postgres/Kafka)
     /----------\        (CreateOrder terhadap database beneran)
    /   Unit     \   <- paling banyak, paling cepat, paling terisolasi
   /--------------\      (processOrder, backoffDuration, dst -- tanpa I/O)

Alasan bentuknya piramida: unit test murah (bisa punya ratusan/ribuan) dan
harus jadi lapisan pertama yang menangkap bug; E2E test mahal (setup penuh,
lambat) jadi jumlahnya sengaja dibatasi ke jalur-jalur kritis saja.
```

### Contoh Kode — Go
```go
package worker

import "testing"

// TestProcessOrder_ValidTotal adalah UNIT TEST -- menguji processOrder
// (topik 55/60) terisolasi, tanpa database/Kafka/HTTP sama sekali, jalan
// dalam hitungan mikrodetik.
func TestProcessOrder_ValidTotal(t *testing.T) {
	order := Order{ID: 1, UserID: 10, Total: 99.99}

	result := processOrder(order)

	if result.Err != nil {
		t.Fatalf("expected no error, got %v", result.Err)
	}
	if result.OrderID != order.ID {
		t.Errorf("expected OrderID %d, got %d", order.ID, result.OrderID)
	}
}

// TestProcessOrder_InvalidTotal adalah UNIT TEST lain untuk jalur error
// yang sama sekali gak butuh dependency eksternal.
func TestProcessOrder_InvalidTotal(t *testing.T) {
	order := Order{ID: 2, UserID: 10, Total: 0}

	result := processOrder(order)

	if result.Err == nil {
		t.Fatal("expected error for zero total, got nil")
	}
}
```

### Contoh Kode — Node.js
```javascript
// processOrder.test.js -- UNIT TEST pakai Jest/Vitest (API keduanya nyaris
// identik: describe/test/expect), menguji processOrder (topik 55/60) tanpa
// dependency eksternal sama sekali.
const { processOrder } = require('./worker-pool');

describe('processOrder', () => {
  test('returns no error for a valid total', () => {
    const order = { id: 1, userId: 10, total: 99.99 };

    const result = processOrder(order);

    expect(result.err).toBeNull();
    expect(result.orderId).toBe(order.id);
  });

  test('returns an error for a non-positive total', () => {
    const order = { id: 2, userId: 10, total: 0 };

    const result = processOrder(order);

    expect(result.err).not.toBeNull();
  });
});
```

### Trade-off & Pitfall
- Unit test yang men-mock TERLALU BANYAK (semua dependency di-mock, termasuk logic internal yang seharusnya ikut diuji) berakhir cuma menguji mock-nya sendiri, bukan logika sungguhan — mock secukupnya, cuma untuk dependency EKSTERNAL (database, network), bukan logika internal yang justru mau divalidasi.
- Integration/E2E test yang lambat (butuh spin up database/Kafka sungguhan) sering digabung jadi satu suite terpisah dari unit test (`go test -short` untuk skip yang lambat, atau tag terpisah di Jest) supaya feedback loop development sehari-hari (jalanin unit test) tetap cepat.
- Piramida testing gampang terbalik jadi "ice cream cone" (banyak E2E test lambat, sedikit unit test) kalau tim cuma fokus nulis test dari sudut pandang user tanpa disiplin menguji logika terisolasi lebih dulu — hasilnya suite test lambat dan rapuh (flaky karena banyak moving parts), sulit dipakai buat debug tepat di mana letak bug-nya.

### Kapan Dipakai
Unit test untuk semua logika murni yang punya banyak variasi kondisi (seperti `processOrder`); integration test untuk memvalidasi query/perintah terhadap dependency sungguhan (seperti `CreateOrder` terhadap Postgres); E2E test dibatasi ke jalur-jalur paling kritis (misalnya "customer bisa checkout dari awal sampai akhir"), karena mahal dijalankan dan mahal di-maintain.

### Sering Ditanya Saat Interview
- "Kenapa testing pyramid berbentuk piramida, bukan kotak/terbalik?" — unit test murah dan cepat sehingga bisa ditulis banyak-banyak untuk menutupi semua variasi kondisi logika; E2E test mahal (setup penuh, lambat, rapuh) jadi sengaja dibatasi jumlahnya ke jalur paling kritis saja, supaya keseluruhan suite tetap cepat dan reliable dijalankan sesering mungkin (tiap commit, misalnya).
- "Apa beda integration test dengan E2E test kalau keduanya sama-sama pakai dependency sungguhan?" — integration test menguji interaksi antar KOMPONEN tertentu (misalnya fungsi `CreateOrder` langsung terhadap database, dipanggil langsung dari kode test); E2E test menguji dari LUAR sistem sepenuhnya (HTTP request ke endpoint publik), mensimulasikan cara client sungguhan memakai sistem, biasanya melibatkan lebih banyak komponen sekaligus (routing, middleware, auth, dst).
- "Kapan sebaiknya nulis integration test alih-alih cukup unit test dengan mock?" — kalau logikanya justru soal INTERAKSI dengan dependency eksternal itu sendiri (misalnya apakah query SQL-nya benar, apakah transaction rollback bekerja sesuai harapan) — mock gak akan menangkap masalah ini karena mock cuma mensimulasikan perilaku yang KITA asumsikan benar, bukan perilaku database sungguhan.

---

## 64. Table-Driven Tests & Mocking

### Apa itu?
Table-driven test adalah pola menulis satu fungsi test yang menjalankan LOGIKA yang sama terhadap sekumpulan kasus (didefinisikan sebagai "tabel" data — biasanya slice/array of struct), alih-alih menulis fungsi test terpisah untuk tiap kasus. Mocking adalah teknik mengganti dependency asli (misalnya service downstream yang dipanggil setelah order diproses) dengan implementasi tiruan yang perilakunya bisa dikontrol penuh oleh test, supaya unit test-nya (topik 63) tetap terisolasi dari dependency eksternal.

### Kenapa dibutuhkan?
`processOrder` (topik 55/60) punya beberapa kondisi berbeda yang perlu diuji (total positif, total nol, total negatif, dst) — menulis fungsi test terpisah untuk tiap kondisi jadi berulang-ulang dan sulit di-maintain kalau ada kondisi baru. Table-driven test mengumpulkan semua kasus itu jadi satu tabel data yang jelas dibaca sebagai spesifikasi perilaku fungsi. Sementara itu, `ProcessOrdersWorkerPool` sering perlu memberi tahu service lain setelah order berhasil diproses — memanggil service sungguhan di unit test bikin test jadi lambat dan rapuh (bisa gagal karena network, bukan karena bug), jadi dependency itu di-mock.

### Cara Kerja
```
Struktur umum table-driven test di Go:

  tests := []struct {
      name string          // deskripsi kasus, muncul di output test kalau gagal
      input  InputType
      want   OutputType
  }{
      {"kasus 1", input1, expected1},
      {"kasus 2", input2, expected2},
      ...
  }

  for _, tt := range tests {
      t.Run(tt.name, func(t *testing.T) {
          got := FunctionUnderTest(tt.input)
          if got != tt.want {
              t.Errorf("got %v, want %v", got, tt.want)
          }
      })
  }

`t.Run` membuat SUBTEST bernama per kasus -- kalau satu kasus gagal, output
test menunjukkan PERSIS kasus mana yang gagal (misalnya
"TestProcessOrder/total_negatif"), bukan cuma "TestProcessOrder gagal".

Mocking: dependency eksternal (Notifier) diganti mockNotifier yang cuma
mencatat pemanggilan di memory, TANPA benar-benar memanggil service lain --
test jadi cepat, deterministik, dan bisa mengatur skenario gagal (notifier
error) yang susah dipicu kalau pakai service sungguhan.
```

### Contoh Kode — Go
```go
package worker

import (
	"context"
	"errors"
	"fmt"
	"testing"
)

// TestProcessOrder adalah TABLE-DRIVEN TEST untuk processOrder (topik
// 55/60) -- menguji job handling ProcessOrdersWorkerPool lewat fungsi
// internalnya, mencakup beberapa kondisi total order sekaligus dalam satu
// tabel.
func TestProcessOrder(t *testing.T) {
	tests := []struct {
		name    string
		order   Order
		wantErr bool
	}{
		{name: "total positif", order: Order{ID: 1, Total: 99.99}, wantErr: false},
		{name: "total nol", order: Order{ID: 2, Total: 0}, wantErr: true},
		{name: "total negatif", order: Order{ID: 3, Total: -10}, wantErr: true},
		{name: "total sangat kecil tapi positif", order: Order{ID: 4, Total: 0.01}, wantErr: false},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			result := processOrder(tt.order)

			gotErr := result.Err != nil
			if gotErr != tt.wantErr {
				t.Errorf("processOrder(%+v): error = %v, wantErr %v", tt.order, result.Err, tt.wantErr)
			}
			if result.OrderID != tt.order.ID {
				t.Errorf("processOrder(%+v): OrderID = %d, want %d", tt.order, result.OrderID, tt.order.ID)
			}
		})
	}
}

// Notifier adalah dependency eksternal (misalnya publish notifikasi ke
// service lain) -- di production diimplementasikan oleh sesuatu yang
// mengirim HTTP call sungguhan; di test diganti mock supaya gak benar-benar
// memanggil service eksternal.
type Notifier interface {
	NotifyProcessed(orderID int64) error
}

// ProcessOrderWithNotifier adalah versi processOrder yang juga memberi tahu
// Notifier setiap kali sebuah order BERHASIL diproses -- dipisah dari
// processOrder (topik 55/60) supaya topik itu tetap sederhana, dan supaya
// bisa didemonstrasikan bersama mocking di sini.
func ProcessOrderWithNotifier(order Order, notifier Notifier) Result {
	result := processOrder(order)
	if result.Err == nil {
		if err := notifier.NotifyProcessed(result.OrderID); err != nil {
			return Result{OrderID: order.ID, Err: fmt.Errorf("notify processed order %d: %w", order.ID, err)}
		}
	}
	return result
}

// mockNotifier adalah MOCK sederhana untuk Notifier -- mencatat pemanggilan
// tanpa efek samping sungguhan, dan bisa diatur untuk mengembalikan error
// tertentu supaya jalur error ProcessOrderWithNotifier juga bisa diuji.
type mockNotifier struct {
	notified []int64
	err      error
}

func (m *mockNotifier) NotifyProcessed(orderID int64) error {
	m.notified = append(m.notified, orderID)
	return m.err
}

// TestProcessOrderWithNotifier menggabungkan table-driven test dengan
// mocking: tiap kasus mengatur perilaku mockNotifier sendiri-sendiri.
func TestProcessOrderWithNotifier(t *testing.T) {
	tests := []struct {
		name         string
		order        Order
		notifierErr  error
		wantErr      bool
		wantNotified bool
	}{
		{name: "valid order, notifier sukses", order: Order{ID: 1, Total: 50}, notifierErr: nil, wantErr: false, wantNotified: true},
		{name: "invalid order, notifier TIDAK dipanggil", order: Order{ID: 2, Total: 0}, notifierErr: nil, wantErr: true, wantNotified: false},
		{name: "valid order, notifier gagal", order: Order{ID: 3, Total: 50}, notifierErr: errors.New("notify service down"), wantErr: true, wantNotified: true},
	}

	for _, tt := range tests {
		t.Run(tt.name, func(t *testing.T) {
			mock := &mockNotifier{err: tt.notifierErr}

			result := ProcessOrderWithNotifier(tt.order, mock)

			gotErr := result.Err != nil
			if gotErr != tt.wantErr {
				t.Errorf("ProcessOrderWithNotifier() error = %v, wantErr %v", result.Err, tt.wantErr)
			}
			gotNotified := len(mock.notified) > 0
			if gotNotified != tt.wantNotified {
				t.Errorf("notifier called = %v, want %v", gotNotified, tt.wantNotified)
			}
		})
	}
}

// TestProcessOrdersWorkerPool_DistributesAllJobs memverifikasi bahwa semua
// job yang dikirim ke channel jobs BENAR-BENAR menghasilkan satu Result di
// channel results, gak ada yang hilang -- menguji koordinasi banyak
// goroutine sekaligus di ProcessOrdersWorkerPool (topik 60).
func TestProcessOrdersWorkerPool_DistributesAllJobs(t *testing.T) {
	ctx := context.Background()
	jobs := make(chan Order, 3)
	jobs <- Order{ID: 1, Total: 10}
	jobs <- Order{ID: 2, Total: 20}
	jobs <- Order{ID: 3, Total: 0} // sengaja invalid, buat mastiin tetap terhitung sebagai Result (dengan Err)
	close(jobs)

	results := ProcessOrdersWorkerPool(ctx, jobs, 2)

	got := map[int64]bool{}
	for result := range results {
		got[result.OrderID] = true
	}

	for _, wantID := range []int64{1, 2, 3} {
		if !got[wantID] {
			t.Errorf("expected result for order %d, but none received", wantID)
		}
	}
}
```

### Contoh Kode — Node.js
```javascript
const { processOrder } = require('./worker-pool');

// processOrder.test.js -- TABLE-DRIVEN TEST pakai test.each (Jest/Vitest),
// padanan langsung dari pola table-driven Go.
describe('processOrder (table-driven / test.each)', () => {
  test.each([
    { name: 'total positif', order: { id: 1, total: 99.99 }, wantErr: false },
    { name: 'total nol', order: { id: 2, total: 0 }, wantErr: true },
    { name: 'total negatif', order: { id: 3, total: -10 }, wantErr: true },
  ])('$name', ({ order, wantErr }) => {
    const result = processOrder(order);
    expect(result.err !== null).toBe(wantErr);
    expect(result.orderId).toBe(order.id);
  });
});

// processOrderWithNotifier meniru ProcessOrderWithNotifier versi Go --
// dipisah dari processOrder supaya bisa didemonstrasikan bersama mocking.
async function processOrderWithNotifier(order, notifier) {
  const result = processOrder(order);
  if (result.err === null) {
    try {
      await notifier.notifyProcessed(result.orderId);
    } catch (err) {
      return { orderId: order.id, err };
    }
  }
  return result;
}

// TestProcessOrderWithNotifier menggabungkan test biasa dengan MOCKING
// lewat jest.fn() -- padanan mockNotifier versi Go, tanpa benar-benar
// memanggil service eksternal.
describe('processOrderWithNotifier (mocking notifier)', () => {
  test('calls notifier when order is valid', async () => {
    const notifier = { notifyProcessed: jest.fn().mockResolvedValue(undefined) };

    const result = await processOrderWithNotifier({ id: 1, total: 50 }, notifier);

    expect(result.err).toBeNull();
    expect(notifier.notifyProcessed).toHaveBeenCalledWith(1);
  });

  test('does not call notifier when order is invalid', async () => {
    const notifier = { notifyProcessed: jest.fn() };

    const result = await processOrderWithNotifier({ id: 2, total: 0 }, notifier);

    expect(result.err).not.toBeNull();
    expect(notifier.notifyProcessed).not.toHaveBeenCalled();
  });

  test('propagates notifier failure as an error', async () => {
    const notifier = { notifyProcessed: jest.fn().mockRejectedValue(new Error('notify service down')) };

    const result = await processOrderWithNotifier({ id: 3, total: 50 }, notifier);

    expect(result.err).not.toBeNull();
    expect(notifier.notifyProcessed).toHaveBeenCalledWith(3);
  });
});

module.exports = { processOrderWithNotifier };
```

### Trade-off & Pitfall
- Table-driven test yang tabelnya terlalu besar (puluhan kasus dalam satu fungsi test) bisa jadi sulit dibaca — kalau assertion tiap kasus mulai butuh logic berbeda-beda (bukan cuma bandingkan `want`), biasanya itu tanda tabelnya harus dipecah jadi beberapa fungsi test terpisah.
- Mock yang terlalu "pintar" (mensimulasikan logic bisnis notifier sungguhan, bukan cuma mencatat pemanggilan) berisiko drift dari perilaku service asli — kalau service asli berubah perilaku, mock yang gak ikut diupdate bikin test tetap hijau padahal integrasi sungguhan sudah rusak (makanya integration test, topik 63, tetap dibutuhkan sebagai pelengkap).
- `jest.fn().mockResolvedValue()`/`mockRejectedValue()` memudahkan simulasi skenario gagal yang susah dipicu di service sungguhan (misalnya "network down"), tapi kemudahan ini juga godaan untuk mem-mock TERLALU BANYAK skenario tanpa pernah memvalidasi lewat integration test bahwa asumsi mock-nya memang sesuai perilaku service asli.

### Kapan Dipakai
Table-driven test dipakai setiap kali sebuah fungsi punya lebih dari 2-3 variasi kondisi input/output yang perlu diuji — hampir selalu lebih baik daripada menulis fungsi test terpisah per kasus. Mocking dipakai setiap kali unit test butuh mengisolasi diri dari dependency eksternal (network, database, service lain) yang lambat, gak deterministik, atau sulit disetel ke skenario tertentu (seperti simulasi kegagalan).

### Sering Ditanya Saat Interview
- "Apa keuntungan table-driven test dibanding fungsi test terpisah per kasus?" — menghindari duplikasi kode test, menjadikan penambahan kasus baru semudah menambah satu baris di tabel, dan `t.Run`/`test.each` memberi output yang jelas menunjukkan kasus mana yang gagal tanpa harus baca stack trace satu-satu.
- "Kapan sebaiknya pakai mock, dan kapan sebaiknya jangan?" — mock dipakai untuk dependency EKSTERNAL yang lambat/gak deterministik (network call, service lain) supaya unit test tetap cepat dan stabil; jangan mock logika internal yang justru mau divalidasi (kalau semua di-mock, test cuma memvalidasi mock-nya sendiri, bukan kode sungguhan).
- "Apa resiko utama dari mocking?" — mock bisa "drift" dari perilaku service asli seiring waktu — test tetap hijau karena mock-nya gak ikut berubah, padahal integrasi sungguhan sudah rusak; ini kenapa mocking di unit test harus dilengkapi integration test (topik 63) yang memvalidasi terhadap dependency sungguhan.

---

**Selanjutnya:** [Phase 07 — Distributed Systems](./phase-07-distributed-systems.md)
