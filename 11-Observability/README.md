# Module 11 — Observability: Metrics, Logs, Traces, Alerts (Weeks 14–15)

> **JD coverage:** "Experience with observability stacks: Datadog, Grafana, Mimir, Loki, Tempo, Zabbix" (Deel SRE — this module is nearly verbatim their stack); "Design and manage different monitoring and observability tools for troubleshooting and resolving production issues" (Deel DevOps); "Prometheus, Grafana, ELK/OpenSearch... using it to form and test hypotheses" (slice SRE-2); "Build scalable metrics, alerting, and logging pipelines" (slice SRE-3).
> Assets: `assets/observability/` (dashboards, rules, helm values).
> Operating principle: **observability exists to answer questions you didn't predict.** Dashboards for knowns; ad-hoc query power (PromQL/LogQL/TraceQL) for unknowns.

---

## Part A — Prometheus & Grafana foundations

### EXERCISE 11.1 — Prometheus from first principles [B] [75 min]

**STEPS**

1. Install `kube-prometheus-stack` via Helm on kind (ArgoCD-managed, per Module 10 discipline). Identify: Prometheus server, Alertmanager, node-exporter DaemonSet, kube-state-metrics, Grafana, Operator + CRDs (ServiceMonitor/PodMonitor/PrometheusRule).
2. The pull model: Prometheus scrapes `/metrics` endpoints on an interval — contrast with push (StatsD/CloudWatch agent). ServiceMonitor for the api (labels must match service — the #1 gotcha; debug a non-appearing target via `/targets` page).
3. Targets anatomy: `up` metric, relabeling (read the generated config; drop noisy labels via metricRelabelings — cardinality control starts here).
4. Storage: TSDB on PVC; retention flags; what happens at disk full (write-amplification + crash — protect with retention + size limits; alert on it).
5. Federation & remote-write concepts: hub-spoke for multi-cluster; **Mimir in Part C is remote-write at scale.**
**VALIDATE:** api's `/metrics` scraped (target up); you can explain the scrape→TSDB→query path.

### EXERCISE 11.2 — PromQL fluency lab [I] [2 hrs]

**GOAL:** PromQL is *the* SRE query language — interviewers watch you write it live.
**STEPS** (load running via gen_load; query everything below and save outputs)

1. Basics: instant vs range vectors; selectors `{job="api", status=~"5.."}; rate() vs irate()` on counters (why rate for alerting); `increase()` over windows.
2. Golden Signals for the api (write each query):

- Latency: `histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))` — understand buckets, why `by (le)`, and why histograms beat avg.
- Traffic: `sum(rate(http_requests_total[5m]))`
- Errors: ratio `sum(rate(...status=~"5..")) / sum(rate(...))` — **this ratio is your SLO's raw material.**
- Saturation: CPU usage vs requests; throttling ratio from Part 5.8.

3. Operators: `by`/`without`, arithmetic between series, `topk`, `avg_over_time`, `delta` on gauges, `absent()` (dead-man's-switch alerting!), `offset` (week-over-week compare), vector matching (`on()/group_left`) — join request rate with pod info.
4. Recording rules: precompute heavy ratios (`job:http_errors:rate5m`) — alert on the recording rule, not the raw expression (speed + consistency).
5. Gotchas lab: counter resets (restart mid-scrape — rate handles it); missing data (`absent`, `unless`); staleness (5-min rule); label cardinality explosion (create a bad metric with user_id label in a scratch exporter → watch series count explode → fix by dropping the label). **Cardinality is the Prometheus bill-killer; Mimir enforces limits in Part C.**
6. `promtool`: `promtool check rules`, `promtool query instant` from CLI, `promtool test rules` — **unit tests for alert rules** (write 3 tests: firing, not-firing, threshold edge).
**VALIDATE:** golden-signals queries in a Grafana dashboard + recording rules + 3 rule unit tests committed.

### EXERCISE 11.3 — Grafana: dashboards as products [I] [90 min]

**STEPS**

1. Provision everything as code: datasources + dashboards via Helm values/ConfigMaps (never click-built in prod; UI is for exploring, code is for keeping). Export one UI-built dashboard → JSON → commit (`assets/observability/dashboards/` starter provided).
2. Build dashboard `NorthPay / API Overview`: rows for golden signals; variables (`$namespace`, `$pod`), `repeat` per pod; thresholds coloring; **SLO panel** (error-budget remaining, burn-rate — placeholder query until Module 15 defines the SLO formally).
3. Annotations: deploys from ArgoCD (webhook → Grafana annotation API) — correlate "errors started at 14:03" with "deploy at 14:02" visually. This single feature shortens every incident.
4. Drill-downs: dashboard links (service → pod → logs in Loki with pre-filled query via Explore links).
5. Alerting in Grafana vs Alertmanager (Part B): know both, standardize on Alertmanager (decoupled, dedupe/routing).
6. Performance: dashboard with 40 panels × high-cardinality queries = slow; recording rules + query best practices; Grafana's own metrics (`grafana_*`, query timing).
**VALIDATE:** dashboard loads < 2s, provisioned from Git, annotations show your last deploy.

---

## Part B — Alerting that doesn't cry wolf

### EXERCISE 11.4 — Alertmanager: routing, silences, inhibition [I] [90 min]

**STEPS**

1. Architecture: Prometheus fires alerts → Alertmanager groups/dedupes/routes → receivers (email now; webhook → SNS; Slack via webhook; PagerDuty/Opsgenie conceptually).
2. Routing tree practice: `severity=critical` → oncall email + SNS; `severity=warning` → daily digest receiver; `team=payments` → their channel; `continue: true` semantics.
3. Grouping/wait/interval tuning: group_by [alertname, namespace]; group_wait 30s (why: collapse the first burst of a real incident into ONE page), repeat_interval 4h.
4. Inhibition: node down inhibits its pod alerts (write the inhibit rule); cluster-wide outage shouldn't page 50 times.
5. Silences: CLI + UI; silence for a planned migration window with expiry + comment (change-management integration).
6. **Burn-rate alerting** (Google SRE multi-window): implement the classic pair — fast burn (2% budget in 1h → page) and slow burn (100% in 3d → ticket). Use provided rules from assets. Even before Module 15 formalizes SLOs, wire the alerts.
7. Dead man's switch: a heartbeat alert that must always fire to a watchdog receiver — if your monitoring dies, you get paged about *that*. Implement with `absent()`/always-firing rule + external healthchecks.io-style ping (free tier) or SNS.
8. On-call simulation: set a rotation with your two email addresses; page yourself via a real alert at 2 AM once (feel it — then tune thresholds so it never happens without cause; write the alert-quality policy: every page actionable, every page has a runbook link).
**VALIDATE:** inhibition proven (one page for node-down, not 20); burn-rate alerts fire under synthetic error injection; dead-man's-switch tested by stopping Prometheus.

### EXERCISE 11.5 — Alert quality engineering [A] [60 min]

**STEPS**

1. Audit your current alerts: tag each as symptom-based (users affected — page) vs cause-based (disk 80% — ticket). Rewrite cause-based paging alerts to tickets. **"Alert on symptoms, not causes" — SRE canon.**
2. Every alert gets: runbook URL annotation, severity, owner, and a `for:` duration review (flapping test: does it flap under noise? add hysteresis).
3. Alert test harness: extend `promtool test rules` coverage to all rules in CI (Module 09 gate).
4. MTTR instrumentation: time from alert-fire to resolution logged in tickets (Module 17 simulation tracks this) — you can't improve what you don't measure.
**VALIDATE:** alert inventory doc with symptom/cause classification; CI gate on rule tests.

---

## Part C — Mimir: Prometheus at scale (Deel stack!)

### EXERCISE 11.6 — Mimir deployment & architecture [A] [2 hrs]

**STEPS**

1. Why Mimir exists (write it): single Prometheus = single point + retention limits + no multi-tenancy + can't scale reads/writes independently. Mimir = horizontally scalable, S3-backed, multi-tenant Cortex descendant.
2. Deploy `mimir-distributed` Helm chart to kind (minIO as S3 stand-in) or small EKS ns. Components to name from memory: distributor, ingester, querier, query-frontend, compactor, store-gateway, ruler. Draw the write path (remote-write → distributor → ingesters → TSDB blocks → S3 via compactor) and read path (querier → frontend cache → ingesters+store-gateway).
3. Remote-write from your existing Prometheus → Mimir (keep local Prom as agent-mode — `prometheus --storage.tsdb.retention=2h` agent pattern; or Grafana Alloy as collector).
4. Multi-tenancy: two tenants (`team-payments`, `team-platform`) via `X-Scope-OrgID`; query isolation proof.
5. Long-term retention: compactor to minIO/S3; query 13-month-old data instantly vs Prometheus local retention reality.
6. Ruler: move your alert rules into Mimir ruler (tenant-scoped rules API) — alert evaluation survives a Prometheus restart.
7. Cardinality governance: per-tenant limits (`max_global_series_per_user`), active-series dashboards; identify your top cardinality offenders (native histograms note: know they exist, reduce bucket waste).
**VALIDATE:** Prometheus stopped → queries still return historical data via Mimir; two-tenant isolation proven; one cardinality limit enforced (deliberately exceeded → rejection observed).
**TRADEOFFS:** Mimir vs Thanos (sidecar vs push model, compactor similarities) vs VictoriaMetrics (simpler ops, different license) vs Amazon Managed Prometheus (managed, pricing per series, lock-in). Know all four columns — Deel runs Mimir, slice runs Prometheus; "how do you scale metrics" is a Staff interview staple.

---

## Part D — Loki: logs the Kubernetes way

### EXERCISE 11.7 — Loki + Alloy: the logging pipeline [I] [2 hrs]

**STEPS**

1. Deploy `loki` (distributed mode, minIO/S3 backend) + `alloy` (or promtail — know both; Alloy is the successor) DaemonSet via Helm/ArgoCD. Architecture: no indexing of content (only labels!) — that's why it's cheap vs Elasticsearch; you run both (ES/OpenSearch in Module 12 for search-heavy use cases).
2. Verify: every pod's stdout lands in Loki; LogQL basics in Explore: `{namespace="prod"} |= "ERROR" | json | latency > 0.5`, `rate(...)` for error-rate-over-time-from-logs (metrics-from-logs recording rules via Loki ruler).
3. Label discipline: Loki labels = low-cardinality only (namespace, app, level) — never request_id (instant death by stream explosion; reproduce cheaply: set a high-cardinality label, watch stream count, revert).
4. Pipeline stages: parse JSON logs into labels/structured metadata; drop noisy healthcheck logs at the agent (cost control at ingestion).
5. Retention per tenant via compactor config (table manager in older versions); compliance retention (90d hot, 13mo cold in S3) — RBI/PCI-style requirement mapping.
6. Log-based alerting: 5xx spike from Loki → Alertmanager; then convert stable patterns to metrics (cheaper + faster) — the logs→metrics maturation path.
7. Correlation: derived fields in Grafana — trace_id in a log line becomes a clickable link jumping to Tempo (Part E); service.name links back to metrics. **The metrics↔logs↔traces triangle is the whole point of the Grafana stack.**
**VALIDATE:** a request's journey traceable: metric spike → LogQL filtered logs → trace link; log-based alert fires; retention config shown.

### EXERCISE 11.8 — Log architecture decisions & cost [A] [45 min]

**STEPS**

1. Write ADR-0011: Loki vs ELK/OpenSearch vs CloudWatch Logs vs Datadog logs — dimensions: cost model (index size vs ingestion), query power (full-text vs label+filter), ops burden, ecosystem. NorthPay: Loki for k8s/app logs, OpenSearch for audit/security search (Module 13), CloudWatch for control-plane + AWS-service logs.
2. Ingestion cost control: sampling (log levels per env), drop rules, per-tenant quotas; estimate your lab's GB/day and project 100-service cost in each option — put the table in the ADR.
3. Multi-tenant log access: Loki tenants per team + Grafana org/team permissions — devs see only their namespace's logs (implement via label enforcement + tenant header).
**VALIDATE:** ADR with cost model committed; tenant isolation demo.

---

## Part E — Tempo & tracing (OpenTelemetry)

### EXERCISE 11.9 — OpenTelemetry instrumentation [I] [2 hrs]

**STEPS**

1. Concepts first (write them): trace = request's journey; spans = operations; context propagation (traceparent headers) across HTTP/SQS; sampling strategies (head vs tail — tail sampling keeps *interesting* traces, costs more; decide per service).
2. Instrument the api: OTel Node.js auto-instrumentation (`@opentelemetry/auto-instrumentations-node`) + manual span around settlement logic; propagate through worker via SQS message attributes (sample code provided in assets — wire it).
3. OTel Collector (the central piece): receivers (otlp) → processors (batch, memory_limiter, attributes) → exporters (Tempo, Prometheus spanmetrics, Loki for span logs). Deploy as Deployment (gateway) + DaemonSet (agent) modes — when each.
4. Deploy Tempo (S3/minIO backend); point collector; query by trace ID; then **TraceQL**: `{ resource.service.name = "api" && duration > 500ms }`, `{ span.http.status_code = 500 }` — find slow/error traces without knowing IDs.
5. Spanmetrics connector: generate RED metrics *from traces* (request rate/errors/duration per service+operation) → feed Mimir → dashboards of service-level latency without app changes.
6. Service graph: Grafana service graph from spanmetrics — live architecture map of NorthPay. Screenshot for your portfolio.
7. Exemplars: metrics → jump to an exemplar trace from a latency spike point. Configure; demo the click-through: spike on graph → trace → spans → logs for that trace_id. **Record this flow — it's artifact #3 material and pure SRE gold in interviews.**
**VALIDATE:** end-to-end trace api→worker visible; TraceQL finds a slow request; exemplar click-through works; service graph generated.

### EXERCISE 11.10 — Tracing in anger: latency forensics drill [A] [60 min]

**STEPS**

1. Plant latency: add a 300ms sleep in worker's Mongo write path (env flag in sample app).
2. Hunt it using only traces + spanmetrics: which service? which span? Compare before/after deploy annotations.
3. Sampling decision under load: crank gen_load ×10; head-sample 10% vs tail-sample errors — compare what you'd see/miss; write the policy (prod: tail-sampling for errors+slow, 5% baseline).
4. Cost math: spans/sec × retention × storage — Tempo on S3 vs Datadog APM pricing; when volume forces sampling discipline.
**VALIDATE:** planted latency located to the exact span within 15 min; sampling policy committed.

---

## Part F — Datadog (the commercial reality check)

### EXERCISE 11.11 — Datadog trial: full evaluation [I] [2 hrs + passive week]

*(Deel runs Datadog; trials are free 14 days — schedule this at a week you can use daily.)*
**STEPS**

1. Sign up (free trial, no card). Install the Datadog Agent via Helm on kind (API key as secret) — DaemonSet + cluster agent; enable APM, logs, NPM (network performance), and KSM core.
2. Instrument api with dd-trace (or reuse OTel → Datadog exporter — know both; OTel is the portable answer).
3. Build: service catalog view, a dashboard cloning your Grafana golden-signals board, SLO definitions with burn-rate alerts (Datadog SLO UI), a monitor with multi-alert (by pod) + anomaly detection.
4. RUM/synthetics: set up a synthetic API test hitting your endpoint every minute from 2 locations (external monitoring — your "outside view"; free-tier limits noted).
5. **The comparison essay (the real deliverable):** Datadog vs self-hosted LGTM — time-to-value (minutes vs days), cost at scale (per-host + per-million-spans + ingestion GB — model NorthPay at 50 hosts/10M series), feature depth (NPM, Watchdog AI, incident management), lock-in/portability (OTel mitigates), data residency (fintech!). Conclude: why Deel runs Datadog *and* Mimir/Loki/Tempo (scale economics + control) — that's a real architecture conversation you can now lead.
6. Cancel/cleanup discipline: uninstall agent, note trial expiry, export your dashboards JSON first.
**VALIDATE:** comparison essay committed; synthetic test data flowing; SLO defined in DD UI.

### EXERCISE 11.12 — Zabbix: the old guard [B] [75 min]

*(Deel SRE lists Zabbix — it's how many enterprises monitor network gear and legacy VMs. Your networking background + Zabbix = instant credibility with infra teams.)*
**STEPS**

1. Deploy zabbix-server + web + db via docker-compose on `legacy-dc` (assets/observability/zabbix-compose.yml).
2. Agents on both onprem VMs + one EC2; **SNMP monitoring of a network device**: your physical router/firewall if it allows SNMP, else install snmpd on a VM simulating a device (interfaces, traffic counters) — you know SNMP; show off.
3. Triggers (Zabbix alerts) vs PromQL rules — translate 3 of your Prometheus rules into Zabbix triggers; note the mental model difference (item/trigger/action vs metric/rule/route).
4. Auto-discovery: network discovery rule finding your onprem subnet hosts — why enterprises love it for sprawling estates.
5. Verdict paragraph: where Zabbix wins (network devices, agent-based legacy, all-in-one licensing-free) vs cloud-native stacks (dynamic infrastructure pain — Zabbix + ephemeral pods = mismatch).
**VALIDATE:** SNMP interface traffic graphs live; trigger→email alert fires on interface down (disable a VM NIC).

---

## Part G — The unified practice

### EXERCISE 11.13 — Blackbox & synthetic monitoring + status page [I] [60 min]

**STEPS**

1. blackbox-exporter: probes for `https://api.northpay.../healthz` from inside (and an external free service for outside); cert-expiry probe (30d warning — remember gauntlet #10).
2. Uptime SLO: 99.9% availability computed from probe success ratio; graph it.
3. Status page: cachet/openstatus on kind or a simple static page updated by CI; the comms side of incidents (Module 15).
**VALIDATE:** cert-expiry alert fires on a short-lived test cert; uptime dashboard shows real probe data.

### EXERCISE 11.14 — Observability capstone: "find the incident" drill [A] [half day]

**STEPS**

1. Have a friend (or a script with a delay) inject one of: DB connection pool exhaustion, bad deploy raising 5xx for one route only, redis eviction storm, DNS intermittent failure, slow Mongo query.
2. You get ONE alert ("error budget burning fast"). Using only your stack: triage → localize service → find span/log evidence → root cause → fix → verify → postmortem.
3. Time yourself. Target: MTTD < 5 min (alert→you looking at the right dashboard), MTTR < 30 min.
4. Write the postmortem with full evidence links (dashboard snapshots, trace IDs, log queries) — this is the Module 15 standard, practiced here.
**VALIDATE:** full evidence chain in postmortem; identify which signal (metric/log/trace) localized the fault fastest — and say why.

---

## Module 11 exit gate

- [ ] LGTM fully operational: Prometheus (+Mimir remote-write, S3 storage, tenants), Loki (retention, per-team tenants), Tempo (TraceQL, spanmetrics, service graph), Grafana (provisioned dashboards, annotations, explore-links).
- [ ] Alerting: burn-rate multi-window, inhibition, silences, dead-man's-switch, rule unit tests in CI.
- [ ] Datadog evaluated with a written cost/value verdict; Zabbix monitoring network devices via SNMP.
- [ ] Exemplar-driven metrics→traces→logs flow recorded.
- [ ] Capstone drill done with measured MTTD/MTTR.

**Onward:** `12-Databases.md` — the stateful tier, where SREs earn their pay.
