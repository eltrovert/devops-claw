# Bab 14: Permission Model, Jangan Kasih Kunci Gudang ke Sembarang Orang

> *"Prinsip least privilege itu kayak aturan lemari es di kost: orang cuma boleh ambil yang dia masak sendiri. Kedengerannya kejam, tapi kalau gak gitu, satu aja yang ambil enak, yang lain kelaparan seminggu."*

---

## Epidemi Izin Berlebihan

Ada statistik yang bikin merinding: di kebanyakan organisasi, **akun non-human (termasuk agent AI, service account, bot) melebihi akun human dengan rasio 90:1**. Sembilan puluh akun machine buat setiap satu akun manusia. Dan mayoritas akun machine ini punya izin yang jauh lebih luas dari yang sebenarnya mereka butuhin, karena pengaturannya mudah dipikir "ah, kasih admin aja biar gak error".

Pola ini bukan cuma masalah hygiene. Ini root cause banyak incident security yang nyata. Karena akun machine jarang di-audit, credential-nya sering long-lived, dan izin-nya gak pernah di-review, mereka jadi attack surface yang gemuk. Pas ada breach, attacker jarang masuk lewat login manusia. Mereka masuk lewat service account yang kelupaan, bot yang punya akses lebih dari kebutuhan, atau agent yang warisin kredensial dari engineer yang deploy-nya.

Aku pernah lihat kasus di mana tim security audit nemu monitoring agent mereka punya `cluster-admin` di Kubernetes. Alasan? "Pas setup, tim lupa scope-nya, dan sekali jalan gak ada yang berani ganggu karena takut merusak monitoring". Udah 2 tahun lebih agent ini running dengan cluster-admin untuk task yang sebenarnya cuma butuh read access ke 3 namespace. Kalau agent ini compromised, attacker bisa delete seluruh cluster. Dan gak ada alert, gak ada audit yang bakal nangkep karena toh dari perspektif sistem, agent itu memang punya authority segitu.

Least privilege bukan cuma buzzword security. Ini alat yang secara konkret turunkan blast radius kalau ada yang salah. Agent yang cuma punya read access gak bisa delete apa-apa, meskipun dia misinterpret instruction atau compromised. Setiap batasan itu compensating control yang membuat worst case scenario jauh lebih survivable.


---

## Masalah Fundamental: Privilege Inheritance

Ada satu pola yang sering bikin privilege explosion: agent mewarisi credential dari human yang deploy atau run dia. Polanya kayak gini:

```
Human launches agent dengan credential-nya sendiri:
├── Human punya: cluster-admin, database superuser, AWS AdministratorAccess
├── Agent mewarisi: cluster-admin, database superuser, AWS AdministratorAccess  
├── Agent butuh: read pod di satu namespace, read log, query Prometheus
└── Agent risk surface: bisa delete seluruh infrastructure
```

Gap antara apa yang agent butuhin dan apa yang agent punya itu namanya **privilege gap**. Dan kebanyakan incident security agent AI terjadi di dalam gap ini. Engineer senior yang punya cluster-admin authority run agent buat task simple, agent mewarisi authority itu, dan tiba-tiba agent punya kekuatan yang jauh di atas yang seharusnya.

Bayangin kasus ini. Support agent diberi tugas: "Liat deployment yang gagal di staging dan bantu debug". Dari deskripsi ini, agent butuh apa? Read akses ke deployment di staging namespace, read akses ke pod log, mungkin read akses ke event. Itu aja. Tapi kalau agent ini inherit cluster-admin dari engineer yang deploy-nya, apa yang bisa dia lakuin kalau salah interpret instruction?

```bash
# Agent dengan cluster-admin bisa execute ini tanpa hambatan:
kubectl delete deployment production-app    # unintended production impact
DROP DATABASE production_db                  # karena salah nafsirin "staging"
rm -rf /var/www/*                             # ngira ini cache yang perlu dibersihin
```

