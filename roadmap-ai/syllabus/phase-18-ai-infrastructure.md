# Phase 18 — AI Infrastructure

> Bagian dari [AI Engineering / Agent Roadmap](../README.md)

---

## 68. Model Gateway

### Apa itu?
Model Gateway adalah evolusi dari `LLMGateway` yang dibangun di Phase 5 topik 20. Pola dasarnya sama:
```
Application → LLM Gateway → Model A / Model B / Model C
```
Bedanya, `LLMGateway` di Phase 5 cuma ngurus satu hal: dispatch ke provider yang benar (OpenAI vs Anthropic). Begitu SupportPilot beneran jalan di production dengan traffic nyata, gateway ini harus mikul tanggung jawab lebih banyak sekaligus di satu tempat: **routing** (pilih model yang tepat buat tiap task — topik 69), **fallback** (kalau model utama gagal, coba model cadangan), **cost tracking** (topik 22 di Phase 5, `track_usage`), **rate limiting** (jangan sampai satu proses nembak provider kelewat kenceng), **logging**, dan **caching** (topik 70). `ModelGateway` di sini adalah class baru yang mewujudkan semua tanggung jawab itu, menggantikan `LLMGateway` yang lebih sederhana.

### Kenapa dibutuhkan?
`LLMGateway` versi Phase 5 udah menyelesaikan masalah "jangan coupled ke satu provider SDK" — tapi begitu SupportPilot beroperasi di skala production, muncul masalah baru yang gak di-cover versi sederhana itu: gimana kalau model yang dipanggil lagi down atau timeout (butuh **fallback**)? Gimana kalau satu proses backend nembak API request kelewat sering sampai kena rate limit dari provider (butuh **rate limiting** di sisi kita sendiri, sebelum request-nya bahkan dikirim)? Semua kekhawatiran ini sifatnya **cross-cutting** — berlaku ke SEMUA request LLM, gak peduli endpoint atau fitur mana yang manggil. Kalau ditangani manual di tiap endpoint, bakal duplikasi logic yang sama berkali-kali dan gampang ada yang kelewat.

`ModelGateway` menyelesaikan ini dengan cara yang sama seperti `LLMGateway` menyelesaikan masalah coupling provider di Phase 5: **satu titik terpusat**. Bedanya sekarang titik terpusat itu ngurus lebih banyak concern sekaligus — begitu ditambahkan di `ModelGateway.generate()`, fallback/rate-limiting/cost-tracking otomatis berlaku ke SEMUA pemanggil, tanpa perlu diulang di tiap endpoint.

### Cara Kerja
```
Kode pemanggil (misal endpoint /chat, atau batch_process topik 71)
    → gateway.generate(messages, model=...)
        1. cek rate limit dulu — kalau kelewat batas, tolak request sebelum manggil provider
        2. coba panggil model utama
        3. kalau gagal (exception) → fallback ke model cadangan yang lebih murah/stabil
        4. hasilnya di-track_usage() (Phase 5 topik 22) buat cost tracking
        5. di-log
    → return teks jawaban (str)
```
Perhatikan urutan: rate limit dicek **sebelum** provider dipanggil (supaya gak buang request ke provider yang bakal ditolak/kena limit), sedangkan fallback baru jalan **setelah** panggilan pertama gagal.

### Contoh Kode — Python
`ModelGateway` menggantikan `LLMGateway` dari Phase 5 — struktur dasarnya (dispatch ke OpenAI/Anthropic) tetap sama, tapi `generate()` sekarang dibungkus rate limiting & fallback, dan di-compose dengan `track_usage` (Phase 5 topik 22) buat cost tracking otomatis:

