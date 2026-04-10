# Bab 17: Audit, Logging & Compliance

> *"Kepercayaan itu gak dibangun dari kata-kata manis. Kepercayaan dibangun dari jejak yang bisa diverifikasi. Di dunia AI agent, tanpa audit trail yang solid, kepercayaan kamu cuma wishful thinking."*

---

## Kenapa Audit Itu Hidup-Mati untuk AI Agent

Coba bayangkan kamu ngasih kunci rumah ke asisten. Kamu pengen dia bisa bantuin kamu masuk pas kamu lupa bawa kunci, tapi di saat yang sama kamu juga pengen yakin bahwa dia gak akan ngajak orang asing masuk. Gimana cara kamu bisa yakin? Ya, dengan CCTV. Dengan log akses. Dengan rekaman siapa masuk jam berapa dan ngapain aja di dalam. Tanpa semua itu, kamu gak akan pernah bener-bener tenang naruh kunci di tangan orang lain.

Nah, prinsip yang sama berlaku buat AI agent. Pas engineer manusia bikin perubahan di infrastructure, ada "paper trail" implisit yang terbentuk secara alami. Mereka buka terminal, ketik command di shell history, ngobrol di Slack minta review, push ke git yang ada commit message-nya. Semua jejaknya bisa di-rekonstruksi. Tapi pas AI agent yang ngerjain? Proses keputusannya gak keliatan. "Kenapa" agent milih jalan A bukan B itu tinggal di dalam context window yang bakal ilang begitu session-nya reset.

Ini yang membuat audit bukan sekadar nice-to-have, tapi wajib banget. Kalau kamu di-audit sama compliance officer, atau investigasi incident, dan bilang "kayaknya sih agent yang jalanin command itu, tapi aku gak punya bukti lengkapnya", selesai kepercayaan kamu. **Kalau kamu gak bisa audit apa yang dilakukan agent, kamu gak bisa percaya sama dia.** Dan tanpa kepercayaan, agent kamu gak akan pernah bisa beroperasi di production.

OpenClaw dari awal desain-nya ngerti banget soal ini. Makanya audit infrastructure itu bukan fitur tambahan, tapi udah jadi core dari gateway kerja. Setiap interaksi direkam, setiap command dicatat, setiap approval decision disimpan. Kamu bisa buka file session sebulan kemudian dan baca ulang persis apa yang terjadi.

> **Istilah Kunci: Audit** adalah proses meninjau dan memverifikasi aktivitas sistem untuk memastikan kepatuhan terhadap standar keamanan dan kebijakan. Di konteks AI agent, audit juga mencakup kemampuan untuk merekonstruksi keputusan dan tindakan agent setelah fakta.

---

## Session Recordings: Kotak Hitam untuk AI Agent Kamu

OpenClaw otomatis ngerekam semua interaksi agent dalam file session. Full conversation history: pesan user, jawaban agent, tool calls, hasil eksekusi, sampai system events kayak approval atau error.

Kenapa ini krusial? Kadang masalah baru muncul seminggu setelah perubahan. Kalau sessionnya gak kerekam, kamu harus nebak-nebak. Dengan session recording, kamu tinggal buka file session dari minggu lalu dan baca persis urutan kejadian.

### Penyimpanan & Format Session

Session disimpan dalam format yang sederhana tapi powerful: line-delimited JSON (JSONL). Setiap baris adalah satu event, jadi kamu bisa proses file ini streaming tanpa harus load semuanya ke memory. Format ini juga ramah buat tools seperti `jq`, `grep`, atau pipeline analisis.

- **Location**: `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl`
- **Format**: Line-delimited JSON (JSONL)
- **Contents**: Complete conversation history termasuk tool calls, results, dan system events

```bash
# List semua session
openclaw sessions

# List session yang aktif (120 menit terakhir)
openclaw sessions --active 120

# Lihat session dalam format JSON
openclaw sessions --json

# Cek path storage session
openclaw status --deep
```

