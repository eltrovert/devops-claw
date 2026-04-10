# Bab 2: Arsitektur OpenClaw Secara Mendalam

> *"Sebelum kamu nyetir mobil, kamu harus tahu dulu mana rem, mana gas, mana setir. Sama kayaknya, sebelum pakai OpenClaw, kamu perlu paham gimana bagian-bagiannya bekerja sama. Tenang, ini gak seserem kedengarannya."*

---

## Gambaran Besar Arsitektur

Blueprint gedung punya fondasi, struktur, instalasi listrik, plumbing, dan interior. Arsitektur OpenClaw juga punya lapisan-lapisan dengan tugas spesifik.

Penting untuk dipahami karena debugging, tuning performance, dan menambahkan fitur kustom. Tanpa pemahaman arsitektur, sama seperti mekanik yang bongkar mesin tapi tidak tahu komponennya.

OpenClaw punya arsitektur **modular dan event-driven** untuk kemudahan ekstensi dan ketahanan.

![Diagram arsitektur high-level OpenClaw yang menunjukkan semua komponen core dan relasinya dengan channel-channel eksternal](images/ch02-architecture-overview.png)

Di diagram di atas ada beberapa layer. Dari atas ke bawah: OpenClaw Core yang jadi "otak" utama, lalu Agent Runtime yang jadi lingkungan kerja agent, dan di bawahnya ada interface yang berkomunikasi dengan dunia luar. Semua komunikasi lewat Message Queue di tengah.

---

## Komponen Inti

### 1. Gateway Protocol: Pintu Gerbang Semua Komunikasi

Gateway Protocol itu seperti resepsionis di kantor. Semua tamu (pesan dari user) harus lewat resepsionis dulu sebelum diteruskan ke agent. Resepsionis ini memeriksa identitas tamunya dan memastikan mereka dikirim ke ruangan yang benar.

Secara teknis, Gateway Protocol menggunakan **WebSocket** untuk komunikasi antara client dan server. WebSocket ini berbeda dengan HTTP biasa. Kalau HTTP itu seperti email, kamu kirim terus menunggu balasan. WebSocket itu seperti telepon, sekali terhubung, dua arah bisa berkomunikasi real-time.

![Flow protokol Gateway menunjukkan komunikasi real-time dari client ke LLM dan kembali](images/ch02-gateway-protocol-flow.png)

Kenapa WebSocket? Karena di konteks DevOps, real-time itu penting banget. Kalau agent kamu sedang menjalankan command yang butuh 30 detik, kamu tidak ingin menunggu 30 detik tanpa feedback kan? WebSocket membuat agent bisa streaming output secara real-time.

**Tanggung jawab Gateway:**

- **Protocol handling**: Mengelola koneksi WebSocket di port 18789.
- **Session management**: Membuat, memelihara, dan merutekan session ke agent yang benar.
- **Authentication**: Memvalidasi token dan memeriksa permission.
- **Message routing**: Mengalirkan pesan ke agent yang tepat.
- **Multi-client support**: Web UI, CLI, mobile apps, semuanya terhubung ke Gateway.

> **Eits, Hati-hati!** Pastikan port 18789 tidak diblok firewall. Koneksi WebSocket yang gagal menyebabkan semua komunikasi mati.


### 2. Session Management: Memori Agent Anda

Session itu seperti short-term memory agent Anda. Tanpa session, setiap kali Anda chat, agent mulai dari nol. Dengan session, agent bisa melacak percakapan, ingat context, dan memberikan jawaban yang koheren.

Kenapa ini krusial di DevOps? Karena troubleshooting itu jarang cuma satu pertanyaan. Biasanya Anda bertanya "kenapa service X down?", terus agent menjawab, lalu Anda bertanya lanjutan "kalau gitu restart aja pod-nya". Tanpa session, agent tidak akan tahu "pod" yang mana.

OpenClaw mengatur percakapan menjadi **sessions**. Setiap pesan diarahkan ke session berdasarkan asalnya:

- **Direct messages**: Shared session secara default supaya agent bisa ingat semua percakapan Anda dari mana saja
- **Group chats**: Terisolasi per grup supaya konteks antar grup tidak bercampur
- **Rooms/channels**: Terisolasi per room sama seperti grup, tapi untuk platform berbasis channel
- **Cron jobs**: Session baru per eksekusi supaya setiap eksekusi cron dimulai bersih
- **Webhooks**: Terisolasi per hook supaya setiap webhook punya session sendiri

