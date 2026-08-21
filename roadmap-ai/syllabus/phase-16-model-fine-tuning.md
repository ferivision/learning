# Phase 16 — Model Fine-Tuning

> Bagian dari [AI Engineering / Agent Roadmap](../README.md)

---

## 63. Fine-Tuning

### Apa itu?
Fine-tuning adalah proses melatih ULANG model yang sudah ada (base model) dengan dataset contoh tambahan, supaya BOBOT internal model itu sendiri berubah. Bedanya sama prompting biasa: prompting cuma "ngasih instruksi" di tiap request tanpa mengubah apa pun di dalam model — begitu request selesai, model "lupa" instruksi itu. Fine-tuning sebaliknya PERMANEN mengubah cara model merespons, karena kita literally menyesuaikan angka-angka (parameters) di dalam neural network-nya lewat proses training tambahan (lihat Phase 1 topik 3 soal training pipeline).

Gampangnya: kalau prompting itu kayak ngasih instruksi tertulis ke pegawai baru setiap kali dia mau kerja, fine-tuning itu kayak melatih pegawai itu selama beberapa minggu sampai kebiasaan barunya "nempel" dan gak perlu diingatkan lagi.

### Kenapa dibutuhkan?
Fine-tuning itu tool buat mengubah PERILAKU model, bukan buat nambahin PENGETAHUAN ke model. Ini beda fundamental dari RAG (Phase 4), dan sering ketuker:

- **RAG** — model tetap sama, tapi dikasih informasi TAMBAHAN lewat context (misal isi knowledge base SupportPilot yang berubah-ubah tiap hari). Cocok kalau kebutuhannya "model harus tau fakta X yang gak ada di training data-nya" atau "informasinya sering update".
- **Fine-tuning** — model diajarkan ULANG supaya PERILAKU defaultnya berubah, tanpa perlu instruksi ulang tiap request. Cocok kalau kebutuhannya:
  - **Konsistensi gaya/tone** — misal SupportPilot mau SEMUA jawaban model selalu pakai house style tertentu (singkat, format tetap, sapaan penutup yang sama) tanpa harus nulis system prompt panjang tiap kali.
  - **Perilaku spesifik domain** — model yang harus SELALU mengikuti format jawaban khusus industri (misal format laporan medis, format respons legal).
  - **Klasifikasi** — task sempit yang harus jalan cepat & murah dalam jumlah besar (misal klasifikasi urgency tiket support), di mana fine-tune model kecil lebih efisien daripada manggil model besar dengan prompt panjang tiap kali.
  - **Structured behavior** — model yang harus SELALU output dalam format tertentu secara konsisten, lebih reliable daripada mengandalkan instruksi prompt semata (walau structured output di Phase 2 sering udah cukup buat kebanyakan kasus).
  - **Specialist task** — model yang scope-nya sengaja dipersempit ke satu jenis tugas doang, dan performanya di tugas SEMPIT itu mau dimaksimalkan.

Aturan simpel buat mutusin: kalau masalahnya "model gak TAU sesuatu" → RAG. Kalau masalahnya "model tau, tapi gak PERILAKU/FORMAT-nya sesuai yang kita mau secara konsisten" → fine-tuning.

### Cara Kerja
Alur fine-tuning lewat OpenAI API secara garis besar:

```
Base Model (gpt-4o-mini)
      │
      │  + dataset contoh percakapan (format JSONL)
      ▼
Upload File (client.files.create)
      │
      ▼
Fine-Tuning Job (client.fine_tuning.jobs.create)
      │   (training berjalan di infrastruktur OpenAI, butuh
      │    waktu menit-jam & MENGELUARKAN BIAYA NYATA)
      ▼
Fine-Tuned Model (ft:gpt-4o-mini-2024-07-18:org::abc123)
      │
      ▼
Dipakai kayak model biasa lewat client.chat.completions.create(model=...)
```

