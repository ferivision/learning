# Phase 05 — Message Queue & Async Processing

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

## 47. Why Message Queue?

### Apa itu?
Message queue adalah komponen middleware yang memungkinkan satu service mengirim pesan (event) ke service lain secara **asinkron** — producer cukup taruh pesan di queue/topic dan lanjut kerja, tanpa perlu nunggu consumer selesai memprosesnya. Di OrderFlow, message queue dipakai supaya `CreateOrder` (phase 03) gak perlu nunggu email konfirmasi terkirim, invoice ter-generate, dan warehouse ter-notifikasi sebelum customer dapat response "order berhasil dibuat".

### Kenapa dibutuhkan?
Tanpa message queue, cara paling gampang buat "kasih tau service lain kalau ada order baru" adalah manggil service-service itu secara langsung (HTTP call) di dalam request `CreateOrder` yang sama. Masalahnya: kalau `email-service` lagi lambat atau down, response `CreateOrder` ke customer ikut lambat atau bahkan gagal — padahal proses kirim email itu sebenarnya gak perlu selesai dulu sebelum customer tau order-nya sukses dibuat. Message queue memutus (decouple) ketergantungan langsung ini.

### Cara Kerja
```
TANPA message queue (synchronous, coupled):
  POST /orders -> CreateOrder -> kirim email (tunggu) -> generate invoice (tunggu)
                                -> notify warehouse (tunggu) -> response ke customer
  Total latency = waktu CreateOrder + waktu SEMUA service downstream, berurutan.
  Kalau email-service down -> CreateOrder ikut gagal, padahal order sudah valid.

DENGAN message queue (asynchronous, decoupled):
  POST /orders -> CreateOrder -> commit ke database -> publish event "OrderCreated"
                                -> response ke customer (LANGSUNG, gak nunggu apa-apa lagi)

  Di belakang layar, terpisah:
  queue "order.created" -> consumer email-service   -> kirim email
                         -> consumer invoice-service -> generate invoice
                         -> consumer warehouse       -> notify warehouse
  Kalau salah satu consumer lambat/down, gak mempengaruhi response CreateOrder sama sekali.
```

### Contoh Kode — Go
```go
package order

import (
	"context"
	"fmt"
	"net/http"
)

// Order dipakai di sini sebagai referensi tipe yang sama dengan Order di
// phase 03 (package db), disederhanakan supaya contoh fokus ke masalahnya.
type Order struct {
	ID     int64
	UserID int64
	Total  float64
}

// NotifyDownstreamSync ANTI-CONTOH: memanggil semua service downstream
// (email, invoice, warehouse) secara SINKRON satu-satu, di dalam request
// yang sama dengan CreateOrder. Kalau salah satu service ini lambat/down,
// response CreateOrder ke customer ikut lambat/gagal juga -- padahal
// ketiganya gak perlu selesai dulu sebelum customer lihat "order berhasil".
func NotifyDownstreamSync(ctx context.Context, httpClient *http.Client, order *Order) error {
	if err := sendEmailConfirmation(ctx, httpClient, order); err != nil {
		return fmt.Errorf("send email: %w", err)
	}
	if err := generateInvoice(ctx, httpClient, order); err != nil {
		return fmt.Errorf("generate invoice: %w", err)
	}
	if err := notifyWarehouse(ctx, httpClient, order); err != nil {
		return fmt.Errorf("notify warehouse: %w", err)
	}
	return nil
}

func sendEmailConfirmation(ctx context.Context, c *http.Client, order *Order) error {
	req, err := http.NewRequestWithContext(ctx, http.MethodPost, "http://email-service/send", nil)
	if err != nil {
		return err
	}
	resp, err := c.Do(req)
	if err != nil {
		return fmt.Errorf("call email-service: %w", err)
	}
	defer resp.Body.Close()
	return nil
}

func generateInvoice(ctx context.Context, c *http.Client, order *Order) error {
	req, err := http.NewRequestWithContext(ctx, http.MethodPost, "http://invoice-service/generate", nil)
	if err != nil {
		return err
	}
	resp, err := c.Do(req)
	if err != nil {
		return fmt.Errorf("call invoice-service: %w", err)
	}
	defer resp.Body.Close()
	return nil
}

func notifyWarehouse(ctx context.Context, c *http.Client, order *Order) error {
	req, err := http.NewRequestWithContext(ctx, http.MethodPost, "http://warehouse-service/notify", nil)
	if err != nil {
		return err
	}
	resp, err := c.Do(req)
	if err != nil {
		return fmt.Errorf("call warehouse-service: %w", err)
	}
	defer resp.Body.Close()
	return nil
}
```

### Contoh Kode — Node.js
```javascript
const http = require('http');

// notifyDownstreamSync ANTI-CONTOH: memanggil semua service downstream
// secara SINKRON satu-satu di dalam request yang sama dengan createOrder.
async function notifyDownstreamSync(order) {
  await sendEmailConfirmation(order);
  await generateInvoice(order);
  await notifyWarehouse(order);
}

function sendEmailConfirmation(order) {
  return callService('email-service', '/send', order);
}

function generateInvoice(order) {
  return callService('invoice-service', '/generate', order);
}

function notifyWarehouse(order) {
  return callService('warehouse-service', '/notify', order);
}

function callService(host, path, order) {
  return new Promise((resolve, reject) => {
    const req = http.request(
      { host, path, method: 'POST', headers: { 'Content-Type': 'application/json' } },
      (res) => {
        res.on('data', () => {});
        res.on('end', resolve);
      }
    );
    req.on('error', (err) => reject(new Error(`call ${host}: ${err.message}`)));
    req.end(JSON.stringify(order));
  });
}

module.exports = { notifyDownstreamSync };
```

### Trade-off & Pitfall
- Message queue nambah komponen infrastruktur baru (Kafka/RabbitMQ) yang harus di-deploy, dimonitor, dan di-maintain — untuk sistem kecil dengan satu-dua downstream call yang cepat dan reliable, overhead ini kadang belum sepadan.
- Async berarti customer gak langsung tau kalau ternyata email gagal terkirim atau invoice gagal di-generate — perlu observability terpisah (dashboard, alert) buat mendeteksi kegagalan yang "tersembunyi" di belakang layar.
- Decoupling lewat queue mengubah model mental development: debugging jadi lebih susah karena flow-nya gak lagi satu call stack lurus, melainkan tersebar ke banyak consumer independen yang jalan di waktu berbeda-beda.

### Kapan Dipakai
Ketika ada pekerjaan downstream yang **gak perlu selesai dulu** sebelum request utama dianggap sukses — seperti kirim email, generate invoice, atau notifikasi warehouse setelah `CreateOrder`. Kalau downstream itu justru harus selesai dulu (misalnya validasi stock sebelum order dibuat), itu tetap harus synchronous, bukan lewat queue.

### Sering Ditanya Saat Interview
- "Kenapa gak panggil langsung service downstream lewat HTTP aja dari `CreateOrder`?" — karena itu bikin `CreateOrder` ikut lambat/gagal kalau salah satu downstream bermasalah, padahal downstream itu (kirim email, dst) gak perlu selesai dulu sebelum customer tau order-nya sukses.
- "Apa downside utama pakai message queue?" — komponen infrastruktur tambahan yang perlu di-maintain, dan debugging jadi lebih susah karena flow-nya tersebar ke banyak consumer independen, bukan satu call stack yang lurus.
- "Kapan sebaiknya TIDAK pakai message queue?" — kalau downstream call itu harus selesai dulu sebelum request dianggap valid (misalnya cek stock), atau kalau sistemnya kecil dan downstream call-nya cepat/reliable sehingga overhead queue belum sepadan.

---

## 48. Producer / Consumer

### Apa itu?
Producer adalah komponen yang mempublish (mengirim) pesan ke message queue/topic, dan consumer adalah komponen yang membaca dan memproses pesan itu. Di OrderFlow, `PublishOrderCreated` adalah producer yang mempublish event `OrderCreated` ke topic `order.created` tepat setelah `CreateOrder` (phase 03) sukses, dan `ConsumeOrderEvents` adalah consumer generik yang membaca event dari topic itu lalu menjalankan sebuah `handler`.