```python
import time
from openai import OpenAI
from anthropic import Anthropic

MODEL_PRICE_PER_1K = {
    # Sama seperti PRICE_PER_1K_TOKENS di Phase 5 topik 22 — harga ilustratif
    "gpt-4o-mini": {"input": 0.00015, "output": 0.0006},
    "gpt-4o": {"input": 0.0025, "output": 0.01},
    "claude-3-5-sonnet-20241022": {"input": 0.003, "output": 0.015},
}

RATE_LIMIT_MAX_CALLS_PER_MINUTE = 60


def track_usage(response, model: str) -> dict:
    """
    Sama seperti Phase 5 topik 22: ekstrak token count dari response,
    hitung estimasi biaya berdasarkan price table.
    """
    input_tokens = response.usage.prompt_tokens
    output_tokens = response.usage.completion_tokens
    harga = MODEL_PRICE_PER_1K.get(model, {"input": 0.0, "output": 0.0})
    estimated_cost = (input_tokens / 1000) * harga["input"] + (output_tokens / 1000) * harga["output"]
    return {
        "model": model,
        "input_tokens": input_tokens,
        "output_tokens": output_tokens,
        "estimated_cost_usd": round(estimated_cost, 6),
    }


class ModelGateway:
    """
    Evolusi dari LLMGateway (Phase 5 topik 20). Selain dispatch ke provider
    yang benar, ModelGateway juga ngurus rate limiting, fallback antar model,
    dan cost tracking (lewat track_usage) — semuanya di satu tempat terpusat.
    """

    def __init__(self, provider: str = "openai"):
        self.provider = provider
        self.openai_client = OpenAI()
        self.anthropic_client = Anthropic()
        self._call_timestamps: list[float] = []  # buat rate limiting sederhana, in-memory

    def generate(self, messages: list[dict], model: str | None = None) -> str:
        """
        Cek rate limit → coba model utama → kalau gagal, fallback ke model
        cadangan → track_usage buat cost tracking → return teks jawaban.
        """
        self._enforce_rate_limit()

        chosen_model = model or "gpt-4o-mini"
        fallback_model = "gpt-4o-mini"

        try:
            reply, response = self._call_provider(messages, chosen_model)
        except Exception as e:
            print(f"[gateway] model {chosen_model} gagal ({e}), fallback ke {fallback_model}")
            reply, response = self._call_provider(messages, fallback_model)
            chosen_model = fallback_model

        usage_info = track_usage(response, chosen_model)
        print(f"[cost-tracking] {usage_info}")  # produksi: kirim ke logging/monitoring system

        return reply

    def _call_provider(self, messages: list[dict], model: str):
        """Dispatch mentah ke provider yang aktif — persis seperti LLMGateway Phase 5."""
        if self.provider == "openai":
            response = self.openai_client.chat.completions.create(model=model, messages=messages)
            reply = response.choices[0].message.content
        elif self.provider == "anthropic":
            response = self.anthropic_client.messages.create(model=model, max_tokens=1024, messages=messages)
            reply = response.content[0].text
        else:
            raise ValueError(f"Provider tidak dikenal: {self.provider}")
        return reply, response

    def _enforce_rate_limit(self) -> None:
        """
        Rate limiting sederhana: simpan timestamp tiap call, buang yang lebih
        tua dari 60 detik, tolak request baru kalau sisa timestamp dalam
        1 menit terakhir sudah mencapai batas.
        """
        now = time.time()
        one_minute_ago = now - 60
        self._call_timestamps = [t for t in self._call_timestamps if t > one_minute_ago]
        if len(self._call_timestamps) >= RATE_LIMIT_MAX_CALLS_PER_MINUTE:
            raise RuntimeError("Rate limit tercapai, coba lagi sebentar")
        self._call_timestamps.append(now)


# Satu instance ModelGateway dipakai di seluruh backend SupportPilot,
# menggantikan instance LLMGateway dari Phase 5.
gateway = ModelGateway(provider="openai")
```

### Trade-off & Pitfall
- **Rate limiting di sini disimpan in-memory (`self._call_timestamps`)** — kalau SupportPilot jalan di banyak proses/instance backend sekaligus (yang lazim di production), tiap instance punya hitungan sendiri-sendiri yang gak saling tahu, jadi batas efektifnya jadi `RATE_LIMIT_MAX_CALLS_PER_MINUTE × jumlah instance`. Buat rate limiting yang benar-benar global, hitungan ini perlu dipindah ke store terpusat (misal Redis, dibahas cara pakainya di topik 70) yang diakses semua instance.
- **Fallback ke model yang lebih murah bisa menurunkan kualitas jawaban** — kalau model utama gagal terus-menerus (bukan cuma sesekali), SupportPilot diam-diam kasih jawaban dari model cadangan yang mungkin gak sekuat model utama, tanpa customer tau ada apa-apa. Perlu ada alerting terpisah kalau fallback kepakai terlalu sering.
- **`ModelGateway` sekarang jauh lebih rumit dibanding `LLMGateway`** — makin banyak concern (rate limit, fallback, cost tracking, nanti caching & routing) digabung di satu class, makin besar juga risiko satu bug di sini berdampak ke SEMUA request LLM SupportPilot, bukan cuma satu fitur.
- **`except Exception` di `generate()` cukup lebar** — dia nangkep SEMUA jenis error (timeout, invalid API key, model gak ada, dst) dan langsung fallback tanpa membedakan mana error yang benar-benar transient (layak di-retry/fallback) dan mana yang bakal gagal lagi walau di-fallback (misal format `messages` yang salah).

### Kapan Dipakai
- Upgrade dari `LLMGateway` (Phase 5) ke `ModelGateway` begitu SupportPilot mulai punya traffic production nyata dan concern seperti "provider kadang down" atau "jangan sampai kena rate limit provider" jadi masalah sungguhan, bukan cuma teori.
- Kalau SupportPilot masih di tahap awal/traffic kecil, `LLMGateway` sederhana dari Phase 5 masih cukup — nambah rate limiting & fallback di titik ini bisa jadi over-engineering buat kebutuhan yang belum ada.
- Rate limiting in-memory di atas cukup buat single-instance deployment; begitu SupportPilot scale ke banyak instance, wajib pindah ke store terpusat (Redis) sebelum rate limiting-nya beneran efektif.

