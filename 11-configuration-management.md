# Bab 11: Config File, Ranjau Darat yang Kamu Pijak Tiap Hari

> *"Config file itu kayak ranjau darat. Kalau bener, kamu lewat tiap hari tanpa sadar ada di bawah kaki. Kalau satu koma kebalik, booom, production down jam 3 pagi pas kamu lagi enak-enaknya tidur."*

---

## Kenapa Config Management itu Diam-diam Jadi Mimpi Buruk

Config management itu topik yang paling gak seksi di DevOps. Gak ada developer yang conference talk-nya bangga pamer "kita punya JSON yang bersih". Tapi ironisnya, config yang berantakan itu sumber incident paling sering. Cuma koma salah tempat, environment variable yang kelupaan di-set, atau policy kelebihan longgar.

Aku pernah debug production outage 3 jam karena satu field di config di-rename tapi lupa update di internal docs. Engineer deploy pakai nama lama, gateway refuse to start, sampai akhirnya nemuin typo di line 247. Cerita kayak gini gak unik - setiap tim DevOps punya cerita trauma serupa.

Masalah config biasanya gak muncul di development. Di laptop jalan mulus, staging juga fine. Baru pas production, pas traffic tinggi, pas kamu liburan, baru deh sistem hang dan error cryptic. Nyari akarnya berjam-jam karena config tersebar di JSON, env var, secret vault, dan ratusan tempat lain.

OpenClaw ngasih pendekatan yang lebih aman. Alih-alih edit config manual, sistem ini enforce strict JSON Schema validation, automated drift detection via `doctor`, dan CLI untuk semua perubahan config yang bisa di-audit. Konfigurasi diperlakukan kayak kode production: setiap perubahan divalidasi, di-log, dan bisa di-rollback.


---

## Lanskap Tool Config Management di 2026

Landscape tool config management sekarang lumayan ramai. Ada yang fokus ke server config (Ansible, Chef, Puppet), ada yang fokus ke infrastructure provisioning (Terraform, CloudFormation), ada yang fokus ke application config (HashiCorp Consul, AWS AppConfig), dan ada yang hybrid kayak OpenClaw yang handle config buat AI agent sekaligus orchestration ke tool lain.

Pertanyaan sering muncul: "aku harus pakai yang mana?". Jawabannya jarang "salah satu", biasanya "kombinasi". Kamu mungkin pakai Terraform buat server, Ansible buat OS, dan OpenClaw buat AI agent. Tool-tool ini saling melengkapi.

- **OpenClaw** — AI agent config dan orchestration — Built-in schema validation dan auto-repair
- **Ansible** — Server configuration, orchestration — Exec tool integration, TaskFlow support
- **Terraform** — Infrastructure as Code — Config generation dari output IaC
- **Chef** — Infrastructure automation — Limited, via exec tool
- **Puppet** — Configuration enforcement — Limited, via exec tool
- **AWS SSM** — AWS-specific config management — Native secret provider integration
- **HashiCorp Consul** — Service discovery dan KV store — SecretRef via exec provider

OpenClaw di level orchestration, ngasih interface unified. Kamu minta "deploy staging" ke OpenClaw, dia trigger Terraform dulu, lanjut Ansible, terus update config internal. Semua langkah ter-audit.

> **Pro Tip:** Jangan pilih tool cuma karena populer. Evaluate berdasarkan siapa yang bakal maintain, learning curve-nya, dan expertise tim di situ. Tool yang kamu kuasai 80% lebih baik dari tool canggih tapi cuma ngerti 20%.

---

## Use Case 1: Generate Config dari Bahasa Indonesia

### Dari Intent ke Config yang Langsung Bisa Digunakan

Setup OpenClaw dari nol biasanya makan berjam-jam: baca docs, coba parameter, salah, baca lagi, fix. Hasilnya sering gak konsisten karena engineer pilih defaults berbeda.

Dengan AI-assisted config generation, kamu bisa lewati banyak langkah. Cukup deskripsi yang kamu butuhkan pakai bahasa biasa, sistem terjemahkan ke config valid. Bukan template kosong. Sistem parse requirement, cross-reference dengan schema, isi defaults masuk akal, hasilnya config yang bisa dipakai langsung.

