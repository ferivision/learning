# Phase 14 — Advanced Backend Concepts

> Bagian dari [Backend Engineer Roadmap](../README.md)

---

## 98. Event-Driven Architecture

### Apa itu?
Event-Driven Architecture (EDA) adalah gaya arsitektur di mana komponen-komponen sistem berkomunikasi lewat **event** (fakta bahwa sesuatu sudah terjadi, misalnya "order sudah dibuat") alih-alih lewat pemanggilan langsung satu sama lain. Di OrderFlow, `CreateOrder` (Phase 03) menyimpan order ke Postgres, lalu `PublishOrderCreated` (Phase 05) mempublish event `OrderCreated` ke Kafka — dan dari titik itu, `payment-service`, `notification-service`, `invoice-service`, dan consumer lain mana pun bisa "mendengarkan" event itu tanpa `CreateOrder` sendiri tau atau peduli siapa saja yang mendengarkan.

### Kenapa dibutuhkan?
Arsitektur request-response murni (service A manggil service B secara langsung lewat HTTP) bikin service A harus tau alamat, kontrak, dan kondisi kesehatan SEMUA service B, C, D yang perlu bereaksi terhadap tindakannya — begitu ada consumer baru (misalnya `loyalty-service` yang perlu tau tiap kali ada order baru), kode `CreateOrder` harus diubah lagi buat menambah satu pemanggilan lagi. EDA memutus ketergantungan ini: `CreateOrder` + `PublishOrderCreated` (Phase 03 & 05) gak pernah berubah walau jumlah consumer bertambah dari satu jadi sepuluh, karena publisher dan consumer cuma "sepakat" lewat bentuk event-nya (`OrderEvent`), bukan lewat pemanggilan langsung.

### Cara Kerja
```
                         CheckoutOrder (orchestrator tipis)
                                   |
                    CreateOrder (Phase 03) -- commit ke Postgres
                                   |
                    PublishOrderCreated (Phase 05) -- publish ke topic "order.created"
                                   |
        +--------------------------+--------------------------+
        |                          |                          |
        v                          v                          v
StartPaymentConsumer       StartNotificationConsumer     (consumer baru di masa
(ConsumeOrderEvents)       (ConsumeOrderEvents)          depan -- loyalty-service,
        |                          |                     dst -- TANPA mengubah
        v                          v                     CheckoutOrder sama sekali)
proses pembayaran          kirim email konfirmasi

Publisher (CheckoutOrder) TIDAK TAU dan TIDAK PEDULI berapa banyak
consumer yang membaca topic "order.created" -- itulah decoupling inti EDA.
```

### Contoh Kode — Go
```go
package eventdriven

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/segmentio/kafka-go"

	orderdb "orderflow/internal/db"
	"orderflow/internal/messaging"
)

// CheckoutOrder adalah entry point event-driven checkout: CreateOrder
// (Phase 03) menyimpan order ke Postgres, lalu PublishOrderCreated
// (Phase 05) mempublish event OrderCreated ke Kafka -- OrderFlow SENDIRI
// berhenti di sini, gak tau dan gak peduli siapa saja yang bereaksi
// terhadap event ini.
func CheckoutOrder(ctx context.Context, db *pgxpool.Pool, w *kafka.Writer, userID int64, items []orderdb.OrderItem) (*orderdb.Order, error) {
	order, err := orderdb.CreateOrder(ctx, db, userID, items)
	if err != nil {
		return nil, fmt.Errorf("create order: %w", err)
	}
	if err := messaging.PublishOrderCreated(ctx, w, order); err != nil {
		return order, fmt.Errorf("order created but publish failed: %w", err)
	}
	return order, nil
}

// StartPaymentConsumer dan StartNotificationConsumer adalah DUA consumer
// group yang BERBEDA, sama-sama membaca topic order.created lewat
// ConsumeOrderEvents (Phase 05) -- keduanya gak saling tau soal keberadaan
// satu sama lain, dan CheckoutOrder di atas gak perlu berubah sama sekali
// walau nanti ditambah consumer ketiga (misalnya loyalty-service).
func StartPaymentConsumer(ctx context.Context, r *kafka.Reader, processPayment func(messaging.OrderEvent) error) error {
	return messaging.ConsumeOrderEvents(ctx, r, processPayment)
}

func StartNotificationConsumer(ctx context.Context, r *kafka.Reader, sendNotification func(messaging.OrderEvent) error) error {
	return messaging.ConsumeOrderEvents(ctx, r, sendNotification)
}
```

### Contoh Kode — Node.js
```javascript
const { createOrder } = require('./order-repository'); // dari Phase 03
const { publishOrderCreated, consumeOrderEvents } = require('./messaging'); // dari Phase 05

// checkoutOrder adalah entry point event-driven checkout: createOrder
// (Phase 03) menyimpan order ke Postgres, lalu publishOrderCreated
// (Phase 05) mempublish event OrderCreated ke Kafka -- OrderFlow sendiri
// berhenti di sini, gak tau dan gak peduli siapa saja yang bereaksi
// terhadap event ini.
async function checkoutOrder(pool, producer, userId, items) {
  const order = await createOrder(pool, userId, items);
  try {
    await publishOrderCreated(producer, order);
  } catch (err) {
    console.error(`order created but publish failed for order ${order.id}:`, err);
  }
  return order;
}

// startPaymentConsumer dan startNotificationConsumer adalah DUA consumer
// group berbeda yang sama-sama membaca topic order.created lewat
// consumeOrderEvents (Phase 05) -- keduanya independen satu sama lain, dan
// checkoutOrder di atas gak perlu berubah walau ditambah consumer baru.
async function startPaymentConsumer(consumer, processPayment) {
  await consumeOrderEvents(consumer, processPayment);
}

async function startNotificationConsumer(consumer, sendNotification) {
  await consumeOrderEvents(consumer, sendNotification);
}

module.exports = { checkoutOrder, startPaymentConsumer, startNotificationConsumer };
```

### Trade-off & Pitfall
- EDA menukar kesederhanaan debugging (satu call stack lurus di request-response) dengan fleksibilitas komposisi — begitu ada bug, developer harus melacak alur lewat banyak consumer independen yang jalan di waktu berbeda-beda, bukan cuma satu stack trace di satu proses.
- Publisher (`CheckoutOrder`) gak pernah tau apakah semua consumer yang "seharusnya" memproses event itu benar-benar berhasil — kalau `payment-service` diam-diam berhenti berjalan (bukan error, tapi crash total), gak ada sinyal langsung ke publisher; observability (Phase 10) jadi wajib, bukan opsional.
- Menambah consumer baru memang gak mengubah publisher, tapi menambah **beban baca** di topic yang sama — kalau volume consumer terus bertambah tanpa partition (topik 53, Phase 05) yang cukup, latency baca antar consumer group bisa saling bersaing untuk resource broker yang sama.

### Kapan Dipakai
Ketika ada satu kejadian bisnis (order dibuat, pembayaran berhasil, dst) yang perlu diketahui oleh lebih dari satu bagian sistem, dan bagian-bagian itu boleh berkembang jumlahnya dari waktu ke waktu tanpa mengubah kode yang mempublish event. Untuk alur yang sifatnya benar-benar satu arah dan gak akan pernah punya consumer kedua (misalnya validasi input sebelum insert), EDA cuma menambah kompleksitas yang gak perlu.

### Sering Ditanya Saat Interview
- "Apa beda EDA dengan sekadar 'pakai message queue'?" — message queue (Phase 05) adalah **mekanisme** transportnya; EDA adalah **gaya arsitektur** yang memutuskan komunikasi antar service lewat event, bukan pemanggilan langsung — message queue adalah salah satu cara paling umum buat mengimplementasikan EDA.
- "Kenapa `CheckoutOrder` gak perlu tau berapa banyak consumer yang ada?" — karena kontrak antara publisher dan consumer cuma bentuk event-nya (`OrderEvent`), bukan daftar consumer; menambah consumer baru cukup dengan subscribe ke topic yang sama, tanpa menyentuh kode publisher sama sekali.
- "Apa risiko terbesar EDA dibanding request-response biasa?" — hilangnya satu call stack yang bisa ditelusuri linear; kegagalan tersebar ke banyak consumer independen yang perlu observability (tracing, metrics, Phase 10) tersendiri supaya masalah tetap bisa dilacak.