### Sering Ditanya Saat Interview
- **Apa perbedaan `ModelGateway` dengan `LLMGateway` di Phase 5?** — `LLMGateway` cuma ngurus dispatch ke provider yang benar; `ModelGateway` nambah rate limiting, fallback antar model, dan cost tracking terpusat di atas fondasi yang sama.
- **Kenapa rate limit dicek SEBELUM provider dipanggil, bukan sesudahnya?** — supaya request yang bakal ditolak gak perlu buang waktu/biaya manggil provider dulu; menolak di sisi kita sendiri jauh lebih murah dan cepat.
- **Apa risiko dari rate limiting yang disimpan in-memory per proses?** — kalau ada banyak instance backend berjalan paralel, tiap instance punya hitungan sendiri yang gak sinkron, jadi batas efektif totalnya jadi jauh lebih besar dari yang diniatkan; perlu store terpusat (Redis) buat rate limiting yang benar-benar akurat lintas instance.
- **Kenapa fallback ke model cadangan bisa jadi masalah tersembunyi?** — kalau model utama gagal terus-menerus, sistem diam-diam terus kasih jawaban dari model cadangan (mungkin kualitasnya lebih rendah) tanpa ada yang notice, kecuali ada alerting terpisah buat memantau seberapa sering fallback kepakai.

---

## 69. Model Routing

### Apa itu?
Model Routing adalah logic buat milih model yang paling cocok berdasarkan **kompleksitas task**, bukan selalu pakai model yang sama buat semua request:
```
Task simpel               → model murah & cepat   (misal "gpt-4o-mini")
Reasoning kompleks         → model kuat            (misal "claude-3-5-sonnet-20241022")
Klasifikasi volume tinggi  → model kecil           (misal "gpt-4o-mini")
```
Di `ModelGateway` (topik 68), routing ini diwujudkan lewat method `route(self, task_complexity: str) -> str`, yang dipanggil `generate()` di awal buat nentuin model mana yang benar-benar dipakai.

### Kenapa dibutuhkan?
Model yang paling pintar (dan paling mahal) gak selalu perlu buat semua task. Kalau SupportPilot pakai model paling mahal buat SEMUA request — termasuk task simpel seperti "klasifikasikan tiket ini urgent atau gak" yang sebenarnya bisa dijawab model kecil dengan akurasi yang gak jauh beda — biaya operasional jadi boros tanpa manfaat kualitas yang signifikan. Sebaliknya, kalau selalu pakai model murah buat SEMUA request termasuk yang butuh reasoning kompleks (misal "analisis akar masalah dari 5 tiket yang saling berkaitan"), kualitas jawabannya bisa jatuh di task yang justru butuh model kuat.

Model Routing menyelesaikan ini dengan mencocokkan **kekuatan model** dengan **kebutuhan task**: task simpel dilempar ke model murah/cepat, task yang butuh reasoning kompleks dilempar ke model yang lebih kuat. Hasilnya: biaya rata-rata per request turun signifikan (karena mayoritas task di aplikasi biasanya simpel) tanpa mengorbankan kualitas di task yang benar-benar butuh model kuat.

### Cara Kerja
```
generate(messages, task_complexity="simple") dipanggil
    → route(task_complexity) dicek dulu:
        "simple" / "classification"  → "gpt-4o-mini"
        "complex" / "reasoning"      → model yang lebih kuat
    → model hasil route() itu yang dipakai buat manggil provider
    → (fallback & cost tracking topik 68 tetap berlaku seperti biasa)
```
Kode pemanggil cukup bilang "ini task simpel" atau "ini task kompleks" lewat parameter `task_complexity` — dia gak perlu tau nama model spesifik apa yang bakal dipakai di baliknya.

### Contoh Kode — Python
`route()` ditambahkan ke `ModelGateway` dari topik 68, dan `generate()` diubah supaya manggil `route()` dulu sebelum menentukan model:

```python
MODEL_BY_COMPLEXITY = {
    "simple": "gpt-4o-mini",
    "classification": "gpt-4o-mini",
    "complex": "gpt-4o",
    "reasoning": "claude-3-5-sonnet-20241022",
}


class ModelGateway:
    # ... __init__, _call_provider, _enforce_rate_limit sama seperti topik 68 ...

    def route(self, task_complexity: str) -> str:
        """
        Pilih nama model berdasarkan task_complexity. Task simpel (klasifikasi,
        ekstraksi field pendek, dst) dilempar ke model kecil/murah; task yang
        butuh reasoning kompleks (analisis multi-langkah, ringkasan panjang
        yang perlu nalar) dilempar ke model yang lebih kuat.
        """
        model = MODEL_BY_COMPLEXITY.get(task_complexity)
        if model is None:
            raise ValueError(
                f"task_complexity tidak dikenal: {task_complexity!r}. "
                f"Pilihan valid: {list(MODEL_BY_COMPLEXITY)}"
            )
        return model

    def generate(self, messages: list[dict], task_complexity: str = "simple", model: str | None = None) -> str:
        """
        Kalau `model` gak dikasih eksplisit, generate() manggil route() dulu
        buat nentuin model berdasarkan task_complexity, SEBELUM manggil provider.
        """
        self._enforce_rate_limit()

        chosen_model = model or self.route(task_complexity)
        fallback_model = "gpt-4o-mini"

        try:
            reply, response = self._call_provider(messages, chosen_model)
        except Exception as e:
            print(f"[gateway] model {chosen_model} gagal ({e}), fallback ke {fallback_model}")
            reply, response = self._call_provider(messages, fallback_model)
            chosen_model = fallback_model

        usage_info = track_usage(response, chosen_model)
        print(f"[cost-tracking] {usage_info}")

        return reply


# Contoh pemakaian: task simpel (klasifikasi urgency tiket) vs task kompleks
gateway = ModelGateway(provider="openai")

hasil_klasifikasi = gateway.generate(
    messages=[{"role": "user", "content": "Klasifikasikan urgency tiket ini: 'Lupa password'"}],
    task_complexity="simple",  # → routed ke "gpt-4o-mini"
)

hasil_analisis = gateway.generate(
    messages=[{"role": "user", "content": "Analisis akar masalah dari 5 tiket refund yang saling berkaitan berikut..."}],
    task_complexity="reasoning",  # → routed ke "claude-3-5-sonnet-20241022"
)
```

