# DevOps & SRE Mastery Program — A-to-Z, Beginner to Expert

**Built for:** a network engineer (6+ yrs: firewalls, VMware, Cisco switching/routing, complex routing protocols) moving into DevOps / SRE / Platform Engineering roles.
**Target roles (from the 6 attached JDs):** DevOps Engineer (Deel), Site Reliability Engineer (Deel), SRE-2 (slice), SRE-3 (slice), Lead Platform Engineer (slice), Senior Staff Engineer – Infrastructure (slice).
**Starting state:** a fresh AWS account with nothing running. All data, code, and manifests are generated for you in `assets/` — import and go.
**End state:** you can design, build, operate, break, fix, and defend a production-grade, multi-cloud, Kubernetes-based platform — and explain the *why, when, what, and tradeoffs* of every choice in an SRE interview.

---

## 1. How this program works

Everything you do happens inside one continuous story: you are the **first DevOps/SRE hire at "NorthPay"**, a fictional fintech/payroll company (deliberately similar to slice + Deel: regulated, multi-account AWS, EKS, GitOps, strict uptime). Every exercise, ticket, and incident belongs to that story, so by the end you don't just know tools — you have *lived* a realistic year of DevOps work in compressed form.

Two tracks run in parallel:

1. **Module track (this folder, files 01–16):** structured exercises per topic, Beginner → Expert. Each exercise is self-contained: goal → why it matters (mapped to a JD line) → concepts → exact steps → validation → tradeoffs.
2. **Job-simulation track (`17-Daily-Job-Simulation-90-Days.md`):** a 90-day board of tickets, incidents, change requests, and standups that force you to *use* the module skills the way a job does — repetitively, under time pressure, with incomplete information.

Do both tracks together: the module for the week teaches the skill; the daily tickets make you apply it.

### Daily operating rhythm (every "workday")

1. **Standup (10 min, written):** open your `daily-log.md`, write yesterday/today/blockers. Interviewers *will* ask how you communicate; practice it here.
2. **Dashboard sweep (15 min):** check whatever observability stack exists so far. Note anything odd, even if no ticket exists. SRE instinct is trained, not read.
3. **Tickets (2–4 hrs):** work the day's tickets from the simulation board.
4. **Module exercise (1–2 hrs):** the current module's next exercise.
5. **Shutdown log (10 min):** what changed, what broke, cost estimate for the day, resources left running.
6. **Weekly Friday retro:** what was toil? Automate one painful thing every Friday. This habit *is* the SRE mindset.

### The golden rules (enforce these on yourself from Day 1)

- **Nothing by hand that can be code.** Console clicks are allowed only in Phase 2 while learning; by Phase 4, every AWS change must be Terraform or a documented exception.
- **Every change is reviewable.** Git commit for everything, even scripts. By Phase 7, Git is the *only* source of truth (GitOps).
- **You break it, you document it.** Every failure gets a 5-line mini-postmortem in `postmortems/`. By Phase 12 these become full blameless postmortems.
- **Cost is a feature.** Budget alerts on Day 1. Every module ends with a teardown checklist. An SRE who burns money silently fails interviews at fintech companies.
- **Runbooks or it didn't happen.** Every repeated task ends with a runbook entry in `runbooks/`.

---

## 2. Program map — phases, modules, exit criteria

