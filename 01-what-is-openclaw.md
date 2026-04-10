# Bab 1: Apa Sih OpenClaw Itu?

> *"Bayangin kamu punya asisten yang gak pernah tidur, gak pernah minta cuti, dan bisa ngobrol lewat Slack, WhatsApp, Telegram, semuanya sekaligus. Bukan asisten manusia, tapi AI yang beneran bisa ngapa-ngapain infrastruktur kamu."*

---

## Cerita dari Sebuah Project Open-Source

Semua bermula di November 2025. Peter Steinberger, developer ternama iOS, meluncurkan project bernama **Clawdbot**. Karena masalah merek dagang dengan Anthropic, nama diubah jadi **Moltbot** Januari 2026, sebelum akhirnya menetap di **OpenClaw**.

Perjalanan rename ini kayak sinetron, tapi di balik drama-nya, OpenClaw adalah project open-source MIT yang berkembang pesat. Banyak developer dan tim DevOps langsung tertarik karena akhirnya ada framework AI agent yang bisa di-host sendiri, transparan, dan gak terkunci ke satu vendor.

Kenapa penting? Sebelum OpenClaw, kebanyakan AI assistant proprietary dan terkunci satu platform. Pakai AI buat infra? Harus bayar subscription mahal, terima fitur vendor apa adanya. OpenClaw nabrak dinding itu dengan kasih kamu full kontrol: tentuin model AI, channel, dan skill. Semua bisa di-customize.


---

## Masalah Apa Sih yang Mau Diselesaikan?

Nah, biar kamu paham kenapa OpenClaw itu *penting*, aku mau cerita dulu tentang realitas tim infrastruktur modern.

Kamu tahu kan gimana rasanya jadi DevOps engineer sekarang? Rata-rata tim SRE harus ngurus **10x lebih banyak service** dibanding lima tahun yang lalu, tapi ukuran timnya? Tetap. Malah mungkin makin kecil karena efisiensi. Setiap hari kamu bolak-balik ngurus Terraform, Kubernetes, pipeline CI/CD, dashboard Grafana, alert PagerDuty, dan security scanner. Seringnya sih secara bersamaan.

AI coding assistant seperti GitHub Copilot atau Cursor membantu nulis kode, tapi mereka berhenti di batas editor. Mereka tidak bisa:

- **Memantau sistem production jam 3 pagi** saat kamu tidur, CPU server nembus 95%
- **Menganalisis log CloudWatch yang membludak** - harus buka dashboard, filter, scroll manual
- **Menjalankan runbook pas PagerDuty bunyi** - harus baca dan eksekusi manual tiap step-nya

Ini bukan teori. Ini realitas yang dihadapi hampir semua tim SRE dan DevOps di luar sana. Kompleksitas sistem tumbuh secara eksponensial, tapi kapasitas manusia linear. Di titik inilah AI agent yang bisa beroperasi secara otonom jadi kebutuhan, bukan cuma *bonus*.

**OpenClaw hadir untuk menutupi celah ini.** Dia bukan coding assistant. Dia adalah **AI agent yang persisten dan otonom**, yang beroperasi di seluruh stack infrastruktur kamu, ngobrol lewat channel yang kamu udah pakai sehari-hari, dan bisa diajari skill spesifik sesuai environment kamu.


> **Coba Bayangin...**
>
> Kamu punya asisten virtual yang gak pernah tidur, selalu siap 24/7, dan bisa ngoperasin seluruh infrastruktur kamu sendirian. Dia bisa jawab pertanyaan kompleks, eksekusi tugas, dan bahkan *belajar* dari pola sistem kamu seiring waktu. Itulah intinya: sebuah "cyber engineer digital" yang kerjanya gak perlu disuruh-suruh.

---

## Apa Saja yang Bisa Dilakukan OpenClaw?

Sekarang kita ke bagian teknis. Tenang, aku bakal jelasin pelan-pelan biar kamu benar-benar paham.

### 1. Session yang Persisten (Bukan Sekadar Chat Biasa)

Pernah pakai ChatGPT atau Claude? Setiap kali buat percakapan baru, mereka mulai dari nol. Gak ingat apa yang kamu omongin sebelumnya. Nah, OpenClaw beda.

OpenClaw punya **session persisten**. Artinya AI agent kamu *ingat* konteks dari percakapan sebelumnya. Dia belajar dari pola infrastruktur kamu, membangun knowledge base seiring waktu, dan gak "amnesia" setiap kali kamu chat.

