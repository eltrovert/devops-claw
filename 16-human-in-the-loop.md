# Bab 16: Human-in-the-Loop, Kapan AI Harus Berhenti dan Nanya

> *"Otonomi AI itu kayak gas mobil. Kalau gak ada gas sama sekali, mobilnya gak gerak. Kalau gas full tanpa rem, kamu nabrak. Yang kamu butuhin bukan gas maksimum, tapi rem yang responsif dan tangan di setir yang tau kapan harus injak."*

---

## Spektrum Otonomi: Dari Full Manual ke Full Autonomous

Ada fallacy yang sering bikin engineer salah arah: asumsi bahwa "lebih otonom = lebih baik". Seolah-olah tujuan kita adalah bikin agent yang bisa kerja sendirian tanpa intervensi manusia sama sekali. Padahal dalam praktiknya, full autonomy itu hampir selalu solusi yang salah.

![Ilustrasi autonomy spectrum dengan slider yang menunjukkan sweet spot di "agent proposes, human approves" region](images/ch16-autonomy-spectrum.png)

Di kiri spektrum, ada full manual: human yang ngelakuin semuanya, AI cuma jadi kalkulator mahal. Ini paling aman tapi tidak scalable. Di kanan spektrum, ada full autonomous: agent yang kerja sendirian tanpa approval apa pun. Ini feel scalable tapi sangat berbahaya.

Sweet spot-nya di tengah: **otonomi maksimum yang masih safe buat task tertentu**. Beda task, beda level otonomi. Restart pod di staging? Agent bisa lakuin sendiri. Restart service payment di production? Butuh human approval. Drop database? Butuh approval dari 2 engineer senior.

Istilah kunci:
- **Autonomy level**: Tingkat kebebasan agent tanpa human approval
- **Human-in-the-loop**: Sistem yang require intervensi manusia
- **Approval gate**: Check point wajib sebelum action critical
- **Progressive autonomy**: Strategy nge-build trust secara gradual

---

## Pattern 1: Exec Approval

### Pengertian Exec Approval

Agent OpenClaw biasanya jalan di sandbox buat isolation. Tapi kadang-kadang dia butuh execute sesuatu di host, di luar sandbox. Misalnya restart service, deploy ke production, atau akses resource yang gak reachable dari sandbox. Exec approval adalah mekanisme kontrol buat akses ke host execution dari agent yang isolated.

Bayangkan asisten virtual di rumah kamu yang bisa menjalankan perintah. Kamu mau dia bisa menjalankan `turn on the lights` otomatis? Mungkin OK. Tapi bagaimana kalau dia mau menjalankan `unlock the front door`? Kamu pasti mau approve dulu. Exec approval ini adalah perbedaan antara perintah yang boleh dijalankan agen secara otonom dan perintah yang memerlukan konfirmasi manusia.

### OpenClaw Approval System yang Native

OpenClaw menyediakan sistem exec approval bawaan dengan mode security dan allowlist yang dapat dikonfigurasi.

```json5
{
  tools: {
    exec: {
      host: "auto",          // auto: sandbox when active, gateway otherwise
      security: "allowlist", // deny | allowlist | full
      ask: "on-miss",        // off | on-miss | always
      safeBins: ["cut", "uniq", "head", "tail", "tr", "wc"],
      safeBinTrustedDirs: ["/bin", "/usr/bin", "/opt/homebrew/bin"],
    },
  },
  
  tools: {
    elevated: {
      enabled: true,
      allowFrom: {
        discord: ["user-id-123"],
        slack: ["U456"],
      },
    },
  },
}
```

Tiga mode security:

**`deny`**: Default block. Gak ada host execution yang boleh jalan kecuali ada allowlist entry. Paling aman tapi restrictive.

**`allowlist`**: Default block, tapi perintah yang sesuai pola allowlist boleh jalan tanpa approval. Paling umum digunakan.

