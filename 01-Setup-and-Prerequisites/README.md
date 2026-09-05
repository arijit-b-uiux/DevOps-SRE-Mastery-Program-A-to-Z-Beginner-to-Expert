# Phase 0 — Setup, Accounts, Tooling, and the NorthPay Bootstrap (Days 1–5)

> Everything here must be finished before any other module. Budget alarms and Git hygiene are not optional — they're Day-1 habits of working SREs.

---

## EXERCISE 0.1 — AWS account hardening baseline [B] [~60 min]

**GOAL:** turn a fresh AWS account into a safe workbench.
**JD LINK:** "least-privilege and just-in-time access as a natural way of working" (slice SRE-2).

**STEPS**

1. Sign in as root. Enable MFA on root immediately (virtual MFA app).
2. Create an administrative IAM Identity Center (SSO) user `you@northpay` in `ap-south-1`. Assign `AdministratorAccess` permission set for now (we tighten it in Module 13 — note that as tech debt in your log; acknowledging known debt is a senior habit).
3. Create IAM group `billing-admins`; attach `Billing` managed policy; add your SSO user. Root should never touch billing again.
4. Set account alias: IAM → Dashboard → `northpay-prod-lab` so sign-in URLs are readable.
5. Enable AWS Cost Explorer. Create two budgets (Cost Explorer console → Budgets):

- `np-soft-10usd` — \$10, alert at 80% actual, email to you.
- `np-hard-30usd` — \$30, alert at 100% actual AND 90% *forecasted*, email to you.

6. Create a Cost Anomaly Detection monitor (service: all, alert threshold \$5) with email subscription; confirm the subscription email.
7. Turn on CloudTrail (multi-region trail, log file validation ON) to a new S3 bucket `northpay-cloudtrail-<your-suffix>` with default encryption. This is your audit trail — required by the slice SRE-2 JD and every fintech.
8. Tag strategy: write into `daily-log.md` the tagging standard you'll enforce all program: `org=northpay`, `env=<dev|staging|prod>`, `owner=you`, `ttl=<date or none>`, `module=<NN>`. You'll automate enforcement later (Module 13 SCPs).

**VALIDATE**

- `aws sts get-caller-identity` (after Exercise 0.2) shows your SSO identity, not root.
- Budgets visible: `aws budgets describe-budgets --account-id <id>` (via CLI with your SSO profile).
- CloudTrail event appears in the S3 bucket within 15 min.

**TRADEOFFS / WHY-WHEN-WHAT**