Kenapa ini beda banget? Karena di dunia DevOps, konteks itu *segalanya*. Kalau agent kamu lupa kalau kemarin ada deployment yang gagal di service X, dia gak bisa ngasih jawaban yang akurat pas kamu tanya "kenapa latency naik hari ini?" Dengan session persisten, agent kamu bisa menghubungkan titik-titik itu. Dia ingat kalau kemarin ada deploy yang bermasalah, dan bisa nyimpulin bahwa latency naik kemungkinan besar berkaitan dengan deploy tersebut.

Ini juga berarti semakin sering kamu pakai, semakin "pintar" agent kamu di konteks kamu. Dia mulai paham sama pola-pola yang ada di infrastruktur kamu: service mana yang sering bermasalah, deployment jam berapa yang paling aman, cluster mana yang resource-nya mepet. Semua ini terbangun secara natural lewat session yang persisten.

```bash
# Instalasi OpenClaw
npm install -g openclaw

# Jalankan wizard onboarding
openclaw onboard --install-daemon

# Mulai gateway
openclaw gateway run --port 18789 --bind loopback

# Kirim pesan ke agent
openclaw agent --message "gimana status cluster production?" --agent infra-agent
```

> **Pro Tip:** Cukup jalankan `openclaw onboard --install-daemon` dan ikuti wizard-nya untuk mulai cepat.


### 2. Multi-Channel: Ketemu Kamu di Mana Aja

Ini salah satu fitur yang bikin OpenClaw beda banget dari tools AI lainnya. Dia terhubung ke **14+ platform pesan** secara langsung. Jadi kamu gak perlu buka aplikasi khusus buat ngobrol sama AI agent kamu, cukup chat dari tempat yang udah kamu pakai sehari-hari.

Kenapa ini penting? Karena gesekan itu musuh terbesar adopsi. Kalau kamu harus buka aplikasi baru, login, setup, baru bisa chat sama AI, kemungkinan besar kamu gak bakal konsisten pakai. Tapi kalau AI agent kamu ada di Slack yang udah kamu buka 10 jam sehari? Atau di WhatsApp yang selalu aktif di HP kamu? Tinggal chat, langsung respon. Tanpa gesekan.

- **Slack** — Alert infra tim dan perintah manual sehari-hari
- **Telegram** — Asisten on-call pribadi yang private
- **Discord** — Support komunitas developer
- **Google Chat** — Integrasi di environment Google Workspace
- **Microsoft Teams** — Integrasi enterprise
- **WhatsApp** — Incident response dari HP, gak perlu buka laptop
- **Signal** — Komunikasi yang end-to-end encrypted
- **iMessage** — Buat kamu yang di ekosistem Apple
- **Matrix** — Deployment self-hosted, privacy-first

Yang seru, kamu bisa koneksi semuanya sekaligus ke satu gateway. Jadi satu agent bisa diakses via Slack sama tim infra, via Telegram sama on-call engineer, dan via WhatsApp sama CTO. Semuanya session sama, konteks sama, skill sama.

> **Coba Bayangin...**
>
> Kamu lagi liburan di Bali, terus dapat notifikasi WhatsApp dari monitoring: *"CPU server production nembus 85%!"*. Daripada panik buka laptop dan SSH ke server, kamu cukup balas di WhatsApp: *"Cek history CPU 1 jam terakhir, kalau konsisten tinggi jalankan scale-up."* Dan OpenClaw langsung eksekusi. Santai, kan?


### 3. Skills System: "Mengajarkan" Kemampuan Baru

OpenClaw pakai sistem yang namanya **AgentSkills**. Skill itu cara kamu "mengajari" AI agent kamu cara pakai tools baru. Setiap skill adalah sekumpulan instruksi yang kasih tahu agent gimana caranya menggunakan tool tertentu.

Bayangin kamu punya karyawan baru yang udah jago coding tapi belum tahu cara pakai tools internal di company kamu. Kamu gak perlu ngajarin dari nol, cukup kasih manual dan dia bisa langsung kerja. Nah, skill itu fungsinya kayak manual tersebut. Kamu kasih tahu agent "ini cara pakai Docker di environment kita", "ini cara eksekusi runbook yang kita punya", dan seterusnya.

Skill bisa datang dari mana saja:

- **Bundled skills**, udah termasuk pas install. Jadi siap pakai agent kamu udah bisa jalanin command terminal, baca file, dll.
- **Workspace skills**, kamu bikin sendiri di project kamu. Ini yang paling powerful karena bisa disesuaikan persis sama kebutuhan project.
- **Personal skills**, skill yang kamu share di `~/.openclaw/skills`. Bisa dipakai di semua workspace kamu.
- **Agent skills**, spesifik per agent di `~/.agents/skills`. Buat kalau kamu punya beberapa agent yang masing-masing butuh skill beda.

Contoh skill yang udah tersedia:

- **Terminal** untuk eksekusi command dengan sandboxing aman
- **File system** untuk navigasi dan manajemen file
- **Git** untuk operasi version control
- **Docker** untuk manajemen container
- **Kubernetes** untuk operasi kubectl dan manajemen cluster
- **Cloud** untuk plugin AWS, GCP, Azure
- **Monitoring** untuk integrasi Prometheus, Grafana
- **Security** untuk scanner dan compliance tools

Yang bikin skills system ini powerful banget adalah kemampuannya buat di-compose. Kamu bisa bikin skill "Deploy to Production" yang di dalamnya memanggil skill Kubernetes untuk rolling update, skill Monitoring untuk cek health check, dan skill Slack untuk ngirim notifikasi ke channel tim. Semua itu jadi satu workflow yang bisa di-trigger dengan satu perintah aja.

> **Pro Tip:** Setiap organisasi punya stack teknologi dan prosedur yang beda. Nah, skills system ini yang bikin OpenClaw bisa disesuaikan *persis* sama kebutuhan kamu. Kamu bisa "mengajari" agent kamu cara kerja di lingkungan unik kamu, mulai dari command khusus sampe prosedur internal yang cuma tim kamu yang tahu.


### 4. Bebas Pilih Model AI

OpenClaw mendukung **35+ provider AI**. Kamu bebas milih model mana yang paling cocok buat kebutuhan kamu. Ini penting banget karena gak semua model itu sama. Ada yang jago reasoning, ada yang jago coding, ada yang murah tapi cukup buat tugas sederhana, ada yang mahal tapi akurat banget buat analisis kompleks.

```json5
{
  providers: [
    {
      name: "anthropic",
      model: "anthropic/claude-opus-4-6",
      apiKey: { source: "env", provider: "default", id: "ANTHROPIC_API_KEY" }
    },
    {
      name: "openai",
      model: "openai/gpt-4o",
      apiKey: { source: "env", provider: "default", id: "OPENAI_API_KEY" }
    },
    {
      name: "google",
      model: "google/gemini-pro",
      apiKey: { source: "env", provider: "default", id: "GOOGLE_API_KEY" }
    }
  ]
}
```

Strategi yang sering dipakai di production:

- **Claude** untuk tugas reasoning kompleks (planning infra, root cause analysis, review arsitektur)
- **GPT-4o** untuk generasi dan review kode, plus integrasi yang lebih murah
- **Gemini** untuk beban kerja yang beragam dan context window yang besar
- **Model lokal** (via Ollama) untuk environment air-gapped atau yang sensitif privasi, misalnya data perbankan yang gak boleh keluar dari network
- **Multiple provider** sekaligus untuk optimasi biaya. Tugas berat pakai model premium, tugas ringan pakai model murah.

Yang juga keren, OpenClaw support **failover otomatis** antar provider. Jadi kalau Anthropic lagi down atau rate-limited, agent bisa otomatis switch ke OpenAI atau provider lain yang kamu configure. Gak perlu manual intervention. Untuk tim yang jalan 24/7, ini fitur yang *kritikal*.


### 5. Plugin Ecosystem

Plugin membuat OpenClaw makin bisa diperluas. Kalau skill itu buat "mengajari" agent cara pakai tool, plugin itu buat nambah *kemampuan* baru ke OpenClaw itu sendiri. Misalnya nambah channel baru, nambah provider baru, atau nambah fitur yang belum ada di core.

Contohnya ada plugin `openclaw-claude-code` yang memungkinkan kamu pakai Claude Code CLI sebagai engine programming yang bisa dijalankan tanpa UI, langsung dari dalam OpenClaw:

```json5
// Contoh konfigurasi plugin
plugins: {
  "openclaw-claude-code": {
    workspace: "/opt/infrastructure",
    allowed_tools: ["Read", "Edit", "Bash", "Grep", "Glob"],
    denied_tools: ["Write"]  // Mencegah pembuatan file baru
  }
}
```