### Trade-off & Pitfall
- **`task_complexity` ditentukan manual oleh kode pemanggil** — routing di sini gak otomatis "mendeteksi" seberapa kompleks sebuah task, dia cuma percaya label yang dikasih pemanggil. Kalau developer salah label (misal task yang sebenarnya butuh reasoning kompleks dikasih label `"simple"`), routing-nya jadi salah arah dan kualitas jawaban ikut turun tanpa error yang jelas.
- **Fallback (topik 68) dan routing bisa saling tabrakan** — kalau task berlabel `"reasoning"` di-route ke model kuat, tapi model itu gagal dan fallback jatuh ke `"gpt-4o-mini"` (model murah), request yang tadinya butuh reasoning kompleks jadi dijawab model yang justru gak cocok buat itu. Versi production biasanya butuh fallback chain yang mempertimbangkan kompleksitas task juga, bukan satu fallback model yang sama buat semua kasus.
- **Menambah kategori `task_complexity` baru berarti nge-update `MODEL_BY_COMPLEXITY` secara manual** — mapping ini gampang jadi basi kalau ada model baru yang lebih baik/murah rilis dan gak keburu di-update.
- **Routing berbasis rule sederhana seperti ini gak mempertimbangkan panjang/isi konten pesan** — task yang dilabel `"simple"` tapi ternyata isinya kompleks (misal customer nanya hal simpel tapi dengan konteks yang panjang dan berlapis) tetap bakal di-route ke model kecil, walau sebenarnya butuh model yang lebih kuat.

### Kapan Dipakai
- Pakai Model Routing begitu SupportPilot punya BEBERAPA jenis task dengan kompleksitas yang jelas beda-beda (misal klasifikasi tiket vs analisis akar masalah multi-tiket) dan volume request-nya cukup besar sehingga selisih biaya per model jadi signifikan secara total.
- Kalau SupportPilot cuma punya satu jenis task yang konsisten kompleksitasnya, routing gak banyak berguna — cukup pakai satu model tetap seperti di Phase 5.
- Evaluasi (Phase 4 topik 18) tetap perlu dijalankan per kategori `task_complexity` buat mastiin model murah yang dipilih buat task "simple" beneran cukup akurat, bukan cuma diasumsikan cukup.

### Sering Ditanya Saat Interview
- **Apa tujuan utama Model Routing?** — mencocokkan kekuatan (dan biaya) model dengan kebutuhan task: task simpel ke model murah/cepat, task reasoning kompleks ke model yang lebih kuat, supaya biaya rata-rata turun tanpa mengorbankan kualitas di task yang benar-benar butuh model kuat.
- **Di titik mana route() dipanggil dalam alur generate()?** — di awal, sebelum provider dipanggil — hasil route() (nama model) itu yang dipakai buat menentukan model final, kecuali pemanggil sudah kasih `model` secara eksplisit.
- **Apa risiko dari routing berbasis label manual (`task_complexity`) seperti ini?** — kalau developer salah label kompleksitas sebuah task, routing-nya jadi salah arah tanpa error yang jelas — task kompleks bisa kepilih model yang terlalu lemah, atau sebaliknya.
- **Kenapa fallback dan routing bisa saling tabrakan?** — kalau model hasil routing gagal dan fallback jatuh ke model murah yang sama buat semua kasus, task yang tadinya butuh reasoning kompleks bisa jadi dijawab model yang gak cocok buat itu — fallback chain production yang baik perlu mempertimbangkan kompleksitas task juga.

---

## 70. AI Caching

### Apa itu?
AI Caching adalah teknik menyimpan hasil jawaban LLM buat kombinasi prompt+context yang identik (atau serupa), supaya pertanyaan yang sama gak perlu manggil LLM lagi — cukup ambil dari cache. Di SupportPilot, ini diwujudkan lewat `AICache`, yang membungkus `ModelGateway.generate()` (topik 68-69) dengan cache **backed by Redis** (Redis dipilih karena cepat buat operasi baca/tulis key-value sederhana, dan mendukung TTL/expiry bawaan buat data yang gak boleh disimpan selamanya).

### Kenapa dibutuhkan?
Banyak pertanyaan customer ke SupportPilot itu **berulang** — banyak orang nanya "gimana cara reset password?" dengan kata-kata yang sama persis, atau sistem internal SupportPilot manggil prompt yang identik berkali-kali (misal template klasifikasi tiket yang sama, dijalankan ke banyak tiket dengan isi mirip). Tanpa caching, tiap pertanyaan identik itu manggil LLM provider dari nol — buang biaya (bayar token yang sama berkali-kali) dan waktu (latency network + generation), padahal jawabannya bisa dipastikan sama.

