# Bab 3: Infrastructure as Code, Tapi Digerakkan AI

> *"IaC tanpa AI itu kayak masak dengan resep 50 halaman tapi gak punya timer dan gak bisa ngecek rasa. Bisa sih, tapi makananmu sering gosong atau kurang garam."*

---

## Tantangan IaC yang Kita Hadapi Sehari-hari

Infrastructure as Code (IaC) itu udah jadi standar de facto di dunia DevOps. Konsepnya simpel: daripada ngatur server secara manual lewat console, kamu tulis aja konfigurasinya dalam bentuk kode. Bisa di-version di git, di-review via PR, di-rollback kalau ada masalah. Elegant kan?

Tapi realitanya, IaC itu gak semudah kedengarannya. Ada HCL syntax yang bikin pusing, state management yang harus di-handle supaya gak corrupt, module dependencies yang kalau salah configure bikin semua resource ikut ke-replace, dan provider versioning yang kadang breaking change tanpa warning. Belum lagi kalau engineer baru join tim, harus pelajari sintaks, struktur module, naming convention, dan best practices yang udah ada. Ini bisa makan minggu bahkan bulan sebelum beneran produktif.

**AI agent gak menggantikan keahlian kamu.** Mereka **menguatkannya**. Kayak calculator gak menggantikan mathematician, tapi bikin dia jauh lebih cepat dan akurat. OpenClaw di context IaC berfungsi kayak itu: dia bisa generate config yang benar, detect masalah yang kamu mungkin kelewat, jalanin routine task secara otonom, dan nge-review PR kamu dengan mata yang gak pernah lelah.

Kenapa ini penting? Karena tanpa bantuan, setiap troubleshooting jadi tugas berulang yang makan waktu. Yang seharusnya 10 menit bisa jadi 2 jam karena kamu harus baca docs, debug, coba-coba. Dengan AI yang udah paham context infra kamu, banyak task ini bisa dipecahkan jauh lebih cepat.


---

## Landscape Tool IaC di 2026

Sebelum kita masuk ke gimana OpenClaw bisa bantu, mari lihat dulu landscape-nya. Tool IaC yang dipakai industri sekarang cukup beragam, dan masing-masing punya karakternya sendiri:

- **Terraform/OpenTofu**: Pemimpin pasar (Terraform v1.14, OpenTofu v1.11, keduanya aktif di-develop) vs Exec tool integration, standing orders
- **Pulumi**: Pertumbuhan kuat (v3.220), terus berinovasi vs Automation hooks
- **CloudFormation**: Spesifik AWS, stabil dengan IaC Generator yang makin bagus vs Cron jobs untuk template management
- **Ansible**: Pemimpin configuration management (v2.18) dengan Lightspeed vs TaskFlow automation
- **CDK/CDKTF**: CDKTF masih maintained, HashiCorp fokus ke native HCL vs Enhanced via exec tool
- **Crossplane**: K8s-native, v2 release major (v2.1), ecosystem makin mature vs Skill integration

Trend utamanya: semua tool IaC besar terus aktif develop dan makin nge-support AI integration. Terraform dan OpenTofu koeksisten, dan AI agent bisa jadi *glue* yang connect semuanya. Jadi kamu gak harus commit ke satu tool aja. Pakai yang paling cocok buat use case kamu, dan biarkan OpenClaw jadi interface yang unified.

> **Pro Tip:** Kalau kamu baru mulai, Terraform/OpenTofu masih yang paling banyak dipakai dan paling banyak resource-nya. Tapi kalau tim kamu lebih nyaman dengan programming language (bukan HCL), Pulumi atau CDKTF bisa jadi pilihan yang lebih natural.

---

## Use Case 1: Generate Infra dengan Standing Orders

### Masalahnya

Bayangin kamu engineer yang baru gabung. Kamu harus bikin infrastructure baru, misalnya PostgreSQL database buat staging environment. Kalau manual, kamu harus: buka docs Terraform, cari module yang benar, pelajari naming convention tim, pastikan security group-nya benar, tambah backup configuration, dll. Butuh waktu lama, dan kemungkinan besar ada yang terlewat.

### Solusinya: Standing Orders

Standing orders itu kayak "deskripsi pekerjaan" buat AI agent. Kamu tentukan aturan-aturan: agent boleh ngapain, gak boleh ngapain, tool apa yang boleh dipakai, dan constraint-nya apa. Terus kamu tinggal kasih perintah dalam bahasa natural, dan agent eksekusi sesuai aturan yang udah ditetapkan.