Tahapannya:
1. **Siapkan dataset** dalam format JSONL (satu baris = satu objek JSON, bukan satu array besar) — tiap baris berisi field `messages` dengan struktur yang SAMA kayak Chat Completions API (Phase 1 topik 1): `system`, `user`, `assistant`.
2. **Upload file dataset** ke OpenAI lewat `client.files.create(..., purpose="fine-tune")`, yang balikin sebuah file ID.
3. **Submit fine-tuning job** lewat `client.fine_tuning.jobs.create(training_file=<file_id>, model=<base_model>)`. Ini yang benar-benar MEMULAI proses training di sisi OpenAI.
4. **Tunggu job selesai** — statusnya bisa dicek lewat `client.fine_tuning.jobs.retrieve(job_id)`, dan begitu `status == "succeeded"`, job itu punya `fine_tuned_model` berupa ID model baru (formatnya `ft:<base>:<org>::<suffix>`).
5. **Pakai model hasil fine-tuning** kayak model biasa — cukup ganti parameter `model=` di `client.chat.completions.create(...)` dengan ID model hasil fine-tuning itu.

### Contoh Kode — Python
Kode di bawah nunjukin DUA hal: (1) cara nyusun dataset fine-tuning dalam format yang bener, dan (2) cara submit fine-tuning job-nya lewat API. Skenarionya: SupportPilot mau model yang SELALU jawab singkat, sopan, dan nutup jawaban dengan kalimat penutup yang konsisten — daripada nulis instruksi itu di system prompt tiap request, tim SupportPilot mau "menanamkan" kebiasaan itu langsung ke model lewat fine-tuning.

Penjelasan sebelum baca kodenya:
- `SUPPORTPILOT_HOUSE_STYLE` — system message yang SAMA dipakai di semua contoh training. Konsistensi system message ini penting: itu yang bikin model belajar "kalau dapat system message kayak gini, jawabnya harus dengan gaya kayak gini".
- `write_fine_tuning_dataset(...)` — nulis tiap contoh sebagai SATU baris JSON (bukan satu file berisi array JSON besar) — ini format JSONL yang WAJIB dipakai fine-tuning API OpenAI.
- `client.files.create(..., purpose="fine-tune")` — upload file dataset; parameter `purpose` HARUS `"fine-tune"` biar OpenAI tau file ini buat training, bukan buat keperluan lain (misal Assistants API).
- `client.fine_tuning.jobs.create(training_file=..., model=..., suffix=...)` — submit job training. `training_file` diisi ID hasil upload, `model` diisi nama base model yang mau di-fine-tune (harus base model yang memang mendukung fine-tuning), `suffix` opsional buat kasih label yang mudah dikenali di ID model hasil akhirnya.