Dengan plugin ini, agent OpenClaw kamu bisa ngoding, baca file, edit config, dan jalanin command, semuanya via chat di Slack atau WhatsApp. Bayangin kamu chat di Slack: "tolong bikin script buat backup database production dan taruh di S3", dan agent kamu beneran bikin scriptnya, test di sandbox, dan kasih hasilnya ke kamu buat di-review.

> **Eits, Hati-hati!**
>
> Pas ngatur plugin, jangan asal kasih akses penuh ya. Ngasih akses ke tool kayak `Write` tanpa batasan itu risiko keamanan. Selalu mulai dari permission minimal, tambah pelan-pelan sesuai kebutuhan. Prinsipnya: *least privilege*, kasih akses secukupnya aja. Plugin yang kebanyakan akses itu kayak kasih kunci gudang ke orang yang cuma butuh ambil satu barang.

---

## OpenClaw vs. yang Lain: Apa Bedanya?

Biar lebih jelas posisinya di ecosystem, ini perbandingan lengkapnya:

- **Tipe**: OpenClaw vs Claude Code vs Cursor vs Aider
  - OpenClaw: Framework agent otonom
  - Claude Code: Asisten coding di terminal
  - Cursor: Asisten di IDE
  - Aider: Asisten coding CLI
- **Open Source**: OpenClaw (Ya, MIT) vs Claude Code (Tidak, milik Anthropic) vs Cursor (Tidak) vs Aider (Ya)
- **Session Persisten**: OpenClaw (Ya) vs Claude Code (Per percakapan) vs Cursor (Per sesi) vs Aider (Per sesi)
- **Multi-Channel**: OpenClaw (14+ platform) vs Claude Code (Terminal saja) vs Cursor (VS Code saja) vs Aider (Terminal saja)
- **Dukungan Model**: OpenClaw (35+ provider) vs Claude Code (Claude saja) vs Cursor (Beberapa) vs Aider (Beberapa)
- **Fokus Infra**: OpenClaw (Use case utama) vs Claude Code (Sekunder) vs Cursor (Minimal) vs Aider (Minimal)
- **Self-Hosted**: OpenClaw (Ya) vs Claude Code (Tidak) vs Cursor (Tidak) vs Aider (Ya)
- **Operasi 24/7**: OpenClaw (Ya) vs Claude Code (Tidak) vs Cursor (Tidak) vs Aider (Tidak)
- **Production Access**: OpenClaw (Bisa di-configure) vs Claude Code (Sandbox saja) vs Cursor (Tidak) vs Aider (Tidak)

Poin pentingnya:

- **Claude Code** juaranya di sesi coding interaktif, pas buat developer yang pengen ngoding bareng AI di terminal
- **OpenClaw** juaranya di operasi infra otonom yang jalan 24/7, pas buat tim yang pengen AI yang *beneran* ngurus infra
- Mereka **komplementer**, bukan saingan. Banyak tim pakai keduanya bersamaan dan dapet benefit dari keduanya.

> **Contoh Nyata:**
>
> Fintech di Jakarta pakai Claude Code buat develop fitur baru (developer-nya ngoding bareng AI di terminal). Sementara itu, mereka pakai OpenClaw buat operasi production: monitoring cluster, jalanin backup otomatis, nangkep incident kecil, dan bahkan ngirim laporan status ke channel Slack management setiap pagi. Dua tools, dua peran, satu ecosystem.

---

## Sedikit Drama: Hubungan dengan Anthropic

Cerita OpenClaw dan Anthropic ini agak *rumit*, bisa dibilang.

- **Nov 2025**: Clawdbot diluncurkan, pakai Claude API sebagai provider utama
- **Jan 2026**: Rename ke Moltbot karena masalah merek dagang
- **Feb 2026**: Rename lagi ke OpenClaw (nama final)
- **Mar 2026**: Peter Steinberger join OpenAI; OpenClaw mulai support OpenAI
- **4 Apr 2026**: Anthropic ngasih tahu bahwa subscriber Claude Code gak bisa lagi pakai limit subscription buat OpenClaw
- **Apr 2026**: Anthropic launch **Claude Code Channels**, fitur yang mirip OpenClaw

**Sekarang (April 2026):**

