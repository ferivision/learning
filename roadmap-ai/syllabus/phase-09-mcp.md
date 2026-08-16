# Phase 09 — MCP

> Bagian dari [AI Engineering / Agent Roadmap](../README.md)

---

## 36. MCP (Model Context Protocol)

### Apa itu?
MCP (Model Context Protocol) adalah **protokol standar** buat menghubungkan agent/aplikasi AI ke tool, resource, dan prompt yang disediakan pihak lain — tanpa protokol standar ini, tiap integrasi (Slack, GitHub, database, dst) harus ditulis manual satu-satu dengan format berbeda-beda. Arsitekturnya selalu punya 3 pihak: **Agent** (LLM yang butuh data/aksi dari luar, misal `run_agent_loop` di Phase 6) → **MCP Client** (kode yang "berbicara" protokol MCP, connect ke satu MCP Server) → **MCP Server** (proses terpisah yang menyediakan **Tools** — fungsi yang bisa dipanggil, **Resources** — data yang bisa dibaca, dan **Prompts** — template prompt siap pakai). Bedanya sama tool calling biasa (Phase 2 topik 8, Phase 6 topik 28): di tool calling biasa, fungsi Python-nya hidup di proses yang sama dengan agent-nya; di MCP, tool itu hidup di proses/server terpisah yang dipanggil lewat protokol standar, sehingga bisa dipakai ulang oleh agent/aplikasi mana pun yang punya MCP client, bukan cuma satu aplikasi tunggal.

### Kenapa dibutuhkan?
SupportPilot sekarang cuma punya tool yang hidup di proses yang sama dengan agent-nya (`TOOL_REGISTRY`, Phase 6 topik 28) — cukup buat satu aplikasi, tapi begitu tool yang sama (misal `get_order_status`) mau dipakai juga oleh aplikasi lain (dashboard internal, bot Slack customer support, atau agent SupportPilot versi lain yang ditulis pakai framework berbeda), kita harus mengulang wiring-nya dari nol tiap kali — gak ada cara standar buat "menawarkan" tool itu ke aplikasi lain. MCP menyelesaikan ini dengan membungkus tool jadi sebuah **server** yang bicara protokol standar: sekali `get_order_status` didaftarkan sebagai MCP tool di satu MCP Server, aplikasi/agent mana pun yang punya MCP client bisa connect dan memanggilnya, gak peduli aplikasi itu ditulis pakai bahasa atau framework apa.

### Cara Kerja
```
Agent (mis. run_agent_loop, Phase 6)
    → MCP Client: connect ke MCP Server (lewat stdio/subprocess, atau HTTP)
    → MCP Client: initialize()          # handshake, tuker info kapabilitas
    → MCP Client: list_tools()          # (opsional) lihat tool apa aja yang tersedia
    → MCP Client: call_tool(nama, args) # panggil satu tool spesifik dengan argumen
    → MCP Server: eksekusi fungsi Python asli di baliknya, kembalikan hasilnya
    → MCP Client: terima hasil (CallToolResult), agent pakai hasil itu buat lanjut mikir
```
MCP Server-nya sendiri didefinisikan dengan mendaftarkan fungsi Python biasa lewat decorator (`@mcp.tool()`) — persis kayak Phase 6 topik 28 (definisikan fungsi, definisikan schema-nya), bedanya di sini SDK `mcp` yang otomatis bikinin schema-nya dari type hint dan docstring fungsi itu, gak perlu nulis manual `parameters` JSON Schema kayak di Phase 2 topik 8.

### Contoh Kode — Python
MCP Server minimal yang membungkus `get_order_status` (fungsi mock yang sama seperti Phase 6 topik 25) jadi satu registered tool, pakai SDK resmi `mcp`:
```python
# mcp_order_server.py
from mcp.server import MCPServer

# MCPServer adalah kelas utama SDK `mcp` buat bikin MCP Server: daftarkan nama
# server-nya, lalu daftarkan tool-tool lewat decorator @mcp.tool() di bawah.
mcp = MCPServer("supportpilot-order-server")


@mcp.tool()
def get_order_status(order_id: str) -> dict:
    """
    Ambil status pengiriman sebuah order berdasarkan order_id.

    Mock function, gaya sama seperti get_order_status di Phase 6 topik 25 —
    belum nyambung ke sistem order beneran. Type hint (order_id: str, -> dict)
    dan docstring ini yang dipakai `mcp` buat otomatis bikin schema tool-nya,
    gak perlu nulis manual "parameters" JSON Schema kayak di Phase 2 topik 8.
    """
    return {
        "order_id": order_id,
        "status": "delayed",
        "estimated_delivery": "2026-08-20",
        "carrier": "JNE",
    }


if __name__ == "__main__":
    # Jalankan server ini lewat transport "stdio": MCP Client bakal spawn file
    # ini sebagai subprocess dan komunikasi lewat stdin/stdout-nya.
    mcp.run(transport="stdio")
```

