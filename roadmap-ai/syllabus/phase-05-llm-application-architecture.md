# Phase 05 — LLM Application Architecture

> Bagian dari [AI Engineering / Agent Roadmap](../README.md)

---

## 19. Basic LLM Backend

### Apa itu?
Basic LLM Backend adalah pola arsitektur paling dasar buat aplikasi yang manggil LLM: ada **backend** (server yang kita kontrol sendiri) yang duduk di antara client (browser/app customer) dan LLM provider (OpenAI, Anthropic, dll). Alurnya:
```
Client → Backend API → LLM Service → LLM Provider
```
Client gak pernah manggil LLM provider secara langsung — dia cuma ngomong ke backend kita, dan backend kita yang urus komunikasi ke LLM provider.

### Kenapa dibutuhkan?
Kalau frontend (kode yang jalan di browser customer) manggil OpenAI/Anthropic API langsung, API key harus ditaruh di kode frontend itu — dan kode frontend itu **bisa dibaca siapa aja** yang buka DevTools browser. Begitu API key SupportPilot bocor kayak gitu, siapapun bisa pakai API key itu buat manggil LLM provider atas nama SupportPilot, dan tagihannya jadi tanggung jawab SupportPilot walau yang makai orang lain sama sekali.

Backend menyelesaikan ini dengan cara paling sederhana: **secret (API key) cuma ada di server, gak pernah dikirim ke client**. Client cuma ngomong ke backend SupportPilot sendiri (yang kita kontrol penuh), dan backend itu yang pegang API key buat manggil LLM provider. Selain soal keamanan, ini juga ngasih kita titik kontrol terpusat buat hal-hal lain — rate limiting per customer, logging tiap request, validasi input sebelum sampai ke LLM, dan (nanti di topik 20-22) cost tracking & provider abstraction.

### Cara Kerja
```
1. Customer ngetik pesan di UI chat SupportPilot
2. Frontend kirim pesan itu ke backend SupportPilot (POST /chat)
3. Backend SupportPilot manggil LLM provider pakai API key yang cuma ada di server
4. LLM provider balikin jawaban ke backend
5. Backend balikin jawaban itu ke frontend sebagai JSON
```
Frontend gak pernah "ngeliat" API key atau berkomunikasi langsung dengan LLM provider — semuanya lewat backend SupportPilot.

### Contoh Kode — Python
Backend minimal SupportPilot pakai FastAPI: satu endpoint `POST /chat` yang nerima pesan customer, teruskan ke LLM, balikin jawabannya sebagai JSON.

> **Catatan Python:** `@app.post("/chat")` di bawah ini adalah **decorator** — sintaks `@sesuatu` yang ditaruh tepat di atas definisi fungsi. Decorator "membungkus" fungsi di bawahnya buat nambahin perilaku tambahan tanpa mengubah isi fungsi itu sendiri. Di sini, `@app.post("/chat")` mendaftarkan fungsi `chat` sebagai handler yang dipanggil FastAPI setiap kali ada request `POST` masuk ke path `/chat` — tanpa decorator ini, `chat` cuma fungsi Python biasa yang gak "nyambung" ke request HTTP apapun.

```python
from fastapi import FastAPI
from pydantic import BaseModel
from openai import OpenAI

app = FastAPI()
client = OpenAI()


class ChatRequest(BaseModel):
    """Bentuk JSON yang diharapkan masuk ke POST /chat."""
    message: str


class ChatResponse(BaseModel):
    """Bentuk JSON yang dibalikin dari POST /chat."""
    reply: str


@app.post("/chat")
def chat(request: ChatRequest) -> ChatResponse:
    """
    Endpoint utama SupportPilot: terima pesan customer, teruskan ke LLM,
    balikin jawabannya. Ini satu-satunya "pintu masuk" yang boleh diakses
    frontend — API key OpenAI cuma ada di sini, di backend, gak pernah
    ikut dikirim ke browser customer.
    """
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": request.message}],
    )
    reply = response.choices[0].message.content
    return ChatResponse(reply=reply)


# Dijalankan pakai: uvicorn main:app --reload
# Contoh request dari frontend: POST /chat {"message": "gimana cara reset password?"}
# Contoh response:              {"reply": "Untuk reset password, klik ..."}
```

### Trade-off & Pitfall
- **Backend ini nambah satu "hop" (loncatan) network** dibanding manggil LLM langsung dari frontend — ada sedikit tambahan latency, tapi ini trade-off yang wajib diterima demi keamanan API key.
- **Kalau backend down, seluruh fitur chat ikut down** — beda dengan (secara teori) manggil LLM langsung dari frontend yang gak butuh backend sama sekali. Ini konsekuensi wajar dari punya server sendiri, dan biasanya jauh lebih ringan risikonya dibanding bocornya API key.
- **Endpoint `/chat` ini masih polos** — belum ada rate limiting, validasi input yang ketat, atau proteksi dari abuse (misal satu customer spam request berkali-kali). Ini biasanya ditambahkan sebagai middleware/dependency terpisah, di luar scope topik ini.
- **Model di-hardcode langsung di kode (`"gpt-4o-mini"`)** — begitu mau ganti model atau nambah provider lain, kode ini harus diubah manual di banyak tempat. Ini yang diselesaikan oleh LLM Gateway (topik 20).