Ini bukan hypothetical. Agent beneran bisa lakukan hal-hal kayak gini kalau dia punya authority dan salah interpret. Dan karena dia "legitimate" dari perspektif sistem (dia pakai credential yang valid), gak ada alert, gak ada block, gak ada gesekan untuk mencegahnya.

> **Contoh Nyata:** Tim support di startup SaaS ngasih akses database production ke bot Slack "biar bisa bantu jawab customer question lebih cepet". Bot punya read access, tapi authentication-nya pake engineer credential yang juga punya write access. Suatu hari, bot disuruh "hapus dummy data yang bikin confused di table customer". Bot interpret itu sebagai DELETE dari table customer (bukan table dummy), dan hapus 3000 record production. Rollback butuh 6 jam. Root cause: pewarisan privilege yang gak scoped.

---

## Empat Dimensi Least Privilege

OpenClaw approach least privilege di empat dimensi berbeda. Masing-masing handle aspek access control yang spesifik, dan bersama-sama mereka memberikan coverage yang comprehensive.

### 1. Channel Access Control

Dimensi pertama adalah kontrol siapa yang bisa ngobrol sama agent, lewat channel mana. Bayangin channel messaging kayak gedung kantor dengan lantai yang berbeda-beda: setiap lantai perlu izin terpisah buat masuk. Agent yang boleh handle support ticket di channel `#support` gak otomatis boleh baca message di channel `#finance-reports`.

```json5
// OpenClaw channel access configuration
{
  channels: {
    policies: {
      "dm": {
        requirePairing: true,
        allowedUsers: ["support-team@company.org"],
        requiredCapability: "operator.write"
      },
      
      "group": {
        requirePairing: false,
        denyGroups: ["#admin-alerts", "#finance-reports"],
        elevatedCapability: "operator.admin"
      }
    }
  },
  
  devicePairing: {
    requireApproval: true,
    autoApproveSameSubnet: false,
    tokenRotation: {
      interval: "24h",
      rotateOnReconnect: true
    }
  }
}
```

Setting yang penting di sini: `requirePairing: true` buat DM berarti agent gak bisa di-DM sembarangan. User harus explicit pair device mereka dulu, dan owner harus approve pairing request itu. Ini mirip kayak ngasih bluetooth device: gak ada yang bisa connect sebelum kamu explicit accept.

`denyGroups` buat group channel itu safety net tambahan. Meskipun agent technically punya akses ke group channel, channel sensitive kayak `#admin-alerts` atau `#finance-reports` permanent di-block.

`autoApproveSameSubnet: false` penting buat production. Di local development, auto-approve itu convenient. Tapi di production atau staging, setiap device baru yang mau pair harus explicit di-approve oleh admin. Ini prevent rogue device dari corporate network bisa seenaknya konek ke agent.

### 2. Tool Execution Control

Dimensi kedua: apa yang bisa agent execute setelah dia udah diconnect ke channel. Bayangin ini kayak dapur dengan resep terbatas. Koki bisa masuk ke dapur, tapi dia cuma bisa masak hidangan yang ada di resep yang disetujui. Mau masak di luar resep? Butuh approval khusus.

```json5
// OpenClaw tool policy configuration
{
  tools: {
    exec: {
      security: "allowlist",
      ask: "on-miss",
      safeBins: ["cut", "uniq", "head", "tail", "tr", "wc"],
      pathPrepend: ["~/bin", "/opt/oss/bin"],
    },
    
    allowed: ["web_fetch", "skills", "exec"],
    denied: ["delete"],
    
    timeouts: {
      exec: "30s",
      web_fetch: "60s",
      skills: "120s"
    }
  },
  
  approvals: {
    node: {
      "build-server-01": {
        "/usr/bin/docker": {
          allow: true,
          args: ["build", "push", "ps"],
          notify: true
        },
        "/usr/bin/kubectl": {
          allow: true,
          args: ["get", "describe", "logs"],
          ask: "on-first-use"
        }
      }
    },
    
    global: {
      requireApproval: true,
      approvalTimeout: "5m",
      recordSessions: true
    }
  }
}
```