```
Engineer: "Set up production-ready OpenClaw instance dengan:
           - WhatsApp channel dengan pairing policy
           - Claude Sonnet sebagai model utama
           - GitHub dan weather skill
           - Session reset harian jam 4 pagi
           - Sandboxing untuk semua non-main agent"

OpenClaw: "Generating configuration with interactive wizard..."
```

Sistem tanya follow-up kalau ada yang ambigu. "Production itu high-traffic atau low-traffic? Berapa concurrent session? Ada PCI-DSS?". Ini bikin hasilnya sesuai kebutuhan, bukan generic defaults.

Output yang di-generate kurang lebih kayak gini:

```json5
// ~/.openclaw/openclaw.json (generated)
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["openai/gpt-5-4"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "openai/gpt-5-4": { alias: "GPT" },
      },
      skills: ["github", "weather"],
      sandbox: {
        mode: "non-main",
        scope: "agent",
      },
      heartbeat: {
        every: "30m",
        target: "last",
      },
    },
    list: [
      { id: "main", default: true },
    ],
  },
  channels: {
    whatsapp: {
      enabled: true,
      dmPolicy: "pairing",
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
  session: {
    dmScope: "per-channel-peer",
    reset: {
      mode: "daily",
      atHour: 4,
      idleMinutes: 120,
    },
  },
  gateway: {
    port: 18789,
    reload: { mode: "hybrid", debounceMs: 300 },
    channelHealthCheckMinutes: 5,
  },
}
```

Config ini sudah termasuk `heartbeat`, `fallbacks` model, dan `reload mode` yang kamu mungkin lupa. Sistem tahu dari konteks "production" defaults masuk akal. Untuk "development", dia kasih heartbeat lebih jarang dan reload mode lebih agresif.

> **Contoh Nyata:** Startup e-commerce butuh 3 agent: customer service via WhatsApp, internal ops via Slack, data analyst dengan documentation skill. Setup selesai dalam 25 menit.

### Config CLI untuk Operasional Harian

Generate initial config itu sekali saja. Yang kamu lakukan tiap hari adalah update, tweak, atau fix config yang sudah ada. Di situlah CLI jadi tulang punggung workflow.

```bash
# Baca field spesifik
openclaw config get agents.defaults.workspace
openclaw config get channels.whatsapp.dmPolicy

# Update satu field (cara yang disarankan)
openclaw config set agents.defaults.heartbeat.every "2h"
openclaw config set channels.telegram.dmPolicy "pairing"

# Hapus field
openclaw config unset plugins.entries.brave.config.webSearch.apiKey

# Lihat seluruh config
openclaw config view

# Reference schema
openclaw config schema           # print JSON Schema
openclaw config schema --yaml    # format alternatif
openclaw config schema --check   # validate current config
```

Setiap command CLI punya built-in validation. Kamu tidak bisa set field yang tidak valid atau tidak ada di schema. Setiap perubahan ter-log di audit trail. Ini beda jauh dengan "buka file pakai vim, edit, save".

> **Eits, Hati-hati!** Godaan edit `~/.openclaw/openclaw.json` pakai vim itu real, terutama kalau perubahannya cuma satu line. Tapi setiap kali bypass CLI, kamu lewati validation. Aku sudah lihat tim incident turun drastis setelah adopt policy: semua config change harus lewat CLI, tanpa kecuali.


---

## Use Case 2: Deteksi Drift, Alarm untuk Config yang Menyimpang

### Kenapa Config Drift Itu Diam-diam Berbahaya

Drift itu technical debt versi config: state real system beda dari config. Atau engineer tweak console lupa update file, auto-scale bikin instance baru tidak di-track.

Dikit-dikit tidak terasa, tapi setelah bulan, config sama reality sistem sudah divergen jauh. Drift sering tidak ketangkep sampai incident. Pas masalah muncul, engineer on-call bingung: "kok config bilang pakai instance type X, tapi real-nya type Y?". Troubleshooting makan waktu dua kali lipat.