### Kapan Dipakai
- Pola ini adalah **fondasi wajib** buat aplikasi apapun yang manggil LLM dan ada secret (API key) yang terlibat — hampir gak ada kasus di mana manggil LLM langsung dari frontend itu aman.
- Cukup dipakai apa adanya (tanpa gateway/streaming/cost tracking) buat prototipe awal atau fitur internal yang trafiknya kecil.
- Begitu SupportPilot butuh dukung banyak provider, tracking biaya, atau UX streaming — lanjut ke topik 20-22 yang membangun di atas fondasi ini.

### Sering Ditanya Saat Interview
- **Kenapa gak boleh manggil LLM provider langsung dari frontend?** — API key bakal ikut terkirim ke kode yang jalan di browser customer, yang bisa dibaca siapa aja lewat DevTools; begitu bocor, siapapun bisa pakai API key itu dan tagihannya jadi tanggung jawab pemilik key.
- **Apa alur dasar Basic LLM Backend?** — Client → Backend API → LLM Service → LLM Provider; client cuma ngomong ke backend sendiri, backend yang urus komunikasi ke LLM provider.
- **Selain keamanan, apa manfaat lain dari punya backend di antara client dan LLM provider?** — titik kontrol terpusat buat rate limiting, logging, validasi input, cost tracking, dan provider abstraction — semuanya jadi lebih gampang diatur karena cuma ada satu tempat yang perlu diubah.
- **Apa itu decorator di Python, dan buat apa `@app.post("/chat")` dipakai di sini?** — decorator adalah sintaks `@sesuatu` yang membungkus fungsi buat nambahin perilaku tanpa mengubah isi fungsinya; `@app.post("/chat")` mendaftarkan fungsi di bawahnya sebagai handler buat request `POST /chat`.

---

## 20. LLM Gateway / Provider Abstraction

### Apa itu?
LLM Gateway adalah layer abstraksi di antara backend SupportPilot dan berbagai LLM provider (OpenAI, Anthropic, dst):
```
Backend → LLM Gateway → Model A / Model B / Model C
```
Daripada kode backend manggil `OpenAI()` atau `Anthropic()` langsung di banyak tempat berbeda, semua panggilan lewat satu class/fungsi terpusat — `LLMGateway` — yang di baliknya baru menentukan provider mana yang benar-benar dipanggil.

### Kenapa dibutuhkan?
Di topik 19, endpoint `/chat` manggil `client.chat.completions.create(...)` langsung — artinya kode itu "terikat" (coupled) ke OpenAI SDK secara spesifik. Kalau SupportPilot suatu saat mau: pindah ke Anthropic buat model tertentu, fallback ke provider B kalau provider A lagi down, atau A/B testing dua model berbeda — kita harus ubah kode di **setiap tempat** yang manggil LLM secara langsung. Kalau ada 10 endpoint yang masing-masing manggil OpenAI SDK langsung, ganti provider berarti ubah 10 tempat.

LLM Gateway menyelesaikan ini dengan menyediakan **satu interface** yang sama buat semua provider. Manfaatnya: model switching & fallback (ganti provider tanpa ubah kode pemanggil), cost control & rate limiting terpusat (satu tempat buat nerapin logic ini ke SEMUA request, gak peduli provider mana), dan konfigurasi terpusat (endpoint URL, API key, default model — semuanya diatur di satu tempat).

### Cara Kerja
```
Kode pemanggil (misal endpoint /chat)
    → gateway.generate(messages, model=...)
        → kalau provider == "openai"    → panggil OpenAI SDK
        → kalau provider == "anthropic" → panggil Anthropic SDK
    → return teks jawaban (str), gak peduli provider mana yang dipanggil di baliknya
```
Kode pemanggil cuma kenal method `generate()` — dia gak perlu tau (dan gak perlu peduli) provider SDK spesifik apa yang lagi jalan di baliknya.