- Kalau kamu mau pakai model Claude di OpenClaw, harus bayar **harga pay-as-you-go** Anthropic (gak bisa pakai bundling subscription)
- Anthropic punya produk sendiri (Claude Code Channels) yang jadi "kompetitor" langsung
- Tapi OpenClaw tetap agnostik, kamu bisa pakai provider lain tanpa masalah
- Persaingan ini justru mendorong inovasi di kedua sisi, yang untungnya buat kita sebagai user


---

## Yuk Mulai!

Udah cukup teorinya, sekarang kita langsung praktik.

### Instalasi

```bash
# Install via npm (cara paling gampang)
npm install -g openclaw

# Atau via Docker, kalau kamu lebih suka container
docker pull openclaw/openclaw:latest

# Atau build dari source, kalau kamu mau contribute
git clone https://github.com/openclaw/openclaw.git
cd openclaw
npm install && npm run build
```


### Onboarding

```bash
# Install dulu
npm install -g openclaw@latest

# Jalankan wizard onboarding
openclaw onboard --install-daemon

# Buka Control UI di browser
openclaw dashboard
```

Wizard-nya bakal nanyain beberapa hal:
1. Provider AI mana yang mau dipakai (Anthropic, OpenAI, dll)
2. Channel mana yang mau dihubungkan (Slack, Telegram, dll)
3. Setting dasar lainnya

Setelah selesai, kamu langsung bisa mulai ngobrol sama agent!

### Konfigurasi Dasar

File konfigurasi ada di `~/.openclaw/openclaw.json`. Ini file JSON5, jadi kamu bisa pakai comment juga:

```json5
{
  channels: {
    slack: {
      allowFrom: ["@user-kamu"],
      groups: { "*": { requireMention: true } }
    }
  }
}
```

Config ini ngatur siapa aja yang boleh chat sama agent kamu, di channel mana, dan dengan aturan apa. Misalnya `requireMention: true` berarti agent cuma bakal respon kalau di-mention di group chat, bukan merespon semua message. Ini penting biar agent gak "ngomong" di setiap percakapan yang gak relevan.

> **Pro Tip:** Setelah onboarding, coba jalankan `openclaw doctor` buat cek apakah semua berjalan lancar. Command ini bakal ngecek koneksi, config, dan kasih saran kalau ada yang perlu diperbaiki. Kayak health check tapi buat setup kamu.

---

## Kenapa Ini Penting Buat Kamu yang DevOps?

Pergeseran dari *assisted coding* ke *autonomous infrastructure agents* ini bukan cuma isu hangat. Ini beneran mengubah cara kita ngurus sistem:

1. **Coverage 24/7.** AI agent gak tidur, gak minta cuti, gak pernah miss alert. Ada anomaly jam 3 pagi? Agent udah handle sebelum kamu bangun.
2. **Knowledge base institusional.** Skills meng-enkode best practice tim jadi workflow yang bisa dijalankan ulang. Senior resign? Knowledge tetap di skills system.
3. **Korelasi multi-sistem.** Agent korelasikan data dari logs, metrics, traces, dan deployment history sekaligus. Manusia butuh 30 menit, agent? Hitungan detik.
4. **Eksekusi konsisten.** Runbook dijalankan persis sama setiap kali. Gak ada langkah terlewat karena ngantuk, gak ada typo karena buru-buru.
5. **Reduced cognitive load.** Kamu fokus ke arsitektur dan strategi, agent ngurus operasional rutin. Ini nilai utama automation.

Tapi kekuatan besar ini datang dengan tanggung jawab besar juga. Agent yang bisa akses production itu *kuat* tapi juga berisiko kalau gak di-setup dengan benar. Di bab-bab selanjutnya, kita bakal bahas gimana cara pakai OpenClaw dengan aman, mulai dari sandboxing, permission model, sampai policy enforcement.

---

## Ringkasan Bab Ini

- OpenClaw adalah AI agent gateway open-source (MIT) yang bisa di-host sendiri
- Terhubung ke 14+ platform pesan: Slack, Telegram, Discord, WhatsApp, dan lainnya
- Punya skills system yang bisa di-customize sesuai kebutuhan organisasi kamu
- Mendukung 35+ provider AI, gak terkunci ke satu vendor
- Bukan pengganti Claude Code, mereka melayani use case yang beda dan saling melengkapi
- Session persisten bikin agent makin paham context kamu seiring waktu

---