MCP Client minimal yang connect ke server di atas (di-spawn sebagai subprocess) dan manggil tool `get_order_status`:
```python
# mcp_order_client.py
import asyncio
import sys

from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

# StdioServerParameters mendeskripsikan CARA nge-spawn MCP Server-nya:
# jalankan `python3 mcp_order_server.py` sebagai subprocess, lalu bicara ke
# subprocess itu lewat stdin/stdout-nya (sesuai transport="stdio" di server).
server_params = StdioServerParameters(
    command=sys.executable,
    args=["mcp_order_server.py"],
)


async def main() -> None:
    # stdio_client() nge-spawn subprocess server-nya dan kasih kita sepasang
    # stream (baca/tulis) buat komunikasi protokol MCP dengan server itu.
    async with stdio_client(server_params) as (read_stream, write_stream):
        # ClientSession membungkus stream itu jadi objek MCP client yang bisa
        # kita pakai buat initialize(), list_tools(), dan call_tool().
        async with ClientSession(read_stream, write_stream) as session:
            await session.initialize()  # handshake wajib sebelum manggil tool apa pun

            result = await session.call_tool(
                "get_order_status", {"order_id": "O-456"}
            )
            # result.content berisi list konten hasil tool (di sini satu blok teks
            # JSON, karena get_order_status mengembalikan dict)
            print(result.content[0].text)


asyncio.run(main())
```
Menjalankan `python3 mcp_order_client.py` bakal nge-print:
```
{
  "order_id": "O-456",
  "status": "delayed",
  "estimated_delivery": "2026-08-20",
  "carrier": "JNE"
}
```
Perhatikan: `mcp_order_client.py` gak pernah `import` fungsi `get_order_status` secara langsung — dia cuma tau NAMA tool-nya (`"get_order_status"`) dan argumennya, persis kayak gimana `run_agent_loop` (Phase 6 topik 25) cuma tau NAMA tool dari model lalu lookup ke `TOOL_REGISTRY`. Bedanya, di sini "registry"-nya ada di proses lain (MCP Server), diakses lewat protokol standar, bukan dictionary Python biasa di proses yang sama.

### Trade-off & Pitfall
- **Overhead proses & latency tambahan** — tiap `call_tool` lewat MCP butuh round-trip lewat subprocess/protokol (serialize argumen, kirim, deserialize hasil), yang lebih lambat dibanding manggil fungsi Python langsung di proses yang sama (Phase 6 topik 28); gak worth it kalau tool-nya cuma dipakai satu aplikasi tunggal yang gak butuh dibagi ke aplikasi lain.
- **Satu server MCP mati bikin semua tool di dalamnya gak bisa diakses** — kalau `mcp_order_server.py` crash atau gak jalan, agent yang bergantung ke tool `get_order_status` lewat MCP client-nya bakal gagal total (`initialize()` atau `call_tool()` bakal raise error) — perlu penanganan error yang sama disiplinnya kayak tool calling biasa (Phase 6 topik 25: `try/except` di sekitar eksekusi tool).
- **SDK `mcp` masih aktif berkembang** — mirip LangChain/LangGraph (Phase 6 topik 23, 27), bentuk API-nya (nama kelas, cara registrasi tool) berpotensi berubah antar versi rilis; kode yang jalan mulus di satu versi SDK belum tentu jalan sama persis di versi lain.
- **Keamanan tool tetap berlaku penuh** — MCP cuma protokol transport buat MEMANGGIL tool, bukan pengganti allowlisting/validation/permission (Phase 6 topik 28); tool yang diekspos lewat MCP Server tetap harus divalidasi argumennya dan dibatasi aksesnya, karena siapa pun yang punya MCP client valid bisa manggilnya.