AI Caching menyelesaikan ini dengan cara paling sederhana: simpan jawaban LLM di Redis, di-key berdasarkan **hash dari prompt + context relevan**. Begitu ada request dengan prompt+context yang identik, `AICache` langsung ambil jawaban dari Redis (**cache hit**) tanpa manggil LLM sama sekali. Tapi caching LLM punya risiko khusus yang gak ada di caching data biasa: **context yang user-specific atau sensitif** (kalau di-cache sembarangan, jawaban buat satu customer bisa "nyasar" ke customer lain kalau key cache-nya gak membedakan mereka), dan **response yang jadi basi (stale)** — kalau informasi yang mendasari jawaban berubah (misal kebijakan refund SupportPilot berubah), jawaban lama yang ke-cache tetap dikembalikan sampai TTL-nya habis, walau udah gak akurat lagi.

### Cara Kerja
```
AICache.generate(messages, task_complexity) dipanggil
    → build cache_key = hash(messages + task_complexity)   # prompt + context relevan
    → cek Redis: apakah cache_key ini sudah ada?
        ADA (cache HIT)   → langsung return jawaban dari Redis, TANPA manggil LLM
        GAK ADA (cache MISS) → panggil ModelGateway.generate() seperti biasa
                                → simpan hasilnya ke Redis dengan TTL tertentu
                                → return jawaban itu
```

### Contoh Kode — Python

> **Catatan Python:** `hashlib.sha256(...)` menghasilkan **hash** — representasi teks panjang-tetap (64 karakter hex) dari input apapun, di mana input yang identik SELALU menghasilkan hash yang sama, dan input yang beda (walau cuma beda satu karakter) hampir selalu menghasilkan hash yang beda total. Ini pas buat jadi cache key: daripada nyimpen seluruh isi `messages` (yang bisa panjang) sebagai key Redis, kita simpan hash-nya saja — pendek, konsisten, dan aman buat dipakai sebagai key. `json.dumps(..., sort_keys=True)` dipakai supaya urutan key dalam dict gak mempengaruhi hasil hash — tanpa `sort_keys=True`, dict yang isinya sama tapi urutan key-nya beda bisa menghasilkan string JSON (dan hash) yang berbeda, padahal secara makna keduanya identik.

```python
import hashlib
import json
import redis


class AICache:
    """
    Cache di depan ModelGateway.generate(), backed by Redis. Key cache dibuat
    dari hash (prompt + context relevan), supaya request yang identik gak
    perlu manggil LLM lagi.
    """

    def __init__(self, gateway: "ModelGateway", redis_client: "redis.Redis | None" = None, ttl_seconds: int = 3600):
        self.gateway = gateway
        # decode_responses=True supaya Redis balikin str, bukan bytes
        self.redis_client = redis_client or redis.Redis(host="localhost", port=6379, decode_responses=True)
        self.ttl_seconds = ttl_seconds  # jawaban di-cache selama 1 jam secara default

    def _build_cache_key(self, messages: list[dict], task_complexity: str) -> str:
        """
        Hash dari prompt + context relevan. Dua request dengan messages dan
        task_complexity yang identik bakal hasilin cache_key yang SAMA persis.
        """
        raw = json.dumps({"messages": messages, "task_complexity": task_complexity}, sort_keys=True)
        digest = hashlib.sha256(raw.encode("utf-8")).hexdigest()
        return f"ai_cache:{digest}"

    def generate(self, messages: list[dict], task_complexity: str = "simple", model: str | None = None) -> str:
        """
        Cek Redis dulu berdasarkan cache_key. Kalau ada (HIT), langsung
        balikin tanpa manggil LLM. Kalau gak ada (MISS), panggil
        ModelGateway.generate() seperti biasa, lalu simpan hasilnya ke Redis.
        """
        cache_key = self._build_cache_key(messages, task_complexity)

        cached_reply = self.redis_client.get(cache_key)
        if cached_reply is not None:
            print(f"[ai-cache] HIT untuk key {cache_key[:24]}...")
            return cached_reply

        print(f"[ai-cache] MISS untuk key {cache_key[:24]}..., manggil LLM")
        reply = self.gateway.generate(messages, task_complexity=task_complexity, model=model)

        self.redis_client.set(cache_key, reply, ex=self.ttl_seconds)
        return reply


# Dua pertanyaan customer yang identik persis — request kedua bakal HIT,
# gak manggil LLM lagi
cache = AICache(gateway=ModelGateway(provider="openai"), ttl_seconds=3600)

jawaban_1 = cache.generate(
    messages=[{"role": "user", "content": "Gimana cara reset password?"}],
    task_complexity="simple",
)  # cache MISS pertama kali — manggil LLM

jawaban_2 = cache.generate(
    messages=[{"role": "user", "content": "Gimana cara reset password?"}],
    task_complexity="simple",
)  # cache HIT — messages & task_complexity identik dengan request pertama
```

