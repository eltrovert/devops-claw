# Bab 9: Cloud Cost Optimization, Aka Jangan Bayar Buat Zombie

> *"Pengeluaran cloud tanpa monitoring itu kayak sapu tangan di warung: gratis dulu karena gak ada meter, terus pas akhir bulan tagihan datang dan bikin jantung serasa ditusuk pisau dingin. Agentic FinOps itu thermostat cerdas buat cloud spending kamu. Gak cuma notif saat overheating, tapi juga self-adjust buat hemat energi."*

---

## Masalah Cloud Spending yang Ngeselin

Cloud computing itu ajaib, bro. Spin-up instance dalam hitungan detik, bayar per jam, scale otomatis pas traffic naik. Konsepnya elegant. Tapi realitanya, billing model pay-per-use yang sama yang bikin cloud jadi menarik juga bikin spending jadi monster yang susah dikontrol.

Di sebagian besar company, cloud spending adalah item budget yang paling cepat tumbuhnya. Gak kayak salary yang previsibel atau office rent yang tetap bulan ke bulan. Cloud bisa tiba-tiba naik 40% karena developer lupa terminate instance test, atau karena orphaned snapshots yang gak pernah dihapus. Pas akhir bulan, finance team kaget sama invoice yang kayak gak pernah terjadi.

Masalahnya lebih dalam lagi. Traditional FinOps bergantung pada dashboard dan manual review. Kamu lihat cost report, minta jem IT "cek ini expensive gak sih", mereka investigate, buat ticket, terus habis 2 minggu ada improvement. Tapi cost bisa naik lagi bulan depan karena proses ini gak otomatis.

**Agentic FinOps** mengubah paradigma ini. Bukan sekadar monitoring, tapi sistem loop tertutup: deteksi waste → analisis dampak → tindakan optimasi → pembelajaran dari hasil. OpenClaw menjadi "Financial Ops Engineer" yang 24/7 mendeteksi waste, punya konteks tentang prioritas bisnis, dan bisa membuat keputusan otonom dengan pengendali yang tepat.


---

## Sistem FinOps Terotomasi: Lihat, Pikir, Ambil Tindakan, Belajar

Kenapa ini penting? Optimasi cost yang reaktif itu kayak pasang patch di ban mobil yang gendut. Kamu bisa jalan dulu, tapi tidak lama lagi akan gendut lagi. Dengan Agentic FinOps, kamu punya sistem yang proaktif: detect trend sebelum jadi bencana, analyze sebelum act, dan improve loop-nya based on data.

Bayangkan sistem ini sebagai thermostat cerdas buat cloud spending. Gak cuma measure suhu (biaya), tapi juga pelajari pola (traffic patterns, seasonal trends, deployment habits), predict changes, dan adjust setting otomatis. Kalau temperature naik unusual, alarm langsung. Kalau dia bisa optimize tanpa risiko, dia do it. Kalau ada trade-off antara reliability dan cost, dia escalate ke manusia.

Loop-nya memiliki empat tahap:

```
      ┌──────────┐
      │ Perceive │◄── Cloud billing APIs, usage metrics, logs
      └────┬─────┘
           │
      ┌────▼─────┐
      │  Reason  │◄── Cost models, business rules, historical patterns
      └────┬─────┘
           │
      ┌────▼─────┐
      │   Act    │──► Resize, schedule, delete, reserve
      └────┬─────┘
           │
      ┌────▼─────┐
      │  Learn   │──► Was the action effective? Update models
      └──────────┘
```

Di tahap Perceive, OpenClay mengakses API penyedia cloud untuk mendapatkan data pengeluaran dan metrik real-time.

Di tahap Reason, agen menerapkan logika bisnis dan model cost. Contoh: "Setiap instance idle 14 hari boleh di-terminasi". Dia merujuk pola historis untuk prediksi.

Di tahap Act, agen menjalankan perubahan dengan alur persetujuan. resize instance, jadwalkan shutdown, beli Reserved Instances.

Di tahap Learn, sistem melacak hasil dari setiap aksi untuk keputusan di masa depan.

