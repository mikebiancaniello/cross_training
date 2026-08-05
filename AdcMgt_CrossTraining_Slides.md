# AdcMgt Cross-Training

## Slide Outline

*(companion to AdcMgt_CrossTraining.md — full detail there; this is presentation-ready bullets)*

---

### Slide 1 — Title

- AdcMgt (x_snc_adcmgt): Managing our ADCs
- Presenter: Mike Biancaniello · Q3 Cross-Training

### Slide 2 — What is it

- Manages ADCs (customer-facing load balancers)
- Migrated off dedicated F5 hardware → containerized, implementation-agnostic
- Three generations: ADCv2 (containers), ADCv3 "Osprey" (Ansible), ADCv4/STM (Kubernetes/"Snowsk8s")
- Core object: ADC Cluster = 4 containers, 2 per site, one shared config

### Slide 3 — Key vocabulary

- adcconfig / adchost / Deployment Group (static, pooled, k8s)
- Running Config vs. Candidate Config
- Commit = applying candidate → running (JunOS-inspired term)

### Slide 4 — Why we need it

- One data model, swappable backends (Ansible / Kubernetes) with no API change
- Declarative + idempotent → prevents config drift
- Atomic commits — all or nothing
- Deliberately NOT touched by Discovery ("No Disco")
- Driving today's work: ADC Resiliency Initiative (see Phased Commit, slide 8)

### Slide 5 — How it works: the commit lifecycle

- Candidate Config → Commit → Running Config
- Manual: "Commit Config" UI action → `API.commitConfig()`, tracked via ChgMgt
- Automated: scheduled job sweeps changed clusters, commits in batches
- Ansible bridge via NCM app; pause/throttle/alert automation parameters protect the pipeline

### Slide 6 — How it works: the engine

- `API` = public facade (adds change tracking)
- `AdcMgtUtils` = ~5,700-line implementation layer underneath
- Supporting classes: AdcConstants, AdcMgtErrors, ChgMgt, AutomationParameters
- (Analogy for EAO folks: same two-layer shape as EaTaskUtil)

### Slide 7 — How it works: Adcmon & AMA (two different monitoring layers)

- **Adcmon** = deployment config generation (event-driven business rules → `AdcmonUtils`)
- **AMA** = Automated Monitoring & Alerting (scheduler → probes → threshold evaluators → dispatcher → PagerDuty/Teams/email)
- Currently migrating V1→V2 — live example of this app evolving its own tooling

### Slide 8 — Live example: Phased Commit redesign

- Problem: today, commits hit both sites at once — bad config = rollback both sites
- Fix: commit site A → validate → commit site B → validate, with rollback gates
- New: `running_config`, `under_maintenance`, `phased_commit` fields + Maintenance Tracking table
- Server-enforced: one maintenance session per ADC at a time
- Already partially shipped (PhasedCommitAPI, new UI actions)

### Slide 9 — How customers/automations interact with it

- Customers directly: mTLS Datacenter API, Instance Settings API
- Customer instances: polling pattern (Custom URL app)
- Internal automations: ~85 scripts, 15+ scopes read/write our tables directly (biggest: VSF provisioning)
- Operators: Commit/Pause/Cost In-Out/Maintenance/Deprovision UI actions
- Scheduled jobs: auto-commit, un-stickers, HUP queue, IPv6 provisioning

### Slide 10 — How I interact with it as an engineer

- sys_script_include (logic) / sys_script (business rules) / sys_ui_action (operator buttons) / sysauto_script (scheduled jobs)
- New path: `sn-sdk-adcmgt` — Fluent/flows-as-code SDK project (TypeScript, now-sdk CLI)
- CHG-gated changes, update-set promotion dev → test → prod

### Slide 11 — Troubleshooting & support (the honest gap)

- What exists: global pause switch, per-cluster pause/unpause, auto-pause after repeated failures, AdcMgtErrors taxonomy, `loggerid` for log correlation
- Rollback failures today = manual debugging, no automated recovery
- **Gap**: no real operator runbook exists yet (`Notes/AdcMgt Operator Guide.md` is a stub)
- Model to copy: Custom URL app's troubleshooting FAQ

### Slide 12 — How we know it's broken

- AMA alerts (commit error volume, threshold-based)
- Auto-pause + alert after N consecutive failed commits
- Self-throttling as a symptom of Ansible Tower/network trouble
- Coming: dashboards for stuck maintenance & config-version drift
- Gap: no dashboards/Splunk searches/SLOs documented yet

### Slide 13 — Where to find docs

- Table reference map, User Guide, Phased Commit HLD + JS API HLD
- CNS DocHub (internal portal)
- (List — see companion doc §8 for full table)

### Slide 14 — Where to get help

- Owning team: Network Systems Development – ADC Dev
- On-call network engineer (escalation for stuck maintenance/rollback)
- cs-integration support group (customer-facing issues)
- Local dev/repro: code.devsnc.com git mirror

### Slide 15 — Closing / Q&A

- ~76 tables, ~85 external consumer scripts, 15+ scopes depend on us
- Open call to action: who wants to help write the ops runbook?
- Questions?