### Kenapa dibutuhkan?
`CreateOrder` cuma tau cara menyimpan order ke database — dia gak (dan seharusnya gak) tau ada berapa banyak service lain yang perlu bereaksi terhadap order baru itu. Dengan pola producer/consumer, `CreateOrder` cukup publish satu event, dan sebanyak apapun consumer (email-service, invoice-service, warehouse-service) bisa berlangganan topic yang sama secara independen, tanpa `CreateOrder` perlu tau atau berubah sama sekali.

### Cara Kerja
```
CreateOrder (phase 03) sukses commit ke database
        |
        v
PublishOrderCreated -> topic "order.created" (Kafka)
        |
        +---> Consumer group "email-service"     -> ConsumeOrderEvents(handler: kirim email)
        +---> Consumer group "invoice-service"    -> ConsumeOrderEvents(handler: generate invoice)
        +---> Consumer group "warehouse-service"  -> ConsumeOrderEvents(handler: notify warehouse)

Tiap consumer group membaca topic yang sama secara independen -- consumer
group berbeda gak saling mempengaruhi (masing-masing punya offset sendiri).
```

### Contoh Kode — Go
```go
package messaging

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"github.com/google/uuid"
	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/segmentio/kafka-go"

	orderdb "orderflow/internal/db"
)

// Order adalah alias untuk tipe Order yang sama dengan phase 03 (package db).
type Order = orderdb.Order

// OrderEvent merepresentasikan payload event yang dipublish tiap kali
// CreateOrder berhasil.
type OrderEvent struct {
	EventID   string    `json:"event_id"`
	OrderID   int64     `json:"order_id"`
	UserID    int64     `json:"user_id"`
	Total     float64   `json:"total"`
	CreatedAt time.Time `json:"created_at"`
}

// PublishOrderCreated mempublish event OrderCreated ke topic order.created,
// dipanggil tepat setelah CreateOrder (phase 03) sukses commit.
func PublishOrderCreated(ctx context.Context, w *kafka.Writer, order *Order) error {
	event := OrderEvent{
		EventID:   uuid.NewString(),
		OrderID:   order.ID,
		UserID:    order.UserID,
		Total:     order.Total,
		CreatedAt: time.Now().UTC(),
	}
	payload, err := json.Marshal(event)
	if err != nil {
		return fmt.Errorf("marshal order event: %w", err)
	}
	return w.WriteMessages(ctx, kafka.Message{
		Key:   []byte(fmt.Sprintf("order-%d", order.ID)),
		Value: payload,
	})
}

// CreateOrderAndPublish memanggil CreateOrder (phase 03) lalu langsung
// mempublish event OrderCreated setelah order tersimpan.
func CreateOrderAndPublish(ctx context.Context, pool *pgxpool.Pool, w *kafka.Writer, userID int64, items []orderdb.OrderItem) (*orderdb.Order, error) {
	order, err := orderdb.CreateOrder(ctx, pool, userID, items)
	if err != nil {
		return nil, fmt.Errorf("create order: %w", err)
	}
	if err := PublishOrderCreated(ctx, w, order); err != nil {
		// order sudah tersimpan di database, tapi event gagal dipublish --
		// ini gak boleh bikin CreateOrder dianggap gagal (order tetap valid),
		// cukup dikembalikan errornya buat di-log/di-retry manual (outbox
		// pattern lebih robust untuk ini, di luar cakupan phase ini).
		return order, fmt.Errorf("order created but publish failed: %w", err)
	}
	return order, nil
}

// ConsumeOrderEvents menjalankan consumer loop untuk topic order.created.
// Pakai FetchMessage (BUKAN ReadMessage) supaya offset baru di-commit
// SETELAH handler sukses -- ReadMessage auto-commit begitu pesan dibaca,
// yang berarti pesan bisa hilang kalau proses crash di tengah handler
// (dibahas lebih detail di topik 49: At-Least-Once Delivery).
func ConsumeOrderEvents(ctx context.Context, r *kafka.Reader, handler func(OrderEvent) error) error {
	for {
		msg, err := r.FetchMessage(ctx)
		if err != nil {
			return fmt.Errorf("fetch message: %w", err)
		}

		var event OrderEvent
		if err := json.Unmarshal(msg.Value, &event); err != nil {
			return fmt.Errorf("unmarshal event: %w", err)
		}

		if err := handler(event); err != nil {
			return fmt.Errorf("handle order event %s: %w", event.EventID, err)
		}

		if err := r.CommitMessages(ctx, msg); err != nil {
			return fmt.Errorf("commit message: %w", err)
		}
	}
}
```

### Contoh Kode — Node.js
```javascript
const crypto = require('crypto');
const { createOrder } = require('./order-repository'); // dari phase 03

// publishOrderCreated mempublish event OrderCreated ke topic order.created,
// dipanggil tepat setelah createOrder (phase 03) sukses.
async function publishOrderCreated(producer, order) {
  const event = {
    eventId: crypto.randomUUID(),
    orderId: order.id,
    // createOrder (phase 03) mengembalikan raw row dari
    // "RETURNING id, user_id, status, total, version" tanpa aliasing, jadi
    // key-nya snake_case (order.user_id), bukan order.userId.
    userId: order.user_id,
    total: order.total,
    createdAt: new Date().toISOString(),
  };
  await producer.send({
    topic: 'order.created',
    messages: [
      {
        key: `order-${order.id}`,
        value: JSON.stringify(event),
      },
    ],
  });
}

// createOrderAndPublish memanggil createOrder (phase 03) lalu langsung
// mempublish event OrderCreated setelah order tersimpan.
async function createOrderAndPublish(pool, producer, userId, items) {
  const order = await createOrder(pool, userId, items);
  try {
    await publishOrderCreated(producer, order);
  } catch (err) {
    // order sudah tersimpan, event gagal dipublish -- jangan gagalkan
    // createOrder, cukup log supaya bisa diselidiki/di-retry manual.
    console.error(`failed to publish OrderCreated for order ${order.id}:`, err);
  }
  return order;
}

// consumeOrderEvents menjalankan consumer loop untuk topic order.created --
// handler dipanggil per event, offset di-commit MANUAL cuma kalau handler
// sukses (at-least-once, dibahas di topik 49).
async function consumeOrderEvents(consumer, handler) {
  await consumer.subscribe({ topic: 'order.created', fromBeginning: false });
  await consumer.run({
    autoCommit: false,
    eachMessage: async ({ topic, partition, message }) => {
      const event = JSON.parse(message.value.toString());
      await handler(event);
      await consumer.commitOffsets([
        { topic, partition, offset: (Number(message.offset) + 1).toString() },
      ]);
    },
  });
}

module.exports = { publishOrderCreated, createOrderAndPublish, consumeOrderEvents };
```

### Trade-off & Pitfall
- `PublishOrderCreated` dipanggil setelah `CreateOrder` commit, tapi keduanya bukan satu operasi atomik — kalau proses crash tepat di antara commit database dan publish event, order tersimpan tapi event-nya gak pernah terkirim (dibahas mitigasinya lewat outbox pattern, di luar cakupan phase ini).
- Consumer group yang beda-beda (email-service, invoice-service, warehouse-service) membaca topic yang sama secara independen — kalau satu consumer group lambat/down, itu gak menghambat consumer group lain, tapi juga berarti tiap consumer group butuh dipantau kesehatannya masing-masing.
- `handler func(OrderEvent) error` di `ConsumeOrderEvents` dipanggil secara sekuensial per partition — kalau handler-nya lambat (misalnya HTTP call ke service lain), throughput consumer buat partition itu ikut terbatas oleh kecepatan handler.

### Kapan Dipakai
Setiap kali sebuah event (seperti "order baru dibuat") perlu diketahui oleh lebih dari satu consumer independen, dan consumer-consumer itu gak perlu tau satu sama lain — pola producer/consumer memungkinkan menambah consumer baru di masa depan (misalnya "loyalty-service") tanpa mengubah kode `CreateOrder` atau `PublishOrderCreated` sama sekali.