**`full`**: Default allow. Mode yang berbahaya dan hampir tidak pernah sesuai untuk production.

> **Eits, Hati-hati!** Jangan pernah pakai `security: "full"` kecuali kamu punya alasan yang sangat spesifik. Mode full ini pada dasarnya menonaktifkan layer approval, dan kamu bergantung pada sandbox plus layer lain untuk keamanan. Jika salah satu layer gagal, tidak ada fallback.

### Approval Flow

Pas agent yang isolated butuh execute di host, flow-nya:

1. **Request**: Agent run `exec` dengan `host=gateway` atau `host=node`
2. **Security check**: Gateway check configured policy
3. **Approval prompt**: Kalau butuh, display approval interface
4. **Execution**: Setelah approval (atau langsung kalau mode full)

Approval request diteruskan ke channel yang kamu pilih. Engineer tidak perlu log in ke dashboard, cukup tap button di Slack.

Contoh approval card di Slack:

```
🔒 Approval Required

Action: kubectl restart deployment/payment-service -n production
Command: kubectl restart deployment/payment-service -n production
Host: gateway-prod-01
Security: allowlist

[✅ Approve] [❌ Deny] [ℹ️ Details]
```

CLI buat management approval:

```bash
# Cek status approval sekarang
openclaw approvals get

# Set approval defaults buat node
openclaw approvals set --node node-01 --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "allowlist",
    ask: "on-miss",
    askFallback: "deny"
  }
}
EOF

# Tambah ke allowlist
openclaw approvals allowlist add --pattern "~/Projects/**/bin/rg" --node node-01
```

> **Contoh Nyata:** Tim DevOps di startup logistik pake Slack buat approval flow. Pas agent mau restart database di production, Slack card muncul dengan informasi lengkap: command apa, impact-nya, di mana bakal di-run, dan siapa yang request. Tim SRE yang on-call bisa review dan approve/deny dengan satu tap. Latency dari request sampai execute rata-rata 45 detik, dan approval rate 95% (hampir semua request legit). Yang 5% di-deny itu justru ketangkep request yang mistaken atau misconfigured sebelum damage terjadi.

---

## Pattern 2: Elevated Mode

### Pecah Kaca untuk Operasi Kritis

Sandbox itu layer security utama, tapi kadang-kadang operasi critical butuh akses di luar sandbox tanpa lewat approval biasa. Elevated mode ini mengikuti konsep "break glass": pecah kaca untuk akses darurat, dengan catatan bahwa akses tersebut tercatat dan dapat diaudit.

Bayangkan ruang server yang selalu terkunci. Biasanya kamu membutuhkan kartu akses untuk masuk, dan akses tersebut tercatat. Tapi kalau ada kebakaran, ada tombol "break glass" yang langsung membuka kunci dan mengaktifkan alarm. Siapa pun yang memecah kaca itu, identitas mereka tercatat, waktu kejadian ter-log, dan laporan insiden otomatis dibuat. Elevated mode bekerja dengan prinsip yang sama.

### Session Control via Slash Command

```
/elevated on    # Run outside sandbox, tetap butuh approval
/elevated ask   # Sama dengan 'on' (alias)
/elevated full  # Run outside sandbox, skip approval
/elevated off   # Kembali ke sandbox confinement
```

Empat level yang distinct:

**`on`**: Menjalankan di luar sandbox, tapi tetap memerlukan approval. Ini elevated yang masih terkontrol, untuk operasi yang membutuhkan akses host.

**`full`**: Menjalankan di luar sandbox, melewati approval. Ini "break glass", hanya boleh digunakan dalam keadaan darurat. Harus ada kebijasan yang jelas kapan `full` diperbolehkan.

**`off`**: Kembali ke normal sandbox confinement.

`allowFrom` menentukan siapa yang boleh mengaktifkan elevated mode. Tidak semua user boleh, hanya user ID tertentu.