### Contoh Kode — Python
```python
from openai import OpenAI
from anthropic import Anthropic

PRICE_PER_1K_TOKENS = {
    "gpt-4o-mini": {"input": 0.00015, "output": 0.0006},
    "claude-3-5-sonnet-20241022": {"input": 0.003, "output": 0.015},
}


class LLMGateway:
    """
    Layer abstraksi di antara backend SupportPilot dan provider LLM.
    Kode yang manggil LLM cukup kenal LLMGateway.generate() — gak perlu tau
    provider mana (OpenAI/Anthropic) yang lagi dipakai di balik layar.
    """

    def __init__(self, provider: str = "openai"):
        self.provider = provider
        self.openai_client = OpenAI()
        self.anthropic_client = Anthropic()

    def generate(self, messages: list[dict], model: str | None = None) -> str:
        """
        Terima messages (format OpenAI-style: list of {"role", "content"}),
        dispatch ke provider yang aktif, balikin teks jawabannya sebagai str.
        """
        if self.provider == "openai":
            return self._generate_openai(messages, model or "gpt-4o-mini")
        elif self.provider == "anthropic":
            return self._generate_anthropic(messages, model or "claude-3-5-sonnet-20241022")
        else:
            raise ValueError(f"Provider tidak dikenal: {self.provider}")

    def _generate_openai(self, messages: list[dict], model: str) -> str:
        response = self.openai_client.chat.completions.create(model=model, messages=messages)
        return response.choices[0].message.content

    def _generate_anthropic(self, messages: list[dict], model: str) -> str:
        # Format Anthropic sedikit beda: max_tokens wajib diisi eksplisit
        response = self.anthropic_client.messages.create(
            model=model,
            max_tokens=1024,
            messages=messages,
        )
        return response.content[0].text


# Satu instance gateway dipakai di seluruh backend SupportPilot
gateway = LLMGateway(provider="openai")
```

Endpoint `/chat` dari topik 19 sekarang tinggal manggil gateway, bukan `client` (OpenAI SDK) langsung lagi:
```python
@app.post("/chat")
def chat(request: ChatRequest) -> ChatResponse:
    """
    Sama seperti topik 19, tapi sekarang manggil lewat LLMGateway.
    Kalau SupportPilot ganti provider default dari OpenAI ke Anthropic,
    baris ini SAMA SEKALI GAK PERLU DIUBAH — cukup ganti konfigurasi gateway.
    """
    reply = gateway.generate(messages=[{"role": "user", "content": request.message}])
    return ChatResponse(reply=reply)
```

### Trade-off & Pitfall
- **Gateway nambah satu layer abstraksi lagi** — buat aplikasi kecil dengan satu provider yang gak bakal berubah, ini bisa jadi over-engineering; manfaatnya baru kerasa begitu ada 2+ provider atau kebutuhan fallback/routing.
- **Format request/response tiap provider beda-beda** (OpenAI vs Anthropic punya struktur message dan response yang gak identik) — gateway harus "menerjemahkan" perbedaan ini di baliknya, dan penerjemahan yang gak lengkap bisa bikin fitur provider tertentu (misal system message, function calling) gak ke-cover dengan baik.
- **Fallback logic (kalau provider A gagal, coba provider B) belum diimplementasi di versi ini** — versi production biasanya nambahin `try/except` di `generate()` buat otomatis pindah provider kalau ada error/timeout, bukan cuma dispatch berdasarkan konfigurasi statis.
- **Kalau gateway sendiri ada bug, SEMUA request kena dampaknya** — karena sekarang jadi satu titik sentral, bug di sini bisa lebih luas dampaknya dibanding bug yang cuma ada di satu endpoint spesifik. Ini trade-off wajar dari sentralisasi (dapat kontrol terpusat, tapi juga jadi single point of failure kalau gak hati-hati).

### Kapan Dipakai
- Pakai LLM Gateway begitu SupportPilot mulai butuh dukung **lebih dari satu provider** (misal OpenAI buat kasus umum, Anthropic buat kasus tertentu yang lebih cocok modelnya), atau butuh **fallback** kalau satu provider down.
- Juga berguna begitu butuh nerapin logic yang sama ke SEMUA request LLM (rate limiting, cost tracking topik 22, logging) tanpa duplikasi kode di tiap endpoint.
- Kalau SupportPilot cuma punya satu endpoint dan gak ada rencana ganti provider, gateway sederhana ini opsional — tapi tetap bagus buat kebiasaan arsitektur yang scalable dari awal.

### Sering Ditanya Saat Interview
- **Apa manfaat utama LLM Gateway / provider abstraction?** — model switching & fallback, cost control & rate limiting terpusat, logging terpusat, dan konfigurasi terpusat — semuanya bisa diubah di satu tempat tanpa nyentuh kode pemanggil di banyak endpoint.
- **Kenapa endpoint `/chat` gak manggil OpenAI SDK langsung?** — supaya kode pemanggil gak coupled ke satu provider SDK spesifik; ganti provider atau nambah fallback cukup ubah gateway, gak perlu ubah tiap endpoint yang manggil LLM.
- **Apa risiko dari sentralisasi lewat gateway kayak gini?** — kalau gateway-nya sendiri ada bug, dampaknya lebih luas (kena ke semua request) dibanding bug yang cuma ada di satu tempat spesifik — trade-off dari dapat kontrol terpusat.
- **Method apa yang biasanya jadi interface utama LLM Gateway?** — biasanya satu method seperti `generate(messages, model=None) -> str` yang dipanggil kode lain tanpa peduli provider mana yang jalan di baliknya.

---

## 21. Streaming