Contoh konfigurasi standing order di `AGENTS.md`:

```text
## Program: Weekly Environment Management
**Authority:** Create, update, and destroy infrastructure environments
**Trigger:** Natural language commands matching "infra|terraform|pulumi|cloudformation"
**Allowed tools:** exec:terraform, exec:pulumi, exec:git, exec:aws
**Parameters:**
- target: environment-specific directories
- sandbox: true for safety
- dangerous_code: "alert" for review-required operations
```

Dengan konfigurasi ini, kamu bisa chat: *"Bikin PostgreSQL database buat staging, high availability, dengan backup 7 hari"* dan agent bakal generate konfigurasi Terraform yang sesuai. Dia tahu module mana yang harus dipakai (karena udah di-configure di context), naming convention apa yang harus diikuti, dan praktik terbaik apa yang harus diterapkan.

**Yang terjadi di belakang layar:** Agent membaca standing order, mengerti bahwa permintaan kamu masuk ke kategori "infra", lalu dia susun context yang relevan (module yang tersedia, convention yang berlaku), generate kode Terraform, dan kasih hasilnya ke kamu buat di-review. Dia gak langsung apply. Kamu yang tetap punya kontrol.

> **Eits, Hati-hati!**
>
> Standing orders itu powerful, tapi juga bisa berbahaya kalau di-configure terlalu longgar. Selalu set `sandbox: true` buat operasi yang modify infrastructure, dan pastikan `dangerous_code: "alert"` aktif biar agent minta konfirmasi sebelum jalanin operasi yang berisiko.

### Exec Tool buat Operasi Infra

OpenClaw punya **exec tool** yang jadi jembatan antara agent dan command-line tools. Lewat exec tool, agent bisa jalanin Terraform, Pulumi, AWS CLI, atau tool apa aja yang ada di system.

```json5
// Jalanin terraform init
{
  "tool": "exec",
  "command": "terraform init terraform/production/",
  "workdir": "/workspace/terraform"
}

// Jalanin terraform plan di background
{
  "tool": "exec",
  "command": "terraform plan terraform/production/ -out=prod.plan",
  "workdir": "/workspace/terraform",
  "background": true,
  "yieldMs": 10000
}
```

Parameter `background: true` bikin command jalan di background, jadi agent gak "tunggu" sampai selesai. Berguna buat `terraform plan` yang bisa makan waktu lama. Kamu bisa ngapain aja sambil nunggu, dan agent bakal beritahu pas hasilnya ready. **`workdir`** itu directory kerja di mana command dijalanin, kayak `cd` di terminal. Selalu set ke directory yang benar biar Terraform nemuin file-file yang dimaksud.

### Track Background Task

Kalau task jalan di background, kamu bisa lacak statusnya:

```bash
# List semua background task
openclaw tasks list

# Cek status task tertentu
openclaw tasks show prod-plan

# Cancel kalau perlu
openclaw tasks cancel prod-plan
```

> **Contoh Nyata:**
>
> Kamu lagi jalanin Terraform plan buat production deployment yang butuh 15 menit. Daripada tunggu, kamu bisa lanjut kerja lain. OpenClaw bakal beritahu kamu lewat Slack pas plan selesai. Kalau ada yang salah (misalnya plan menunjukkan ada resource yang bakal di-replace), kamu bisa cancel dan telusuri lebih lanjut.


---

## Use Case 2: Troubleshooting Infra dengan AI

Ini salah satu use case yang paling *berdampak*. Troubleshooting infra itu menegangkan, terutama kalau terjadi di tengah malam atau di saat sibuk. AI agent bisa bantu diagnose masalah jauh lebih cepat dari manual.

### State Lock Issues

State lock itu masalah klasik di Terraform. Terjadi kalau dua proses coba ubah state secara bersamaan. Kayak dua orang mau ngerjain file yang sama di Google Docs tanpa mode kolaborasi. Hasilnya: conflict.

```
Kamu: "Terraform stuck di state lock dari CI run yang gagal"

OpenClaw: "Oke, aku cek dulu state lock-nya."

// Investigate state lock
{
  "tool": "exec",
  "command": "terraform force-unlock -force -lock=abc-123-def",
  "workdir": "terraform/production"
}

// Alternative: backup state dulu, terus refresh
{
  "tool": "exec",
  "command": "mv terraform.tfstate terraform.tfstate.backup && terraform refresh",
  "workdir": "terraform/production"
}
```

