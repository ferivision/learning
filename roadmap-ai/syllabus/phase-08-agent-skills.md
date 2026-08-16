# Phase 08 — Agent Skills

> Bagian dari [AI Engineering / Agent Roadmap](../README.md)

---

## 34. What is a Skill?

### Apa itu?
Skill adalah paket pengetahuan/instruksi yang bisa dipakai ulang, dibundel bareng daftar tool yang relevan, buat menangani satu kategori task tertentu — misalnya "Coding Skill" (instruksi + tool buat nulis/review kode), "Research Skill" (instruksi + tool buat riset), atau di SupportPilot: "Refund Policy Skill" (instruksi + tool buat menangani permintaan refund). Bedanya dengan cara "default" yang sering dipakai orang di awal — jejalin SEMUA instruksi buat SEMUA kategori task ke dalam satu system prompt raksasa — adalah skill cuma **dimuat (loaded) saat dibutuhkan**, bukan selalu nempel di setiap request, apa pun yang sebenarnya ditanyakan customer.

### Kenapa dibutuhkan?
Bayangin SupportPilot punya instruksi detail buat: menangani refund, menangani komplain pengiriman, menangani pertanyaan seputar akun, menangani eskalasi VIP customer, dst — kalau semua itu ditulis sekaligus di satu system prompt dasar (yang dikirim di SETIAP request, topik 24-25), tiga masalah muncul: (1) **system prompt jadi bengkak** — makin banyak kategori task yang didukung, makin panjang prompt-nya, padahal buat pertanyaan simpel semacam "gimana status tiket T-123?" gak ada satu pun instruksi refund/eskalasi/dll yang benar-benar relevan; (2) **biaya dan latency naik** — token system prompt yang gak relevan tetap ikut terkirim dan dibayar di SETIAP request, bahkan yang gak nyentuh topik itu sama sekali; (3) **model bisa "ke-distract"** — instruksi yang gak relevan buat request saat itu tetap ada di konteks, berpotensi bikin model salah fokus. Skill menyelesaikan ini dengan memisahkan instruksi per kategori task ke file-nya sendiri-sendiri, lalu cuma dimuat ke konteks kalau memang dibutuhkan buat request yang sedang ditangani.

### Cara Kerja
```
Tanpa skill (system prompt raksasa, SELALU dikirim penuh):
  System prompt = instruksi refund + instruksi komplain pengiriman
                  + instruksi akun + instruksi eskalasi VIP + ...
  → dikirim UTUH di setiap request, apa pun pertanyaan customer

Dengan skill (dimuat sesuai kebutuhan):
  System prompt dasar = instruksi umum SupportPilot SAJA (pendek)

  Request masuk
      → cek: request ini butuh skill kategori apa? (misal: "refund" terdeteksi)
      → load_skill("refund_policy_skill")  → baca file skills/refund_policy_skill.yaml
      → tambahkan instructions dari skill itu ke system prompt SAAT ITU SAJA
      → tools yang ditawarkan ke model dipersempit ke tools milik skill itu
      → lanjut ke agent loop (topik 25) seperti biasa

  Request lain yang gak nyentuh refund sama sekali
      → skill refund_policy_skill TIDAK dimuat, system prompt tetap pendek
```
Satu skill = satu file (di SupportPilot, `.yaml` di folder `skills/`) berisi minimal dua bagian: **instructions** (teks panduan buat model, spesifik ke kategori task itu) dan **tools** (daftar nama tool yang relevan buat kategori itu — merujuk ke tool yang sudah didaftarkan di `TOOL_REGISTRY`, topik 25/28). `load_skill` cuma bertugas baca file itu dan mem-parsenya jadi dictionary yang gampang dipakai kode Python.