Yang keren dari pendekatan file-based ini adalah portability-nya. Mau backup ke S3? Tinggal sync folder. Mau analisa pake Python? Tinggal baca file. Mau archive ke cold storage? Tinggal move. Gak ada vendor lock-in. Semua terbuka dan standar.

> **Eits, Hati-hati!** Jangan hapus file session secara manual! OpenClaw punya retention policy otomatis yang ngurus file-file ini biar gak meledak disk. Kalau kamu `rm -rf` di folder sessions, kamu bisa aja hapus bukti investigasi yang butuhin minggu depan. Biarkan OpenClaw yang ngurus lifecycle.

### Session Lifecycle

Session di OpenClaw di-reuse selama belum expire. Ada beberapa cara session bisa di-reset:

- **Daily reset** (default): Session baru tiap jam 4 pagi. Timing dipilih karena biasanya sepi trafik.
- **Idle reset** (optional): Session baru kalau gak ada aktivitas selama periode tertentu. Set via `session.reset.idleMinutes`.
- **Manual reset**: Ketik `/new` atau `/reset` di chat. `/new <model>` bahkan bisa ganti model.

Kalau kamu configure daily dan idle reset barengan, yang duluan expire yang menang. Ini flexible karena kamu bisa mix strategi sesuai kebutuhan.

> **Contoh Nyata:** Bayangin kamu lagi debugging masalah di production jam 2 pagi. Session sebelumnya udah penuh sama percakapan deployment kemarin dan agent masih nyambung-nyambungin ke konteks lama. Daripada nunggu daily reset di jam 4, ketik `/new` dan langsung punya session fresh. Lima detik, clean slate.

### Session Maintenance: Jangan Sampai Disk Jebol

OpenClaw otomatis ngebatasi storage session biar gak ngabisin disk. Tanpa maintenance, file session bisa menumpuk menjadi masalah operasional. Bayangin tiap hari ada 50 session baru, masing-masing 2 MB, setahun kamu udah nimbun 36 GB data. Nyesek kalau gateway down cuma gara-gara disk penuh.

```json5
{
  session: {
    maintenance: {
      mode: "enforce",        // "warn" | "enforce"
      pruneAfter: "30d",      // Hapus session setelah 30 hari
      maxEntries: 500,        // Simpan maksimal 500 session
    },
  },
}
```

Kombinasikan `pruneAfter` (age-based cleanup) dan `maxEntries` (count-based cleanup). Yang duluan hit, itu yang nentuin file mana yang di-prune. Mode `enforce` langsung hapus, mode `warn` cuma kasih peringatan.

Preview dulu pakai `openclaw sessions cleanup --dry-run`. Kamu bakal liat tepatnya file apa aja yang bakal kena hapus.

> **Pro Tip:** Untuk compliance regulated kayak finance atau healthcare, jangan set `pruneAfter` terlalu pendek. Beberapa regulator minta retention minimal 1 tahun. Sesuaikan policy sama requirement hukum, archive ke cold storage perlu.

---

## Logging Infrastructure: Mata dan Telinga Sistem

OpenClaw punya logging yang komprehensif di semua component-nya. Session recording itu detail percakapan, logging itu lebih kayak "pulse" sistem.

Ibaratnya, session recording kayak rekaman interview, log kayak CCTV koridor. Dua-duanya penting. Tanpa logging yang proper, kamu cuma liat yang di depan hidung doang.

### Gateway Logs

Log dari gateway disimpan lokal di host tempat gateway jalan, dan bisa kamu tail dari jarak jauh pake RPC command. Kamu gak perlu SSH langsung ke server gateway buat debug masalah. Semua bisa diakses via CLI yang udah kamu setup di laptop.

```bash
# Tail gateway log lokal
tail -f ~/.openclaw/agents/<agentId>/logs/gateway.log

# Tail log remote (via RPC)
openclaw logs --follow

# Follow log dengan interval custom
openclaw logs --follow --interval 2000

# Lihat log dalam format JSON
openclaw logs --json --limit 500

# Batasi berdasarkan ukuran file
openclaw logs --max-bytes 1048576

# Lihat log dengan timezone lokal
openclaw logs --local-time
```