---

## 99. Saga Pattern

### Apa itu?
Saga Pattern adalah cara mengelola satu transaksi bisnis yang melibatkan **beberapa langkah di sistem berbeda-beda** (Postgres, inventory service, payment provider) tanpa bisa membungkusnya dalam satu database transaction tunggal. Alur checkout OrderFlow yang lengkap adalah contoh sempurna: Create Order (Phase 03) → Reserve Inventory → Charge Payment (lewat `CallPaymentProviderWithCircuitBreaker`, Phase 09) → Ship — tiap step dijalankan berurutan, dan kalau salah satu gagal, semua step SEBELUMNYA yang sudah sukses dibatalkan lewat **compensating action** (aksi kebalikan), bukan di-rollback otomatis seperti transaction database biasa.

### Kenapa dibutuhkan?
`BEGIN ... COMMIT` di Postgres cuma bisa melindungi operasi yang seluruhnya terjadi di dalam satu database yang sama — begitu salah satu step butuh memanggil inventory service lewat HTTP atau payment provider eksternal (topik 79, Phase 09), gak ada `ROLLBACK` yang bisa "membatalkan" panggilan HTTP yang sudah terjadi. Tanpa Saga, kegagalan di step 3 (charge payment) meninggalkan sistem dalam keadaan tidak konsisten: order sudah dibuat, inventory sudah direservasi, tapi pembayaran gagal — dan gak ada satu pun mekanisme otomatis yang membereskan inventory yang sudah kadung dikunci itu. Saga mendefinisikan eksplisit: kalau step N gagal, jalankan compensating action untuk step N-1, N-2, ..., 1 secara berurutan terbalik.

### Cara Kerja
```
CheckoutSaga (orchestration-based -- satu fungsi koordinator yang eksplisit
memanggil tiap step dan compensating action-nya, dibanding choreography-based
yang tersebar sebagai reaksi antar event):

Step 1: CreateOrder (Phase 03)              -- sukses
Step 2: ReserveInventory                     -- sukses
Step 3: CallPaymentProviderWithCircuitBreaker (Phase 09) -- GAGAL
                    |
                    v
        Compensate step 2: ReleaseInventory (urutan TERBALIK)
        Compensate step 1: cancelOrder

Kalau step 4 (ShipOrder) yang gagal, SEMUA compensating action step 1-3
dijalankan terbalik: refund payment -> release inventory -> cancel order.
Kalau semua step sukses, gak ada compensating action yang dijalankan sama
sekali -- saga "selesai" begitu step terakhir (Ship) sukses.
```

### Contoh Kode — Go
```go
package saga

import (
	"context"
	"fmt"

	"github.com/jackc/pgx/v5/pgxpool"

	orderdb "orderflow/internal/db"
	"orderflow/internal/payment"
)

// InventoryClient dibuat sebagai interface (bukan concrete HTTP client)
// supaya CheckoutSaga bisa di-unit-test dengan fake/mock, tanpa perlu
// inventory service beneran nyala.
type InventoryClient interface {
	ReserveInventory(ctx context.Context, orderID int64, items []orderdb.OrderItem) error
	ReleaseInventory(ctx context.Context, orderID int64, items []orderdb.OrderItem) error
}

// CheckoutSaga menjalankan orchestration-based saga: tiap step dijalankan
// berurutan, dan kalau salah satu GAGAL, semua step SEBELUMNYA yang sudah
// sukses di-compensate (dibatalkan) dengan urutan TERBALIK -- bukan lewat
// satu transaction database besar, karena step-step ini menyentuh sistem
// yang berbeda-beda (Postgres, inventory service, payment provider) yang
// gak bisa ikut dalam satu transaction yang sama.
func CheckoutSaga(ctx context.Context, db *pgxpool.Pool, inv InventoryClient, userID int64, items []orderdb.OrderItem, paymentToken string) (*orderdb.Order, error) {
	order, err := orderdb.CreateOrder(ctx, db, userID, items)
	if err != nil {
		return nil, fmt.Errorf("step 1 create order: %w", err)
	}

	if err := inv.ReserveInventory(ctx, order.ID, items); err != nil {
		// step 1 (CreateOrder) sudah sukses -- compensate dengan cancel order.
		cancelOrder(ctx, db, order.ID)
		return nil, fmt.Errorf("step 2 reserve inventory: %w", err)
	}

	resp, err := payment.CallPaymentProviderWithCircuitBreaker(ctx, payment.PaymentRequest{
		OrderID: order.ID,
		Amount:  order.Total,
		Token:   paymentToken,
	})
	if err != nil {
		// step 1 & 2 sudah sukses -- compensate KEDUANYA, urutan terbalik:
		// release inventory dulu, baru cancel order.
		inv.ReleaseInventory(ctx, order.ID, items)
		cancelOrder(ctx, db, order.ID)
		return nil, fmt.Errorf("step 3 charge payment: %w", err)
	}

	if err := shipOrder(ctx, db, order.ID); err != nil {
		// step 1, 2, 3 sudah sukses -- compensate SEMUANYA, urutan terbalik:
		// refund payment, release inventory, cancel order.
		refundPayment(resp.TransactionID)
		inv.ReleaseInventory(ctx, order.ID, items)
		cancelOrder(ctx, db, order.ID)
		return nil, fmt.Errorf("step 4 ship order: %w", err)
	}

	return order, nil
}

func cancelOrder(ctx context.Context, db *pgxpool.Pool, orderID int64) {
	if _, err := db.Exec(ctx, `UPDATE orders SET status = 'cancelled' WHERE id = $1`, orderID); err != nil {
		fmt.Printf("compensate: failed to cancel order %d: %v\n", orderID, err)
	}
}

func shipOrder(ctx context.Context, db *pgxpool.Pool, orderID int64) error {
	_, err := db.Exec(ctx, `UPDATE orders SET status = 'shipped' WHERE id = $1`, orderID)
	return err
}

func refundPayment(transactionID string) {
	fmt.Printf("compensate: refunding transaction %s\n", transactionID)
}
```

### Contoh Kode — Node.js
```javascript
const { createOrder } = require('./order-repository'); // dari Phase 03
const { callPaymentProviderWithCircuitBreaker } = require('./payment-circuit-breaker'); // dari Phase 09

// checkoutSaga menjalankan orchestration-based saga: tiap step dijalankan
// berurutan, dan kalau salah satu gagal, semua step sebelumnya yang sudah
// sukses di-compensate dengan urutan TERBALIK.
async function checkoutSaga(pool, inventoryClient, userId, items, paymentToken) {
  const order = await createOrder(pool, userId, items);

  try {
    await inventoryClient.reserveInventory(order.id, items);
  } catch (err) {
    await cancelOrder(pool, order.id);
    throw new Error(`step 2 reserve inventory: ${err.message}`);
  }

  let paymentResult;
  try {
    paymentResult = await callPaymentProviderWithCircuitBreaker({
      orderId: order.id,
      amount: order.total,
      token: paymentToken,
    });
  } catch (err) {
    await inventoryClient.releaseInventory(order.id, items);
    await cancelOrder(pool, order.id);
    throw new Error(`step 3 charge payment: ${err.message}`);
  }

  try {
    await shipOrder(pool, order.id);
  } catch (err) {
    await refundPayment(paymentResult.transactionId);
    await inventoryClient.releaseInventory(order.id, items);
    await cancelOrder(pool, order.id);
    throw new Error(`step 4 ship order: ${err.message}`);
  }

  return order;
}

async function cancelOrder(pool, orderId) {
  await pool.query("UPDATE orders SET status = 'cancelled' WHERE id = $1", [orderId]);
}

async function shipOrder(pool, orderId) {
  await pool.query("UPDATE orders SET status = 'shipped' WHERE id = $1", [orderId]);
}

async function refundPayment(transactionId) {
  console.log(`compensate: refunding transaction ${transactionId}`);
}

module.exports = { checkoutSaga };
```

