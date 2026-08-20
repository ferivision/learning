# Phase 11 — Agent Runtimes / Harnesses

> Bagian dari [AI Engineering / Agent Roadmap](../README.md)

---

## 42. Agent Runtime

### Apa itu?
Agent Runtime (sering juga disebut **harness**) adalah semua infrastruktur yang mengelilingi sebuah bare LLM call supaya jadi agent yang layak production — bukan cuma `run_agent_loop` (Phase 6, topik 25) doang. Loop itu sendiri cuma jantungnya; runtime adalah **seluruh badan** di sekitarnya: session management, context management, skill loading, memory, permission check, sandboxing, human approval, sampai scheduling/background execution. SupportPilot yang jalan di laptop lu sendiri buat belajar (Phase 6) beda jauh sama SupportPilot yang jalan di production, dilihat ribuan customer, dan bisa manggil tool yang efeknya nyata (bikin tiket, eskalasi ke manusia, dst) — bedanya ada di runtime ini.

### Kenapa dibutuhkan?
`run_agent_loop` dari Phase 6 udah cukup buat belajar konsep "mikir → tindakan → amati hasil → ulang". Tapi begitu SupportPilot beneran dipakai production dengan banyak customer sekaligus, banyak pertanyaan baru muncul yang gak dijawab loop itu sendiri: siapa yang lagi ngobrol (session)? Context percakapan mana yang perlu dikirim ke LLM biar gak overflow (Phase 7 topik 32)? Skill/instruksi mana yang relevan buat request ini (Phase 8)? Apakah role agent ini emang boleh manggil tool tertentu (topik 45)? Kalau tool-nya berisiko (eksekusi kode), gimana cara ngejalaninnya dengan aman (topik 43)? Kalau aksinya berbahaya, siapa yang harus approve dulu (topik 44)? Tanpa lapisan-lapisan ini, `run_agent_loop` yang "polos" gampang disalahgunakan atau kolaps begitu skalanya naik — inilah yang bikin tools seperti Hermes Agent (Phase 12) dan OpenClaw (Phase 13) relevan: mereka adalah runtime siap pakai yang udah menyediakan semua lapisan ini.

### Cara Kerja
Bare LLM call (Phase 1-2) dibandingkan dengan agent runtime production:
```
Bare LLM call:
  Prompt -> LLM -> Output
  (satu putaran, gak ada state, gak ada tool, gak ada pengaman apapun)

Production Agent Runtime (SupportPilot):
  User Request
      |
      v
  Session Management        <- siapa user-nya, sesi mana, riwayat percakapan
      |
      v
  Context Management        <- susun context yang dikirim ke LLM (Phase 7, topik 32)
      |
      v
  Skill Loading              <- muat instruksi/tools yang relevan (Phase 8)
      |
      v
  +---------------------------------------------+
  |         AGENT LOOP  (Phase 6, topik 25)      |
  |  OBSERVE -> THINK/DECIDE -> ACT -> REPEAT     |
  +--------------------+--------------------------+
                       |  (tiap kali ACT == tool call)
                       v
              Permission Check   (topik 45 di bawah)
                       |
                       v
              Sandboxed Execution (topik 43 di bawah)
                       |
             +---------+----------+
             v                    v
      Tool Execution        Human Approval
      (get_order_status,     (topik 44, untuk aksi
       create_support_          berbahaya seperti
       ticket, dst)            escalate_to_human)
             |
             v
      Memory (Phase 7)      <- simpan fakta/hasil penting
             |
             v
  Scheduling / Background Execution
  (jalan di belakang tanpa blocking user,
   atau terjadwal berkala)
             |
             v
        Jawaban ke Customer
```
Ide intinya: agent loop cuma satu kotak kecil di tengah diagram itu. Semua kotak di sekitarnya adalah "harness" — dan justru kotak-kotak itulah yang bikin sebuah agent layak dipakai production, bukan cuma demo.

