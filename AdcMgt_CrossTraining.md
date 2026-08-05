# AdcMgt Cross-Training — Brown Bag (30–60 min)

**Presenter:** Mike Biancaniello
**App:** AdcMgt (`x_snc_adcmgt`) — ADC (Application Delivery Controller / customer-facing load balancer) management
**Audience:** Team members moving into the flattened ADC / Snowsk8s / AI pillar structure, who may not have touched this app before

---

## 1. What is it

AdcMgt manages **ADCs** — the load balancers that front customer instances. It grew out of a migration off dedicated **F5 hardware** onto a **containerized, implementation-agnostic** architecture, so the same data model can drive different backends over time:

- **ADCv2** — legacy container-based
- **ADCv3 ("Osprey")** — Ansible-driven deployment (tables use `_v3` suffix)
- **ADCv4 / STM** — Kubernetes-based ("Snowsk8s" in code)

Core object: an **ADC Cluster** = 4 containers, 2 per site (primary/standby VIPs per site), sharing one generated config.

Key vocabulary:

| Term | Meaning |
| --- | --- |
| **adcconfig** | Implementation-agnostic JSON representation of a cluster's config |
| **adchost** | Linux server hosting ADC containers |
| **Deployment Group** | Group of adchosts — `static` (all), `pooled` (one per pool), or `k8s` |
| **Running Config** | The config actually live on the ADCs right now |
| **Candidate Config** | The desired config, built from live GlideRecord data |
| **Commit** | Applying the Candidate Config so it becomes the Running Config (JunOS-inspired term) |

---

## 2. Why do we need it

- Lets the platform swap backend implementations (Ansible, Kubernetes) **without changing the front-end API or data model** — one config type, multiple deployment targets.
- Declarative + idempotent by design: prevents **config drift** between "actual" and "expected" state.
- Commits are **atomic** — the whole config succeeds or fails as one unit, no partial application.
- Explicitly **not** touched by Discovery — AdcMgt tables are treated as the single authoritative source of intent ("No Disco" design goal). Discovery writing to these tables would break things.
- Newer driver: **ADC Resiliency Initiative** — today a commit deploys to both sites simultaneously, so a bad config means rolling back both sites at once. The in-flight **Phased Commit** redesign (see §3) fixes this.

---

## 3. How does it work

### Commit lifecycle

`Candidate Config → Commit → Running Config`. Editing ADC records marks the cluster `is_changed`. A commit snapshots the config tables into a JSON `adcconfig`; on success that becomes the Running Config.

- **Manual commit**: UI Action *Commit Config* → `new x_snc_adcmgt.API().commitConfig(clusterId)`, wrapped in Change Management tracking (`ChgMgt`); blocked if a job is already running.
- **Automated commit**: scheduled job *Commit Changed ADC Configs* sweeps changed/non-paused/non-maintenance-locked ADCv3 clusters and commits them (batch limit via automation parameter, default 25); legacy ADCv2 path pushes via `AdcMgtUtils.pushChangedClusterConfigsToNetwork()`.
- **Stuck-state recovery**: *ADCv2/ADCv3 Commit Un-Sticker* scheduled jobs detect and clear stuck commit states.
- **Ansible integration** (ADCv2/v3): via **Ansible Tower API through the NCM app** (`x_snc_ncm.AnsibleControl`). Key automation parameters: poll interval/timeout, `ansible.pause.global` (platform-wide kill switch — queues instead of failing), commit throttling after repeated Tower timeouts (default: 5 timeouts → throttle to 5 concurrent jobs), and failure alerting (`failed.job.limit`, default 5 → cluster auto-paused + alert raised via `PauseUtils`).

### The engine (two layers, like `EaTaskUtil` for EAO)

- **`x_snc_adcmgt.API`** — the public facade. Wraps core operations (`commitConfig`, `provisionVipCluster`, `deprovisionVipClusterById`, etc.) with automatic Change Management tracking. This is the one entry point both UI Actions and REST methods call into.
- **`AdcMgtUtils`** — the ~5,700-line implementation layer underneath `API`, doing the actual GlideRecord work, legacy network pushes, and orchestrating `SettingsUtils`, `IPUtils`, `DeploymentUtils`, `Snowsk8sUtils`, and the Ansible bridge.
- Supporting classes worth knowing by name: `AdcConstants` (table/enum registry, ~76 tables), `AdcMgtErrors` (structured error taxonomy), `ChgMgt` (change correlation), `AutomationParameters`.

### Adcmon — the monitoring/deployment-config layer

`AdcmonUtils` generates per-container deployment configs (schema 3 = "in-rack Adcmon", schema 4 = "pop site L4TR — Layer 4 Traffic Router") and computes VIP routing cost/metrics. Regeneration is **event-driven**: many business rules fire on entity changes (cost change, host change, healthcheck change, container deploy) and each calls back into `AdcmonUtils.updateConfigContainers()` — there's no single central "build" call.