### Trade-off & Pitfall
- Compensating action gak selalu bisa "membatalkan sempurna" — refund payment butuh waktu buat diproses payment provider, dan kalau `ReleaseInventory` sendiri gagal (misalnya inventory service lagi down saat compensate dijalankan), sistem bisa berakhir di keadaan lebih tidak konsisten daripada sebelum saga dimulai; compensating action sendiri idealnya idempotent dan di-retry (topik 51, Phase 05).
- Orchestration-based saga (satu fungsi koordinator eksplisit seperti `CheckoutSaga`) lebih mudah dipahami alurnya dibanding choreography-based (tiap service bereaksi ke event tanpa koordinator pusat), tapi orchestrator jadi single point yang tau seluruh alur bisnis — kalau alurnya makin kompleks (belasan step), orchestrator bisa jadi god-object yang susah dipelihara.
- Saga TIDAK memberi isolasi seperti transaction database — antara step 2 (inventory direservasi) dan step 3 (payment) selesai, ada jendela waktu di mana stock sudah dikunci tapi order belum benar-benar "dibayar"; request checkout lain yang butuh stock yang sama tetap harus menunggu, bukan langsung ditolak.

### Kapan Dipakai
Ketika satu transaksi bisnis melibatkan lebih dari satu sistem/service yang gak bisa disatukan dalam satu database transaction — checkout OrderFlow adalah contoh klasik karena melibatkan Postgres (order), inventory service, dan payment provider eksternal sekaligus. Untuk operasi yang seluruhnya terjadi di satu database yang sama, `BEGIN...COMMIT` biasa (Phase 03) sudah cukup dan jauh lebih sederhana daripada saga.

### Sering Ditanya Saat Interview
- "Kenapa gak pakai distributed transaction (two-phase commit) aja buat checkout ini?" — 2PC butuh SEMUA sistem yang terlibat (Postgres, inventory service, payment provider pihak ketiga) mendukung protokol yang sama dan saling mengunci resource sampai semua pihak setuju commit, yang gak realistis untuk payment provider eksternal yang gak kita kendalikan; saga menerima trade-off eventual consistency demi tetap bisa jalan lintas sistem yang independen.
- "Apa beda orchestration-based dan choreography-based saga?" — orchestration-based punya satu koordinator eksplisit (seperti `CheckoutSaga`) yang memanggil tiap step dan compensating action-nya secara langsung; choreography-based gak punya koordinator pusat, tiap service bereaksi ke event dari service sebelumnya (mirip topik 98) dan mempublish event kompensasinya sendiri kalau gagal.
- "Apa yang terjadi kalau compensating action-nya sendiri gagal?" — sistem bisa berakhir dalam keadaan tidak konsisten yang butuh intervensi manual; compensating action idealnya idempotent dan punya retry sendiri (topik 51, Phase 05), dan kegagalan compensate harus dialertkan, bukan diam-diam diabaikan.

---

## 100. Outbox Pattern

### Apa itu?
Outbox Pattern adalah cara menjamin sebuah perubahan database DAN sebuah event yang harus dipublish akibat perubahan itu **sama-sama tersimpan atau sama-sama gak tersimpan** — dengan cara menyimpan event itu sebagai baris di tabel `outbox_events` pada Postgres yang sama, di dalam transaction yang sama dengan perubahan datanya. Di OrderFlow, `SaveOrderWithOutbox` mengganti pola `CreateOrder` lalu `PublishOrderCreated` terpisah (Phase 05) dengan satu transaction yang meng-INSERT baris `orders` DAN baris `outbox_events` sekaligus.

### Kenapa dibutuhkan?
`CreateOrderAndPublish` di Phase 05 (topik 48) punya celah yang sudah disinggung tapi belum diselesaikan di sana: `CreateOrder` commit ke Postgres, LALU `PublishOrderCreated` mempublish ke Kafka sebagai langkah terpisah — kalau proses OrderFlow crash tepat di antara kedua langkah itu (commit sukses, publish belum sempat jalan), order tersimpan di database tapi event `OrderCreated`-nya HILANG selamanya, gak ada mekanisme retry apa pun yang bisa menyadarinya karena Kafka gak pernah tau event itu seharusnya ada. Outbox Pattern menutup celah ini dengan trik sederhana: "mempublish event" untuk sementara cukup berarti INSERT satu baris di tabel Postgres yang SAMA dengan tabel `orders` — INSERT ke dua tabel dalam satu transaction itu ATOMIC secara native oleh Postgres, jauh lebih mudah dijamin dibanding atomicity lintas Postgres dan Kafka.

### Cara Kerja
```
SaveOrderWithOutbox (SATU transaction Postgres):
  BEGIN
    INSERT INTO orders (...)         RETURNING id  -- step A
    INSERT INTO outbox_events (...)                -- step B, pakai id dari step A
  COMMIT

  Kalau step A gagal -> step B TIDAK PERNAH terjadi (rollback).
  Kalau step B gagal -> step A ikut di-rollback (order-nya juga gak tersimpan).
  Gak ada kemungkinan "order tersimpan tapi outbox event-nya hilang" --
  keduanya SELALU sama-sama ada atau sama-sama gak ada.

Proses TERPISAH (relay), berjalan independen dari request checkout:
  RelayPendingOutboxEvents (polling berkala, atau trigger via LISTEN/NOTIFY)
        |
        v
  SELECT ... FROM outbox_events WHERE published_at IS NULL FOR UPDATE SKIP LOCKED
        |
        v
  publish tiap baris ke Kafka topic "order.created"
        |
        v
  UPDATE outbox_events SET published_at = now()

  Kalau relay ini sendiri crash SEBELUM sempat UPDATE published_at, baris
  outbox_events itu masih ada -- relay berikutnya bakal mempublish ULANG
  (at-least-once, sama seperti topik 49 di Phase 05), BUKAN hilang.
```