### Trade-off & Pitfall
- **Context user-specific bikin caching berbahaya kalau gak dipikirkan matang** — kalau `messages` yang di-hash kebetulan gak menyertakan identitas customer (misal cuma isi pertanyaannya tanpa nomor akun), dua customer berbeda yang nanya hal yang mirip-mirip bisa dapat cache_key yang sama dan saling "menerima" jawaban satu sama lain. Sebaliknya, kalau context yang disertakan justru terlalu spesifik per-customer (misal termasuk nomor tiket unik), cache hit rate-nya jadi rendah karena hampir gak ada dua request yang benar-benar identik. Perlu dipikirkan dengan sengaja: bagian mana dari context yang boleh ikut di-hash (general/shared) dan mana yang harus dikeluarkan (sensitif/user-specific).
- **Response bisa jadi stale (basi)** — kalau kebijakan/informasi yang mendasari jawaban LLM berubah (misal SupportPilot ubah kebijakan refund dari 7 hari jadi 14 hari), jawaban lama yang sudah ke-cache tetap dikembalikan sampai `ttl_seconds` habis, walau udah gak akurat. TTL yang terlalu panjang berarti hemat biaya tapi risiko stale lebih besar; TTL yang terlalu pendek berarti cache hit rate lebih rendah (lebih sering MISS, lebih mirip gak pakai cache sama sekali).
- **Ini exact-match caching** — hash dibuat dari isi `messages` secara harfiah, jadi dua pertanyaan yang MAKSUDNYA sama tapi ditulis beda kata (misal "gimana cara reset password?" vs "cara reset password gimana ya?") bakal dianggap beda total dan tetap MISS. Caching yang lebih canggih (semantic caching, berbasis kesamaan embedding — Phase 3) bisa nangkep kasus ini, tapi itu di luar scope topik ini dan datang dengan kompleksitas serta biaya tambahannya sendiri (perlu hitung embedding dulu buat tiap request, plus vector similarity search).
- **Kalau Redis-nya down, `AICache.generate()` di atas ikut gagal total** — versi production sebaiknya membungkus panggilan ke Redis dengan try/except supaya, kalau cache-nya bermasalah, SupportPilot tetap bisa jalan (langsung ke LLM tanpa cache) daripada seluruh fitur ikut down gara-gara cache layer gagal.

### Kapan Dipakai
- Pakai AI Caching buat pertanyaan yang **sering berulang secara identik atau nyaris identik** dan jawabannya relatif stabil dari waktu ke waktu — contoh klasik: FAQ umum ("gimana cara reset password?", "jam operasional customer service?").
- **Hindari** caching buat context yang user-specific atau sensitif (data akun pribadi, riwayat transaksi customer tertentu) kecuali cache_key-nya benar-benar menyertakan identitas customer secara eksplisit dan sengaja.
- **Hindari** caching buat jawaban yang bergantung pada informasi yang sering berubah (harga terkini, status tiket yang sedang berjalan) kecuali TTL-nya diset cukup pendek buat menjaga kesegaran, atau ada mekanisme invalidasi cache manual begitu informasi yang mendasarinya berubah.

### Sering Ditanya Saat Interview
- **Bagaimana cache_key dibuat di `AICache`, dan kenapa dibuat begitu?** — dari hash (SHA-256) atas gabungan `messages` + `task_complexity` yang di-serialize secara deterministik (`sort_keys=True`); dua request dengan isi identik menghasilkan cache_key yang sama, sehingga request kedua bisa langsung ambil dari cache tanpa memanggil LLM.
- **Apa risiko utama caching LLM response yang gak ada di caching data biasa?** — context user-specific/sensitif bisa "nyasar" ke customer lain kalau cache_key gak membedakan mereka dengan tepat, dan response bisa jadi stale (basi) kalau informasi yang mendasarinya berubah sebelum TTL cache habis.
- **Kenapa AI Caching di sini disebut "exact-match", dan apa keterbatasannya?** — karena cache_key dibuat dari hash isi prompt secara harfiah, dua pertanyaan yang maknanya sama tapi kata-katanya beda tetap dianggap request yang berbeda (MISS); menangkap kesamaan makna butuh semantic caching berbasis embedding, yang lebih kompleks.
- **Apa trade-off dari TTL yang panjang vs pendek pada AI Caching?** — TTL panjang berarti cache hit rate lebih tinggi (lebih hemat biaya & latency) tapi risiko jawaban jadi stale lebih besar; TTL pendek lebih segar tapi cache-nya lebih sering MISS, mendekati kondisi tanpa cache.

---

## 71. Batch Processing

### Apa itu?
Batch Processing adalah pola buat memproses volume data yang besar lewat LLM secara efisien, dengan membagi data itu jadi kelompok-kelompok kecil (batch) yang diproses berurutan, sementara di dalam tiap batch, item-itemnya diproses **secara konkuren**:
```
1.000.000 records (misal tiket support) → dibagi jadi batch-batch kecil → tiap batch diproses ke LLM → hasilnya disimpan
```
Di SupportPilot, ini diwujudkan lewat `batch_process(records: list[dict], batch_size: int = 10) -> list[dict]`, yang memproses daftar tiket support lewat LLM dalam batch berukuran `batch_size`, dengan retry otomatis kalau ada item yang gagal.

### Kenapa dibutuhkan?
Bayangin SupportPilot mau meringkas 1 juta tiket support lama sekali jalan (misal buat migrasi data atau analisis tren tahunan). Kalau diproses **satu-satu secara berurutan** (loop biasa, tunggu tiket 1 selesai baru mulai tiket 2, dst), dan tiap panggilan LLM butuh ~2 detik, total waktunya jadi sekitar 2.000.000 detik (~23 hari) — jelas gak praktis. Sebaliknya, kalau SEMUA 1 juta tiket ditembak ke LLM provider SECARA BERSAMAAN tanpa kontrol sama sekali, provider bakal langsung nolak lewat rate limiting (dan biaya melonjak gak terkontrol dalam waktu singkat).