### Contoh Kode — Python
File skill buat kategori "refund" — `skills/refund_policy_skill.yaml`. Ini BUKAN kode Python, tapi file YAML yang dibaca oleh kode Python di bawahnya:
```yaml
name: refund_policy_skill
description: >
  Skill untuk menangani permintaan refund customer SupportPilot -
  mencakup instruksi kebijakan refund dan tool yang relevan.
instructions: |
  Ketika customer meminta refund, ikuti langkah berikut:
  1. Cek dulu status order lewat get_order_status. Refund cuma boleh
     diproses kalau status order "delivered", atau "delayed" dengan
     estimasi keterlambatan lebih dari 7 hari dari estimasi awal.
  2. Kalau order memenuhi syarat di atas, buka tiket refund lewat
     create_support_ticket dengan subject "Refund request - <order_id>".
  3. Jangan pernah menjanjikan refund akan cair dalam waktu tertentu -
     itu keputusan tim finance, bukan wewenang agent ini.
  4. Kalau order belum "delivered" dan belum telat, jelaskan ke customer
     bahwa refund baru bisa diajukan setelah barang diterima.
tools:
  - get_order_status
  - create_support_ticket
```

`load_skill` — baca file skill di atas dan parse jadi dictionary:
```python
import yaml


def load_skill(skill_name: str) -> dict:
    """
    Baca file skill YAML dari folder skills/, lalu parse isinya jadi
    dictionary berisi instructions dan daftar tool yang relevan. Ini yang
    memungkinkan agent "memuat" satu skill secara eksplisit, saat dibutuhkan
    saja - bukan selalu tersedia di system prompt dasar.
    """
    path = f"skills/{skill_name}.yaml"
    with open(path, "r", encoding="utf-8") as f:
        skill_data = yaml.safe_load(f)

    return {
        "name": skill_data["name"],
        "instructions": skill_data["instructions"],
        "tools": skill_data["tools"],
    }
```

Simulasi agent yang memuat skill `refund_policy_skill` HANYA kalau pesan customer memang menyinggung refund — kalau enggak, system prompt tetap pendek dan skill itu gak pernah disentuh:
```python
BASE_SYSTEM_PROMPT = (
    "Kamu adalah agent customer support SupportPilot. Gunakan tool yang "
    "tersedia buat cari data yang kamu butuhkan. Jawab customer secara "
    "langsung kalau informasi sudah cukup."
)

# tools lengkap SupportPilot (topik 28, Phase 6), dipersempit per-request
ALL_TOOLS = [
    {"type": "function", "function": {"name": "get_ticket_status"}},
    {"type": "function", "function": {"name": "get_order_status"}},
    {"type": "function", "function": {"name": "create_support_ticket"}},
    {"type": "function", "function": {"name": "escalate_to_human"}},
]


def prepare_agent_context(user_message: str) -> dict:
    """
    Tentukan system prompt dan daftar tool aktif buat SATU request, sebelum
    masuk ke run_agent_loop (topik 25). Skill refund cuma dimuat kalau
    pesannya memang mengandung kata "refund" - request lain (misal cek
    status tiket biasa) sama sekali gak menyentuh skill ini.
    """
    system_prompt = BASE_SYSTEM_PROMPT
    active_tool_names = {"get_ticket_status", "get_order_status", "escalate_to_human"}

    if "refund" in user_message.lower():
        skill = load_skill("refund_policy_skill")
        system_prompt = system_prompt + "\n\n" + skill["instructions"]
        active_tool_names = set(skill["tools"])

    active_tools = [t for t in ALL_TOOLS if t["function"]["name"] in active_tool_names]
    return {"system_prompt": system_prompt, "tools": active_tools}


# Request yang menyinggung refund -> skill dimuat, system prompt bertambah
context_refund = prepare_agent_context("Saya mau refund order O-456 dong")
print("refund" in context_refund["system_prompt"].lower())  # True
print([t["function"]["name"] for t in context_refund["tools"]])
# ['get_order_status', 'create_support_ticket']

# Request biasa -> skill TIDAK dimuat, system prompt tetap pendek
context_status = prepare_agent_context("Gimana status tiket T-123 saya?")
print("refund" in context_status["system_prompt"].lower())  # False
print([t["function"]["name"] for t in context_status["tools"]])
# ['get_ticket_status', 'get_order_status', 'escalate_to_human']
```