Perhatiin pattern `allow` plus `args` plus `ask` di level per-node. Ini yang membuat policy menjadi granular. Agent boleh jalanin `/usr/bin/docker`, tapi cuma dengan argument `build`, `push`, atau `ps`. Dia gak bisa jalanin `docker exec` atau `docker system prune`. Mau jalanin `/usr/bin/kubectl`? Cuma dengan `get`, `describe`, atau `logs`. Dan ada `ask: "on-first-use"` yang butuh human approval di first call.

> **Eits, Hati-hati!** Mode `"allowlist"` selalu lebih aman dari `"denylist"`. Dengan allowlist, kamu explicit tentang apa yang aman: hanya yang ada di list yang boleh jalan. Dengan denylist, kamu harus inget semua kemungkinan berbahaya dan block-nya satu-satu. Attacker cuma perlu nemuin satu command berbahaya yang kamu lupa block, dan sistem kamu jebol. Allowlist aman dari gagal, denylist gagal terbuka. Selalu pilih aman dari gagal.

### 3. Gateway Authentication

Dimensi ketiga: siapa yang boleh connect ke gateway itu sendiri. Gateway adalah pintu masuk utama ke OpenClaw, dan setiap connection yang masuk lewat situ harus diverifikasi dulu identitasnya. OpenClaw support beberapa mode authentication, masing-masing dengan trade-off security yang berbeda.

```json5
// OpenClaw gateway authentication configuration
{
  gateway: {
    bind: "lan",
    
    auth: {
      mode: "token",
      
      token: {
        value: "your-secret-token-here",
        rotateOnStart: false
      },
      
      password: {
        value: "your-password-here",
        hash: "argon2id-hash-here"
      },
      
      trustedProxy: {
        // CRITICAL: Hanya tambahin IP proxy kamu di sini
        trustedProxies: ["10.0.0.1", "172.17.0.1"],
        userHeader: "x-forwarded-user",
        allowUsers: ["nick@example.com", "admin@company.org"],
        requiredHeaders: ["x-forwarded-proto", "x-forwarded-host"]
      }
    },
    
    controlUi: {
      allowedOrigins: ["https://app.openclaw.ai"],
      dangerouslyAllowHostHeaderOriginFallback: false
    },
    
    http: {
      securityHeaders: {
        strictTransportSecurity: "max-age=31536000; includeSubDomains",
        contentSecurityPolicy: "default-src 'self'",
        xContentTypeOptions: "nosniff",
        xFrameOptions: "DENY"
      }
    }
  }
}
```

Tiga mode authentication utama: **token** buat pengaturan sederhana, **password** buat interactive setup, dan **trustedProxy** buat pengaturan di mana reverse proxy (kayak nginx atau cloudflare) handle authentication.

Mode trusted proxy itu paling fleksibel buat production. Load balancer di depan OpenClaw yang bertanggung jawab authenticate user, dan OpenClaw cukup trust header dari proxy. Tapi ini juga mode paling dangerous kalau salah config. Kalau kamu gak explicit list IP proxy di `trustedProxies`, attacker bisa spoof header.

> **Pro Tip:** Kalau pakai trusted proxy mode, **selalu list IP proxy secara eksplisit**. Jangan pakai wildcard, jangan pake CIDR range yang kelebar. Satu misconfiguration di sini bisa jadi identity bypass total.

### 4. Node Device Management

Dimensi keempat: management device node, kayak iOS companion app, Android app, atau macOS desktop agent. Device ini butuh pairing eksplisit, punya capability terbatas, dan punya approval flow yang terpisah dari user desktop.

```bash
# List pending device pairing request
openclaw devices list

# Approve device pairing request
openclaw devices approve <requestId>

# Reject device pairing request
openclaw devices reject <requestId>

# Cek status device dan capabilities
openclaw devices status

# Rotate device credential
openclaw devices rotate <deviceId>

# Revoke akses device
openclaw devices revoke <deviceId>
```