```python
import json

from openai import OpenAI

client = OpenAI()  # baca API key dari environment variable OPENAI_API_KEY

# System message yang KONSISTEN di semua contoh training -- ini yang
# "mengajarkan" house style SupportPilot ke model lewat fine-tuning.
SUPPORTPILOT_HOUSE_STYLE = (
    "Kamu adalah asisten customer support SupportPilot. Selalu balas dengan "
    "singkat (maksimal 3 kalimat), sopan, dan SELALU tutup jawaban dengan "
    "kalimat 'Ada lagi yang bisa dibantu?'"
)

training_examples = [
    {
        "messages": [
            {"role": "system", "content": SUPPORTPILOT_HOUSE_STYLE},
            {
                "role": "user",
                "content": "Kenapa order saya belum sampai padahal sudah 5 hari?",
            },
            {
                "role": "assistant",
                "content": (
                    "Order kamu saat ini masih dalam proses pengiriman dan "
                    "diperkirakan tiba dalam 1-2 hari kerja. Ada lagi yang "
                    "bisa dibantu?"
                ),
            },
        ]
    },
    {
        "messages": [
            {"role": "system", "content": SUPPORTPILOT_HOUSE_STYLE},
            {
                "role": "user",
                "content": "Gimana cara refund barang yang salah ukuran?",
            },
            {
                "role": "assistant",
                "content": (
                    "Kamu bisa ajukan refund lewat menu 'Pesanan Saya' > "
                    "'Ajukan Retur' dalam 7 hari sejak barang diterima. Ada "
                    "lagi yang bisa dibantu?"
                ),
            },
        ]
    },
    {
        "messages": [
            {"role": "system", "content": SUPPORTPILOT_HOUSE_STYLE},
            {
                "role": "user",
                "content": "Apakah bisa ganti alamat pengiriman setelah checkout?",
            },
            {
                "role": "assistant",
                "content": (
                    "Alamat pengiriman cuma bisa diganti sebelum status "
                    "order berubah jadi 'Diproses'. Setelah itu, silakan "
                    "hubungi kurir langsung. Ada lagi yang bisa dibantu?"
                ),
            },
        ]
    },
]


def write_fine_tuning_dataset(examples: list[dict], path: str) -> None:
    """Tulis contoh percakapan ke file JSONL sesuai format fine-tuning OpenAI.

    Tiap baris file adalah SATU objek JSON valid berdiri sendiri (bukan
    satu array JSON besar) -- ini format yang WAJIB dipakai OpenAI
    fine-tuning API.
    """
    with open(path, "w", encoding="utf-8") as f:
        for example in examples:
            f.write(json.dumps(example, ensure_ascii=False) + "\n")


DATASET_PATH = "supportpilot_finetune.jsonl"
write_fine_tuning_dataset(training_examples, DATASET_PATH)

# CATATAN PENTING: 3 contoh di atas cuma buat nunjukin MEKANISME dan
# FORMAT-nya. Fine-tuning yang beneran efektif butuh dataset ASLI dengan
# jumlah contoh jauh lebih banyak -- OpenAI merekomendasikan minimal
# puluhan sampai ratusan contoh berkualitas tinggi (idealnya ribuan buat
# hasil yang lebih stabil), yang beneran representatif terhadap variasi
# pertanyaan customer SupportPilot. Dataset kecil kayak di atas TIDAK
# akan menghasilkan model yang generalize dengan baik.

# Upload file dataset -- WAJIB dilakukan sebelum bikin fine-tuning job.
with open(DATASET_PATH, "rb") as dataset_file:
    uploaded_file = client.files.create(file=dataset_file, purpose="fine-tune")

# Submit fine-tuning job. MENJALANKAN baris ini akan MEMULAI PROSES
# TRAINING SUNGGUHAN di infrastruktur OpenAI -- MENGELUARKAN BIAYA NYATA
# (dihitung per token dataset training, dikali jumlah epoch) dan butuh
# waktu nyata (menit sampai jam, tergantung ukuran dataset & model).
# Contoh di sini menunjukkan MEKANISME pemanggilannya, bukan sesuatu
# yang harus dijalankan begitu saja tanpa dataset produksi yang matang.
fine_tuning_job = client.fine_tuning.jobs.create(
    training_file=uploaded_file.id,
    model="gpt-4o-mini-2024-07-18",
    suffix="supportpilot-style",
)

print(fine_tuning_job.id, fine_tuning_job.status)
# -> 'ftjob-abc123' 'validating_files'

# Status job bisa dicek berkala sampai selesai (biasanya lewat polling
# atau webhook, bukan blocking di satu request):
job_status = client.fine_tuning.jobs.retrieve(fine_tuning_job.id)
print(job_status.status, job_status.fine_tuned_model)
# -> 'succeeded' 'ft:gpt-4o-mini-2024-07-18:supportpilot::supportpilot-style'

# Setelah job 'succeeded', model hasil fine-tuning dipakai kayak model biasa:
# client.chat.completions.create(model=job_status.fine_tuned_model, messages=[...])
```