### Kapan Dipakai
- Pakai MCP begitu satu tool (atau katalog tool) perlu dipakai ulang oleh **lebih dari satu aplikasi/agent** — misal `get_order_status` mau diakses baik dari agent chat SupportPilot maupun dari bot Slack internal, tanpa nulis ulang wiring-nya di masing-masing aplikasi.
- Kalau tool-nya cuma dipakai satu aplikasi tunggal yang gak ada rencana dibagi ke aplikasi lain, tool calling biasa (Phase 6 topik 28, `TOOL_REGISTRY` di proses yang sama) lebih sederhana dan lebih cepat — gak perlu overhead subprocess/protokol MCP.
- Manfaat MCP makin kelihatan begitu jumlah tool ATAU jumlah aplikasi yang butuh tool itu bertambah — nilai penuhnya (satu client, banyak server) dibahas di topik 37.

### Sering Ditanya Saat Interview
- **Apa itu MCP, dan komponen utamanya apa aja?** — protokol standar buat menghubungkan agent ke tool/resource/prompt eksternal; tiga komponennya: MCP Client (bicara protokol, dipakai agent), MCP Server (menyediakan Tools/Resources/Prompts), dan protokol itu sendiri yang menjembatani keduanya.
- **Apa beda MCP tool dengan tool calling biasa (Phase 2/Phase 6)?** — konsepnya sama (fungsi + schema + eksekusi), bedanya tool di MCP hidup di proses/server TERPISAH yang dipanggil lewat protokol standar, sehingga bisa dipakai ulang lintas aplikasi; tool calling biasa hidup di proses yang sama dengan agent-nya.
- **Bagaimana MCP Client tau schema/argumen sebuah tool tanpa import fungsinya langsung?** — MCP Server otomatis membuat schema tool dari type hint dan docstring fungsi yang didaftarkan lewat `@mcp.tool()`, lalu mengekspos schema itu lewat protokol (`list_tools()`) supaya client bisa tau cara manggilnya tanpa perlu akses ke kode sumber fungsi itu.
- **Kenapa `initialize()` wajib dipanggil sebelum `call_tool()`?** — itu langkah handshake protokol MCP (tuker informasi kapabilitas antara client dan server) yang harus selesai dulu sebelum request lain (seperti `call_tool`) dianggap valid oleh server.

---

## 37. Why MCP?

### Apa itu?
Topik 36 nunjukin MCP buat SATU server; nilai sebenarnya dari MCP baru kelihatan penuh begitu ada BANYAK server. Tanpa protokol standar, tiap integrasi ke aplikasi luar (GitHub, Slack, database, knowledge base) butuh kode custom yang berbeda-beda: agent SupportPilot harus punya satu adapter khusus buat tiap aplikasi, masing-masing dengan cara autentikasi, format request/response, dan error handling sendiri-sendiri. Dengan MCP, semua integrasi itu bicara protokol yang SAMA — agent cukup punya SATU MCP client, yang bisa connect ke MCP Server MANA PUN (GitHub MCP, Slack MCP, DB MCP, atau MCP Server buatan sendiri kayak `mcp_order_server.py` di topik 36) dengan cara pemanggilan yang identik: `initialize()` lalu `call_tool()`.

### Kenapa dibutuhkan?
Bayangin SupportPilot butuh nambah kemampuan baru: agent-nya sekarang juga harus bisa `search_knowledge_base` (Phase 6 topik 28, wrapper di atas `retrieve_relevant_chunks` Phase 4) selain `get_order_status`. Tanpa MCP, "custom integration per app" berarti: kode yang manggil order system beda sama kode yang manggil knowledge base — dua cara koneksi berbeda, dua cara handle error berbeda, dan makin banyak aplikasi baru (Slack, GitHub, dst) makin banyak juga kode custom yang harus dirawat terpisah-pisah (`Agent → custom integration per app`, N aplikasi = N cara integrasi berbeda). Dengan MCP, tiap aplikasi baru itu cukup "berbicara" lewat MCP Server-nya masing-masing, dan agent SupportPilot memanggil semuanya lewat **client interface yang sama persis** (`Agent → MCP → GitHub MCP / Slack MCP / DB MCP` — N aplikasi, tapi cuma SATU cara integrasi yang perlu dipahami dan dirawat).

### Cara Kerja
```
TANPA standar (custom integration per app):
  Agent → adapter khusus Order System   (cara koneksi A, error handling A)
  Agent → adapter khusus Knowledge Base (cara koneksi B, error handling B)
  Agent → adapter khusus Slack          (cara koneksi C, error handling C)
  (N aplikasi = N cara integrasi berbeda yang harus dipelajari & dirawat terpisah)

DENGAN MCP (satu client, banyak server):
  Agent → MCP Client → MCP Server "Order"          (initialize + call_tool)
                     → MCP Server "Knowledge Base"  (initialize + call_tool -- SAMA)
                     → MCP Server "Slack"           (initialize + call_tool -- SAMA)
  (N aplikasi, tapi cuma SATU pola pemanggilan yang perlu dipahami:
   initialize() lalu call_tool() -- siapa pun MCP Server-nya)
```