> **Contoh Nyata:**
>
> Sebuah startup yang kami assist pake Agentic FinOps buat 6 bulan. Hasilnya: detect dan fix 47 separate cost waste issues, total savings $15.000/bulan. Itu cukup buat hire 2 engineer junior! Yang penting, gak ada single incident dimana cost saving bikin application down. Sistem tahu apa yang bisa dipotong tanpa affect reliability.


---

## Kasus 1: Deteksi Waste Terotomasi

### Masalah Resource Zombie

Bayangin punya 23 karyawan yang kamu bayar full-time tapi mereka gak ngapain selama 14 hari. Pake space di office, makan snack, tapi literally zero output. Ini boros banget, kan? Itulah exactly kondisinya 23 EC2 instances yang idle dalam kasus nyata yang pernah kami lihat.

Resource zombie ini tidak jarang. Developer meluncurkan instance untuk load testing, lupa mengakhiri. Menyiapkan environment staging baru, tidak menghapus yang lama. Backup database lama disimpan "just in case". Orphan snapshots dan elastic IP yang terpisah semuanya secara diam-diam menguras anggaran.

OpenClaw mendeteksi waste dengan kecerdasan. Dia bukan hanya "instance A idle", tetapi memberikan konteks lengkap: durasi, pembuat, status backup, penilaian risiko. Ini berbeda dengan dashboard tool yang hanya memberikan metrik mentah.

```
OpenClaw (Monday 9 AM report): "Cloud Cost Optimization Report

  Bulan lalu: $34.200 spend
  Bulan sebelumnya: $30.500 spend
  Perubahan: +12% (naik $3.700)

  ✓ WASTE IDENTIFIED: $10.380/month (30% dari spend)
  
  Breakdown per kategori:
  
  ┌─────────────────────────────────────────────────────────┐
  │ Kategori          │ Resources │ Cost/Bulan │ Risiko   │
  ├───────────────────┼───────────┼────────────┼──────────┤
  │ Idle EC2          │ 23        │ $3.800     │ LOW      │
  │ Oversized RDS     │ 5         │ $2.500     │ MEDIUM   │
  │ Unused disk       │ 47        │ $1.600     │ LOW      │
  │ Orphan snapshot   │ 312       │ $1.100     │ LOW      │
  │ Unattached EIP    │ 18        │ $780       │ LOW      │
  │ Dev env 24/7      │ 3 env     │ $600       │ MEDIUM   │
  └─────────────────────────────────────────────────────────┘

  TOP FINDINGS:

  1. 23 IDLE EC2 INSTANCES ($3.800/month)
     • 15 dari load test 3 minggu lalu
     • 5 old staging environment (replaced 2 minggu lalu)
     • 3 dev instance @charlie (0 CPU 14+ hari)
     Recommended: Delete dengan grace period 7 hari

  2. 5 OVERSIZED RDS INSTANCES ($2.100/month)
     - analytics-db: db.r6g.2xlarge → recommend db.r6g.large
       Savings: $800/month, Risk: Downtime ~30sec
     - staging-db: Multi-AZ enabled → disable
       Savings: $400/month, Risk: LOW

  3. 312 ORPHAN SNAPSHOTS ($890/month)
     Oldest: 18 bulan lalu, 100% SAFE to delete

  4. DEV ENVIRONMENTS 24/7 ($600/month)
     Schedule down Mon-Fri 9AM-6PM: Save 73% = $440/month

  RECOMMENDED ACTIONS (priority):
  
  ✓ Delete 312 orphan snapshots (SAFE, $1.100/month)
  ✓ Release 18 unattached EIPs (SAFE, $780/month)
  ✓ Schedule dev environments (SAFE, $440/month)
  ? Terminate idle instances (ASK OWNER, $3.800/month)
  ? Resize databases (MAINTENANCE WINDOW, $1.200/month)
```

Lihat selisih report ini dengan dashboard biasa? Di sini ada reasoning, risk assessment, dan prioritization. OpenClaw sudah melakukan berpikir untukmu.