### Trade-off & Pitfall
- **Fine-tuning gak bisa "nambahin pengetahuan baru yang berubah-ubah"** — kalau kebutuhannya informasi yang sering update (harga, stok, kebijakan yang berubah), fine-tuning BUKAN solusinya karena model perlu di-training ULANG (biaya & waktu) setiap kali informasinya berubah. RAG (Phase 4) jauh lebih cocok buat kasus ini.
- **Butuh dataset berkualitas dalam jumlah cukup** — dataset kecil atau gak representatif bisa bikin model overfit ke contoh-contoh spesifik itu doang (menghafal, bukan generalize), atau malah gak berubah perilakunya sama sekali.
- **Biayanya nyata dan gak murah** — beda dari sekadar manggil API biasa, fine-tuning job dihitung berdasarkan jumlah token dataset training dikali jumlah epoch, DITAMBAH biaya inference model hasil fine-tuning biasanya lebih mahal per token dibanding base model-nya.
- **Model hasil fine-tuning bisa "lupa" kemampuan umum (catastrophic forgetting)** — kalau dataset training terlalu sempit/spesifik, model bisa jadi kurang jago di task-task lain yang gak ada di dataset training, walau jadi jauh lebih konsisten di task yang di-fine-tune.
- **Gak reversibel secara instan** — begitu model di-fine-tune, mengubah perilakunya lagi butuh fine-tuning ULANG (job baru), beda dari sekadar edit system prompt yang efeknya langsung kelihatan di request berikutnya.

### Kapan Dipakai
- Pakai fine-tuning kalau masalahnya PERILAKU/FORMAT yang harus konsisten TANPA harus nulis instruksi panjang tiap request, dan sudah punya dataset contoh yang cukup & representatif.
- JANGAN pakai fine-tuning cuma buat "ngajarin fakta baru" ke model — itu domain RAG.
- Pertimbangkan dulu apakah prompting/structured output (Phase 2) atau few-shot examples di prompt udah cukup buat kebutuhan yang ada — fine-tuning itu investasi (waktu + biaya + maintenance dataset), jadi baru relevan kalau prompting biasa udah gak cukup konsisten ATAU butuh model yang lebih murah/cepat buat task sempit yang high-volume.
- Cocok banget buat task klasifikasi/structured behavior yang high-volume, di mana biaya inference per-request jadi pertimbangan besar dan model kecil yang di-fine-tune bisa gantiin model besar yang di-prompt panjang.

### Sering Ditanya Saat Interview
- **Apa beda fine-tuning dan RAG?** — fine-tuning mengubah PERILAKU model secara permanen lewat training ulang bobotnya (butuh job training, biaya, waktu), RAG menambahkan PENGETAHUAN lewat context di setiap request tanpa mengubah model sama sekali (Phase 4).
- **Kapan fine-tuning lebih tepat dipakai daripada RAG?** — kalau kebutuhannya konsistensi gaya/format/perilaku yang gak berubah-ubah (bukan fakta yang sering update), atau buat task klasifikasi/specialist sempit yang high-volume di mana model kecil yang di-fine-tune lebih efisien.
- **Format apa yang dipakai buat dataset fine-tuning OpenAI, dan kenapa harus JSONL?** — format JSONL, satu baris satu objek JSON berisi field `messages` (struktur sama kayak Chat Completions API); dipakai karena file besar bisa diproses baris-per-baris tanpa harus parsing satu array JSON raksasa sekaligus.
- **Apa risiko utama fine-tuning dengan dataset yang terlalu kecil atau gak representatif?** — model bisa overfit/menghafal contoh spesifik itu doang tanpa generalize ke variasi input lain, atau malah gak menunjukkan perubahan perilaku yang diharapkan sama sekali.

---

## 64. LoRA / PEFT

