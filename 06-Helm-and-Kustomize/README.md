# Module 06 — Helm & Kustomize: Packaging for Kubernetes (Week 12; also used from Module 05 onward)

> **JD coverage:** "Hand-on experience with Kubernetes, including Helm and ArgoCD" (Deel SRE). Helm is how you'll ship the NorthPay app to every environment; Kustomize is the no-templating alternative you must be able to argue for/against. Starter chart: `assets/helm/northpay-chart/`.

---

## Part A — Helm

### EXERCISE 6.1 — Helm as a consumer first [B] [30 min]

**STEPS**

1. Add repos: bitnami, ingress-nginx, prometheus-community, jetstack, grafana. `helm search repo postgres --versions`.
2. Install bitnami/redis to kind with `--set` overrides; inspect what got created (`helm status`, `helm get manifest`); upgrade with new values; rollback (`helm rollback`); uninstall and notice what Helm leaves behind (PVCs! — `helm uninstall` doesn't delete PVCs created by StatefulSets; verify, clean, document).
3. `helm show values bitnami/redis` — reading values schemas before installing anything is a professional habit; find 3 dangerous defaults (persistence size, password handling, architecture).
4. Version pinning: `--version` on install; why `helm upgrade` without pinning in a script is an incident generator.
**VALIDATE:** install→upgrade→rollback→uninstall cycle with evidence of leftover PVCs.

### EXERCISE 6.2 — Author the northpay-chart [I] [2 hrs]

**STEPS** (skeleton provided; you extend it)

1. Structure review: `Chart.yaml` (apiVersion v2, appVersion vs version — know the difference), `values.yaml`, `templates/`, `templates/tests/`, `NOTES.txt`.
2. Templates to author/fix: Deployment + Service for api, worker, web; Ingress; HPA; PDB; ServiceAccount (IRSA-annotation via values); NetworkPolicy; ConfigMap from values.
3. Templating mastery: `{{ .Values }}`, `with`, `range` over a list of env entries, `include`/`define` for labels (the chart has `_helpers.tpl` — study `chart.labels` and use everywhere), `tpl` for values-with-templates, `required` for must-set values, `quote`/`nindent` correctness.
4. Values design for 3 environments: `values.yaml` (defaults) + `values-dev.yaml`, `values-staging.yaml`, `values-prod.yaml` (replicas, resources, ingress hosts, image tags, IRSA role ARN). **Rule: env files only override; never fork templates.**
5. Schema validation: `values.schema.json` — enforce `image.tag` required, `resources` present. Test with a bad values file → clean error (this is the "guardrails" the slice SRE-3 JD wants in IaC/packaging).
6. Hooks + tests: pre-install hook running DB migration Job (from 5.15) — hook weights, delete policies (`before-hook-creation`); a helm test pod hitting /healthz (`helm test`).
7. Checksum-reload: `checksum/config` annotation on the Deployment so ConfigMap changes trigger rollouts (solves 5.6's stale-config problem properly).
8. `helm lint`, `helm template` (diff against what you expect), `helm install --dry-run --debug`, and **chart-testing (`ct lint`)** locally.
**VALIDATE:** `helm upgrade --install northpay ./northpay-chart -f values-dev.yaml` brings the whole stack up on kind; a deliberate values error fails schema validation; config change rolls pods automatically.

### EXERCISE 6.3 — Helm lifecycle operations & failure modes [I] [60 min]

**STEPS**

1. Release secrets: `kubectl get secret -l owner=helm` — Helm v3 stores state in cluster secrets; understand `helm history` from them.
2. Failed upgrade recovery: introduce a template error mid-upgrade; release stuck `pending-upgrade`; recovery options: `helm rollback`, `helm history`, last resort deleting the release secret (document why that's dangerous).
3. `--atomic` + `--wait` + `--timeout`: the deploy-or-rollback combo your CI uses (Module 09 wires it); test with a broken image tag.
4. Drift: manually `kubectl edit` a Helm-managed Deployment; `helm diff` plugin shows drift; ArgoCD will *enforce* against this (Module 10) — note the progression: Helm releases → Helm+CI → GitOps.
5. OCI registries: push chart to ECR as OCI artifact (`helm push oci://...`); charts-as-artifacts next to images.
6. Library charts + subcharts: extract common bits into a library chart; dependency management (`Chart.yaml dependencies`, `helm dependency update`).
**VALIDATE:** stuck-release recovery documented; atomic rollback demonstrated; chart published to ECR.

### EXERCISE 6.4 — Helm security & supply chain [I] [30 min]

**STEPS**

1. Provenance: `helm package --sign` with GPG, `--verify` on install; or cosign on OCI artifacts.
2. Never commit decrypted values: `helm-secrets` plugin + sops (age key) for `secrets.yaml` in Git — encrypt/decrypt round trip. (This is the GitOps-safe secret pattern Module 10 adopts.)
3. Third-party chart vetting checklist (write it): maintained? values schema? securityContext defaults? image sources? You will quote this checklist in "how do you vet dependencies" interview answers.
**VALIDATE:** sops-encrypted values file committed; decrypted only at deploy time.

---

## Part B — Kustomize

### EXERCISE 6.5 — Kustomize: overlays without templating [I] [75 min]

**STEPS**

1. Base: plain manifests for the api (deployment, service, configmap). `kustomize build` (and `kubectl apply -k`).
2. Overlays dev/staging/prod: namePrefix, commonLabels, replica patches, image tag patches, configMapGenerator with behavior: merge.
3. Strategic merge patches vs JSON6902 patches — do one change each way (resources via SMP; add nodeSelector via JSON6902); when each is readable.
4. `images:` transformer (tag updates without patches) — CI-friendly.
5. Components: a `monitoring` component (adds prometheus annotations + service monitor) applied to some envs.
6. Head-to-head write-up (ADR-0003): **Helm vs Kustomize** — Helm: packaging/distribution, hooks, rich templating (and its footguns: template complexity, `helm template` vs install drift); Kustomize: declarative patches, built into kubectl/ArgoCD, no release state, weaker parameterization. NorthPay: Helm for the app chart (env values files), Kustomize for cluster-addons baseline. Many shops run exactly this hybrid.
**VALIDATE:** same base renders 3 valid envs; ADR-0003 committed.

---

## Module 06 exit gate

- [ ] northpay-chart v1.0.0: schema-validated, 3 env value files, hooks/tests, checksum-reload, signed, in ECR.
- [ ] Kustomize overlay tree for one component; ADR on Helm vs Kustomize.
- [ ] You can recover a stuck release and explain Helm's in-cluster state.

**Onward:** `07-Terraform-IaC.md` — rebuild *everything* from Modules 03–05 as code.