Tapi Anda juga bisa mengonfigurasi session isolation yang lebih granular untuk DM:

- `main` (default): semua DM berbagi satu session
- `per-peer`: isolasi berdasarkan pengirim
- `per-channel-peer`: isolasi berdasarkan channel + pengirim (direkomendasikan)
- `per-account-channel-peer`: isolasi berdasarkan account + channel + pengirim (paling ketat)

> **Pro Tip:** Gunakan `per-channel-peer` untuk engineer tim besar.


### 3. Agent Runtime: Otak Operasional

Kalau Gateway itu resepsionis, Agent Runtime itu eksekutif yang benar-benar bekerja. Di sinilah semua logika bisnis dieksekusi, semua interaksi terjadi, dan semua keputusan dibuat.

Agent Runtime menggabungkan Pi Agent Core dengan OpenClaw Layer yang menambahkan kemampuan spesifik seperti plugin system, context engine, dan UI tools. Jadi bukan cuma AI yang bisa ngobrol, tapi AI yang bisa *benar-benar bekerja* di infrastruktur Anda.

Agent berjalan dalam **workspace** - "ruang kerja" dengan akses ke files, skills, dan memory. Penting untuk keamanan dengan membatasi scope.

![Diagram layered architecture Agent Runtime, menunjukkan Pi Agent Core di dalam dan OpenClaw Layers di luarnya](images/ch02-agent-runtime.png)

**Bootstrap files** menyediakan identitas dan context. File seperti `AGENTS.md`, `SOUL.md`, `TOOLS.md`.

> **Eits, Hati-hati!** Jangan berikan akses penuh filesystem workspace tanpa batasan. Risiko keamanan serius. Selalu gunakan sandbox untuk eksekusi perintah.


### 4. Context Engine: Sistem "Memori" Agent

Context Engine itu seperti long-term memory agent. Kalau session itu short-term memory, Context Engine itu long-term memory (ingat fakta, pola, dan knowledge yang sudah terkumpul).

Kenapa ini berbeda jauh dari AI chat biasa? Karena AI chat biasa cuma punya short-term memory. Kalau Anda chat baru, mereka lupa semua. Context Engine membuat OpenClau bisa mempertahankan knowledge yang persisten, jadi agent Anda makin "pintar" seiring waktu.

Context Engine menggabungkan data dari beberapa sumber:

- **Session state**: State percakapan sekarang dan history
- **Memory search**: Penyimpanan memory jangka panjang dan fakta-fakta
- **Workspace files**: Bootstrap files, skills, dan context project
- **Tool results**: Hasil eksekusi tool terkini

Semua sumber ini digabung menjadi satu **context** yang dikirim ke model AI. Jadi setiap kali agent Anda merespon, dia punya akses ke informasi dari semua sumber ini secara bersamaan.


### 5. System Prompt Assembly: "Job Description" Agent

Setiap kali agent Anda ingin merespon pesan, OpenClaw tidak hanya mengirim pesan Anda ke model AI. Dia juga mengirim **system prompt**, yaitu instruksi-instruksi awal yang menentukan bagaimana agent harus berperilaku, tools apa yang bisa dipakai, dan batasan-batasan apa yang harus diikuti.

Bayangkan Anda mulai kerja baru. Bos Anda memberikan job description yang jelas: tugasmu apa, tools yang bisa Anda pakai apa, aturan yang harus Anda ikuti apa. System prompt fungsinya sama persis itu.

OpenClaw membangun system prompt ini secara **dinamis** setiap kali agent dieksekusi. Prompt-nya dikompilasi dari berbagai sumber:

- **Tooling**: Daftar tools yang tersedia dan cara pakainya
- **Safety**: Guardrail reminders
- **Skills**: Kalau ada skills, beri tahu model cara load dan pakai
- **Workspace info**: Working directory, project context
- **Bootstrap files**: `AGENTS.md`, `SOUL.md`, `TOOLS.md`, dll
- **Sandbox constraints**: Batasan runtime kalau sandboxing aktif
- **Current datetime**: Info timezone dan waktu sekarang

**Bootstrap files** yang di-inject ke system prompt:

- **`AGENTS.md`**: Konfigurasi agent dan capabilities vs Daftar tools yang boleh dipakai, persona
- **`SOUL.md`**: Kepribadian dan nada suara vs "Anda adalah SRE assistant yang friendly tapi teliti"
- **`TOOLS.md`**: Dokumentasi tools yang tersedia vs Cara pakai kubectl, docker, dll
- **`IDENTITY.md`**: Identitas agent vs Nama, role, spesialisasi
- **`USER.md`**: Preferensi pengguna vs Bahasa, format output, preferensi troubleshooting
- **`MEMORY.md`**: Konteks memory vs Fakta-fakta yang sudah dikumpulkan

Yang keren, Anda bisa edit file-file ini untuk menyesuaikan perilaku agent Anda. Misalnya edit `SOUL.md` untuk membuat agent Anda lebih *suka bicara* atau lebih *langsung ke intinya*.

### 6. Model Provider dan Failover

Kita sudah bahas di Bab 1 kalau OpenClaw mendukung 35+ provider AI. Sekarang kita masuk lebih dalam: gimana provider ini dikonfigurasi dan gimana failover-nya bekerja.

Failover itu seperti backup generator di gedung. Kalau listrik PLN mati, otomatis generator menyala. Di OpenClaw, kalau provider A bermasalah (down, rate-limited, timeout), otomatis beralih ke provider B. Anda tidak perlu intervensi manual.

```json5
// Contoh konfigurasi provider dengan failover
{
  "agents": {
    "defaults": {
      "model": "claude-sonnet-4-6",
      "provider": "anthropic",
      "maxTokens": 8192,
      "temperature": 0.3
    },
    "list": [
      {
        "id": "infra-agent",
        "name": "Infrastructure Agent",
        "provider": "anthropic",
        "model": "claude-sonnet-4-6",
        "auth": "my-auth-profile"
      }
    ]
  },
  "providers": {
    "anthropic": {
      "auth": {
        "my-auth-profile": {
          "apiKey": "${ANTHROPIC_API_KEY}"
        }
      }
    },
    "openai": {
      "auth": {
        "gpt-4": {
          "apiKey": "${OPENAI_API_KEY}"
        }
      }
    },
    "ollama": {
      "auth": {
        "local": {
          "endpoint": "http://localhost:11434"
        }
      }
    }
  }
}
```

**Failover flow:** Agent → primary (error) → secondary → semua gagal notifikasi user. Transparan tapi krusial untuk system 24/7.

> **Pro Tip:** Minimal 2 provider - satu primary, satu fallback.


### 7. Multi-Agent Architecture

Satu gateway, banyak agent. Konsep powerful tapi sering di-skip. Bayangkan perusahaan dengan divisi infra (server/network), deployment (CI/CD), dan security (vulnerability scan).

OpenClaw mendukung **multi-agent architecture**: setiap agent punya session, skill, model (kalau perlu), dan workspace sendiri. Semua berbagi satu gateway dan pool provider.

![Diagram Multi-Agent System yang menunjukkan beberapa agent berbagi Session Store](images/ch02-multi-agent.png)

**Agent routing** berdasarkan konfigurasi:
- Slack → "infra-agent"
- Telegram → "security-agent"  
- CI/CD webhook → "deploy-agent"

Spesialisasi: infra-agent (Kubernetes/server), security-agent (vulnerability scan), deploy-agent (pipeline/rollback).

### 8. Sandboxing System

Agent bisa menjalankan command berbahaya. Sandboxing membuat "ruang karantina" untuk eksekusi command, dampak terbatas di dalam karantina.

Penting banget karena AI *benar-benar* bisa execute `rm -rf`, `kubectl delete`, `docker system prune` di server. Satu hallucination = data production hilang.

```json5
// Konfigurasi sandbox
{
  "sandbox": {
    "enabled": true,
    "mode": "container",  // "container" atau "nodejs"
    "container": {
      "image": "openclaw/sandbox:latest",
      "network": "isolated",
      "readOnly": true,
      "allowCommands": ["bash", "kubectl", "docker"],
      "allowedPaths": ["/tmp", "/workspace"],
      "maxExecutionTime": 300000,  // 5 menit
      "memoryLimit": "512Mi"
    }
  }
}
```

**Mode sandbox:**
- **Container**: Docker container terisolasi, network terpisah, filesystem read-only (paling aman)
- **Node.js**: Sandbox ringan, cocok untuk pengembangan