### Apa itu?
LoRA (Low-Rank Adaptation) adalah salah satu teknik PEFT (Parameter-Efficient Fine-Tuning) — cara buat fine-tune model TANPA harus melatih ULANG semua parameter model itu. Idenya: BEKUKAN (freeze) semua bobot base model (gak diubah sama sekali), lalu tambahkan sepasang matriks kecil ("adapter") di beberapa layer tertentu yang DILATIH secara terpisah. Hasil akhirnya, cuma adapter kecil itu yang parameternya berubah selama training — base model-nya tetap persis sama kayak semula.

Analogi paling gampang: bayangin base model itu buku tebal yang gak boleh ditulisi langsung. LoRA itu kayak nempelin catatan tempel (sticky notes) kecil di halaman-halaman tertentu — catatan itu yang "mengoreksi/menyesuaikan" cara buku itu dibaca, tapi teks aslinya di buku gak pernah diubah.

### Kenapa dibutuhkan?
Full fine-tuning (melatih ULANG semua parameter model) itu MAHAL secara komputasi & memori — buat model dengan miliaran parameter, itu berarti nyimpen gradient dan optimizer state buat SETIAP parameter, yang butuh GPU/memori raksasa dan waktu training lama. LoRA menyelesaikan ini dengan cara:

```
FULL FINE-TUNING:
┌─────────────────────────────────┐
│   SEMUA parameter base model    │  ← semua dilatih ulang
│   (miliaran parameter)          │    (mahal: memori + compute)
└─────────────────────────────────┘

LoRA / PEFT:
┌─────────────────────────────────┐
│   Base model (FROZEN/dibekukan) │  ← TIDAK dilatih, cuma dipakai
│                                  │    buat forward pass
│      ┌──────────────────┐       │
│      │  Adapter LoRA     │  ←── │  ← HANYA bagian kecil ini
│      │  (matriks kecil,  │      │    yang dilatih (ribuan-jutaan
│      │  rank rendah)     │      │    parameter, bukan miliaran)
│      └──────────────────┘       │
└─────────────────────────────────┘
```

Karena cuma adapter kecil yang dilatih (biasanya <1% dari total parameter model), kebutuhan memori buat training turun drastis (gradient & optimizer state cuma buat parameter adapter, bukan seluruh model), training jadi jauh lebih cepat, dan hasilnya file adapter yang kecil (bisa cuma beberapa MB) yang bisa "dipasang-lepas" di atas base model yang sama — beda dari full fine-tuning yang hasilnya adalah copy LENGKAP model baru berukuran penuh.

### Cara Kerja
Secara teknis, LoRA menambahkan matriks tambahan berukuran kecil ("rank rendah", makanya nama "Low-Rank") di layer-layer tertentu (biasanya layer attention) dari base model. Selama training, HANYA matriks tambahan ini yang punya gradient dihitung dan diupdate — bobot asli base model tetap dibekukan (`requires_grad=False`). Saat inference, output dari base model dan output dari adapter LoRA DIGABUNG buat menghasilkan prediksi akhir.

Library `peft` dari Hugging Face menyediakan cara praktis buat menerapkan LoRA ke model apa pun dari `transformers`:
1. **Load base model** biasa lewat `AutoModelForCausalLM.from_pretrained(...)`.
2. **Definisikan `LoraConfig`** — konfigurasi rank adapter (`r`), skala (`lora_alpha`), layer mana yang mau dikasih adapter (`target_modules`), dan task type-nya.
3. **Bungkus model** dengan `get_peft_model(base_model, lora_config)` — ini yang otomatis membekukan base model dan nambahin adapter LoRA di layer yang ditentukan.
4. **Training berjalan seperti biasa** (loop training standar `transformers`/PyTorch), tapi cuma parameter adapter yang keupdate.

