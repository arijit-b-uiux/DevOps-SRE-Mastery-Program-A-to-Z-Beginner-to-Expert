# Module 18 — SRE Interview Mastery: Scenarios, Hands-On Gauntlets, System Design (Week 24+, then forever)

> Target interviews: scenario-based ("a deploy broke prod, walk me through..."), hands-on ("this pod is CrashLooping — the laptop is yours"), system design ("design a multi-region payments platform"), and behavioral ("a time you disagreed with..."). This module trains all four, using evidence from your 90 days.
> **Method:** for every question: answer out loud in ≤ 3 minutes using the scaffold shown, then check against the model points, then find the matching evidence in your ticket log. Record yourself monthly; cringe; improve.

**Universal answer scaffolds:**

- **Incident scenario:** stabilize → scope → communicate → diagnose (hypothesis ladder) → mitigate → resolve → follow-up (postmortem + prevention). Say blast radius and rollback thinking early.
- **Design question:** clarify SLOs/scale/compliance → sketch the boring-correct architecture → name the failure modes → name the tradeoffs you chose and the ones you rejected.
- **"Why X over Y":** answer in 3 axes — operational cost, failure modes, team/org fit. Never just feature lists.

---

## Section 1 — Incident & troubleshooting scenarios (45)

*Answer each aloud. Model points follow in italics — cover at least 70%.*