### Automation: Cron Job buat Weekly Waste Analysis

```bash
# Schedule weekly waste cleanup report tiap Senin jam 8 pagi
openclaw cron add \
  --name "Cloud Waste Analysis" \
  --cron "0 8 * * 1" \
  --session isolated \
  --message "Analyze cloud resources buat waste, generate recommendations, execute safe actions"
```

Cron job ini run isolated session, jalanin analysis, generate report. Bisa auto-execute "safe" actions dan escalate "risky" actions ke approval.

### Real-Time Cost Tracking

OpenClaw memberikan visibilitas untuk tracking penggunaan. Periksa cost real-time:

```bash
/status
/usage full
/usage cost
openclaw status --usage
```

---

## Program: Cloud Waste Cleaner

**Wewenang:** Mengidentifikasi resource yang terbuang, menghasilkan laporan pembersihan, menjalankan tindakan pembersihan yang aman

**Pemicu:** Jadwal mingguan + pemicu manual untuk analisis on-demand

**Pintu persetujuan:** Tindakan aman dieksekusi otomatis (orphan snapshots, unattached IPs); tindakan berisiko memerlukan konfirmasi per resource

**Eskalasi:** Memberi tahu pemilik resource, menandai untuk penghapusan jika tidak ada respons dalam 7 hari

### Execution Steps

1. **Resource Analysis**
   - Query AWS/GCP APIs buat utilization metrics
   - Check CPU utilization (<5% selama 14 hari)
   - Identify unattached volumes, orphaned snapshots
   - Find oversized instances, unused load balancers

2. **Risk Assessment**
   - Tag "backup", "archive" resources sebagai protected
   - Check dependencies sebelum deletion
   - Identify resources yang butuh maintenance window

3. **Cleanup Actions**
   - Notify owners dengan report
   - Tag resources untuk deletion dengan grace period
   - Execute safe deletions
   - Generate before/after cost report

### Hook untuk Real-Time Monitoring

Monitoring resource bisa diotomatisasi dengan hook:

```bash
# Create hook yang monitor perubahan resource
echo '#!/usr/bin/env node
const hook = {
  name: "resource-cost-monitor",
  event: ["resource:create", "resource:update"],
  handler: async (ctx) => {
    if (shouldFlagAsWaste(resource)) {
      await ctx.notify({
        channel: "#cost-alerts",
        message: `⚠️ Potential waste: ${resource.id}`
      });
    }
  }
};
' > ~/hooks/resource-cost-monitor.js

openclaw hooks enable resource-cost-monitor
```


---

## Kasus 2: Perencanaan Reserved Instance Cerdas

### Kenapa RI itu Penting (tapi Sering Salah)

Reserved Instance bisa save hingga 75% biaya compute kalau di-use dengan benar. Tapi banyak company salah gunakan - commit ke instance type yang gak consistent dipakai, atau commit 3 tahun untuk teknologi yang cepat obsolete.

OpenClaw membuat perencanaan RI menjadi cerdas dan berbasis data. Dia menganalisis penggunaan historis 6-12 bulan, mengidentifikasi komputasi baseline yang stabil, lalu merekomendasikan komitmen RI optimal. Dia melacak coverage rate dan memberi alert jika pola penggunaan berubah.

### Monthly RI/Savings Plan Analysis

```bash
# Schedule monthly RI coverage analysis tiap awal bulan
openclaw cron add \
  --name "RI Coverage Analysis" \
  --cron "0 6 5 * *" \
  --session isolated \
  --message "Analyze RI coverage, cost vs benefit, recommendation buat next month"
```

### Program: Reserved Instance Planner

**Wewenang:** Menganalisis pola penggunaan komputasi, merekomendasikan coverage RI, melacak realisasi penghematan

**Pemicu:** Analisis terjadwal bulanan + pemicu berbasis threshold (jika pola penggunaan berubah signifikan)

**Pintu persetujuan:** Menyajikan rekomendasi ke finance, memerlukan persetujuan sebelum menjalankan pembelian RI

**Eskalasi:** Memberi alert jika pola penggunaan berubah, risiko over-commitment yang potensial

