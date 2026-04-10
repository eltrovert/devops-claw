# Bab 15: Blast Radius, Seberapa Jauh Ledakannya Bisa Merembet

> *"Blast radius itu kayak jari-jari ledakan bom. Kamu gak selalu bisa cegah bom-nya meledak, tapi kamu bisa atur seberapa jauh ledakan itu merembet. Bedanya sama dunia nyata: di engineering, kamu bisa bikin ledakan yang cuma merusak satu ruangan, bukan satu kota."*

---

## Apa Sih Blast Radius Itu?

Istilah "blast radius" asalnya dari dunia safety engineering, literal tentang ledakan. Kalau ada bom meledak, radius-nya nentuin seberapa jauh damage yang terjadi. Di dunia software, konsep-nya sama tapi metaphorical: kalau ada yang salah di sistem kamu, seberapa luas damage yang terjadi sebelum sistem-nya bisa ditahan?

Buat agent AI, blast radius adalah **semua sistem yang bisa diakses agent selama satu session**. Setiap API yang bisa dia panggil, setiap database yang bisa dia query atau tulis, setiap pesan eksternal yang bisa dia kirim, setiap file yang bisa dia baca atau modify. Semua itu bikin total "jangkauan" damage kalau agent itu bermasalah, entah karena bug, misinterpretation, atau malicious input.

Yang penting buat dipahami: blast radius itu bukan tentang **kemungkinan** sesuatu salah, tapi tentang **seberapa buruk kalau salah**. Kamu mungkin punya agent dengan probabilitas gagal yang rendah, tapi kalau blast radius-nya massive (punya akses cluster-admin di production), probabilitas rendah kali impact massive masih bisa jadi risk yang gak acceptable.

Containment itu seni nge-minimize blast radius tanpa ngelumpuhin agent. Kamu gak bisa nge-nol-in blast radius, tapi kamu bisa bikin dia se-kecil yang feasible. Goal-nya: kalau terjadi yang terburuk, damage-nya terbatas di scope yang kecil.


---

## Rumus Blast Radius

Cara mental gampang buat mikirin blast radius:

```
Blast Radius = Sistem yang Diakses × Action yang Dibolehkan × Time Window
```

Tiga variabel, masing-masing bisa diperkecil secara independen. Bayangin kayak taman bermain:

- **Sistem yang Diakses**: Seberapa luas taman bermainnya
- **Action yang Dibolehkan**: Permainan apa yang boleh dimainkan
- **Time Window**: Berapa lama anak bisa main

Taman bermain yang luas tapi dengan permainan terbatas dan waktu singkat punya risk profile yang beda sama taman bermain kecil dengan permainan apa aja boleh dan waktu tak terbatas. Dua konfigurasi itu bisa punya blast radius effective yang sama, tapi dengan characteristic berbeda.

Containment yang efektif artinya nge-shrink semua tiga variabel secara simultan. Kamu batasin sistem yang agent bisa akses, kamu batasin action yang agent bisa lakuin, dan kamu batasin berapa lama session bisa jalan. Multiply tiga ini dan kamu dapet effective blast radius yang jauh lebih kecil dari max theoretical-nya.

> **Eits, Hati-hati!** Kalau ada satu variabel aja yang unbounded, total blast radius kamu juga effectively unbounded. Containment itu game multiplication, bukan addition. Lemah di satu dimensi bikin dimensi lain kurang berarti.

---

## Strategi 1: Execution Environment yang Ephemeral dan Isolated

### Satu Container Per Task

Kalau koki pake peralatan yang sama buat masak ikan dan masak dessert tanpa sterilisasi, dessert-nya bakal berasa ikan. Di software, ini disebut "cross-contamination" antar task atau session. Ephemeral execution environment solve masalah ini dengan bikin environment baru yang bersih setiap kali ada task baru.

Konkretnya: tiap kali agent mulai session baru, dia spawn container baru (atau MicroVM, atau VM, tergantung backend). Container itu berisi tools yang dia butuh, tapi zero state dari session sebelumnya. Setelah session selesai, container di-destroy sepenuhnya. Gak ada residue yang bisa mempengaruhi session berikutnya.

```json5
// OpenClaw sandbox configuration untuk isolation
{
  sandbox: {
    mode: "all",      // Blok akses ke semua branch buat security maksimal
    scope: "session", // Isolate tiap session
    backend: "docker", // Container-based isolation
    
    workspaceAccess: false,  // Gak ada akses file di luar workspace
    
    runtime: "docker",
    policy: "strict"
  }
}
```