### Contoh Kode — Python
Kode di bawah nunjukin cara wrap model Hugging Face kecil (`distilgpt2`) dengan LoRA, dan mencetak berapa persen parameter yang beneran "trainable" — ini yang bikin klaim "LoRA jauh lebih hemat resource" jadi konkret, bukan cuma klaim di atas kertas. Kode ini SUDAH diverifikasi jalan (lihat catatan verifikasi di bawah).

Penjelasan sebelum baca kodenya:
- `AutoModelForCausalLM.from_pretrained("distilgpt2")` — load base model kecil (cocok buat contoh yang bisa dijalankan tanpa GPU besar; di produksi biasanya model yang jauh lebih besar).
- `LoraConfig(r=8, lora_alpha=16, ...)` — `r` adalah rank matriks adapter (makin kecil, makin sedikit parameter tambahan tapi makin terbatas kapasitas adaptasinya — 8 adalah nilai umum buat mulai). `lora_alpha` adalah faktor skala buat output adapter (biasanya diset 2x nilai `r` sebagai starting point).
- `target_modules=["c_attn"]` — nama layer di dalam arsitektur GPT-2 (dan turunannya kayak `distilgpt2`) yang jadi tempat nempelnya adapter LoRA; ini adalah layer attention gabungan query-key-value milik GPT-2.
- `bias="none"` — bias layer gak ikut dilatih (opsi lain: `"all"` atau `"lora_only"`).
- `fan_in_fan_out=True` — wajib diset `True` khusus buat arsitektur GPT-2 karena layer `c_attn`-nya pakai `Conv1D` (bentuk matriksnya "terbalik" dibanding `nn.Linear` biasa); tanpa ini, `peft` akan otomatis mengoreksinya sambil ngasih warning.
- `task_type=TaskType.CAUSAL_LM` — kasih tau `peft` bahwa model ini dipakai buat causal language modeling (Phase 1 topik 2), bukan task lain kayak sequence classification.
- `get_peft_model(...)` — bungkus base model jadi PEFT model: base model dibekukan otomatis, adapter LoRA ditambahkan di layer yang cocok dengan `target_modules`.
- `.print_trainable_parameters()` — method bawaan PEFT yang mencetak perbandingan jumlah parameter yang trainable vs total parameter model.

```python
from peft import LoraConfig, TaskType, get_peft_model
from transformers import AutoModelForCausalLM, AutoTokenizer

MODEL_NAME = "distilgpt2"  # model kecil, cocok buat contoh yang jalan tanpa GPU besar

tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME)
base_model = AutoModelForCausalLM.from_pretrained(MODEL_NAME)

lora_config = LoraConfig(
    r=8,                          # rank adapter -- makin kecil, makin hemat parameter
    lora_alpha=16,                # faktor skala output adapter (umumnya 2x nilai r)
    target_modules=["c_attn"],    # layer attention GPT-2 yang dikasih adapter LoRA
    lora_dropout=0.05,
    bias="none",
    fan_in_fan_out=True,          # wajib True untuk layer Conv1D khas arsitektur GPT-2
    task_type=TaskType.CAUSAL_LM,
)

peft_model = get_peft_model(base_model, lora_config)

peft_model.print_trainable_parameters()
# -> trainable params: 147,456 || all params: 82,060,032 || trainable%: 0.1797

# Verifikasi manual angka di atas: cuma adapter LoRA yang requires_grad=True,
# seluruh bobot asli distilgpt2 (82 juta parameter) tetap dibekukan.
trainable_params = sum(p.numel() for p in peft_model.parameters() if p.requires_grad)
total_params = sum(p.numel() for p in peft_model.parameters())
print(f"{trainable_params:,} dari {total_params:,} parameter dilatih "
      f"({100 * trainable_params / total_params:.4f}%)")
# -> 147,456 dari 82,060,032 parameter dilatih (0.1797%)
```