### Execution Steps

1. **Usage Analysis**
   - Analyze 6-month historical compute usage
   - Identify consistently utilized instances
   - Calculate current RI coverage
   - Predict future demand

2. **Recommendation Engine**
   - Calculate optimal RI coverage (target 75%)
   - Compare 1-year vs 3-year trade-offs
   - Evaluate payment options

3. **ROI Analysis**
   - Project monthly savings
   - Assess usage turun risk
   - Generate business case

### Contoh Analysis Output

```markdown
## Reserved Instance / Savings Plan Optimization

**Current situation:**
- Total compute spend: $8.200/month
- RI coverage: 42% dari spend
- Average savings rate: 31% dari RI that we have

**Analysis period:** Last 6 months

**Recommended actions:**

### 1. Compute Savings Plan (3-year, partial upfront)
   Commitment: $3.000/bulan
   Savings: $1.330/bulan (31% discount)
   Covers: 85% dari stable baseline compute
   Break-even: Month 4
   Risk level: LOW
   Rationale: Covers konsisten usage dari production API servers

### 2. RDS Reserved Instances (1-year, no upfront)
   Current on-demand cost: $3.100/bulan
   RI cost with commitment: $2.150/bulan
   Savings: $950/bulan
   Coverage: 70% dari database spend
   Risk level: MEDIUM (analytics-db mungkin di-resize Q3)

### 3. DO NOT COMMIT YET:
   - Staging compute (may be terminated saat K8s migration)
   - Batch processing (usage bervariasi, gak predictable)
   - Development environment (scheduled, not constant)

**Financial Summary:**
- Annual savings dengan recommendations: $27.360
- Total commitment: $36.000/year
- ROI: 76% dalam year pertama
- Approval needed from: Finance team + Engineering lead

**Risk considerations:**
- K8s migration planned Q3 might change instance requirements
- Consider option: buy 6-month RI dulu, reassess after migration
- Alternatively: buy 3-year tapi focus ke instance type yang definitely stable
```


---

## Kasus 3: Deteksi Anomali Cost Real-Time

### Kenapa Anomaly Detection Penting

Cost anomaly detection deteksi spike sebelum jadi bencana. Cost bisa spike karena deployment salah, security breach, atau batch job large scale.

Tanpa detection, kamu cuma tahu saat invoice datang. Dengan detection, kamu bisa respond dalam jam, saving ribuan dollar.

### Cost Anomaly Detection Hook

OpenClaw bisa monitor cost real-time via webhook dan alert pas ada abnormal pattern:

```bash
# Create webhook buat cost anomaly alerts
curl -X POST https://your-openclaw-gateway/webhooks \
  -H "Content-Type: application/json" \
  -d '{
    "name": "cost-alert-webhook",
    "url": "https://slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX",
    "events": ["billing:threshold", "cost:anomaly"],
    "thresholds": {
      "hourly_spike": 2.0,
      "daily_variance": 1.5
    }
  }'
```

### Program: Cost Anomaly Detector

**Wewenang:** Memantau cost cloud real-time, mendeteksi anomali, memicu alert dan remediasi

**Pemicu:** Pemeriksaan cost per jam via billing API + event webhook dari penyedia cloud

**Pintu persetujuan:** Alert pada anomali; persetujuan diperlukan untuk setiap remediasi (scale down, terminate instance)

**Eskalasi:** Memanggil engineer on-call jika cost spike kritis (>5x baseline)

### Execution Steps

1. **Baseline Establishment**
   - Calculate 30-day moving average per service
   - Establish hourly/daily baselines dengan confidence interval
   - Set thresholds (2x baseline = warning, 5x = critical)

2. **Real-Time Monitoring**
   - Query billing APIs setiap jam
   - Compare current cost vs baseline
   - Project spend vs budget
   - Correlate dengan deployment events

3. **Anomaly Response**
   - Alert di Slack dengan detail
   - Notify service owners
   - Create ticket buat tracking
   - Recommend action kalau cause identified

### Hook Implementation Example