Batch Processing menyelesaikan ini dengan jalan tengah: bagi data jadi batch berukuran wajar (`batch_size`, misal 10), lalu di dalam SATU batch, proses item-itemnya **secara konkuren** (bukan satu-satu berurutan) supaya total waktu tunggu network buat 10 item itu numpuk jadi kira-kira SATU kali waktu tunggu, bukan 10 kali. Setelah satu batch selesai, lanjut ke batch berikutnya. Ini butuh mempertimbangkan beberapa hal sekaligus: **concurrency** (berapa banyak yang diproses bersamaan), **batch size** (berapa item per batch), **rate limit** (jangan sampai concurrency-nya melebihi kapasitas provider), **retry** (item yang gagal — misal karena error transient — dicoba ulang, bukan langsung dianggap gagal permanen), dan **cost** (total biaya buat memproses semua record).

### Cara Kerja
```
batch_process(records, batch_size=10) dipanggil
    → bagi records jadi kelompok-kelompok berukuran batch_size
    → untuk tiap kelompok (batch):
        → proses SEMUA item dalam batch itu SECARA KONKUREN
          (pakai ThreadPoolExecutor — lihat Catatan Python di bawah)
        → tiap item yang gagal di-retry sampai MAX_RETRIES kali,
          dengan sedikit delay (backoff) di antara percobaan
        → kumpulkan hasil (sukses ATAU gagal permanen) dari batch ini
    → lanjut ke batch berikutnya
    → return semua hasil setelah SEMUA batch selesai
```

### Contoh Kode — Python

> **Catatan Python:** `concurrent.futures.ThreadPoolExecutor` adalah cara buat menjalankan beberapa fungsi **secara konkuren** pakai kumpulan (pool) thread, tanpa harus nulis manajemen thread manual. Ini pas buat kasus kita karena tiap panggilan LLM itu **I/O-bound** — sebagian besar waktunya dihabiskan NUNGGU response dari network (provider LLM), bukan komputasi berat di CPU. Selama nunggu itu, Python bisa "mengalihkan" CPU ke thread lain yang juga sedang nunggu response-nya masing-masing, jadi beberapa panggilan LLM efektif berjalan bersamaan meski cuma pakai satu proses Python. (Kalau task-nya CPU-bound — misal komputasi berat, bukan nunggu network — `ThreadPoolExecutor` gak akan membantu karena Python punya GIL yang membatasi eksekusi bytecode paralel di banyak thread; buat kasus itu baru `multiprocessing` yang relevan, di luar scope topik ini.) `executor.submit(fungsi, *args)` menjadwalkan satu pemanggilan fungsi dan langsung balikin objek `Future` (representasi "hasil yang akan ada nanti"), sementara `concurrent.futures.as_completed(...)` mengembalikan tiap `Future` begitu SELESAI — bukan berdasarkan urutan submit, tapi berdasarkan urutan siapa yang kelar duluan.

```python
import concurrent.futures
import time

MAX_RETRIES = 3
RETRY_BACKOFF_SECONDS = 2


def process_one_record(gateway: "ModelGateway", record: dict) -> dict:
    """
    Proses SATU tiket support lewat LLM (ringkas isinya), dengan retry
    sederhana kalau gagal (misal error transient dari provider).
    """
    ticket_id = record["ticket_id"]
    messages = [{"role": "user", "content": f"Ringkas tiket support ini dalam satu kalimat: {record['body']}"}]

    last_error: Exception | None = None
    for attempt in range(1, MAX_RETRIES + 1):
        try:
            summary = gateway.generate(messages, task_complexity="simple")
            return {"ticket_id": ticket_id, "summary": summary, "status": "success"}
        except Exception as e:
            last_error = e
            print(f"[batch] tiket {ticket_id} gagal (attempt {attempt}/{MAX_RETRIES}): {e}")
            if attempt < MAX_RETRIES:
                time.sleep(RETRY_BACKOFF_SECONDS * attempt)  # backoff makin lama tiap percobaan

    # Sudah dicoba MAX_RETRIES kali dan tetap gagal — dianggap gagal permanen
    return {"ticket_id": ticket_id, "summary": None, "status": "failed", "error": str(last_error)}


def batch_process(records: list[dict], batch_size: int = 10) -> list[dict]:
    """
    Proses SEMUA records lewat LLM, dibagi jadi batch berukuran batch_size.
    Di dalam satu batch, tiap record diproses SECARA KONKUREN pakai thread
    pool — I/O-bound (nunggu network response LLM), jadi thread sudah cukup,
    gak perlu multiprocessing.
    """
    gateway = ModelGateway(provider="openai")
    results: list[dict] = []

    for start in range(0, len(records), batch_size):
        batch = records[start : start + batch_size]

        with concurrent.futures.ThreadPoolExecutor(max_workers=batch_size) as executor:
            future_to_ticket = {
                executor.submit(process_one_record, gateway, record): record["ticket_id"]
                for record in batch
            }
            for future in concurrent.futures.as_completed(future_to_ticket):
                results.append(future.result())

    return results


# Contoh pemakaian: 25 tiket, diproses dalam batch berukuran 10
# (batch 1: 10 tiket, batch 2: 10 tiket, batch 3: 5 tiket — masing-masing
# batch diproses konkuren di dalamnya, batch-nya sendiri berurutan)
tiket_support = [
    {"ticket_id": i, "body": f"Customer #{i} nanya soal refund yang belum cair"}
    for i in range(1, 26)
]
hasil = batch_process(tiket_support, batch_size=10)

berhasil = [r for r in hasil if r["status"] == "success"]
gagal = [r for r in hasil if r["status"] == "failed"]
print(f"Sukses: {len(berhasil)}, Gagal permanen: {len(gagal)}")
```