Yang bagus, agent gak langsung eksekusi begitu aja. Dia bakal jelasin opsi-opsi yang ada dan nanya kamu mau pakai yang mana. Ini penting banget karena `force-unlock` itu operasi yang gak bisa di-undo.

### Drift Detection

**Drift** itu kondisi dimana infrastruktur real udah berbeda dari kode IaC. Biasanya terjadi karena modifikasi manual lewat console (iya, kita semua pernah ngelakuin ini saat urgent).

```
Kamu: "Cek apakah production infra udah drift dari Terraform state"

OpenClaw: "Aku jalanin drift detection."

// Detect drift
{
  "tool": "exec",
  "command": "terraform plan -out=drift.plan",
  "workdir": "terraform/production"
}
```

Kamu bisa schedule drift detection otomatis:

```bash
# Schedule drift detection tiap hari jam 2 pagi
openclaw cron add --name "Drift Check" --cron "0 2 * * *" --session isolated --command "terraform plan -out=drift.plan" --workdir "terraform/production" --announce
```

Jadi tiap pagi kamu bisa lihat report: apakah ada yang berubah semalam, dan kalau ada, apa yang perlu di-reconcile. Tanpa cron ini, drift bisa gak terdeteksi sampai menyebabkan incident.

> **Pro Tip:** Jadwalkan drift detection di jam sepi (2-3 pagi) biar gak ganggu operasional. Set `--announce` biar hasilnya otomatis di-post ke Slack.

### Module Dependency Issues

Masalah lain yang sering terjadi: upgrade module bikin resource yang tak terduga ikut ke-replace. Misalnya kamu upgrade VPC module dan tiba-tiba subnet ikut di-recreate.

```
Kamu: "Upgrade module VPC kita bikun resource yang unexpected ke-replace"

OpenClaw: "Aku analyze plan-nya dulu."

// Analyze upgrade impact
{
  "tool": "exec",
  "command": "terraform plan -out=vpc-upgrade.plan",
  "workdir": "terraform/production"
}

// Buat TaskFlow buat resolusi otomatis
{
  "tool": "taskflow",
  "flow": "module-upgrade-fix",
  "variables": {
    "plan_file": "vpc-upgrade.plan",
    "module_path": "modules/vpc"
  }
}
```

Agent bisa menganalisis output `terraform plan`, identifikasi resource yang bakal ke-replace, dan kasih rekomendasi. Misalnya: *"Subnet A dan B bakal di-replace karena attribute `map_public_ip_on_launch` berubah. Tambah `lifecycle { ignore_changes = [map_public_ip_on_launch] }` untuk hindari replacement."*


---

## Use Case 3: Workflow Otomatis dengan TaskFlow

Kalau standing orders itu buat perintah individual, **TaskFlow** itu buat workflow yang multi-step. Kayak resep masak yang punya urutan langkah yang jelas: bahan harus disiapin dulu, terus dicampur, terus dipanggang. Gak bisa dibalik urutan.

### TaskFlow buat Production Deployment

Contoh nyata: production deployment dengan Terraform. Ini process yang gak bisa di-skip langkah-langkahnya. Kamu harus validate, plan, review plan, baru apply. TaskFlow memastikan urutan ini diikuti persis.

```json5
{
  "tool": "taskflow",
  "flow": "production-deploy",
  "description": "Production deployment with validation and rollback",
  "steps": [
    {
      "name": "validation",
      "command": "terraform validate",
      "workdir": "terraform/production"
    },
    {
      "name": "plan",
      "command": "terraform plan -out=prod.plan",
      "workdir": "terraform/production",
      "background": true
    },
    {
      "name": "approval",
      "type": "manual",
      "prompt": "Review terraform show prod.plan sebelum lanjut?"
    },
    {
      "name": "apply",
      "command": "terraform apply prod.plan",
      "workdir": "terraform/production",
      "depends": ["approval"]
    },
    {
      "name": "verify",
      "command": "terraform output",
      "workdir": "terraform/production",
      "depends": ["apply"]
    }
  ]
}
```

Hal penting di TaskFlow ini:

1. **Urutan dijaga**: Step "apply" punya `depends: ["approval"]`, gak bisa jalan sebelum approval.
2. **Manual gate**: Step "approval" tipenya `manual", harus ada manusia yang approve.
3. **Background execution**: Step "plan" jalan di background biar gak blocking.
4. **Verification**: Step terakhir verify output, biar yakin deployment berhasil.

> **Coba Bayangin...**
>
> TaskFlow itu kayak conveyor belt di pabrik. Setiap item harus lewat pemeriksaan kualitas di setiap station sebelum lanjut. Kalau ada yang fail, proses berhenti dan manusia di-pemberitahuan. Gak ada cara buat skip quality check.


### Scheduled Infrastructure Management

TaskFlow bisa dijadwalkan lewat cron jobs:

```bash
# Morning health check tiap hari jam 7
openclaw cron add --name "Morning Health Check" --cron "0 7 * * *" --session isolated --message "Check infrastructure status and report any issues" --announce

# Cost optimization review tiap Senin jam 9
openclaw cron add --name "Cost Optimization" --cron "0 9 * * 1" --session isolated --command "terraform plan -out=cost-opt.plan -var-file=cost_optimization.tfvars" --workdir "terraform/production" --announce

# Drift report bulanan
openclaw cron add --name "Monthly Drift Report" --cron "0 6 1 * *" --session main --message "Generate infrastructure drift report for all environments" --announce

# Security scan tiap Minggu jam 2 pagi
openclaw cron add --name "Security Scan" --cron "0 2 * * 0" --session custom --command "tfsec ." --workdir "terraform" --announce
```

Parameter `--session`: kamu bisa pilih `isolated` (session baru, bersih), `main` (pakai session utama), atau `custom` (sesuai kebutuhan).

> **Pro Tip:** Set `--announce` di semua cron job biar hasilnya otomatis di-post. Tim bisa lihat hasil tanpa ngecek manual.

---

## Use Case 4: Code Review Otomatis buat IaC

Code review itu penting, tapi jujur aja, kadang kita kelewat hal-hal penting karena banyaknya PR yang harus di-review. Apalagi kalau PR-nya 500 line HCL yang bikin mata pegal. AI agent bisa jadi "sepasang mata kedua" yang gak pernah lelah.

Contoh output review dari OpenClaw:

```
OpenClaw (automated PR review): "Review of terraform/production/main.tf:

  CRITICAL:
  - Line 47: Security group allows 0.0.0.0/0 on port 3306 (MySQL).
    Ini expose database ke internet. Chiaw!
    Fix: Restrict ke VPC CIDR atau IP range tertentu.

  WARNING:
  - Line 23: RDS instance pakai db.t3.micro buat production.
    Instance class ini burstable CPU cuma 1GB RAM.
    Recommendation: Minimal db.r6g.large buat production.

  - Line 89: Belum ada backup_retention_period.
    Default cuma 1 hari. Production seharusnya 7-35 hari.

  INFO:
  - Line 12: AWS provider version ~> 5.0.
    Latest 5.x.x. Pertimbangkan pin ke ~> 5.x.x
    buat dapat security patch terbaru.

  - Missing: Belum ada lifecycle prevent_destroy di RDS instance.
    Tambahin buat mencegah accidental deletion via terraform destroy."
```

Ini level detail yang sering kelewat di manual review. Security group yang open ke internet, instance size yang kekecilan buat production, backup yang belum di-set. Semua ini bisa kelewat kalau reviewer lagi buru-buru atau gak familiar dengan Terraform.

Yang penting, review ini jalan **otomatis** pas PR dibuat. Jadi sebelum reviewer manusia lihat, AI udah flag hal-hal yang critical. Reviewer bisa fokus ke business logic dan architecture decision, biar yang detail-detail kayak di atas di-handle AI.

> **Contoh Nyata:**
>
> Junior developer baru aja bikin PR buat nambah database. Dia lupa set backup retention. Tanpa AI review, ini bisa masuk ke production dan baru ketahuan pas data hilang 3 bulan kemudian. Dengan AI review, issue ini terdeteksi dalam hitungan detik setelah PR dibuat.


---

## Use Case 5: Skill Khusus buat Infrastructure

OpenClaw punya skill system yang bisa di-customize buat kebutuhan infrastructure. Skill itu kayak plugin khusus yang kasih agent kemampuan tambahan di domain tertentu.

### Contoh Skill: Terraform Automation

```markdown
---
name: terraform-automation
description: Otomasi Terraform tingkat tinggi dengan drift detection dan optimasi cost
metadata:
  {
    "openclaw":
      {
        "requires": { "bins": ["terraform", "tfsec", "jq"] },
        "primaryEnv": "AWS_ACCESS_KEY_ID",
      },
  }