Kenapa ini penting? State yang persistent antar session itu attack vector yang subtle. Agent di session 1 nge-write file ke `/tmp`, agent di session 2 read file itu dan misinterpret isinya sebagai data baru. Dengan ephemeral environment, vector ini close sepenuhnya.

### Perbandingan Teknologi Isolation

Gak semua isolation sama kuat. Ada trade-off antara security, overhead, dan kompatibilitas. Ini yang umum dipakai di industri:

- **Firecracker MicroVM** — Terkuat, kernel dedicated / ~125ms startup, ~5MB memory / Production, untrusted code
- **Kata Containers** — Kuat, VM-level di K8s / ~1s startup, ~30MB memory / Workload Kubernetes
- **gVisor** — Baik, user-space kernel / ~50ms startup, ~10MB memory / General purpose
- **Hardened Container** — Baseline, shared kernel / Minimal / Trusted code aja
- **No isolation** — Gak ada / Gak ada / **Gak pernah buat production**

Firecracker itu yang terkuat. Tiap microVM punya kernel sendiri, jadi kalau attacker escape sandbox, mereka cuma escape ke microVM kernel yang independent dari host kernel. Trade-off-nya: startup lebih lambat dan overhead memory lebih tinggi dibanding plain container. Layak buat workload yang butuh trust boundary yang kuat.

gVisor itu middle ground yang bagus. Dia implement sebagian besar syscall di user-space, jadi kernel real gak di-touch langsung sama container. Startup cepat (50ms), overhead rendah, tapi isolation jauh lebih kuat dari plain container. Kebanyakan kasus produksi bisa pakai gVisor tanpa masalah.

Plain hardened container adalah baseline minimum buat production. Shared kernel dengan host, jadi kalau ada kernel exploit, container bisa escape. Tapi kalau code yang running itu trusted, hardened container udah cukup.

```json5
// OpenClaw sandbox security configuration
{
  sandbox: {
    mode: "non-main",
    backend: "docker",
    workspaceAccess: false,
    
    security: {
      run_as_non_root: true,
      read_only_root_filesystem: true
    },
    
    capabilities: {
      drop: ["ALL"],
      add: []
    }
  }
}
```

Pattern `drop: ["ALL"]` plus empty `add` itu prinsip minimum capability. Container spawn tanpa Linux capability sama sekali. Kalau nanti ternyata butuh capability tertentu, kamu explicit add satu per satu. Default deny, explicit allow.


---

## Strategi 2: Network Segmentation

### Mikrosegmentasi untuk Traffic Agent

Container isolation handle filesystem dan process boundary, tapi network itu cerita terpisah. Bahkan agent yang isolated di container bisa tetep ngomong ke internet, ke database, ke API eksternal. Kalau network-nya gak di-kontrol, blast radius meluas lewat jaringan meskipun container-nya ketat.

Bayangin gedung kantor dengan banyak ruangan. Setiap ruangan punya pintu yang cuma bisa dibuka dengan card access khusus. Kamu bisa masuk ruangan A cuma kalau card kamu authorized buat itu. Ini disebut micro-segmentation di networking.

```
┌──────────────────────────────────────────────────┐
│                 Agent Network                     │
│                                                  │
│  ┌──────────┐     ┌──────────┐                  │
│  │ OpenClaw │───▶│  Proxy   │                   │
│  │  Agent   │     │ Gateway  │                   │
│  └──────────┘     └────┬─────┘                  │
│                        │                         │
│         ┌──────────────┼──────────────┐          │
│         │              │              │          │
│    ┌────▼────┐   ┌────▼────┐   ┌────▼────┐     │
│    │  LLM    │   │  K8s    │   │ Metrics │     │
│    │  API    │   │  API    │   │  API    │     │
│    │(egress) │   │(read)   │   │(read)   │     │
│    └─────────┘   └─────────┘   └─────────┘     │
│                                                  │
│    ✗ Database    ✗ SSH     ✗ Internet            │
│    (blocked)   (blocked) (blocked)               │
└──────────────────────────────────────────────────┘
```

Agent boleh akses LLM API, K8s API (read-only), dan metrics API. Semua traffic lain explicitly blocked. Database direct access? Blocked, meski dia running di host yang sama. SSH? Blocked. General internet? Blocked. Ini default-deny network policy.