### Apa itu?
Streaming adalah pola di mana LLM ngirim jawabannya **token demi token** begitu dihasilkan, bukan nunggu keseluruhan jawaban selesai baru dikirim sekaligus:
```
Tanpa streaming: Request → LLM → wait 10s → full response
Dengan streaming: LLM → token, token, token, ...
```
Di web, cara paling umum buat ngirim stream data dari server ke browser adalah **SSE (Server-Sent Events)** — protokol sederhana di atas HTTP biasa, di mana server ngirim event demi event lewat satu koneksi yang tetap terbuka, dan browser bisa "mendengarkan" event itu satu-satu begitu datang.

### Kenapa dibutuhkan?
Tanpa streaming, customer yang nanya sesuatu ke SupportPilot harus nunggu LLM selesai generate SELURUH jawaban (bisa 5-10 detik buat jawaban panjang) sebelum ngeliat satu huruf pun muncul di layar. Ini bikin chat terasa lambat dan "diam" — customer gak tau apakah sistemnya lagi kerja atau macet.

Streaming menyelesaikan ini dengan cara mengirim tiap token begitu LLM menghasilkannya, jadi customer langsung ngeliat jawaban "ngetik" secara bertahap di layar — mirip pengalaman ChatGPT. Ini gak bikin LLM-nya jadi lebih cepat generate jawaban lengkap, tapi bikin **persepsi kecepatan** jauh lebih baik karena customer langsung dapat feedback visual.

Konsep terkait yang penting dipahami: **backpressure** — kalau consumer (browser customer) lebih lambat "menerima" data dibanding kecepatan LLM menghasilkan token, data harus di-buffer di suatu tempat (biasanya ditangani otomatis oleh web server/framework), supaya token yang belum sempat dikirim gak hilang begitu aja.

### Cara Kerja
```
1. Customer kirim POST/GET ke /chat/stream
2. Backend mulai manggil LLM provider dengan opsi stream=True
3. Begitu LLM provider ngirim satu token/chunk, backend langsung
   nge-yield token itu ke response (belum nunggu token berikutnya)
4. Browser nerima tiap token via SSE, langsung ditampilkan ke UI
5. Proses berulang sampai LLM selesai (token terakhir dikirim)
```
FastAPI mendukung ini lewat `StreamingResponse`, yang nerima sebuah **async generator function** sebagai sumber datanya.

### Contoh Kode — Python

> **Catatan Python:** `async def` di bawah ini adalah fungsi **asynchronous** — fungsi yang bisa "pause" (berhenti sejenak) saat nunggu operasi I/O (misal nunggu token berikutnya dari LLM provider lewat network), lalu "resume" lagi begitu datanya siap, TANPA memblokir seluruh program buat ikutan nunggu. Ini beda dari fungsi biasa (`def`) yang begitu dipanggil harus selesai dulu sepenuhnya sebelum kode lain bisa jalan.
>
> **Catatan Python:** Fungsi ini juga sekaligus sebuah **generator**, ditandai lewat keyword `yield` di dalamnya: alih-alih `return` satu nilai lalu selesai, generator "menghasilkan" (yield) beberapa nilai satu per satu — tiap kali dipanggil lagi, dia lanjut dari titik terakhir dia berhenti. Ini pas banget buat streaming: kita gak punya "satu jawaban lengkap" buat di-`return` sekaligus, yang ada cuma token yang datang satu-satu, dan `yield` memungkinkan tiap token langsung dikirim begitu tersedia.

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import json

app = FastAPI()


async def generate_stream(message: str):
    """
    Async generator: manggil LLM provider dengan stream=True, lalu yield tiap
    token begitu datang (bukan nunggu jawaban lengkap dulu baru dikirim).
    """
    stream = gateway.openai_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": message}],
        stream=True,
    )
    for chunk in stream:
        token = chunk.choices[0].delta.content
        if token:
            # Format SSE: tiap event diawali "data: " dan diakhiri baris kosong
            yield f"data: {json.dumps({'token': token})}\n\n"


@app.get("/chat/stream")
async def chat_stream(message: str):
    """
    Endpoint streaming: balikin StreamingResponse yang isinya diambil dari
    generate_stream() di atas, dikirim ke browser sebagai SSE.
    """
    return StreamingResponse(generate_stream(message), media_type="text/event-stream")


