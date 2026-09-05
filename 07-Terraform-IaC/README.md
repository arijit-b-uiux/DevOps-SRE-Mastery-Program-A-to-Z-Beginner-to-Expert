# Module 07 — Terraform & IaC (Weeks 7–8)

> **JD coverage:** "Strong expertise in Terraform, Kubernetes, AWS, CI/CD" (slice Staff); "reusable modules, guardrails, and a clean Terraform SDLC" (slice SRE-3); "writing reusable, scalable modules, with a good feel for plan/apply, state, and code review" (slice SRE-2); "GitOps workflows for Terraform" (slice Staff). By Friday of Week 8, **every AWS resource from Modules 03–05 exists only as code**. If it's not in Terraform, it gets deleted — that's the discipline that makes IaC real.
> Starter code: `assets/terraform/`.

---

## Part A — Language & state fundamentals

### EXERCISE 7.1 — HCL mechanics on throwaway resources [B] [75 min]

**STEPS**

1. First resource by hand: `aws_s3_bucket` in a single `main.tf`; `init/plan/apply/destroy` cycle; read the plan output line by line.
2. Variables/locals/outputs: parameterize bucket name + tags; `terraform.tfvars`, env vars (`TF_VAR_`), precedence order (write it out — interview question).
3. Types & functions lab: `map(object({...}))` variable for 3 buckets; `for_each` over it; `for` expressions; `lookup`, `merge`, `coalesce`, `try`, `cidrsubnet` (you'll love this one — compute all Module 03 subnet CIDRs from a VPC cidr with `cidrsubnet`, never hardcode again); `count` vs `for_each` (why for_each survives reordering — prove it by removing the middle element with each).
4. Data sources: `aws_availability_zones`, `aws_ami` (latest AL2023), `aws_caller_identity` in names. Data vs resource vs import — the three ways Terraform learns about the world.
5. `terraform console` for expression debugging; `terraform fmt/validate`; `tflint` install.
6. Depends_on: implicit (references) vs explicit; build a VPC→IGW→route-table chain implicitly, then find a case needing explicit (route table association ordering) and feel why explicit is a smell.
**VALIDATE:** 3 buckets from one for_each map with computed names/tags; plan shows zero surprises on re-apply (idempotence).

### EXERCISE 7.2 — State: the heart of Terraform [B→I] [90 min]

**STEPS**

1. Local state anatomy: open `terraform.tfstate` (it's JSON — read it), `terraform state list/show/rm/mv`. Rename a resource in code → `state mv` instead of destroy/create (feel state-as-identity).
2. Remote backend: bootstrap problem (state bucket can't manage itself) → `assets/terraform/bootstrap/`: S3 bucket (versioning, encryption, block public) + **S3-native locking** (`use_lockfile = true` — modern; DynamoDB locking is legacy, know it for interviews). Migrate local → remote with `terraform init -migrate-state`.
3. State corruption drill: hand-edit a resource in console (drift), `plan` shows it; `apply` reverts it. Then simulate a state/ reality mismatch (delete bucket in console): `plan` wants to create; `terraform refresh`-only behavior; recovery options (`import`, `state rm` + import).
4. Import lab: import your Module 03 hand-made VPC (write minimal matching config, `terraform import aws_vpc.this vpc-xxx`, plan → fix diffs until clean). **Importing legacy infra into TF is a real job task — this drill is gold.**
5. `terraform_remote_state` data source: root `network` state exposes VPC/subnet IDs; `app` state consumes them. Loose coupling via outputs = the multi-team pattern.
6. State security: it contains secrets (RDS passwords!) — encryption at rest, bucket policy restricting to your role, `sensitive = true` on outputs/variables (still lands in state — know it), no state in Git ever.
7. `moved` blocks (refactor without state mv), `removed` blocks (drop from management without destroying).
**VALIDATE:** legacy VPC fully under TF with clean plan; broken-state recovery documented in runbook `terraform-state-surgery.md`.

### EXERCISE 7.3 — Modules: authoring reusable infrastructure [I] [2 hrs]

**STEPS**

1. Convert your VPC code into `modules/vpc`: inputs (cidr, az_count, name), outputs (vpc_id, subnet_ids maps, route tables), sane defaults, `cidrsubnet` math inside.
2. Registry-style hygiene: README with usage, `variables.tf` validation blocks (`validation { condition = ... }` — guardrails!), semantic version tags in Git.
3. Consume it: `module "vpc_prod"` and `module "vpc_nonprod"` from the same source with different inputs — two VPCs, one module. This *is* the multi-account/multi-env pattern.
4. Compose a `modules/eks` wrapper (over terraform-aws-modules/eks — using community modules is professional, not cheating; vetting checklist from Module 06 applies) and `modules/rds-postgres`.
5. Module versioning in consumption: `source = "git::https://...//modules/vpc?ref=v1.2.0"` — pin, upgrade via PR, never `main`.
6. Anti-patterns to feel once: module that takes 40 variables (just use resources); module per resource (noise); provider blocks inside child modules (legacy — pass providers down).
**VALIDATE:** two environments from same modules with clean plans; module tagged v1.0.0 and consumed by ref.

---

## Part B — Terraform SDLC (the SRE-3 phrase, made concrete)

### EXERCISE 7.4 — Code quality gates for Terraform [I] [60 min]

**STEPS**

1. Pre-commit hooks: fmt, validate, tflint, `terraform-docs` (auto module READMEs), gitleaks.
2. Security scanning: `checkov` and `tfsec`/`trivy config` on your modules; triage findings (real vs accepted-risk with inline suppress + comment + ADR reference); fail pipeline on HIGH.
3. Policy-as-code preview: OPA/Rego or Sentinel concepts — write one Rego rule ("all resources must have env tag") and run with `conftest` against plan JSON (`terraform show -json`). Module 13 scales this up.
4. Cost estimation: `infracost breakdown` on the repo; budget gate in CI (Module 09).
**VALIDATE:** all four gates run locally in one `make tf-check`; a deliberately untagged resource fails the Rego gate.

### EXERCISE 7.5 — The PR workflow: plan in CI, apply on merge [I] [90 min] (completes in Module 09)

**STEPS**

1. Design first (write it): PR opened → CI runs fmt/validate/tflint/checkov/plan → plan output posted as PR comment → human review → merge → CD applies. `apply` never runs on unreviewed code.
2. Implement locally as scripts now (CI wiring in Module 09): `scripts/tf-plan.sh <dir>` producing planfile + human diff; `scripts/tf-apply.sh` applying only with `-auto-approve` off unless CI.
3. Drift detection: nightly `terraform plan -detailed-exitcode` job → exit 2 = drift → ticket yourself. Implement as a cron on your workstation now, GitHub Action schedule in Module 09. **Drift detection is a top-10 SRE interview answer.**
4. Locking contention: two plans, one applies mid-way → lock error; read the lock info; `force-unlock` only with the lock ID and after confirming the holder is dead — runbook entry.
**VALIDATE:** drift deliberately introduced is caught by the nightly job with a readable report.

### EXERCISE 7.6 — Workspaces vs directory-per-env [I] [45 min]

**STEPS**

1. Try workspaces: `terraform workspace new staging`; same code, two states. Feel the pain: one codebase must serve divergent envs via conditionals; invisible coupling; same backend ACLs for all envs (prod state readable from dev context!).
2. Adopt directory-per-env (`environments/dev`, `environments/prod` calling shared modules, separate state keys, eventually separate AWS accounts — Module 13). 
3. ADR-0004: workspaces vs directories — the industry settled on directories/terragrunt-style for serious multi-account; workspaces survive for ephemeral/env-parallel testing. (Know `terragrunt` exists as the DRY-wrapper option; note when it earns its complexity.)
**VALIDATE:** ADR committed; dev+prod live from separate states.

---

## Part C — Rebuild NorthPay as code (the capstone of this module)

### EXERCISE 7.7 — VPC + networking stack in Terraform [I] [3 hrs]

**STEPS**

1. `environments/prod/network/`: module calls for VPC (3 AZ), subnets via cidrsubnet math, IGW, NAT per AZ (variable `nat_ha = true`), route tables, flow logs → CW, S3 gateway endpoint + SSM interface endpoints, NACLs for data subnets.
2. Outputs consumed by later stacks via remote state.
3. Import-or-rebuild decision: your console VPC is running the app. Safer path: build `np-vpc2` in code, migrate app, destroy console VPC. Document why in-place import of a *running* prod VPC is riskier than build-migrate-destroy (blue/green for infrastructure!).
4. Validation gate: plan reviewed like a PR even though you're solo — write the self-review checklist (blast radius? tag propagation? CIDR overlaps with on-prem sim + future clouds? cost delta via infracost?).
**VALIDATE:** `terraform destroy` + `apply` reproduces the entire network in ~4 min; connectivity matrix re-verified by scripts.

### EXERCISE 7.8 — Compute + data stacks in Terraform [I] [3 hrs]

**STEPS**

1. `environments/prod/compute/`: ALB, TG, launch template, ASG (3 AZ), ACM cert (DNS validation via route53 record resources — fully automated), Route53 alias + weighted records, CloudWatch alarms + SNS topic, S3 buckets with lifecycle from Module 03.
2. `environments/prod/data/`: RDS Postgres (parameter group, SG, secrets manager password via `random_password` + rotation), SQS queues + DLQs + redrive, SNS topics, ECR repos with lifecycle policies, ElastiCache redis (small).
3. Secrets flow: TF writes RDS password into Secrets Manager; app reads from SM at boot — no password in any tfvar. Prove: `grep -r` your repo finds zero passwords.
4. RDS gotchas to hit once deliberately: changing instance class without `apply_immediately` (waits for window), final-snapshot behavior on destroy (`skip_final_snapshot=false` in prod, true in dev), deletion protection blocks destroy (good!).
**VALIDATE:** `terraform apply` from empty account → working https app end-to-end. Then targeted destroy/apply cycles for one stack only (state isolation pays off).

### EXERCISE 7.9 — EKS in Terraform [A] [3 hrs] [COST window — plan a same-day build/teardown first pass]

**STEPS**

1. `modules/eks` wrapping the community module: cluster (private endpoint + limited public CIDRs), OIDC provider, managed nodegroups via `eks_managed_node_groups` (and note Karpenter-as-TF alternative), aws-auth via `aws-auth` module or access entries API, addons with versions pinned, cluster encryption with KMS (secrets at rest — ties to 5.6).
2. IRSA roles in TF: `iam-role-for-service-accounts` pattern — external-dns, aws-load-balancer-controller, cert-manager, worker-sqs. Map SA names as variables (chart values must match — the TF↔Helm contract; get it wrong once and debug the 403 → write runbook).
3. Post-cluster: `terraform_data`/null_resource vs the `kubernetes`/`helm` providers debate — installing controllers via TF helm provider (fast, but plan-time cluster dependency pain) vs GitOps (Module 10 installs everything via ArgoCD). Decide in ADR-0005: **TF creates the cluster + IAM; ArgoCD installs/manages in-cluster software.** (This division is the modern standard.)
4. kubectl_config via `aws eks update-kubeconfig` output; smoke test.
**VALIDATE:** cluster up from `apply`; `kubectl get nodes`; IRSA for worker verified; full destroy works (watch ENI/ALB leftovers — orphan cleanup runbook: SG dependencies block VPC destroy; find with `aws ec2 describe-network-interfaces --filters vpc-id`).

### EXERCISE 7.10 — Terraform + Ansible boundary [B] [30 min]

**STEPS**

1. Rule-writing exercise (ADR-0006): **Terraform = provision & cloud APIs; Ansible = OS/instance configuration & app deployment onto pets.** Overlap exists (user_data can do everything) — decide: user_data only bootstraps (install agent, join cluster); Ansible owns ongoing config.
2. Hand-off mechanics: TF outputs instance IPs → Ansible dynamic inventory (`aws_ec2` plugin) filtered by tags. Run a ping across the fleet.
**VALIDATE:** zero manual copy of IPs; ADR committed.

---

## Part D — Advanced Terraform (Staff-level material)

### EXERCISE 7.11 — Testing Terraform [A] [90 min]

**STEPS**

1. `terraform test` (native test framework): unit-test the VPC module — assert subnet count, CIDR math, NAT gateway count vs `nat_ha` flag; mock providers for speed.
2. Terratest (Go) for integration: apply module to real AWS in a test, assert with AWS SDK (route exists), destroy. (This is the "test automation frameworks" line in the slice Staff JD — screenshot-worthy.)
3. Contract testing pattern: outputs consumed downstream — test that `subnet_ids` keys stay stable (breaking change detection).
**VALIDATE:** `terraform test` green; one Terratest run with real apply/destroy captured.

### EXERCISE 7.12 — GitOps for Terraform (slice Staff JD verbatim) [A] [60 min]

**STEPS**

1. Options survey (write the matrix): Atlantis (PR-comment applies, self-hosted), Terraform Cloud/HCP (runs + policy sets + cost), Spacelift/env0 (platform), GitHub Actions DIY (Module 09 builds this), ArgoCD+TF-controller (niche).
2. Build DIY now (final wiring Module 09): S3+Dynamo-style backend lock + GH Actions `plan` on PR + `apply` on merge with environment protection rules (required reviewers on `prod` environment).
3. State access control: who/what can `terraform apply` — CI role only (humans read-only on prod state bucket) → your first real "changes only via pipeline" control. **This sentence wins interviews: "humans can't apply; only merge does."**
**VALIDATE:** matrix + ADR-0007 (DIY now, evaluate Atlantis later); CI role separation designed on paper, implemented in Module 09/13.

### EXERCISE 7.13 — Failure & recovery drills [A] [60 min]

**STEPS**

1. Mid-apply crash: `kill -9` terraform during RDS creation → lock held, resource tainted/unknown; recover (wait lock TTL/force-unlock, plan to converge).
2. Partial state: `state rm` a subnet, plan → TF wants to create duplicate; `import` it back.
3. Taint cycle: `terraform taint` an ASG → replace on next apply; `-replace` flag modern equivalent.
4. Provider upgrade drill: AWS provider 5.x→6.x style major bump in a branch; read CHANGELOG; plan review for surprising replacements; canary on dev first.
5. Write `runbooks/terraform-emergencies.md` with all four.
**VALIDATE:** runbook committed; each drill actually executed.

---

## Module 07 exit gate

- [ ] **Zero console-created resources remain.** Tag audit script (`aws resourcegroupstaggingapi`) proves 100% TF-managed coverage.
- [ ] Repo layout: `modules/` (vpc, eks, rds, iam-irsa) + `environments/{dev,prod}/{network,compute,data,eks}`; plans clean; drift detection running.
- [ ] State surgery runbook; testing pyramid (native tests + one Terratest); GitOps-for-TF design written.
- [ ] You can whiteboard: state lifecycle, plan/apply/lock flow, module versioning strategy, why directories > workspaces.

**Onward:** `08-Ansible.md` — configure everything the Terraform way can't.