> **Eits, Hati-hati!** Jangan pernah rely pada agent buat self-limit network access. "Ah, agent gak akan coba akses database kok, kan ada instruksi di system prompt". Salah. Enforce network restriction di infrastructure level (firewall rule, service mesh policy, network policy di Kubernetes), bukan di code level. Kalau agent bisa bypass instruction-nya sendiri, attacker juga bisa.

```json5
// OpenClaw sandbox network configuration
{
  sandbox: {
    network: {
      egress: [
        {
          pattern: "api.anthropic.com:443",
          allow: true,
          comment: "Izinkan Anthropic API call"
        },
        {
          pattern: "api.openai.com:443",
          allow: true,
          comment: "Izinkan OpenAI API call"
        },
        
        {
          pattern: "prometheus.internal:9090",
          allow: true,
          comment: "Izinkan Prometheus metric"
        },
        {
          pattern: "grafana.internal:3000",
          allow: true,
          comment: "Izinkan Grafana dashboard"
        },
        
        {
          pattern: "kubernetes.default.svc:443",
          allow: true,
          comment: "Izinkan K8s API access"
        },
        
        {
          pattern: "registry.npmjs.org:443",
          allow: true,
          comment: "Izinkan NPM registry"
        },
        
        {
          pattern: "api.github.com:443",
          allow: true,
          comment: "Izinkan GitHub API"
        },
        {
          pattern: "raw.githubusercontent.com:443",
          allow: true,
          comment: "Izinkan GitHub raw content"
        },
        
        {
          pattern: "*:*",
          allow: false,
          comment: "Tolak semua egress lainnya"
        }
      ],
    },
  }
}
```

Perhatiin urutan rules: explicit allow yang spesifik dulu, terakhir catch-all deny. Ini pattern standard firewall: spesifik menang, fallback adalah deny. Kalau agent coba konek ke `example.com`, dia match `*:*` yang deny, jadi di-block.

Plus, perhatiin bahwa tiap rule punya `comment`. Ini penting buat maintenance. Enam bulan dari sekarang, engineer yang review policy ini bakal nanya "kenapa kita allow `raw.githubusercontent.com`?". Comment jawab pertanyaan itu tanpa perlu googling atau git-blame.

---

## Strategi 3: Circuit Breaker dan Kill Switch

### Automatic Circuit Breaker

Istilah "circuit breaker" asalnya dari dunia listrik. Kalau ada short circuit atau beban berlebih, circuit breaker otomatis putus aliran listrik sebelum ada damage atau kebakaran. Di software, konsep-nya sama: kalau agent mulai berperilaku abnormal (error rate tinggi, latency spike, rate action yang gak normal), sistem otomatis putus agent sebelum damage membesar.

```json5
// OpenClaw exec tool configuration buat safety
{
  tools: {
    exec: {
      notifyOnExit: true,
      approvalRunningNoticeMs: 10000,
      
      security: "allowlist",
      ask: "on-miss",
      
      timeout: 1800  // 30 menit timeout
    },
    
    allowed: ["web_fetch", "skills", "exec"],
    denied: ["delete"],
    
    elevated: {
      enabled: true,
      allowFrom: {
        discord: ["user-id-123"]
      }
    }
  }
}
```

Tiga jenis circuit breaker yang penting:

**Rate limiting**: Maximum action per unit waktu. Kalau agent tiba-tiba execute 100 command dalam 1 menit (pas biasanya dia execute 5-10 per menit), itu signal abnormal yang layak diselidiki. Rate limit bikin agent slow down atau stop sampai ada review manusia.

**Error rate limiting**: Maximum error rate yang tolerable. Kalau dari 10 command terakhir, 8 gagal, sistem asumsi ada sesuatu yang salah (entah agent error, target sistem down, atau configuration issue) dan circuit break sebelum cascade failure.

**Cost limiting**: Maximum resource atau cost per session. Buat workload yang bisa spending real money (kayak agent yang deploy AWS resource), cost limit prevent runaway spending karena agent yang misbehave.

Keempat kombinasi ini bikin safety net multi-dimensional. Satu sensor gagal, yang lain masih catch.

### External Kill Switch

Circuit breaker itu automatic. Tapi kadang-kadang kamu butuh manual override: button yang bisa ditekan human buat langsung stop agent, no matter what. Ini disebut kill switch, dan harus selalu ada di setup production.

```json5
// OpenClaw elevated mode menyediakan controlled sandbox escape
{
  elevated: {
    enabled: true,
    allowFrom: {
      discord: ["user-id-123"]
    },
    
    commands: {
      on: "Aktifkan elevated mode",
      off: "Non-aktifkan elevated mode",
      full: "Skip semua approval (butuh akses)",
      ask: "Butuh approval untuk elevated command"
    },
  }
}
```