### Contoh Kode — Python
`AgentRuntime` di bawah adalah versi kecil dari harness itu: strukturnya identik dengan `run_agent_loop` (Phase 6, topik 25), tapi setiap kali model minta tool call, eksekusinya WAJIB lewat `execute_tool_call` dulu — titik itulah tempat permission check (`check_permission`, topik 45) dan sandboxed execution (`SandboxedExecutor`, topik 43) benar-benar ditegakkan, sebelum tool-nya jalan. Masing-masing lapisan itu dibahas detail satu-satu di topik 43 dan 45 di bawah; di sini fokusnya cuma nunjukin gimana potongan-potongannya nyambung jadi satu:
```python
import json


class AgentRuntime:
    """
    Membungkus siklus yang sama seperti run_agent_loop (Phase 6, topik 25),
    tapi TIAP tool call yang diminta model wajib lewat execute_tool_call dulu
    -- LLM sama sekali gak bisa mem-bypass langkah ini karena eksekusi tool
    ada di kode kita, bukan di dalam LLM.
    """

    # Tool yang eksekusinya berisiko tinggi (mis. jalanin kode arbitrary)
    # dan karena itu wajib lewat SandboxedExecutor, bukan dieksekusi langsung.
    SANDBOXED_TOOLS = {"execute_code"}

    def __init__(self, client, agent_role: str, tool_registry: dict, sandbox: "SandboxedExecutor"):
        self.client = client
        self.agent_role = agent_role
        self.tool_registry = tool_registry
        self.sandbox = sandbox

    def execute_tool_call(self, tool_name: str, tool_arguments: dict):
        """
        Enforcement point: satu-satunya jalan buat sebuah tool benar-benar
        dieksekusi. Urutannya penting -- permission dicek DULU, sebelum kita
        peduli soal sandboxing atau eksekusi sama sekali.
        """
        if not check_permission(self.agent_role, tool_name):
            return {"error": f"Role '{self.agent_role}' tidak punya izin memanggil '{tool_name}'"}

        if tool_name not in self.tool_registry:
            return {"error": f"Tool '{tool_name}' tidak dikenal"}

        if tool_name in self.SANDBOXED_TOOLS:
            # Lolos permission check bukan berarti langsung dieksekusi mentah --
            # tool berisiko tinggi tetap wajib lewat sandbox (topik 43).
            return self.sandbox.run(tool_arguments.get("code", ""))

        try:
            return self.tool_registry[tool_name](**tool_arguments)
        except Exception as e:
            return {"error": str(e)}

    def run(self, user_message: str, tools: list[dict], max_steps: int = 5) -> str:
        """
        Struktur loop-nya sama persis dengan run_agent_loop (Phase 6, topik
        25): panggil call_llm_with_tools (Phase 2) berulang sampai model
        kasih jawaban final atau max_steps habis. Satu-satunya beda: tool
        dieksekusi lewat self.execute_tool_call, bukan langsung lookup ke
        registry mentah-mentah.
        """
        messages = [
            {
                "role": "system",
                "content": "Kamu adalah agent customer support SupportPilot.",
            },
            {"role": "user", "content": user_message},
        ]

        for _ in range(max_steps):
            result = call_llm_with_tools(self.client, messages, tools)

            if result["type"] == "message":
                return result["content"]

            tool_name = result["tool_name"]
            tool_arguments = result["tool_arguments"]
            tool_result = self.execute_tool_call(tool_name, tool_arguments)

            messages.append(
                {
                    "role": "assistant",
                    "tool_calls": [
                        {
                            "id": result["tool_call_id"],
                            "type": "function",
                            "function": {
                                "name": tool_name,
                                "arguments": json.dumps(tool_arguments),
                            },
                        }
                    ],
                }
            )
            messages.append(
                {
                    "role": "tool",
                    "tool_call_id": result["tool_call_id"],
                    "content": json.dumps(tool_result),
                }
            )

        return (
            "Maaf, saya butuh lebih banyak langkah untuk menyelesaikan "
            "permintaan ini. Tim support kami akan bantu proses lebih lanjut."
        )
```
Cara pakainya di SupportPilot: instead of manggil `run_agent_loop` polos kayak di Phase 6, production code bikin satu `AgentRuntime` per role (`support_agent`, `billing_agent`, dst — role-nya dijelasin di topik 45), dan setiap request customer dijalankan lewat `runtime.run(...)`, bukan lewat `run_agent_loop` langsung. Dengan begini, permission dan sandboxing otomatis berlaku di SETIAP tool call, tanpa engineer harus ingat nambahin cek itu manual di tiap tempat baru yang manggil agent.

### Trade-off & Pitfall
- **Lapisan tambahan = latency dan kompleksitas tambahan** — tiap tool call sekarang lewat `check_permission` dan (kalau relevan) `SandboxedExecutor` sebelum benar-benar jalan; ini penting buat keamanan, tapi tetap nambah sedikit overhead dibanding `run_agent_loop` polos di Phase 6.
- **Enforcement harus konsisten di SATU titik** — kalau ada jalur lain di codebase yang manggil tool langsung dari `TOOL_REGISTRY` tanpa lewat `AgentRuntime.execute_tool_call`, permission check dan sandboxing itu otomatis kebobol di jalur itu; harness cuma efektif kalau semua tool call BENERAN dipaksa lewat satu pintu yang sama.
- **Runtime yang makin banyak lapisan makin susah di-debug** — begitu ada 5-6 layer (session, context, skill, permission, sandbox, memory) di sekitar satu tool call yang gagal, nyari layer mana yang jadi sumber masalah butuh observability yang baik (Phase 15), bukan cuma baca satu log tunggal.
- **Jangan bangun semua lapisan ini dari nol kalau gak perlu** — runtime siap pakai seperti Hermes Agent (Phase 12) atau OpenClaw (Phase 13) udah menyediakan sebagian besar lapisan ini; membangun ulang dari nol cuma masuk akal kalau kebutuhan SupportPilot benar-benar spesifik dan gak terpenuhi runtime yang ada.