Device node punya surface area yang berbeda dari desktop. iPhone bisa akses camera, microphone, location, photo library. macOS desktop bisa execute shell command. Tiap device type butuh policy yang berbeda sesuai capability uniknya.

```json5
// Node device configuration
{
  nodes: {
    allowCommands: [
      "canvas.*",
      "camera.*",
      "device.*",
      "location.*",
      "notifications.*",
      "system.which"
    ],
    
    denyCommands: [
      "system.run",
      "system.run.prepare"
    ],
    
    execApprovals: {
      "ios-device-01": {
        "/usr/bin/xcodebuild": {
          allow: true,
          args: ["-list", "-project"],
          notify: true
        }
      },
      
      "android-device-02": {
        "/usr/bin/adb": {
          allow: true,
          args: ["shell", "pm", "list", "packages"],
          ask: "on-first-use"
        }
      }
    }
  }
}
```

Perhatiin pattern `"system.run"` di denyCommands global, tapi boleh `/usr/bin/xcodebuild` di specific device. Ini principle least privilege in action: default-nya deny semua, then explicit allow exception yang beneran dibutuhin buat specific device dan specific use case.


---

## Pattern Implementasi

### Pattern 1: Channel-Based Access Control

Pattern pertama buat access control adalah routing berbasis channel. Idenya sederhana: tiap channel punya policy-nya sendiri, dan agent match channel policy itu sebelum process message. Kayak stiker nama di gedung kantor: setiap orang cuma bisa masuk ruangan yang sesuai dengan label di stiker mereka.

```bash
# Cek status channel access
openclaw channels status

# List semua channel dan policy-nya
openclaw channels list

# Cek apakah channel bisa di-akses
openclaw channels check --channel "#devops-alerts"

# View permission buat channel spesifik
openclaw channels permissions --channel "dm-user-123"
```

Pattern ini cocok buat setup di mana kamu punya banyak channel dengan sensitivity level berbeda. Channel `#general` bisa di-access sama agent apa pun, `#devops-alerts` cuma sama infra agent, `#finance-reports` di-deny buat semua agent. Kamu gak perlu tiap agent punya logic filtering sendiri, policy di-enforce di gateway level.

### Pattern 2: Device Identity Management

Pattern kedua: setiap device dikenalin dengan identitas unik, dan tiap identitas punya capability set yang spesifik. Kayak ID badge di kantor: badge iOS bisa akses camera tapi gak bisa run system command, badge macOS bisa run command tapi butuh approval buat yang sensitive.

```json5
// Device pairing configuration
{
  devicePairing: {
    pairingTimeout: "5m",
    requireApproval: true,
    autoApproveSameSubnet: false,
    
    capabilities: {
      ios: {
        allowed: [
          "canvas.*",
          "camera.*",
          "photos.latest",
          "device.status",
          "notifications.*"
        ],
        denied: [
          "system.run",
          "device.*.sensitive"
        ]
      },
      
      android: {
        allowed: [
          "canvas.*",
          "camera.*",
          "sms.send",
          "device.*",
          "calendar.*",
          "photos.*"
        ],
        denied: [
          "system.run"
        ]
      },
      
      macos: {
        allowed: [
          "system.run",
          "system.which",
          "system.notify",
          "canvas.*",
          "camera.*"
        ],
        requireApproval: [
          "system.run"
        ]
      }
    }
  }
}
```

Coba compare capability set buat iOS, Android, dan macOS. iOS sangat restricted (gak boleh `system.run` sama sekali), Android ada beberapa kelonggaran (bisa `sms.send`), macOS boleh run system command tapi butuh approval. Ini realistic karena memang platform-nya berbeda capability-nya, dan security model harus mencerminkan reality itu.

### Pattern 3: Multi-Agent Routing

Pattern ketiga: bikin multiple agent dengan scope berbeda, dan route message ke agent yang appropriate berdasarkan content atau channel. Kayak operator call center: ada operator buat support, ada operator buat sales, ada operator buat technical, dan calls di-route otomatis ke yang tepat.