OpenClaw berikan drift detection built-in lewat command `doctor`. Command ini scan semua aspek config kamu (gateway, channel, agent, secret, workspace), bandingkan dengan ideal state di file, dan report apa aja yang beda. Kalau ada yang bisa di-repair otomatis, dia tawarkan untuk apply fix-nya. Kalau butuh keputusan manual, dia kasih opsi buat kamu pilih.

```bash
# Health check dasar
openclaw doctor

# Auto-repair dengan konfirmasi
openclaw doctor --repair

# Auto-repair aggressive (langsung apply tanpa tanya)
openclaw doctor --repair --force

# Deep scan buat detect extra service atau orphan process
openclaw doctor --deep

# Cek integrity config aja
openclaw config schema --check

# Test connectivity semua channel
openclaw channels status --probe
```

Contoh output `doctor` yang nemuin beberapa drift:

```bash
$ openclaw doctor
Scanning OpenClaw configuration...

✓ Configuration schema valid
⚠ Drift detected:
  - Legacy key: routing.allowFrom (deprecated di v2.0)
  - Should be: channels.whatsapp.allowFrom
  - Action: Automated migration available

🔧 Health checks:
  ✓ Gateway running on port 18789
  ✓ WhatsApp channel responsive
  ✓ Agent sessions clean
  ⚠ OpenRouter API key expires in 7 days
  ✓ No orphaned processes

✗ Security warnings:
  - telegram dmPolicy "open" (recommend "pairing" or "allowlist")
  - Agent workspace ~/.openclaw/workspace-work not accessible

? Apply recommended repairs? [Y/n] y
Migrated routing.allowFrom → channels.whatsapp.allowFrom
Updated telegram.dmPolicy: open → pairing
Workspace ~/.openclaw/workspace-work created
System healthy.
```

`doctor` kasih context: "expires in 7 days", "recommend pairing", "not accessible". Ini bikin kamu bisa langsung bertindak.

> **Contoh Nyata:** Fintech company schedule `openclaw doctor` tiap jam. Suatu hari nemu API key primary payment provider expired 3 hari. Mereka gak sadar karena backup provider masih jalan. Kalau gak ada drift detection, mereka baru sadar pas backup provider fail.

### Hot Reload: Terapkan Perubahan Tanpa Restart

Dulu kalau kamu change config, kamu harus restart service. Ini painful di production karena ada downtime, dan bikin tim ragu buat update config. OpenClaw support hot reload: perubahan config tertentu bisa di-apply tanpa restart, cuma yang butuh state reset aja yang trigger restart.

- **`hybrid`** (default) — Hot-apply yang aman, auto-restart yang critical — Production, balanced
- **`hot`** — Hot-apply only, warn kalau butuh restart — Development, fast iteration
- **`restart`** — Full restart tiap ada perubahan — Testing, safety-first
- **`off`** — Disable watching, restart manual — Controlled environment

```json5
{
  gateway: {
    reload: { mode: "hybrid", debounceMs: 300 },
    channelHealthCheckMinutes: 5,
  },
}
```

`debounceMs: 300` artinya kalau kamu ubah 5 field dalam 300ms, sistem tunggu sebelum apply. Ini prevent cascade restart kalau ada automation yang push banyak config change. Misalnya `openclaw config set` beberapa kali berturut-turut, sistem batch jadi satu reload.

Hot reload bikin workflow config management jauh lebih longgar. Kamu bisa experiment, tweak timeout value, coba skill baru, atau adjust threshold tanpa takut ganggu production traffic. Friction turun drastis, dan tim jadi lebih berani improve config karena cost eksperimennya rendah.

> **Pro Tip:** Mode `hybrid` default bijak buat production. Di development, pakai `hot` biar tidak ada restart. Di production, stick sama `hybrid`.


---

## Use Case 3: Multi-Agent Workflow Konfigurasi

### Ketika Satu Agent Sudah Tidak Cukup

Di awal, satu agent cukup. Tapi seiring waktu, ini jadi berantakan. Context task pribadi bocor ke diskusi work, agent bingung mana yang sensitive.

OpenClaw mendukung multi-agent dengan workspace terpisah. Bayangkan seperti kantor dengan beberapa ruangan terpisah. Ruang CEO terpisah dari ruang IT, ruang IT terpisah dari warehouse. Setiap ruangan punya access control sendiri, data sendiri, tidak bocor ke ruangan lain. Isolation penting buat task dengan sensitivity level berbeda.