Kill switch design principle yang penting:

**Gak bisa di-disable sama agent itu sendiri**. Kill switch yang proper ada di luar agent's control sphere: signal dari external system, environment variable yang di-set di orchestrator, atau endpoint di gateway yang agent gak punya credential buat akses.

**Simple banget buat di-trigger**. Kalau kill switch butuh 10 langkah sebelum bisa dipake, engineer di panic mode gak bakal bisa pake-nya dengan benar. Best case: one command, one button, one API call. Yang penting: obvious, documented, dan tested rutin.

**Terpisah dari normal operational path**. Kill switch gak boleh rely sama sistem yang mungkin ikut rusak. Kalau kill switch kamu depends on gateway yang lagi overloaded, kamu gak bisa kill agent pas paling kamu butuh. Make sure kill switch punya independent infrastructure yang still works even saat main system down.


---

## Strategi 4: Environment-Based Segmentation

### Akses Berperingkat Antar Environment

Setiap environment di sistem kamu punya characteristic risk yang berbeda. Development itu sandbox engineer, ngapain aja boleh, kesalahan gak impact siapa-siapa. Staging itu mirror production, lebih ketat tapi masih ada room buat experimentation. Production itu suci, setiap perubahan harus deliberate dan terverifikasi. Policy agent harus reflect perbedaan ini.

```
Development        Staging              Production
┌──────────┐      ┌──────────┐        ┌──────────┐
│ Full     │      │ Write    │        │ Read     │
│ Access   │      │ Access   │        │ Only     │
│          │      │ +        │        │ + Strong │
│ No       │      │ Approval │        │ Approval │
│ Approval │      │ Some     │        │ Full     │
│          │      │ Approval │        │ Approval │
│          │      │          │        │          │
│ Sandbox  │      │ gVisor   │        │ MicroVM  │
│ Isolation│      │ Isolation│        │ Isolation│
└──────────┘      └──────────┘        └──────────┘
```

Progression-nya jelas: semakin naik ke production, restriction semakin ketat dan isolation semakin kuat. Di development, agent bisa full access tanpa approval karena konsekuensi error itu minimal. Di staging, ada approval buat write operation, tapi read masih bebas. Di production, default read-only, dan semua write butuh multi-layer approval plus strong isolation.

> **Contoh Nyata:** E-commerce platform dengan setup 3-tier. Development agent bisa full CRUD ke dev database, spawn resource AWS sendiri, deploy ke staging. Staging agent bisa read semua dari production database (buat test dengan real data shape), write cuma ke staging tables. Production agent cuma bisa read production metrics dan logs, dan trigger deploy workflow yang butuh approval dari 2 engineer senior. Kalau ada bug di development agent, damage-nya limited ke dev env. Kalau ada bug di production agent, containment sudah sampai ke tingkat "maksimum 1 deploy per 15 menit dengan 2 human approval", jadi impact real-nya minimal.

```json5
// OpenClaw agent configuration buat berbagai environment
{
  agents: [
    {
      id: "main",
      tools: {
        exec: {
          security: "allowlist",
          ask: "on-miss",
          notifyOnExit: true,
          approvalRunningNoticeMs: 10000
        }
      },
      sandbox: {
        mode: "non-main",
        scope: "session",
        backend: "docker"
      }
    },
    
    {
      id: "development",
      tools: {
        exec: {
          security: "allowlist",
          ask: "on-miss"
        }
      },
      sandbox: {
        mode: "non-main",
        backend: "docker"
      }
    },
    
    {
      id: "production",
      tools: {
        exec: {
          security: "allowlist",
          ask: "always",
          notifyOnExit: true,
          approvalRunningNoticeMs: 5000
        }
      },
      sandbox: {
        mode: "all",
        scope: "shared",
        backend: "docker"
      }
    }
  ]
}
```

Compare config buat development dan production. Development pake `ask: "on-miss"`, production pake `ask: "always"`. Notification threshold juga beda: 10 detik di development, 5 detik di production.

---

## Strategi 5: Penilaian Dampak Sebelum Tindakan

### Analisis Dampak Sebelum Eksekusi

Strategi sebelumnya itu semua reactive: mereka berlaku setelah action di-execute. Strategy 5 itu proactive: sistem analyze potential impact sebelum action di-execute, dan decide apakah action itu acceptable atau butuh additional approval.