---
```

Skill ini membutuhkan binary `terraform`, `tfsec`, dan `jq` terinstall di system. Dia juga butuh environment variable `AWS_ACCESS_KEY_ID`. Kalau dependensi ini gak ada, skill-nya gak bakal aktif dan agent bakal beritahu kamu.

Konfigurasi skill di `openclaw.json`:

```json5
{
  skills: {
    entries: {
      "terraform-automation": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "AWS_ACCESS_KEY_ID" },
        env: {
          AWS_ACCESS_KEY_ID: "YOUR_AWS_KEY",
          AWS_SECRET_ACCESS_KEY: "YOUR_AWS_SECRET"
        },
        config: {
          environment: "production",
          regions: ["us-east-1", "us-west-2"],
          cost_threshold: 1000
        },
      },
      "cost-optimizer": {
        enabled: true,
        env: {
          TF_COST_API_KEY: "cost_api_key_here"
        }
      }
    },
  },
}
```

Pakai skill-nya gampang:

```bash
# Jalankan cost optimization
openclaw skill cost-optimizer --target terraform/production

# Generate drift report
openclaw skill terraform-automation --action drift-report --environment production

# Apply dengan validasi otomatis
openclaw skill terraform-automation --action apply-with-verify --plan-file prod.plan
```

> **Pro Tip:** Kamu bisa bikin skill sendiri yang spesifik buat environment kamu. Misalnya skill "company-deploy" yang ngerti internal module registry dan approval flow tim kamu.


---

## Anti-Patterns: Jangan Lakukan Ini!

Bagian ini penting banget. AI agent itu powerful, dan dengan power datang responsibility. Ini hal-hal yang **jangan pernah** kamu lakukan:

### 1. Blind Apply (Auto-Apply tanpa Review)

Ini dosa terbesar di dunia AI + IaC. Jangan pernah, dalam kondisi apapun, biarkan AI agent jalanin `terraform apply` tanpa review manusia dulu.

```json5
// SALAH: Auto-apply tanpa review
steps: [
  "terraform plan",
  "terraform apply -auto-approve"  // BAHAYA BANGET
]

// BENAR: Require human approval
steps: [
  "terraform plan -out=plan.tfplan",
  "human_review(plan.tfplan)",
  "terraform apply plan.tfplan"     // Cuman apply plan yang udah di-review
]
```

Kenapa? Karena `terraform apply -auto-approve` itu kayak kasih kunci rumah ke orang asing dan bilang "silakan masuk dan lakukan apain aja yang kamu mau". Kamu gak tahu apa yang bakal dia lakukan di dalam. AI bisa halusinasi dan ngerjain hal yang gak kamu mau.

### 2. State Manipulation oleh AI

AI agent **jangan pernah** langsung manipulate Terraform state:

```json5
// Command yang harus di-BLOCK buat AI:
blocked_commands: [
  "terraform state rm",
  "terraform state mv",
  "terraform import",
  "terraform taint"
]

// Yang boleh: agent propose perubahan, manusia eksekusi
allowed_actions: [
  "suggest_state_change",  // Jelaskan apa yang perlu diubah
  "generate_moved_blocks"  // Pakai moved{} blocks sebagai gantinya
]
```

> **Contoh Nyata:** Sebuah company hampir kehilangan database production karena AI otomatis jalanin `terraform state rm`. Untungnya ada engineer yang ngecek sebelum kerusakannya permanen. Pelajarannya: state itu sacred, hanya manusia yang boleh modify.

### 3. Secret Leakage

Config IaC sering banget berisi atau merujuk ke secrets. Pastikan agent gak pernah nge-expose secret di logs atau chat:

```json5
// Filter output untuk mencegah kebocoran secret
output_filters: [
  {
    pattern: "(password|secret|key|token)\\s*=\\s*\"[^\"]+\"",
    action: "redact",
    replacement: "***REDACTED***"
  },
  {
    pattern: "AKIA[0-9A-Z]{16}",  // AWS access key pattern
    action: "redact"
  }
]
```

Kalau agent nemuin secret di output, dia otomatis replace jadi `***REDACTED***`. Ini mencegah secret kebaca di Slack, Telegram, atau channel lain yang mungkin gak aman yang kamu kira.

### 4. Ignore Convention yang Udah Ada

Tim kamu udah punya module yang teruji dan naming convention yang jelas? Jangan bikin yang baru cuma karena AI bisa generate dari nol. Selalu pilih module yang udah ada:

```json5
// Referensi module dan convention yang udah ada
context: {
  module_registry: "https://registry.internal/terraform/modules",
  naming_convention: "{project}-{env}-{resource}-{region}",
  required_tags: [
    "environment",
    "team",
    "cost-center",
    "managed-by"
  ],
  existing_modules: [
    { source: "registry.internal/vpc/gcp" },
    { source: "registry.internal/gke/gcp" },
    { source: "registry.internal/cloudsql/gcp" },
  ]
}
```

> **Pro Tip:** Di bootstrap files, selalu sertakan referensi ke module registry dan naming convention tim kamu. Ini bikin agent generate kode yang konsisten.

---

## Use Case 6: Webhook-Triggered Infrastructure

Webhook itu cara buat bikin infrastructure bereaksi secara otomatis ke event dari system lain. Misalnya setiap kali ada pull request, otomatis jalanin Terraform plan dan post hasilnya sebagai comment di PR.

### Konfigurasi Webhook

```bash
# Bikin webhook buat PR-based deployment
openclaw cron add-webhook --name "PR Deployment" --url "https://your-gateway/hooks/deploy" --events ["pull_request"] --secret "your-secret"