```bash
# Create hook buat real-time cost monitoring
echo '#!/usr/bin/env node
const hook = {
  name: "cost-monitor",
  event: ["billing:hourly", "deployment:complete"],
  handler: async (ctx) => {
    // Get current spending
    const costs = await getHourlyCosts();
    const baseline = await getBaseline();
    const spikeRatio = costs.hourly / baseline.hourly;
    
    if (spikeRatio > 2) {
      // Major spike detected
      const deployments = await getRecentDeployments();
      
      await ctx.notify({
        channel: "#cost-alerts",
        message: `⚠️ Cost spike detected: $${costs.hourly}/hr (${spikeRatio.toFixed(1)}x baseline)`,
        metadata: {
          baseline: baseline.hourly,
          current: costs.hourly,
          deployments_last_hour: deployments.length,
          recommendation: deployments.length > 0 ? "Check deployment configs" : "Unknown cause"
        }
      });
    }
  }
};
' > ~/hooks/cost-monitor.js

openclaw hooks enable cost-monitor
```

### Investigation Flow

```markdown
## Cost Anomaly Detected

**Alert:** 2024-01-15 11:30 AM
**Service:** data-pipeline
**Current burn:** $54/hr
**Baseline:** $14/hr
**Spike factor:** 3.8x

**Cause:** 15 new instances launched by GitHub Actions workflow
- PR #1234 merge triggered
- Processing 2TB historical data
- Status: Running

**Assessment:**
- Estimated completion: 6 hours
- Total cost: $324
- Auto-termination: Yes

**Decision:** Expected behavior, approved by team lead.
**Action:** Monitor completion time.

Perbedaan antara "expected spike" dan "dangerous spike" kadang subtle. Context penting: deployment events, team communication, approval trails. Don't just look at cost numbers isolation.


---

## Kasus 4: Optimisasi Cost Model AI

### LLM Cost bisa Signifikan

Kalau tim kamu heavily menggunakan AI models, model costs jadi material line item. Claude Opus expensive tapi powerful. Haiku murah tapi less capable. OpenAI GPT-4 different pricing structure.

Route request ke model appropriate untuk task = penghematan signifikan. Bayangkan tim asisten: satu McKinsey (mahal, expert), satu fresh grad (murah). OpenClaw melakukan routing serupa untuk AI model.

### Model Failover & Cost Management

OpenClaw support model selection strategy:

```yaml
# Model fallback dengan cost optimization
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
      fallback:
        - "openai/gpt-4-turbo"      # 60% cost of Opus
        - "openai/gpt-4o"           # 50% cost
        - "openai/gpt-3.5-turbo"    # 10% cost
```

Routing cerdas:
- Opus untuk complex reasoning
- GPT-4 buat general coding
- GPT-4o buat creative work
- GPT-3.5 kalau Opus over-quota

```bash
openclaw models list --show-cost
openclaw models status --json
```

### Local Models buat Sensitive Data

Opsi model lokal untuk data sensitif. Run Ollama atau vLLM secara lokal, bayar sekali untuk hardware, zero API cost:

```yaml
# Ollama local model
agents:
  defaults:
    model:
      primary: "ollama/llama3.1"
    providers:
      ollama:
        url: "http://localhost:11434"

# vLLM untuk high-throughput
agents:
  defaults:
    model:
      primary: "vllm/Qwen2.5-7B-Instruct"
    providers:
      vllm:
        url: "http://localhost:8000"