1. **"Users report intermittent 502s after a deploy. Walk me through."** — *Check deploy annotation vs error start; 5xx rate per pod (one bad canary?); rollout status/history; app logs of failing pods; upstream connect errors → DB pool?; mitigate: rollback (git revert → ArgoCD) vs fix-forward decision criteria; verify via SLO; postmortem + action items (why didn't canary analysis catch it?).*
2. **"A pod is in CrashLoopBackOff. Go."** — *describe (exit code, reason), logs --previous, recent events; common causes: config/env, OOM (137), liveness misconfig, missing secret/configmap, port conflict, bad image; fix at cause not symptom; prevention: CI gates, schema validation.*
3. **"Latency doubled across the board at 14:00. No deploys happened."** — *dashboards: which component's p95 moved first (spanmetrics!); infra changes (auto-scaling, node events, Karpenter consolidation at 14:00?); dependency saturation (RDS CPU/locks, redis evictions, DNS); network (conntrack, NAT port exhaustion, cross-AZ); certificate/clock edges; neighbor noise (noisy neighbor on node).*
4. **"Disk full on a DB node, writes failing."** — *immediate: stop the bleed safely (don't delete WAL/binlog!); find growth source (data, logs, temp, bloat); short-term: expand volume online (EBS modify), clean logs; then: retention policies, alerts at 70/80/90, capacity model review; postmortem.*
5. **"Kube-dns resolution is intermittently failing cluster-wide."** — *CoreDNS health/scaling, ndots/search-domain amplification, conntrack/UDP drops on nodes, NetworkPolicy blocking 53, node-local DNS cache as mitigation; measure with per-pod dns latency metrics.*
6. **"Your queue depth is growing 10k/min. Consumers look healthy."** — *Little's Law math aloud; consumer errors silently acking? poison messages cycling (DLQ empty?); visibility timeout vs processing time; downstream DB slow (consumers slow, not broken); scale consumers (KEDA), shed load, throttle producer; find the trigger event.*
7. **"SSL certificate expired in production."** — *impact scope (which endpoints), hotfix (ACM auto-renews — why didn't it? DNS validation record deleted!), emergency issuance, then: expiry alerting at 30/14/7d (blackbox), cert-manager audit, postmortem on the monitoring gap.*
8. **"RDS CPU pegged at 100% after a feature launch."** — *pg_stat_statements top-by-time; new query missing index (seq scan on big table); N+1 from new code path; lock contention; mitigate: kill worst queries, add index CONCURRENTLY, scale read replica offload; prevention: load-test with prod-like data volume, query review in CI, p95 query budget in CI.*
9. **"Pods Pending after you scaled up."** — *describe → Insufficient cpu/mem vs taints vs PVC binding vs quota vs IP exhaustion (VPC CNI!); then the right fix per cause; prevention: capacity alerts, overprovision pause-pods.*
10. **"Terraform apply failed halfway; some things exist, some don't."** — *state lock held? plan to converge; never hand-delete without import; refresh/import/state-mv runbook; prevention: smaller blast-radius stacks, CI serialization.*
11. **"Your canary deploy passed analysis, but errors spiked after full rollout."** — *analysis window too short? metric lag? error only at scale (pool exhaustion at 100% traffic)? dark traffic?; improve analysis: longer soak, scale-sensitive metrics, synthetic load in canary phase.*
12. **"A developer accidentally deleted a namespace."** — *blast radius (what lived there?), GitOps restore (argocd app resync — state in Git!), etcd backup if not, PVC/data impact (reclaim policy!), then RBAC review + Kyverno prevent-deletion policy + backup verification.*
13. **"MongoDB primary flapping between nodes."** — *elections: network partitions, resource starvation (CPU steal), oplog window too small under load, priorities misconfigured; stabilize (priority pinning), fix resource pressure, enlarge oplog; w:majority impact during flaps.*
14. **"Night batch job silently stopped running."** — *CronJob concurrencyPolicy=Forbid + a stuck job blocked subsequent runs; startingDeadlineSeconds missed-window; dead-man's-switch alert (absent metric) catches silence — the alert you built in 11.4/5.15.*
15. **"Intermittent connection timeouts from EKS pods to an external API."** — *NAT gateway port exhaustion (ErrorPortAllocation!), SNAT port reuse, conntrack, TCP MSS over tunnel, DNS TTL pinning in app (connection reuse across IP changes); measure: NAT metrics, conntrack count, per-dest breakdown.*
16. **"Two services can't talk after NetworkPolicy rollout."** — *default-deny plus forgotten egress DNS rule (gauntlet #8); policy diff; test matrix; staged rollout of policies (audit mode first via Kyverno/Cilium).*
17. **"EBS volume stuck 'attaching', pod won't start."** — *Multi-Attach error (node failure vs volume detach lag), force-detach path, node AZ vs volume AZ mismatch (WaitForFirstConsumer lesson), CSI driver logs.*
18. **"Error budget for the month burned in a day."** — *invoke policy: freeze, swarm on reliability, exec comms; root cause + budget math; then the honest discussion: was the SLO right? was this a known-risk tradeoff? policy worked as designed = success, not shame.*
19. **"Your monitoring stack died and nobody noticed for 6 hours."** — *dead-man's-switch absent; meta-monitoring (external watchdog), monitoring the monitors; comms to stakeholders about observability gap; add to DR dependencies.*
20. **"Deploy pipeline is green but the app is broken for 5% of users."** — *sticky canary? feature flag partial rollout? LB target group with one bad AZ? geo-specific (edge rules)? client version skew; per-dimension error breakdown (user segment, region, version).*
21. **"CloudWatch bills tripled."** — *custom-metric cardinality explosion (a user_id label leaked), log ingestion spike (debug level left on), alarm evaluation frequency; cost anomaly response runbook; cardinality limits.*
22. **"API works from inside the VPC, fails from internet."** — *ALB SG vs listener vs WAF block vs DNS vs cert vs Cloudflare origin lockdown over-applied; the path-by-hop method (you're a network engineer — show off: dig → curl -v to each hop → SG/WAF logs).*
23. **"A secret was committed to GitHub."** — *immediate rotation FIRST (assume compromised), then history purge (filter-repo/BFG — and why purging ≠ safety after push), gitleaks gates, root cause: how did it pass pre-commit + CI?; audit access logs (CloudTrail for the secret's use).*
24. **"ArgoCD shows everything OutOfSync after someone edited live resources."** — *selfHeal reverts; but first: was the hand-edit an emergency fix? communication failure, not tooling; emergency-change path; revisit policy friction.*
25. **"Post-patch reboot loop on EC2 fleet."** — *ASG instance refresh vs health check grace, user-data failing → unhealthy → replace → loop; stop the loop (suspend ReplaceUnhealthy), fix user-data, canary one instance; launch template versioning.*
26. **"Elasticsearch cluster red."** — *unassigned shards: disk watermark (85/90/95%), node loss, allocation explain; free disk or expand, reroute; then ILM/watermark alerting review.*
27. **"Metrics show healthy but users complain."** — *probe-vs-real gap: SLIs measure the wrong thing (healthz vs real journey), synthetic checks per journey, SLO redefinition; "monitoring green, users red" is an SLI-quality bug.*
28. **"Cross-region failover drill took 2× the promised RTO."** — *timeline breakdown: detection, decision, promote, scale, DNS TTL reality, cold caches; each measured; DR plan updated with real numbers; RTO promise adjusted or automation added (that's what drills are FOR).*
29. **"A team's deploys keep breaking staging."** — *staging as shared resource: deploy serialization, namespace isolation, preview-envs as the fix (Module 09!); platform thinking: make the right path easy.*
30. **"Kubernetes node NotReady storm during peak."** — *resource exhaustion cascade (memory pressure → eviction → rescheduling → more pressure), PDB interaction, Karpenter/CA scale-up latency vs surge; overprovisioning policy, priority classes protecting critical pods.*
31. **"You're on call. Two alerts fire: DB replica lag growing AND deploy stuck. Related?"** — *correlation thinking: deploy's migration locking tables → replica apply lag; mitigate migration (kill/pause), deploy unblocks; lesson: migration windows + lock-timeout discipline.*
32. **"Payment API p99 fine, p50 fine, but total throughput collapsed."** — *queueing outside the service (ALB target queue? worker pool? HPA maxed?), client-side timeouts masking retries, connection pool serialization; throughput vs latency decomposition.*
33. **"Someone asks: can we skip staging and test in prod?"** — *honest senior answer: you already do (canary + flags + rollbacks ARE prod testing with guardrails); the question is blast-radius engineering, not purity; describe your progressive-delivery stack as "safe prod testing."*
34. **"Redis OOM and app erroring."** — *eviction policy noeviction mistake; maxmemory sizing; is it a cache (evict) or a store (persist)? bigkey analysis; memory alerts; app graceful degradation (cache miss ≠ outage).*
35. **"An AWS region is down. Your move?"** — *verify (not a you-problem first: status page, your blackbox external probes), declare SEV1, DR runbook (15.7): promote replica, scale DR, DNS flip, comms cadence; data-loss/RPO expectation set with business BEFORE flipping; postmortem even when AWS's fault (your detection/decision speed is yours).*
36. **"IAM suddenly denies a working CI deploy."** — *what changed: SCP added? permission boundary? role trust? session policy? CloudTrail as the answer engine (denied events with reasons); policy-change review process.*
37. **"Intermittent TLS handshake failures for one partner only."** — *TLS version/cipher overlap, SNI, cert chain completeness (missing intermediate!), partner's clock skew, MTU/MSS over their VPN — network-engineer territory: capture the handshake, compare working vs not.*
38. **"Log pipeline costs more than compute."** — *cardinality/volume audit (which ns/team/level), drop rules, sampling, retention tiers, metrics-maturation for alertable patterns; per-team showback; then architecture: is full-fidelity worth it (compliance says yes for audit logs, no for debug logs).*
39. **"A canary analysis metric went away (Prometheus scrape gap)."** — *fail-closed vs fail-open design for automation gates: analysis with missing data should halt and alert, not pass; fix template; add up{} checks to AnalysisTemplates.*
40. **"kubectl commands slow cluster-wide."** — *API server pressure: expensive LISTs (no label selectors, all-namespaces), etcd latency, webhook latency (admission chain), audit log volume; APF (priority & fairness) concept.*
41. **"A new microservice deploys fine but never gets traffic."** — *Service selector mismatch (gauntlet #7), ingress route ordering, readiness failing silently, ArgoCD app in wrong ns; the debug ladder in 90 seconds.*
42. **"Your on-prem VPN tunnels drop every night at 02:00."** — *rekey/DPD mismatch, ISP maintenance window, NAT timeout on stateful middlebox (keepalives), IKE lifetime mismatch; logs both sides, correlate; BGP hold timers masking it?*
43. **"S3 costs doubled but data didn't."** — *request costs (LIST storms, small-object churn), incomplete multipart uploads accumulating (lifecycle abort rule!), versioned-deleted objects kept, replication traffic; S3 storage lens/inventory.*
44. **"How do you investigate a 'site slow' report with no other information?"** — *the meta-answer: golden signals per journey → compare across dimensions (region, version, endpoint, user segment) → dependency waterfall via traces → recent-change correlation → resource saturation. Demonstrate with your Grafana live if allowed.*
45. **"Design your ideal first hour of a SEV1."** — *declare + roles + comms channel + status page early ("investigating" beats silence) → mitigation-first mindset (rollback/failover/throttle BEFORE full diagnosis) → timeline discipline → all-hands prevention after. Contrast with junior instinct (debug silently for an hour).*

---

## Section 2 — Kubernetes & platform depth (20)

46. Explain what happens when you `kubectl apply` a Deployment — full control-loop anatomy. *(API authz → etcd → controllers → scheduler → kubelet → CNI → kube-proxy/Envoy; say it smoothly in 90s.)*
47. Requests vs limits vs QoS vs eviction — and the CPU-limit debate. *(Your 5.8 data.)*
48. How does a Service actually route? iptables vs IPVS vs eBPF; conntrack at scale. *(5.4 + 2.5.)*
49. Design namespace/RBAC/NetworkPolicy tenancy for 20 teams. *(Projects+AppProjects, default-deny, quota/LimitRange, Kyverno guardrails, onboarding via ApplicationSet git-generator = self-service.)*
50. Upgrade strategy for 30 clusters. *(N-1 policy, waves, pluto deprecation gates, PDB contracts with teams, canary cluster cohort.)*
51. EKS networking at 5k pods: IP exhaustion answers. *(prefix delegation, secondary CIDRs CGNAT, custom networking, or Cilium/ebpf replacing kube-proxy+VPC CNI tradeoffs.)*
52. etcd: what to monitor, backup/restore, defrag, why 3/5 nodes. *(Your snapshot drill + latency metrics + quorum math.)*
53. When DaemonSet vs sidecar vs node agent? *(5.2 + 16.1 ambient discussion.)*
54. HPA vs VPA vs KEDA vs Karpenter — compose them without conflicts. *(Metric ownership rules; VPA recommend-only with HPA; KEDA on external signals; Karpenter node-side.)*
55. Design secrets management for a bank on EKS. *(ESO+SM, IRSA, no Git secrets, rotation propagation, audit, break-glass — 10.6 + 13.8.)*
56. Admission control: what would you enforce org-wide? *(non-root, resources, no :latest, signed images only (cosign verify), registry allowlist, ingress host uniqueness; rollout: audit→warn→enforce per ns.)*
57. Debug: `kubectl get pods` fine, service unreachable, DNS fine. *(endpoints object → kube-proxy rules → NetworkPolicy → CNI datapath → target pod readiness gates.)*
58. Stateful workloads on K8s: rules of engagement. *(Operators, storage classes, backup operators (Velero), but default answer: managed services for prod data — your ADR-0002.)*
59. Multi-tenancy hard vs soft. *(vCluster/Capsule vs ns-isolation; cost/blast-radius tradeoff.)*
60. GitOps at 500 apps: pains and fixes. *(repo-server scale, sharding, app-of-apps hierarchy, drift noise, PR promotion tooling — 10.5/10.8.)*
61. Image supply chain end-to-end. *(sign at CI, verify at admission, digest pinning, SBOM, private registry, ECR immutability.)*
62. Your cluster autoscaling didn't trigger during a flash sale. Why? *(pending pods didn't fit node shapes Karpenter can provision; provisioning latency vs ramp; overprovisioner pause pods; scheduled pre-scaling — 15.9.3.)*
63. Explain finalizers and a namespace stuck Terminating. *(9.4 runbook; orphan finalizer from dead operator; patch removal after investigation.)*
64. Design in-cluster dev/PR environments at scale. *(9.4 + vCluster option; data strategy; cost reaping; TTLs.)*
65. What breaks first when a cluster 10×'s? *(etcd size/latency, kube-proxy rules, CoreDNS QPS, scheduler throughput, API LIST cost, CNI IP space — with your monitoring for each.)*

---

## Section 3 — AWS, networking & multicloud (20)

66. Design VPC architecture for a 3-tier fintech, multi-account. *(13.1+13.3: org, per-account VPCs, TGW route domains, central egress, endpoints, flow logs — draw it.)*
67. SG vs NACL; SG referencing across TGW/peering; limits. *(Statefulness, layer, default posture, evaluation logic.)*
68. ALB vs NLB vs GWLB vs CloudFront vs Global Accelerator — pick for: partner webhook ingress (static IP needed), gRPC internal, global API, DDoS posture. *(NLB/GA for static IP, NLB gRPC, CF/GA global, Shield+WAF.)*
69. Site-to-site VPN deep: why two tunnels, BGP vs static, IKE/IPsec params, throughput ceilings, failover timing. *(13.4 numbers.)*
70. Direct Connect vs VPN vs both. *(Consistency/egress-cost vs cost/instant; hybrid: VPN backup over DX.)*
71. Hybrid DNS design. *(13.5: resolver endpoints, forwarding rules, split-horizon, failure modes.)*
72. PrivateLink vs TGW vs peering — with overlapping CIDRs. *(13.6 matrix.)*
73. Route53 routing policies + health-check math; why DNS failover can't meet a 1-min RTO alone. *(3.12 numbers.)*
74. NAT gateway: cost model, port exhaustion, per-AZ HA, alternatives (endpoints, egress control). *(3.5/3.6/15.11.)*
75. Design WAF/edge for a payments API used only from India. *(geo-restriction, rate rules, managed rules, Cloudflare strict-TLS + origin lockdown, logging to Athena — 13.7.)*
76. IAM at scale: permission boundaries, SCPs, ABAC, access reviews, break-glass. *(3.2/13.1/13.10.)*
77. Encryption strategy for a bank: KMS key design, envelope encryption, cross-account KMS gotchas, rotation gaps. *(13.8 + 13.2.4.)*
78. Multicloud networking: how would you connect AWS, GCP, and on-prem with no transit leaks? *(14.8–14.9: ASN plan, route filters, no-transit policies, BFD, master route registry.)*
79. Multicloud active-active: what's genuinely hard? *(data gravity, egress economics, identity, consistency; your ADR-0015 verdict: one writer, async replication, RPO honesty.)*
80. S3 durability vs availability (11-9s meaning), consistency model, and design of a backup strategy that survives account compromise. *(Logically-air-gapped: cross-account, MFA-delete, object-lock/immutability, separate creds — ransomware-grade backups.)*
81. "Move this monolith VM workload to AWS with near-zero downtime." *(VPN/DX, PrivateLink/DNS cutover patterns, DMS for DB, replication-first-then-flip, rollback = flip back; your on-prem sim makes this demo-able.)*
82. Cost optimization program design. *(15.11: baselines, allocation tags, coverage (SP/RI), rightsizing loop, spot policy, data-transfer review, unit economics, governance cadence.)*
83. EBS vs EFS vs S3 vs FSx — choose for: shared config, DB storage, static assets, Windows shares. 
84. What is the Instance Metadata Service and why did Capital One-style breaches happen? *(IMDSv1 SSRF; v2 token requirement; SG egress least-privilege as layered defense.)*
85. Multi-region: what services fight you? *(Regional by default: ACM, AMI, KMS keys, SSM params, ECR, secrets — your 15.6 dependency inventory; global: IAM, Route53, CloudFront.)*

---

## Section 4 — Reliability, delivery & design (15)

86. Define SLI/SLO/SLA/error budget with an example; what happens on exhaustion? *(15.1 policy — quote your own document.)*
87. Design alerting for a new payment service from scratch. *(symptom alerts on user journeys, multi-window burn rate, dependency tickets, dead-man's switch, runbook links, inhibition; zero pages for cause-only signals.)*
88. Rolling vs blue/green vs canary vs flags — decision tree. *(Your 9.9 measured table.)*
89. Design CI/CD for 100 engineers: gates, speed, safety. *(PR checks <10min, dynamic test selection, preview envs, trunk-based, progressive delivery, DORA metrics, platform team owns the golden path.)*
90. Design a DR program (not a plan — the program). *(tiers, RPO/RTO per journey, drill cadence, one-click automation, dependency inventory, game days, postmortems of drills.)*
91. Chaos engineering: how do you start at a company that's scared of it? *(staging first, hypothesis discipline, tiny blast radius, abort conditions, publish learnings — fear dies from evidence.)*
92. Capacity planning methodology. *(15.9: knee-finding load tests, Little's Law queues, growth forecasting, headroom policy, festival pre-scaling.)*
93. How do you reduce toil systematically? *(measure it (ticket tags), automate the top offender weekly (your Friday habit), delete work before automating it; SRE book's <50% toil rule.)*
94. Observability stack selection: build vs buy. *(11.11 essay: Datadog time-to-value vs LGTM economics at scale; OTel as the portability insurance.)*
95. Design multi-tenant logging with compliance retention. *(11.7/11.8: Loki tenants, per-team access, audit namespace → OpenSearch, retention tiers, immutability.)*
96. How do you make 200 developers follow golden paths without mandate? *(Lead JD answer: make the paved road fastest (templates, preview envs, one-command deploys), measure adoption, treat devs as customers, docs+workshops, deprecate gently.)*
97. Explain GitOps to a skeptical senior sysadmin. *(Reconciliation vs push; drift healing; audit trail; credential containment; "your hand-edit will be reverted and logged" as a feature.)*
98. Design database migration strategy for zero-downtime. *(12.6 expand/contract + 12.5 clone rehearsals + DMS for hetero + fallback plans.)*
99. What would your first 30/60/90 days look like here? *(30: observe, access, runbooks, on-call shadow, ship one small fix; 60: own a service's reliability, one automation, alert hygiene pass; 90: lead an incident review, deliver one platform improvement, draft SLOs. Tailor to the JD in front of you — literally reference their stack.)*
100. "Why you, a network engineer, for this SRE role?" *(Your narrative: production incident instincts from networking (methodical L1-L7 debugging), already-operated infrastructure at scale, plus this program's evidence: repos, postmortems, measured DR drills. Then show the ticket log.)*

---

## Section 5 — Hands-on gauntlets (interviewer-watching-you type)

Practice each under 20 minutes, on your own cluster, saying your reasoning aloud the whole time:

1. **CrashLoopBackOff pod** (bad env var) — fix + prevent (schema validation).
2. **OOMKilled batch job** — diagnose 137, right-size from VPA rec.
3. **Failed rollout mid-progress** — read Progressing condition, undo safely.
4. **Service with zero endpoints** — selector typo hunt.
5. **DNS NXDOMAIN cluster-wide** — CoreDNS down; then the ndots variant.
6. **Pending pods with free-looking capacity** — taints/affinity/quota trio.
7. **Ingress 502s on one path only** — service port mismatch vs probe confusion.
8. **Node NotReady** — kubelet down; eviction behavior; PDB-aware recovery.
9. **NetworkPolicy blackhole incl. DNS** — rebuild the allow matrix live.
10. **Terraform: state drift + lock + import** — the triple combo drill.
11. **PromQL live:** write error-ratio, p95 latency, and burn-rate queries from scratch.
12. **AWS CLI live:** find the instance that changed SG rules at 14:00 (CloudTrail from CLI), and produce a cost-by-service report.
13. **Broken Dockerfile:** make it small, cached, non-root, live on the spot.
14. **SQL live:** find and fix the slow query (EXPLAIN, index) against a 500k-row table.
15. **Git live:** bisect a planted bug; recover a "lost" commit via reflog; craft the hotfix cherry-pick flow.

**Gauntlet rules:** narrate hypotheses before commands ("I expect X because Y; if not, next I'll check Z"). Interviewers score the *reasoning path*, not the typing.

---

## Section 6 — Behavioral & leadership (12, with your evidence hooks)

1. "A time you broke production." → your INC-0023 PITR postmortem; emphasize detection, honesty, prevention.
2. "Disagreement with a teammate about approach." → Helm-vs-Kustomize ADR debate story; decision records as conflict resolution.
3. "Automated something that saved real time." → Friday automations; quantify hours/month.
4. "Taught/mentored others." → runbooks + workshops you ran for "the team" (your simulated org counts as practice telling it).
5. "Incident where you were wrong." → a drill where your first hypothesis failed; how the ladder saved you.
6. "Balancing speed vs safety." → error-budget policy; change windows + emergency path.
7. "A cost-saving you drove." → 15.11 numbers.
8. "Pushing back on a risky request." → data-residency SCP story; compliance as enabler.
9. "Your on-call philosophy." → 15.5 essay: sustainable, alert hygiene, blameless.
10. "Influence without authority." → golden-path adoption strategy (Q96).
11. "A project end-to-end you're proud of." → the DR drill program or the multicloud BGP build — pick by the interviewer's JD.
12. "Where do you want to grow?" → Go tooling depth, mesh multi-cluster, org-level reliability leadership.

---

## Section 7 — Mock interview protocol (weeks 24+)

1. **Weekly self-mock:** pick 5 scenario Qs at random, answer aloud timed, self-score vs model points; log weak spots → next week's tickets revisit that module.
2. **Monthly full mock (2 hrs):** 30 min behavioral, 45 min scenarios, 30 min hands-on gauntlet (pick 2), 15 min "questions for them" practice (ask about their SLO maturity, on-call load, deploy cadence, platform team — senior candidates interview back).
3. **Whiteboard reps:** draw from memory, ≤ 10 min each: (a) NorthPay full AWS architecture; (b) request path internet→pod with every failure point annotated; (c) multicloud BGP topology; (d) CI/CD+GitOps flow commit→canary→rollback; (e) DR failover sequence with timings.
4. **The portfolio walkthrough:** rehearse a 10-minute tour of your GitHub org + Grafana + one postmortem + the DR drill video. End with DORA/MTTR trend graphs from your 90-day review.

**You are interview-ready when:** any Section 1–4 question gets a structured 3-minute answer with a personal-evidence hook; gauntlets are sub-20-min; the five whiteboards are muscle memory.