# Contoh pemakaian di sisi browser (JavaScript, bukan bagian topik ini):
# const source = new EventSource("/chat/stream?message=halo");
# source.onmessage = (event) => console.log(JSON.parse(event.data).token);
```

### Trade-off & Pitfall
- **Streaming lebih rumit di-debug dan di-test** dibanding request/response biasa — response-nya gak "satu nilai utuh" yang gampang di-assert, tapi serangkaian event yang datang bertahap.
- **Gak semua client mendukung SSE dengan mudah** (misal beberapa environment mobile/proxy corporate kadang buffer response sebelum diteruskan, yang bikin efek streaming "hilang" di sisi customer walau backend-nya sudah benar ngirim per-token) — perlu dites di environment production yang sesungguhnya, bukan cuma localhost.
- **Backpressure jadi pertimbangan nyata kalau volume traffic tinggi** — kalau ribuan customer streaming bersamaan, tiap koneksi SSE yang terbuka lama makan resource server (memory, file descriptor); ini beda dengan request biasa yang cepat selesai dan resource-nya langsung dilepas.
- **Cost tracking (topik 22) jadi sedikit lebih rumit buat response streaming** — total token/biaya baru diketahui setelah SEMUA chunk selesai diterima, jadi logging biaya biasanya dilakukan di akhir stream, bukan di tengah-tengah proses streaming-nya.

### Kapan Dipakai
- Pakai streaming buat semua interaksi chat yang mengharapkan respons cepat secara persepsi — ini standar UX buat aplikasi chat berbasis LLM modern (mirip ChatGPT, Claude.ai, dst).
- Kalau jawaban LLM biasanya pendek (beberapa detik saja buat selesai lengkap) dan UX-nya bukan chat interaktif (misal batch processing di background), streaming manfaatnya lebih kecil — request/response biasa (topik 19) sudah cukup.
- Gak cocok dipakai kalau consumer-nya bukan browser interaktif (misal integrasi backend-to-backend yang cuma butuh hasil akhir) — di kasus ini request/response biasa lebih sederhana dan gak butuh kompleksitas SSE.

### Sering Ditanya Saat Interview
- **Kenapa streaming penting buat UX chat berbasis LLM?** — tanpa streaming, customer harus nunggu SELURUH jawaban selesai digenerate (bisa 5-10 detik) sebelum ngeliat apapun; streaming ngirim token bertahap sehingga customer langsung dapat feedback visual, meningkatkan persepsi kecepatan walau waktu total generate-nya sama.
- **Apa itu SSE, dan kenapa dipakai buat streaming LLM?** — Server-Sent Events, protokol sederhana di atas HTTP biasa di mana server ngirim event demi event lewat satu koneksi yang tetap terbuka; dipakai karena simpel diimplementasikan dan didukung native lewat `EventSource` di browser.
- **Apa itu backpressure dalam konteks streaming?** — situasi di mana consumer (browser) lebih lambat menerima data dibanding kecepatan producer (LLM) menghasilkan token; data yang belum sempat dikirim perlu di-buffer, bukan hilang begitu saja.
- **Apa perbedaan fungsi `async def` biasa dengan generator (`yield`) dalam konteks endpoint streaming ini?** — `async def` bikin fungsi bisa pause/resume saat nunggu I/O tanpa blocking program lain; `yield` bikin fungsi itu menghasilkan banyak nilai satu-satu (bukan satu return sekaligus) — kombinasi keduanya (async generator) pas buat kirim token LLM begitu tiap token itu tersedia.

---

## 22. AI Cost Management

### Apa itu?
AI Cost Management adalah praktik melacak (tracking) berapa biaya yang dihabiskan tiap kali SupportPilot manggil LLM, dan mengoptimalkan biaya itu tanpa mengorbankan kualitas jawaban secara signifikan. Hal-hal yang biasanya dilacak per request: **input tokens**, **output tokens**, **model** yang dipakai, **latency**, dan **estimasi biaya (cost)**.

### Kenapa dibutuhkan?
LLM provider nge-charge berdasarkan jumlah token yang diproses (input + output), dan harga per token beda-beda tiap model (model yang lebih pintar biasanya lebih mahal). Tanpa tracking, SupportPilot gak akan tau: fitur mana yang paling mahal biaya operasionalnya, apakah ada request yang boros token tanpa alasan jelas (misal context yang kelewat panjang), atau apakah biaya bulanan bakal proporsional sama pertumbuhan jumlah customer.

Tracking cost per request menyelesaikan ini dengan cara paling praktis: tiap kali `LLMGateway.generate()` dipanggil, kita catat berapa token yang dipakai dan estimasi biayanya, supaya bisa dianalisis belakangan (fitur mana yang paling boros, tren biaya harian/bulanan, dll). Begitu ada masalah biaya yang kedeteksi, ada beberapa lever optimasi yang bisa ditarik: pakai **model yang lebih kecil/murah** buat kasus yang gak butuh model paling pintar, **persingkat context** yang dikirim ke LLM (buang bagian yang gak relevan), **caching** buat request yang sering berulang (dibahas lengkap di Phase 18), **prompt optimization** (bikin instruksi lebih ringkas tanpa kehilangan makna), dan **batching** (gabungkan beberapa request kecil jadi satu panggilan, juga dibahas lebih lanjut di Phase 18).

### Cara Kerja
```
LLMGateway.generate() dipanggil
    → LLM provider balikin response (termasuk info usage: token count)
    → track_usage(response, model) ekstrak token count & hitung estimasi biaya
    → hasil tracking di-log (misal ke database/logging system)
    → reply akhir tetap dibalikin seperti biasa ke pemanggil
```

### Contoh Kode — Python
`track_usage` mengekstrak jumlah token dari response OpenAI, lalu menghitung estimasi biaya pakai price table sederhana (dict `model → harga per 1000 token`):

```python
PRICE_PER_1K_TOKENS = {
    # Harga ilustratif ($/1K token) — angka production sebaiknya disinkronkan
    # berkala dengan halaman pricing resmi tiap provider.
    "gpt-4o-mini": {"input": 0.00015, "output": 0.0006},
    "gpt-4o": {"input": 0.0025, "output": 0.01},
    "claude-3-5-sonnet-20241022": {"input": 0.003, "output": 0.015},
}


