# OpenClaw untuk DevOps

> *"Panduan praktis buat kamu yang pengen jadiin OpenClaw sebagai asisten AI buat operasi infrastructure kamu, dari deployment automation sampe incident response di jam 3 pagi."*

---

## Tentang Buku Ini

OpenClaw adalah gateway multi-channel yang open-source dan bisa di-self-hosted, yang nge-connect chat apps kamu ke AI coding agent. Kamu jalanin satu Gateway process di mesin kamu sendiri (atau server), dan gateway itu jadi jembatan antara messaging app favorit kamu sama AI assistant yang selalu standby 24/7. Gak ada vendor lock-in, gak ada biaya per-seat yang bikin budget IT berantakan, dan kamu punya full kontrol atas data, model, dan behavior-nya.

Buku ini nunjukin gimana DevOps engineer bisa pake OpenClaw di skenario production nyata: nge-automatisasi deployment, nge-monitor log, nanganin incident, nge-manage konfigurasi, dan nerapin security policy. Bukan teori doang, tapi pola-pola yang udah di-validasi sama tim-tim beneran di company yang beneran, lengkap dengan config, command, dan lesson dari insiden yang beneran terjadi.

Kenapa buku ini ditulis sekarang? Karena kita lagi ada di titik balik yang penting. AI agent buat infrastructure udah cukup matang buat di-deploy ke production, tapi masih cukup baru buat bikin banyak orang bingung gimana cara mulainya dengan aman. Tanpa panduan yang tepat, kamu bisa nge-deploy setup yang tampil bagus di demo tapi jadi bom waktu di production. Buku ini bakal nge-guide kamu dari awal sampe deployment yang matang, dengan fokus khusus ke safety dan governance yang sering di-skip di tutorial lain.

Yang bikin buku ini beda dari dokumentasi official adalah perspektifnya. Dokumentasi ngajarin kamu "apa yang bisa dilakuin", buku ini ngajarin kamu "apa yang sebaiknya kamu lakuin dan gimana caranya". Dua hal yang sangat berbeda.

### Target Audience

Kalau kamu termasuk salah satu dari profil di bawah, buku ini buat kamu:

- **DevOps dan platform engineer** yang pengen AI assistant yang bisa diakses dari channel manapun (Discord, Google Chat, iMessage, Matrix, Microsoft Teams, Signal, Slack, Telegram, WhatsApp, dan lainnya). Kamu udah capek buka 10 tab dashboard berbeda tiap hari.
- **SRE dan on-call engineer** yang butuh agent yang selalu tersedia buat nge-triage incident, ngejalanin diagnostic, dan eksekusi remediation playbook. Kamu pengen tidur tenang di malem hari.
- **Security engineer** yang pengen nge-enforce tool policy, nge-manage credential, dan nge-audit setiap aksi agent. Kamu paranoid, dan itu adalah kekuatan kamu.
- **Team lead** yang pengen setup multi-agent routing dengan workspace ter-isolasi per tim atau service. Kamu pengen agent yang scalable seluruh organisasi.

Kalau kamu baru mulai ngoprek AI agent dan belum tau harus mulai dari mana, buku ini juga cocok. Bab-bab awal ngasih konteks yang cukup, bab-bab tengah ngasih praktik yang solid, dan bab-bab akhir ngasih safety framework yang essential buat production.

### Prerequisites

Biar perjalanan kamu lancar, siapin dulu hal-hal berikut:

- **Node.js 24** direkomendasikan (Node 22.14+ juga didukung). Kalau kamu belum install, bisa pake `nvm` atau installer official dari nodejs.org.
- **API key dari model provider yang didukung**: Anthropic, OpenAI, Google, DeepSeek, Mistral, xAI, Ollama, atau banyak provider lain via OpenRouter, Together, Fireworks, dan seterusnya. Pilih provider yang cocok sama budget dan kebutuhan kamu.
- **Familiarity dasar dengan CLI tools dan JSON configuration.** Gak perlu jadi ahli, cukup bisa edit file config dan jalanin command tanpa panik.

Gak butuh pengetahuan mendalam tentang LLM internal atau machine learning. Buku ini ngebahas OpenClaw dari sudut pandang operasional, bukan research.

### Conventions

Biar konsisten dan gampang diikutin, ada beberapa konvensi yang aku pake:

- **Command** ditunjukin dengan prompt `$` buat terminal command dan `openclaw` polos buat CLI subcommand.
- **Config snippet** pake JSON5 (format yang dipake di `~/.openclaw/openclaw.json`). JSON5 itu kayak JSON tapi bisa ada comment dan trailing comma. Jauh lebih mudah dibaca.
- **`// comments`** di dalam config block itu buat penjelasan dan valid di JSON5, jangan kamu hapus.
- **Channel name** pake lowercase (`whatsapp`, `telegram`, `discord`, `signal`, `slack`, dan seterusnya).
- **Model name** ngikutin konvensi provider (`claude-3-5-sonnet-20241022`, `gpt-4o`, dan seterusnya).

---

## Referensi Cepat

Biar kamu gak harus bolak-balik buat nyari command dasar, berikut cheat sheet yang bisa kamu bookmark.

### Instal dan Mulai

```bash
# Install OpenClaw secara global
npm install -g openclaw@latest

# Jalanin interactive onboarding (setup provider, gateway, channel)
openclaw onboard --install-daemon

# Buka Control UI di browser
openclaw dashboard
```

### Command Inti

```bash
openclaw status                          # Cek gateway + health session
openclaw health                          # Health probe gateway
openclaw channels status --probe         # Cek konektivitas channel
openclaw config get agents.defaults.model.primary  # Baca config value
openclaw config set agents.defaults.model.primary "gpt-4o"  # Set config value
openclaw models list                     # List model yang tersedia
openclaw models scan                     # Auto-discover dan probe model
openclaw doctor                          # Health check + quick fix
openclaw logs --follow                   # Tail gateway log
```

Command yang paling sering dipakai sehari-hari adalah `status`, `doctor`, dan `logs --follow`. Tiga command ini yang bakal jadi teman terbaik kamu pas debugging. `doctor` khususnya sangat worth kamu jalanin pas ada yang aneh, karena dia otomatis mendeteksi dan kadang memperbaiki masalah dasar tanpa kamu harus manual dig.

### Lokasi File Konfigurasi

```
~/.openclaw/openclaw.json
```

### Contoh Konfigurasi

```json5
{
  // Allowlist channel
  channels: {
    telegram: {
      allowFrom: ["123456789"],
      groups: { "*": { requireMention: true } },
    },
  },
  // Konfigurasi model dan fallback
  agents: {
    defaults: {
      model: {
        primary: "claude-3-5-sonnet-20241022",
      },
    },
  },
  // Policy eksekusi tool
  tools: {
    exec: {
      ask: true,  // wajib approval buat exec command
    },
  },
}
```

Config di atas adalah "minimum viable setup" buat single-user deployment. Dia lock down channel ke satu user, wajib @mention di group, dan minta approval buat exec command. Tiga setting simple yang udah kasih kamu level security yang wajar buat mulai.

---

## Glosarium Cepat

Sebelum nyemplung ke bab pertama, ada baiknya kamu kenalan dulu sama istilah-istilah kunci yang bakal sering muncul. Gak perlu hafalin semuanya, cukup familiar biar gak bingung pas ketemu di bab-bab berikutnya.

- **Gateway** — Process central yang nge-route pesan antara channel dan AI agent
- **Channel** — Permukaan messaging (WhatsApp, Telegram, Discord, Slack, dll.)
- **Agent** — AI assistant yang punya workspace, config model, dan history session sendiri
- **Session** — Conversation yang ter-isolasi antara sender dan agent
- **Plugin** — Extension yang nambahin channel, provider, tool, atau capability
- **Skill** — Automation atau behavior terpaket yang di-install dari ClawHub
- **Sandbox** — Runtime ter-isolasi buat ngejalanin task agent dengan aman
- **Node** — Device yang di-pair (iOS/Android) yang nge-extend kemampuan agent (kamera, lokasi, dll.)
- **SecretRef** — Referensi ke secret yang disimpan di luar config (env var, file, atau exec command)

Yang paling penting di-ingat adalah konsep **Gateway**, **Agent**, dan **Session**. Tiga ini fondasi yang bakal muncul di hampir semua bab. Gateway itu process utamanya, Agent itu AI instance-nya, Session itu percakapannya. Kalau kamu paham tiga ini, kamu udah paham 80% mental model OpenClaw.

---

## Channel yang Dukung

OpenClaw dukung banyak messaging channel, baik yang built-in maupun via bundled plugin. Keberagaman ini adalah salah satu kekuatan utama OpenClaw: kamu bisa pake channel yang tim kamu udah familiar, gak perlu maksa mereka belajar tools baru.

**Built-in**: Discord, Google Chat, iMessage, IRC, Signal, Slack, Telegram, WebChat, WhatsApp