### Contoh Kode — Go
```go
package outbox

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

// Order dipakai di sini sebagai alias tipe yang sama dengan Order di
// Phase 03 (package db).
type Order = orderdb.Order

// orderCreatedPayload adalah bentuk payload event yang disimpan di kolom
// payload outbox_events -- strukturnya sama dengan OrderEvent di Phase 05,
// supaya RelayPendingOutboxEvents di bawah bisa langsung mempublish isinya
// apa adanya tanpa perlu membangun ulang event dari nol.
type orderCreatedPayload struct {
	EventID   string    `json:"event_id"`
	OrderID   int64     `json:"order_id"`
	UserID    int64     `json:"user_id"`
	Total     float64   `json:"total"`
	CreatedAt time.Time `json:"created_at"`
}

// SaveOrderWithOutbox menyimpan order baru DAN sebuah baris outbox_events
// (event OrderCreated) dalam SATU transaction Postgres yang sama -- kalau
// salah satu INSERT gagal, KEDUANYA di-rollback. Ini menutup celah yang ada
// di CreateOrderAndPublish (Phase 05, topik 48): di sana ada jeda antara
// commit order dan publish ke Kafka yang bisa bikin event hilang kalau
// proses crash tepat di celah itu; di sini "mempublish" cukup berarti
// INSERT satu baris di tabel yang sama, jadi ATOMIC terhadap penyimpanan
// order itu sendiri.
func SaveOrderWithOutbox(ctx context.Context, db *pgxpool.Pool, order *Order) error {
	tx, err := db.Begin(ctx)
	if err != nil {
		return fmt.Errorf("begin tx: %w", err)
	}
	defer tx.Rollback(ctx)

	if err := tx.QueryRow(ctx,
		`INSERT INTO orders (user_id, status, total, version)
		 VALUES ($1, 'pending', $2, 1)
		 RETURNING id, user_id, status, total, version`,
		order.UserID, order.Total,
	).Scan(&order.ID, &order.UserID, &order.Status, &order.Total, &order.Version); err != nil {
		return fmt.Errorf("insert order: %w", err)
	}

	payload, err := json.Marshal(orderCreatedPayload{
		EventID:   uuid.NewString(),
		OrderID:   order.ID,
		UserID:    order.UserID,
		Total:     order.Total,
		CreatedAt: time.Now().UTC(),
	})
	if err != nil {
		return fmt.Errorf("marshal outbox payload: %w", err)
	}

	// outbox_events di-INSERT dalam transaction YANG SAMA dengan orders di
	// atas -- kalau tx.Commit di bawah gagal, order maupun event outbox-nya
	// SAMA-SAMA gak tersimpan; kalau berhasil, keduanya SAMA-SAMA tersimpan.
	if _, err := tx.Exec(ctx,
		`INSERT INTO outbox_events (aggregate_type, aggregate_id, event_type, payload, created_at)
		 VALUES ('order', $1, 'OrderCreated', $2, now())`,
		order.ID, payload,
	); err != nil {
		return fmt.Errorf("insert outbox event: %w", err)
	}

	if err := tx.Commit(ctx); err != nil {
		return fmt.Errorf("commit tx: %w", err)
	}
	return nil
}

// RelayPendingOutboxEvents membaca baris outbox_events yang belum
// di-publish, mempublish payload-nya apa adanya ke Kafka, lalu menandai
// published_at -- proses TERPISAH ini (dijalankan berkala, bukan bagian
// dari request checkout) yang benar-benar mengirim event ke Kafka.
// FOR UPDATE SKIP LOCKED penting kalau ada lebih dari satu instance relay
// berjalan bersamaan, supaya dua instance gak sama-sama mengambil baris
// yang sama untuk dipublish dua kali.
func RelayPendingOutboxEvents(ctx context.Context, db *pgxpool.Pool, w *kafka.Writer) error {
	tx, err := db.Begin(ctx)
	if err != nil {
		return fmt.Errorf("begin relay tx: %w", err)
	}
	defer tx.Rollback(ctx)

	rows, err := tx.Query(ctx,
		`SELECT id, aggregate_id, payload FROM outbox_events
		 WHERE published_at IS NULL
		 ORDER BY id
		 LIMIT 100
		 FOR UPDATE SKIP LOCKED`,
	)
	if err != nil {
		return fmt.Errorf("query pending outbox events: %w", err)
	}

	type pendingEvent struct {
		id            int64
		aggregateID   int64
		payload       []byte
	}
	var pending []pendingEvent
	for rows.Next() {
		var e pendingEvent
		if err := rows.Scan(&e.id, &e.aggregateID, &e.payload); err != nil {
			rows.Close()
			return fmt.Errorf("scan outbox event: %w", err)
		}
		pending = append(pending, e)
	}
	rows.Close()

	for _, e := range pending {
		if err := w.WriteMessages(ctx, kafka.Message{
			Key:   []byte(fmt.Sprintf("order-%d", e.aggregateID)),
			Value: e.payload,
		}); err != nil {
			return fmt.Errorf("publish outbox event %d: %w", e.id, err)
		}
		if _, err := tx.Exec(ctx,
			`UPDATE outbox_events SET published_at = now() WHERE id = $1`, e.id,
		); err != nil {
			return fmt.Errorf("mark outbox event %d published: %w", e.id, err)
		}
	}

	if err := tx.Commit(ctx); err != nil {
		return fmt.Errorf("commit relay tx: %w", err)
	}
	return nil
}
```

### Contoh Kode — Node.js
```javascript
const { randomUUID } = require('crypto');

// saveOrderWithOutbox menyimpan order baru DAN sebuah baris outbox_events
// dalam SATU transaction Postgres yang sama -- kalau salah satu INSERT
// gagal, keduanya di-rollback bareng. Ini menutup celah yang ada di
// createOrderAndPublish (Phase 05, topik 48), di mana ada jeda antara
// commit order dan publish ke Kafka yang bisa bikin event hilang kalau
// proses crash tepat di celah itu.
async function saveOrderWithOutbox(pool, order) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    const orderResult = await client.query(
      `INSERT INTO orders (user_id, status, total, version)
       VALUES ($1, 'pending', $2, 1)
       RETURNING id, user_id, status, total, version`,
      [order.userId, order.total]
    );
    const savedOrder = orderResult.rows[0];

    const payload = JSON.stringify({
      eventId: randomUUID(),
      orderId: savedOrder.id,
      userId: savedOrder.user_id,
      total: savedOrder.total,
      createdAt: new Date().toISOString(),
    });

    // outbox_events di-INSERT dalam transaction YANG SAMA dengan orders di
    // atas -- kalau COMMIT di bawah gagal, order maupun event outbox-nya
    // sama-sama gak tersimpan.
    await client.query(
      `INSERT INTO outbox_events (aggregate_type, aggregate_id, event_type, payload, created_at)
       VALUES ('order', $1, 'OrderCreated', $2, now())`,
      [savedOrder.id, payload]
    );

    await client.query('COMMIT');
    return savedOrder;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

// relayPendingOutboxEvents membaca baris outbox_events yang belum
// di-publish, mempublish payload-nya apa adanya ke Kafka, lalu menandai
// published_at -- proses TERPISAH ini (dijalankan berkala) yang benar-benar
// mengirim event ke Kafka. FOR UPDATE SKIP LOCKED penting kalau ada lebih
// dari satu instance relay berjalan bersamaan.
async function relayPendingOutboxEvents(pool, producer) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const { rows } = await client.query(
      `SELECT id, aggregate_id, payload FROM outbox_events
       WHERE published_at IS NULL
       ORDER BY id
       LIMIT 100
       FOR UPDATE SKIP LOCKED`
    );

    for (const row of rows) {
      const value = typeof row.payload === 'string' ? row.payload : JSON.stringify(row.payload);
      await producer.send({
        topic: 'order.created',
        messages: [{ key: `order-${row.aggregate_id}`, value }],
      });
      await client.query('UPDATE outbox_events SET published_at = now() WHERE id = $1', [row.id]);
    }

    await client.query('COMMIT');
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

module.exports = { saveOrderWithOutbox, relayPendingOutboxEvents };
```

### Trade-off & Pitfall
- Outbox Pattern menambah **latency publish**: event gak lagi terkirim ke Kafka SAAT ITU JUGA seperti `PublishOrderCreated` (Phase 05), tapi menunggu relay berikutnya jalan (polling interval, misalnya tiap 1 detik) — untuk kasus yang butuh event sampai ke consumer secepat mungkin, ini trade-off yang perlu disadari.
- `RelayPendingOutboxEvents` menambah satu proses/komponen baru yang perlu dijalankan terus-menerus dan dipantau kesehatannya — kalau relay ini sendiri berhenti berjalan (bukan crash di tengah proses, tapi memang berhenti total), outbox_events akan terus menumpuk tanpa pernah dipublish, walau datanya sendiri aman tersimpan.
- Tabel `outbox_events` akan terus bertambah baris seiring waktu — perlu proses pembersihan berkala (misalnya hapus baris yang `published_at` sudah lebih dari beberapa hari) supaya tabel gak membengkak tanpa batas.
- Outbox Pattern menjamin event PASTI dipublish (minimal sekali), tapi TIDAK menjamin exactly-once — `RelayPendingOutboxEvents` bisa saja mem-publish baris yang sama dua kali kalau proses crash tepat setelah `WriteMessages` tapi sebelum `UPDATE published_at`; consumer di sisi lain tetap butuh idempotent consumer (topik 50, Phase 05 / topik 101 di phase ini).

### Kapan Dipakai
Wajib dipakai ketika sebuah perubahan database HARUS selalu diikuti event yang terpublish, tanpa kemungkinan salah satu tersimpan sementara yang lain hilang — order yang dibuat tapi event `OrderCreated`-nya gak pernah terkirim berarti `payment-service` dan `notification-service` gak akan pernah tau ada order itu sama sekali. Untuk event yang sifatnya "boleh hilang sesekali" (misalnya event analytics non-kritis), latency tambahan dari outbox biasanya gak sepadan — publish langsung seperti Phase 05 sudah cukup.