### Trade-off & Pitfall
- **Urutan hasil di `results` GAK sama dengan urutan `records` masuk** — karena `as_completed()` mengembalikan `Future` berdasarkan siapa yang kelar duluan, bukan urutan submit, item yang lebih cepat diproses (misal karena responsnya lebih pendek) bisa nongol lebih dulu di `results` walau di-submit belakangan. Kalau urutan penting (misal hasil harus disimpan sejajar dengan urutan input asli), perlu disertakan `ticket_id`/index di tiap hasil (sudah dilakukan di atas) dan di-sort ulang belakangan, atau pakai `executor.map()` yang mempertahankan urutan (dengan trade-off sedikit lebih kaku).
- **`batch_size` yang menentukan `max_workers` bisa tetap kena rate limit provider kalau kelewat besar** — batching per se gak otomatis aman dari rate limit; `batch_size` harus disesuaikan dengan rate limit provider yang sebenarnya (dan idealnya dikombinasikan dengan `ModelGateway._enforce_rate_limit` dari topik 68, bukan berdiri sendiri).
- **Retry di sini nge-retry SEMUA jenis exception dengan cara yang sama** — error yang benar-benar transient (timeout, rate limit sesaat) memang layak di-retry, tapi error permanen (misal `record["body"]` isinya format yang gak valid buat prompt) bakal tetap gagal walau di-retry 3 kali, cuma buang waktu (dan `RETRY_BACKOFF_SECONDS` yang bertambah tiap attempt). Versi production biasanya membedakan jenis error yang retryable vs yang gak, supaya gak retry error yang jelas-jelas gak akan pernah berhasil.
- **Antar-batch tetap berjalan berurutan (bukan semua batch sekaligus)** — ini SENGAJA (buat mengontrol beban ke provider secara keseluruhan), tapi berarti total waktu proses tetap `jumlah_batch × waktu_per_batch`, bukan bisa dipercepat lagi cuma dengan menaikkan `batch_size` tanpa batas (karena `max_workers` yang terlalu besar dalam satu batch balik lagi ke risiko rate limit di atas).

### Kapan Dipakai
- Pakai Batch Processing buat memproses volume data besar lewat LLM di luar jalur customer-facing real-time — migrasi data, analisis tren historis, backfill ringkasan buat tiket-tiket lama, dan job-job serupa yang dijalankan sebagai background job/scheduled task.
- **Jangan** dipakai buat jalur interaktif customer (misal endpoint `/chat`) — batching menambahkan latency (nunggu batch penuh atau proses batch selesai) yang gak cocok buat interaksi real-time; buat itu, request/response biasa (Phase 5 topik 19) atau streaming (topik 21) lebih tepat.
- `batch_size` dan jumlah `max_workers` perlu disetel berdasarkan rate limit provider yang sebenarnya (bisa dicek di dashboard/dokumentasi provider) — nilai `10` di contoh ini cuma titik awal yang wajar, bukan angka baku.

### Sering Ditanya Saat Interview
- **Kenapa Batch Processing membagi data jadi batch, bukan proses semuanya sekaligus atau satu-satu berurutan?** — satu-satu berurutan terlalu lambat buat volume besar (total waktu numpuk linear); semuanya sekaligus tanpa kontrol bisa langsung kena rate limit provider dan bikin biaya melonjak; batch adalah jalan tengah yang mengontrol berapa banyak yang diproses konkuren dalam satu waktu.
- **Kenapa `ThreadPoolExecutor` (bukan `multiprocessing`) yang dipakai buat konkurensi di dalam batch di sini?** — karena panggilan LLM itu I/O-bound (dominan nunggu network), thread sudah cukup buat mendapat manfaat konkurensi; `multiprocessing` lebih relevan buat task CPU-bound yang dibatasi GIL Python.
- **Apa yang menyebabkan urutan hasil `batch_process` bisa berbeda dari urutan input records?** — `as_completed()` mengembalikan `Future` berdasarkan urutan siapa yang selesai duluan, bukan urutan submit; kalau urutan penting, hasil harus disertakan identifier (misal `ticket_id`) dan diurutkan ulang setelah semua selesai.
- **Apa risiko dari retry logic yang nge-retry semua jenis exception secara sama?** — error permanen (misal input yang formatnya invalid) tetap akan gagal walau di-retry berkali-kali, cuma buang waktu dan resource; retry logic yang lebih matang membedakan error transient (layak di-retry) dari error permanen (langsung gagal tanpa retry).

---

**Selanjutnya:** [Phase 19 — Practical Projects](./phase-19-practical-projects.md)