> **Eits, Hati-hati!** Mode `full` mengabaikan semua approval. Gunakan hanya untuk keadaan darurat. Jika kamu menggunakan `full` lebih dari sekali seminggu, itu adalah tanda bahwa alur approval normal kamu terlalu restriktif. Break-glass yang terlalu sering digunakan menjadi shortcut, bukan mekanisme keamanan.

### Resolution Order

Pas ada conflict antara setting di level berbeda, OpenClaw resolve dengan urutan ini:

1. **Inline directive**: `/elevated on run deployment script` (per-command basis)
2. **Session override**: `/elevated full` (set default buat seluruh session)
3. **Global default**: `agents.defaults.elevatedDefault` di config

Inline menang atas session, session menang atas global. Ini pattern yang intuitif: yang lebih spesifik menang dari yang lebih general.


---

## Pattern 3: Session Override

### Kontrol Execution Per-Session

Gak semua session sama. Session debugging mungkin butuh akses lebih longgar sementara engineer lagi investigate production issue. Session production deployment harus lebih ketat dari biasa karena impact-nya tinggi. Session override kasih kamu kemampuan adjust per-session, tanpa harus edit global config tiap kali.

```bash
# Set session defaults
/exec host=auto security=allowlist ask=on-miss node=node-01

# Cek setting sekarang
/exec

# Override buat single message
/exec security=full ask=off restart pods
```

Perubahan hanya mempengaruhi session tersebut. Setelah session berakhir, override hilang dan kamu kembali ke default. Tidak ada risiko "oh iya lupa revert security setting" yang membuat config kamu menjadi lemah secara permanen.

### Authorization Model

Session override hanya bekerja untuk pengirim yang memiliki otorisasi. Update hanya mempengaruhi state session, tidak ditulis ke config permanen. Host approval masih berlaku kecuali kamu secara eksplisit mengatur `security=full` plus `ask=off`.

> **Contoh Nyata:** Session debugging buat troubleshoot error production. Engineer on-call nge-set `/exec host=auto security=allowlist ask=on-miss` di awal session biar bisa investigate cepat tapi masih ada safety net. Setelah issue resolved, session end, dan setting otomatis reset. Gak ada residual state.

---

## Pattern 4: Safe Bins

### Pre-Approved Stream Filter

Beberapa command itu beneran simple dan zero-risk. Command kayak `head`, `tail`, `wc`, `cut` yang cuma baca input dari stdin dan print hasil ke stdout. Mereka gak bisa write file, gak bisa fork process, gak bisa delete apa-apa. Meminta approval buat setiap penggunaan command-command ini itu overkill yang bikin workflow jadi lambat.

Safe bins ini adalah daftar perintah yang OpenClaw anggap sudah pre-approved karena karakteristiknya yang terbukti aman. Mereka dapat dijalankan tanpa gate approval, yang mengurangi gesekan di alur kerja tanpa mengorbankan keamanan.

```json5
{
  tools: {
    exec: {
      safeBins: ["cut", "uniq", "head", "tail", "tr", "wc"],
      safeBinTrustedDirs: ["/bin", "/usr/bin", "/opt/homebrew/bin"],
      safeBinProfiles: {
        grep: {
          minPositional: 0,
          maxPositional: 0,
          allowedValueFlags: ["-e", "--regexp"],
          deniedFlags: ["-f", "--file", "-r", "-R"]
        }
      }
    },
  },
}
```

Characteristic safe bins:
- **Input only**: Stream processing dari stdin, gak baca file operand
- **No side effects**: Gak bisa modify filesystem atau execute subcommand
- **Literal token**: Gak ada globbing atau variable expansion
- **Flag validation**: Restricted argument profile

`grep` adalah tool yang powerful yang bisa menjadi berbahaya dalam beberapa mode. Jadi OpenClaw tidak secara otomatis menyertakan `grep` di safe bins, tetapi memberikan profile yang secara eksplisit mengizinkan penggunaan dasar.