```json5
{
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-sonnet-4-6" },
      workspace: "~/.openclaw/workspace",
    },
    list: [
      { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
      { id: "work", workspace: "~/.openclaw/workspace-work" },
      { id: "docs", workspace: "~/.openclaw/workspace-docs" },
    ],
  },
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
    { agentId: "docs", match: { channel: "telegram", accountId: "docs-team" } },
  ],
}
```

`bindings` handle routing otomatis. Message dari WhatsApp personal route ke agent "home", business ke "work", Telegram grup docs-team ke "docs". Semua transparan.

Multi-agent architecture nyelesein beberapa masalah sekaligus. Pertama, **isolation**: task personal tidak bercampur dengan task work. Kedua, **scalability**: kalau agent "work" butuh resource lebih, scale itu aja tanpa pengaruhi agent lain. Ketiga, **security**: kalau satu agent kena compromise, agent lain tetap aman karena workspace-nya isolated. Keempat, **audit**: setiap agent punya session log terpisah, gampang di-trace.

```bash
# Status semua agent
openclaw status --all

# Kirim message ke agent spesifik
openclaw message send --agent home "Reminder meeting sore jam 4"
openclaw message send --agent work "Deploy ke staging dan report hasilnya"

# Config skill per agent
openclaw config set agents.list.home.skills ["reminder", "note"]
openclaw config set agents.list.work.skills ["github", "aws", "slack"]

# List session per agent
openclaw sessions --agent home
openclaw sessions --agent work
```

> **Coba Bayangkan...** Kamu punya asisten pribadi yang handle semua urusan kamu, dari jadwal dokter sampai deploy production. Satu orang yang tahu semua detail hidup kamu. Convenient, tapi creepy, dan kalau dia sakit kamu lumpuh total. Multi-agent itu kebalikannya: spesialis untuk setiap domain, kalau satu ada issue, yang lain masih jalan.

### Setup Multi-Agent yang Umum

Setup yang sering aku lihat:

- **Agent "home"**: Reminder, note, calendar. Skill minimum, model cheap. Session retention 7 hari.
- **Agent "work"**: Production, deployment, monitoring. Full skill set. Model capable. Session retention 30 hari.
- **Agent "docs"**: Documentation search, knowledge management. Skill khusus. Model fast.

Setiap agent punya budget, skill set, dan security profile sendiri. Ini bikin scaling mudah: kalau volume "work" naik, allocate resource tanpa tweak agent lain. Perubahan di "home" tidak berpengaruh ke production.

> **Pro Tip:** Kalau tim distributed across timezone, consider bikin agent per timezone. Misal agent "asia-pacific" dan "americas". Setup seperti ini bikin global operations lebih reliable.


---

## Use Case 4: Secret dan Variabel Environment Management

### Brankas untuk Kunci-Kunci Rahasia

Secret leakage masalah besar. Hardcode API key di JSON config? Buruk. Pas share config ke teman, secret kelihatan. Commit API key ke git? Lebih parah, key permanent di git history.

Aku lihat startup panik setelah junior commit `.env` ke public repo. Bot scrape key-nya dan mulai mining Bitcoin. Tagihan AWS $12,000. Pelajaran mahal kalau dari awal pakai secret management yang proper.

OpenClaw berikan flexible secret management yang aman secara default:

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: { GROQ_API_KEY: "gsk-..." },
    shellEnv: { enabled: true, timeoutMs: 15000 },
  },
  
  models: {
    providers: {
      openai: { 
        apiKey: { 
          source: "env",
          provider: "default", 
          id: "OPENAI_API_KEY" 
        } 
      },
      custom: {
        apiKey: { 
          source: "file",
          provider: "filemain", 
          id: "/etc/secrets/custom-provider.txt" 
        }
      },
    },
  },
  
  channels: {
    googlechat: {
      serviceAccountRef: {
        source: "exec",
        provider: "vault",
        id: "channels/googlechat/serviceAccount",
      },
    },
  },
}
```

Parameter `source` tentukan dari mana sistem ambil secret:

- **`env`**: Environment variable. Di-set via `export OPENAI_API_KEY=...` atau di file `.env`. Paling mudah dan umum dipakai di development.
- **`file`**: File lokal yang isinya secret. Berguna untuk secret yang panjang seperti service account JSON, atau yang butuh access control khusus di filesystem level.
- **`exec`**: Output dari command. Ini yang paling powerful karena bisa integrate sama HashiCorp Vault, AWS Secrets Manager, atau Google Secret Manager. Command yang kamu berikan yang fetch actual secret-nya, jadi secret tidak pernah ada di config file.

Environment variable precedence (yang lebih tinggi override yang lebih rendah):

1. Parent process env var (dari shell)
2. `.env` di current working directory
3. `~/.openclaw/.env` (global fallback)
4. Inline env var di config

Contoh kasus:

```bash
# Di shell
export OPENAI_API_KEY=sk-abc123