### Sering Ditanya Saat Interview
- "Apa masalah konkret yang diselesaikan Outbox Pattern dibanding CreateOrderAndPublish di Phase 05?" — CreateOrderAndPublish commit database dan publish Kafka sebagai DUA operasi terpisah yang gak atomic; kalau proses crash di antara keduanya, order tersimpan tapi event-nya hilang selamanya. Outbox Pattern membuat "mempublish" jadi cukup INSERT satu baris di database yang sama, sehingga atomic terhadap penyimpanan order lewat transaction Postgres biasa.
- "Kenapa perlu proses relay terpisah, bukan langsung publish ke Kafka di dalam transaction yang sama?" — Kafka bukan bagian dari transaction Postgres, jadi gak bisa ikut di-rollback atau dijamin atomic bareng INSERT SQL; relay yang membaca outbox_events SETELAH transaction commit adalah cara memindahkan tanggung jawab "benar-benar mengirim ke Kafka" ke proses yang bisa retry sebebas mungkin tanpa mengorbankan atomicity penyimpanan datanya.
- "Kenapa `RelayPendingOutboxEvents` pakai FOR UPDATE SKIP LOCKED?" — supaya kalau ada lebih dari satu instance relay berjalan bersamaan (untuk scaling atau high availability), tiap instance mengambil baris outbox_events yang BERBEDA untuk diproses, bukan sama-sama mengambil baris yang sama dan mempublish event yang sama dua kali secara bersamaan.

---

## 101. Distributed Idempotency

### Apa itu?
Distributed Idempotency adalah memastikan sebuah operasi yang melibatkan LEBIH dari satu sistem (Redis, Postgres, payment provider eksternal) tetap aman dijalankan berkali-kali dengan input yang sama, di SETIAP lapisan sistem itu — bukan cuma di satu titik. Di OrderFlow, `IsEventProcessed` (Phase 05) melindungi di lapisan MESSAGE (event Kafka yang sama gak diproses dua kali oleh consumer), sementara kolom `payments.idempotency_key` melindungi di lapisan PAYMENT PROVIDER (charge yang sama gak dikirim dua kali ke provider eksternal) — keduanya harus dipasang BERSAMA karena masing-masing menutup celah yang berbeda.

### Kenapa dibutuhkan?
`IsEventProcessed` (topik 50, Phase 05) menandai sebuah `eventID` sebagai "processed" SEBELUM handler-nya (di sini: proses pembayaran) dipastikan benar-benar sukses. Itu artinya kalau `CallPaymentProviderWithCircuitBreaker` (Phase 09) gagal di tengah jalan — misalnya request ke provider timeout PADAHAL charge-nya sebenarnya sudah berhasil di sisi provider — event itu tetap dianggap "sudah diproses" dan gak akan di-retry otomatis lewat redelivery Kafka. Kalau retry-nya dilakukan lewat jalur LAIN (admin retry manual, atau proses recovery setelah incident), `CallPaymentProviderWithCircuitBreaker` bisa terpanggil KEDUA KALINYA untuk order yang sama — dan tanpa `idempotency_key` yang deterministic, payment provider akan menganggapnya sebagai charge baru, alias customer di-charge dua kali. `idempotency_key` yang deterministic dari `orderID` (bukan dari `eventID` yang beda-beda tiap redelivery) memastikan SEMUA jalur retry, apa pun bentuknya, selalu mengirim key yang sama ke provider.

### Cara Kerja
```
Lapisan 1 -- MESSAGE (IsEventProcessed, Redis, Phase 05):
  mencegah event Kafka yang SAMA PERSIS (eventID sama) diproses dua kali
  akibat redelivery (at-least-once, topik 49).

Lapisan 2 -- PAYMENT PROVIDER (idempotency_key, Postgres):
  idempotencyKey := fmt.Sprintf("order-%d-payment", orderID)   <- deterministic
                                                                    dari orderID,
                                                                    BUKAN eventID

  ProcessPaymentForOrder(event):
    IsEventProcessed(eventID)?           --> true  --> skip (lapisan 1)
                                          --> false --> lanjut
    SELECT ... WHERE idempotency_key = idempotencyKey?
                                          --> ADA   --> skip (lapisan 2, sudah pernah charge)
                                          --> TIDAK ADA --> lanjut
    CallPaymentProviderWithCircuitBreaker(token: idempotencyKey)
    INSERT INTO payments (..., idempotency_key) ON CONFLICT DO NOTHING

Kenapa perlu DUA lapisan: kalau proses crash SETELAH charge sukses tapi
SEBELUM INSERT payments selesai, redelivery event yang sama akan LOLOS
lapisan 1 (event dianggap belum processed di kasus tertentu) TAPI tetap
ditahan lapisan 2 (idempotency_key sudah/segera tercatat) -- kedua lapisan
saling menutupi celah yang gak bisa ditutup satu lapisan saja.
```

### Contoh Kode — Go
```go
package paymentprocessing

import (
	"context"
	"errors"
	"fmt"

	"github.com/jackc/pgx/v5"
	"github.com/jackc/pgx/v5/pgxpool"
	"github.com/redis/go-redis/v9"

	"orderflow/internal/messaging"
	"orderflow/internal/payment"
)

// ProcessPaymentForOrder menggabungkan DUA lapisan idempotency yang berbeda:
// IsEventProcessed (Phase 05, level MESSAGE di Redis) mencegah event Kafka
// yang sama diproses dua kali; idempotencyKey (level PAYMENT PROVIDER,
// disimpan di kolom payments.idempotency_key) mencegah charge dobel walau
// CallPaymentProviderWithCircuitBreaker terpanggil lebih dari satu kali
// untuk order yang sama lewat jalur LAIN (misalnya retry manual, bukan
// cuma redelivery Kafka).
func ProcessPaymentForOrder(ctx context.Context, rdb *redis.Client, db *pgxpool.Pool, event messaging.OrderEvent) error {
	processed, err := messaging.IsEventProcessed(ctx, rdb, event.EventID)
	if err != nil {
		return fmt.Errorf("check event processed: %w", err)
	}
	if processed {
		return nil
	}

	// idempotencyKey deterministic dari orderID (BUKAN dari event.EventID) --
	// supaya jalur retry APAPUN untuk order yang sama (redelivery Kafka,
	// retry manual, dst) selalu mengirim key yang SAMA ke payment provider.
	idempotencyKey := fmt.Sprintf("order-%d-payment", event.OrderID)

	var existing string
	err = db.QueryRow(ctx,
		`SELECT idempotency_key FROM payments WHERE idempotency_key = $1`,
		idempotencyKey,
	).Scan(&existing)
	if err == nil {
		// payment dengan idempotency_key ini sudah pernah tercatat -- order
		// ini sudah pernah di-charge, jangan charge lagi.
		return nil
	}
	if !errors.Is(err, pgx.ErrNoRows) {
		return fmt.Errorf("check existing payment: %w", err)
	}

	resp, err := payment.CallPaymentProviderWithCircuitBreaker(ctx, payment.PaymentRequest{
		OrderID: event.OrderID,
		Amount:  event.Total,
		Token:   idempotencyKey,
	})
	if err != nil {
		return fmt.Errorf("call payment provider: %w", err)
	}

	if _, err := db.Exec(ctx,
		`INSERT INTO payments (order_id, idempotency_key, transaction_id, status)
		 VALUES ($1, $2, $3, $4)
		 ON CONFLICT (idempotency_key) DO NOTHING`,
		event.OrderID, idempotencyKey, resp.TransactionID, resp.Status,
	); err != nil {
		return fmt.Errorf("insert payment: %w", err)
	}

	return nil
}
```

### Contoh Kode — Node.js
```javascript
const { isEventProcessed } = require('./idempotent-consumer'); // dari Phase 05
const { callPaymentProviderWithCircuitBreaker } = require('./payment-circuit-breaker'); // dari Phase 09

// processPaymentForOrder menggabungkan DUA lapisan idempotency yang
// berbeda: isEventProcessed (Phase 05, level MESSAGE di Redis) mencegah
// event Kafka yang sama diproses dua kali; idempotencyKey (level PAYMENT
// PROVIDER, disimpan di kolom payments.idempotency_key) mencegah charge
// dobel walau callPaymentProviderWithCircuitBreaker terpanggil lebih dari
// satu kali untuk order yang sama lewat jalur lain (retry manual, dst).
async function processPaymentForOrder(redisClient, pool, event) {
  const alreadyProcessed = await isEventProcessed(redisClient, event.eventId);
  if (alreadyProcessed) {
    return;
  }

  // idempotencyKey deterministic dari orderId (BUKAN dari event.eventId) --
  // supaya jalur retry apa pun untuk order yang sama selalu mengirim key
  // yang sama ke payment provider.
  const idempotencyKey = `order-${event.orderId}-payment`;

  const existing = await pool.query(
    'SELECT idempotency_key FROM payments WHERE idempotency_key = $1',
    [idempotencyKey]
  );
  if (existing.rows.length > 0) {
    // payment dengan idempotencyKey ini sudah pernah tercatat -- skip.
    return;
  }

  const result = await callPaymentProviderWithCircuitBreaker({
    orderId: event.orderId,
    amount: event.total,
    token: idempotencyKey,
  });

  await pool.query(
    `INSERT INTO payments (order_id, idempotency_key, transaction_id, status)
     VALUES ($1, $2, $3, $4)
     ON CONFLICT (idempotency_key) DO NOTHING`,
    [event.orderId, idempotencyKey, result.transactionId, result.status]
  );
}

module.exports = { processPaymentForOrder };
```