**Fitur sandbox:**
- Network isolation (hanya yang allowed)
- Filesystem read-only
- Command allowlist
- Resource limits (CPU/memory)
- Timeout auto-kill

> **Eits, Hati-hati!** Jangan pernah matikan sandbox untuk kode dari sumber yang tidak terpercaya! Container isolation adalah garis pertahanan pertama.


### 9. Streaming dan Chunking

Streaming jawaban huruf per huruf (gaya ChatGPT) juga didukung OpenClaw dengan kompleksitas multi-channel.

Streaming penting: pengguna mulai lihat jawaban tanpa tunggu agent selesai generate semua. Pengalaman jauh lebih baik.

**Dua lapisan streaming:**

**Block streaming (all channels):**
- Completed blocks (per paragraph, bukan token)
- Configurable chunk size
- Break preference: paragraph/sentence/message end

**Preview streaming (Telegram/Discord/Slack):**
- Update temporary preview saat generating
- Pesan final muncul setelah selesai

> **Pro Tip:** Aktifkan `streaming: "partial"` di Slack/Telegram untuk respons real-time.

---

## Agent Loop: Siklus Eksekusi

Agent Loop itu jantung OpenClaw, berjalan setiap kali agent menerima pesan.

### Siklus Eksekusi

![Siklus eksekusi Agent Loop dari pesan masuk hingga respons dikirim](images/ch02-agent-loop.png)

**Step by step:**

1. **Queueing**: Pesan diatur urutan dan concurrency
2. **Agent Loop**: Ambil pesan, assemble context, compile system prompt, siapkan request
3. **Provider Call**: Kirim ke AI provider (lama karena latency dan inference)
4. **Response Processing**: Parse response, jalankan tool call di sandbox (kalau aktif)
5. **Session Update**: Update context percakapan di session store
6. **Send**: Kirim response ke channel asal

**Hook points**: Interceptor yang memodifikasi pesan sebelum/sesudah diproses.

### Tool Execution di Dalam Loop

Saat merespon, agent perlu menjalankan tool (misal `kubectl get pods`). Tool execution di dalam loop:
- Di **sandboxed environment** kalau aktif
- Hasil disimpan di memory untuk context
- Timeout dan resource limits di-enforce
- Tool gagal: error report, agent coba cara lain

### Messaging Tools

- **`notify`**: Kirim pesan ke channel (misal alert ke #ops)
- **`reply`**: Reply ke percakapan berjalan
- **`echo`**: Echo text untuk debugging

---

## Workspace Management

Workspace adalah lingkungan kerja agent dengan struktur folder mirip laptop project. Menyimpan semua konfigurasi, context, skills, dan session history.

### Struktur Folder Workspace

![Diagram struktur folder workspace dengan penjelasan fungsi masing-masing file dan folder](images/ch02-workspace-structure.png)

**Peran masing-masing file:**

- **`AGENTS.md`**: Konfigurasi utama agent (tools yang boleh dipakai, model)
- **`SOUL.md`**: Kepribadian dan tone agent (formal/casual, approval style)
- **`TOOLS.md`**: Dokumentasi tools yang tersedia
- **`IDENTITY.md`**: Nama, role, spesialisasi agent
- **`MEMORY.md`**: Fakta dan knowledge yang terkumpul

> **Pro Tip:** Simpan bootstrap files ringkas. File panjang = lebih token = lebih mahal.


---

## Ringkasan Penting

- OpenClaw punya arsitektur **modular dan event-driven** dengan WebSocket untuk komunikasi real-time
- **Gateway Protocol** jadi pintu gerbang semua komunikasi, mengatur session, auth, dan message routing
- **Session Management** membuat agent bisa ingat context percakapan, dengan berbagai mode isolation
- **Agent Runtime** adalah otak operasional yang menggabungkan Pi Agent Core dengan OpenClaw Layer
- **Context Engine** menggabungkan data dari session, memory, workspace files, dan tool results menjadi context yang lengkap
- **System Prompt** dikompilasi secara dinamis dari bootstrap files, tooling info, safety guardrails, dll
- **Multi-agent** support membuat Anda bisa punya beberapa agent spesialis yang berbagi satu gateway
- **Sandboxing** menyediakan isolation yang krusial untuk keamanan eksekusi command
- **Streaming** membuat respons pengalaman lebih baik, terutama di channel seperti Slack dan Telegram

---