### Trade-off & Pitfall
- **Skill masih perlu "trigger" yang menentukan kapan dimuat** — di atas kita pakai cek string sederhana (`"refund" in user_message.lower()`), yang gampang meleset (customer bisa nulis "duit saya kembaliin dong" tanpa kata "refund" sama sekali); di sistem production, trigger ini biasanya diperkuat pakai klasifikasi (Phase 3) atau bahkan dibiarkan model sendiri yang memilih skill mana yang mau dimuat.
- **Skill yang terlalu banyak/terlalu granular jadi susah di-maintain** — kalau tiap skenario kecil dijadikan skill terpisah, jumlah file `skills/*.yaml` bisa membengkak dan malah bikin developer bingung skill mana yang harus dipakai buat kasus tertentu; skill idealnya dikelompokkan per kategori task yang cukup luas (refund, komplain pengiriman), bukan per-kalimat customer.
- **`instructions` dan `tools` di file skill harus tetap sinkron dengan `TOOL_REGISTRY`** — kalau skill menyebut tool yang namanya salah ketik atau belum didaftarkan di registry (topik 25/28), eksekusinya bakal gagal begitu model benar-benar mencoba memanggilnya.
- **Skill BUKAN pengganti tool** — skill gak bisa "melakukan" apa pun sendiri, dia cuma instruksi + daftar tool; eksekusi aksi nyata tetap lewat tool (topik 35 membahas ini lebih detail).

### Kapan Dipakai
- Pakai skill begitu SupportPilot punya lebih dari satu kategori task yang masing-masing butuh instruksi detail sendiri (refund, komplain pengiriman, eskalasi VIP, dst) — supaya system prompt dasar tetap pendek dan tiap kategori bisa dikembangkan/di-review terpisah.
- Kalau instruksinya cuma pendek dan berlaku buat SEMUA request (misal "selalu jawab dengan sopan"), taruh langsung di system prompt dasar — gak perlu dijadikan skill terpisah.
- Pertimbangkan skill kalau tim yang berbeda (misal tim finance buat refund, tim logistik buat komplain pengiriman) perlu bisa mengubah instruksi kategori mereka sendiri tanpa harus menyentuh system prompt dasar SupportPilot secara keseluruhan.

### Sering Ditanya Saat Interview
- **Apa itu skill, dan kenapa bukan cuma taruh semua instruksi di satu system prompt?** — skill adalah paket instruksi + tool yang bisa dipakai ulang untuk satu kategori task, dimuat cuma saat dibutuhkan; kalau semua instruksi dijejalin ke satu system prompt, promptnya bengkak, biaya/latency naik di setiap request (bahkan yang gak relevan), dan model berisiko ke-distract instruksi yang gak nyambung.
- **Apa isi minimal sebuah file skill?** — instructions (panduan spesifik buat kategori task itu) dan tools (daftar nama tool yang relevan, merujuk ke TOOL_REGISTRY yang sudah ada).
- **Kapan skill sebaiknya dimuat ke konteks?** — cuma saat request yang sedang ditangani memang butuh kategori itu (misal terdeteksi menyinggung refund) — bukan selalu ada di setiap request seperti system prompt dasar.
- **Apa risiko utama kalau terlalu banyak skill kecil-kecil dibuat?** — susah di-maintain karena developer jadi bingung skill mana yang cocok buat kasus tertentu; skill sebaiknya dikelompokkan per kategori task yang cukup luas, bukan per skenario super spesifik.

---

## 35. Tool vs Skill

### Apa itu?
Tool dan skill sering ketuker karena keduanya "dikasih" ke agent, tapi levelnya beda. **Tool** adalah sesuatu yang agent BISA LAKUKAN — satu fungsi konkret yang bisa dipanggil (`get_order_status(order_id)`), dengan input/output yang jelas, dan efeknya langsung (baca data, atau ubah state). **Skill** adalah pengetahuan/instruksi tentang BAGAIMANA menangani satu kategori task — bisa membungkus beberapa tool sekaligus plus panduan kapan dan bagaimana tool-tool itu dipakai bareng-bareng. Analogi sederhana: tool itu kayak satu alat di kotak perkakas (obeng, palu); skill itu kayak manual/SOP yang bilang "buat masalah kategori X, pakai obeng dulu buat langkah ini, baru palu buat langkah itu, dan jangan lupa cek Y sebelum mulai".