### Kapan Dipakai
- Pindah dari `run_agent_loop` polos (Phase 6) ke `AgentRuntime` (atau runtime pihak ketiga) begitu SupportPilot mulai punya tool yang efeknya nyata ke luar (bikin tiket beneran, eskalasi ke manusia beneran) dan/atau lebih dari satu role agent dengan izin yang berbeda-beda.
- Kalau agentnya masih prototipe/belajar dengan tool mock semua (seperti sepanjang Phase 6), `run_agent_loop` polos tetap cukup — gak perlu buru-buru bungkus semuanya dengan `AgentRuntime` sebelum ada kebutuhan nyata.
- Begitu ada tool yang bisa mengeksekusi kode arbitrary atau butuh approval manusia, itu sinyal kuat runtime lu butuh minimal lapisan permission (topik 45) + sandboxing (topik 43) + human approval (topik 44) di depan tool itu.

### Sering Ditanya Saat Interview
- **Apa beda "agent" dan "agent runtime/harness"?** — agent (Phase 6) adalah loop "mikir → tindakan → amati hasil"; runtime/harness adalah seluruh infrastruktur di sekitarnya (permission, sandboxing, memory, session, scheduling, human approval) yang bikin loop itu aman dan layak dipakai production.
- **Kenapa `run_agent_loop` dari Phase 6 belum cukup buat production?** — karena dia gak punya mekanisme buat menegakkan siapa boleh manggil tool apa (permission), gimana ngejalanin tool berisiko dengan aman (sandbox), atau siapa yang harus approve aksi berbahaya (human-in-the-loop) — semua itu perlu ditambahkan sebagai lapisan di sekitarnya.
- **Di titik mana persisnya enforcement (permission/sandbox) harus terjadi?** — tepat sebelum tool benar-benar dieksekusi (di `execute_tool_call`), bukan di dalam LLM — LLM cuma boleh "minta", kode kitalah yang memutuskan boleh/tidaknya dan gimana cara ngejalaninnya.
- **Apa risiko kalau ada jalur lain yang bypass `AgentRuntime` dan manggil tool langsung?** — semua permission check dan sandboxing yang udah dibangun jadi gak berlaku di jalur itu; harness cuma efektif kalau jadi satu-satunya pintu buat eksekusi tool.

---

## 43. Sandboxing

### Apa itu?
Sandboxing adalah menjalankan sebuah operasi berisiko (eksekusi kode, shell command, browsing web) di lingkungan yang **dibatasi**, supaya kalaupun operasinya jahat atau salah, dampaknya gak bisa nyampe ke sistem asli. `SandboxedExecutor` di bawah adalah versi kecil dan ilustratif dari itu: dia cuma boleh menjalankan potongan kode Python yang lolos allowlist ketat (gak ada `import`, gak ada akses ke fungsi berbahaya kayak `open`/`exec`/`eval`), dijalankan dengan `builtins` yang juga dibatasi.

**Penting buat digarisbawahi**: `SandboxedExecutor` di bawah TIDAK setara sandbox production. Sandbox production sungguhan pakai isolasi level OS — container Docker sekali pakai, micro-VM seperti Firecracker, atau remote sandbox service (mis. E2B) yang benar-benar memisahkan proses secara fisik/kernel. Filter berbasis AST kayak di bawah ini bisa saja ditembus teknik sandbox-escape Python yang lebih canggih (misal lewat introspeksi objek `__class__.__bases__`, dst). Kode ini nunjukin **prinsip dan interface**-nya (validasi sebelum eksekusi, allowlist ketimbang blocklist, restricted execution environment) — bukan jaminan keamanan level production.

### Kenapa dibutuhkan?
Tool seperti `execute_code()`, `run_shell()`, atau `browse_web()` pada dasarnya ngasih agent kemampuan buat berinteraksi langsung dengan sistem operasi atau internet — kalau dijalankan tanpa batasan apapun, agent (atau siapapun yang berhasil memanipulasi promptnya lewat prompt injection, Phase 14) bisa menghapus file, mengakses data sensitif di luar scope-nya, atau memicu efek yang gak diinginkan di luar SupportPilot sama sekali. Prinsip yang menjawab ini: **least privilege** (cuma kasih akses seminimal yang benar-benar dibutuhkan), **isolation** (jalankan di lingkungan terpisah dari sistem utama), dan **approval** (buat aksi yang benar-benar berisiko, minta konfirmasi manusia dulu — topik 44).

### Cara Kerja
```
Kode dari LLM (string)
    -> parse jadi AST (Abstract Syntax Tree), BUKAN langsung dieksekusi
    -> walk seluruh AST, cek satu-satu:
         - ada statement "import"?           -> REJECT
         - ada pemanggilan nama terlarang
           (open, exec, eval, os, subprocess, dst)? -> REJECT
         - ada akses ke dunder attribute
           (__class__, __globals__, dst)?     -> REJECT
    -> kalau semua lolos: compile & exec() dengan builtins yang
       DIBATASI ke allowlist (cuma print, len, sum, dst) -- bukan
       builtins Python yang penuh
    -> tangkap stdout, balikin sebagai hasil (atau pesan REJECT/ERROR)
```
Ini "defense in depth" dua lapis: pengecekan AST menolak pola kode yang jelas berbahaya SEBELUM dieksekusi sama sekali, dan `builtins` yang dibatasi jadi lapisan kedua kalau ada sesuatu yang lolos dari pengecekan AST.