```json5
// Agent routing dan access control
{
  agents: [
    {
      id: "support-agent",
      capabilities: [
        "operator.read",
        "operator.write"
      ],
      allowedChannels: [
        "#support-tickets",
        "dm-*"
      ],
      deniedChannels: [
        "#admin-alerts",
        "#finance-reports"
      ],
      tools: {
        exec: {
          security: "allowlist",
          allowed: ["/usr/bin/kubectl", "/usr/bin/docker"]
        }
      }
    },
    
    {
      id: "infra-agent",
      capabilities: [
        "operator.read",
        "operator.write",
        "operator.admin"
      ],
      allowedChannels: [
        "#devops-alerts",
        "#infra-monitoring"
      ],
      tools: {
        exec: {
          security: "allowlist",
          allowed: ["/usr/bin/kubectl", "/usr/bin/terraform", "/usr/bin/aws"]
        }
      }
    }
  ],
  
  routing: {
    dmRouting: {
      pattern: "support|help|ticket",
      target: "support-agent",
      confidence: 0.8
    },
    
    alertRouting: {
      pattern: "alert|critical|error|failed",
      target: "infra-agent",
      confidence: 0.9
    }
  }
}
```

Support agent cuma punya basic `operator.read` dan `operator.write`, plus akses ke `kubectl` dan `docker`. Dia gak bisa apa-apa di `terraform` atau `aws`. Infra agent punya `operator.admin` dan akses penuh ke tool infrastructure, tapi dia gak bisa message di `#support-tickets`. Separation of duties yang clean.

> **Contoh Nyata:** E-commerce platform dengan 5 agent berbeda: support-agent, infra-agent, deploy-agent, monitoring-agent, dan analytics-agent. Setiap agent punya scope minimal. Pas ada insiden di mana support-agent accidentally disuruh "restart semua service", dia gak bisa lakuin itu karena `service.*` gak ada di allowlist-nya. Task di-route ulang ke deploy-agent yang memang authorized. Gak ada damage, gak ada surprise, dan root cause analysis jadi cepet.

---

## Implementasi Platform-Specific

### AWS: IAM Policy Buat Agent

Buat agent yang interact sama AWS, kamu butuh IAM policy yang narrow focused. Kayak daftar tamu di pesta: tamu boleh liat barang-barang display, tapi gak boleh bawa pulang.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "OpenClawReadOnly",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ecs:Describe*",
        "ecs:List*",
        "rds:Describe*",
        "cloudwatch:GetMetricData",
        "cloudwatch:ListMetrics",
        "logs:GetLogEvents",
        "logs:FilterLogEvents",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-west-2"]
        }
      }
    },
    {
      "Sid": "DenyDangerous",
      "Effect": "Deny",
      "Action": [
        "iam:*",
        "organizations:*",
        "ec2:TerminateInstances",
        "rds:DeleteDBInstance",
        "s3:DeleteBucket",
        "lambda:DeleteFunction"
      ],
      "Resource": "*"
    }
  ]
}
```

Policy ini izinkan semua `Describe*`, `List*`, `Get*` action (operasi baca), plus spesifik `cloudwatch` dan `logs` buat monitoring. Explicit Deny buat operasi berbahaya: `iam:*` (mencegah privilege escalation), `organizations:*` (mencegah kerusakan akun), dan aksi destruktif kayak `Terminate`, `Delete`, `Destroy`. Condition `aws:RequestedRegion` juga membatasi scope ke 2 region saja, jadi kalau agent secara tidak sengaja coba akses region lain, dia auto-ditolak.

Yang penting: explicit Deny di IAM selalu menang di atas explicit Allow. Pola ini bagus buat high-assurance restriction.

### Kubernetes: RBAC Buat Agent

Buat agent yang interact sama Kubernetes, pakai ClusterRole yang narrow. Kayak stiker warna di gedung: biru buat area umum, kuning buat area khusus.

```json5
{
  apiVersion: "rbac.authorization.k8s.io/v1",
  kind: "ClusterRole",
  metadata: {
    name: "openclaw-agent-readonly"
  },
  rules: [
    {
      apiGroups: [""],
      resources: ["pods", "services", "endpoints", "events"],
      verbs: ["get", "list", "watch"]
    },
    {
      apiGroups: [""],
      resources: ["pods/log"],
      verbs: ["get"]
    },
    {
      apiGroups: ["apps"],
      resources: ["deployments", "replicasets", "statefulsets"],
      verbs: ["get", "list", "watch"]
    },
    {
      apiGroups: ["batch"],
      resources: ["jobs", "cronjobs"],
      verbs: ["get", "list", "watch"]
    },
    {
      apiGroups: [""],
      resources: ["secrets"],
      verbs: ["list"]
    }
  ]
}
```

Perhatiin detail di rule buat `secrets`: cuma `list`, gak ada `get`. Artinya agent bisa liat nama-nama secret yang exist, tapi gak bisa baca isinya. Ini penting banget buat use case monitoring di mana agent perlu tau secret apa aja yang ada.

Verbs `get`, `list`, `watch` itu read-only. Jadi agent ini literal gak bisa modify anything di cluster, apa pun yang dia coba. Ini target state buat monitoring agent: zero write capability, maximum read visibility.


---

## Pola Buruk: Kesalahan Umum yang Harus Dihindari

### 1. Shared Credentials

Pola bad: agent pake credential yang sama dengan user manusia. Analogi: semua karyawan pake satu card access. Kalau ada yang salah, gak tau siapa yang lakuin.

```json5
// SALAH: Agent pake token yang sama dengan user
{
  gateway: {
    auth: {
      token: "shared-user-token"
    }
  }
}