### Trade-off & Pitfall
- Dua lapisan idempotency berarti dua tempat yang bisa salah konfigurasi — kalau `idempotencyKey` ternyata dibuat dari `event.EventID` (yang beda tiap redelivery) alih-alih dari `orderID` (tetap sama), seluruh manfaat lapisan kedua hilang karena tiap redelivery akan dianggap key yang baru.
- Ada jendela kecil antara `SELECT ... WHERE idempotency_key = ...` dan `INSERT ... ON CONFLICT DO NOTHING` di mana dua proses bisa sama-sama lolos SELECT (belum ada baris) sebelum salah satunya sempat INSERT — `ON CONFLICT DO NOTHING` (butuh unique constraint di kolom `idempotency_key`) adalah pengaman terakhir yang WAJIB ada, SELECT di awal cuma optimisasi supaya gak perlu memanggil payment provider kalau sudah jelas-jelas pernah diproses.
- Constraint UNIQUE di `payments.idempotency_key` sendiri gak melindungi dari kegagalan SETELAH charge sukses tapi SEBELUM INSERT tercatat (crash di antara keduanya) — kasus ini butuh reconciliation job terpisah yang mencocokkan status di payment provider dengan tabel `payments` OrderFlow secara berkala.

### Kapan Dipakai
Wajib dipasang di setiap operasi yang (a) melibatkan lebih dari satu sistem yang gak bisa dibungkus satu transaction, DAN (b) efek sampingnya berbahaya kalau dijalankan dua kali — charge payment adalah contoh paling jelas karena "dijalankan dua kali" berarti customer kehilangan uang. Untuk operasi yang idempotent secara alami di semua lapisannya (misalnya `UPDATE ... SET status = 'notified'` yang hasilnya sama walau dijalankan berkali-kali), lapisan kedua ini opsional.

### Sering Ditanya Saat Interview
- "Kenapa gak cukup pakai IsEventProcessed dari Phase 05 aja?" — IsEventProcessed cuma melindungi dari redelivery Kafka SPESIFIK (eventID yang sama persis); kalau retry-nya datang lewat jalur lain (misalnya admin memicu retry manual dengan orderID yang sama tapi eventID baru), IsEventProcessed gak akan pernah tau itu adalah percobaan kedua untuk order yang sama.
- "Kenapa idempotencyKey dibuat dari orderID, bukan dari eventID?" — eventID berbeda setiap kali Kafka redeliver atau setiap kali proses lain membuat event baru untuk order yang sama, sementara orderID tetap konsisten; idempotencyKey harus deterministic terhadap MAKNA operasinya (charge order X), bukan terhadap pembawa pesannya (event mana yang memicu).
- "Apa fungsi ON CONFLICT DO NOTHING di sini?" — sebagai pengaman terakhir terhadap race condition: kalau dua proses somehow sama-sama lolos pengecekan SELECT sebelum salah satunya sempat INSERT, unique constraint di idempotency_key memastikan cuma satu baris payment yang benar-benar tersimpan untuk key yang sama, INSERT kedua gagal diam-diam (DO NOTHING) alih-alih membuat duplikat.

---

## 102. Webhooks

### Apa itu?
Webhook adalah kebalikan dari API biasa: alih-alih OrderFlow yang memanggil sistem lain, sistem lain (payment provider) yang memanggil OrderFlow lewat HTTP POST begitu ada kejadian di sisi mereka (misalnya "pembayaran berhasil dikonfirmasi"). Karena endpoint webhook OrderFlow terbuka ke internet tanpa mekanisme login/token biasa, `VerifyWebhookSignature` memverifikasi bahwa request itu benar-benar berasal dari payment provider (bukan pihak lain yang mengaku-ngaku) lewat HMAC signature yang dikirim di header request.

### Kenapa dibutuhkan?
Endpoint webhook OrderFlow (misalnya `POST /webhooks/payment`) HARUS bisa diakses publik tanpa API key biasa, karena payment provider yang memanggilnya, bukan client OrderFlow yang sudah login. Tanpa verifikasi apa pun, siapa saja yang tau URL endpoint ini bisa mengirim payload palsu "payment berhasil" untuk order siapa pun, dan kalau OrderFlow langsung mempercayainya, produk bisa "terbayar" tanpa uang benar-benar berpindah. Payment provider (dan penyedia webhook pada umumnya) menandatangani tiap payload dengan HMAC memakai shared secret yang cuma diketahui OrderFlow dan provider — `VerifyWebhookSignature` menghitung ulang HMAC itu dan membandingkannya dengan signature yang dikirim, menolak request kalau gak cocok.

### Cara Kerja
```
Payment provider (server mereka)                 OrderFlow (VerifyWebhookSignature)
        |                                                    |
   hitung HMAC-SHA256(payload, secret)                       |
        |                                                    |
   POST /webhooks/payment                                    |
   body: payload mentah                                      |
   header X-Signature: <hasil HMAC>  ------------------->     |
                                                    hitung ULANG HMAC-SHA256
                                                    (payload, secret) dari body
                                                    yang diterima
                                                               |
                                          bandingkan hasil hitungan sendiri
                                          dengan header X-Signature PAKAI
                                          constant-time compare (hmac.Equal /
                                          crypto.timingSafeEqual) -- BUKAN
                                          == atau === biasa
                                                               |
                          cocok -----> proses payload (update status payment)
                          gak cocok -> 401, payload DITOLAK mentah-mentah

Kenapa constant-time compare wajib: perbandingan == / === berhenti di byte
pertama yang beda, jadi WAKTU eksekusinya sedikit berbeda tergantung berapa
banyak byte awal yang kebetulan cocok -- penyerang yang teliti secara
teoritis bisa menebak signature valid byte demi byte lewat selisih waktu
respons itu (timing attack). hmac.Equal/timingSafeEqual selalu membandingkan
SELURUH panjang input dalam waktu yang konstan, gak peduli di mana letak
byte yang beda.
```

### Contoh Kode — Go
```go
package webhook

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"io"
	"net/http"
)

// VerifyWebhookSignature memverifikasi signature webhook dari payment
// provider: HMAC-SHA256 dari payload memakai secret, dibandingkan dengan
// signature yang dikirim provider di header (biasanya X-Signature).
// hmac.Equal dipakai (BUKAN ==) supaya perbandingannya constant-time --
// kalau pakai == atau bytes.Equal biasa, waktu komparasi bergantung pada di
// mana byte pertama yang beda ditemukan, yang secara teoritis bisa
// dieksploitasi penyerang buat menebak signature valid byte demi byte
// (timing attack).
func VerifyWebhookSignature(payload []byte, signature, secret string) bool {
	mac := hmac.New(sha256.New, []byte(secret))
	mac.Write(payload)
	expected := hex.EncodeToString(mac.Sum(nil))

	return hmac.Equal([]byte(expected), []byte(signature))
}

type paymentWebhookEvent struct {
	OrderID       int64  `json:"order_id"`
	TransactionID string `json:"transaction_id"`
	Status        string `json:"status"`
}

// PaymentWebhookHandler membaca body MENTAH dulu (io.ReadAll) sebelum
// decode JSON apa pun, karena VerifyWebhookSignature harus menghitung HMAC
// dari byte PERSIS seperti yang dikirim provider -- decode-lalu-encode-ulang
// bisa menghasilkan byte yang berbeda (urutan key, spasi) dan bikin
// signature gak akan pernah cocok.
func PaymentWebhookHandler(secret string, handleEvent func(paymentWebhookEvent) error) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		body, err := io.ReadAll(r.Body)
		if err != nil {
			http.Error(w, "cannot read body", http.StatusBadRequest)
			return
		}

		signature := r.Header.Get("X-Signature")
		if !VerifyWebhookSignature(body, signature, secret) {
			http.Error(w, "invalid signature", http.StatusUnauthorized)
			return
		}

		var event paymentWebhookEvent
		if err := json.Unmarshal(body, &event); err != nil {
			http.Error(w, "invalid payload", http.StatusBadRequest)
			return
		}

		if err := handleEvent(event); err != nil {
			http.Error(w, "internal error", http.StatusInternalServerError)
			return
		}
		w.WriteHeader(http.StatusOK)
	}
}
```