Fitur `--follow` via RPC ini sangat berguna untuk troubleshooting. Kamu bisa stream log dari gateway ke terminal, sambil trigger perintah ke agent dan melihat reaksi sistem langsung. Kombinasi yang powerful untuk investigation aktif.

### Format Log yang Terstruktur

Log di OpenClaw itu structured, bukan free-form text. Setiap entry punya field konsisten: timestamp, level, component, message, sessionId, dan metadata.

```json
{
  "timestamp": "2024-01-15T10:30:00.123Z",
  "level": "info",
  "component": "gateway",
  "message": "Session started for slack channel",
  "sessionId": "b0c8c0b3-2c2d-4f8a-9a3c-5a4b3c2d1e0f",
  "metadata": {
    "channel": "slack",
    "channelId": "C1234567890",
    "userId": "U1234567890"
  }
}
```

Format JSON ini membuat integrasi dengan tools seperti Elasticsearch, Datadog, atau Grafana Loki menjadi mudah. Tinggal kirim log ke stack observability kamu.

> **Istilah Kunci: JSONL** adalah JavaScript Object Notation Line-delimited, format JSON di mana setiap baris file adalah object JSON terpisah. Keunggulannya: bisa di-stream, bisa di-append tanpa rewrite file, dan bisa diprocess pake tools line-by-line kayak `grep`, `awk`, atau `jq`.

### Konfigurasi Logging

Log level dan format bisa kamu configure. Di development, pakai `debug` atau `trace`. Di production, `info` atau `warn` biasanya cukup.

```json5
{
  logging: {
    level: "info",           // "trace" | "debug" | "info" | "warn" | "error"
    format: "json",          // "json" | "compact"
    file: {
      enabled: true,
      path: "~/.openclaw/agents/{agentId}/logs/gateway.log",
      maxSize: "100MB",
      maxFiles: 5
    },
    openTelemetry: {
      enabled: true,
      endpoint: "http://localhost:4317"
    }
  }
}
```

OpenClaw support ngirim log ke OpenTelemetry collector, jadi kamu bisa integrate ke distributed tracing system. Ada pengaturan rotation: `maxSize` buat ukuran maksimal file log, `maxFiles` buat jumlah file log lama.

> **Eits, Hati-hati!** Level `trace` sangat verbose dan bisa menghasilkan gigabytes data dalam waktu singkat, apalagi kalau gateway busy. Jangan nyalain trace level di production untuk jangka panjang. Pakai cuma sebentar buat investigasi spesifik, terus turunin lagi ke `info`. Pernah ada case tim lupa matiin trace dan disk penuh dalam 3 jam.


---

## System Events: Notifikasi Real-Time dari Sistem

OpenClaw nge-emit system events buat operasi kunci. Ini penting banget buat awareness real-time. Log itu buat historical analysis, system events itu kayak stream notifikasi.

Kenapa penting? Karena ada banyak hal di sistem AI agent yang gak cukup dilihat dari log. Misalnya, pas ada approval request butuh action, kamu perlu tau sekarang juga. Atau pas ada exec command running lama, kamu pengen auto-kill atau dapet alert.

### 1. Session Events

Event yang berkaitan lifecycle session. Ini membantu ngelacak pola penggunaan dan deteksi anomali kayak session tiba-tiba banyak banget dibuat dalam waktu singkat (potential indicator of abuse).

- `Session started`: Session percakapan baru dimulai
- `Session reset`: Session di-clear (manual atau otomatis)
- `Session compacted`: Session di-summarize buat percakapan yang udah panjang

### 2. Execution Events

Event yang berkaitan eksekusi command. Ini paling critical buat audit karena setiap command yang jalan di sistem harus trackable.

- `Exec running`: Command yang running-nya lama (melebihi threshold)
- `Exec finished`: Command udah selesai
- `Exec denied`: Command di-block sama approval policy

### 3. Approval Events