# Di ~/.openclaw/.env
ANTHROPIC_API_KEY=sk-ant-xyz789

# OpenClaw bakal pakai OPENAI_API_KEY dari shell
# dan ANTHROPIC_API_KEY dari ~/.openclaw/.env
```

Kalau config kamu butuh reference ke secret, kamu bisa pakai placeholder:

```json5
{
  gateway: {
    auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" },
  },
  models: {
    providers: {
      custom: { apiKey: "${CUSTOM_API_KEY}" },
    },
  },
}
```

Saat runtime, sistem substitute `${VAR}` dengan nilai dari environment. Kalau variable-nya tidak ada, sistem error. Ini lebih baik daripada silent fail dengan empty string, karena kamu langsung tahu ada yang kurang dan bisa fix sebelum deploy. Silent fail itu biang kerok banyak incident yang susah di-debug.

> **Eits, Hati-hati!** Jangan hardcode API key di config file, meskipun di `.gitignore`. Selalu ada risiko file ke-commit tidak sengaja. Investasi awal untuk setup proper secret management itu cuma beberapa menit, tapi save kamu dari potential breach.

### Scale Secret Management ke Banyak Environment

Kalau kamu punya banyak environment, managing secret jadi kompleks. OpenClaw mendukung multiple secret provider:

```json5
{
  secrets: {
    providers: {
      filemain: { 
        root: "~/.openclaw/secrets" 
      },
      env: {},
      vault: {
        address: "https://vault.example.com:8200",
        token: "${VAULT_TOKEN}",
        timeout: "5s",
      },
      exec: {
        command: "aws secretsmanager get-secret-value --secret-id openclaw/prod",
        timeoutMs: 10000,
      },
    },
  },
}
```

Contoh workflow pakai vault buat production:

```bash
# Ambil secret dari vault
openclaw secret get --provider vault --path secret/openclaw/api-keys/openai

# Set secret ke vault (disarankan buat production)
openclaw secret set --provider vault --path secret/openclaw/api-keys/openai --value "sk-..."

# List semua secret di vault
openclaw secret list --provider vault
```

Setup mature pakai layer berbeda: development pakai file lokal, staging pakai AWS Secrets Manager, production pakai Vault. Ini membuat blast radius breach kecil.

> **Contoh Nyata:** Company SaaS dengan 3 environment. Dev pakai local file, staging pakai AWS Secrets Manager, production pakai Vault. Laptop developer hilang, tim security rotate secret dev environment saja. Staging dan production tidak perlu di-rotate.


---

## Use Case 5: Validasi Konfigurasi

### Pre-Apply Validation Itu Mirip Sabuk Pengaman

Validation itu membosankan tapi critical. Bayangkan seperti sabuk pengaman di mobil. Kamu tidak pernah sadar dia ada sampai kamu butuh. OpenClaw enforce strict JSON Schema validation.

Schema itu definisi formal tentang config yang valid. Field apa yang boleh ada, tipe data yang diharapkan, mana yang required, mana yang optional. Kalau config violate schema, sistem tolak dengan error message spesifik.

```bash
# Print JSON Schema (buat reference)
openclaw config schema

# Validate config sekarang
openclaw config validate --strict

# Comprehensive health check
openclaw doctor --check