| Phase | Weeks | Module file | What you master | Exit gate (you may not skip these) |
|---|---|---|---|---|
| 0 | Days 1–5 | `01-Setup-and-Prerequisites.md` | Accounts, tooling, cost guardrails, Git baseline, NorthPay bootstrap | Budget alarms fire; CLI works; repo structure exists |
| 1 | Wk 1–2 | `02-Linux-Git-Scripting.md` | Linux internals, Bash, Python automation, Git workflows, cloud networking refresh | You debug a hung process, full disk, and bad systemd unit without googling basics |
| 2 | Wk 2–4 | `03-AWS-Foundations.md` | IAM, VPC, EC2/ASG, S3, RDS, ELB, Route53, CloudWatch, SQS/SNS/SES, ECR | NorthPay 3-tier app runs manually on AWS, multi-AZ |
| 3 | Wk 5–6 | `04-Containers-Docker.md`, `05-Kubernetes.md` | Docker deep, K8s core → day-2 (CrashLoopBackOff, OOMKill, DNS, failed rollouts) | kind cluster runs the full app; you debug 10 injected failures |
| 4 | Wk 7–8 | `07-Terraform-IaC.md` | HCL, state, modules, workspaces, remote state, TF SDLC, GitOps-for-Terraform | Entire AWS estate rebuilds from `terraform apply`; nothing in console |
| 5 | Wk 9 | `08-Ansible.md` | Inventory, playbooks, roles, vault, idempotency, AWX | All EC2 config (nginx, monitoring agents) via Ansible |
| 6 | Wk 10–11 | `09-CI-CD-Pipelines.md` | GitHub Actions, Jenkins (self-hosted), Spinnaker concepts, quality gates, artifact management | Commit → test → build → scan → push → deploy pipeline for all services |
| 7 | Wk 12–13 | `06-Helm-and-Kustomize.md`, `10-GitOps-ArgoCD.md` | Helm authoring, Kustomize overlays, ArgoCD App-of-Apps, sync waves, rollback | ArgoCD manages every environment; Git push = deploy |
| 8 | Wk 14–15 | `11-Observability.md` | Prometheus, Grafana, Mimir, Loki, Tempo, Alertmanager, Datadog, Zabbix, OpenSearch/ELK | Full LGTM stack on EKS; SLO dashboards; alert routing; Datadog trial comparison |
| 9 | Wk 16–17 | `12-Databases.md` | PostgreSQL (RDS/Aurora + self-hosted), MongoDB/DocumentDB, Elasticsearch/OpenSearch, DynamoDB, ElastiCache, backup/restore drills | You run PITR restore, Mongo replica failover, ES reindex under load |
| 10 | Wk 18 | `13-AWS-Advanced-Networking-and-Security.md` | Transit Gateway, PrivateLink, hybrid DNS, multi-account AWS Organizations, WAF/Cloudflare, KMS, secrets, compliance guardrails (RBI/PCI-style) | Multi-account hub-and-spoke network with shared services; security baseline enforced by code |
| 11 | Wk 19–20 | `14-Multicloud.md` | GCP + Azure core services, GKE/AKS, multicloud networking (VPN/BGP, anycast, multi-cloud LB), portable Terraform | Same app runs on AWS + GCP with cross-cloud connectivity and one Terraform repo |
| 12 | Wk 21–22 | `15-SRE-Reliability-Engineering.md` | SLI/SLO/error budgets, incident command, blameless postmortems, DR (RPO/RTO), chaos engineering, capacity planning, FinOps | Full DR drill executed + measured; 3 real postmortems written; error-budget policy adopted |
| 13 | Wk 23 | `16-Service-Mesh-Istio.md` | Istio traffic management, mTLS, canary, observability | mTLS everywhere; 5/95 canary with automatic rollback |
| 14 | Wk 24+ | `18-SRE-Interview-Prep.md` + capstones | Scenario questions, hands-on debug gauntlets, system-design for SRE | 2 timed mock interviews passed; capstone demo recorded |

**Track-B overlay:** `17-Daily-Job-Simulation-90-Days.md` (start Day 1, in parallel with modules).

---

## 3. JD requirement → where it's covered

Every line of the six JDs is covered somewhere. Spot-check:

| JD requirement | Covered in |
|---|---|
| Docker & Kubernetes strong working experience (Deel DevOps) | Modules 04, 05; tickets throughout |
| AWS EKS, S3, Postgres, Mongo, SES, SNS, SQS (Deel DevOps) | Modules 03, 12; Phase 10 |
| CI/CD tools — Drone/Jenkins/GitHub Actions (Deel DevOps) | Module 09 |
| K8s ingress, services, Load Balancers (Deel DevOps) | Modules 05, 13 |
| Helm + ArgoCD hands-on (Deel SRE) | Modules 06, 10 |
| Datadog, Grafana, Mimir, Loki, Tempo, Zabbix (Deel SRE) | Module 11 |
| Incident triage, escalation, post-incident reviews (Deel SRE) | Module 15; simulation board weeks 6+ |
| Terraform expertise, reusable modules, TF SDLC (slice SRE-3) | Module 07; Phase 10 |
| Istio service mesh (slice SRE-3) | Module 16 |
| Multi-AZ/region HA, DR with RPO/RTO (slice SRE-3, SRE-2) | Modules 03, 12, 13, 15 |
| Spinnaker/Jenkins/ArgoCD (slice Staff/Lead) | Modules 09, 10 |
| SLOs, SLIs, error budgets (slice Staff) | Module 15 |
| One-click DR drills, AMI rotation, cost baselines (slice Staff) | Modules 03, 15; tickets |
| Ephemeral environments (slice Staff/Lead) | Modules 09, 10 (PR preview envs) |
| Test automation, code quality gates (slice Staff/Lead) | Module 09 |
| CrashLoopBackOff, OOMKill, failed rollout, in-cluster DNS (slice SRE-2) | Module 05; interview gauntlet 18 |
| Linux depth: hung process, inode exhaustion, systemd (slice SRE-2) | Module 02 |
| Prometheus/Grafana, ELK/OpenSearch (slice SRE-2) | Module 11 |
| Bash required, Python preferred (slice SRE-2) | Module 02; everywhere |
| Multi-account AWS, least-privilege, JIT access, audit trails (slice SRE-2) | Modules 03, 13 |
| Compliance guardrails RBI/NPCI/PCI-style (slice SRE-2) | Module 13 |
| PostgreSQL, DynamoDB, Elasticsearch (slice SRE-2) | Module 12 |
| Multicloud exposure (slice SRE-2) | Module 14 |
| nginx/Apache (slice SRE-2) | Modules 02, 03, 05 |
| Go/Python tooling (multiple JDs) | Module 02 (Python); Go pointers in 18 |
| Node.js advantage (Deel) | Sample app is Node.js (assets) |

---

## 4. The NorthPay end-state architecture (what you'll have built)

```text
                         ┌──────────── Route53 / Cloudflare DNS ────────────┐
                         │                                                  │
                    AWS org: NorthPay                                GCP project / Azure sub
   ┌─────────────────────┼───────────────────────────┐              ┌──────────────────────┐
   │ mgmt account        │  prod account             │              │ GKE cluster (Phase 11)│
   │  - CloudTrail hub   │   VPC 10.10.0.0/16        │              │  same app, Helm       │
   │  - budgets, SSO     │   EKS prod (3 AZ)         │◄───VPN/BGP──►│  Cloud SQL (Postgres) │
   │                     │   RDS Aurora Postgres     │   Phase 11   └──────────────────────┘
   │ nonprod account     │   ElastiCache, S3, SQS    │
   │  VPC 10.20.0.0/16   │   MSK-less: SQS/SNS/SES   │
   │  EKS staging +      │   ALB/NLB + Istio mesh    │
   │  ephemeral PR envs  │   LGTM: Mimir/Loki/Tempo  │
   └───────── Transit Gateway hub-and-spoke ─────────┘
        GitHub org: app repos + northpay-infra (Terraform) + northpay-gitops (ArgoCD)
        Jenkins (self-hosted, later Spinnaker trial) │ GitHub Actions for app CI
        Datadog trial + self-hosted Grafana LGTM + Zabbix (comparison lab)
        On-prem simulation: 2 VMs (your VMware) = "branch office" + "legacy DC"
```

By the end you can whiteboard this from memory — that whiteboard *is* the answer to half of SRE system-design interviews.

---

## 5. What's in `assets/`