Approval request membuat system event yang muncul di session. Ini membuat approval visible di conversation flow, jadi reviewer bisa lihat konteks penuh pas bikin keputusan approve atau deny.

```
🔒 Approval Required

Action: `kubectl restart deployment/payment-service -n production`
Command: kubectl restart deployment/payment-service -n production
Host: gateway-prod-01
Security: allowlist

[✅ Approve] [❌ Deny] [ℹ️ Details]
```

Format ini membuat approver tidak perlu melihat log untuk memahami konteks. Semua info penting - command, host, security level - ada di satu tempat dengan action button jelas.

### 4. Contoh Event

Event ini di-structure sebagai JSON konsisten, jadi kamu gampang banget buat automation. Misalnya bisa set up webhook yang kirim Slack notification setiap ada `Exec denied`, atau trigger PagerDuty alert setiap ada command `Exec running` lebih dari 5 menit.

```json
// Event exec finished
{
  "timestamp": "2024-01-15T10:30:15Z",
  "event": "Exec finished",
  "command": "kubectl get pods -n production",
  "exitCode": 0,
  "durationMs": 13247,
  "runId": "b0c8c0b3-2c2d-4f8a-9a3c-5a4b3c2d1e0f"
}

// Event exec denied
{
  "timestamp": "2024-01-15T10:30:15Z",
  "event": "Exec denied",
  "command": "rm -rf /",
  "host": "gateway-prod-01",
  "reason": "Blocked by approval policy"
}
```

> **Contoh Nyata:** AI agent nyoba jalanin `rm -rf /var/log/*` karena salah interpretasi instruksi user. Gateway langsung nolak karena gak ada di allowlist, dan ngirim `Exec denied` event. Tim security dapet alert real-time, langsung investigasi.

---

## Transcript Hygiene: Biar AI Provider Gak Rewel

OpenClaw melakukan sanitasi transcript spesifik per provider sebelum membangun context. Setiap AI provider punya aturan format yang beda: Gemini rewel soal tool call ID, Anthropic pengen tool result pairing, OpenAI lebih forgiving tapi rewel di image format, Mistral punya aturan unik.

Tanpa transcript hygiene, kamu bakal sering dapet error aneh dari provider. Misalnya request ditolak dengan "Invalid tool call ID format", padahal di session sebelumnya gak ada masalah. Ini biasanya karena transcript dari provider lain di-load ke session yang sekarang pake provider berbeda. OpenClaw otomatis handle ini.

### Behavior Spesifik per Provider

Berikut behavior sanitasi per provider. Kamu gak perlu hafalin semua, tapi good to know kalau kamu suka nemuin error mencurigakan berkaitan format.

**Google (Gemini)**
- Tool call ID sanitization: strict alphanumeric
- Turn validation (Gemini-style alternation)
- Google turn ordering fixup

**Anthropic**
- Tool result pairing repair
- Turn validation (merge consecutive user turns)

**OpenAI**
- Image sanitization saja
- Tidak ada tool call ID sanitization

**Mistral**
- Tool call ID sanitization: strict9 (alphanumeric length 9)

Setiap provider punya quirk yang beda, dan OpenClaw punya logic terpisah buat handle masing-masing. Ini "plumbing work" yang kamu gak kelihatan tapi membuat failover antar provider jadi mulus. Kamu bisa mulai session pakai Claude, terus switch ke GPT-4 di tengah, dan OpenClaw bakal bersihin transcript yang lama biar compatible.

> **Istilah Kunci: Transcript Sanitization** adalah proses membersihkan dan memformat transkripsi percakapan untuk memenuhi persyaratan AI provider. Tujuannya: biar pesan yang udah ada di session bisa diteruskan ke provider manapun tanpa error format.

### Sanitasi Image

Image otomatis di-downscale buat ngehindarin penolakan dari provider. Kebanyakan AI provider punya batasan ukuran image, dan kalau image upload ngelebihin batas, request langsung di-reject tanpa penjelasan jelas.

```json
{
  "content": [
    {
      "type": "image",
      "source": {
        "type": "base64",
        "media_type": "image/jpeg",
        "data": "downscaled base64 data..."
      }
    }
  ]
}
```

