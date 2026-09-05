# Module 09 — CI/CD Pipelines (Weeks 10–11)

> **JD coverage:** "Experience with CI/CD tools (Drone, Jenkins, GitHub actions or similar)" (Deel); "Spinnaker/Jenkins/ArgoCD" (slice Staff/Lead); "test automation frameworks, dynamic test selection, code quality gates" (slice Staff/Lead); "Design and build internal tools/platforms that optimize CI/CD" (slice Lead); "GitOps-driven deployments using ArgoCD and CI/CD pipelines (GitHub Actions or equivalent)" (slice SRE-3).
> Pipeline definitions: `assets/ci/`. Principle: **CI produces immutable artifacts; CD deploys them; GitOps (Module 10) makes Git the trigger.** Mixing those responsibilities is the #1 pipeline design smell.

---

## Part A — CI fundamentals with GitHub Actions

### EXERCISE 9.1 — Actions mechanics deep [B] [75 min]

**STEPS**

1. First workflow for `northpay-app`: on push → lint (eslint), unit tests, build. Understand: workflows/jobs/steps, runner model (ubuntu-latest), matrix builds (node 20/22), caching (npm cache via actions/cache — measure build time before/after), artifacts vs caches.
2. Debugging: `ACTIONS_STEP_DEBUG`, `act` (local Actions runner) for fast iteration — run your workflow locally.
3. Triggers mastered: push paths-filter, pull_request types, workflow_dispatch inputs, schedule (nightly), workflow_call (reusable workflows), concurrency groups (cancel stale PR runs — cost + noise).
4. Expressions & contexts: `github.*`, `env`, `secrets`, `needs`, `outputs` between jobs; conditional jobs (`if: failure()` → notification job).
5. Security basics NOW, not later: pin actions to SHA (`actions/checkout@<sha> # v4.2.2`), minimal `permissions:` block (default is too broad — prove it by reading a token's scope), no `pull_request_target` with checkout of untrusted code (the classic pwn-request; write the note).
**VALIDATE:** matrix CI green; cache hit visible in logs; SHA-pinned actions throughout.

### EXERCISE 9.2 — The production CI pipeline for northpay-app [I] [2 hrs]

**STEPS** (workflow `ci.yml` — full file in assets/ci/github/; build step by step)

1. **Quality gate stage:** eslint + `npm test` (coverage threshold 70% — gate on it), `shellcheck` for scripts, `gitleaks` scan, `npm audit --audit-level=high` (triage: fail on fixable highs).
2. **Build stage:** docker build with BuildKit cache-to/from GitHub cache; tag = git SHA; trivy scan gate (CRITICAL fixable = fail, with `.trivyignore` governance from 4.5); SBOM artifact uploaded.
3. **Publish stage (main only):** push to ECR. **No long-lived AWS keys:** configure IAM OIDC provider for `token.actions.githubusercontent.com` + role `np-gha-ecr-push` with trust condition on repo+ref (`sub: repo:northpay-lab/northpay-app:ref:refs/heads/main`). `aws-actions/configure-aws-credentials` with role-to-assume. Verify: zero AWS secrets in repo settings.
4. **Sign + attest:** cosign keyless sign the image; `attest-build-provenance` (SLSA provenance). Verify signature in a later job (and Module 13 enforces in admission).
5. **Change metadata:** conventional-commits → auto changelog + version bump (semantic-release or release-please); git tag = image tag = chart appVersion. Traceability rule: every running container answers "which commit built you."
6. **Notifications:** failure → SNS/email (later: Slack) with run URL.
**VALIDATE:** push to main → SHA image in ECR, signed, SBOM attached, no stored AWS creds; a planted CRITICAL CVE fails the build; coverage drop below 70% fails.

### EXERCISE 9.3 — Monorepo-grade CI techniques & dynamic test selection [A] [90 min]

*(slice Staff/Lead explicitly test this)*
**STEPS**

1. Path-based job triggering: `dorny/paths-filter` — api changes don't run worker tests.
2. **Dynamic test selection demo:** in a scratch repo with 200 generated trivial tests, implement changed-file→test mapping (pytest --collect-only + import graph or `pytest-testmon`); measure CI time saved; articulate the risk (missed transitive impact) and mitigations (nightly full runs, coverage-based selection).
3. Flaky test management: rerun-failed (3x) with quarantine list; flaky-test detection job comparing pass/fail across reruns; policy: flaky = P1 bug (flaky tests erode trust → people ignore red CI → deploys break).
4. Self-hosted runner: run a runner on `legacy-dc` VM (docker) — when needed (VPC-internal deps, big RAM builds, cost at scale) + the security model (ephemeral runners, no public-repo forks on self-hosted — EVER; write why).
5. Build metrics: export workflow timing to a dashboard (simple: jq the API into CSV → later into your Grafana). You can't optimize what you don't measure (Lead JD: "developer velocity").
**VALIDATE:** test-selection saves >50% time with documented risk controls; self-hosted runner executes a job.

### EXERCISE 9.4 — Ephemeral (PR preview) environments [A] [2 hrs] *(named in both slice JDs)*

**GOAL:** every PR to northpay-app gets a live, isolated `pr-<n>.northpay.local` environment on the kind/staging cluster; closed PR = environment destroyed.
**STEPS**

1. PR workflow: build image tagged `pr-<n>-<sha>`; `helm upgrade --install northpay-pr<n> chart/ -f values-dev.yaml --set image.tag=... -n pr-<n> --create-namespace` with per-PR ingress host + an **ephemeral Postgres** (in-cluster postgres:16 container seeded from `assets/seed-data/postgres_seed.sql` — small dataset for speed).
2. Comment bot: workflow posts the env URL as a PR comment.
3. TTL reaper: cron workflow daily → helm list all `pr-*` namespaces older than 48h or with closed PRs → uninstall + delete ns (namespace deletion hangs? finalizers — debug once, write runbook).
4. Cost/guardrails: ResourceQuota per PR namespace (from 5.8), NetworkPolicy isolation between PR envs.
5. Product thinking: who uses previews (devs, QA, PMs); when previews beat staging; data strategy (never prod data in previews — synthetic only; compliance!).
**VALIDATE:** open PR → env live → comment posted → close PR → namespace gone within 15 min. This exact demo is a Staff-level interview answer ("how do you accelerate developer velocity with platforms").

---

## Part B — Jenkins (self-hosted, the enterprise reality)

### EXERCISE 9.5 — Jenkins controller setup done right [I] [90 min]

**STEPS**

1. Run Jenkins via `assets/ci/jenkins/docker-compose.yml` (jenkins:lts + agent container). Unlock, plugins: **use Configuration-as-Code (JCasC)** — write `casc.yaml` defining security (own user, no anonymous), credentials, and seed job. Never click-config what you can code (reload JCasC → reproducible controller).
2. Agent model: inbound agent container with docker-in-docker (or docker socket mount — note the security tradeoff, rootless/podman alternative); static agents vs Kubernetes plugin (dynamic pod agents on your kind cluster — set it up: jenkins agent pods spinning per build).
3. Folders + RBAC (role-based strategy plugin via JCasC): devs see their folder only.
4. Backup: `$JENKINS_HOME` volume snapshot strategy; thinBackup or plain tar — test a restore (nobody believes backups until restore works — say this in interviews).
**VALIDATE:** destroying and recreating the controller from compose+JCasC loses nothing except build history.

### EXERCISE 9.6 — Jenkinsfile pipelines for northpay-app [I] [2 hrs]

**STEPS**

1. Multibranch pipeline from GitHub org scan (webhooks via smee.io or polling since local).
2. Declarative `Jenkinsfile` (assets/ci/jenkins/Jenkinsfile): stages = lint → test → build image (kaniko or dind) → trivy gate → push ECR (aws creds from Jenkins credential store; note OIDC options) → input step for staging deploy (approval gate!) → deploy via helm.
3. Shared libraries: extract `buildAndScan()` into `vars/` in a `jenkins-shared-lib` repo — versioning the library like code (this is how enterprises standardize 500 pipelines; the Lead JD's "frameworks" line).
4. Parallel stages with `failFast`; `post { unsuccessful }` notifications; build discarding/log rotation policy (controller disk fills — the classic Jenkins self-inflicted outage; set it now, plus an alert).
5. Credentials hygiene: string credentials masked in logs (prove masking works; then show a bypass via base64 — and the mitigation: least-scope creds + audit).
6. Blue Ocean or classic — know both exist; pipeline-stage-view for demos.
**VALIDATE:** full Jenkinsfile run green incl. trivy gate; shared library consumed; approval gate pauses staging deploy until clicked.

### EXERCISE 9.7 — Jenkins ops: upgrades, plugins, security [A] [60 min]

**STEPS**

1. Plugin vulnerability day: check update center warnings; upgrade a plugin; restart safely (`/safeRestart` with running builds — what happens?).
2. LTS upgrade: backup → new container image → verify JCasC + jobs → rollback plan (previous image tag + home snapshot).
3. Agent compromise scenario (tabletop + write-up): agent has docker socket → root on agent → creds in workspace env → exfil path. Controls: ephemeral agents, least-privilege creds, network segmentation of build agents (they're production-adjacent — treat them so).
4. When Jenkins vs Actions (ADR-0009): Jenkins = control/customization/heavy enterprise integration, you own ops burden; Actions = zero-ops, GitHub-native, pricing at scale, less flexible runner internals. NorthPay: Actions for app CI, Jenkins kept as the "enterprise integration" skill + DR path. **Spinnaker next.**
**VALIDATE:** LTS upgrade executed with rollback plan written; threat model one-pager committed.

### EXERCISE 9.8 — Spinnaker: concepts + minimal deployment [A] [2 hrs theory + optional install]

*(JDs say "Spinnaker/Jenkins/ArgoCD" — you must speak Spinnaker fluently even if ArgoCD is your daily tool.)*
**STEPS**

1. Study + write the mental model: Spinnaker = CD platform (not CI): applications/clusters/server groups; pipelines with stages (bake, deploy manifest, manual judgment, canary); integrations with cloud providers; Deck/Gate/Clouddriver/Orca/Igor/Echo/Rosco (microservice architecture — diagram it once).
2. Key differentiators vs ArgoCD (write the table): Spinnaker owns deployment *orchestration* (multi-cloud bake/deploy/canary, UI-driven pipelines), imperative pipelines stored in its DB; ArgoCD is GitOps *reconciliation* (declarative desired state from Git). Modern verdict: ArgoCD + CI for most; Spinnaker survives in multi-cloud VM/bake-heavy estates (Netflix heritage).
3. Optional install (RAM-heavy): `hal`/Halyard is legacy — try the Spinnaker Operator on kind with minio as artifact store; deploy one pipeline: manual judgment → deploy manifest to kind ns. If resources are tight, do a guided video + write the design doc instead (log the tradeoff honestly).
4. Automated canary analysis (Kayenta + Prometheus/Datadog) — read docs; you implement the *concept* yourself in Module 16 with Istio+Argo Rollouts, which is the Kubernetes-native successor. Write that lineage in your log.
**VALIDATE:** comparison table + architecture diagram in your notes; (optional) one pipeline executed.

---

## Part C — CD design & deployment strategies

### EXERCISE 9.9 — Deployment strategy taxonomy, hands-on [A] [2 hrs]

**STEPS** (implement each against the api on kind; measure user-visible errors with `np-deploy-check` running continuously)

1. **Recreate:** downtime measured (feel why never for prod).
2. **Rolling (native k8s):** tune maxSurge/maxUnavailable; note zero downtime but mixed versions serve simultaneously — implication: **backward-compatible changes only** (DB schema changes across rolling deploys = the expand/migrate/contract pattern; write it out, Module 12 drills it).
3. **Blue/green with Services:** two deployments (`api-blue`, `api-green`), svc selector flips; instant switch + instant rollback (flip back); cost: 2× capacity during cutover.
4. **Canary by hand:** 1 canary pod among 4 stable (20% traffic); watch error rates; abort = scale canary to 0. Then Argo Rollouts (install): `Rollout` CR with `setWeight` steps + analysis using Prometheus metrics (Module 11 supplies them; revisit this exercise after Module 11 for full automation — note the dependency).
5. **Feature flags vs deployments:** add a flagsmith-style or env-var flag to the api for "new settlement engine"; deploy dark, enable per-customer. When flags beat canaries (long-lived experiments, per-user targeting) and their cost (flag debt — scheduled flag-removal tickets).
6. Write the decision tree: when rolling / blue-green / canary / flags — with rollback time, capacity cost, DB-compat constraint columns. **This tree is a guaranteed interview artifact.**
**VALIDATE:** measured error rates per strategy in a table; decision tree committed.

### EXERCISE 9.10 — Pipeline governance & the "safe change" system [A] [60 min]

*(slice SRE-2: "changes stay reviewable, repeatable, and safe"; change windows)*
**STEPS**

1. Environments in GitHub: `staging` (auto) and `prod` (required reviewers + wait timer) protection rules; deployment branch policy.
2. Change windows as code: workflow condition — prod deploys only Mon–Thu 09:00–17:00 IST unless label `emergency-change` + extra approval (document the emergency process).
3. Change-request integration: workflow opens a GitHub Issue from your `CHANGE-REQUEST.md` template with plan diff summary, linked to the deploy run — audit trail for free.
4. Audit query practice: "who deployed what to prod last month" — answer via GitHub deployments API + ArgoCD history (Module 10) + CloudTrail. Build the one-liner script now (`npctl deploys --since 30d`).
5. Blast-radius controls: per-service deploy isolation, one-env-at-a-time progression (dev→staging→prod with bake times), auto-rollback triggers (SLO burn — full wiring in Modules 11/16).
**VALIDATE:** out-of-window deploy attempt blocked; emergency path used once and documented; audit one-liner works.

---

## Module 09 exit gate

- [ ] northpay-app CI: quality gates → signed image in ECR → SLSA attestation; zero stored cloud creds (OIDC).
- [ ] PR preview environments live and self-cleaning.
- [ ] Jenkins rebuilt from code (JCasC), shared-library pipeline green, threat model written.
- [ ] Deployment-strategy decision tree with measured evidence; change-window governance enforced by the pipeline itself.
- [ ] Spinnaker vs ArgoCD vs Jenkins narrative rehearsed out loud (yes, actually say it).

**Onward:** `10-GitOps-ArgoCD.md` — Git becomes the deploy button.