### Contoh Kode — Node.js
```javascript
const crypto = require('crypto');

// verifyWebhookSignature menghitung HMAC-SHA256 dari payload memakai
// secret, lalu membandingkannya dengan signature yang dikirim provider
// pakai crypto.timingSafeEqual -- BUKAN === -- supaya perbandingannya
// constant-time dan gak bocorin informasi lewat timing side-channel.
function verifyWebhookSignature(payload, signature, secret) {
  const expected = crypto.createHmac('sha256', secret).update(payload).digest('hex');

  const expectedBuf = Buffer.from(expected, 'utf8');
  const signatureBuf = Buffer.from(signature, 'utf8');

  // timingSafeEqual MELEMPAR error kalau panjang buffer beda -- signature
  // yang salah/dipalsukan bisa punya panjang berbeda, jadi panjang harus
  // dicek dulu SEBELUM timingSafeEqual (perbandingan panjang sendiri bukan
  // bagian yang perlu constant-time, karena panjang hex digest bukan rahasia).
  if (expectedBuf.length !== signatureBuf.length) {
    return false;
  }

  return crypto.timingSafeEqual(expectedBuf, signatureBuf);
}

module.exports = { verifyWebhookSignature };
```
Wiring-nya butuh body MENTAH (Buffer), bukan hasil parsing `express.json()`, karena `verifyWebhookSignature` harus menghitung HMAC dari byte persis seperti yang dikirim provider:
```javascript
// webhook-route.js
const express = require('express');
const { verifyWebhookSignature } = require('./webhook-signature');

function buildWebhookRoute(app, secret, handlePaymentEvent) {
  // express.raw() penting -- kalau pakai express.json(), body sudah
  // di-parse jadi object dan di-serialize ulang bisa menghasilkan byte
  // yang beda (urutan key, spasi) dari body ASLI yang di-sign provider.
  app.post(
    '/webhooks/payment',
    express.raw({ type: 'application/json' }),
    (req, res) => {
      const signature = req.get('X-Signature') ?? '';
      if (!verifyWebhookSignature(req.body, signature, secret)) {
        return res.status(401).json({ error: 'invalid signature' });
      }

      const event = JSON.parse(req.body.toString('utf8'));
      handlePaymentEvent(event)
        .then(() => res.sendStatus(200))
        .catch(() => res.status(500).json({ error: 'internal error' }));
    }
  );
}

module.exports = { buildWebhookRoute };
```

### Trade-off & Pitfall
- Kalau perbandingan signature dilakukan pakai `==`/`===` biasa alih-alih `hmac.Equal`/`timingSafeEqual`, itu bukan cuma soal gaya kode — perbandingan non-constant-time membuka celah timing attack yang secara teoritis bisa dieksploitasi buat memalsukan signature valid; ini termasuk bug keamanan nyata, bukan sekadar micro-optimization yang boleh dilewatkan.
- Body request WAJIB dibaca dalam bentuk mentah (raw bytes) sebelum di-parse JSON — kalau framework HTTP sudah keburu men-decode-lalu-encode-ulang body sebelum sampai ke handler webhook, HMAC yang dihitung ulang OrderFlow gak akan pernah cocok dengan yang dihitung provider, walau payload-nya "sama" secara logis.
- Endpoint webhook idealnya tetap idempotent di sisi consumer-nya juga (topik 101 di atas) — payment provider sering mengirim webhook yang sama lebih dari sekali (mirip at-least-once, topik 49 Phase 05) kalau OrderFlow gagal membalas 200 tepat waktu, jadi `handleEvent` sebaiknya aman dipanggil berkali-kali untuk `transaction_id` yang sama.
- Secret HMAC yang dipakai buat verifikasi harus disimpan sebagai environment variable/secret manager, BUKAN hardcoded di kode — kalau secret ini bocor, siapa pun bisa menghitung signature valid untuk payload palsu apa pun.

### Kapan Dipakai
Setiap kali OrderFlow menerima notifikasi dari sistem eksternal lewat HTTP callback (payment provider, shipping provider, dst) yang gak melalui autentikasi user/API key biasa — verifikasi signature adalah syarat wajib, bukan opsional, untuk endpoint semacam ini.

### Sering Ditanya Saat Interview
- "Kenapa gak pakai == biasa buat bandingin signature, kan lebih sederhana?" — == berhenti membandingkan begitu ketemu byte pertama yang beda, sehingga waktu eksekusinya bocorin informasi seberapa banyak byte awal yang cocok; penyerang yang sabar secara teoritis bisa memanfaatkan selisih waktu itu buat menebak signature valid byte demi byte. hmac.Equal/timingSafeEqual selalu makan waktu yang sama berapa pun jumlah byte yang cocok.
- "Kenapa handler webhook butuh body mentah, bukan body yang sudah di-parse JSON?" — signature yang dikirim provider dihitung dari byte PERSIS yang mereka kirim; kalau OrderFlow menghitung ulang HMAC dari hasil serialize ulang JSON yang sudah di-parse (urutan key atau spasi bisa berbeda), hasil HMAC-nya gak akan cocok walau isi datanya "sama".
- "Kenapa endpoint webhook perlu idempotent juga?" — karena provider webhook sering mengirim event yang sama berkali-kali sampai mereka menerima response 200 (mirip semantik at-least-once di message queue, topik 49 Phase 05), jadi memproses `transaction_id` yang sama dua kali gak boleh menyebabkan efek samping ganda.

---

## 103. File Upload

### Apa itu?
File Upload di sini spesifik soal validasi dan penyimpanan gambar produk yang di-upload seller ke OrderFlow — bukan cuma "terima file dan simpan", tapi memastikan file yang diterima benar-benar gambar (bukan file executable yang disamarkan namanya), gak melebihi batas ukuran, dan disimpan dengan nama yang gak bisa dipakai buat menyerang filesystem/object storage. `UploadProductImage` membungkus semua validasi ini sebelum benar-benar meng-upload ke object storage (S3-compatible).

### Kenapa dibutuhkan?
Menerima file upload dari user (bahkan seller yang sudah terautentikasi) berarti menerima input yang paling gampang dipalsukan: nama file bisa apa saja, header `Content-Type` yang dikirim client gampang dipalsukan (client bisa bilang "image/png" padahal isinya script), dan ukuran file yang gak dibatasi bisa dipakai buat menghabiskan storage/bandwidth (denial of service sederhana). Tanpa validasi ini, seller (atau penyerang yang menyamar sebagai seller) bisa upload file berbahaya yang nantinya diakses lewat URL publik CDN, atau membanjiri storage dengan file raksasa. `UploadProductImage` mendeteksi tipe file dari ISI file (magic bytes), bukan dari nama atau header yang dikirim client, dan mengganti nama file jadi UUID supaya gak ada path traversal (`../../etc/passwd`) atau filename yang bentrok antar produk.

### Cara Kerja
```
UploadProductImage(ctx, file, filename):
        |
        v
  cek ekstensi filename (.jpg/.jpeg/.png/.webp)  --> gak diizinkan? TOLAK
        |
        v
  baca isi file, batasi ukuran maksimum (5MB)    --> kelebihan?      TOLAK
        |
        v
  deteksi content-type dari ISI file (magic bytes,
  BUKAN dari header Content-Type yang dikirim client)
        |
        v
  bukan image/* ?                                --> TOLAK
        |
        v
  generate nama object baru: products/<uuid>.<ext>
  (BUKAN memakai filename asli -- mencegah path traversal & filename bentrok)
        |
        v
  upload ke object storage (S3-compatible)
        |
        v
  return URL publik: https://cdn.orderflow.example.com/products/<uuid>.<ext>
```