### Contoh Kode — Python
```python
import ast
import builtins
import contextlib
import io

# Nama-nama yang gak boleh dipanggil sama sekali di dalam sandbox -- semuanya
# adalah pintu keluar dari lingkungan terbatas (baca file, jalankan proses,
# eksekusi kode dinamis).
DISALLOWED_NAMES = {
    "open", "exec", "eval", "compile", "__import__",
    "os", "subprocess", "sys", "socket", "shutil",
}

# Builtins yang benar-benar diperlukan buat operasi "hitung-hitungan" biasa --
# semua yang gak ada di sini otomatis TIDAK bisa dipanggil dari dalam sandbox,
# walau lolos dari pengecekan AST di bawah.
ALLOWED_BUILTIN_NAMES = {
    "print", "len", "range", "str", "int", "float",
    "sum", "min", "max", "sorted", "abs", "round",
}


class SandboxedExecutor:
    """
    Sandbox ILUSTRATIF berbasis allowlist AST -- bukan production-grade jail.
    Production sungguhan pakai Docker/Firecracker/remote sandbox service
    (isolasi level OS), bukan cuma filter di level source code Python seperti
    ini. Kode ini nunjukin PRINSIP-nya: validasi dulu sebelum eksekusi,
    allowlist ketimbang blocklist, dan restricted execution environment.
    """

    def run(self, code: str) -> str:
        try:
            tree = ast.parse(code, mode="exec")
        except SyntaxError as e:
            return f"[REJECTED] Syntax error: {e}"

        for node in ast.walk(tree):
            if isinstance(node, (ast.Import, ast.ImportFrom)):
                return "[REJECTED] Statement 'import' tidak diizinkan di sandbox"
            if isinstance(node, ast.Name) and node.id in DISALLOWED_NAMES:
                return f"[REJECTED] Pemanggilan '{node.id}' tidak diizinkan di sandbox"
            if isinstance(node, ast.Attribute) and node.attr.startswith("__"):
                return f"[REJECTED] Akses ke attribute '{node.attr}' tidak diizinkan di sandbox"

        safe_builtins = {
            name: getattr(builtins, name)
            for name in ALLOWED_BUILTIN_NAMES
            if hasattr(builtins, name)
        }
        restricted_globals = {"__builtins__": safe_builtins}

        output = io.StringIO()
        try:
            with contextlib.redirect_stdout(output):
                exec(compile(tree, "<sandboxed>", "exec"), restricted_globals, {})
        except Exception as e:
            return f"[ERROR] {e}"

        return output.getvalue()


sandbox = SandboxedExecutor()

# Operasi benign (cuma hitung-hitungan, print hasil) -> DIIZINKAN
benign_code = "print(sum([15000, 32000, 8000]))"
print(sandbox.run(benign_code))
# -> "55000\n"

# Operasi berbahaya: baca file di luar direktori yang diizinkan -> DITOLAK
disallowed_code = "open('/etc/passwd').read()"
print(sandbox.run(disallowed_code))
# -> "[REJECTED] Pemanggilan 'open' tidak diizinkan di sandbox"

# Operasi berbahaya lain: jalankan shell command lewat os -> DITOLAK
disallowed_code_2 = "import os\nos.system('rm -rf /')"
print(sandbox.run(disallowed_code_2))
# -> "[REJECTED] Statement 'import' tidak diizinkan di sandbox"
```
Tiga hasil di atas nunjukin `SandboxedExecutor` genuinely membedakan operasi benign dan berbahaya — bukan cuma diklaim di prosa: kode `sum([...])` biasa benar-benar dieksekusi dan hasilnya `55000` dikembalikan, sementara dua percobaan yang mengakses filesystem/OS di luar scope benar-benar ditolak SEBELUM `exec()` sempat dipanggil sama sekali.

### Trade-off & Pitfall
- **Allowlist berbasis AST bisa ditembus teknik yang lebih canggih** — sandbox-escape di Python asli (misal lewat rantai `().__class__.__bases__` buat nyampe ke object dasar, lalu ke subclass yang "berguna") bisa aja lolos dari pengecekan sederhana macam ini; makanya production butuh isolasi level OS (Docker/Firecracker), bukan cuma filter source code.
- **Allowlist yang terlalu ketat bikin tool jadi gak berguna** — kalau `ALLOWED_BUILTIN_NAMES` kekecilan, kode legit yang sebenarnya aman (misal butuh `list()` atau `dict()`) malah ditolak; ini trade-off klasik antara keamanan dan kegunaan, dan harus terus di-tuning berdasarkan use case nyata.
- **Sandbox yang lambat bikin latency tool call membengkak** — production sandbox (container/VM) butuh waktu buat spin-up per eksekusi; ini salah satu alasan kenapa `execute_code()` biasanya tool yang jauh lebih mahal/lambat dibanding tool baca-data biasa seperti `get_order_status`.
- **Sandboxing bukan pengganti permission check** — sandbox membatasi APA yang bisa dilakukan sebuah eksekusi kode; itu gak menjawab pertanyaan "apakah role ini boleh manggil tool ini sama sekali" — itu tugas `check_permission` (topik 45), dan keduanya harus dipakai BERSAMAAN, bukan salah satu doang.