# Cek aspek spesifik
openclaw models status           # cek API key valid
openclaw channels status --probe   # test connectivity semua channel
openclaw config ports --check-conflicts  # cek port conflict
```

Contoh validation error yang helpful:

```bash
$ openclaw gateway start
Error: Configuration validation failed

Field "agents.default.workspace": 
  unknown field "default" at ~/.openclaw/openclaw.json:12
  Did you mean "defaults"? (plural form)

Field "channels.whatsapp.dmPolicy": 
  expected one of [open, pairing, allowlist, disabled]
  got "opened" at ~/.openclaw/openclaw.json:47
  Did you mean "open"?

Field "gateway.port":
  expected integer between 1 and 65535
  got "18789a" at ~/.openclaw/openclaw.json:89
  Looks like trailing character. Did you mean 18789?

Please fix these errors and try again.
Run 'openclaw doctor' for automated repair suggestions.
```

Error message kasih nama field yang keliru, path file, line number, value expected dan actual. Ini bikin debugging cepat.

Validation penting karena beberapa alasan.

**Prevent startup failure**. Gateway gak bisa start karena typo config. Validation catch typo sebelum deploy.

**Catch typo pas development**. Lebih baik discover typo di laptop jam 2 siang daripada production jam 3 pagi.

**Document expected structure**. Schema itu living documentation.

**Enable safer automation**. Kalau kamu bikin tool yang auto-generate config, strict schema jadi guardrail.

```bash
# Real validation scenario dengan auto-suggest
$ openclaw config set agents.list.main.mode "sandboxd"  # typo!
Error: Invalid enum value "sandboxd"
Valid values: sandbox, permissive, disabled
Did you mean "sandbox"?

Apply fix? [y/n] y
Config updated: agents.list.main.mode = "sandbox"
```

Suggestion based on similarity. Typo kecil seperti `sandboxd` dia suggest `sandbox`.


---

## Anti-Pattern: Jangan Lakukan Ini!

### 1. Edit Manual File Config Langsung

Ini dosa nomor satu yang paling sering bikin masalah. Kenapa sering kejadian? Karena menggoda banget, terutama kalau kamu engineer senior yang sudah terbiasa edit config manual. "Ah, cuma ganti satu field, gampang kok", kata si engineer, terus buka vim.

```bash
# SALAH: Edit langsung pakai text editor
vim ~/.openclaw/openclaw.json
# Typo koma, miss bracket, gateway tidak start
# Sekarang debug JSON yang rusak jam 3 pagi

# BENAR: Pakai CLI dengan built-in validation
openclaw config set agents.defaults.workspace "~/.openclaw/ws"
# Validation langsung, error message helpful, safe rollback
```

Problem-nya bukan cuma typo. Saat edit manual, kamu bypass audit log, skip validation, dan tidak ada rollback otomatis. Kalau config bikin masalah, kamu harus inget sendiri apa yang berubah dan restore manual. Dengan CLI, semua perubahan ter-track.

### 2. Policy Security yang Tidak Konsisten Antar Channel

```json5
// SALAH: Policy beda buat threat model yang sama
{
  channels: {
    whatsapp: { dmPolicy: "open" },      // siapa aja bisa DM
    telegram: { dmPolicy: "pairing" },    // hanya paired user
    slack: { dmPolicy: "allowlist" },     // explicit allowlist
  },
}

// BENAR: Consistent policy buat threat model yang sama
{
  channels: {
    whatsapp: { dmPolicy: "pairing" },
    telegram: { dmPolicy: "pairing" },
    slack: { dmPolicy: "pairing" },
  },
}
```

Inconsistency ini bug halus. "WhatsApp public, jadi open. Telegram private, pakai pairing. Slack internal, allowlist". Tapi dari security, inconsistency bikin mental model tim模糊. Engineer harus inget policy mana untuk channel mana, dan pas urgent situation, mereka bakal assume defaults yang salah.

> **Eits, Hati-hati!** Engineer tidak sengaja kirim data sensitive ke grup WhatsApp public karena pikir policy-nya ketat (padahal config-nya `open`). Data di-forward ke competitor. Set policy consistent dan dokumentasi jelas.

### 3. Mengabaikan Configuration Drift

```bash
# SALAH: Jarang atau gak pernah cek drift
# Config jadi out of sync sama yang terdokumentasi
# Pas incident, debugging jadi susah karena actual state
# gak match expected state