| Path | Contents | First used |
|---|---|---|
| `assets/app/` | NorthPay demo platform: `api` (Node.js/Express + Postgres + Mongo + Redis clients), `worker` (queue consumer), `web` (static nginx), Dockerfiles, docker-compose | Module 04 |
| `assets/seed-data/` | `postgres_seed.sql` (schema + 50k rows generator), `mongo_seed.js`, `elasticsearch_bulk.ndjson`, `dynamodb_seed.py` | Module 12 |
| `assets/k8s/` | Raw manifests for all app components + broken-manifest debug set | Module 05 |
| `assets/helm/northpay-chart/` | Starter Helm chart you improve through the program | Module 06 |
| `assets/terraform/` | Bootstrap (S3 backend), VPC module, EKS skeleton, security-baseline module | Modules 07, 13 |
| `assets/ansible/` | Inventory, baseline role, nginx role, vault example | Module 08 |
| `assets/ci/` | GitHub Actions workflows, Jenkinsfile, Jenkins docker-compose, quality-gate configs | Module 09 |
| `assets/scripts/` | `gen_load.sh`, `chaos/`, `backup/`, `cost_report.sh`, `pitr_drill.sh` | Throughout |
| `assets/templates/` | Runbook, postmortem, SLO definition, ADR, change-request templates | Day 1 onward |
| `assets/observability/` | Grafana dashboards JSON, Prometheus rules, Loki/Tempo values files, Datadog notes | Module 11 |

> **Note on file sizes:** seed-data generators create the bulk data on *your* machine (scripts included), so downloads stay small and you control volume.

---

## 6. Cost & safety contract (read once, follow always)

1. **Region discipline:** default region `ap-south-1` (Mumbai — matches the fintech story and Indian compliance angle). One secondary region `ap-southeast-1` only when a module says so.
2. **Budgets on Day 1:** \$30/month hard alarm, \$10 soft. If an alarm fires you stop and tear down — that incident discipline is itself interview material.
3. **Nightly teardown:** Phase 2–4 EC2/RDS resources get stopped/terminated nightly unless a ticket says "leave running" (some observability exercises need overnight metrics). A script `assets/scripts/nightly_teardown.sh` + tags (`northpay:ttl`) automate it.
4. **EKS is the expensive part:** run most Kubernetes work on local `kind`; EKS only for the exercises that require it (Phase 3 wk 6, then continuously from Phase 7). A 2-node `t3.medium` EKS cluster ≈ \$0.15/hr — schedule creation/teardown windows.
5. **Free-tier awareness:** the program flags exercises that exceed free tier before you run them.
6. **Never commit secrets.** `git-secrets` or `gitleaks` installed on Day 1; pre-commit hook enforced in Module 09 pipelines too.

---

## 7. How to prove you're done (portfolio artifacts)

Interviewers trust artifacts over claims. Finish the program with:

1. **A public (or shareable) GitHub org** containing: app repo with real CI, `northpay-infra` Terraform with module structure, `northpay-gitops` with ArgoCD apps, runbooks, and 5+ postmortems (sanitized).
2. **3 recorded Loom-style demos** (5 min each): (a) Git push → canary deploy → auto-rollback on SLO burn; (b) DR drill with measured RTO; (c) debugging a planted incident using only LGTM.
3. **A one-page architecture doc** (the whiteboard above, filled with your real resource names).
4. **Your daily log + ticket history** — 90 days of evidence you operate like a working SRE.

These four artifacts map directly to what Deel and slice JDs ask for: production ownership, GitOps discipline, observability depth, incident maturity.

---

## 8. Reading the exercise format

Every exercise in every module looks like this:

```text
EXERCISE 5.14 — Debug an OOMKilled pod            [Level: Intermediate]  [~45 min]
GOAL      : what you'll be able to do
JD LINK   : which job requirement this serves
CONCEPTS  : 3–8 lines of what to understand first
STEPS     : numbered, copy-pasteable, end-to-end
VALIDATE  : exact commands/outputs that prove success
TRADEOFFS : the why/when discussion — what an interviewer probes
WRECK-IT  : (many exercises) a controlled way to break it, then fix
```

Levels: **B**eginner (first contact), **I**ntermediate (production-plausible), **A**dvanced (multi-component, failure modes), **E**xpert (design-level, tradeoff-first, often whiteboard + build).

When a step says `TODO(student)` it's deliberate — you fill it in. Struggling there is the learning.

Now go to `01-Setup-and-Prerequisites.md`. Day 1 starts with a budget alarm and a Git repo — like every good SRE job.
