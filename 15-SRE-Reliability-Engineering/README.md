# Module 15 — SRE Practice: SLOs, Incidents, DR, Chaos, Capacity, Cost (Weeks 21–22)

> **JD coverage:** "define and enforce SLOs, SLIs, error budgets, and reliability standards" (slice Staff); "incident management, on-call excellence, postmortems" (slice Staff/SRE-3); "DR strategies with clearly articulated RPO/RTO aligned to business SLAs" + "one-click DR drill execution" (slice SRE-3/Staff); "capacity planning, performance engineering, and cost optimization at scale" (slice Staff); "toil reduction" (all). This module converts everything you've built into an *operated system*.
> Templates: `assets/templates/` (SLO.yaml, POSTMORTEM.md, RUNBOOK.md, CHANGE-REQUEST.md).

---

## Part A — SLIs, SLOs, error budgets

### EXERCISE 15.1 — Define NorthPay's SLOs [I] [90 min]

**STEPS**

1. Catalog user journeys (not services!): `browse web`, `place order`, `settlement batch`, `search transactions`. For each: what does "working" mean to the user?
2. Write SLI specs (use the SLO template):

- Availability SLI for `place order`: `sum(rate(http_requests_total{route="/api/orders",method="POST",status!~"5.."}[5m])) / sum(rate(...))` over rolling windows.
- Latency SLI: fraction of requests < 400ms via histogram buckets: `sum(rate(..._bucket{le="0.4"}))/sum(rate(..._count))`.
- Freshness SLI for settlement batch: `time() - max(settlement_last_success_timestamp)` < 2h.
- Quality note: why ratio-of-good-events beats uptime-percent (probes lie; user events don't).

3. Set SLO targets by journey criticality: order placement 99.9% (43min/mo budget), browse 99.5%, settlement 99% with freshness. **Justify why not 99.99% everywhere** (each 9 costs ~10× effort; match business pain, write the cost-of-a-9 table).
4. Implement in Prometheus: recording rules for SLIs; Grafana SLO panels (30d rolling compliance + budget remaining %); burn-rate alerts already exist from 11.4 — point them at these SLIs.
5. Error-budget policy (the document): when budget < 25% remaining → freeze feature deploys, reliability work only, escalation to eng leads; when exhausted → full freeze + exec notification. **Get the fictional org's "CTO" (your alter ego) to sign it — practice the conversation by writing the one-page proposal.** This policy is a real artifact Staff interviews ask about.
6. Window choices: rolling 30d vs calendar month (alerting fairness discussion); multi-window burn rates recap.
**VALIDATE:** SLO.yaml per journey committed; dashboards live; burn-rate page fires on synthetic error storm; policy doc written.

### EXERCISE 15.2 — SLIs for everything else (dependencies included) [I] [60 min]

**STEPS**

1. Dependency SLOs: RDS (successful connections/queries ratio), SQS (enqueue/dequeue success, oldest-message age), redis (hit ratio + evictions), ES (search success + p95), Mongo (operation success + replication lag).
2. Composite reasoning: api SLO 99.9% while depending on DB at 99.95% — serial-dependency math (0.999×0.9995...); when a dependency's SLO *is* your ceiling.
3. Blackbox/external SLO (probe-based) vs internal event-based — where they disagree during a real brown-out (retries masking user pain) and which one you page on.
4. Write the "SLO of our SLOs" review: quarterly — which SLOs were useless (never close to breach = maybe too lax or measuring wrong thing) vs noisy.
**VALIDATE:** dependency SLO dashboard exists; serial-dependency math written for the checkout path.

---

## Part B — Incident management (the craft)

### EXERCISE 15.3 — Incident command & communication drills [I] [half day, then ongoing]

**STEPS**

1. Adopt a severity taxonomy: SEV1 (user-facing outage/security), SEV2 (degraded/partial), SEV3 (internal/no user impact), SEV4 (cosmetic). Map alerting severities to them.
2. Roles: Incident Commander (coordinates, doesn't debug), Ops Lead (debugs/changes), Comms Lead (stakeholders), Scribe (timeline). Solo-lab variant: you rotate hats deliberately and *say the transitions out loud* — the muscle is the discipline, not the headcount.
3. Drill (use a chaos trigger from Part D): page fires → acknowledge (start incident log from `npctl incident`) → declare SEV → assign roles → **status updates every 15 min even if "no change"** (write them — customers/comms practice) → mitigate → resolve → schedule postmortem.
4. Status page update per phase; internal vs external comms tone (write both for the same incident — external: no jargon, no blame, next-update time promised).
5. Escalation design: when to wake a DBA/Network/Dev (write the matrix: symptom → first responder action → escalation trigger + contact path). Your Module 02-14 runbooks get linked into it.
6. Handover drill: mid-incident "shift change" — write a handover note good enough that fresh-you could continue (follow-the-sun model from the Deel JD).
7. Tooling: incident.io/FireHydrant/PagerDuty concepts; NorthPay uses GitHub Issues + a `#incident` convention + timeline bot (`npctl incident` writes timestamps) — free and sufficient.
**VALIDATE:** two full drills executed with complete timelines + comms artifacts; escalation matrix committed.

### EXERCISE 15.4 — Blameless postmortems that change things [I] [ongoing; 3 full ones required]

**STEPS**

1. Use `assets/templates/POSTMORTEM.md`: summary, impact (quantified — users, minutes, SLO burn), timeline (from incident log), root cause (techniques below), what-went-well, action items with owners+dates.
2. Root-cause techniques practice (one per postmortem): 5-Whys; fishbone on a multi-factor incident; "contributing conditions vs trigger" split. **Ban the phrase "human error" as a cause** — the human operated a system that made the error easy; find *that*.
3. Write 3 full postmortems from real lab incidents this program generated (you have material: the OOMKill loop, the non-concurrent index lock, a deploy regression). Add the evidence links (dashboards, traces).
4. Action-item discipline: each item = ticket in the simulation board, with a verification method ("add alert X" is weak; "alert X fires within 2m of condition Y, tested" is strong). Track closure rate in your weekly retro.
5. Read 2 public postmortems (Cloudflare, AWS post-event summaries) and write a one-paragraph critique of each — calibrate your bar.
**VALIDATE:** 3 postmortems pass the "6-months-later stranger can understand it" test; action items closed with verification evidence.

### EXERCISE 15.5 — On-call simulation week [A] [1 week, overlaid]

**STEPS**

1. You are primary on-call for NorthPay for 7 days: alert routing to your phone/email; a chaos cron (`assets/scripts/chaos/random-fault.sh`) fires 1–2 faults per day at random times (including one at night — configurable mercy mode).
2. Per page: ack ≤ 5 min, triage, mitigate, timeline, postmortem if SEV1/2. Log MTTA/MTTR per incident.
3. End-of-week review: alert quality stats (pages, actionable %, false-positive %), toil list, and one automation implemented from the week's pain.
4. Sustainable on-call essay: rotations, handoffs, alert hygiene, compensation/culture — one page (interviewers probe on-call philosophy).
**VALIDATE:** week report with stats; false-positive rate reduced week-over-week.

---

## Part C — Disaster Recovery

### EXERCISE 15.6 — DR strategy design: RPO/RTO engineering [A] [90 min]

**STEPS**

1. Define tiers per journey: settlement DB RPO≤5min/RTO≤1h (Tier-1); app tier RPO=0(stateless)/RTO≤15min; analytics RPO=24h/RTO=next-day. Map to patterns: backup/restore (pilot light money) → pilot light → warm standby → active-active (write the cost curve).
2. NorthPay DR plan: **warm standby** in ap-southeast-1 — EKS cluster exists (0–1 nodes, Karpenter scales on failover), Aurora cross-region read replica, S3 CRR (done in 3.10), ECR replication (done in 4.8), Route53/Cloudflare failover (13.7/14.10), secrets replicated, Terraform per-region state ready.
3. Dependency inventory: list every hidden region-lock (ACM certs are regional! AMIs! SSM parameters!) — the classic DR-plan killer is the forgotten regional resource; your inventory is the fix.
4. Failure-mode table: AZ loss vs region loss vs AWS-account loss vs DNS/provider loss — different playbooks for each (region loss ≠ account compromise response).
**VALIDATE:** DR design doc with per-tier RPO/RTO + the regional-dependency inventory.

### EXERCISE 15.7 — One-click DR drill (slice Staff verbatim) [E] [1 day]

**GOAL:** `assets/scripts/dr/dr_drill.sh` (you build it) executes a full regional failover rehearsal unattended, with validation and a report.
**STEPS**

1. Script phases: (1) pre-checks (replica lag ~0, images present in DR ECR, secrets synced); (2) promote Aurora replica (timed); (3) terraform apply DR region app stack (pre-baked plan); (4) scale EKS DR, deploy apps via ArgoCD (DR ApplicationSet); (5) smoke tests (np-deploy-check against DR endpoints); (6) DNS weight flip (drill mode: a test record, not prod); (7) measure: RTO actual, data-loss check vs RPO target; (8) failback plan documented; (9) report generation (markdown with timings, auto-committed).
2. Run the drill for real. First run will fail somewhere — that's the point; fix and rerun until clean. **Postmortem the drill itself.**
3. Tabletop variants (write + walk through with a timer): region loss during settlement batch; DR region also degraded; someone fat-fingers the failover mid-drill.
4. Drill cadence policy: quarterly full, monthly component (DB promote only), per-release pre-flight checks; drills are change-managed events with windows (they *can* hurt prod if careless).
5. Cost note: warm-standby steady-state cost (~\$X/mo — compute yours) vs RTO promise — the business tradeoff conversation, written.
**VALIDATE:** clean drill run with measured RTO ≤ target and evidence; drill postmortem #1; cadence policy committed.

---

## Part D — Chaos engineering (breaking on purpose, scientifically)

### EXERCISE 15.8 — Controlled fault injection [A] [half day]

**STEPS**

1. Tooling: install Chaos Mesh on kind (or use `assets/scripts/chaos/` — tc/netem, stress, kill scripts for EC2/EKS). Principles first (write them): hypothesis → blast radius → steady-state metric → abort condition.
2. Experiment catalog (run each with a hypothesis + SLO dashboard watching):

- pod kill (api leader... there is none — stateless; note what changes when stateful)
- AZ simulation (cordon all nodes in one "AZ" label on kind / stop instances in one AZ on EKS)
- network latency +200ms api↔db (netem) — watch connection pools, p95s, timeout cascades; then fix with proper timeouts/retries (retry budget: max 2 retries + jitter — observe retry amplification without budgets)
- DNS delay (CoreDNS chaos) — remember gauntlet #6, now with SLOs burning
- disk pressure on a node (fallocate flood) — eviction behavior
- certificate expiry (short-lived cert + blackbox alert — validate that alert!)

3. GameDay: combine two faults (latency + pod kills during a deploy) — the realistic composite; run the full incident process from 15.3 around it.
4. Chaos in CI/CD: pre-prod chaos gate concept (deploy to staging under latency fault — does the canary analysis catch degradation?).
5. Write `runbooks/chaos-playbook.md`: experiment template, safety checklist (abort conditions, prod-off-limits-until-mature), findings log. Each finding becomes a reliability ticket — **chaos without ticket follow-through is vandalism.**
**VALIDATE:** 6 experiments with hypothesis/result/fix each; one composite GameDay with full incident process; at least 3 reliability tickets filed from findings.

---

## Part E — Capacity & performance engineering

### EXERCISE 15.9 — Capacity planning with data [A] [90 min]

**STEPS**

1. Baseline: load-test api to saturation (k6/locust — script in assets): find the knee (RPS where p95 departs linear), per-pod capacity, bottleneck resource (CPU? DB conns? — evidence!).
2. Model: current traffic ×6 (Black-Friday-style festival load — Indian fintech reality), required pods/nodes/DB capacity; headroom policy (N+1 AZ loss + 30% burst).
3. Karpenter behavior under flash load: provisioning latency vs traffic ramp — pre-scaling policy via scheduled HPA min-raise (EventBridge → k8s API) for known peaks.
4. Queue-based absorption: settlement spikes → SQS backlog growth math (arrival vs service rate; Little's Law applied to your queue — show the calculation); size workers from it.
5. DB capacity: connections ceiling, IOPS headroom, read replica offload thresholds; when vertical (instance up) vs horizontal (reads/caching/sharding) — write the ladder.
6. Forecasting from your own Mimir data: 90-day growth rate → when does the DB need the next tier (linear fit is fine; state assumptions).
**VALIDATE:** capacity model doc with the knee evidence, Little's Law math, and a forecast with dates.

### EXERCISE 15.10 — Performance deep-dive drills [A] [half day]

**STEPS**

1. Find a planted perf regression (script flips a config: pool size 20→2): detect via latency SLI, localize via spanmetrics (Module 11), root-cause via config diff — full loop, timed.
2. Node.js event-loop stall: the sample app has `/api/block` (sync crypto loop) — detect (event-loop lag metric), explain single-thread consequences, fix (worker_threads/offload), verify.
3. JVM-style vs Node memory behaviors (concept note for polyglot fleets); Go service profiling with pprof (write a tiny Go loadgen, pprof it — ties to 2.18).
4. Kernel-level: one EC2 iperf/fio/`perf` session tracing a slow disk to burst-balance exhaustion (gp3 baseline vs burst credits — CloudWatch `BurstBalance` metric; fix = baseline IOPS purchase).
**VALIDATE:** regression loop timed and documented; event-loop stall explained + fixed; burst-balance diagnosis written.

---

## Part F — FinOps / cost engineering

### EXERCISE 15.11 — Cost baselines, allocation, and optimization [A] [90 min]

*(slice Staff: "Establish cloud cost baselines, drive optimization initiatives, and implement cost reporting and governance.")*
**STEPS**

1. Baseline: `np-cost-today` + CUR/Athena → weekly cost by service/account/tag-env; commit `cost-baseline.md` (the "before" picture, honest).
2. Allocation: enforce cost tags (env, team, service) — untagged spend report → SCP/OPA gate ideas; showback report per "team" (simulate two).
3. Optimization pass (do each, record savings):

- right-size: EC2/RDS from metrics (Compute Optimizer + your data); 
- Spot: Karpenter spot for stateless nodes (already partly) + a pure-spot batch nodegroup for CI runners;
- Savings Plans/RI analysis: model 1-yr no-upfront compute SP for your steady baseline vs on-demand (even if you won't buy in a lab — produce the recommendation memo);
- S3 lifecycle (done 3.10 — quantify), orphaned resources sweep (unattached EBS, old snapshots, idle LBs/NATs, aged AMIs — script it: `np-cost-janitor`);
- data transfer: NAT GB, cross-AZ chatter (place chatty services same-AZ option), CloudFront offload of ALB bytes.

4. Governance: budget alerts (exist), cost anomaly response runbook (anomaly → triage like an incident: find the resource, owner, decision), monthly cost review meeting (with yourself — write minutes; practice the CFO conversation).
5. Unit economics: cost per 1k orders processed — the metric leadership actually wants; graph it in Grafana from CE data + app metrics.
**VALIDATE:** baseline doc + janitor script + recommendation memo + ≥30% lab cost reduction demonstrated (before/after graphs) + unit-economics panel.

### EXERCISE 15.12 — One-click AMI rotation (slice Staff verbatim) [A] [half day]

**GOAL:** automated, safe worker-node AMI refresh for EKS — the JD's literal line item.
**STEPS**

1. Mechanism: managed nodegroups `update-config` after AMI releases vs Karpenter drift (auto node replacement on new AMI) — enable drift; understand consolidation interplay.
2. Pipeline: weekly EventBridge → check new EKS-optimized AMI (SSM public parameter) → open PR bumping nodegroup AMI release version (or rely on drift) → rolling upgrade honoring PDBs → smoke checks → report.
3. Custom AMI variant: Packer build (hardened base with your Module 08 CIS role baked via ansible provisioner) → share to accounts → nodegroup uses it → rotation pipeline同上. Compare managed-AMI vs custom-AMI ops burden (ADR).
4. Safety: canary nodegroup first, PDB-blocking behavior tested (from 5.10/5.19), automatic halt on elevated 5xx during rotation.
**VALIDATE:** rotation executed with zero downtime evidence; halt-on-error tested with an injected failure; ADR committed.

---

## Module 15 exit gate

- [ ] SLOs live for 4 journeys + 5 dependencies; burn-rate paging; signed error-budget policy.
- [ ] 3 blameless postmortems + on-call week report + chaos playbook with ≥3 reliability tickets.
- [ ] One-click DR drill executed clean (measured RTO/RPO vs targets) + drill cadence policy.
- [ ] Capacity model + perf regression loop + cost baseline with demonstrated optimization and unit economics.
- [ ] AMI rotation pipeline — the JD line, done.

**Onward:** `16-Service-Mesh-Istio.md` — the last infrastructure frontier; then interview prep.