Max image size bisa di-configure via `agents.defaults.imageMaxDimensionPx` (default: 1200). Resolusi ini balance antara ukuran file manageable dan detail visual cukup buat model AI "melihat". Kamu bisa turunin buat hemat bandwidth, atau naikin kalau model support resolusi lebih tinggi.

> **Eits, Hati-hati!** Image sanitization bisa ngurangin kualitas gambar besar. Kalau upload screenshot error atau diagram detail, verifikasi hasil downscale masih kebaca. Kalau perlu, crop dulu ke area relevan.


---

## Session Integrity: Ngebedakan Perintah Langsung vs Routed

File session menyertakan provenance tracking buat inter-session message. Ini fitur subtle tapi penting banget buat keamanan. Bayangin agent A di session X nerima instruksi dari user, terus ngeroute sebagian tugas ke agent B di session Y. Tanpa provenance tracking, agent B bakal melihat pesan dari agent A sebagai "user message" dan menanganinya dengan trust level sama kayak dari user beneran.

Itu masalah keamanan. Karena instruksi dari agent lain seharusnya gak punya trust level sama kayak dari human operator. Bisa aja agent A lagi di-manipulate sama prompt injection, terus ngirim instruksi berbahaya ke agent B yang nerima tanpa curiga.

```json
{
  "role": "user",
  "content": "Can you check the system status?",
  "provenance": {
    "kind": "inter_session"
  }
}
```

Pas context rebuild, OpenClaw nambahin marker `[Inter-session message]` di depan pesan. Jadi model AI bisa bedain mana instruksi dari user langsung, dan mana dari routing internal. Agent bisa apply policy lebih strict ke pesan routed.

Fitur ini makin penting kalau kamu punya setup multi-agent complex, dimana agent saling route task. Tanpa provenance tracking, kamu rawan ke "confused deputy problem" di security, dimana agent trusted jalanin command jahat karena ngiranya itu instruksi sah.

### Schema Session Store

File session menggunakan format structured. Setiap event punya timestamp presisi, tipe jelas, dan metadata relevan. Ini membuat session file jadi bukti digital lengkap dan bisa diverifikasi integritasnya.

```json
{
  "sessionId": "b0c8c0b3-2c2d-4f8a-9a3c-5a4b3c2d1e0f",
  "createdAt": "2024-01-15T10:30:00Z",
  "events": [
    {
      "type": "user_message",
      "timestamp": "2024-01-15T10:30:01Z",
      "content": "Deploy the new version"
    },
    {
      "type": "tool_call",
      "timestamp": "2024-01-15T10:30:02Z",
      "tool": "exec",
      "command": "kubectl apply -f deployment.yaml"
    },
    {
      "type": "system_event",
      "timestamp": "2024-01-15T10:30:15Z",
      "event": "Exec finished",
      "metadata": {
        "exitCode": 0,
        "durationMs": 13247
      }
    }
  ]
}
```

Urutan kronologis penting banget buat forensic analysis. Kamu bisa lihat tepatnya "user bilang apa, jam berapa, agent ngerespon apa, tool apa yang dipanggil, berapa lama jalannya, dan hasilnya apa".

---

## Exec Audit Trails: CCTV untuk Command Execution

OpenClaw menyediakan audit trail lengkap untuk eksekusi command lewat approval workflow. Kalau session recording itu rekaman percakapan, exec audit trail lebih spesifik ke "apa yang sebenarnya dieksekusi di level OS".

Kenapa butuh layer audit terpisah buat exec? Karena risikonya paling tinggi. Command salah bisa membuat layanan down, data hilang, atau security breach. Regulator minta bukti spesifik tentang command apa yang dijalankan, siapa yang mengizinkan, dan hasilnya. Tanpa exec audit trail solid, kamu bakal susah lulus compliance.

### Konfigurasi Approval

Approval diatur di `~/.openclaw/exec-approvals.json`. File ini nentuin policy siapa boleh jalanin apa, dan dalam kondisi butuh approval.