# Configure webhook buat trigger TaskFlow
openclaw taskflow create --name "pr-deploy-flow" --trigger "webhook" --event "pull_request" --flowfile "taskflows/pr-deploy.json"
```

### Workflow Webhook

```json5
{
  "flow": "pr-deploy",
  "trigger": "webhook",
  "events": ["pull_request"],
  "steps": [
    {
      "name": "validate-pr",
      "command": "terraform validate",
      "workdir": "terraform/production"
    },
    {
      "name": "plan",
      "command": "terraform plan -out=pr.plan",
      "workdir": "terraform/production"
    },
    {
      "name": "comment",
      "type": "github-comment",
      "template": "Terraform plan generated for PR #{{PR_NUMBER}}"
    },
    {
      "name": "approve",
      "type": "manual",
      "prompt": "Approve PR #{{PR_NUMBER}} deployment?"
    },
    {
      "name": "deploy",
      "command": "terraform apply pr.plan",
      "workdir": "terraform/production",
      "depends": ["approve"]
    }
  ]
}
```

Flow-nya: developer bikin PR, webhook trigger OpenClaw, OpenClaw jalanin validate dan plan, hasilnya di-post sebagai comment, reviewer approve/reject, kalau approve baru di-apply. Semua otomatis kecuali approval.

> **Contoh Nyata:** Tim frontend bikin PR dan otomatis staging environment di-deploy. Mereka bisa uji fitur baru tanpa tunggu DevOps team deploy manual.


---

## Checklist: IaC dengan AI yang Aman

Sebelum kamu setup AI-assisted IaC, pastikan ini semua udah di-check:

- [ ] AI agent generate IaC tapi **gak pernah auto-apply** ke production
- [ ] Semua kode yang di-generate lewat standard PR review process
- [ ] Output `terraform plan` di-review manusia sebelum apply
- [ ] Command state manipulation di-block buat AI agent
- [ ] Sensitive values di-redact dari output agent
- [ ] Kode yang di-generate referensi module dan convention yang udah ada
- [ ] Cost estimate di-include di generated plans
- [ ] Drift detection jalan sesuai schedule dengan report yang di-review manusia
- [ ] Agent punya read-only access ke production state, write access cuma ke staging
- [ ] Semua IaC yang di-generate agent di-tag `managed-by: openclaw` buat traceability
- [ ] Standing orders di-configure dengan authority dan constraint yang proper
- [ ] TaskFlow workflows include manual approval gates di critical steps
- [ ] Cron jobs pakai session isolation yang sesuai (isolated/main/custom)
- [ ] Webhook integration pakai secret dan validation yang proper


---

## Key Takeaways

- AI agent bisa **generate, review, dan troubleshoot** IaC, tapi **manusia tetap yang approve** sebelum apply
- **Standing orders** kayak deskripsi pekerjaan buat AI: tentukan apa yang boleh dan gak boleh dilakukan
- **TaskFlow** bikin workflow multi-step jadi reliable dan terurut, dengan manual approval gate di critical points
- **State manipulation dan auto-apply** adalah risiko terbesar, selalu ada review manusia di antara mereka
- **Drift detection** yang ter-schedule jadi asuransi infra kamu: pastikan real world selalu cocok sama kode
- **Code review otomatis** bisa tangkap hal-hal yang sering kelewat di manual review (security group open, instance kekecilan, dll)
- **Skill system** bikin agent bisa di-customize sesuai convention dan tool internal tim kamu

---