### Contoh Kode — Python
MCP Server KEDUA yang berbeda dari topik 36: sebuah "knowledge base MCP server" hipotetis yang membungkus `retrieve_relevant_chunks` (Phase 4) jadi tool `search_knowledge_base`:
```python
# mcp_kb_server.py
from mcp.server import MCPServer

mcp = MCPServer("supportpilot-knowledge-base-server")


def retrieve_relevant_chunks_mock(query: str, top_k: int = 3) -> list[dict]:
    """
    Mock function, gaya sama seperti retrieve_relevant_chunks (Phase 4) --
    di real case fungsi itu query ke database vector beneran lewat `db_conn`
    (lihat Phase 4, dan search_knowledge_base di Phase 6 topik 28); di sini
    disederhanakan jadi data statis supaya fokus ke pola "satu client, banyak
    MCP server", bukan ke detail retrieval-nya.
    """
    chunks = [
        {"source": "faq-pengiriman.md", "content": "Estimasi pengiriman JNE reguler 3-5 hari kerja."},
        {"source": "kebijakan-eskalasi.md", "content": "Tiket yang telat lebih dari 5 hari otomatis layak dieskalasi."},
    ]
    return chunks[:top_k]


@mcp.tool()
def search_knowledge_base(query: str, top_k: int = 3) -> list[dict]:
    """Cari artikel/dokumentasi SupportPilot yang relevan dengan sebuah pertanyaan."""
    return retrieve_relevant_chunks_mock(query, top_k=top_k)


if __name__ == "__main__":
    mcp.run(transport="stdio")
```

Satu MCP client yang sama, dipakai buat manggil KEDUA server berbeda (`mcp_order_server.py` dari topik 36, dan `mcp_kb_server.py` di atas) lewat pola pemanggilan yang identik:
```python
# mcp_multi_client.py
import asyncio
import sys

from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client


async def call_server(script: str, tool_name: str, arguments: dict) -> None:
    """
    Satu fungsi generik buat connect ke MCP Server MANA PUN (asal jalan lewat
    transport stdio) dan manggil satu tool di dalamnya -- pola pemanggilannya
    (stdio_client -> ClientSession -> initialize -> call_tool) SAMA PERSIS
    gak peduli server-nya "Order" atau "Knowledge Base" atau server MCP lain
    (GitHub MCP, Slack MCP, dst) di kasus riil.
    """
    server_params = StdioServerParameters(command=sys.executable, args=[script])
    async with stdio_client(server_params) as (read_stream, write_stream):
        async with ClientSession(read_stream, write_stream) as session:
            await session.initialize()
            result = await session.call_tool(tool_name, arguments)
            print(f"[{script}] {result.content[0].text}")


async def main() -> None:
    # Server PERTAMA: order system (topik 36) -- lewat MCP Server "Order"
    await call_server("mcp_order_server.py", "get_order_status", {"order_id": "O-456"})

    # Server KEDUA: knowledge base (baru dikenalin di topik ini) -- lewat MCP
    # Server "Knowledge Base" yang BEDA sama sekali dari server pertama, tapi
    # dipanggil lewat interface client YANG SAMA PERSIS (call_server di atas)
    await call_server(
        "mcp_kb_server.py",
        "search_knowledge_base",
        {"query": "kebijakan eskalasi tiket lama", "top_k": 2},
    )


asyncio.run(main())
```
Menjalankan `python3 mcp_multi_client.py` bakal nge-print hasil dari KEDUA server, lewat satu client yang sama:
```
[mcp_order_server.py] {
  "order_id": "O-456",
  "status": "delayed",
  "estimated_delivery": "2026-08-20",
  "carrier": "JNE"
}
[mcp_kb_server.py] {
  "source": "faq-pengiriman.md",
  "content": "Estimasi pengiriman JNE reguler 3-5 hari kerja."
}
```
Itu inti dari "why MCP": `call_server()` gak perlu tau APA-APA soal detail internal `mcp_order_server.py` atau `mcp_kb_server.py` -- gak perlu tau order system-nya pakai API apa, atau knowledge base-nya pakai database vector apa (Phase 4). Yang dia tau cuma: kasih nama script server-nya, nama tool-nya, dan argumennya -- protokol MCP yang menjembatani sisanya. Nambah server MCP KETIGA (misal GitHub MCP di real case) cuma butuh satu baris tambahan manggil `call_server(...)` yang sama, bukan nulis adapter custom baru dari nol.