Default safe bins: `cut`, `uniq`, `head`, `tail`, `tr`, `wc`. Command kayak `grep` dan `sort` butuh explicit configuration.

> **Pro Tip:** Mulai saja dengan safe bins default. Jika ada perintah yang sangat sering kamu gunakan dan kamu yakin aman dalam pola penggunaan kamu, baru tambahkan ke safe bins dengan restriction profile yang sesuai. Jangan membuat safe bins terlalu permisif di awal, karena hampir tidak mungkin kamu memprediksi semua edge case yang bisa membuat perintah "aman" menjadi berbahaya.


---

## Pattern 5: Approval Forwarding

### Approve dari Channel Mana Aja

Engineer gak selalu di channel yang sama pas approval request masuk. Mungkin request datang dari Discord (tempat interactive dev chat), tapi engineer yang on-call lagi di Slack (tempat incident channel). Atau engineer lagi liburan dan cuma pantau Telegram personal-nya. Approval forwarding kasih flexibility buat route request ke channel yang tepat.

```json5
{
  approvals: {
    exec: {
      enabled: true,
      mode: "session",  // "session" | "targets" | "both"
      agentFilter: ["main"],
      sessionFilter: ["slack"],
      targets: [
        { channel: "slack", to: "U12345678" },
        { channel: "telegram", to: "123456789" },
      ],
    },
  },
}
```

Tiga mode forwarding yang worth dipahami:

**`session`**: Approval di-request dari session channel yang sama. Kalau request datang dari Slack, approval juga lewat Slack.

**`targets`**: Approval selalu di-forward ke target spesifik, regardless dari mana request datang. Useful kalau kamu pengen centralized approval di satu channel.

**`both`**: Request di-forward ke session channel dan targets. Multiple people bisa approve, first-approve wins.

### Approval Command

```
/approve <id> allow-once    # Approve single execution
/approve <id> allow-always  # Add ke allowlist dan approve
/approve <id> deny          # Block execution
```

`allow-always`: approve request ini dan tambahkan pattern ke allowlist untuk request di masa depan.

Channel seperti Slack, Discord, dan Matrix dapat berfungsi sebagai native approval client dengan interactive button, tapi `/approve` command masih berfungsi sebagai fallback.

> **Contoh Nyata:** Tim distributed dengan engineer di 3 timezone. Mereka setup approval forwarding dengan targets `slack #oncall` dan `telegram @oncall-backup`. Pas primary oncall engineer sleep (timezone-nya udah malem), approval otomatis di-forward ke telegram backup yang awake. Gak ada delay berjam-jam nunggu primary bangun, gak ada approval yang stuck unanswered. Coverage 24/7 tanpa nambah headcount.

---

## Pattern 6: Async Approval Workflow

### Non-Blocking Approval Buat Action Gak Urgent

Tidak semua action memerlukan respons seketika. Agent yang sedang melakukan task kompleks mungkin memerlukan approval di tengah jalan, tapi tidak perlu memblokir seluruh task sambil menunggu approval datang. Async approval membuat workflow yang lebih efisien: agent submit request, melanjutkan pekerjaan lain, dan melanjutkan task ketika approval datang.

```json5
{
  async_approvals: {
    urgent: {
      timeout: "5m",
      channels: ["slack_dm", "pagerduty"],
      escalation: "Kalau gak ada response dalam 5m, page secondary on-call"
    },
    
    normal: {
      timeout: "4h",
      channels: ["slack_channel", "email"],
      behavior: |
        1. Submit approval request
        2. Lanjutin kerjaan lain yang gak dependent
        3. Resume action pas approved
        4. Kalau di-deny, log alasan dan adjust approach
    },
    
    batch: {
      collect_window: "30m",
      channels: ["slack_channel"],
      behavior: |
        Group approval request yang similar:
        "5 config change menunggu approval:
         1. Update redis.maxmemory di cache-01
         2. Update redis.maxmemory di cache-02
         3. Update redis.maxmemory di cache-03
         4. Update nginx.worker_connections di proxy-01
         5. Update nginx.worker_connections di proxy-02
         
         [Approve All] [Approve Selected] [Deny All]"
    }
  }
}
```