// BENAR: Gateway token dedicated
{
  gateway: {
    auth: {
      token: "dedicated-gateway-token-rotate-quarterly"
    }
  }
}
```

Yang bikin shared credential berbahaya: kamu kehilangan accountability. Pas ada incident, audit log tunjuk "token X yang execute command", tapi token X itu dipake sama 10 engineer dan 3 agent. Siapa yang benar-benar melakukannya? Gak tau. Investigasi menjadi buntu. Plus, kalau satu dari pengguna token itu kena compromise (laptop curian, phishing attack), kamu harus rotate token untuk semua pengguna sekaligus, yang menyebabkan downtime massal.

### 2. Long-Lived Token

Pola bad: token yang gak pernah expire, gak pernah rotate. Analogi: kunci rumah yang pernah dipake sejak 10 tahun lalu dan gak pernah diganti. Siapapun yang pernah pegang copy kunci itu masih bisa masuk.

```json5
// SALAH: Token yang gak pernah expire
{
  gateway: {
    auth: {
      token: "static-token-that-never-changes"
    }
  }
}

// BENAR: Short-lived, auto-rotating
{
  gateway: {
    auth: {
      token: "current-token",
      rotateOnStart: true,
      rotationInterval: "24h"
    }
  }
}
```

Rotation itu game-changer untuk security. Bahkan kalau token kamu kecuri hari ini, besok token itu udah gak valid lagi karena udah di-rotate. Attacker cuma punya 24 jam (atau less) buat exploit sebelum akses-nya hilang. Ini jauh lebih baik dari token yang valid selamanya.

Yang penting buat rotation: **automate it**. Auto-rotation lewat config atau tool bikin rotation happen tanpa intervensi manusia.

### 3. Broad Permission "Biar Aman"

Pola bad yang paling sering: engineer kasih permission tinggi karena "nanti biar gak error permission". Alasan ini terlihat pragmatis tapi sebenernya anti-pragmatis: kamu trade short-term convenience sama long-term security risk yang exponential lebih besar.

```json5
// SALAH: "Kasih admin aja biar gak error permission"
{
  agents: [
    {
      id: "support-agent",
      capabilities: ["operator.admin"]
    }
  ]
}