```

Trade-off: usaha setup tinggi, throughput terbatas, tapi cost dihilangkan setelah infra setup.

### Cost Optimizer

```bash
# Auto-switch ke cheaper model buat routine work
openclaw config set agents.defaults.model.fast true
```


---

## Kasus 5: Governance Tag & Alokasi Cost

### Kenapa Tag itu Critical

Tanpa penandaan yang konsisten, kamu tidak bisa menjawab "Berapa sih pengeluaran tim Data per bulan?" atau "Manakah infrastruktur yang paling mahal?". Tanpa tracking, tidak ada akuntabilitas dan tidak bisa dioptimalkan.

Penandaan yang konsisten itu dasar untuk visibilitas finansial. Penandaan manual menyebalkan, jadi gunakan otomasi OpenClaw.

### Tag Governance Hook

```bash
# Create hook yang enforce tagging standard
echo '#!/usr/bin/env node
const hook = {
  name: "tag-enforcer",
  event: ["resource:create", "resource:update"],
  handler: async (ctx) => {
    const resource = ctx.event.resource;
    const requiredTags = ["environment", "team", "cost-center", "project", "owner"];
    const allowedEnvironments = ["production", "staging", "development", "sandbox"];
    
    // Check required tags
    const missing = requiredTags.filter(tag => !resource.tags[tag]);
    if (missing.length > 0) {
      await ctx.notify({
        channel: "#infra-alerts",
        message: `⚠️ Resource ${resource.id} missing tags: ${missing.join(", ")}`,
        severity: "warning"
      });
      return;
    }
    
    // Validate environment value
    if (!allowedEnvironments.includes(resource.tags.environment)) {
      await ctx.notify({
        channel: "#infra-alerts",
        message: `⚠️ Invalid environment tag: ${resource.tags.environment}`,
        severity: "error"
      });
    }
  }
};
' > ~/hooks/tag-enforcer.js

openclaw hooks enable tag-enforcer
```

### Program: Tag Governance

**Wewenang:** Menegaskan standar penandaan, memvalidasi akurasi alokasi cost, menghasilkan laporan kepatuhan

**Pemicu:** Event pembuatan resource + pemindaian haruan terjadwal

**Pintu persetujuan:** Memberi tahu dan eskalasi untuk pelanggaran; auto-delete resource dev/sandbox yang tidak ditandai setelah periode grace

**Eskalasi:** Melaporkan ke tim finance jika resource yang tidak ditandai menghabiskan >$1000/bulan

### Execution Steps

1. **Tag Validation**
   - Check required tags saat new resource create
   - Validate tag values (prod/staging/dev/sandbox)
   - Cross-check team tag
   - Auto-tag saat possible

2. **Compliance Scanning**
   - Daily scan buat untagged resources
   - Categorize violation: critical, medium, low
   - Calculate cost

3. **Enforcement Actions**
   - Notify owner
   - Set deadline (7 hari)
   - Auto-delete untagged dev/sandbox setelah 14 hari

### Monthly Cost Allocation Report

```bash
# Schedule monthly cost report generation
openclaw cron add \
  --name "Monthly Cost Allocation Report" \
  --cron "0 7 1 * *" \
  --session isolated \
  --message "Generate cost allocation report by team, project, environment"
```

### Sample Report Output

```markdown
## Monthly Cost Allocation Report (January 2024)

### By Team:
- Team Product: $53.800 (39%)
- Team Engineering: $46.400 (34%)
- Team Data: $33.500 (24%)
- Team Platform: $8.600 (6%)
- Unallocated: $3.200 (2%) ← needs investigation

### By Environment:
- Production: $81.500 (59%)
- Staging: $30.700 (22%)
- Development: $18.200 (13%)
- Sandbox: $12.900 (9%)

### By Cost Center (internal billing):
- CC-001 (Backend): $65.200 (47%)
- CC-002 (Data Pipeline): $43.200 (31%)
- CC-003 (Frontend): $18.900 (14%)
- CC-004 (Infra): $14.600 (11%)

### Untagged Resources: $3.200 (2%)
- Action: Tag enforcement improved compliance by 40% this month
- Outstanding: 12 resources masih untagged, owner di-notified

### Cost Optimization Opportunities:
- Dev environment schedule: $440/month savings
- RDS resize: $950/month savings
- Clean orphan snapshots: $1.100/month savings
- Total opportunity: $2.490/month
- Projected impact kalau semua diimplement: Budget 20% reduction