Tiga tier async approval:

**Urgent**: Timeout pendek (5 menit), multiple channel (Slack DM plus pagerduty), ada escalation kalau gak ada response. Buat action yang time-sensitive.

**Normal**: Timeout panjang (4 jam), channel yang kurang mengganggu (Slack channel plus email), no escalation. Untuk action yang bisa ditunggu tanpa kegentingan.

**Batch**: Collection window 30 menit, grouped approval UI. Untuk action yang repetitive dan serupa.

> **Contoh Nyata:** Agent yang handle rolling config update buat 50 cache server. Tanpa batch, dia kirim 50 approval request, engineer harus klik 50 kali. Dengan batch, request di-group, muncul 1 UI dengan option "Approve All", "Approve Selected", atau "Deny All". Engineer review sekali, click sekali, selesai. Waktu yang tersimpan: dari 15 menit jadi 30 detik. Dan yang lebih penting: engineer memiliki mental bandwidth untuk benar-benar review 50 perubahan itu, bukan klik approve terus-menerus tanpa membaca.

---

## Pattern 7: Standing Order dengan Approval Gate

### Program Agent Otonom dengan Oversight Manusia

Standing order adalah cara untuk mendefinisikan program agent yang otonom dengan gate approval bawaan di titik kritis.

```json5
{
  standingOrders: {
    autoDeploy: {
      enabled: true,
      description: "Auto-deploy approved change ke staging",
      steps: [
        {
          tool: "git",
          args: ["commit", "-m", "{{message}}"],
          approvals: {
            required: true,
            criteria: "contains 'breaking' || contains 'security'",
            channels: ["slack"],
            approvers: ["U12345678"]
          }
        },
        {
          tool: "kubectl",
          args: ["apply", "-f", "{{artifact}}"],
          approvals: {
            required: true,
            criteria: "namespace == 'production'",
            channels: ["slack_dm"],
            approvers: ["U87654321"]
          }
        }
      ]
    }
  }
}
```

Yang menarik dari pattern ini: approval criteria itu conditional. Commit yang contain 'breaking' atau 'security' butuh approval, commit biasa auto-approve. Deploy ke production butuh approval dari senior engineer, deploy ke staging auto-approve.

Ini bikin standing order powerful. Kamu bisa define workflow yang automated di 80% kasus, dengan safety net di 20% kasus yang critical.

---

## Pattern 8: Progressive Approval di Onboarding

### Build Trust Secara Gradual

Setup awal adalah moment kritis untuk security posture. Jika dari awal kamu mengatur policy yang terlalu permisif, sulit untuk mengetatnya nanti. Jika terlalu restriktif, engineer menjadi frustrasi dan mulai menonaktifkan safeguard. Onboarding wizard OpenClaw menggunakan pendekatan progresif: mulai dari default yang restriktif, menanyakan preferensi user dengan penjelasan yang jelas, dan menyesuaikan secara bertahap berdasarkan pola penggunaan yang sebenarnya.

```
Welcome to OpenClaw! Mari set up security preferences kamu.

1. Exec Approval Mode:
   [ ] deny - Block semua host execution
   [x] allowlist - Allow cuma command yang approved
   [ ] full - Allow semua (tidak direkomendasi)

2. Safe Bin Selection:
   - [x] cut, uniq, head, tail, tr, wc (default)
   - [ ] grep, sort (butuh approval)

3. Elevated Mode Access:
   Allow elevated mode dari:
   - [x] Discord: user-id-123
   - [x] Slack: U456
   - [ ] WhatsApp: +15555550123

[Continue] [Back] [Cancel]
```