### Kapan Dipakai
- Pakai sandboxing (minimal prinsipnya) buat SETIAP tool yang bisa mengeksekusi kode arbitrary, shell command, atau mengakses filesystem/network secara terbuka — `execute_code()`, `run_shell()`, `browse_web()`, dan sejenisnya.
- Tool yang cuma baca data lewat fungsi yang sudah didefinisikan ketat (seperti `get_order_status`, `get_ticket_status` di Phase 6) gak butuh sandboxing seberat ini — risikonya udah dibatasi lewat validasi argumen (Phase 2 topik 8) dan permission (topik 45), karena fungsinya sendiri gak bisa dipaksa melakukan hal di luar yang dia tulis buat lakukan.
- Begitu SupportPilot beneran butuh eksekusi kode arbitrary di production (misal fitur "hitung custom" buat customer power-user), itu saat buat mempertimbangkan Docker/Firecracker/remote sandbox service sungguhan — bukan lagi cukup dengan filter AST seperti contoh ilustratif di atas.

### Sering Ditanya Saat Interview
- **Kenapa tool seperti `execute_code()` atau `run_shell()` berbahaya kalau gak dibatasi?** — karena mereka ngasih agent kemampuan berinteraksi langsung dengan sistem operasi/filesystem/network; tanpa batasan, kesalahan model (atau prompt injection, Phase 14) bisa memicu efek nyata yang gak diinginkan di luar scope yang dimaksudkan.
- **Apa tiga prinsip utama di balik sandboxing?** — Least Privilege (akses seminimal yang dibutuhkan), Isolation (jalankan di lingkungan terpisah), dan Approval (aksi berisiko tinggi tetap butuh konfirmasi manusia, topik 44).
- **Kenapa `SandboxedExecutor` di atas bukan sandbox production-grade?** — karena cuma memfilter di level source code Python (AST + restricted builtins), yang secara teoretis bisa ditembus teknik sandbox-escape yang lebih canggih; sandbox sungguhan butuh isolasi level OS seperti Docker atau Firecracker.
- **Kenapa allowlist lebih disukai dibanding blocklist buat sandboxing?** — blocklist gampang bocor (lu harus nebak SEMUA hal berbahaya di depan, dan gampang lupa satu), sedangkan allowlist secara default menolak apapun yang gak eksplisit diizinkan — jauh lebih aman kalau ada sesuatu yang belum kepikiran sebelumnya.

---

## 44. Human-in-the-Loop

### Apa itu?
Human-in-the-loop adalah pola menggantungkan (gate) aksi yang berisiko/berbahaya di belakang persetujuan manusia, sebelum aksi itu benar-benar dieksekusi — agent boleh MEMUTUSKAN mau melakukan sesuatu, tapi eksekusinya ditahan sampai ada manusia yang bilang "boleh". `require_human_approval(action_description: str) -> bool` di bawah adalah fungsi yang menegakkan pola ini.

### Kenapa dibutuhkan?
Beberapa aksi punya konsekuensi yang gak bisa "di-undo" dengan mudah kalau ternyata keputusan agent-nya salah: menghapus file, mengirim email ke customer, mengeksekusi SQL langsung ke database production, deploy code, atau — relevan buat SupportPilot — mengeluarkan refund uang customer. Kalau semua aksi ini dibiarkan jalan otomatis begitu agent "memutuskan" untuk melakukannya, satu kesalahan reasoning model (atau satu prompt injection yang sukses, Phase 14) bisa berakibat nyata dan mahal buat diperbaiki. Human-in-the-loop menyisipkan **titik henti wajib** sebelum aksi seperti itu benar-benar terjadi.

### Cara Kerja
```
Agent memutuskan mau melakukan aksi X
    -> require_human_approval("deskripsi aksi X")
         -> (di production) notifikasi terkirim ke human reviewer
            (Slack/dashboard/email), lalu BLOCK/tunggu -- bisa
            makan waktu menit sampai jam, bukan instan
         -> (di ilustrasi ini) prompt CLI sederhana lewat input()
    -> human approve?
         -> YA  : aksi X benar-benar dieksekusi
         -> TIDAK: aksi X dibatalkan, agent lapor balik ke customer
```
Implementasi CLI berbasis `input()` di bawah cuma buat ilustrasi interface & prinsipnya di lingkungan belajar; di production, fungsi ini akan mengirim notifikasi ke manusia lewat channel (Slack, dashboard internal, email) dan benar-benar BLOCK sampai mereka merespons, bukan menunggu input dari terminal.