Bayangin dokter bedah yang selalu review CT scan sebelum operasi. Ada assessment phase yang help dokter understand resiko sebelum insisi pertama. Agent layak treatment yang sama.

```json5
// OpenClaw skill: blast-radius-analyzer
{
  name: "blast-radius-analyzer",
  description: "Assess potential impact before executing any action",

  pre_action_check: {
    analyze: |
      Sebelum execute action apa pun, determine:
      
      1. DIRECT IMPACT: Resource apa yang bakal di-modify?
      2. INDIRECT IMPACT: Apa yang depend sama resource itu?
      3. USER IMPACT: Berapa user/request yang terpengaruh?
      4. REVERSIBILITY: Bisa di-undo gak?
      5. TIME SENSITIVITY: Seberapa cepat damage bisa spread?
  },
  
  classification: {
    low: {
      criteria: [
        "Cuma affect non-production resource",
        "Fully reversible dalam 5 menit",
        "Zero user-facing impact"
      ],
      action: "proceed"
    },
    medium: {
      criteria: [
        "Affect staging atau limited production scope",
        "Reversible dalam 30 menit",
        "Minimal user impact"
      ],
      action: "require_single_approval"
    },
    high: {
      criteria: [
        "Affect production resource",
        "Partially reversible",
        "Potential user impact"
      ],
      action: "require_approval_with_rollback_plan"
    },
    critical: {
      criteria: [
        "Affect critical production system",
        "Difficult to reverse",
        "Significant user atau data impact"
      ],
      action: "require_two_approvals_and_maintenance_window"
    }
  }
}
```

Classification system ini powerful karena bikin policy jadi adaptive. Agent yang lagi lakuin task low-risk (misal read log) jalan tanpa friction. Agent yang lagi lakuin task critical (misal drop database table) di-block sampai ada multi-approval. Policy yang sama di-apply ke kedua kasus, tapi response-nya graduated sesuai risk.

Yang tricky dari assessment: quality-nya tergantung sama gimana kamu define criteria. Kalau definition kamu terlalu loose, action berbahaya bisa lolos ke lower tier. Kalau terlalu ketat, agent jadi useless karena semua action di-classify sebagai high-risk. Refinement iteratif based on real incident data bikin assessment improve over time.

---

## Incident Response: Kalau Containment Gagal

### Damage Assessment Protocol

Meskipun semua containment kamu sempurna, sometimes shit happens. Agent escape sandbox, network policy punya gap, human approval di-trick, atau ada zero-day di isolation technology. Kalau itu terjadi, kamu butuh incident response protocol yang clear.

```json5
// OpenClaw exec approval process buat risk management
{
  exec_approvals: {
    defaults: {
      security: "deny",
      ask: "on-miss",
      askFallback: "deny",
      autoAllowSkills: false
    },
    
    agents: {
      main: {
        security: "allowlist",
        ask: "on-miss",
        askFallback: "deny",
        autoAllowSkills: true,
        allowlist: [
          {
            "id": "B0C8C0B3-2C2D-4F8A-9A3C-5A4B3C2D1E0F",
            "pattern": "~/Projects/**/bin/rg",
            "lastUsedAt": 1737150000000,
            "lastUsedCommand": "rg -n TODO",
            "lastResolvedPath": "/Users/user/Projects/.../bin/rg"
          }
        ]
      }
    },
    
    workflow: [
      {
        request_approval: "Kalau security=allowlist dan command gak di allowlist"
      },
      {
        forward_to_channel: "Kirim approval request ke channel designated"
      },
      {
        wait_for_response: "Tunggu keputusan allow/deny"
      },
      {
        execute_or_block: "Run command yang di-approve, block yang di-deny"
      }
    ]
  }
}
```

Key element dari incident response protocol:

**Immediate isolation**: Pas incident terdeteksi, langsung isolate affected agent dari network dan data source.

**Damage scoping**: Determine apa yang udah dilakukan agent sebelum incident terdeteksi. Check audit log, action history, resource state. Penting buat rollback decision.

**Rollback plan**: Berdasarkan damage scope, determine apakah rollback feasible. Kadang full rollback, kadang partial, kadang no rollback.

**Root cause investigation**: Pas dust udah settle, investigate kenapa containment gagal. Bug di sandbox? Policy gap? Human error? Fix root cause supaya insiden sejenis gak ulang.

**Communication**: Inform stakeholder sesuai severity. Transparency membangun kepercayaan, penyembunyian merusaknya.

---

## Anti-Pattern

### 1. Percaya Agent Buat Self-Limit