### Variance Analysis vs Budget:
- Budgeted: $138.000
- Actual: $145.800
- Variance: +$7.800 (+5.7%)
- Drivers: Unplanned load testing (Team Data +$5.2k), extra staging env (Team Eng +$2.6k)
```


---

## Kasus 6: Optimisasi Caching Prompt

### Context Window & Caching Trade-off

Modern LLM models support context windows 100K+ token. Tapi processing banyak context punya cost. Claude Opus 4.6 charge $3 per 1M input tokens tapi $15 per 1M cached read tokens.

Bayangkan menganalisis riwayat percakapan pelanggan (5000 token) secara berulang:
1. **Tanpa caching:** 10 panggilan = 50k token × $3 = $150
2. **Dengan caching:** Panggilan pertama $15 + 9 panggilan cache $27 = $42

Cache menghemat $108 (72% penghematan!). Tradeoff: panggilan pertama lebih lambat, panggilan berikutnya lebih cepat.

### Cache Configuration

```yaml
# Konfigurasi prompt caching buat efficiency
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"  # "none" | "short" | "long"
  
  # Heartbeat untuk menjaga cache tetap hangat saat idle period
  heartbeat:
    every: "55m"
```

`cacheRetention: "long"` berarti cache tetap di server hingga 5 menit. `heartbeat` setiap 55 menit menjaga cache tetap hidup.

### Cache Strategy per Workload

Beberapa workload memiliki kebutuhan cache yang berbeda:

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"  # default
  
  # Research agent: benefit dari long cache
  research:
    default: true
    heartbeat:
      every: "55m"
  
  # Alert agent: burst traffic, short cache lebih efficient
  alerts:
    params:
      cacheRetention: "short"
  
  # Sensitive data: prefer local models, no caching
  sensitive:
    model:
      primary: "ollama/llama3.1"
    params:
      cacheRetention: "none"
```


---

## Anti-Patterns: Jangan Lakukan Ini!

### 1. Cutting Corners pada Reliability buat Save Cost

Cost optimization tidak boleh sacrifice reliability dan uptime. Ini gak bisnis decision yang baik: kamu save $500/month tapi downtime cost kamu $50k/month dalam lost revenue.

**SALAH:**
```bash
# Nonaktifkan Multi-AZ untuk database produksi, hemat $400/bulan
openclaw cron add \
  --name "Nonaktifkan HA" \
  --message "Nonaktifkan Multi-AZ production-db untuk hemat $400/bulan"
```

**BENAR:**
```markdown
## Optimasi Cost dengan Batasan Reliability

**Prinsip:** Optimasi cost menjadi prioritas sekunder terhadap reliability

**Tingkat reliability:**
- Produksi: 99.95% SLA → Multi-AZ diperlukan, tidak ada single-point-of-failure
- Staging: 99.0% SLA → Multi-AZ opsional, bisa toleransi downtime singkat
- Development: 95% SLA → Single zone OK, optimasi cost agresif

**Strategi Optimasi Cost per Tingkat:**
- Produksi: Fokus pada right-sizing instance, reserved instances, tidak mengurangi availability
- Staging: Aman untuk mengoptimasi instance type, adjust auto-scaling agresif
- Development: Aman untuk shut down saat tidak digunakan, lifecycle berbasis jadwal
```

### 2. Optimizing tanpa Business Context

Optimasi cost harus sadar terhadap kalender bisnis. Jangan resize database produksi di tengah penutupan akhir bulan, jangan delete environment staging saat tim sedang testing intensif.

**Integrasi kalender:**
- Akhir bulan: Freeze perubahan produksi
- Musim puncak: Jeda optimasi
- Musim sepi: Optimasi agresif
- Jendela maintenance: Hanya perubahan yang ramah

### 3. Deleting tanpa Grace Period

Developer ya develop terus, dan kadang instance di-create terus lupa di-terminate. Tapi kadang gak sengaja ke-delete resource yang sebenernya lagi di-pake (long-running process di server yang kelihatannya idle). Jangan pernah delete immediately. Selalu ada notification period.

**SALAH:**
```bash
# Hapus seketika
openclaw cron add --message "Delete idle instance immediately"
```