### Sering Ditanya Saat Interview
- "Kenapa `PublishOrderCreated` dipanggil setelah `CreateOrder`, bukan di dalam transaction yang sama?" — publish ke Kafka bukan operasi database, jadi gak bisa ikut di-rollback lewat transaction Postgres; keduanya sengaja dipisah, dengan trade-off ada kemungkinan kecil order tersimpan tapi event gagal terkirim kalau crash di antara keduanya.
- "Apa beda `ReadMessage` dan `FetchMessage` di kafka-go?" — `ReadMessage` membaca sekaligus auto-commit offset-nya, sementara `FetchMessage` cuma membaca tanpa commit, sehingga aplikasi bisa memutuskan sendiri kapan commit lewat `CommitMessages` (biasanya setelah handler sukses).
- "Kenapa `ConsumeOrderEvents` menerima `handler` sebagai parameter, bukan logic proses-nya ditulis langsung di dalam fungsi itu?" — supaya loop consume (fetch, unmarshal, commit) bisa dipakai ulang oleh banyak consumer group dengan logic proses yang berbeda-beda (kirim email, generate invoice, dst), tanpa duplikasi kode loop-nya.

---

## 49. At-Least-Once Delivery

### Apa itu?
At-least-once delivery berarti sebuah message dijamin sampai ke consumer **minimal satu kali** — tapi bisa juga lebih dari sekali (duplikat), tergantung kapan offset di-commit relatif terhadap kapan message diproses. Ini beda dengan exactly-once (tiap message diproses persis sekali, sulit dijamin sepenuhnya di sistem terdistribusi) dan at-most-once (message bisa hilang, tapi gak pernah diproses dua kali).

### Kenapa dibutuhkan?
`ConsumeOrderEvents` (topik 48) sengaja commit offset **setelah** handler sukses, bukan sebelum. Kalau proses consumer crash di tengah handler (misalnya setelah berhasil kirim email tapi sebelum sempat commit offset), Kafka akan mengira message itu belum pernah dibaca dan mengirimkannya lagi ke consumer berikutnya saat restart. Itu artinya email bisa saja terkirim dua kali. Trade-off ini sengaja diambil karena kehilangan event (`OrderCreated` gak pernah diproses sama sekali) jauh lebih berbahaya buat bisnis dibanding event diproses dua kali — asalkan consumer-nya idempotent (topik 50).

### Cara Kerja
```
Commit SEBELUM proses (anti-pattern, resiko at-most-once):
  baca message -> commit offset -> proses handler
  kalau crash di antara commit dan proses selesai -> message DIANGGAP SUDAH DIBACA,
  gak akan pernah dikirim ulang -> handler-nya gak pernah kelar -> MESSAGE HILANG

Commit SETELAH proses (yang benar, at-least-once):
  baca message -> proses handler -> commit offset
  kalau crash SEBELUM commit (misalnya proses handler-nya sendiri crash di tengah,
  atau berhasil tapi crash sebelum sempat commit) -> offset belum maju ->
  Kafka kirim ulang message yang sama -> handler dipanggil LAGI -> DUPLIKAT,
  tapi TIDAK PERNAH HILANG
```

### Contoh Kode — Go
```go
package messaging

import (
	"encoding/json"
	"fmt"

	"github.com/segmentio/kafka-go"
)

// ConsumeOrderEventsAutoCommitAntiPattern ANTI-CONTOH: ReadMessage auto-commit
// offset begitu message selesai DIBACA, sebelum handler sempat dijalankan.
// Kalau proses crash tepat setelah ReadMessage tapi sebelum handler selesai,
// message itu dianggap Kafka sudah "beres" dan TIDAK akan pernah dikirim
// ulang -- event OrderCreated itu hilang, bukan cuma telat.
func ConsumeOrderEventsAutoCommitAntiPattern(r *kafka.Reader, handler func(OrderEvent) error) error {
	for {
		msg, err := r.ReadMessage(nil) // ReadMessage auto-commit offset saat dibaca
		if err != nil {
			return fmt.Errorf("read message: %w", err)
		}

		var event OrderEvent
		if err := json.Unmarshal(msg.Value, &event); err != nil {
			return fmt.Errorf("unmarshal event: %w", err)
		}

		if err := handler(event); err != nil {
			// offset SUDAH ke-commit oleh ReadMessage sebelum baris ini --
			// event ini TIDAK akan pernah di-retry walau handler-nya gagal.
			fmt.Printf("handler failed for event %s (message lost): %v\n", event.EventID, err)
			continue
		}
	}
}

// ConsumeOrderEvents (topik 48) adalah versi yang benar: commit lewat
// CommitMessages HANYA dipanggil setelah handler sukses, sehingga kalau
// terjadi crash sebelum commit, Kafka akan mengirim ulang message yang sama
// -- itulah yang membuatnya at-least-once, bukan at-most-once.
```

### Contoh Kode — Node.js
```javascript
// consumeOrderEventsCommitFirstAntiPattern ANTI-CONTOH: commit offset SEBELUM
// handler dijalankan -- kalau proses crash di tengah handler (setelah commit,
// sebelum efek sampingnya selesai, misalnya kirim email), event itu dianggap
// "sudah diproses" oleh Kafka padahal belum -> berpotensi HILANG
// (at-most-once), bukan at-least-once seperti consumeOrderEvents (topik 48).
async function consumeOrderEventsCommitFirstAntiPattern(consumer, handler) {
  await consumer.subscribe({ topic: 'order.created', fromBeginning: false });
  await consumer.run({
    autoCommit: false,
    eachMessage: async ({ topic, partition, message }) => {
      await consumer.commitOffsets([
        { topic, partition, offset: (Number(message.offset) + 1).toString() },
      ]);
      const event = JSON.parse(message.value.toString());
      await handler(event); // kalau ini gagal/crash, offset SUDAH ke-commit di atas
    },
  });
}

// consumeOrderEvents (topik 48) adalah versi yang benar: commitOffsets HANYA
// dipanggil setelah handler(event) sukses -- itulah yang membuatnya
// at-least-once (message di-redeliver kalau gagal), bukan at-most-once.

module.exports = { consumeOrderEventsCommitFirstAntiPattern };
```

### Trade-off & Pitfall
- At-least-once berarti consumer HARUS siap menerima message yang sama lebih dari sekali — kalau handler-nya gak idempotent (misalnya "kirim email" tanpa pengecekan apapun), customer bisa dapat email konfirmasi dobel setiap kali ada redelivery.
- Redelivery gak cuma terjadi karena crash — rebalance consumer group (misalnya instance baru join/leave), network blip, atau restart deployment juga bisa memicu message yang sama dibaca ulang oleh consumer yang berbeda.
- Trade-off at-least-once vs at-most-once itu soal risiko mana yang lebih bisa diterima bisnis: OrderFlow memilih at-least-once (dengan idempotent consumer di topik 50) karena kehilangan event `OrderCreated` sama sekali jauh lebih parah dibanding email terkirim dua kali.

### Kapan Dipakai
Nyaris selalu, untuk event-event penting seperti `OrderCreated` yang gak boleh sampai hilang — dikombinasikan dengan idempotent consumer (topik 50) supaya redelivery yang duplikat gak menyebabkan efek samping ganda (kirim email dobel, generate invoice dobel, dst).

### Sering Ditanya Saat Interview
- "Kenapa OrderFlow milih at-least-once, bukan at-most-once?" — karena kehilangan event `OrderCreated` sepenuhnya (customer gak pernah dapat email konfirmasi sama sekali) jauh lebih buruk buat bisnis dibanding email terkirim dua kali, yang bisa dicegah dengan idempotent consumer.
- "Apa yang bikin sebuah sistem messaging jadi at-least-once, bukan at-most-once?" — urutan antara memproses message dan commit offset: kalau commit terjadi SETELAH proses berhasil, message akan di-redeliver kalau proses gagal/crash (at-least-once); kalau commit terjadi SEBELUM proses, message dianggap beres walau prosesnya belum tentu sukses (at-most-once, resiko hilang).
- "Apa saja penyebab redelivery selain proses crash?" — consumer group rebalance (instance baru join/leave group), network timeout saat commit offset gagal terkirim ke broker, atau restart deployment yang terjadi di tengah pemrosesan sebuah batch message.

---

## 50. Idempotent Consumer

