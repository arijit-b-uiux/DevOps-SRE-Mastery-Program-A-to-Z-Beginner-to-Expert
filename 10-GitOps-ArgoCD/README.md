# Module 10 — GitOps & ArgoCD (Weeks 12–13)

> **JD coverage:** "Hands-on experience with Kubernetes, including Helm and ArgoCD" (Deel SRE); "GitOps-driven deployments using ArgoCD" (slice SRE-3); "Git as the source of truth... mindful of blast radius" (slice SRE-2); "GitOps workflows for Terraform" (slice Staff — designed in Module 07, here you learn the philosophy it borrows).
> **The mental shift:** CI *pushes* changes (imperative, credentialed pipelines, drift invisible); GitOps *pulls* (an in-cluster agent reconciles Git→cluster continuously). Your cluster credentials never leave the cluster, and drift self-heals. Everything in-cluster at NorthPay becomes ArgoCD-managed this module — including ArgoCD itself.

---

## Part A — ArgoCD core

### EXERCISE 10.1 — Install ArgoCD (the GitOps way, on itself) [B] [60 min]

**STEPS**

1. Install once by hand on kind: `kubectl create ns argocd && kubectl apply -n argocd -f <stable install.yaml>`. Get initial admin password, port-forward, log in (CLI + UI).
2. Immediately bootstrap: create `northpay-gitops` repo layout:

```text
   northpay-gitops/
     bootstrap/root-app.yaml            # the app-of-apps
     argocd/                            # argocd's own config as code
     apps/                              # Application manifests per component
     envs/{dev,staging,prod}/           # kustomize/helm value overlays
     clusters/np-eks-dev/               # cluster-specific config
```

3. Apply `bootstrap/root-app.yaml` — it points at `apps/` which contains an Application for **argocd itself** (self-management). From now on ArgoCD config changes = PRs. Prove it: change `argocd-cm` (e.g., add a banner) in Git → watch it reconcile.
4. RBAC in ArgoCD: read-only account + a `deployer` role; SSO via OIDC (Dex config — wire to GitHub org teams; map `platform-team` → admin, `devs` → read+sync in dev only). This mirrors real access control.
5. Notifications controller: sync-failure → email/SNS webhook.
**VALIDATE:** ArgoCD manages itself (change via Git appears); RBAC denies dev-role sync in prod.
**WHY-WHEN-WHAT:** App-of-apps pattern = one root Application whose resources are more Applications. It's how 200 microservices stay organized: per-team app namespaces, wave ordering, and blast-radius control (freeze one app, not the system).

### EXERCISE 10.2 — Your first Applications: helm, kustomize, plain yaml [B] [75 min]

**STEPS**

1. App CRD anatomy (memorize the fields): `source` (repoURL, path/chart, targetRevision, helm.valuesFiles / kustomize), `destination` (server, namespace), `syncPolicy` (automated? prune? selfHeal?).
2. Deploy the NorthPay stack as three Applications: `web`, `api`, `worker` — all pointing at your Helm chart with `valueFiles: [$values/envs/dev/api-values.yaml]` (the `$values` multi-source pattern — chart from one ref, values from gitops repo).
3. Sync modes hands-on:

- Manual sync first (watch Diff tab; use `argocd app diff/sync`).
- Then `automated: {prune: true, selfHeal: true}` on dev; staging: automated without prune (why? destructive removals deserve eyes); prod: **manual sync gate** or sync windows.