### Contoh Kode — Python
```python
def require_human_approval(action_description: str) -> bool:
    """
    Placeholder produksi: di real system, fungsi ini akan mengirim notifikasi
    ke human reviewer (Slack/dashboard/email) dan BLOCK sampai mereka
    approve/reject -- bisa makan waktu menit/jam, bukan instan. Implementasi
    CLI sederhana pakai input() ini cuma buat ilustrasi interface & prinsipnya.
    """
    print(f"\n[APPROVAL REQUIRED] Agent mau melakukan aksi berikut:")
    print(f"  {action_description}")
    response = input("Setujui aksi ini? (yes/no): ").strip().lower()
    return response in ("yes", "y")


def escalate_to_human(ticket_id: str) -> dict:
    """
    Mock function, gaya sama seperti versi di Phase 6 (topik 28): pura-pura
    eskalasi sebuah tiket ke antrian agent manusia.
    """
    return {
        "ticket_id": ticket_id,
        "escalated": True,
        "assigned_to": "human-agent-queue",
        "escalated_at": "2026-08-20T09:00:00Z",
    }


def escalate_to_human_with_approval(ticket_id: str) -> dict:
    """
    Wiring human-in-the-loop DI DEPAN escalate_to_human -- agent boleh
    memutuskan "tiket ini perlu dieskalasi", tapi eksekusi eskalasinya
    ditahan sampai manusia approve dulu.
    """
    approved = require_human_approval(
        f"Eskalasi tiket {ticket_id} ke antrian agent manusia"
    )
    if not approved:
        return {
            "ticket_id": ticket_id,
            "escalated": False,
            "reason": "Approval ditolak oleh human reviewer",
        }
    return escalate_to_human(ticket_id)


def issue_refund(order_id: str, amount: int) -> dict:
    """
    Mock function: aksi HIPOTETIS yang belum ada di Phase 6 -- pura-pura
    mengeluarkan refund uang customer. Contoh klasik aksi yang WAJIB lewat
    human-in-the-loop karena melibatkan uang customer secara langsung.
    """
    return {"order_id": order_id, "refunded_amount": amount, "status": "refunded"}


def issue_refund_with_approval(order_id: str, amount: int) -> dict:
    approved = require_human_approval(
        f"Refund order {order_id} sebesar Rp{amount:,} ke customer"
    )
    if not approved:
        return {
            "order_id": order_id,
            "refunded_amount": 0,
            "status": "rejected",
            "reason": "Approval ditolak oleh human reviewer",
        }
    return issue_refund(order_id, amount)
```
Kalau `require_human_approval` dites langsung dengan menjawab `"no"` di prompt CLI-nya, `escalate_to_human_with_approval("T-555")` akan mengembalikan `{"ticket_id": "T-555", "escalated": False, "reason": "Approval ditolak oleh human reviewer"}` — `escalate_to_human` yang sesungguhnya SAMA SEKALI gak sempat dipanggil. Ini bedanya sama sekadar "prosa" — approval-nya genuinely menahan eksekusi, bukan cuma dicatat sebagai log setelah aksinya kadung jalan.

### Trade-off & Pitfall
- **Menambah latency yang gak bisa diprediksi** — begitu sebuah aksi butuh approval manusia, waktu tunggu jawaban customer jadi tergantung seberapa cepat manusia itu merespons, bisa detik sampai jam; UX-nya harus didesain buat kondisi ini (misal: "permintaan kamu sedang direview tim kami"), bukan berharap approval selalu instan.
- **Terlalu banyak aksi yang di-gate bikin manusia kelelahan (approval fatigue)** — kalau HAMPIR SEMUA aksi minta approval, reviewer manusia jadi asal klik "yes" tanpa benar-benar mengevaluasi, yang bikin seluruh mekanisme ini jadi formalitas kosong; gate cuma aksi yang benar-benar berisiko tinggi (uang, data sensitif, aksi irreversible).
- **`require_human_approval` berbasis `input()` gak scalable buat production** — versi CLI ini cuma buat ilustrasi; production butuh sistem notifikasi asinkron (webhook, queue, dashboard) yang bisa nunggu respons manusia tanpa nge-block satu proses/thread selamanya.
- **Approval yang datang telat/gak konsisten tetap harus ditangani** — kode yang memanggil `require_human_approval` harus punya fallback yang jelas (misal timeout, default deny) kalau manusianya gak merespons dalam waktu wajar — jangan biarkan aksi nge-hang selamanya menunggu approval yang gak pernah datang.

### Kapan Dipakai
- Gate aksi yang **irreversible atau berdampak finansial/data sensitif** di belakang human approval: hapus file, kirim email massal, eksekusi SQL production, deploy code, refund uang, eskalasi yang butuh keputusan manusia (seperti contoh `escalate_to_human_with_approval` dan `issue_refund_with_approval` di atas).
- Aksi read-only atau yang dampaknya kecil dan mudah di-reverse (misal `get_order_status`, `get_ticket_status`) TIDAK perlu human-in-the-loop — gate yang berlebihan cuma bikin sistem lambat tanpa manfaat keamanan nyata.
- Kombinasikan dengan `check_permission` (topik 45): permission check menjawab "role ini boleh manggil tool ini secara umum?", human approval menjawab "instance spesifik dari aksi ini, sekarang, butuh sign-off manusia?" — dua pertanyaan yang berbeda dan keduanya bisa berlaku sekaligus untuk tool yang sama.