### Apa itu?
Idempotent consumer adalah consumer yang aman diproses berkali-kali dengan message/event yang sama persis, tanpa menghasilkan efek samping ganda. Di OrderFlow, `IsEventProcessed` adalah fungsi yang mengecek (sekaligus menandai) apakah sebuah `eventID` sudah pernah diproses sebelumnya, dipakai untuk membuat consumer `OrderCreated` jadi idempotent walau topic-nya sendiri cuma menjamin at-least-once (topik 49).

### Kenapa dibutuhkan?
Karena `ConsumeOrderEvents` (topik 48) menjamin at-least-once, event `OrderCreated` yang sama bisa saja diterima dua kali oleh consumer (misalnya karena proses crash tepat sebelum commit offset). Kalau handler-nya langsung kirim email tanpa pengecekan apapun, customer akan dapat dua email konfirmasi untuk satu order yang sama. `IsEventProcessed` mencegah ini dengan menandai `eventID` yang sudah diproses di Redis, supaya redelivery yang sama persis dilewati (skip), bukan diproses ulang.

### Cara Kerja
```
Event OrderCreated (eventID = "evt-123") diterima consumer pertama kali:
  IsEventProcessed(rdb, "evt-123") -> SET key NX -> BERHASIL di-set -> return false (belum diproses)
  -> handler dijalankan -> kirim email konfirmasi

Kafka redeliver event yang SAMA (eventID = "evt-123") karena crash sebelum commit:
  IsEventProcessed(rdb, "evt-123") -> SET key NX -> GAGAL (key sudah ada) -> return true (sudah diproses)
  -> handler DI-SKIP, email TIDAK dikirim lagi

SET ... NX bersifat atomic di Redis -- penting kalau ada lebih dari satu
consumer instance yang menerima eventID yang sama nyaris bersamaan, supaya
gak ada race condition di mana keduanya sama-sama lolos pengecekan.
```

### Contoh Kode — Go
```go
package messaging

import (
	"context"
	"fmt"
	"time"

	"github.com/redis/go-redis/v9"
)

// IsEventProcessed mengecek SEKALIGUS menandai apakah eventID sudah pernah
// diproses, pakai SET ... NX supaya check-and-mark jadi satu operasi atomic
// -- ini penting supaya dua consumer instance yang menerima eventID yang
// sama nyaris bersamaan gak sama-sama lolos pengecekan (race condition),
// yang bisa menyebabkan event diproses dua kali walau IsEventProcessed ada.
func IsEventProcessed(ctx context.Context, rdb *redis.Client, eventID string) (bool, error) {
	key := fmt.Sprintf("processed_event:%s", eventID)
	// SetNX/SET-NX return true kalau key BERHASIL di-set (berarti event ini
	// BELUM pernah ditandai diproses sebelumnya).
	setOK, err := rdb.SetNX(ctx, key, 1, 24*time.Hour).Result()
	if err != nil {
		return false, fmt.Errorf("check event processed %s: %w", eventID, err)
	}
	// setOK == true  -> key baru di-set sekarang -> event BELUM diproses -> return false
	// setOK == false -> key sudah ada sebelumnya -> event SUDAH diproses  -> return true
	return !setOK, nil
}

// HandleOrderCreatedIdempotent membungkus handler asli dengan pengecekan
// IsEventProcessed -- hasilnya dipakai sebagai `handler` yang dioper ke
// ConsumeOrderEvents (topik 48), supaya event yang di-redeliver Kafka
// (at-least-once, topik 49) gak diproses dua kali oleh next().
func HandleOrderCreatedIdempotent(ctx context.Context, rdb *redis.Client, next func(OrderEvent) error) func(OrderEvent) error {
	return func(event OrderEvent) error {
		processed, err := IsEventProcessed(ctx, rdb, event.EventID)
		if err != nil {
			return fmt.Errorf("check idempotency for event %s: %w", event.EventID, err)
		}
		if processed {
			// event ini sudah pernah ditandai diproses -- skip, jangan
			// panggil next() lagi supaya gak double-processing.
			return nil
		}
		return next(event)
	}
}
```

### Contoh Kode — Node.js
```javascript
const PROCESSED_EVENT_TTL_SECONDS = 24 * 60 * 60;

// isEventProcessed mengecek SEKALIGUS menandai apakah eventId sudah pernah
// diproses, pakai SET ... NX supaya check-and-mark jadi satu operasi atomic.
async function isEventProcessed(redisClient, eventId) {
  const key = `processed_event:${eventId}`;
  // set(key, value, 'EX', ttl, 'NX') return 'OK' kalau berhasil di-set
  // (event belum pernah diproses), atau null kalau key sudah ada
  // (event sudah pernah diproses) -- atomic check-and-mark.
  const result = await redisClient.set(key, '1', 'EX', PROCESSED_EVENT_TTL_SECONDS, 'NX');
  return result !== 'OK'; // true = sudah pernah diproses sebelumnya
}

// handleOrderCreatedIdempotent membungkus handler asli dengan pengecekan
// isEventProcessed -- hasilnya dipakai sebagai handler yang dioper ke
// consumeOrderEvents (topik 48), supaya event yang di-redeliver gak
// diproses dua kali oleh next().
function handleOrderCreatedIdempotent(redisClient, next) {
  return async (event) => {
    const alreadyProcessed = await isEventProcessed(redisClient, event.eventId);
    if (alreadyProcessed) {
      return; // sudah pernah ditandai diproses -- skip
    }
    await next(event);
  };
}

module.exports = { isEventProcessed, handleOrderCreatedIdempotent };
```

### Trade-off & Pitfall
- `IsEventProcessed` menandai event sebagai "processed" **sebelum** `next(event)` (proses sebenarnya, misalnya kirim email) benar-benar dipastikan sukses — kalau `next(event)` gagal setelah `IsEventProcessed` sudah men-set key di Redis, event itu dianggap sudah diproses dan TIDAK akan di-retry lagi walau Kafka redeliver event yang sama. Ini trade-off sengaja: mencegah race condition antar consumer instance (yang lebih berbahaya, bisa bikin double-send) dengan konsekuensi butuh alerting/replay manual kalau `next(event)` gagal.
- TTL 24 jam pada key Redis bukan angka sembarangan — kalau TTL terlalu pendek, event yang di-redeliver lewat dari TTL itu (jarang, tapi bisa terjadi kalau consumer down lama) akan dianggap "belum pernah diproses" lagi dan diproses ulang; kalau TTL terlalu panjang, key numpuk terus di Redis untuk event yang gak akan pernah di-redeliver lagi.
- Idempotency ini cuma melindungi level "consumer OrderFlow ini" — kalau `next(event)` sendiri memanggil service eksternal yang gak idempotent (misalnya payment gateway tanpa idempotency key sendiri), duplikasi tetap bisa terjadi di level service eksternal itu.

### Kapan Dipakai
Wajib dipasang di semua consumer yang membaca dari topic dengan jaminan at-least-once (topik 49) dan efek sampingnya gak aman dijalankan dua kali — seperti kirim email konfirmasi, generate invoice, atau charge payment. Kalau efek sampingnya sendiri sudah idempotent secara alami (misalnya cuma `UPDATE ... SET status = 'notified'` yang hasilnya sama walau dijalankan berkali-kali), `IsEventProcessed` jadi opsional, walau tetap disarankan buat menghindari kerja berulang yang gak perlu.

### Sering Ditanya Saat Interview
- "Kenapa `IsEventProcessed` pakai `SET ... NX`, bukan `GET` dulu baru `SET` belakangan?" — `GET` lalu `SET` terpisah punya race window: dua consumer instance bisa sama-sama `GET` dan lihat key belum ada, lalu sama-sama lanjut memproses event yang sama sebelum salah satunya sempat `SET`. `SET ... NX` menyatukan cek-dan-tandai jadi satu operasi atomic di Redis, jadi cuma satu instance yang bisa "menang".
- "Apa yang terjadi kalau proses gagal SETELAH `IsEventProcessed` menandainya sebagai processed?" — event itu gak akan di-retry otomatis lewat redelivery Kafka, karena sudah dianggap processed; ini trade-off yang diterima demi mencegah race condition antar consumer instance, dengan konsekuensi butuh mekanisme alerting/replay manual terpisah untuk kegagalan seperti ini.
- "Kenapa key di Redis dikasih TTL, bukan disimpan selamanya?" — supaya Redis gak numpuk data event yang gak akan pernah di-redeliver lagi (setelah beberapa waktu, kemungkinan redelivery sudah sangat kecil); TTL dipilih cukup panjang untuk menutupi skenario redelivery yang realistis (misalnya consumer down beberapa jam).