Setelah setup, wizard memberikan notice untuk setting yang berisiko:

```
⚠️ Security Notice: Elevated Mode Access

Kamu enable elevated mode buat Discord. Ini bikin agent bisa:
- Run command di luar sandbox
- Skip approval prompt kalau set ke 'full'

Mau proceed? [Yes] [No] [Learn More]
```

Notice seperti ini membuat user sadar akan implikasi dari pilihan mereka. Banyak security disaster terjadi bukan karena user jahat, tapi karena user tidak memahami konsekuensi dari setting yang mereka pilih. Penjelasan yang jelas plus confirmation prompt mencegah banyak kesalahan.

> **Pro Tip:** Untuk tim baru yang baru mengadopsi OpenClaw, mulailah dengan preset yang strict (deny default, minimal safe bins, elevated disabled). Setelah 2-4 minggu, review pola penggunaan yang sebenarnya, identifikasi friction point yang legitimate, dan longgar setting secara selektif. Pendekatan ini adalah kebalikan dari "mulai permisif dan mengetat nanti" yang jarang benar-benar terjadi karena tim sudah terbiasa dengan workflow yang longgar.

---

## Anti-Pattern: Jangan Lakuin Ini!

### 1. Approval Fatigue

```json5
// SALAH: Semua butuh approval (nobody reads them anymore)
{
  tools: {
    exec: {
      security: "allowlist",
      ask: "always"  // Too much!
    }
  }
}

// BENAR: Risk-based approval level
{
  tools: {
    exec: {
      security: "allowlist",
      ask: "on-miss",
      // Cuma prompt buat non-safe operation
    }
  }
}
```

Approval fatigue adalah musuh yang sebenarnya dari human-in-the-loop. Ketika engineer harus approve 500 request per hari, dia berhenti membaca. Approve menjadi refleks, bukan decision yang disengaja. Ketika akhirnya ada request yang actually dangerous, dia approve juga karena sudah dalam mode autopilot. Semua safety benefit dari approval gate hilang.

Solusinya: klasifikasikan action ke tier berdasarkan risiko. Low-risk action (safe bins, read-only operation) auto-approve. Medium-risk memerlukan single approval. High-risk memerlukan multi-approval plus context. Distribusi yang sehat adalah mungkin 90% auto, 9% single approval, 1% multi-approval. Jika kamu menemukan dirimu menyetujui 50 request per hari, policy kamu mungkin terlalu restriktif di tempat yang salah.

> **Eits, Hati-hati!** Approval fatigue membuat approval system kamu lebih buruk daripada tidak ada approval sama sekali. Dengan tidak ada approval, kamu posisinya jelas. Dengan approval fatigue, kamu ada illusion of safety padahal secara efektif tidak ada human oversight. Illusion of safety lebih berbahaya daripada absence of safety.

### 2. Drive-By Approval

Pola buruk: approval button tanpa context. Engineer klik approve tanpa benar-benar tahu apa yang mereka setujui.

```json5
// SALAH: Approve button tanpa context
{
  message: "Approve action? [Yes/No]"
}

// BENAR: Full context buat informed decision
{
  message: |
    Action: {{action}}
    Impact: {{impact}}
    Risk: {{risk_level}}
    Evidence: {{supporting_data}}
    Alternative: {{alternatives}}
    [Approve] [Deny] [More Info]
}
```

Yang bagus dari approval message: dia memberikan cukup context untuk engineer membuat decision yang informed dalam hitungan detik. Action apa yang akan dijalankan? Impact-nya apa (downtime, perubahan data, biaya)? Risk level berapa? Ada evidence yang mendukung decision? Ada alternative yang mungkin lebih baik?

Compare approval yang kayak gini:
- "Approve restart database?" (bad)
- "Action: Restart production PostgreSQL. Impact: 10 min downtime, affect 500 active session. Risk: High. Evidence: Slow query dari past 2 hours (attached). Alternative: Wait sampai maintenance window jam 2 pagi. [Approve] [Deny] [Details]" (good)