**BENAR:**
```markdown
## Proses Pembersihan Resource yang Graceful

**Langkah 1 (Hari 0):** Deteksi resource idle 14 hari, tanda untuk penghapusan, beri tahu pemilik
**Langkah 2 (Hari 7):** Ingatkan pemilik, tetapkan deadline 7 hari lagi, eskalasi ke manager
**Langkah 3 (Hari 14):** Peringatan terakhir, menyebutkan akan dihapus dalam 3 hari kecuali diperpanjang
**Langkah 4 (Hari 17):** Hapus setelah periode grace selesai

**Oksi perpanjangan:**
- Pemilik bisa meminta perpanjangan dengan justifikasi bisnis
- Perpanjangan bisa diberikan hingga 2x (proteksi maksimal 28 hari)
- Auto-renewal jika resource tiba-tiba digunakan lagi

**Audit trail:** Semua penghapusan tercatat, dapat ditelusur kembali ke pemilik resource
```

### 4. Ignoring Model-Specific Cost Nuances

Model AI yang berbeda memiliki struktur cost yang berbeda. Beberapa mengenakan biaya per token, beberapa per menit, beberapa bulanan tetap. Mix-and-match tanpa pemahaman = overrun anggaran.

**BENAR:**
```yaml
# Pemilihan model tiered dengan kesadaran cost
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
      fallback:
        - "openai/gpt-4-turbo"     # 60% cost
        - "openai/gpt-4o"          # 50% cost
        - "openai/gpt-3.5-turbo"   # 10% cost
```

---

## Checklist: Agentic FinOps Implementation

Sebelum launch agentic FinOps system, ensure ini tercover:

**Foundation:**
- [ ] Akses API billing cloud dikonfigurasi
- [ ] Cost baseline ditetapkan (30-day moving average)
- [ ] Threshold alert budget ditetapkan
- [ ] Tag alokasi cost didefinisikan
- [ ] Tim dilatih tentang penandaan

**Automation:**
- [ ] Cron job deteksi waste mingguan
- [ ] Monitoring anomali cost per jam
- [ ] Analisis RI bulanan terjadwal
- [ ] Pemindaian kepatuhan tag harian
- [ ] Pelaporan alokasi cost diotomatisasi

**Safety Gates:**
- [ ] Tindakan destruktif memerlukan persetujuan manual
- [ ] Periode grace (7 hari min) sebelum penghapusan
- [ ] Template notifikasi Slack terstandarisasi
- [ ] Path eskalasi didefinisikan
- [ ] Audit trail dipertahankan

**Intelligence:**
- [ ] Cost model terintegrasi dengan kalender bisnis
- [ ] Pola musiman dikenali
- [ ] Event deployment dikorelasikan dengan cost spike
- [ ] Rekomendasi RI divalidasi

**Monitoring:**
- [ ] Dashboard cost dapat diakses
- [ ] Laporan alokasi cost bulanan
- [ ] Variance cost vs budget dipantau
- [ ] Realisasi penghematan dipantau

**Model Optimization:**
- [ ] Tier cost model didokumentasikan
- [ ] Chain fallback dikonfigurasi
- [ ] Strategi cache disesuaikan
- [ ] Opsi model lokal dievaluasi
- [ ] Tracking penggunaan disiapkan

---

## Key Takeaways

1. **Agentic FinOps** buat closed-loop system: detect → analyze → act → learn
2. **Autonomous waste detection** identify dan fix 20-30% cost waste
3. **Reserved Instance planning** perlu data-driven analysis untuk ROI maksimal
4. **Real-time anomaly detection** catch cost spikes sebelum jadi disaster
5. **Intelligent model selection** route ke appropriate model, save cost
6. **Consistent tagging** prerequisite buat cost allocation visibility
7. **Prompt caching** save dramatic cost (70%+ untuk repeated context)
8. **Grace period** enforced sebelum deletion, human di loop buat critical decision
9. **Business context** essential. Jangan optimize pas peak season
10. **Reliability tier** harus di-respect. Jangan sacrifice uptime buat cost
11. **Approval flow** required sebelum major action
12. **Tracking & audit** essential buat learning improvement


---