### Sering Ditanya Saat Interview
- **Kapan sebuah aksi agent harus digantung di belakang human approval?** — kalau aksinya irreversible atau berdampak besar (uang, data sensitif, produksi) — bukan buat semua aksi, karena approval fatigue bikin manusia asal klik approve.
- **Apa perbedaan human-in-the-loop dengan permission check (topik 45)?** — permission check menjawab "apakah role ini secara umum boleh memanggil tool ini", human-in-the-loop menjawab "apakah instance spesifik dari aksi ini butuh sign-off manusia sebelum benar-benar dieksekusi" — bisa dipakai bersamaan.
- **Kenapa implementasi `require_human_approval` berbasis `input()` gak cocok buat production?** — karena bersifat sinkron dan blocking di satu proses; production butuh notifikasi asinkron (Slack/dashboard/webhook) yang bisa menunggu respons manusia tanpa mengunci satu thread/proses selamanya, plus mekanisme timeout/fallback kalau manusianya gak pernah merespons.
- **Bagaimana human-in-the-loop mencegah kerugian dari kesalahan reasoning model?** — dengan menahan eksekusi aksi berisiko sampai manusia benar-benar mengonfirmasi, sehingga satu kesalahan keputusan LLM (atau hasil prompt injection) gak otomatis berubah jadi kerugian nyata sebelum ada yang sempat mengecek.

---

## 45. Agent Permissions

### Apa itu?
Agent Permissions adalah scoping akses agent berdasarkan **role**-nya — bukan ngasih satu agent kredensial luas yang bisa melakukan apa saja, tapi membatasi tiap role cuma boleh memanggil tool tertentu yang memang jadi tanggung jawabnya. `check_permission(agent_role: str, action: str) -> bool` di bawah adalah fungsi lookup sederhana yang menegakkan batasan ini, didukung sebuah dictionary permission.

### Kenapa dibutuhkan?
Prinsipnya sama persis dengan security backend biasa: **least privilege**. Kalau agent `support_agent` SupportPilot dikasih akses penuh ke semua kredensial (database production, API pembayaran, kemampuan eskalasi, dst), satu kesalahan reasoning model atau satu prompt injection yang sukses (Phase 14) bisa langsung berakibat luas — model "hanya" perlu dibujuk melakukan SATU hal salah, dan dampaknya sebesar akses yang dia punya. Dengan scoping permission per-role (read-only DB tertentu, tool spesifik, direktori spesifik, command shell terbatas), kalaupun agent salah mengambil keputusan, dampaknya terbatas cuma sebesar scope yang dia punya, gak sampai ke seluruh sistem.

### Cara Kerja
```
PERMISSIONS = {
    "support_agent": [...tool yang boleh dia panggil...],
    "billing_agent": [...tool yang boleh dia panggil...],
    ...
}

check_permission(agent_role, action):
    ambil daftar action yang diizinkan buat agent_role itu
    (kalau role-nya gak dikenal sama sekali -> anggap daftar KOSONG,
     JANGAN anggap "boleh semua" -- default HARUS deny, bukan allow)
    -> True kalau action ada di daftar itu
    -> False kalau enggak
```
Ini murni dictionary lookup — sengaja sesederhana mungkin, karena kekuatan permission check justru dari kesederhanaannya: gampang di-review manusia ("role ini boleh apa aja?" tinggal baca satu dictionary), dan gampang dites.

### Contoh Kode — Python
```python
# Satu sumber kebenaran: role apa boleh manggil tool/aksi apa. Baris ini yang
# dibaca kalau ada pertanyaan "kenapa support_agent gak bisa manggil X?".
PERMISSIONS: dict[str, list[str]] = {
    "support_agent": [
        "get_ticket_status",
        "get_order_status",
        "create_support_ticket",
    ],
    "billing_agent": [
        "get_order_status",
        "issue_refund",
    ],
    "escalation_agent": [
        "escalate_to_human",
    ],
}


def check_permission(agent_role: str, action: str) -> bool:
    """
    Lookup sederhana: role yang gak dikenal (atau action yang gak ada di
    daftar role itu) otomatis DITOLAK -- default deny, bukan default allow.
    """
    allowed_actions = PERMISSIONS.get(agent_role, [])
    return action in allowed_actions


# support_agent MEMANG scoped buat baca status & bikin tiket -> diizinkan
print(check_permission("support_agent", "get_order_status"))
# -> True
print(check_permission("support_agent", "create_support_ticket"))
# -> True

# support_agent BUKAN scoped buat eskalasi manusia -- itu tanggung jawab
# escalation_agent -- jadi WAJIB ditolak, walau secara teknis tool-nya "ada"
print(check_permission("support_agent", "escalate_to_human"))
# -> False

# billing_agent juga gak scoped buat eskalasi -- role apapun yang gak
# eksplisit didaftarkan buat sebuah action, otomatis ditolak
print(check_permission("billing_agent", "escalate_to_human"))
# -> False

# role yang sama sekali gak dikenal (typo, atau belum pernah didaftarkan)
# -> default-nya ditolak, BUKAN dianggap "boleh semua"
print(check_permission("unknown_role", "get_order_status"))
# -> False
```
Lima pemanggilan di atas nunjukin `check_permission` genuinely menegakkan scoping-nya lewat dictionary lookup biasa: dua aksi yang memang ada di daftar `support_agent` kembali `True`, sementara `escalate_to_human` — walau itu tool asli yang ada di SupportPilot (Phase 6, topik 28) — kembali `False` buat `support_agent` maupun `billing_agent` karena action itu cuma didaftarkan di bawah `escalation_agent`. Role yang sama sekali gak dikenal pun default-nya ditolak, bukan diperlakukan seolah boleh apa saja.