// BENAR: Mulai dari permission minimal
{
  agents: [
    {
      id: "support-agent",
      capabilities: ["operator.read", "operator.write"],
      tools: {
        exec: {
          allowed: ["/usr/bin/kubectl", "/usr/bin/docker"]
        }
      }
    }
  ]
}
```

Approach yang benar: start dari nol, dan tambah permission cuma kalau beneran ada kebutuhan yang terdefinisi. Kalau agent crash karena kurang permission, investigate dulu apakah permission itu memang necessary, baru tambah yang spesifik dibutuhin. Jangan langsung kasih blanket admin.

> **Eits, Hati-hati!** "Kasih aja admin biar cepat" itu pemicu dari banyak breach. Dua tahun kemudian, saat ada security audit, tim sadar bahwa half of their agents punya privilege yang dangerous, dan fix-nya butuh berminggu-minggu. Investasi awal buat set up least privilege dari awal itu murah. Refactoring nanti itu mahal.

---

## Checklist: Agent Permission Management

Sebelum deploy agent baru, verify semua ini:

- [ ] Agent punya **identity dedicated** (gak sharing dengan human user)
- [ ] Credential **short-lived** (hitungan menit atau jam, bukan bulan)
- [ ] Access mengikuti **least privilege** (cuma yang beneran dibutuhin)
- [ ] Production access **read-only by default**
- [ ] Write operation butuh **human approval yang eksplisit**
- [ ] Dangerous command **permanent blocked** di deny list
- [ ] Channel access **pairing-based** (bukan wildcard allow all)
- [ ] Secret value **gak pernah bisa dibaca** (metadata only)
- [ ] Access **scoped per capability** (bukan blanket grant)
- [ ] Semua access **ter-log dan auditable**
- [ ] Permission di-review minimal setiap季度
- [ ] Ada automated test yang verify permission matches expected state

Kalau ada satu box aja yang gak ke-check, jangan deploy. Pending sampai gap-nya di-fix. Short-term friction dari delay deploy itu jauh lebih murah dari long-term damage dari permission yang terlalu luas.

### Essential Commands Reference

```bash
# Channel permission management
openclaw channels list                     # list semua channel
openclaw channels status                   # status channel access
openclaw channels check --channel "#x"     # cek akses channel spesifik
openclaw channels permissions --channel "dm-user-123"  # view permission

# Device management
openclaw devices list                      # list pending pairing
openclaw devices approve <requestId>       # approve pairing
openclaw devices reject <requestId>        # reject pairing
openclaw devices revoke <deviceId>         # revoke device
openclaw devices rotate <deviceId>         # rotate credential

# Tool policy
openclaw config get tools.exec.security    # cek mode policy
openclaw config set tools.exec.security allowlist  # set ke allowlist
openclaw config get tools.allowed          # list allowed tool
openclaw config get tools.denied           # list denied tool

# Audit
openclaw security audit                    # run security audit
openclaw approvals get                     # get current approval policy
```


---

## Key Takeaways

- **Privilege inheritance itu sumber masalah**. Agent mewarisi credential dari human yang run dia, dan tiba-tiba punya authority yang jauh lebih tinggi dari yang seharusnya. Selalu scope credential agent secara eksplisit
- **Least privilege itu alat pengurangi blast radius**. Kalau agent cuma punya read access, worst case scenario-nya jauh lebih survivable dari agent yang punya admin
- **Empat dimensi least privilege**: channel access, tool execution, gateway auth, dan device management. Cover semua empat dimensi buat comprehensive protection
- **Allowlist beats denylist always**. Fail-safe approach yang explicit allow known-safe, bukan try to enumerate all known-dangerous
- **Shared credential merusak accountability**. Tiap agent butuh identity sendiri, bukan warisan atau shared token dari user manusia
- **Long-lived token itu ticking bomb**. Implement rotation otomatis, idealnya setiap 24 hours atau less
- **"Kasih admin aja biar cepat" itu anti-pattern**. Investment kecil buat set up least privilege di awal itu expensive refactoring later
- **Platform-specific tools seperti AWS IAM dan K8s RBAC** punya built-in support buat least privilege. Gunakan mereka secara agresif, bukan sebagai afterthought

---