```json
{
  "version": 1,
  "socket": {
    "path": "~/.openclaw/exec-approvals.sock",
    "token": "base64url-token"
  },
  "defaults": {
    "security": "deny",
    "ask": "on-miss",
    "askFallback": "deny",
    "autoAllowSkills": false
  },
  "agents": {
    "main": {
      "security": "allowlist",
      "ask": "on-miss",
      "askFallback": "deny",
      "autoAllowSkills": true,
      "allowlist": [
        {
          "id": "B0C8C0B3-2C2D-4F8A-9A3C-5A4B3C2D1E0F",
          "pattern": "~/Projects/**/bin/rg",
          "lastUsedAt": 1737150000000,
          "lastUsedCommand": "rg -n TODO",
          "lastResolvedPath": "/Users/user/Projects/.../bin/rg"
        }
      ]
    }
  }
}
```

Perhatikan setiap entry di allowlist punya metadata audit: `lastUsedAt`, `lastUsedCommand`, `lastResolvedPath`. Ini bukti historical buat investigation. Kalau ada yang nanya "kapan terakhir kali binary ini dipake?", kamu langsung punya jawabannya.

Struktur dengan `defaults` dan per-agent override juga fleksibel. Kamu bisa set default global yang konservatif, terus loosen restriction untuk agent tertentu. Prinsip least-privilege yang enforced.

> **Istilah Kunci: Allowlist** adalah daftar command atau aplikasi yang diizinkan untuk dijalankan. Segala sesuatu yang gak ada di allowlist akan otomatis ditolak. Pendekatan ini kebalikan dari blocklist dan lebih aman karena mengasumsikan "default deny".

### Approval Workflow

Pas approval dibutuhkan, exec tool langsung return dengan approval ID. Ini desain async smart karena agent gak perlu block nunggu user respon.

```json
{
  "status": "approval-pending",
  "approvalId": "b0c8c0b3-2c2d-4f8a-9a3c-5a4b3c2d1e0f"
}
```

Begitu disetujui (atau ditolak / timeout), Gateway emit system event yang dicatat di session. Jadi approval flow ini sepenuhnya auditable: kamu bisa lihat kapan request dibuat, kapan respon datang, siapa yang approve, dan hasil akhirnya.

### Safe Bins: Balance Antara Keamanan dan Kenyamanan

Safe bins adalah executable stdin-only yang bisa dijalankan tanpa entry allowlist eksplisit. Ini konsep penting karena kalau setiap command kecil harus di-approve satu-satu, produktivitas bakal anjlok. Command kayak `cut`, `head`, `tail`, `wc` itu terlalu common dan aman.

```json5
{
  tools: {
    exec: {
      safeBins: ["cut", "uniq", "head", "tail", "tr", "wc"],
      safeBinTrustedDirs: ["/opt/homebrew/bin", "/usr/local/bin"],
      safeBinProfiles: {
        jq: {
          minPositional: 0,
          maxPositional: 0,
          allowedValueFlags: ["-n", "--limit"],
          deniedFlags: ["-f", "--file", "-c", "--command"]
        }
      }
    }
  }
}
```

Yang menarik adalah `safeBinProfiles`. Ini fitur granular dimana kamu bisa allow binary tertentu, tapi dengan restriction ke flag atau argument. Contoh di atas, `jq` diizinkan tapi gak boleh pake flag `-f` atau `-c` karena bisa dipake buat arbitrary code execution.

> **Contoh Nyata:** Perintah dasar seperti `cut`, `head`, dan `tail` dibolehin tanpa approval karena dianggap aman. Ini seperti ngebiarin anak pake sendok makan tanpa pengawasan ketat. Tapi `jq`, meskipun juga utility umum, di-profile dengan ketat karena punya fitur yang bisa jadi "lockpick" kalau disalahgunakan. Distingsi ini yang membuat safe bins system jadi praktis tanpa ngorbanin keamanan.

### CLI untuk Approval Management