### Kenapa dibutuhkan?
Tanpa pembedaan yang jelas, developer gampang salah taruh sesuatu di tempat yang salah — misalnya mencoba menjadikan panduan kebijakan refund sebagai satu fungsi tool tunggal (`handle_refund()` yang isinya hardcode semua logic kebijakan), padahal itu sebenarnya adalah PENGETAHUAN yang lebih pas dijadikan instruksi buat model, dengan model sendiri yang memutuskan tool spesifik mana (`get_order_status`, `create_support_ticket`) yang perlu dipanggil dan kapan, berdasarkan hasil tiap langkah (persis pola agent loop, topik 25). Sebaliknya, kalau semua "cara menangani refund" ditulis ulang manual sebagai kode workflow tanpa memanfaatkan tool yang sudah ada, developer jadi duplikasi logic yang sebenarnya sudah dibungkus rapi lewat tool. Memahami tool vs skill bikin developer tahu persis: fungsi baru yang kita tulis itu SATU AKSI KONKRET (jadi tool), atau PANDUAN buat menangani satu kategori masalah yang mungkin butuh beberapa tool (jadi skill).

### Cara Kerja
```
TOOL — satu aksi konkret, callable, efeknya langsung:
  Nama    : get_order_status
  Input   : order_id (string)
  Aksi    : query ke sistem order, return data
  Output  : {"order_id": ..., "status": ..., ...}
  → didaftarkan di `tools` (schema) + TOOL_REGISTRY (fungsi asli), topik 25/28

SKILL — pengetahuan tentang MENANGANI satu kategori task,
        membungkus instruksi + REFERENSI ke beberapa tool:
  Nama         : refund_policy_skill
  Instructions : "cek get_order_status dulu, baru create_support_ticket
                  kalau syarat X terpenuhi, jangan janjikan waktu cair..."
  Tools        : [get_order_status, create_support_ticket]  <- cuma NAMA,
                                                                 bukan fungsi
                                                                 itu sendiri
  → dimuat lewat load_skill() (topik 34), lalu tool yang direferensikan
    di dalamnya tetap dieksekusi lewat TOOL_REGISTRY yang sama seperti biasa

Hubungan keduanya: skill TIDAK PERNAH menggantikan tool. Skill cuma
menunjuk tool mana yang relevan dan kapan memakainya; eksekusi aksi
nyatanya tetap sepenuhnya lewat tool (dan TOOL_REGISTRY) yang sudah ada.
```

### Contoh Kode — Python
Side-by-side: definisi TOOL `get_order_status` (schema + fungsi asli, gaya sama seperti Phase 6/topik 25) versus struktur SKILL `refund_policy_skill` (hasil `load_skill`, topik 34) — perhatikan `get_order_status` cuma sekadar NAMA di dalam `skill["tools"]`, bukan didefinisikan ulang:
```python
# --- TOOL: satu aksi konkret yang bisa dipanggil langsung ---
def get_order_status(order_id: str) -> dict:
    """Mock function: query status pengiriman sebuah order (Phase 6, topik 25)."""
    return {
        "order_id": order_id,
        "status": "delayed",
        "estimated_delivery": "2026-08-20",
    }


get_order_status_tool_schema = {
    "type": "function",
    "function": {
        "name": "get_order_status",
        "description": "Ambil status pengiriman sebuah order berdasarkan order_id.",
        "parameters": {
            "type": "object",
            "properties": {
                "order_id": {"type": "string", "description": "ID order, contohnya 'O-456'."}
            },
            "required": ["order_id"],
        },
    },
}


# --- SKILL: pengetahuan tentang cara menangani KATEGORI task "refund",
#     membungkus REFERENSI ke tool di atas, bukan implementasi ulang ---
refund_policy_skill = load_skill("refund_policy_skill")

print(refund_policy_skill["name"])
# 'refund_policy_skill'
print(refund_policy_skill["tools"])
# ['get_order_status', 'create_support_ticket']  <- cuma nama tool,
#                                                     bukan fungsi Python-nya
print(callable(get_order_status))                # True  -> tool BISA dipanggil
print(callable(refund_policy_skill["tools"]))    # False -> skill CUMA daftar nama
```
Perbedaannya jadi kelihatan jelas dari kode di atas: `get_order_status` adalah fungsi yang bisa langsung `callable(...)`-nya `True` — dia sendiri yang mengeksekusi aksi. `refund_policy_skill["tools"]` cuma `list[str]` berisi nama-nama tool yang relevan — dia gak bisa dieksekusi sama sekali, tugasnya cuma memberi tahu agent "buat kategori refund, tool-tool inilah yang relevan, dan begini cara memakainya" (lewat `refund_policy_skill["instructions"]`). Eksekusi aksi nyatanya tetap lewat `get_order_status` yang asli, dicari lewat `TOOL_REGISTRY` seperti biasa (topik 25).