**Plugin channel**: BlueBubbles, Feishu, LINE, Matrix, Mattermost, Microsoft Teams, Nextcloud Talk, Nostr, QQ Bot, Synology Chat, Tlon, Twitch, Zalo, Zalo Personal

Kamu bisa connect multiple channel ke satu Gateway secara bersamaan. Setiap channel support allowlist, mention rule, dan per-account configuration. Jadi kamu bisa punya Slack buat komunikasi internal tim infra, Telegram buat notifikasi on-call, dan WhatsApp buat emergency access dari HP, semuanya di-route ke gateway dan agent yang sama. Satu otak, banyak wajah.

> **Coba Bayangin...** Jam 2 pagi kamu lagi tidur, terus HP bergetar. Alert dari production. Biasanya kamu harus bangun, buka laptop, SSH ke server, jalanin command, analisis hasilnya. Sekarang? Cukup balas di WhatsApp dari tempat tidur: "cek CPU usage production cluster, kalau di atas 90% scale up 2 node". Selesai. Tidur lagi.

---

## Penyedia Model

OpenClaw works sama berbagai AI model provider. Kamu gak terkunci ke satu vendor, bisa pilih yang paling cocok sama budget dan use case kamu. Bahkan bisa mix-and-match: pake Claude buat reasoning kompleks, GPT buat general task, dan Ollama local buat data sensitif yang gak boleh keluar network.

- **Anthropic** — API key, Claude CLI, setup-token. Model Claude, CLI reuse di-sanctioned
- **OpenAI** — API key, Codex. Model GPT
- **Google** — API key, Gemini CLI. Model Gemini
- **DeepSeek** — API key. Model DeepSeek
- **Mistral** — API key. Model Mistral
- **xAI** — API key. Model Grok
- **Ollama** — Local. Model self-hosted
- **OpenRouter** — API key. Proxy ke banyak provider
- **Together** — API key. Model open-source
- **Fireworks** — API key. Fast inference
- **Groq** — API key. High-speed inference
- **Dan banyak lagi** — Chutes, Hugging Face, NVIDIA, Qwen, MiniMax, xiaomi, Z.AI, dll.

Configure via `openclaw onboard` (interaktif, cocok buat pemula) atau `openclaw config set` (cocok buat automation dan scripting).

> **Pro Tip:** Kalau kamu baru mulai, pake satu provider aja dulu (biasanya Anthropic atau OpenAI), terus explore provider lain setelah kamu comfortable. Mix-and-match itu powerful, tapi bisa bikin debugging lebih kompleks kalau kamu belum familiar sama quirks masing-masing provider.

---

## Bacaan Lebih Lanjut

Buku ini bakal ngebahas semua konsep dengan detail yang cukup buat kamu bisa deploy ke production dengan aman. Tapi kalau kamu pengen dive lebih dalam ke topik tertentu atau nyari referensi tambahan, berikut resource official yang worth kamu bookmark:

- **Dokumentasi official**: [docs.openclaw.ai](https://docs.openclaw.ai)
- **Repository**: [github.com/openclaw/openclaw](https://github.com/openclaw/openclaw)
- **Getting started**: [docs.openclaw.ai/start/getting-started](https://docs.openclaw.ai/start/getting-started)
- **Reference konfigurasi**: [docs.openclaw.ai/gateway/configuration](https://docs.openclaw.ai/gateway/configuration)
- **Plugin SDK**: [docs.openclaw.ai/plugins/sdk-overview](https://docs.openclaw.ai/plugins/sdk-overview)
- **Channel**: [docs.openclaw.ai/channels](https://docs.openclaw.ai/channels)
- **Provider**: [docs.openclaw.ai/providers](https://docs.openclaw.ai/providers)
- **Reference CLI**: [docs.openclaw.ai/cli](https://docs.openclaw.ai/cli)

---

## Yuk, Mulai!

Udah cukup pembuka. Kalau kamu udah siap, lanjut ke bab pertama buat kenalan lebih dekat sama OpenClaw dan mulai perjalanan kamu jadi DevOps engineer di era AI agent.

Satu pesan terakhir sebelum kita mulai: **jangan takut buat bereksperimen, tapi jangan sembrono di production**. Buku ini nyediain rem buat kamu, gunakan. Setup environment test dulu, pelajarin behavior-nya, baru scale ke production. Step-by-step bukan tanda kamu lambat, tapi tanda kamu smart.

Selamat belajar, dan selamat nge-build masa depan DevOps kamu.

---