Pola paling berbahaya: rely sama agent untuk batesin diri sendiri lewat system prompt. "Kamu agent yang hati-hati, jangan lakuin hal yang berbahaya". Ini lawan yang konkret, bukan nasihat.

```json5
// SALAH: "Agent punya instruksi buat hati-hati"
{
  safety: {
    method: "system_prompt_says_be_careful"
  }
}

// BENAR: Technical control enforce limit
{
  safety: {
    method: "infrastructure_enforced",
    controls: [
      "network_policy",
      "rbac",
      "sandbox",
      "circuit_breaker"
    ]
  }
}
```

System prompt itu instruction yang agent **coba** ikutin, bukan hard constraint. Agent bisa misinterpret, hallucinate, atau kena prompt injection yang override instruction-nya. Technical control (network policy, RBAC, sandbox, circuit breaker) di-enforce di level infrastructure yang agent gak punya control atas. Ini bedanya "aturan" dan "hukum": aturan bisa dilanggar, hukum tidak.

### 2. Shared Execution Context

Pola bad kedua: ngerun agent langsung di host tanpa sandbox. Alasannya biasanya performance ("sandbox terlalu lambat") atau convenience ("lebih gampang debug kalau langsung di host").

```json5
// SALAH: Agent run command langsung di host tanpa isolation
{
  agent: {
    exec: {
      host: "gateway",           // Run di host tanpa sandbox
      security: "full",         // Gak ada restriction
      ask: "off"                // Gak ada approval required
    }
  }
}

// BENAR: Sandboxed execution dengan isolation proper
{
  agent: {
    exec: {
      host: "auto",
      security: "allowlist",
      ask: "on-miss",
      notifyOnExit: true
    },
    
    sandbox: {
      mode: "non-main",
      scope: "session",
      backend: "docker"
    }
  }
}
```

Performance argument itu usually specious. Modern container overhead itu minimal (sub-second startup), dan kalau kamu beneran bottlenecked, ada optimization yang bisa di-apply. Convenience argument valid cuma buat local development, bukan production.

> **Eits, Hati-hati!** Kalau kamu nemuin config dengan `host: "gateway"` atau sandbox disabled di production, treat itu sebagai security incident. Investigate kenapa config ini ada dan fix secepatnya. Satu exception di policy containment bisa bikin seluruh effort containment kamu useless.

---

## Checklist: Blast Radius Containment

Sebelum deploy agent ke production, verify:

- [ ] Setiap task agent run di **isolated environment** (container, MicroVM, atau gVisor)
- [ ] Network access **explicitly allowlisted**, bukan wildcard allow
- [ ] **Circuit breaker** limit rate action, error, cost, dan scope
- [ ] Ada **external kill switch** yang agent gak bisa disable
- [ ] **Environment segmentation** kasih tingkat akses (dev, staging, prod)
- [ ] **Pre-action impact analysis** classify risk sebelum execution
- [ ] Ada **incident response protocol** kalau containment gagal
- [ ] Agent pake **database connection terpisah** dari application
- [ ] Security di-enforce **technically**, bukan lewat prompt instruction
- [ ] Lifetime session **terbatas** (maksimum 4 jam atau less)
- [ ] Audit log ter-enable dan gak bisa di-modify sama agent
- [ ] Rollback procedure ter-test dan documented


---

## Key Takeaways

- **Blast radius adalah max damage potential**, bukan probabilitas damage. Low probability kali high impact masih bisa jadi unacceptable risk
- **Rumus multiplication**: blast radius = systems × actions × time. Shrink semua tiga, gak cuma salah satu
- **Isolation ephemeral** mencegah cross-contamination. Container yang spawn per session dan destroy setelah itu hilangin state attack vector
- **Network segmentation default-deny**. Agent cuma bisa akses endpoint yang explicit di-allowlist, semua lainnya diblokir
- **Circuit breaker** berhenti otomatis pada perilaku abnormal. Rate limit, error rate, dan cost limit kasih safety net multi-dimensional
- **Kill switch eksternal** yang agent gak bisa nonaktifkan itu wajib. Simple, accessible, dan independent dari main system
- **Environment segmentation** kasih respons risiko berperingkat. Development permissive, production strict, dengan progression di antara
- **Pre-action assessment** klasifikasi risk sebelum damage terjadi. Low-risk lolos tanpa hambatan, high-risk butuh multiple approval
- **Jangan pernah percaya agent buat self-limit**. Enforce technically di level infrastructure, bukan lewat prompt

---