---

## 51. Retry (Queue)

### Apa itu?
Retry queue adalah topic terpisah yang dipakai buat mencoba ulang event yang gagal diproses, dengan jeda (backoff) sebelum dicoba lagi — alih-alih retry langsung di tempat (blocking) pada topic utama. Di OrderFlow, event `OrderCreated` yang gagal diproses handler-nya dipublish ulang ke topic `order.created.retry` dengan informasi jumlah percobaan (`retry-count`) dan waktu boleh dicoba lagi (`retry-after-unix`).

### Kenapa dibutuhkan?
Kalau handler gagal (misalnya `email-service` lagi down) dan consumer langsung retry di tempat sebelum lanjut ke message berikutnya di partition yang sama, semua event `OrderCreated` lain yang antre di belakangnya ikut tertahan (head-of-line blocking) — padahal event-event itu gak ada hubungannya dengan kegagalan tadi. Dengan memindahkan event yang gagal ke topic retry terpisah, consumer utama bisa langsung lanjut memproses message berikutnya, sementara retry-nya diproses di jalur lain dengan backoff yang sesuai.

### Cara Kerja
```
handler(event) gagal di consumer utama (topic order.created)
        |
        v
PublishToRetryQueue -> topic "order.created.retry"
        header retry-count = 1, retry-after-unix = now + 2s

ConsumeRetryQueue baca dari topic retry:
  - tunggu sampai waktu retry-after-unix lewat (backoff)
  - coba proses lagi lewat handler
      - berhasil -> selesai
      - gagal & attempt < maxRetries -> publish ulang ke retry queue, attempt+1,
                                          backoff makin panjang (2s, 4s, 8s, 16s, 32s)
      - gagal & attempt >= maxRetries -> kirim ke Dead Letter Queue (topik 52)
```

### Contoh Kode — Go
```go
package messaging

import (
	"context"
	"encoding/json"
	"fmt"
	"strconv"
	"time"

	"github.com/segmentio/kafka-go"
)

const (
	headerRetryCount = "retry-count"
	headerRetryAfter = "retry-after-unix"
	maxRetries       = 5
)

// backoffDuration menghasilkan exponential backoff: 2s, 4s, 8s, 16s, 32s
// untuk attempt 1..5.
func backoffDuration(attempt int) time.Duration {
	return time.Duration(1<<uint(attempt)) * time.Second
}

// PublishToRetryQueue mempublish ulang event yang gagal diproses ke topic
// retry terpisah (order.created.retry) -- BUKAN retry langsung di tempat --
// supaya event lain di partition topic utama gak ikut head-of-line-blocking
// nunggu backoff event yang gagal ini.
func PublishToRetryQueue(ctx context.Context, w *kafka.Writer, event OrderEvent, attempt int) error {
	payload, err := json.Marshal(event)
	if err != nil {
		return fmt.Errorf("marshal retry event: %w", err)
	}
	retryAfter := time.Now().Add(backoffDuration(attempt)).Unix()
	return w.WriteMessages(ctx, kafka.Message{
		Key:   []byte(fmt.Sprintf("order-%d", event.OrderID)),
		Value: payload,
		Headers: []kafka.Header{
			{Key: headerRetryCount, Value: []byte(strconv.Itoa(attempt))},
			{Key: headerRetryAfter, Value: []byte(strconv.FormatInt(retryAfter, 10))},
		},
	})
}

// ConsumeRetryQueue memproses topic retry: nunggu sampai waktu backoff-nya
// lewat (header retry-after-unix), lalu coba proses ulang lewat handler --
// kalau masih gagal dan attempt sudah mentok maxRetries, event dikirim ke
// dead letter queue (topik 52).
func ConsumeRetryQueue(ctx context.Context, r *kafka.Reader, retryWriter, dlqWriter *kafka.Writer, handler func(OrderEvent) error) error {
	for {
		msg, err := r.FetchMessage(ctx)
		if err != nil {
			return fmt.Errorf("fetch retry message: %w", err)
		}

		attempt, retryAfter := parseRetryHeaders(msg.Headers)
		if wait := time.Until(time.Unix(retryAfter, 0)); wait > 0 {
			time.Sleep(wait)
		}

		var event OrderEvent
		if err := json.Unmarshal(msg.Value, &event); err != nil {
			return fmt.Errorf("unmarshal retry event: %w", err)
		}

		if err := handler(event); err != nil {
			if attempt >= maxRetries {
				if dlqErr := PublishToDeadLetterQueue(ctx, dlqWriter, event, err); dlqErr != nil {
					return fmt.Errorf("publish to dlq: %w", dlqErr)
				}
			} else if pubErr := PublishToRetryQueue(ctx, retryWriter, event, attempt+1); pubErr != nil {
				return fmt.Errorf("publish to retry queue: %w", pubErr)
			}
		}

		if err := r.CommitMessages(ctx, msg); err != nil {
			return fmt.Errorf("commit retry message: %w", err)
		}
	}
}

func parseRetryHeaders(headers []kafka.Header) (attempt int, retryAfter int64) {
	for _, h := range headers {
		switch h.Key {
		case headerRetryCount:
			attempt, _ = strconv.Atoi(string(h.Value))
		case headerRetryAfter:
			retryAfter, _ = strconv.ParseInt(string(h.Value), 10, 64)
		}
	}
	return attempt, retryAfter
}
```

### Contoh Kode — Node.js
```javascript
const RETRY_HEADER_COUNT = 'retry-count';
const RETRY_HEADER_AFTER = 'retry-after-unix';
const MAX_RETRIES = 5;

// backoffMs menghasilkan exponential backoff: 2s, 4s, 8s, 16s, 32s untuk
// attempt 1..5.
function backoffMs(attempt) {
  return 2 ** attempt * 1000;
}

// publishToRetryQueue mempublish ulang event yang gagal diproses ke topic
// retry terpisah (order.created.retry), supaya event lain di topic utama
// gak ikut head-of-line-blocking nunggu backoff event yang gagal ini.
async function publishToRetryQueue(producer, event, attempt) {
  const retryAfter = Date.now() + backoffMs(attempt);
  await producer.send({
    topic: 'order.created.retry',
    messages: [
      {
        key: `order-${event.orderId}`,
        value: JSON.stringify(event),
        headers: {
          [RETRY_HEADER_COUNT]: String(attempt),
          [RETRY_HEADER_AFTER]: String(retryAfter),
        },
      },
    ],
  });
}

// consumeRetryQueue memproses topic retry: nunggu sampai waktu backoff-nya
// lewat, lalu coba proses ulang lewat handler -- kalau masih gagal dan
// attempt sudah mentok MAX_RETRIES, event dikirim ke dead letter queue
// (topik 52).
async function consumeRetryQueue(consumer, retryProducer, dlqProducer, handler) {
  await consumer.subscribe({ topic: 'order.created.retry', fromBeginning: false });
  await consumer.run({
    autoCommit: false,
    eachMessage: async ({ topic, partition, message }) => {
      const attempt = Number(message.headers[RETRY_HEADER_COUNT]?.toString() ?? '0');
      const retryAfter = Number(message.headers[RETRY_HEADER_AFTER]?.toString() ?? '0');
      const wait = retryAfter - Date.now();
      if (wait > 0) {
        await new Promise((resolve) => setTimeout(resolve, wait));
      }

      const event = JSON.parse(message.value.toString());
      try {
        await handler(event);
      } catch (err) {
        if (attempt >= MAX_RETRIES) {
          await publishToDeadLetterQueue(dlqProducer, event, err);
        } else {
          await publishToRetryQueue(retryProducer, event, attempt + 1);
        }
      }

      await consumer.commitOffsets([
        { topic, partition, offset: (Number(message.offset) + 1).toString() },
      ]);
    },
  });
}

module.exports = { publishToRetryQueue, consumeRetryQueue, backoffMs };
```