### Trade-off & Pitfall
- **"Satu client, banyak server" cuma jalan mulus kalau semua server memang comply ke protokol MCP** — begitu ada aplikasi yang belum punya MCP Server resmi (banyak aplikasi third-party belum tentu nyediain), kita tetap harus bikin adapter custom (atau MCP Server wrapper sendiri) buat aplikasi itu -- MCP mengurangi, bukan menghilangkan total, kebutuhan integrasi custom.
- **Menambah server MCP baru berarti menambah satu lagi proses yang bisa gagal** — makin banyak MCP Server yang di-spawn (Order, Knowledge Base, Slack, GitHub, dst), makin banyak juga titik kegagalan independen yang perlu dipantau; satu server down gak bikin server lain ikut down, tapi tetap butuh observability tersendiri per server.
- **Konsistensi penamaan tool antar server jadi tanggung jawab tersendiri** — kalau dua MCP Server berbeda kebetulan punya tool dengan nama mirip tapi perilaku beda, agent/developer bisa salah asumsi soal tool mana yang dipanggil; dokumentasi tool per server (description di `@mcp.tool()`) jadi makin penting begitu jumlah server bertambah.
- **Overhead subprocess terakumulasi begitu banyak server dipanggil bersamaan** — tiap `call_server()` di atas nge-spawn subprocess baru; buat SupportPilot yang butuh manggil banyak MCP Server sekaligus per request customer, ini bisa menambah latency yang perlu diperhitungkan dibanding tool calling biasa (Phase 6 topik 28) yang gak ada overhead subprocess sama sekali.

### Kapan Dipakai
- Pakai pola "satu client, banyak MCP server" begitu SupportPilot butuh integrasi ke **lebih dari satu aplikasi eksternal** (order system, knowledge base, Slack, GitHub, dst) dan pengin agent-nya punya SATU cara pemanggilan yang konsisten ke semuanya, bukan adapter custom per aplikasi.
- Prioritaskan MCP buat aplikasi yang sudah punya (atau berencana punya) MCP Server resmi/community -- makin banyak aplikasi yang comply ke protokol ini, makin besar juga manfaat "satu client, banyak server" dibanding effort bikin MCP Server sendiri buat tiap satu aplikasi.
- Kalau SupportPilot cuma pernah dan akan terus cuma butuh SATU integrasi eksternal, manfaat "banyak server" ini gak kerasa -- MCP Server tunggal (topik 36) atau bahkan tool calling biasa (Phase 6 topik 28) sudah cukup.

### Sering Ditanya Saat Interview
- **Apa masalah utama yang diselesaikan MCP dibanding custom integration per aplikasi?** — tanpa MCP, tiap aplikasi eksternal (GitHub, Slack, DB, dst) butuh adapter custom dengan cara koneksi dan error handling masing-masing; dengan MCP, semua aplikasi itu diakses lewat SATU protokol standar dan SATU client interface yang sama.
- **Kenapa `call_server()` di contoh topik ini bisa dipakai buat server yang beda-beda tanpa diubah?** — karena pola pemanggilan MCP (`stdio_client` → `ClientSession` → `initialize()` → `call_tool()`) protokolnya SAMA gak peduli server-nya apa; yang beda cuma nama script server, nama tool, dan argumennya -- bukan cara koneksinya.
- **Apakah MCP menghilangkan SELURUH kebutuhan integrasi custom?** — enggak sepenuhnya; aplikasi yang belum punya MCP Server resmi tetap butuh dibikinin satu (atau adapter custom) dulu -- MCP mengurangi biaya integrasi jangka panjang (satu pola dipakai berkali-kali), bukan menghilangkan biaya bikin server pertamanya.
- **Apa risiko begitu SupportPilot punya banyak MCP Server yang berjalan sekaligus?** — makin banyak proses independen yang bisa gagal (butuh observability per server) dan makin banyak juga overhead subprocess yang terakumulasi kalau banyak server dipanggil bersamaan dalam satu request customer.

---

**Selanjutnya:** [Phase 10 — Agent Orchestration](./phase-10-agent-orchestration.md)