Semua konfigurasi approval bisa di-manage lewat CLI. Ini penting buat automation dan infrastructure-as-code, karena kamu bisa script pengaturan approval dan version control-nya.

```bash
# Dapatkan konfigurasi approval saat ini
openclaw approvals get

# Dapatkan konfigurasi approval untuk node tertentu
openclaw approvals get --node node-01

# Set konfigurasi approval
openclaw approvals set --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "allowlist",
    ask: "on-miss",
    askFallback: "deny"
  }
}
EOF

# Kelola entry allowlist
openclaw approvals allowlist add "~/Projects/**/bin/rg"
openclaw approvals allowlist remove "~/Projects/**/bin/rg"
openclaw approvals allowlist list
```

Dengan CLI ini, kamu bisa membuat pipeline yang otomatis deploy approval config sebagai bagian dari GitOps workflow. Config lama di-backup, config baru di-apply, dan kalau ada masalah bisa instant rollback. Jauh lebih safe daripada editing file JSON manual.


---

## Pelacakan Penggunaan: Dashboard untuk Operasional

OpenClaw melacak eksekusi command dan penggunaan sistem untuk audit dan operasional. Layer reporting ini memberikan insight agregat dari session dan log, fokus di agregat (per hari/minggu) bukan granular events.

### Tracking Exec Command

Semua command dieksekusi dengan metadata lengkap. Struktur data konsisten dengan system events, gampang di-korelasikan.

```json
{
  "timestamp": "2024-01-15T10:30:15Z",
  "event": "Exec finished",
  "command": "npm run build",
  "exitCode": 0,
  "durationMs": 45234,
  "runId": "b0c8c0b3-2c2d-4f8a-9a3c-5a4b3c2d1e0f",
  "metadata": {
    "host": "gateway",
    "security": "allowlist",
    "ask": "off"
  }
}
```

Dari data ini bisa dibuat berbagai metric: frequency distribution, latency, error rate, dan pola penggunaan. Semua ini jadi input buat capacity planning.

### Statistik Approval

```bash
# Review approval terbaru
openclaw approvals list --recent 50

# Cek statistik approval
openclaw approvals stats --period week

# Lihat history approval berdasarkan node
openclaw approvals get --node node-01 --history
```

Gunakan `approvals stats` buat weekly review. Lihat berapa approval masuk, approve/deny, dan siapa yang sering request. Penting buat deteksi anomali.

> **Eits, Hati-hati!** Periksa statistics seminggu sekali. Pola an bisa jadi sinyal aktivitas mencurigakan.

---

## Compliance Features: Ngelegalin Setup Kamu

Compliance di OpenClaw bukan cuma checklist, tapi tools dan config buat memenuhi requirement regulasi. Baik SOC 2, ISO 27001, HIPAA, PCI-DSS, atau UU PDP, ada common patterns yang must address.

### 1. Security Audit

Verifikasi konfigurasi keamanan dengan audit built-in. Jalankan command ini tiap major config change atau minimal bulanan.

```bash
# Jalankan security audit
openclaw security audit

# Cek area spesifik
openclaw security audit --area approvals
openclaw security audit --area secrets
openclaw security audit --area sandbox
```

Audit nge-scan config dan ngecek against best practices. Dia nge-flag approval policy kelewat loose, secrets ke-expose, atau default setting rawan. Output report dengan rekomendasi.

### 2. Session Isolation

Pastiin session isolation proper buat lingkungan multi-user. Ini tricky banget di setup dimana satu agent dipake banyak orang.

```json5
{
  session: {
    dmScope: "per-channel-peer",  // Isolasi berdasarkan channel + sender
    identityLinks: [              // Hubungkan identitas lintas channel
      {
        "pattern": "alice@company.*",
        "linkTo": "alice-primary"
      }
    ]
  }
}
```