### Trade-off & Pitfall
- `ConsumeRetryQueue`/`consumeRetryQueue` pakai `time.Sleep`/`setTimeout` buat nunggu backoff langsung di dalam loop consumer — sederhana untuk dipahami, tapi menahan satu consumer instance idle selama backoff berjalan; sistem dengan volume retry besar biasanya butuh scheduler terpisah (misalnya beberapa topic retry dengan delay berjenjang: 5 detik, 1 menit, 10 menit) alih-alih blocking sleep.
- Backoff eksponensial mencegah retry membanjiri service yang lagi down (thundering herd), tapi juga berarti event yang gagal butuh waktu lebih lama buat akhirnya berhasil diproses kalau memang cuma gangguan sesaat — ada trade-off antara "gak membebani service yang down" dan "kecepatan pemulihan".
- `maxRetries` yang terlalu kecil bikin event cepat masuk DLQ (butuh penanganan manual) walau sebenarnya cuma butuh sedikit lebih banyak percobaan; terlalu besar bikin event "menggantung" lama di retry queue sebelum akhirnya ketahuan gagal total.

### Kapan Dipakai
Ketika kegagalan memproses sebuah event kemungkinan besar bersifat sementara (service downstream lagi down/lambat, network blip) dan diperkirakan akan berhasil kalau dicoba lagi setelah beberapa saat — seperti kegagalan kirim email karena `email-service` lagi restart deployment.

### Sering Ditanya Saat Interview
- "Kenapa gak retry langsung di consumer utama pas handler gagal?" — retry langsung (blocking) menahan semua event lain yang antre di partition yang sama (head-of-line blocking); memindahkan ke topic retry terpisah membiarkan consumer utama lanjut memproses event lain yang gak berhubungan dengan kegagalan itu.
- "Kenapa backoff-nya eksponensial, bukan interval tetap?" — supaya gak langsung membanjiri service yang lagi down dengan retry beruntun (thundering herd); backoff yang makin panjang memberi waktu service tersebut pulih sebelum dicoba lagi.
- "Apa yang terjadi kalau event gagal terus sampai `maxRetries` habis?" — event itu dipublish ke Dead Letter Queue (topik 52) lengkap dengan alasan error terakhirnya, buat diperiksa dan ditangani manual, bukan di-retry selamanya.

---

## 52. Dead Letter Queue

### Apa itu?
Dead Letter Queue (DLQ) adalah topic khusus tempat menampung event yang gagal diproses berkali-kali sampai melebihi batas retry (topik 51), lengkap dengan alasan kegagalan terakhirnya. Di OrderFlow, event `OrderCreated` yang sudah gagal diproses `maxRetries` kali dipublish ke topic `order.created.dlq` alih-alih hilang begitu saja atau di-retry selamanya.

### Kenapa dibutuhkan?
Ada kegagalan yang memang gak akan pernah berhasil walau di-retry berapa kali pun — misalnya payload event yang corrupt, atau bug di handler yang selalu gagal untuk kombinasi data tertentu. Tanpa DLQ, event seperti ini akan terus di-retry tanpa henti (buang-buang resource) atau, lebih buruk, hilang begitu saja tanpa jejak kalau retry-nya dihentikan tanpa disimpan di mana pun. DLQ memastikan event yang gagal total tetap tercatat dan bisa diperiksa/di-replay manual setelah bug-nya diperbaiki.

### Cara Kerja
```
ConsumeRetryQueue (topik 51): attempt sudah mencapai maxRetries, handler masih gagal
        |
        v
PublishToDeadLetterQueue -> topic "order.created.dlq"
        header error-reason = "email-service: connection refused"
        header failed-at    = "2026-08-14T10:00:00Z"

ConsumeDeadLetterQueue baca topic dlq:
  - simpan event + alasan gagal ke tabel dlq_events (buat audit/replay manual)
  - kirim alert ke tim (Slack/PagerDuty) -- TIDAK mencoba proses ulang otomatis
```

### Contoh Kode — Go
```go
package messaging

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/segmentio/kafka-go"
)

// PublishToDeadLetterQueue mengirim event yang sudah gagal diproses melebihi
// maxRetries (topik 51) ke topic order.created.dlq, lengkap dengan alasan
// error terakhirnya -- supaya bisa diperiksa/di-replay manual, bukan hilang
// begitu saja.
func PublishToDeadLetterQueue(ctx context.Context, w *kafka.Writer, event OrderEvent, lastErr error) error {
	payload, err := json.Marshal(event)
	if err != nil {
		return fmt.Errorf("marshal dlq event: %w", err)
	}
	return w.WriteMessages(ctx, kafka.Message{
		Key:   []byte(fmt.Sprintf("order-%d", event.OrderID)),
		Value: payload,
		Headers: []kafka.Header{
			{Key: "error-reason", Value: []byte(lastErr.Error())},
			{Key: "failed-at", Value: []byte(time.Now().UTC().Format(time.RFC3339))},
		},
	})
}

// ConsumeDeadLetterQueue cuma menyimpan event yang gagal total ke tabel
// dlq_events buat diperiksa manual dan mengirim alert -- TIDAK mencoba
// memproses ulang secara otomatis (kalau bisa otomatis, harusnya event ini
// belum sampai masuk DLQ).
func ConsumeDeadLetterQueue(ctx context.Context, r *kafka.Reader, db *pgxpool.Pool, alert func(string) error) error {
	for {
		msg, err := r.FetchMessage(ctx)
		if err != nil {
			return fmt.Errorf("fetch dlq message: %w", err)
		}

		var event OrderEvent
		if err := json.Unmarshal(msg.Value, &event); err != nil {
			return fmt.Errorf("unmarshal dlq event: %w", err)
		}
		reason := headerValue(msg.Headers, "error-reason")

		if _, err := db.Exec(ctx,
			`INSERT INTO dlq_events (event_id, order_id, payload, reason) VALUES ($1, $2, $3, $4)`,
			event.EventID, event.OrderID, msg.Value, reason,
		); err != nil {
			return fmt.Errorf("insert dlq event: %w", err)
		}
		if err := alert(fmt.Sprintf("Order event %s masuk DLQ: %s", event.EventID, reason)); err != nil {
			return fmt.Errorf("send dlq alert: %w", err)
		}

		if err := r.CommitMessages(ctx, msg); err != nil {
			return fmt.Errorf("commit dlq message: %w", err)
		}
	}
}

func headerValue(headers []kafka.Header, key string) string {
	for _, h := range headers {
		if h.Key == key {
			return string(h.Value)
		}
	}
	return "unknown"
}
```

### Contoh Kode — Node.js
```javascript
// publishToDeadLetterQueue mengirim event yang sudah gagal diproses melebihi
// MAX_RETRIES (topik 51) ke topic order.created.dlq, lengkap dengan alasan
// error terakhirnya.
async function publishToDeadLetterQueue(producer, event, lastErr) {
  await producer.send({
    topic: 'order.created.dlq',
    messages: [
      {
        key: `order-${event.orderId}`,
        value: JSON.stringify(event),
        headers: {
          'error-reason': lastErr.message,
          'failed-at': new Date().toISOString(),
        },
      },
    ],
  });
}

// consumeDeadLetterQueue cuma menyimpan event yang gagal total ke tabel
// dlq_events buat diperiksa manual dan mengirim alert -- TIDAK mencoba
// memproses ulang secara otomatis.
async function consumeDeadLetterQueue(consumer, pool, alertFn) {
  await consumer.subscribe({ topic: 'order.created.dlq', fromBeginning: false });
  await consumer.run({
    eachMessage: async ({ message }) => {
      const event = JSON.parse(message.value.toString());
      const reason = message.headers['error-reason']?.toString() ?? 'unknown';
      await pool.query(
        'INSERT INTO dlq_events (event_id, order_id, payload, reason) VALUES ($1, $2, $3, $4)',
        [event.eventId, event.orderId, JSON.stringify(event), reason]
      );
      await alertFn(`Order event ${event.eventId} masuk DLQ: ${reason}`);
    },
  });
}

module.exports = { publishToDeadLetterQueue, consumeDeadLetterQueue };
```