Engineer yang approve option kedua punya context yang cukup buat make real decision. Yang approve option pertama basically coin flip.

### 3. Gak Ada Timeout di Approval

```json5
// SALAH: Approval request nunggu selamanya
{
  timeout: "never"  // Agent stuck indefinitely
}

// BENAR: Timeout dengan safe default
{
  timeout: "15m",
  on_timeout: "deny",  // Safe default: gak lakuin action
  notify: "Approval request expired. Re-request kalau masih needed."
}
```

Approval tanpa timeout membuat agent stuck selamanya jika approver tidak tersedia. Engineer liburan, network issue, channel bermasalah.

Solusinya: selalu atur timeout dengan safe default. Untuk kebanyakan kasus, safe default adalah deny (jangan jalankan action). Untuk beberapa edge case, safe default mungkin rollback. Jangan pernah safe default adalah allow.

---

## Checklist: Human-in-the-Loop Readiness

Sebelum deploy agent ke production dengan human-in-the-loop:

- [ ] Action ter-classified ke risk tier dengan approval level yang sesuai
- [ ] Decision presentation include situation, options, dan recommendation
- [ ] Approval request punya **timeout** dengan safe default (deny)
- [ ] **Progressive autonomy** mulai restrictive dan build trust over time
- [ ] Approval request include **enough context** buat informed decision
- [ ] **Batch approval** reduce fatigue buat action yang similar
- [ ] Trust **menurun** setelah incident (tidak hanya meningkat)
- [ ] Approval fatigue dimonitor (terlalu banyak = re-classify tier)
- [ ] Async approval support non-blocking workflow
- [ ] Setiap approval/denial ter-log dengan approver identity
- [ ] Break-glass (elevated mode) terdokumentasi dan dimonitor
- [ ] Emergency procedure di-test minimal quarterly

### Referensi Perintah Esensial

```bash
# Approval management
openclaw approvals get                           # dapat config sekarang
openclaw approvals set --stdin < policy.json     # set new policy
openclaw approvals allowlist add --pattern "..."  # tambah ke allowlist
openclaw approvals allowlist remove --id "..."   # hapus dari allowlist

# Session control
/exec host=auto security=allowlist ask=on-miss   # set session default
/exec                                             # cek setting session
/elevated on                                      # activate elevated
/elevated off                                     # deactivate elevated

# Approval response
/approve <id> allow-once                          # approve single
/approve <id> allow-always                        # approve plus allowlist
/approve <id> deny                                # block execution
```


---

## Key Takeaways

- **Keseimbangan adalah kunci**. Terlalu otonom itu berbahaya, terlalu restrictive bikin AI jadi useless. Sweet spot adalah right level of autonomy per action, bukan uniform setting
- **Risk-based approval tier**. Low-risk auto-approve (safe bins), medium-risk single approval, high-risk multi-approval plus context. Distribusi ideal: 90% auto, 9% single, 1% multi
- **Progressive autonomy** build trust secara gradual. Mulai dari restrictive default, loosen based on real usage pattern, tighten pas ada incident
- **Context is everything**. Approval request harus include enough information buat decision yang benar-benar informed, bukan cuma yes/no button
- **Avoid approval fatigue**. Timeout, batch, tiered approval bikin human tetap effective. Approval fatigue lebih buruk dari no approval
- **Async beat sync** buat action yang tidak urgent. Agent lanjutin kerjaan lain nunggu approval, bukan blocked indefinitely
- **Log everything**. Tiap approval dan denial ter-log dengan identity approver, buat audit plus continuous improvement
- **Break-glass hanya untuk emergency**. Elevated mode itu last resort, bukan shortcut untuk bypass approval yang mengganggu. Monitor penggunaannya dan atur normal flow kalau break-glass digunakan terlalu sering

---