Angka di atas nunjukin klaimnya secara konkret: dari total 82 juta parameter `distilgpt2`, cuma sekitar 147 ribu (kurang dari 0.2%) yang beneran dilatih waktu fine-tuning pakai LoRA — sisanya (base model) TIDAK ikut dihitung gradient-nya sama sekali. Ini yang bikin LoRA bisa fine-tune model yang jauh lebih besar dengan memori GPU yang jauh lebih kecil dibanding full fine-tuning.

### Trade-off & Pitfall
- **`target_modules` harus sesuai arsitektur model** — nama layer attention BEDA-BEDA antar arsitektur (`c_attn` buat GPT-2, `q_proj`/`v_proj` buat model berbasis Llama, dst). Salah nama modul bikin `get_peft_model` gagal nemuin layer yang dimaksud atau (lebih berbahaya) diam-diam gak nempelin adapter ke layer yang seharusnya.
- **Rank (`r`) yang terlalu kecil bisa membatasi kapasitas adaptasi** — kalau task-nya butuh perubahan perilaku yang kompleks, rank rendah mungkin gak cukup buat capture itu; sebaliknya, rank yang terlalu besar mengurangi keuntungan efisiensi yang jadi alasan utama pakai LoRA.
- **LoRA tetap butuh base model dimuat penuh ke memori** buat forward pass (base model gak dilatih, tapi tetap dipakai buat inference) — penghematan LoRA itu di GRADIENT & OPTIMIZER STATE, bukan di ukuran model yang harus dimuat.
- **Hasil LoRA adalah adapter, bukan model utuh baru** — buat deploy, adapter ini biasanya perlu di-"merge" ke base model dulu (atau dimuat sebagai layer tambahan saat inference), beda dari full fine-tuning yang hasilnya langsung berupa satu model utuh yang siap pakai.

### Kapan Dipakai
- Pakai LoRA/PEFT kalau mau fine-tune model open-source (misal model dari Hugging Face) sendiri dengan resource GPU/memori terbatas — ini adalah cara paling umum buat fine-tune model besar tanpa infrastruktur skala enterprise.
- Cocok buat eksperimen cepat dengan banyak varian fine-tuning (misal nyoba beberapa dataset/task berbeda), karena tiap adapter kecil dan bisa "ditukar-tukar" di atas base model yang sama tanpa nyimpen banyak copy model penuh.
- Kurang relevan kalau pakai layanan fine-tuning terkelola kayak OpenAI fine-tuning API (topik 63) — di situ detail LoRA-vs-full-fine-tuning ditangani di sisi provider, bukan sesuatu yang dikonfigurasi manual oleh pengguna API.

### Sering Ditanya Saat Interview
- **Apa itu LoRA, dan kenapa disebut "parameter-efficient"?** — teknik fine-tuning yang membekukan seluruh bobot base model dan hanya melatih matriks adapter kecil ("rank rendah") yang ditambahkan di layer tertentu; disebut parameter-efficient karena cuma sebagian kecil (biasanya <1%) dari total parameter model yang benar-benar dilatih.
- **Apa beda LoRA dengan full fine-tuning?** — full fine-tuning melatih ULANG semua parameter model (butuh memori & compute besar buat gradient/optimizer state seluruh model), LoRA cuma melatih adapter kecil sambil base model tetap dibekukan.
- **Kenapa `target_modules` penting waktu setup `LoraConfig`?** — karena itu menentukan layer MANA di arsitektur model yang bakal dikasih adapter LoRA; nama layer ini beda-beda antar arsitektur model, jadi salah setting bisa bikin adapter gak terpasang di layer yang seharusnya.
- **Apakah LoRA mengurangi kebutuhan memori buat MEMUAT model, atau cuma buat TRAINING-nya?** — cuma buat training (gradient & optimizer state jadi jauh lebih kecil karena cuma buat parameter adapter); base model tetap harus dimuat penuh ke memori buat forward pass, sama seperti kalau dipakai inference biasa.

---

**Selanjutnya:** [Phase 17 — Advanced AI Architecture](./phase-17-advanced-ai-architecture.md)