### Trade-off & Pitfall
- DLQ cuma berguna kalau ada proses (manual atau semi-otomatis) buat benar-benar memeriksa dan menindaklanjuti event yang masuk ke sana — DLQ yang dibiarkan menumpuk tanpa pernah diperiksa sama aja dengan membuang event itu, cuma lebih lambat dan lebih menipu (seolah-olah "aman" karena tercatat).
- Menyimpan payload lengkap event di tabel `dlq_events` penting buat bisa di-replay setelah bug-nya diperbaiki, tapi juga berarti data sensitif (kalau ada) ikut tersimpan di tempat tambahan — perlu kebijakan retensi/pembersihan data DLQ yang jelas.
- Alert yang dikirim tiap kali ada event masuk DLQ bisa jadi noise kalau volumenya besar (misalnya satu bug menyebabkan ratusan event gagal sekaligus) — perlu rate limiting atau agregasi alert, bukan satu notifikasi per event.

### Kapan Dipakai
Setiap kali ada mekanisme retry (topik 51) yang punya batas maksimum percobaan — DLQ adalah "jaring pengaman terakhir" supaya event yang gagal total tetap punya jejak yang bisa diperiksa, bukan hilang diam-diam setelah retry terakhirnya gagal.

### Sering Ditanya Saat Interview
- "Apa bedanya DLQ dengan retry queue (topik 51)?" — retry queue untuk kegagalan yang diperkirakan sementara dan akan berhasil kalau dicoba lagi setelah backoff; DLQ untuk event yang sudah melebihi batas retry maksimum, dianggap gagal total dan butuh penanganan manual, bukan retry otomatis lagi.
- "Kenapa `ConsumeDeadLetterQueue` gak mencoba proses ulang event-nya secara otomatis?" — kalau memang bisa berhasil dengan retry otomatis, event itu seharusnya sudah berhasil di tahap retry queue sebelum sampai masuk DLQ; event yang sampai DLQ biasanya butuh investigasi manusia (bug fix, data correction) sebelum bisa di-replay dengan aman.
- "Apa risiko kalau DLQ dibiarkan menumpuk tanpa pernah diperiksa?" — sama saja dengan kehilangan event-event itu secara diam-diam, cuma lebih menipu karena tercatat seolah "aman"; perlu proses rutin (dashboard, alert, on-call) buat benar-benar menindaklanjuti isi DLQ.

---

## 53. Kafka Basics

### Apa itu?
Kafka adalah distributed log/streaming platform yang jadi message broker utama OrderFlow. Konsep dasarnya: **topic** (kategori event, misalnya `order.created`), **partition** (topic dibagi jadi beberapa partition supaya bisa diproses paralel), **offset** (posisi/urutan message di dalam satu partition), dan **consumer group** (sekumpulan consumer yang berbagi beban baca satu topic, tiap partition cuma dibaca satu consumer dalam group yang sama).

### Kenapa dibutuhkan?
Semua fungsi yang sudah dibahas — `PublishOrderCreated`, `ConsumeOrderEvents`, retry queue, DLQ — butuh koneksi dan konfigurasi dasar ke cluster Kafka: alamat broker, nama topic, balancer (strategi pembagian partition), dan consumer group ID. Tanpa memahami konsep dasar ini, sulit men-debug masalah seperti "kenapa urutan event per order kadang gak konsisten" (jawabannya soal partitioning) atau "kenapa cuma sebagian instance service yang kebagian event" (jawabannya soal consumer group).

### Cara Kerja
```
Topic "order.created" dengan 3 partition:

  Partition 0: [evt-1] [evt-4] [evt-7] ...
  Partition 1: [evt-2] [evt-5] [evt-8] ...
  Partition 2: [evt-3] [evt-6] [evt-9] ...

Key = "order-<id>" dipakai buat menentukan partition (lewat hashing) --
semua event dengan order ID yang sama SELALU masuk ke partition yang sama,
jadi urutan event per order tetap terjaga walau topic-nya punya banyak partition.

Consumer group "email-service" dengan 3 instance:
  instance-1 -> baca partition 0
  instance-2 -> baca partition 1
  instance-3 -> baca partition 2
Kalau instance-1 mati, Kafka rebalance: partition 0 dipindah ke instance lain
yang masih hidup dalam group yang sama.
```

### Contoh Kode — Go
```go
package messaging

import (
	"github.com/segmentio/kafka-go"
)

// NewOrderCreatedWriter bikin kafka.Writer buat topic order.created --
// Balancer Hash memastikan semua event dengan Key yang sama (order ID yang
// sama) selalu masuk ke partition yang sama, jadi urutan event per order
// tetap terjaga.
func NewOrderCreatedWriter(brokers []string) *kafka.Writer {
	return &kafka.Writer{
		Addr:         kafka.TCP(brokers...),
		Topic:        "order.created",
		Balancer:     &kafka.Hash{},
		RequiredAcks: kafka.RequireAll,
	}
}

// NewOrderCreatedReader bikin kafka.Reader dengan consumer GroupID --
// consumer group memastikan tiap partition topic order.created cuma dibaca
// oleh SATU consumer instance dalam group yang sama di satu waktu, jadi
// kalau ada 3 partition dan 3 instance service, tiap instance kebagian satu
// partition.
func NewOrderCreatedReader(brokers []string, groupID string) *kafka.Reader {
	return kafka.NewReader(kafka.ReaderConfig{
		Brokers: brokers,
		Topic:   "order.created",
		GroupID: groupID,
	})
}
```

### Contoh Kode — Node.js
```javascript
const { Kafka } = require('kafkajs');

// createKafkaClient bikin koneksi dasar ke Kafka cluster -- dipakai buat
// bikin producer dan consumer.
function createKafkaClient(brokers) {
  return new Kafka({ clientId: 'orderflow', brokers });
}

// createOrderCreatedProducer bikin producer buat topic order.created.
function createOrderCreatedProducer(kafka) {
  return kafka.producer();
}

// createOrderCreatedConsumer bikin consumer dengan groupId -- consumer
// group memastikan tiap partition cuma dibaca satu instance dalam group
// yang sama, jadi beban baca event tersebar ke semua instance service yang
// jalan.
function createOrderCreatedConsumer(kafka, groupId) {
  return kafka.consumer({ groupId });
}

module.exports = {
  createKafkaClient,
  createOrderCreatedProducer,
  createOrderCreatedConsumer,
};
```

### Trade-off & Pitfall
- Jumlah partition sebuah topic gak bisa dikurangi setelah dibuat (cuma bisa ditambah), dan menambah partition bisa mengubah pemetaan key-ke-partition untuk key yang sama — rencanakan jumlah partition dari awal berdasarkan perkiraan throughput dan jumlah consumer instance maksimum yang bakal dipakai.
- Consumer group yang lebih banyak instance-nya dibanding jumlah partition topic bakal punya instance yang menganggur (idle) — jumlah consumer instance efektif maksimum sama dengan jumlah partition topic-nya.
- `RequiredAcks: kafka.RequireAll` (tunggu semua in-sync replica ack) lebih aman dari kehilangan data dibanding `RequireOne`, tapi latency publish jadi sedikit lebih tinggi karena harus menunggu lebih banyak broker konfirmasi.

### Kapan Dipakai
Kafka cocok dipakai OrderFlow untuk event stream dengan volume tinggi yang butuh durability dan replay (bisa dibaca ulang dari offset tertentu), seperti `order.created` yang dikonsumsi banyak service berbeda-beda secara independen.

### Sering Ditanya Saat Interview
- "Kenapa key dipakai buat tentuin partition, bukan random?" — supaya semua event dengan key yang sama (misalnya order ID yang sama) selalu masuk ke partition yang sama, sehingga urutan event untuk entity itu tetap terjaga (Kafka cuma menjamin urutan di dalam satu partition, bukan across partition).
- "Apa yang terjadi kalau jumlah consumer instance dalam satu group lebih banyak dari jumlah partition?" — instance kelebihan itu gak kebagian partition sama sekali dan jadi idle, karena satu partition cuma bisa dibaca satu consumer dalam group yang sama di satu waktu.
- "Apa beda consumer group yang berbeda-beda membaca topic yang sama?" — masing-masing consumer group punya offset tracking sendiri-sendiri dan gak saling mempengaruhi; itu sebabnya `email-service` dan `invoice-service` bisa sama-sama membaca topic `order.created` dari awal sampai akhir secara independen.