4. Drift self-heal demo: `kubectl scale` the api to 9 manually → ArgoCD marks OutOfSync → selfHeal reverts in seconds. Capture timestamps. Then delete a managed NetworkPolicy → resurrected. **This is your compliance story: unauthorized in-cluster change auto-reverts with an event trail.**
5. Sync options per resource: `argocd.argoproj.io/sync-options: Prune=false` (protect a hand-tuned HPA), `SkipDryRunOnMissingResource` (CRD+CR same app — hit this error once on purpose).
6. Ignore-differences: HPA-managed replica counts vs Git (set `ignoreDifferences` for `/spec/replicas` when HPA owns it — classic fight between GitOps and autoscaler; solve it, don't suppress everything).
**VALIDATE:** three apps Healthy+Synced; drift experiments captured; HPA/ArgoCD conflict resolved by policy, not by disabling selfHeal.

### EXERCISE 10.3 — Sync waves, hooks, and ordered rollouts [I] [75 min]

**STEPS**

1. Sync waves: order a stack — namespaces (-5) → secrets/config (-3) → database migration Job (-1, as PreSync hook) → api/worker (0) → ingress (+1) → smoke-test Job (PostSync). Intentionally break ordering (remove wave annotations) and watch the migration race the api — feel why waves exist.
2. Hooks deep: PreSync migration with `hook-succeeded` delete policy; a failed hook = failed sync (deployment doesn't proceed — correct!). Rollback story: **GitOps rollback = `git revert` + sync**, not `argocd rollback` (history-based rollback exists but bypasses Git — know why it's an anti-pattern; your ADR-0010).
3. Health assessment: built-in health checks per resource type; write a custom health Lua script for your CronJob (healthy = last run succeeded).
4. Sync windows: prod only deployable 09:00–17:00 IST (matches Module 09 governance — now enforced at the CD layer too; defense in depth).
5. Resource pruning safety: `Prune=confirm` on prod via UI; cascade behavior when deleting an Application (finalizer `resources-finalizer.argocd.argoproj.io` — remove it and the app deletes but resources stay; add it and everything prunes — choose per environment consciously).
**VALIDATE:** failed PreSync migration blocks rollout (evidence); `git revert` rollback executed end-to-end; sync window blocks an off-hours sync.

### EXERCISE 10.4 — Multi-environment promotion with GitOps [I] [90 min]

**GOAL:** the daily fintech flow: dev → staging → prod via Git, with review gates.
**STEPS**

1. Model: `envs/dev|staging|prod` directories; image tags live in env values (or kustomize image transformer). Promotion = PR bumping the tag/values in the next env. CI (Module 09) opens promotion PRs automatically after staging soak.
2. Implement: after staging deploy succeeds + 30-min soak + smoke tests green → workflow opens PR `promote api 0.4.2 to prod` with diff summary, test evidence, and change-request link. Human approves → merge → ArgoCD (manual sync gate or window) applies.
3. Release hygiene: prod pins **digests** not tags (tags mutable; digests immutable — supply-chain posture; renovate/dependabot opens digest-bump PRs).
4. Config-only changes: rotating a feature flag in prod = PR to `envs/prod` values; observe how GitOps makes even "tiny config tweaks" reviewable — write the cultural point in your log (this is "approved patterns" from the SRE-2 JD, lived).
5. Emergency path: break-glass manual sync with reason annotation + auto-created post-incident issue. Use it once, deliberately, and document.
**VALIDATE:** full promotion cycle dev→staging→prod with PR evidence; digest pinning live in prod; emergency path exercised and logged.

---

## Part B — Scaling GitOps: multi-cluster, secrets, ApplicationSets

### EXERCISE 10.5 — ApplicationSets & multi-cluster [A] [90 min]

**STEPS**

1. ApplicationSet generators: `list` (deploy monitoring-agent to 3 clusters), `git` directories (every dir under `teams/*` becomes an app — the self-service namespace onboarding pattern), `cluster` generator with labels as values.
2. Register a second cluster: kind #2 as `staging` while EKS is `prod` (or two EKS if budget allows a day). `argocd cluster add` — understand the secret it creates (this is the credential you must protect; ArgoCD is a cluster-admin robot).
3. Progressive delivery across clusters: ApplicationSet rolling out canary-tagged releases to `env=dev` clusters first (generator with goTemplate + postSelection). Even simplified, you've built the fleet-rollout skeleton used at scale.
4. Disaster thinking: ArgoCD itself dies on the prod cluster — recovery = reinstall + root-app apply from Git (drill it: uninstall, reinstall, full recovery < 15 min, timed). Add to DR runbooks (Module 15).
**VALIDATE:** one change → two clusters converge; ArgoCD self-recovery drill timed and documented.

### EXERCISE 10.6 — Secrets in GitOps (the hard problem) [I] [75 min]

**STEPS**

1. Survey (write the matrix): Sealed Secrets (kubeseal — encrypted-in-Git, cluster-scoped keys, rotation pain) vs sops+helm-secrets (Module 06 — good for values) vs **External Secrets Operator** (sync from AWS SM — NorthPay's choice; rotation in SM flows to pods, Git never holds secrets at all) vs Vault (dynamic secrets, more platform).
2. Implement ESO on kind + EKS: ClusterSecretStore (IRSA role), ExternalSecret mapping `northpay/prod/api` SM keys → K8s Secret refreshed every 1h; verify rotation propagates (update SM value → wait → pod restart via reloader → new value).
3. Git hygiene audit: `gitleaks detect` across gitops repo history; policy: no Secret manifests with data, only ExternalSecrets; pre-commit + CI gate.
4. Tradeoff paragraph: with ESO, AWS SM is source of truth for *secrets*, Git for *everything else* — two sources, each authoritative for its domain. Interviewers probe this nuance.
**VALIDATE:** rotation demo captured; zero secrets in Git (scanner proof).

### EXERCISE 10.7 — Argo Rollouts: progressive delivery as GitOps [A] [2 hrs]

*(Completes Exercise 9.9's canary story with automation)*
**STEPS**

1. Install Argo Rollouts + kubectl plugin. Convert api Deployment → Rollout CR (managed by Helm chart value — keep packaging consistent).
2. Canary strategy: steps `setWeight: 10 → 30 → 60 → 100` with pauses; manual promotion gate between staging and full (`kubectl argo rollouts promote`).
3. **Analysis with metrics:** AnalysisTemplate querying Prometheus (Module 11 dependency — use a simple success-rate query; if Module 11 isn't done, use a trivial always-pass query now and harden later): auto-rollback when error rate > 2% for 2 min. Inject failure (deploy the broken image) → watch automatic abort + rollback. **This auto-rollback-on-SLO-burn is your flagship demo artifact.**
4. BlueGreen Rollout variant: preview service URL for QA before cutover.
5. Header-based canary (needs ingress/mesh support): route `x-beta: true` users to canary — full version in Module 16 (Istio).
6. Dashboard: rollouts plugin in ArgoCD UI; rollout metrics into Grafana.
**VALIDATE:** injected bad release auto-rolls-back with timeline captured; good release promotes through gates; you can narrate the whole flow on a whiteboard.

### EXERCISE 10.8 — ArgoCD operations at production grade [A] [60 min]

**STEPS**

1. HA install (2+ replicas, redis HA, ApplicationSet sharding concept) — design doc if not on EKS.
2. Backup: `argocd-backup` or velero; what's actually precious? (Answer: app definitions are in Git — backup is mostly cluster credentials + UI settings; stateless-by-design = GitOps dividend.)
3. Scale/perf knobs: repo-server concurrency, status processors; webhook from GitHub (instant reconcile vs 3-min poll — wire it).
4. Observability of ArgoCD: its own /metrics → your Prometheus; alert on `app OutOfSync > 30m`, `sync failures`.
5. Security posture review: disable admin user post-SSO, audit log locations, projects (AppProject) as tenancy boundaries: restrict source repos + destination namespaces per team. Create `team-payments` project limited to their repo+ns; prove a rogue Application pointing elsewhere is rejected.
**VALIDATE:** AppProject boundary test fails correctly; webhook-driven sync < 10s; ArgoCD metrics on your dashboard.

---

## Module 10 exit gate

- [ ] Everything in-cluster (apps, ingress, monitoring, cert-manager, even ArgoCD) is ArgoCD-managed from `northpay-gitops`; nothing applied by hand (policy: hand-applies get reverted by selfHeal — prove it to yourself).
- [ ] Promotion flow with PR gates + digest pinning; emergency path documented.
- [ ] Auto-rollback canary demo recorded (your artifact #1 from README §7).
- [ ] App-of-apps + ApplicationSets + AppProjects = multi-team, multi-cluster story you can draw.

**Onward:** `11-Observability.md` — you can't SRE what you can't see.