# BENAR: Schedule drift detection rutin
openclaw cron add --name "Config Drift Check" \
  --cron "0 2 * * *" \
  --command "openclaw doctor --repair" \
  --announce
```

Drift itu akumulatif. Hari pertama drift-nya satu field kecil. Seminggu kemudian sudah 5 field. Sebulan kemudian kamu sudah tidak bisa bilang apa yang sebenarnya running di production. Dan pas ada incident, engineer on-call bingung karena config di git tidak sama dengan config di system.

Solusinya simple: schedule drift detection rutin, tiap hari jam sepi. Kasih flag `--announce` biar hasilnya otomatis di-post ke channel tim. Dalam 1-2 bulan, drift kamu bakal stabil di angka nol karena tim jadi sadar pakai CLI.

### 4. Secret yang Hardcoded di File Config

```json5
// SALAH: Hardcode secret di config
{
  models: {
    providers: {
      openai: { apiKey: "sk-abc123xyz789" }  // JANGAN PERNAH
    }
  }
}

// BENAR: Pakai environment variable atau secret provider
{
  models: {
    providers: {
      openai: { 
        apiKey: { 
          source: "env",
          id: "OPENAI_API_KEY"
        }
      }
    }
  }
}
// Set via: export OPENAI_API_KEY=sk-abc123xyz789
```

Ini klasik. Setiap tahun ada cerita company breach karena secret di-commit. Solusinya mudah: pakai environment variable atau secret provider. Setup awalnya cuma 5 menit.

Kalau nemuin hardcoded secret di config existing, **rotate key-nya sekarang juga**, sebelum refactor. Anggap secret sudah compromised. Urutan penting: rotate dulu, refactor kemudian.

---

## Checklist: Config Management yang Aman untuk Production

Sebelum deploy ke production dengan OpenClaw, pastiin semua ini sudah terpenuhi:

- [ ] Config pakai JSON5 syntax yang valid
- [ ] Schema validation passes (`openclaw config validate`)
- [ ] `openclaw doctor` jalan dengan hasil healthy
- [ ] Semua secret pakai environment variable atau secret provider
- [ ] Config file di-backup (di git atau storage lain)
- [ ] Channel DM policy consistent across channel
- [ ] Multi-agent routing terkonfigurasi benar
- [ ] Gateway reload mode sesuai kebutuhan
- [ ] Semua API key belum expired dan masih valid
- [ ] Agent workspace path exist dan accessible
- [ ] Drift detection ter-schedule jalan rutin
- [ ] Audit log config change aktif
- [ ] Emergency rollback procedure ter-test

### Referensi Command Esensial

```bash
# Setup dan validation
openclaw onboard                 # interactive setup wizard
openclaw doctor                  # health check plus auto repair
openclaw config schema           # print schema reference

# Daily config management
openclaw config get KEY          # baca field spesifik
openclaw config set KEY VALUE    # update field
openclaw config unset KEY        # hapus field
openclaw config view             # lihat seluruh config
openclaw config validate         # validate config sekarang
openclaw config revert           # rollback ke versi sebelumnya

# Operations
openclaw status --all            # status semua komponen
openclaw health                  # health check cepat
openclaw sessions                # list active session
openclaw channels status         # status connectivity channel
```


---

## Poin Penting

- Config itu **fondasi** sistem yang robust. Jangan di-skip cuma karena kelihatan boring
- **Schema validation** bukan pembatas, tapi sabuk pengaman. Dia catch typo sebelum jadi outage
- **CLI > manual edit**, selalu. Setiap kali pakai `openclaw config set`, kamu dapet validation, audit trail, dan rollback gratis
- **Drift detection rutin** jadi asuransi config kamu. Schedule `openclaw doctor` tiap hari
- **Secret management proper** itu wajib. Hardcoded secret penyebab breach umum
- **Multi-agent architecture** bikin scaling clean. Pisahkan concerns berdasarkan sensitivity
- **Consistent security policy** prevent confusion. Policy tidak konsisten bikin engineer salah assume
- **Hot reload** kurangi friction. Pakai mode `hybrid` buat production, `hot` buat development

---