def track_usage(response, model: str) -> dict:
    """
    Ekstrak jumlah input/output token dari response LLM (format OpenAI),
    hitung estimasi biayanya berdasarkan PRICE_PER_1K_TOKENS, dan balikin
    semuanya sebagai dict yang siap di-log.
    """
    input_tokens = response.usage.prompt_tokens
    output_tokens = response.usage.completion_tokens

    # Kalau model belum ada di price table, anggap harganya 0 (belum diketahui)
    # daripada bikin error — lebih baik cost-nya keliatan 0 & jadi tanda buat dicek
    # daripada bikin seluruh request gagal cuma gara-gara tracking biaya.
    harga = PRICE_PER_1K_TOKENS.get(model, {"input": 0.0, "output": 0.0})
    estimated_cost = (input_tokens / 1000) * harga["input"] + (output_tokens / 1000) * harga["output"]

    return {
        "model": model,
        "input_tokens": input_tokens,
        "output_tokens": output_tokens,
        "estimated_cost_usd": round(estimated_cost, 6),
    }
```

Wire ke `LLMGateway.generate` supaya SETIAP request SupportPilot otomatis ke-log biayanya:
```python
class LLMGateway:
    # ... __init__ sama seperti topik 20 ...

    def generate(self, messages: list[dict], model: str | None = None) -> str:
        model = model or "gpt-4o-mini"

        if self.provider == "openai":
            response = self.openai_client.chat.completions.create(model=model, messages=messages)
            reply = response.choices[0].message.content
        elif self.provider == "anthropic":
            response = self.anthropic_client.messages.create(model=model, max_tokens=1024, messages=messages)
            reply = response.content[0].text
        else:
            raise ValueError(f"Provider tidak dikenal: {self.provider}")

        usage_info = track_usage(response, model)
        print(f"[cost-tracking] {usage_info}")  # produksi: kirim ke logging/monitoring system, bukan print

        return reply
```

### Trade-off & Pitfall
- **Price table (`PRICE_PER_1K_TOKENS`) gampang basi** — harga LLM provider berubah dari waktu ke waktu; kalau lupa update, estimasi biaya yang ditampilkan jadi gak akurat dibanding tagihan sesungguhnya.
- **Estimasi biaya di sini gak termasuk biaya lain** (misal biaya embedding buat RAG di Phase 3-4, biaya reranking di topik 16) — buat gambaran biaya total SupportPilot, semua komponen ini perlu di-aggregate bareng, bukan cuma biaya LLM generate saja.
- **`track_usage` yang gagal (misal karena `response.usage` gak ada di response tertentu) bisa bikin seluruh request gagal** kalau gak ditangani hati-hati — sebaiknya dibungkus supaya kegagalan tracking biaya gak sampai menggagalkan fitur utamanya (balikin jawaban ke customer).
- **Optimasi model lebih kecil/context lebih pendek bisa menurunkan kualitas jawaban** kalau diterapkan sembarangan — perlu dievaluasi (Phase 4 topik 18) sebelum diputuskan sebagai perubahan permanen, bukan cuma diasumsikan "pasti masih cukup bagus".

### Kapan Dipakai
- Wire `track_usage` ke SEMUA panggilan LLM sejak awal (lewat `LLMGateway`, bukan di tiap endpoint terpisah) — lebih gampang dapat gambaran biaya menyeluruh sejak hari pertama, dibanding nambahin tracking belakangan setelah biaya sudah membengkak tanpa jejak yaitu tanpa data historis buat dianalisis.
- Pakai data cost tracking buat memutuskan optimasi mana yang paling worth it — misal kalau satu fitur ternyata dominan menyumbang biaya, fokus optimasi (model lebih kecil/caching) di fitur itu dulu, bukan optimasi merata ke semua fitur.
- Caching dan batching (disinggung di sini, dibahas lengkap di Phase 18) baru relevan begitu volume request sudah cukup tinggi dan ada pola request yang repetitif — buat traffic kecil, dua optimasi ini manfaatnya belum signifikan.

### Sering Ditanya Saat Interview
- **Apa saja yang biasanya di-track dalam AI cost management?** — input tokens, output tokens, model yang dipakai, latency, dan estimasi biaya (cost) per request.
- **Sebutkan beberapa lever optimasi biaya LLM.** — pakai model lebih kecil/murah, persingkat context, caching (Phase 18), prompt optimization, dan batching (Phase 18).
- **Kenapa price table butuh dijaga tetap up-to-date?** — harga LLM provider berubah dari waktu ke waktu; price table yang basi bikin estimasi biaya jadi gak akurat dibanding tagihan sesungguhnya, sehingga keputusan optimasi bisa salah arah.
- **Kenapa cost tracking sebaiknya diletakkan di LLMGateway, bukan di tiap endpoint?** — supaya SEMUA request LLM otomatis ke-log biayanya dari satu tempat, tanpa perlu duplikasi logic tracking di tiap endpoint yang manggil LLM.

---

## 23. LangChain — LLM Orchestration Framework

### Apa itu?
LangChain (tersedia buat Python & JavaScript/Node.js) adalah framework buat "menyambungkan" komponen-komponen LLM application — **prompt template**, **model**, **retriever**, **output parser** — jadi satu pipeline (disebut **chain**), daripada nulis semua glue code itu manual satu-satu. Konsep intinya:
- **Chains** — rangkaian step yang dijalankan berurutan, disusun pakai operator `|` (disebut **LCEL**, LangChain Expression Language).
- **Prompt Templates** — template string dengan slot yang bisa diisi variabel, gantinya string formatting manual.
- **Output Parsers** — komponen yang "membersihkan"/mem-parsing hasil mentah dari model jadi bentuk yang lebih siap dipakai (misal string bersih, atau struktur JSON tertentu).
- **Retrievers** — komponen buat ambil dokumen relevan dari vector store (bakal jauh lebih relevan begitu Phase 4's RAG pipeline SupportPilot direvisit lagi nanti di konteks agent, Phase 6 dst).

### Kenapa dibutuhkan?
Begitu SupportPilot butuh compose banyak step LLM sekaligus (misal: susun prompt → retrieve dokumen → generate jawaban → parse hasilnya jadi format tertentu), nulis semuanya manual (seperti fungsi-fungsi di topik 19-22) mulai berulang dan berantakan begitu kombinasinya makin banyak — misal prompt A dipasangkan parser B di satu fitur, tapi prompt A yang sama dipasangkan parser C di fitur lain. LangChain menyediakan **interface standar** antar step (lewat `|` di LCEL) sehingga komponen-komponen ini bisa saling ditukar tanpa nulis ulang boilerplate penyambung tiap kali.

### Cara Kerja
```
prompt (ChatPromptTemplate) | model (ChatOpenAI) | output_parser (StrOutputParser)
    → .invoke({variabel}) menjalankan seluruh chain berurutan:
        1. prompt diisi variabel
        2. hasil prompt dikirim ke model
        3. hasil model diparsing lewat output_parser
    → hasil akhir dikembalikan