---

## 54. RabbitMQ Basics

### Apa itu?
RabbitMQ adalah message broker berbasis model **exchange** dan **queue**: producer mempublish message ke exchange dengan sebuah routing key, exchange lalu meneruskan message itu ke queue mana saja yang binding-nya cocok dengan routing key tersebut, dan consumer membaca dari queue. Ini beda dari Kafka yang berbasis topic/partition/offset — RabbitMQ lebih menekankan fleksibilitas routing (satu event bisa di-routing ke banyak queue berbeda berdasarkan pola routing key).

### Kenapa dibutuhkan?
Beberapa kasus di OrderFlow lebih pas dimodelkan lewat routing yang fleksibel ala RabbitMQ dibanding topic Kafka yang lebih statis — misalnya kalau ada banyak jenis event order (`order.created`, `order.cancelled`, `order.refunded`) dan tiap consumer cuma mau berlangganan sebagian jenis event tertentu lewat pola routing key, tanpa harus subscribe ke topic besar lalu filter sendiri di application code. RabbitMQ dipakai sebagai alternatif broker untuk kasus seperti ini.

### Cara Kerja
```
Producer -> publish ke exchange "order.events" dengan routing key "order.created"
                    |
                    v
        exchange (type: topic) mencocokkan routing key dengan binding tiap queue

  queue "email-notifications"   <- binding: "order.created", "order.cancelled"
  queue "warehouse-updates"     <- binding: "order.created", "order.refunded"
  queue "analytics-all-events"  <- binding: "order.*"  (semua event order)

Consumer baca dari queue masing-masing, ack manual SETELAH handler sukses
(sama prinsipnya dengan commit offset di Kafka -- at-least-once).
```

### Contoh Kode — Go
```go
package messaging

import (
	"encoding/json"
	"fmt"
	"time"

	"github.com/google/uuid"
	amqp "github.com/rabbitmq/amqp091-go"
)

// PublishOrderCreatedRabbitMQ alternatif PublishOrderCreated (topik 48) kalau
// broker yang dipakai RabbitMQ, bukan Kafka -- exchange topic "order.events"
// dengan routing key "order.created" memungkinkan consumer lain bikin
// binding sendiri (misalnya cuma mau dengar "order.cancelled").
func PublishOrderCreatedRabbitMQ(ch *amqp.Channel, order *Order) error {
	event := OrderEvent{
		EventID:   uuid.NewString(),
		OrderID:   order.ID,
		UserID:    order.UserID,
		Total:     order.Total,
		CreatedAt: time.Now().UTC(),
	}
	payload, err := json.Marshal(event)
	if err != nil {
		return fmt.Errorf("marshal order event: %w", err)
	}
	return ch.Publish(
		"order.events",  // exchange
		"order.created", // routing key
		false,            // mandatory
		false,            // immediate
		amqp.Publishing{
			ContentType:  "application/json",
			DeliveryMode: amqp.Persistent,
			Body:         payload,
		},
	)
}

// ConsumeOrderEventsRabbitMQ alternatif ConsumeOrderEvents (topik 48) versi
// RabbitMQ -- ack manual cuma dikirim SETELAH handler sukses (at-least-once,
// sama prinsipnya dengan CommitMessages di Kafka).
func ConsumeOrderEventsRabbitMQ(ch *amqp.Channel, queueName string, handler func(OrderEvent) error) error {
	msgs, err := ch.Consume(queueName, "", false, false, false, false, nil)
	if err != nil {
		return fmt.Errorf("consume queue %s: %w", queueName, err)
	}

	for msg := range msgs {
		var event OrderEvent
		if err := json.Unmarshal(msg.Body, &event); err != nil {
			msg.Nack(false, false) // buang, gak perlu di-requeue -- payload-nya emang rusak
			continue
		}
		if err := handler(event); err != nil {
			msg.Nack(false, true) // requeue supaya bisa dicoba ulang
			continue
		}
		msg.Ack(false)
	}
	return nil
}
```

### Contoh Kode — Node.js
```javascript
const crypto = require('crypto');

// publishOrderCreatedRabbitMQ alternatif publishOrderCreated (topik 48) kalau
// broker yang dipakai RabbitMQ, bukan Kafka.
function publishOrderCreatedRabbitMQ(channel, order) {
  const event = {
    eventId: crypto.randomUUID(),
    orderId: order.id,
    // createOrder (phase 03) mengembalikan raw row dari
    // "RETURNING id, user_id, status, total, version" tanpa aliasing, jadi
    // key-nya snake_case (order.user_id), bukan order.userId.
    userId: order.user_id,
    total: order.total,
    createdAt: new Date().toISOString(),
  };
  return channel.publish(
    'order.events',
    'order.created',
    Buffer.from(JSON.stringify(event)),
    { contentType: 'application/json', persistent: true }
  );
}

// consumeOrderEventsRabbitMQ alternatif consumeOrderEvents (topik 48) versi
// RabbitMQ -- ack manual cuma dikirim SETELAH handler sukses.
async function consumeOrderEventsRabbitMQ(channel, queueName, handler) {
  await channel.prefetch(1);
  await channel.consume(
    queueName,
    async (msg) => {
      if (!msg) return;
      try {
        const event = JSON.parse(msg.content.toString());
        await handler(event);
        channel.ack(msg);
      } catch (err) {
        channel.nack(msg, false, true); // requeue supaya bisa dicoba ulang
      }
    },
    { noAck: false }
  );
}

module.exports = { publishOrderCreatedRabbitMQ, consumeOrderEventsRabbitMQ };
```

### Trade-off & Pitfall
- Model routing RabbitMQ (exchange + binding pattern) lebih fleksibel dibanding topic Kafka yang statis, tapi RabbitMQ secara umum punya throughput maksimum lebih rendah dibanding Kafka untuk skenario streaming volume sangat tinggi dengan banyak consumer group independen membaca ulang riwayat yang sama.
- RabbitMQ secara default gak menyimpan message setelah semua consumer selesai mengonsumsinya (beda dari Kafka yang menyimpan log sesuai retention policy) — kalau butuh replay event lama, RabbitMQ bukan pilihan yang cocok tanpa mekanisme tambahan (misalnya menyimpan salinan event ke tempat lain).
- `channel.prefetch(1)` (Node.js) membatasi jumlah unacknowledged message yang boleh beredar ke satu consumer — kalau prefetch terlalu besar dan consumer crash, banyak message yang belum di-ack sekaligus di-requeue ulang, memperbesar dampak satu kegagalan.

### Kapan Dipakai
RabbitMQ cocok dipakai OrderFlow ketika kebutuhan utamanya adalah routing yang fleksibel antar banyak jenis event ke consumer yang berbeda-beda (lewat routing key pattern), atau untuk task queue klasik (satu message diproses satu worker) yang gak butuh replay riwayat event lama seperti yang disediakan Kafka.

### Sering Ditanya Saat Interview
- "Apa beda fundamental Kafka dan RabbitMQ?" — Kafka adalah distributed log yang menyimpan event berurutan per partition dan bisa di-replay dari offset manapun sesuai retention policy; RabbitMQ berbasis exchange/queue dengan routing yang lebih fleksibel, tapi begitu message di-ack oleh consumer, message itu hilang dari queue (gak ada replay bawaan).
- "Kenapa contoh RabbitMQ di OrderFlow pakai exchange type topic, bukan direct atau fanout?" — exchange topic memungkinkan binding pakai pola routing key (misalnya `order.*` buat semua event order), memberi fleksibilitas mana consumer yang mau dengar event jenis apa, tanpa harus bikin binding satu-satu untuk tiap kombinasi.
- "Apa fungsi `prefetch`/QoS di consumer RabbitMQ?" — membatasi berapa banyak message yang boleh dikirim ke satu consumer sebelum consumer itu meng-ack message sebelumnya, supaya satu consumer yang lambat gak kebanjiran message sekaligus dan supaya dampak crash-nya (message yang di-requeue) gak terlalu besar.

---

**Selanjutnya:** [Phase 06 — Go Concurrency, Testing & Backend Performance](./phase-06-concurrency-testing-performance.md)