### Trade-off & Pitfall
- **Permission dictionary butuh perawatan aktif** — begitu SupportPilot nambah tool baru, seseorang harus secara sadar memutuskan role mana yang boleh memanggilnya dan update `PERMISSIONS`; lupa melakukan ini bikin tool baru itu otomatis gak bisa dipanggil role manapun (aman, tapi bisa bikin bingung kenapa tool "gak jalan") — beda dari kelupaan di `TOOL_REGISTRY` (Phase 6) yang bikin error saat eksekusi.
- **Default HARUS deny, bukan allow** — `PERMISSIONS.get(agent_role, [])` sengaja mengembalikan list kosong (bukan melempar error atau mengembalikan "semua tool") kalau role-nya gak dikenal; kalau default-nya kebalik jadi "allow semua kalau role gak dikenal", satu typo nama role bisa membuka akses penuh tanpa disadari.
- **Permission scoping per-role belum menjawab semuanya** — `check_permission` cuma menjawab "role ini boleh manggil tool ini secara umum atau enggak"; dia gak menjawab "apakah instance spesifik dari pemanggilan ini butuh approval manusia" (itu tugas `require_human_approval`, topik 44) atau "apakah argumennya sendiri valid/aman" (itu tugas validasi & sandboxing, Phase 2 topik 8 dan topik 43) — ketiganya perlu dipakai bersamaan, bukan saling menggantikan.
- **Role yang terlalu granular bikin permission dictionary sulit di-maintain**, sementara role yang terlalu kasar (misal cuma satu role generic buat semua agent) mengalahkan tujuan least privilege sama sekali — granularity role perlu disesuaikan dengan struktur tim/tanggung jawab riil SupportPilot, bukan dibuat sembarangan.

### Kapan Dipakai
- Definisikan role dan scoping-nya SEJAK tool pertama yang bisa mengubah data (bukan cuma baca) didaftarkan — jangan tunggu sampai ada agent baru yang "kebetulan" bisa manggil tool yang seharusnya bukan tanggung jawabnya.
- Pisahkan role berdasarkan **domain tanggung jawab** yang jelas bedanya (seperti `support_agent` vs `billing_agent` vs `escalation_agent` di atas) — bukan berdasarkan siapa yang menulis kodenya atau kebetulan urutan development.
- Kombinasikan `check_permission` dengan `AgentRuntime.execute_tool_call` (topik 42) supaya penegakannya otomatis berlaku di SETIAP tool call, bukan harus diingat manual satu-satu di tiap tempat baru yang memanggil tool.

### Sering Ditanya Saat Interview
- **Apa prinsip di balik agent permission scoping?** — Least Privilege: tiap agent role cuma dapat akses seminimal yang benar-benar dibutuhkan buat tugasnya, bukan kredensial luas yang bisa melakukan apa saja.
- **Kenapa `check_permission` harus default DENY untuk role yang gak dikenal, bukan default ALLOW?** — supaya typo nama role atau role baru yang belum sempat dikonfigurasi tidak diam-diam mendapat akses penuh; default deny bikin kesalahan konfigurasi gagal dengan aman (fail closed), bukan gagal dengan berbahaya (fail open).
- **Apa yang TIDAK dijawab oleh `check_permission`, walau sebuah tool call lolos permission check?** — apakah argumen yang dikirim ke tool itu valid (Phase 2, topik 8), apakah eksekusinya perlu dijalankan di sandbox (topik 43), dan apakah instance spesifik dari aksi itu butuh persetujuan manusia (topik 44) — permission scoping cuma satu dari beberapa lapisan yang perlu berjalan bersamaan.
- **Kenapa role `support_agent` dan `escalation_agent` perlu dipisah, padahal keduanya "melayani customer yang sama"?** — karena eskalasi ke manusia adalah aksi dengan bobot tanggung jawab berbeda dari sekadar baca status/bikin tiket; memisah role membatasi blast radius kalau salah satu role ternyata disalahgunakan atau salah mengambil keputusan.

---

**Selanjutnya:** [Phase 12 — Hermes Agent](./phase-12-hermes-agent.md)