```

### Contoh Kode — Python
Contoh SupportPilot: klasifikasi sentimen pesan customer jadi satu kata (`positif`/`negatif`/`netral`), pakai LCEL chain:

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template(
    "Klasifikasikan sentimen pesan customer berikut dalam SATU KATA saja "
    "(positif/negatif/netral): {pesan}"
)
model = ChatOpenAI(model="gpt-4o-mini")
chain = prompt | model | StrOutputParser()

sentimen = chain.invoke({"pesan": "Kenapa refund saya belum cair juga, udah seminggu nunggu!"})
print(sentimen)
# diharapkan: "negatif"
```

### Contoh Kode — Node.js
Chain yang sama, versi LangChain JS:

```javascript
import { ChatOpenAI } from "@langchain/openai";
import { ChatPromptTemplate } from "@langchain/core/prompts";
import { StringOutputParser } from "@langchain/core/output_parsers";

const prompt = ChatPromptTemplate.fromTemplate(
  "Klasifikasikan sentimen pesan customer berikut dalam SATU KATA saja " +
  "(positif/negatif/netral): {pesan}"
);
const model = new ChatOpenAI({ model: "gpt-4o-mini" });
const chain = prompt.pipe(model).pipe(new StringOutputParser());

const sentimen = await chain.invoke({ pesan: "Kenapa refund saya belum cair juga, udah seminggu nunggu!" });
console.log(sentimen);
// diharapkan: "negatif"
```

### Cara Manual (From Scratch) — biar paham LangChain itu ngapain aja di balik layar
LangChain sebenarnya cuma bungkus rapi dari pola: **susun prompt → panggil API model → parse hasilnya**. Gak ada magic di dalamnya. Ini versi manual tanpa library LangChain sama sekali, buat kasus klasifikasi sentimen yang sama:

**Python (manual, cuma pakai SDK resmi OpenAI):**
```python
from openai import OpenAI

client = OpenAI()


def build_prompt(pesan: str) -> str:
    # Ini yang digantiin ChatPromptTemplate di LangChain — cuma string formatting
    return (
        "Klasifikasikan sentimen pesan customer berikut dalam SATU KATA saja "
        f"(positif/negatif/netral): {pesan}"
    )


def call_model(prompt: str) -> str:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
    )
    return response.choices[0].message.content


def parse_output(raw_text: str) -> str:
    # Ini yang digantiin StrOutputParser — di sini simpel, strip + lowercase
    return raw_text.strip().lower()


# "chain" manual: tinggal panggil fungsi berurutan
prompt = build_prompt("Kenapa refund saya belum cair juga, udah seminggu nunggu!")
raw_result = call_model(prompt)
sentimen = parse_output(raw_result)
print(sentimen)
# diharapkan: "negatif"
```

