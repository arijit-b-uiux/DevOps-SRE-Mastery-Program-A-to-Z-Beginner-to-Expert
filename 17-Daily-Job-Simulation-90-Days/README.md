# Track B — The NorthPay Job Simulation: 90 Days of Tickets, Incidents & Standups

> This is the "live in a day-to-day job" track. You are NorthPay's first DevOps/SRE engineer. Every workday: **standup entry → dashboard sweep → tickets → module exercise → shutdown log**. Tickets assume module progress (noted per week); if you're ahead/behind, drag the board, don't skip tickets.
> **Rules of engagement:** tickets have acceptance criteria (AC) but deliberately incomplete context — like real ones. Ask "the business" (= decide reasonably, then *document your assumption in the ticket*). Estimates are points, not hours: you estimate before starting, compare after (calibration practice).

---

## How the board works

- **Ticket types:** `TASK` (build/work), `INC` (incident — drop everything, run the incident process, postmortem if SEV1/2), `CHG` (change request — needs plan + rollback + window), `SVC` (service request from a colleague), `DEBT` (tech debt), `SEC` (security/compliance).
- **Every INC** gets: timeline, and SEV classification practice. Every **CHG** gets the change-request template. Every Friday: retro + one toil-killing automation.
- **Recurring daily duties (once the stack exists):** check dashboards/alerts; check DLQ depths; check nightly backup job success; check cost script output; check cert expiry panel; note anything odd in the log. These take 15 min and build the "operational guardian" instinct from the Deel SRE JD.
- **Your manager writes you weekly.** Each week below opens with that note.

---

## WEEK 1 — "Welcome aboard" (Module 01/02 era)

**Manager note:** "Get your bearings. Accounts, guardrails, and I want to trust our backups before we build anything on top. Also — finance already asked me what this lab costs."