### Phased Commit — the current in-flight redesign (great live example)

- **Problem**: today, commits hit both sites at once; a bad config means a full rollback + re-validation across both sites.
- **Solution**: opt-in workflow (human CHGs only — automated commits unaffected) that commits site A, validates, then proceeds to site B, with explicit rollback gates at each stage.
- New fields: `running_config` (per-container reference, enables drift detection), `under_maintenance` (blocks automation while a maintenance session is active), `phased_commit` (suppresses auto-completion until the operator explicitly finishes).
- New table: **Maintenance Tracking** (CHG ticket, initiator, start/end, status: in_progress/completed/rolled_back/cancelled/stuck, error journal).
- Server-side maintenance locking — only one active session per ADC; a second operator is rejected and shown who currently owns the session.
- State machine: `Ready → UnderMaintenance → CommitInProgress → SiteA Executing/Validate → SiteB Executing/Validate → CommitCompleted → Ready`, with rollback branches at every validation gate.
- Already partially shipped: `PhasedCommitAPI.js`, and UI Actions *Commit Config (Phased)*, *Manage Phased Commit*, *Proceed with Phase*.

### REST API surface

RPC-style endpoints under `/api/x_snc_adcmgt/v1/...`:

- `iputils/rpc_internal/{method}` — VIP provisioning in IPAM
- `clustermgt/rpc_internal/{method}` — `commit_config`, `cost_in`/`cost_out`, `get_config`, `pause_commits`/`unpause_commits`, `validate_config`, etc.

Standard envelope: `{uri, params, error, result}`; errors include a `loggerid` specifically for support log correlation.

---

## 4. How do customers and/or automations interact with it

- **Directly by customers**: the mTLS Datacenter API and Instance Settings API are customer-facing.
- **Customer-instance automation**: adjacent Custom URL app has customer instances polling a job by ID until `Completed`, authenticating via Basic Auth against `datacenter.service-now.com`.
- **Internal automations (the biggest surface)**: ~85 scripts across 15+ other scopes read/write AdcMgt tables directly (not via REST) — heaviest consumer is `x_snc_vsf` (instance provisioning/retake-to-pool flows), plus `x_snc_tea`, `x_snc_shaas`, `x_snc_ncs`, `x_snc_ipam`, `x_snc_rgm`. AdcMgt is a depended-upon platform service, not an isolated app.
- **Operators (human)**: UI Actions on ADC/Cluster records — Commit Config, Pause/unPause Commits, Cost In/Out, Start/End Maintenance, phased-commit actions, Deprovision, re-gen Adcmon config, Launch/Re-poll Ansible.
- **Scheduled automation with no human in the loop**: auto-commit changed clusters, stale-config commits, commit un-stickers, HUP queue drain, proactive IPv6 VIP provisioning, unpause-on-schedule.

---

## 5. How do I as an engineer interact with it

- **Source layout** (`x_snc_adcmgt`): `sys_script_include/` (272 files — where almost all real logic lives), `sys_script/` (~254 business rules, mostly validation/cascade/sync-on-save), `sys_ui_action/` (~88 operator-facing actions), `sysauto_script/` (7 scheduled jobs), `sp_widget/` (Service Portal UI, e.g. the Phased Commit Manager), plus QUnit test suites (real testing discipline exists here).
- **"Flows as code" SDK project**: `sn-sdk-adcmgt` — package `x_snc_adcmgt_flows`, built on `@servicenow/sdk` + `@servicenow/glide`, TypeScript, `now-sdk` CLI (`build`/`deploy`/`transform`/`types`). Source organized under `src/fluent/{script-includes,records,ui-actions,scripts,actions,flows,widgets}`. This is the modern path for building new automation (Fluent subflows) rather than hand-editing server scripts directly in the instance UI.
- **Change discipline**: production changes go through CHG tickets; `ChgMgt` auto-correlates changes made via `API`. Phased Commit will soon *require* a CHG number for manual commits.
- **Release process**: standard ServiceNow update sets promoted dev → test → production.

---

## 6. How do we troubleshoot and support it

**Honest gap to call out live**: the one doc meant to cover this (`Notes/AdcMgt Operator Guide.md`) is currently a 4-line placeholder — literally a note-to-self saying "we need a real ops KB, e.g. 'if there's a problem and we need to stop commits globally, do this.'** This is a real, open gap for the team, not something hidden — good candidate for a follow-up doc sprint.

What does exist today:

- **Stop commits globally**: `ansible.pause.global` automation parameter (queues changes instead of failing). Per-cluster: `Pause Commits`/`unPause Commits` UI actions, or REST `pause_commits`/`unpause_commits`/`is_paused`.
- **"Why is this cluster paused?"**: check the cluster's pause reason via `PauseUtils` — type `auto` (system-triggered after repeated failures), `manual/maintenance`, or `manual/permanent`.
- **Error taxonomy for triage** (`AdcMgtErrors`): `Conflict` (409, job already running), `Paused`, `ConfigValidationError`, `RollbackError` ("something probably needs manual cleanup"), `MissingData`, `BadArgument`.
- **`loggerid`** in every RPC error response — use it to correlate logs during an investigation.
- **Rollback failures are explicitly manual today** — if a rollback fails, the engineer doing the maintenance has to debug and remediate by hand; this is called out as unsolved/out of scope in the Phased Commit design.
- **Good model to borrow**: the adjacent Custom URL (`x_snc_curl`) troubleshooting FAQ is a much richer example of what an AdcMgt ops runbook should look like (job→action→error-message chains, common cert/DNS failure modes, local repro steps against `code.devsnc.com` git mirrors). Worth pointing the audience at it as a template.

---

## 7. How do we know it's not working properly

- **AMA subsystem** is the primary signal source: commit-error-volume probes evaluated against per-monitor warning/critical/failure thresholds, dispatched to PagerDuty-style webhook / Teams / email.
- **Auto-pause + alert**: a cluster that fails commits repeatedly (default: 5 in a row) is automatically paused and an alert is raised — a very concrete "something's wrong" signal.
- **Commit throttling as a symptom**: if you see the platform throttling itself (5 sequential Ansible Tower timeouts by default), that's itself a sign of Tower/network health issues worth escalating.
- **Planned additions (Phased Commit)**: dashboard for ADCs stuck `under_maintenance`, and for containers whose `running_config` versions have drifted apart; Stuck Maintenance Detector and an upgraded Commit Un-Sticker job to catch stuck phased commits.
- **Gap**: no existing dashboard links, Splunk saved searches, or documented SLA/SLOs were found in the source material — another candidate for the same doc-gap conversation as §6.

---

## 8. Where are documents for it

| Doc | Path |
| --- | --- |
| Table reference map | `Documentation/x_snc_adcmgt_table_references.md` |
| User Guide | `Documentation/AdcMgt User Guide.html` |
| Operator Guide (stub — needs writing) | `Notes/AdcMgt Operator Guide.md` |
| Phased ADC Commit HLD | `Documentation/Phased ADC Commit.md` |
| Phased ADC Commit JS API HLD | `Documentation/API_HLD_Phased_ADC_Commit_JS_API.md` |
| Phased commit engineering notes | `Notes/pacmgt.md`, `Notes/Phased Commit API.md` |
| ADC Builder UX backlog | `Notes/ADC Builder.md` |
| Cert Management notes (adjacent) | `Notes/Cert Management.md` |
| Custom URL troubleshooting FAQ (adjacent app, good model doc) | `Documentation/Custom Url Troubleshooting F.A.Q.wiki` |
| Automation Parameters deep-dives | `Documentation/AutomationParameters_Analysis.md`, `AutomationParameters_Comprehensive.md`, `AutomationParameters_QuickFix.md` |
| QUnit testing practices | `Documentation/QUnit_Testing_Best_Practices.md`, `GlideUtilsTesting.md` |
| Internal doc portal | CNS DocHub — `https://cns.pages.servicenow.net/documents/cns_doc` |

Referenced-but-not-found in this workspace: STM (ADCv4) HLD, original ADCv2/ADCv3 ART documents, "AdcMgt FAQ" — worth tracking down from the team if needed.

---

## 9. Where can I go to get more help

- **Owning team**: Network Systems Development – ADC Dev (owns full design + deployment).
- **Doc author for the newest design work**: Mike Biancaniello (Phased Commit HLD + JS API HLD).
- **On-call network engineer** — escalation point for stuck maintenance / rollback failures / phased-commit validation decisions (exact paging mechanism not documented — worth capturing in the ops runbook).
- **cs-integration support group** — for customer-facing issues on the adjacent Custom URL app.
- **"Contact CNS"** — the org this app lives under (Cloud/Core Network Systems).
- **Git mirror for local dev/repro**: `code.devsnc.com` (e.g. `dev/adcmgt.git`) — outside-the-instance clone used for local debugging.

---

## Closing note for the session

This app is bigger and more depended-upon than it looks from the outside — ~76 tables, ~85 external consumer scripts across 15+ scopes, and it's in the middle of a real architectural shift (Phased Commit) that's a good lens for explaining the whole commit lifecycle. The one thing genuinely missing is an operator/troubleshooting runbook — a good ask-the-room moment: who wants to help write that, modeled on the Custom URL FAQ.