### Trade-off & Pitfall
- **Jangan jadikan skill sebagai pengganti tool** — kalau butuh satu aksi konkret baru (misal `check_refund_eligibility(order_id)`), itu tetap harus jadi TOOL baru yang didaftarkan di `TOOL_REGISTRY`, bukan ditulis sebagai bagian dari `instructions` skill — instructions cuma bisa "menyuruh" model, bukan benar-benar mengeksekusi apa pun.
- **Jangan jadikan tool sebagai pengganti skill** — kalau butuh panduan multi-langkah yang melibatkan beberapa tool sekaligus (kayak alur refund di atas: cek order dulu, baru bikin tiket), jangan dipaksa jadi SATU fungsi tool raksasa yang meng-hardcode seluruh logic; itu bikin model kehilangan fleksibilitas buat menyesuaikan langkah berdasarkan hasil tiap tool (persis alasan kenapa agent loop dibutuhkan, topik 24).
- **Skill yang mereferensikan tool yang gak ada di `TOOL_REGISTRY`** akan lolos saat `load_skill` (karena cuma baca string nama), tapi gagal total begitu model benar-benar mencoba memanggil tool itu — validasi bahwa semua nama di `skill["tools"]` benar-benar ada di registry sebaiknya dilakukan sebelum skill dipakai di production.
- **Menduplikasi instruksi tool ke dalam skill (dan sebaliknya)** bikin dua sumber kebenaran yang gampang out-of-sync — deskripsi "kapan pakai tool ini" idealnya cukup ada SEKALI, entah di `description` tool (buat konteks umum) atau di `instructions` skill (buat konteks spesifik kategori), jangan ditulis berulang-ulang di keduanya dengan risiko saling bertentangan.

### Kapan Dipakai
- Pakai **tool** kalau yang dibutuhkan adalah satu aksi konkret dengan input/output jelas yang bisa langsung dieksekusi kode (baca data, ubah data, panggil API eksternal) — polanya tetap sama seperti tool calling dasar (Phase 2, topik 8).
- Pakai **skill** kalau yang dibutuhkan adalah panduan tentang MENANGANI satu kategori task yang melibatkan beberapa tool sekaligus, plus aturan/kebijakan yang gak bisa direduksi jadi satu fungsi tunggal (misal syarat refund yang bercabang-cabang).
- Kalau ragu: tanya "apakah ini sesuatu yang agent LAKUKAN (satu aksi), atau sesuatu yang agent perlu TAHU (panduan menangani kategori masalah)?" — jawaban pertama berarti tool, jawaban kedua berarti skill.

### Sering Ditanya Saat Interview
- **Apa perbedaan mendasar antara tool dan skill?** — tool adalah satu aksi konkret yang bisa dieksekusi (fungsi callable dengan input/output jelas); skill adalah pengetahuan/instruksi tentang cara menangani satu kategori task, yang bisa membungkus referensi ke beberapa tool sekaligus tapi tidak bisa dieksekusi sendiri.
- **Apakah skill bisa menggantikan tool?** — tidak — skill cuma menunjuk tool mana yang relevan dan memberi panduan pemakaiannya; eksekusi aksi nyata tetap sepenuhnya lewat tool yang didaftarkan di TOOL_REGISTRY.
- **Kenapa `refund_policy_skill["tools"]` cuma berupa daftar string, bukan daftar fungsi?** — karena skill cuma perlu MEREFERENSIKAN nama tool yang relevan; fungsi aslinya tetap satu-satunya sumber kebenaran di TOOL_REGISTRY, supaya gak ada duplikasi definisi tool di banyak tempat.
- **Kapan sebaiknya sesuatu dijadikan tool baru, dan kapan dijadikan bagian dari skill?** — kalau butuh aksi konkret baru yang bisa dieksekusi, itu tool baru; kalau butuh panduan/kebijakan tentang cara memakai tool-tool yang sudah ada untuk kategori task tertentu, itu masuk ke instructions sebuah skill.

---

**Selanjutnya:** [Phase 09 — MCP](./phase-09-mcp.md)