**Node.js (manual, pakai SDK resmi OpenAI):**
```javascript
import OpenAI from "openai";

const client = new OpenAI();

function buildPrompt(pesan) {
  // Setara ChatPromptTemplate — cuma string formatting biasa
  return `Klasifikasikan sentimen pesan customer berikut dalam SATU KATA saja (positif/negatif/netral): ${pesan}`;
}

async function callModel(prompt) {
  const response = await client.chat.completions.create({
    model: "gpt-4o-mini",
    messages: [{ role: "user", content: prompt }],
  });
  return response.choices[0].message.content;
}

function parseOutput(rawText) {
  // Setara StrOutputParser
  return rawText.trim().toLowerCase();
}

// "chain" manual: panggil fungsi berurutan
const prompt = buildPrompt("Kenapa refund saya belum cair juga, udah seminggu nunggu!");
const rawResult = await callModel(prompt);
const sentimen = parseOutput(rawResult);
console.log(sentimen);
// diharapkan: "negatif"
```

**Insight penting:** begitu SupportPilot punya lebih dari 2-3 step yang mau di-reuse berkali-kali dengan kombinasi berbeda (misal prompt klasifikasi sentimen + parser sederhana di satu fitur, tapi prompt yang sama + retriever + parser JSON terstruktur di fitur lain), nulis manual jadi berantakan — di situlah LangChain kasih value: standardisasi interface antar step (`|` di LCEL) biar komponen bisa saling ditukar tanpa nulis ulang glue code tiap kali kombinasinya berubah. Buat satu-dua chain sederhana yang jarang berubah (seperti contoh klasifikasi sentimen di atas), versi manual sudah cukup dan lebih gampang di-debug karena gak ada abstraksi tambahan yang perlu dipahami.

### Trade-off & Pitfall
- **Abstraksinya kadang berlapis** — begitu ada masalah (misal chain gagal di step tertentu), kita perlu paham apa yang sebenarnya terjadi di balik `|` LCEL, gak cukup cuma baca kode chain-nya doang. LangSmith (tooling tracing resmi LangChain) membantu, tapi berarti ada tooling tambahan yang perlu di-setup.
- **Versi framework dan library pendukungnya (langchain, langchain-openai, dst) sering update dan kadang breaking change** — kode yang jalan di satu versi belum tentu jalan mulus di versi berikutnya tanpa penyesuaian.
- **Buat chain yang sangat sederhana (seperti contoh sentimen di atas), overhead belajar LangChain bisa lebih besar dibanding manfaatnya** — versi manual (Cara Manual di atas) sudah cukup dan lebih transparan buat kasus sesederhana ini.
- **Retriever LangChain baru benar-benar kepakai maksimal begitu terhubung ke RAG pipeline yang sudah dibangun** (Phase 4) — mempelajari retriever tanpa konteks RAG pipeline yang jelas bisa terasa abstrak.

### Kapan Dipakai
- Pakai LangChain begitu SupportPilot butuh compose banyak step LLM (prompt → retrieve → generate → parse) yang **sering di-reuse dengan kombinasi berbeda-beda** — standardisasi interface antar step mulai kasih value nyata di titik ini.
- Kalau chain-nya sederhana, jarang berubah, dan cuma dipakai di satu-dua tempat (seperti contoh klasifikasi sentimen di atas) — versi manual (Cara Manual) sudah cukup, lebih gampang dipahami tim yang belum familiar LangChain, dan lebih gampang di-debug.
- Retriever LangChain jadi makin relevan begitu Phase 4's RAG pipeline SupportPilot mau diintegrasikan ke pola chain yang lebih kompleks (misal dikombinasikan dengan agent di Phase 6).

### Sering Ditanya Saat Interview
- **Apa itu LCEL, dan apa fungsinya di LangChain?** — LangChain Expression Language, sintaks pakai operator `|` buat menyambungkan step-step (prompt, model, parser, dst) jadi satu chain yang dijalankan berurutan.
- **Apa yang sebenarnya terjadi di balik satu chain LangChain sederhana (prompt | model | parser)?** — sama persis dengan pola manual: susun prompt dari template, kirim ke API model, lalu parse hasil mentahnya — LangChain cuma menstandardisasi cara menyambungkan ketiga step itu.
- **Kapan LangChain mulai worth-it dibanding nulis manual?** — begitu ada banyak step yang mau di-reuse dengan kombinasi berbeda-beda (prompt A + parser B di satu fitur, prompt A + parser C di fitur lain) — standardisasi interface antar step jadi kasih value nyata; buat satu-dua chain sederhana yang jarang berubah, manual masih lebih transparan.
- **Apa peran retriever dalam LangChain, dan kapan itu relevan buat SupportPilot?** — retriever adalah komponen buat ambil dokumen relevan dari vector store, dipakai dalam pola RAG; jadi makin relevan begitu RAG pipeline SupportPilot (Phase 4) diintegrasikan ke dalam chain LangChain yang lebih kompleks.

---

**Selanjutnya:** [Phase 06 — Agents](./phase-06-agents.md)