### Contoh Kode — Go
```go
package upload

import (
	"bytes"
	"context"
	"fmt"
	"io"
	"net/http"
	"path/filepath"
	"strings"

	"github.com/aws/aws-sdk-go-v2/feature/s3/manager"
	"github.com/aws/aws-sdk-go-v2/service/s3"
	"github.com/google/uuid"
)

const (
	maxImageSizeBytes = 5 * 1024 * 1024 // 5MB
	bucketName        = "orderflow-product-images"
)

var allowedImageExtensions = map[string]bool{
	".jpg":  true,
	".jpeg": true,
	".png":  true,
	".webp": true,
}

// uploader adalah package-level S3 upload client, di-set sekali saat
// startup lewat SetUploader -- pola yang sama seperti paymentCircuit di
// topik 79 (Phase 09), supaya UploadProductImage bisa punya signature
// sederhana tanpa perlu mengoper client di tiap pemanggilan.
var uploader *manager.Uploader

func SetUploader(u *manager.Uploader) {
	uploader = u
}

// UploadProductImage memvalidasi lalu meng-upload gambar produk ke S3: cek
// ekstensi filename, batasi ukuran file, deteksi content-type dari ISI file
// (bukan dari header Content-Type yang gampang dipalsukan client), lalu
// upload dengan nama object yang di-generate ulang (UUID) supaya gak ada
// path traversal atau filename yang bentrok antar produk.
func UploadProductImage(ctx context.Context, file io.Reader, filename string) (string, error) {
	ext := strings.ToLower(filepath.Ext(filename))
	if !allowedImageExtensions[ext] {
		return "", fmt.Errorf("extension %q not allowed", ext)
	}

	limited := io.LimitReader(file, maxImageSizeBytes+1)
	content, err := io.ReadAll(limited)
	if err != nil {
		return "", fmt.Errorf("read file: %w", err)
	}
	if len(content) > maxImageSizeBytes {
		return "", fmt.Errorf("file exceeds max size of %d bytes", maxImageSizeBytes)
	}

	contentType := http.DetectContentType(content)
	if !strings.HasPrefix(contentType, "image/") {
		return "", fmt.Errorf("file is not an image (detected %q)", contentType)
	}

	key := fmt.Sprintf("products/%s%s", uuid.NewString(), ext)
	bucket := bucketName
	if _, err := uploader.Upload(ctx, &s3.PutObjectInput{
		Bucket:      &bucket,
		Key:         &key,
		Body:        bytes.NewReader(content),
		ContentType: &contentType,
	}); err != nil {
		return "", fmt.Errorf("upload to storage: %w", err)
	}

	return fmt.Sprintf("https://cdn.orderflow.example.com/%s", key), nil
}
```

### Contoh Kode — Node.js
```javascript
const crypto = require('crypto');
const path = require('path');
const { fileTypeFromBuffer } = require('file-type');
const { PutObjectCommand } = require('@aws-sdk/client-s3');

const MAX_IMAGE_SIZE_BYTES = 5 * 1024 * 1024; // 5MB
const BUCKET_NAME = 'orderflow-product-images';
const ALLOWED_EXTENSIONS = new Set(['.jpg', '.jpeg', '.png', '.webp']);

// s3Client adalah module-level S3 client, di-set sekali saat startup lewat
// setS3Client -- sama seperti pattern package-level client di versi Go,
// supaya uploadProductImage bisa punya signature sederhana (fileBuffer,
// filename) tanpa perlu mengoper client di tiap pemanggilan.
let s3Client;

function setS3Client(client) {
  s3Client = client;
}

// uploadProductImage memvalidasi lalu meng-upload gambar produk ke S3: cek
// ekstensi filename, batasi ukuran buffer, deteksi content-type dari ISI
// file (bukan dari header yang gampang dipalsukan client), lalu upload
// dengan nama object yang di-generate ulang (UUID) supaya gak ada path
// traversal atau filename yang bentrok antar produk.
async function uploadProductImage(fileBuffer, filename) {
  const ext = path.extname(filename).toLowerCase();
  if (!ALLOWED_EXTENSIONS.has(ext)) {
    throw new Error(`extension ${ext} not allowed`);
  }

  if (fileBuffer.length > MAX_IMAGE_SIZE_BYTES) {
    throw new Error(`file exceeds max size of ${MAX_IMAGE_SIZE_BYTES} bytes`);
  }

  const detected = await fileTypeFromBuffer(fileBuffer);
  if (!detected || !detected.mime.startsWith('image/')) {
    throw new Error(`file is not an image (detected ${detected ? detected.mime : 'unknown'})`);
  }

  const key = `products/${crypto.randomUUID()}${ext}`;
  await s3Client.send(
    new PutObjectCommand({
      Bucket: BUCKET_NAME,
      Key: key,
      Body: fileBuffer,
      ContentType: detected.mime,
    })
  );

  return `https://cdn.orderflow.example.com/${key}`;
}

module.exports = { uploadProductImage, setS3Client };
```

### Trade-off & Pitfall
- Deteksi tipe file dari magic bytes (`http.DetectContentType` / `file-type`) jauh lebih aman daripada percaya `Content-Type` header dari client, tapi tetap bukan jaminan 100% — file yang sengaja dibuat menyerupai format image di header-nya (polyglot file) secara teoritis masih bisa lolos deteksi ini; validasi tambahan seperti scanning malware terpisah dibutuhkan untuk sistem dengan risiko tinggi.
- Membatasi ukuran file lewat `io.LimitReader`/pengecekan panjang buffer di level aplikasi itu penting, tapi idealnya dikombinasikan dengan limit di level infrastruktur (reverse proxy, load balancer) supaya request raksasa ditolak SEBELUM sempat membebani proses aplikasi sama sekali.
- Nama object yang di-generate ulang (UUID) menghilangkan filename asli seller sepenuhnya — kalau UX butuh menampilkan nama file asli (misalnya di halaman admin), nama itu perlu disimpan terpisah di database sebagai metadata, bukan dipakai langsung sebagai path penyimpanan.
- URL CDN publik yang dihasilkan berarti siapa pun yang tau/menebak UUID-nya bisa mengakses gambar itu — untuk gambar produk publik ini biasanya gak masalah, tapi pola yang sama TIDAK boleh dipakai buat file privat (misalnya invoice PDF) tanpa access control tambahan di lapisan CDN/storage.

### Kapan Dipakai
Setiap kali OrderFlow menerima file dari user (seller upload gambar produk, dst) yang nantinya disimpan dan diakses ulang — validasi tipe-dari-isi-file, batas ukuran, dan penggantian nama file adalah tiga langkah yang gak boleh dilewati, terlepas dari seberapa "terpercaya" seller yang meng-upload.

### Sering Ditanya Saat Interview
- "Kenapa deteksi tipe file harus dari isi file, bukan dari header Content-Type atau ekstensi nama file?" — keduanya sepenuhnya dikontrol client dan gampang dipalsukan (upload script `.php` dengan header `Content-Type: image/png`); magic bytes di awal isi file jauh lebih sulit dipalsukan tanpa benar-benar membuat file itu valid sebagai format gambar tersebut.
- "Kenapa nama file di-generate ulang jadi UUID, bukan dipakai apa adanya dari upload?" — filename asli bisa berisi karakter path traversal (`../../`) atau bentrok dengan produk lain yang kebetulan upload file dengan nama sama; UUID menjamin nama object selalu unik dan gak pernah berupa path yang bisa dimanipulasi.
- "Kenapa perlu membatasi ukuran file di level aplikasi kalau reverse proxy juga bisa membatasinya?" — dua lapisan ini melindungi hal yang beda: limit di reverse proxy mencegah request raksasa membebani jaringan/koneksi sebelum sampai ke aplikasi; limit di aplikasi (io.LimitReader) tetap dibutuhkan sebagai pengaman kalau limit di proxy longgar atau salah konfigurasi, supaya proses aplikasi sendiri gak ikut membaca file yang jauh lebih besar dari yang diharapkan ke memory.

---

**Selanjutnya:** [Phase 15 — Interview Thinking](./phase-15-interview-thinking.md)