| Day | Tickets |
| --- | --- |
| 1 | **NP-0001** `TASK` Bootstrap access + budget alarms (Ex 0.1/0.2). AC: alarms tested, you never log in as root again. **NP-0002** `TASK` Repo layout + GitHub org (Ex 0.3). |
| 2 | **NP-0003** `TASK` On-prem sim VMs built and hardened by hand (Ex 0.4). AC: SSH keys only, ufw on, doc in runbooks. **NP-0004** `TASK` App smoke test via compose (Ex 0.5); write the one-line-per-service failure-impact list. |
| 3 | **NP-0005** `TASK` Linux drills: process states, signals, D-state NFS repro (Ex 2.1–2.2). **NP-0006** `SVC` "Finance wants yesterday's spend by service" → build `np-cost-today` (Ex 2.14.5) and email (log) the answer. |
| 4 | **NP-0007** `INC` *First blood:* the legacy-dc app is "slow." (It's the planted disk-fill from `fallocate`.) Triage with Module 02 tools, fix, 5-line postmortem. **NP-0008** `TASK` systemd unit for legacy app + journald persistence (Ex 2.4). |
| 5 | **NP-0009** `TASK` `np-health` + `np-backup-pg` scripts (Ex 2.14); cron nightly backup. AC: restore test (!) on day 8. **Friday retro + toil automation #1** (suggestion: the health check as a cron'd Nagios-exit-code script writing to a status file). |

## WEEK 2 — "Scripting & networking baseline" (Module 02 era)

**Manager note:** "Auditors will eventually ask who can touch what. Also our legacy DB backups scare me. And get that text-processing speed up — logs are coming."

| Day | Tickets |
| --- | --- |
| 6 | **NP-0010** `TASK` Text gauntlet on sample-access.log (Ex 2.13): top IPs, error rate/min, p95 estimate. |
| 7 | **NP-0011** `SEC` sudoers tightening + auditd watch on secret paths (Ex 2.7). AC: prove deploy user can't edit unit files; audit event captured. |
| 8 | **NP-0012** `TASK` **Restore drill #1:** restore Monday's pg backup to a scratch DB, verify row counts, time it. (Backup trust established.) **NP-0013** `TASK` clock-skew TLS repro + runbook (Ex 2.8.3). |
| 9 | **NP-0014** `TASK` Python: retry decorator + flaky-endpoint demo (Ex 2.15). **NP-0015** `TASK` boto3 inventory script (Ex 2.16.1). |
| 10 | **NP-0016** `TASK` netns/veth/bridge/NAT-by-hand lab (Ex 2.5) — write the "containers are namespaces" note. **Friday retro + automation #2** (e.g., inode-watch cron on all hosts). |

## WEEK 3 — "Cloud week one: foundations" (Module 03 Part A–B)

**Manager note:** "We got the green light for real AWS work. Security wants least-privilege from day one. Do NOT hand me a surprise bill."

| Day | Tickets |
| --- | --- |
| 11 | **NP-0017** `TASK` IAM roles/policies + policy simulator lab (Ex 3.1–3.2). **NP-0018** `SEC` Access Analyzer run; fix the wildcard-principal finding. |
| 12 | **NP-0019** `TASK` Build np-vpc by hand: subnets, IGW, NAT, SGs, flow logs (Ex 3.4). AC: connectivity matrix proven (public OK / private-NAT OK / data-no-internet OK). |
| 13 | **NP-0020** `TASK` VPC endpoints (S3 gateway + SSM interfaces), private DNS proof (Ex 3.5). **NP-0021** `DEBT` Record single-AZ NAT as debt ticket with options+prices. |
| 14 | **NP-0022** `CHG` Multi-AZ NAT rollout + failure drill (Ex 3.6) — first real change request: plan, window, rollback. **NP-0023** `TASK` Session Manager only; disable key SSH (Ex 3.3) + session logging to S3. |
| 15 | **NP-0024** `INC` "Something in the private subnet can't call the SQS API." (You removed a route in yesterday's drill — flow-logs forensics.) Fix + postmortem + NACL/SG doc update. **Friday retro + automation #3** (flow-log top-rejected query saved). |

## WEEK 4 — "Three tiers, TLS, and a failover" (Module 03 Part C–E)

| Day | Tickets |
| --- | --- |
| 16 | **NP-0025** `TASK` EC2 app node from user-data + IMDSv2 enforced (Ex 3.7). |
| 17 | **NP-0026** `TASK` ALB+ASG 3-AZ build (Ex 3.8 steps 1–4). |
| 18 | **NP-0027** `INC` Instance killed at 14:07 (you, via chaos script, unannounced-ish). Measure detection→replacement time; tune health checks; log the RTO number. |
| 19 | **NP-0028** `TASK` ACM + HTTPS + redirect + TLS policy lock (Ex 3.9). **NP-0029** `SVC` "Support needs an order-lookup page" → web tier behind ALB path rule. |
| 20 | **NP-0030** `CHG` Route53 failover policy build + measured DNS failover (Ex 3.12.2). AC: failover time written down. **Friday retro + automation #4** (ALB log → awk report cron). |

## WEEK 5 — "Data and queues" (Module 03 Part F–H)

| Day | Tickets |
| --- | --- |
| 21 | **NP-0031** `TASK` RDS Postgres + Secrets Manager rotation + IAM DB auth (Ex 3.15). |
| 22 | **NP-0032** `TASK` Load seed data; app on RDS; `/readyz` green. **NP-0033** `SEC` Prove db SG has no 0.0.0.0/0; write the evidence into the security file. |
| 23 | **NP-0034** `INC` *Fat-finger Friday came early:* rows deleted from orders at 10:12. PITR drill (Ex 3.16). Full postmortem (real one — template). |
| 24 | **NP-0035** `TASK` SQS settlements queue + worker consume + visibility-timeout duplicate repro + fix (Ex 3.13). |
| 25 | **NP-0036** `TASK` DLQ + poison-pill + redrive + alarm (Ex 3.13.3). **NP-0037** `TASK` SNS topic `np-alerts` wired to CloudWatch alarms (Ex 3.14). **Friday retro + automation #5** (DLQ-depth cron check). |

## WEEK 6 — "CloudWatch week + containers begin" (Module 03 Part H → Module 04)

| Day | Tickets |
| --- | --- |
| 26 | **NP-0038** `TASK` CW agent + custom metric + `np-overview` dashboard (Ex 3.18). |
| 27 | **NP-0039** `TASK` Composite alarms + alarm-fatigue cleanup: classify all current alerts symptom/cause. |
| 28 | **NP-0040** `TASK` EventBridge rules: state-change→SNS, nightly stop schedule (Ex 3.19). |
| 29 | **NP-0041** `TASK` Container-by-hand lab (Ex 4.1). Write the namespace/cgroup mapping table. |
| 30 | **NP-0042** `TASK` Dockerfiles hardened: multi-stage, non-root, distroless variant, trivy gates (Ex 4.4–4.5). **Friday retro + automation #6** (orphaned-EIP/EBS sweeper script). |

## WEEK 7 — "Kubernetes, crash course with your name on it" (Module 05 Part A)

| Day | Tickets |
| --- | --- |
| 31 | **NP-0043** `TASK` kind 3-node + control-plane tour + etcd snapshot (Ex 5.1). |
| 32 | **NP-0044** `TASK` Deploy api/worker/web raw manifests; probes done right (Ex 5.2–5.3). |
| 33 | **NP-0045** `INC` Broken image tag deployed (ImagePullBackOff). `rollout status` gate + undo (Ex 5.3.4). Postmortem: why did CI let a bad tag exist? (Seed for Module 09.) |
| 34 | **NP-0046** `TASK` Services + DNS + the debug ladder runbook (Ex 5.4). |
| 35 | **NP-0047** `TASK` Ingress + TLS + rate limit (Ex 5.5). **Friday retro + automation #7** (endpoint-vs-selector checker script). |

## WEEK 8 — "Day-2 Kubernetes" (Module 05 Part B–C)

| Day | Tickets |
| --- | --- |
| 36 | **NP-0048** `TASK` Resources/QoS + OOMKill lab (Ex 5.8). |
| 37 | **NP-0049** `INC` Batch job OOM-loops at 03:00 (chaos cron). Fix = right-size + restartPolicy reasoning. Postmortem with kernel-vs-kubelet OOM explanation. |
| 38 | **NP-0050** `TASK` HPA on CPU + on queue depth via KEDA (Ex 5.9). |
| 39 | **NP-0051** `TASK` PDB + drain drill; anti-affinity pending-puzzle (Ex 5.10). |
| 40 | **NP-0052** `SEC` RBAC personas + NetworkPolicy default-deny + Kyverno policies (Ex 5.11–5.12). **Friday retro + automation #8** (pending-pods alerter). |

## WEEK 9 — "EKS: it's real now" (Module 05 Part E) [COST week — watch the budget]

| Day | Tickets |
| --- | --- |
| 41 | **NP-0053** `CHG` EKS cluster build (console once), VPC CNI study, prefix delegation math (Ex 5.17). |
| 42 | **NP-0054** `TASK` ALB controller + ACM + external-dns; api live at https://api.northpay… (Ex 5.17.6). |
| 43 | **NP-0055** `SEC` IRSA for worker; strip node role; prove isolation (Ex 5.18). |
| 44 | **NP-0056** `TASK` Karpenter + spot + consolidation evidence (Ex 5.17.7). |
| 45 | **NP-0057** `INC` Pods Pending: CNI IP exhaustion planted (Ex 5.20.1). Diagnose, fix (prefix delegation), runbook. **Friday retro + automation #9** (EKS orphan-ENI sweeper). |

## WEEK 10 — "Upgrade season + Ansible" (Module 05 Part E + 08)

| Day | Tickets |
| --- | --- |
| 46 | **NP-0058** `CHG` EKS N-1→N upgrade drill incl. stall scenario (Ex 5.19). Zero-downtime evidence required. |
| 47 | **NP-0059** `TASK` Ansible inventory (dynamic AWS + static onprem) + baseline role (Ex 8.1–8.2). |
| 48 | **NP-0060** `TASK` nginx + node-exporter roles; site.yml converges onprem VMs (Ex 8.3). |
| 49 | **NP-0061** `SEC` Ansible Vault everywhere; no_log audit; gitleaks gate (Ex 8.4). |
| 50 | **NP-0062** `CHG` Legacy-dc Postgres minor-version patch window, Ansible-orchestrated (Ex 8.7.3). **Friday retro + automation #10** (ansible-lint in pre-commit). |

## WEEK 11 — "Everything as code, or it dies" (Module 07 Part A–B)

**Manager note:** "New rule from me (your manager): console is read-only from Monday. Migrate us. Also — I got asked what happens to our infra if *you* get hit by a bus. State + docs are the answer."

| Day | Tickets |
| --- | --- |
| 51 | **NP-0063** `TASK` TF bootstrap backend + locking; migrate first resources (Ex 7.1–7.2). |
| 52 | **NP-0064** `TASK` VPC module authored; cidrsubnet math; two envs from one module (Ex 7.3). |
| 53 | **NP-0065** `TASK` Import legacy console VPC → clean plan (Ex 7.2.4). |
| 54 | **NP-0066** `INC` State surgery: kill a resource in console, drift detected, recover via import (Ex 7.2.3 + runbook). |
| 55 | **NP-0067** `TASK` tf gates: fmt/tflint/checkov/conftest/infracost as `make tf-check` (Ex 7.4). **Friday retro + automation #11** (nightly drift-detection cron). |

## WEEK 12 — "The great rebuild" (Module 07 Part C)

| Day | Tickets |
| --- | --- |
| 56 | **NP-0068** `CHG` network stack in TF; blue/green VPC migration plan (Ex 7.7). |
| 57 | **NP-0069** `CHG` compute + data stacks in TF; secrets via SM only (Ex 7.8). |
| 58 | **NP-0070** `CHG` EKS + IRSA in TF (Ex 7.9); destroy-proof = apply from empty twice. |
| 59 | **NP-0071** `DEBT` Console-resource tag audit → zero unmanaged resources (exit-gate check). |
| 60 | **NP-0072** `TASK` TF native tests + one Terratest (Ex 7.11). **Friday retro + automation #12** (orphan cleanup in TF CI). |

## WEEK 13 — "Pipelines: make shipping boring" (Module 09 Part A–B)

| Day | Tickets |
| --- | --- |
| 61 | **NP-0073** `TASK` GH Actions CI: lint/test/coverage gate/matrix/caching (Ex 9.1–9.2 steps 1–2). |
| 62 | **NP-0074** `SEC` OIDC to AWS (no keys!) + cosign sign + SLSA attest (Ex 9.2.3–4). |
| 63 | **NP-0075** `TASK` trivy gate + SBOM + changelog automation (Ex 9.2). |
| 64 | **NP-0076** `TASK` Jenkins from JCasC; docker+k8s agents (Ex 9.5). |
| 65 | **NP-0077** `TASK` Jenkinsfile + shared library + approval gate (Ex 9.6). **Friday retro + automation #13** (flaky-test quarantine job). |

## WEEK 14 — "Preview envs & deployment strategies" (Module 09 Part C)

| Day | Tickets |
| --- | --- |
| 66 | **NP-0078** `TASK` PR preview environments + comment bot (Ex 9.4). |
| 67 | **NP-0079** `TASK` TTL reaper + per-PR quotas (Ex 9.4.3–4). |
| 68 | **NP-0080** `INC` A preview env leaked prod-like data (planted: someone seeded it with a prod dump copy). Contain, rotate, write the data-handling policy. (SEV2 — data exposure drill.) |
| 69 | **NP-0081** `TASK` Strategy lab: rolling/blue-green/manual-canary measured (Ex 9.9). |
| 70 | **NP-0082** `TASK` Change windows + emergency-change path as code (Ex 9.10). **Friday retro + automation #14** (deploy-frequency/lead-time metrics export — DORA start). |

## WEEK 15 — "GitOps conversion" (Module 10 Part A)

| Day | Tickets |
| --- | --- |
| 71 | **NP-0083** `TASK` ArgoCD install + self-management + RBAC/SSO (Ex 10.1). |
| 72 | **NP-0084** `TASK` All apps as Applications; selfHeal/prune policy per env (Ex 10.2). |
| 73 | **NP-0085** `INC` Hand-edited prod Deployment "fixed" by someone (you) at lunch — selfHeal reverted it and the hotfix is gone. Handle the confusion, then write the "why hand-edits die" note + emergency-change path. |
| 74 | **NP-0086** `TASK` Sync waves + PreSync migration hooks (Ex 10.3). |
| 75 | **NP-0087** `TASK` Promotion PRs dev→staging→prod + digest pinning (Ex 10.4). **Friday retro + automation #15** (OutOfSync>30m alert). |

## WEEK 16 — "GitOps at scale + rollouts" (Module 10 Part B)

| Day | Tickets |
| --- | --- |
| 76 | **NP-0088** `TASK` ApplicationSets + second cluster registered (Ex 10.5). |
| 77 | **NP-0089** `SEC` External Secrets Operator; SM rotation propagation proof (Ex 10.6). |
| 78 | **NP-0090** `TASK` Argo Rollouts canary + analysis + **auto-rollback demo** (Ex 10.7). Record it (portfolio artifact #1). |
| 79 | **NP-0091** `INC` Bad release at 11:20 — canary analysis catches it, auto-rollback. Verify the machine worked; postmortem focuses on *why the bug passed CI* (add the missing test). |
| 80 | **NP-0092** `TASK` ArgoCD ops: HA design, backup, webhook, AppProjects (Ex 10.8). **Friday retro + automation #16** (promotion-PR opener bot polish). |

## WEEK 17 — "Observability: see everything" (Module 11 Part A–D)

| Day | Tickets |
| --- | --- |
| 81 | **NP-0093** `TASK` kube-prometheus-stack via ArgoCD; ServiceMonitor for all services (Ex 11.1). |
| 82 | **NP-0094** `TASK` PromQL golden-signals dashboard + recording rules + rule unit tests (Ex 11.2–11.3). |
| 83 | **NP-0095** `TASK` Alertmanager routing/inhibition/silences + dead-man's switch (Ex 11.4). |
| 84 | **NP-0096** `INC` 2 AM page (chaos cron mercy-off): node down → ONE page (inhibition works) → drain+replace → timeline + postmortem. |
| 85 | **NP-0097** `TASK` Loki + Alloy; LogQL fluency; logs→metrics maturation of 2 alerts (Ex 11.7). **Friday retro + automation #17** (cardinality report script). |

## WEEK 18 — "Traces, Mimir, Datadog, Zabbix" (Module 11 Part C–F)

| Day | Tickets |
| --- | --- |
| 86 | **NP-0098** `TASK` OTel + Tempo + spanmetrics + exemplar click-through (Ex 11.9). Record demo (artifact #3). |
| 87 | **NP-0099** `TASK` Mimir deploy + remote-write + tenants + ruler (Ex 11.6). |
| 88 | **NP-0100** `TASK` Datadog trial: agent, APM, synthetics, SLO UI (Ex 11.11). Start the comparison essay. |
| 89 | **NP-0101** `TASK` Zabbix on legacy-dc + SNMP of network device (Ex 11.12). |
| 90 | **NP-0102** `INC` **The capstone mystery alert** (Ex 11.14): one burn-rate page, unknown cause. MTTD/MTTR measured; postmortem with full evidence chain. **Friday retro: 90-day review — DORA metrics, MTTR trend, toil eliminated count, cost trend. Write it. This document becomes interview material.** |

---

## WEEK 19+ — "Keep living the job" (Modules 12–16 era)

The board continues procedurally — every week from your module work, generate tickets in these patterns (a generator list you reuse):

- **DB weeks:** CHG "Aurora migration rehearsal on clone" (Ex 12.5); INC "Mongo primary died during batch" (12.7.4); TASK "ES reindex to fix mapping" (12.9.3); INC "redis eviction storm" (12.12.3); TASK "DynamoDB hot-partition fix" (12.11.2); SEC "DB encryption-at-rest audit" (13.8.2).
- **Network/security weeks:** TASK "TGW route-domain isolation" (13.3); CHG "Hybrid VPN + BGP build" (13.4) — the multi-day flagship; TASK "WAF rate-limit tuning after attack simulation" (13.7); SEC "quarterly access review" (13.10.4); INC "tunnel 1 down at 06:00 — did tunnel 2 take over? prove it" (13.4.4).
- **Multicloud weeks:** TASK "GKE parity deploy" (14.3); CHG "AWS↔GCP HA VPN" (14.8); INC "cross-cloud latency doubled — iperf3 + BGP forensics"; TASK "three-cloud cost report v1" (14.11.5); CHG "DNS-steered failover drill" (14.10.2).
- **SRE weeks:** TASK "SLO definitions + error-budget policy" (15.1); CHG "one-click DR drill" (15.7); INC from chaos catalog weekly (15.8); TASK "capacity model before festival season" (15.9); CHG "AMI rotation pipeline rollout" (15.12).
- **Mesh weeks:** CHG "ISTIO STRICT mTLS migration" (16.3 — multi-day, wave-based); TASK "header-canary for partner API" (16.2.3); INC "istiod down — what broke, what didn't" (16.5.3).

**Standing weekly tickets (forever):** Monday: cost report + week plan. Tuesday: patch/vuln triage (ECR scans + Inspector + OS updates). Wednesday: backup/restore spot-check (rotate engines). Thursday: alert-quality review (any noisy alert? tune or ticket). Friday: retro + one automation + runbook updates. Monthly: DR component drill, access review, capacity review, compliance evidence refresh.

---

## How to use this history in interviews

By Day 90 you'll have ~100 closed tickets, ~15 INC postmortems, weekly retros, and metrics trends. When an interviewer says *"tell me about a production incident you handled"* — you'll pick a real one from your log with detection source, MTTR, root cause, and the prevention that followed. That specificity is what 6-years-experience answers sound like.