Opsi DM isolation:
- `main` (default): Semua DM berbagi satu session. Simple tapi gak isolated.
- `per-peer`: Isolasi berdasarkan sender. Alice di Slack dan Alice di Telegram share session, beda dari Bob.
- `per-channel-peer`: Isolasi berdasarkan channel + sender (recommended). Alice di Slack beda session dari Alice di Telegram.
- `per-account-channel-peer`: Isolasi berdasarkan account + channel + sender. Paling strict, cocok multi-tenant.

Pilih mode sesuai threat model. Default `main` cocok personal use atau tim kecil. Untuk company besar, `per-channel-peer` biasanya sweet spot.

### 3. Audit Workflow Approval

Lacak keputusan approval dan pola-polanya. Ini sering diminta auditor pas review compliance.

```bash
# Review approval terbaru
openclaw approvals list --recent 50

# Cek statistik approval
openclaw approvals stats --period week

# Lihat history approval
openclaw approvals get --node node-01
```

> **Contoh Nyata:** Di fintech, hanya dev lead yang bisa approve deployment ke production. Dengan approval workflow audit, tim compliance bisa generate report bulanan nunjukkan siapa approve apa dan kapan. Pas audit eksternal, mereka export report sebagai bukti policy enforced.

---

## Preservasi Evidence

### File Session sebagai Bukti

Session recording menyediakan audit trail lengkap yang siap jadi bukti digital. Session file menjadi evidence yang powerful.

- **Full conversation context**: Semua pesan dari awal sampai akhir
- **Tool calls dan results**: Apa yang dipanggil agent dan hasilnya
- **Approval decisions**: Siapa approve apa dan kapan
- **System events**: Semua kejadian di level sistem

Kombinasi ini membuat session file menjadi bukti yang sulit di-dispute. Kalau ada sengketa "siapa yang ngejalanin command itu", file session punya jawaban definitive.

### Evidence Chain

File session menjaga integritas kronologis. Setiap event di-stamp dengan timestamp presisi sampai millisecond, dan urutannya immutable (append-only). Ini properti penting banget buat evidence di lingkungan hukum.

```json
{
  "sessionId": "b0c8c0b3-2c2d-4f8a-9a3c-5a4b3c2d1e0f",
  "createdAt": "2024-01-15T10:30:00Z",
  "events": [
    {
      "type": "user_message",
      "timestamp": "2024-01-15T10:30:01Z",
      "content": "Deploy the new version"
    },
    {
      "type": "tool_call",
      "timestamp": "2024-01-15T10:30:02Z",
      "tool": "exec",
      "command": "kubectl apply -f deployment.yaml"
    },
    {
      "type": "system_event",
      "timestamp": "2024-01-15T10:30:15Z",
      "event": "Exec finished",
      "metadata": {
        "exitCode": 0,
        "durationMs": 13247
      }
    }
  ]
}
```

Buat meningkatkan integritas, banyak organisasi nge-sync file session ke write-once storage kayak S3 Object Lock.

### Session Compaction

Untuk percakapan panjang, OpenClaw bisa nge-summarize session sambil simpan informasi kunci.

```bash
# Auto-compact session panjang
/session compact

# Compaction manual
openclaw sessions compact --prune-after 30d
```

Yang tetap dipertahankan: approval decisions, hasil command penting, dan key decision point. Yang di-summarize: chit-chat.

> **Eits, Hati-hati!** Session compaction ngehasilin ringkasan. Informasi detail bisa hilang meskipun approval decisions tetap tersimpan. Untuk forensic, pertimbangkan backup file session original dulu.


---

## Checklist: Audit & Compliance

- [ ] Session isolation udah di-configure buat akses multi-user
- [ ] Retention policy session udah diset (minimum 30 hari, atau sesuai requirement regulator)
- [ ] Session maintenance udah di-configure (prune after 30d, max 500 entries)
- [ ] Security audit udah dijalankan dan issue-nya udah di-resolve
- [ ] Gateway log udah dimonitor buat error
- [ ] Approval workflow udah di-review secara rutin
- [ ] File session disimpan buat investigasi incident
- [ ] Log rotation udah di-configure
- [ ] Requirement compliance udah didokumentasiin
- [ ] Prosedur incident response termasuk review session

---