- *Why Identity Center over IAM users?* Short-lived credentials, central for multi-account (you'll enable AWS Organizations in Module 13), no long-lived keys to leak. IAM users with access keys are legacy for humans; acceptable only for some CI use cases — and even there, OIDC (Module 09) is better.
- *Why budgets before servers?* Fintech interviews probe cost ownership ("establish cloud cost baselines" — slice Staff JD). Cost governance is a Day-1 control, not a Phase-12 afterthought.

---

## EXERCISE 0.2 — Workstation toolchain [B] [~90 min]

**GOAL:** a reproducible dev machine. Works on Linux/macOS/WSL2.

**STEPS**

1. Install and verify (versions are floor, not ceiling):

```bash
   aws --version        # AWS CLI v2
   terraform version    # >= 1.9
   kubectl version --client   # >= 1.30
   helm version         # >= 3.15
   docker --version     # >= 26, with buildx
   kind version         # >= 0.23
   ansible --version    # >= 2.16 (pip install ansible is fine)
   git --version; python3 --version  # >= 3.11
   k9s version          # optional but recommended daily driver
   jq --version; yq --version
```

2. Configure AWS CLI SSO profile:

```bash
   aws configure sso    # start URL from Identity Center, region ap-south-1
   # profile name: np-admin
   aws sso login --profile np-admin
   export AWS_PROFILE=np-admin   # add to shell rc
   aws sts get-caller-identity
```

3. Git identity + safety:

```bash
   git config --global user.name "Your Name"
   git config --global user.email "you@example.com"
   git config --global init.defaultBranch main
   git config --global pull.rebase true
   pip install gitleaks 2>/dev/null || brew install gitleaks || (download from gitleaks releases)
```

4. Create the master repo layout (this structure is referenced by all modules):

```text
   ~/northpay/
     northpay-app/        # the demo application (assets/app copied here Day 1)
     northpay-infra/      # Terraform (starts in Module 07)
     northpay-gitops/     # ArgoCD desired state (Module 10)
     northpay-ansible/    # Module 08
     runbooks/  postmortems/  adrs/  daily-log.md
```

`git init` each; commit an initial `README.md` in each.

5. Copy the provided assets into place:

```bash
   cp -r <downloaded>/assets/app/*        ~/northpay/northpay-app/
   cp -r <downloaded>/assets/seed-data    ~/northpay/seed-data
   cp -r <downloaded>/assets/templates/*  ~/northpay/
   cp -r <downloaded>/assets/scripts      ~/northpay/scripts
```

6. Docker sanity: `docker run --rm hello-world` and `docker compose version`.
7. kind sanity: `kind create cluster --name np-sandbox && kubectl get nodes && kind delete cluster --name np-sandbox`.

**VALIDATE:** every version command succeeds; `kind` cluster creates and deletes cleanly; `gitleaks version` works.

**TRADEOFFS**

- *kind vs minikube vs k3d for local K8s:* kind is the CNCF-conformant, CI-friendly choice (runs in Docker, used by K8s project itself) → default here. k3d is lighter on RAM and great for Istio later; minikube has better driver options on odd setups. You'll use kind first, k3d in Module 16 (Istio) if RAM is tight.
- *Why SSO login in shell rc?* Credentials expire in hours — that *is* just-in-time access; feel the friction now so you appreciate it at work.

---

## EXERCISE 0.3 — GitHub organization + repository plumbing [B] [~45 min]

**STEPS**

1. Create a free GitHub organization `northpay-lab-<you>` (personal account is fine if you prefer; org makes later team/permission exercises realistic).
2. Create private repos: `northpay-app`, `northpay-infra`, `northpay-gitops`, `northpay-ansible`, `northpay-runbooks`. Push your local repos from 0.2.
3. Branch protection on `main` for `northpay-app`: require PR, require 1 approval (you can self-approve for now), require status checks (we'll add checks in Module 09 — note the placeholder).
4. Create a fine-grained PAT for local pushes over HTTPS or set up SSH keys. Store in your OS keychain — never in files.
5. Add `.gitignore` (Terraform, Python, Node, Ansible) and a pre-commit hook running `gitleaks protect --staged`.
6. Enable GitHub's free secret scanning + Dependabot alerts on all repos (org settings → security). These become pipeline gates in Module 09.

**VALIDATE:** push a test branch to `northpay-app`, open a PR, confirm merge is blocked without checks.

**TRADEOFFS**

- *Monorepo vs polyrepo:* you are deliberately running polyrepo (app/infra/gitops separate) because that matches both target companies' implied setups and makes ArgoCD/GitOps boundaries crisp. Interview tradeoff: monorepo simplifies atomic cross-cutting changes and code sharing; polyrepo sharpens ownership, CI scoping, and access control. Know both sides.

---

## EXERCISE 0.4 — On-prem simulation lab (your networking advantage) [I] [~2 hrs]

**GOAL:** use your VMware knowledge to simulate "NorthPay's legacy data center and branch office" — you'll VPN into it from AWS in Phase 10/11 (hybrid connectivity), which most cloud-native candidates can't demo.

**STEPS**

1. In VMware Workstation/ESXi, create 2 VMs (Ubuntu 22.04/24.04, 2 vCPU, 4 GB each):

- `legacy-dc` — will run: strongSwan (IPsec endpoint), a PostgreSQL "legacy DB", an nginx "legacy app".
- `branch-office` — will run: strongSwan + dnsmasq; simulates a branch with its own subnet.

2. Networks: `legacy-dc` on `192.168.100.0/24` (host-only or dedicated vSwitch), `branch-office` on `192.168.200.0/24`. Add a router VM or use your physical router with port-forwarding for IPsec (UDP 500/4500) to `legacy-dc`'s public-facing NIC.
3. Baseline hardening on both: create `ops` user with sudo, disable password SSH (keys only), enable `ufw` with only needed ports, install `fail2ban`. (Do it by hand now; Ansible will redo it in Module 08 — you'll diff the approaches.)
4. On `legacy-dc`: install PostgreSQL, create `northpay_legacy` DB, load `assets/seed-data/legacy_seed.sql`. Install nginx serving a static "Legacy Payroll Portal" page on port 8080.
5. Document the subnets, IPs, and routes in `runbooks/onprem-sim.md`. Diagram it in ASCII. This document is load-bearing in Modules 13–14.

**VALIDATE:** from `branch-office`, `ping 192.168.100.10` (or your DC IP) fails today (no routing yet) — that's *correct*; you'll make it pass via VPN later. SSH to both VMs with keys works.

**TRADEOFFS**

- *Why bother simulating on-prem?* Because "hybrid or private connectivity patterns" (slice SRE-3) and multicloud networking questions are where a network engineer crushes cloud-native-only candidates. Site-to-site VPN, BGP over tunnels, split-horizon DNS — you already know the hard 70%; the labs bolt AWS onto it.

---

## EXERCISE 0.5 — Install the NorthPay app locally (smoke test) [B] [~45 min]

**STEPS**

1. `cd ~/northpay/northpay-app && docker compose up --build` (uses `assets/app/docker-compose.yml`).
2. Endpoints to verify:

- `curl localhost:3000/healthz` → `{"status":"ok"}`
- `curl localhost:3000/readyz` → checks Postgres+Mongo+Redis connectivity
- `curl -X POST localhost:3000/api/orders -H 'Content-Type: application/json' -d '{"customer_id":1,"amount":499.00,"currency":"INR"}'`
- `curl localhost:3000/api/orders` → your order listed
- `curl localhost:3000/metrics` → Prometheus-format metrics (used from Module 11 on)

3. Read the compose file end to end. Identify every service, volume, network, and healthcheck. In your log, write one line per service: what it is, what breaks if it dies.
4. `docker compose down -v` — clean slate. (You will rebuild this app a dozen times this program; treat compose as the dev quick-loop, not production.)

**VALIDATE:** order created via API appears in Postgres (`docker exec` into the db container, `psql -c 'select * from orders;'`).

**TRADEOFFS**

- *Why a payroll-flavored demo app?* Domain realism. When a mock incident says "settlement batch delayed," you must reason about SQS queues and DB locks, not abstract pods. Interviewers ask scenario questions in *their* domain — practice in a similar one.

---

## EXERCISE 0.6 — Templates + logging discipline [B] [~30 min]

**STEPS**

1. Copy templates from `assets/templates/` into your repos: `RUNBOOK.md`, `POSTMORTEM.md`, `SLO.yaml`, `ADR.md`, `CHANGE-REQUEST.md`.
2. Write ADR-0001: "Why polyrepo, why ap-south-1, why EKS over ECS." Three paragraphs max. (Architecture Decision Records are how Staff-level engineers communicate — the slice Staff JD wants exactly this muscle.)
3. Start `daily-log.md` with today's standup entry. Format:

```text
   ## 2026-09-07 (Day 1)
   Done: ...
   Doing: ...
   Blockers: ...
   Cost today: $... | Resources left running: ...
   Toil noticed: ... (one thing I'll automate this week)
```

**VALIDATE:** templates exist in Git; ADR committed.

---

## Phase 0 exit gate

- [ ] Root MFA on; you only use SSO identity; billing alerts armed and tested (send a test email from the budget console).
- [ ] Full toolchain verified; kind cluster up/down in < 2 min.
- [ ] GitHub org with 5 repos, branch protection, secret scanning.
- [ ] On-prem sim VMs running and documented.
- [ ] NorthPay app runs locally; you can explain every compose service.
- [ ] Day 1 log entry + ADR-0001 committed.

**When done:** start `17-Daily-Job-Simulation-90-Days.md` Day 1 *in parallel* with Module 02. The simulation board tells you which module each day's tickets assume.
