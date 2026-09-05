# Module 08 — Ansible: Configuration Management (Week 9)

> **JD coverage:** listed in your target stack ("Ansible") and implied everywhere ("Install, configure, update and troubleshoot our product microservices and tools including databases" — Deel). Terraform builds the stage; Ansible dresses it: OS hardening, agents, middleware, and anything that lives *inside* instances — including your on-prem sim VMs, which no cloud API can touch.
> Starter code: `assets/ansible/`.

---

## Part A — Core mechanics

### EXERCISE 8.1 — Inventory, facts, and the ping that isn't ping [B] [45 min]

**STEPS**

1. Static inventory `inventories/onprem/hosts.ini`: `legacy-dc`, `branch-office` with vars (ansible_host, ansible_user). Groups: `[dc]`, `[branch]`, `[onprem:children]`.
2. Ad-hoc: `ansible all -m ping` (it's an SSH+Python module, not ICMP — read the docs), `ansible all -m setup` — read the fact tree; `ansible_facts` in playbooks later.
3. Dynamic inventory for AWS: `inventories/aws/aws_ec2.yml` keyed by tag `env`; groups auto-created; `ansible-inventory --graph`.
4. Patterns: `dc:&debian`, `--limit`, `--tags` — surgical targeting is the daily skill.
5. Config: `ansible.cfg` (forks=20, fact caching to jsonfile, stdout callback yaml for readability, retry_files off).
**VALIDATE:** one command gathers facts from both VMs + all tagged EC2 simultaneously.

### EXERCISE 8.2 — Playbooks, handlers, idempotency [B] [75 min]

**STEPS**

1. First playbook `baseline.yml` on onprem VMs: users (ops, deploy), authorized_keys, packages (htop, sysstat, fail2ban, ufw), timezone, sysctl tuning file + handler (`sysctl --system` only when file changed — handlers = change-triggered actions), journald persistent config + restart handler.
2. Idempotency proof: run twice; second run all `ok`, zero `changed`. If anything changes twice, fix it (lineinfile vs blockinfile vs template — choose right).
3. `template` (jinja2) for `/etc/motd` with facts (hostname, IP, env); `copy` vs `template` vs `get_url`.
4. `failed_when`/`changed_when`/`ignore_errors` — command modules always report changed; teach them truth (creates/removes args).
5. Check mode + diff: `--check --diff` as your "terraform plan for servers"; limits (some modules don't support check).
**VALIDATE:** two consecutive runs → 0 changed; `--check` output reads like a plan.

### EXERCISE 8.3 — Roles, collections, and structure [I] [60 min]

**STEPS**

1. `ansible-galaxy init roles/nginx` — role skeleton (tasks, handlers, templates, defaults vs vars precedence — know the 22-level precedence chain at least at "defaults lose, extra vars win" level).
2. Write roles: `nginx` (install, hardened config from template — TLS-only vhost, security headers), `node-exporter` (binary from GitHub releases, systemd unit template, version pinned by default var), `postgres-client`.
3. `requirements.yml` for community collections (community.postgresql, ansible.posix) — pin versions; `ansible-galaxy install -r`.
4. Site playbook composing roles with tags; run only nginx role via tags.
5. Molecule (optional stretch): test the nginx role in a docker container — infra unit tests.
**VALIDATE:** `site.yml` converges a fresh VM to: hardened nginx serving the legacy portal + node-exporter running, one command.

### EXERCISE 8.4 — Secrets with Vault [B] [45 min]

**STEPS**

1. `ansible-vault create group_vars/all/vault.yml` (db password, api keys); vaulted file in Git — safe.
2. Vault IDs + password files (gitignored) vs `--ask-vault-pass` vs script; CI pattern (env var).
3. `no_log: true` on tasks that echo secrets (otherwise logs leak — prove it by forgetting it once).
4. Boundary doc: Ansible Vault (config-time secrets) vs AWS Secrets Manager/SSM (runtime secrets). NorthPay: Ansible delivers *bootstrap* secrets only; apps fetch runtime secrets from SM. Write the rule.
**VALIDATE:** repo contains zero plaintext secrets (`gitleaks` agrees); playbook runs with vaulted vars.

---

## Part B — Real fleet work

### EXERCISE 8.5 — The EKS-node/EC2 hardening role (CIS-flavored) [I] [90 min]

**STEPS**

1. Role `cis-baseline` against your EC2 app instances + onprem VMs: disable unused filesystems, sshd hardening (Module 02's config as template now), password policy, auditd rules (watch /etc, secrets paths), AIDE init (file integrity), automatic security updates (unattended-upgrades / dnf-automatic), login banners (compliance artifact!).
2. Reboot-required handling: `needs-restarting` → reboot handler with `wait_for_connection`.
3. Report: playbook generating a per-host compliance summary (jinja template → markdown report artifact).
4. Diff against manual Module 02 hardening: what did you miss then? (Humility artifact — great interview story about why automation > runbooks for config.)
**VALIDATE:** fresh instance → hardened in one run; report artifact generated; second run idempotent.

### EXERCISE 8.6 — Application deployment with Ansible (pets, not cattle) [I] [75 min]

**GOAL:** deploy the NorthPay api onto the legacy-dc VM (simulating the "legacy server that can't containerize yet") — the pattern for non-K8s estates.
**STEPS**

1. Role `northpay-api`: artifact fetch from S3 (versioned zip), node install, systemd unit from template (your Module 02 unit, templated: MemoryMax, env file), env file from template + vault/SM lookups.
2. Rolling deploy by hand: `serial: 1` across a 2-host group, pre_tasks health-drain (remove from nginx upstream), post_tasks health-gate (uri module polling /healthz, fail = stop the batch). **This is manual canarying — feel it, because Module 09/16 automate it.**
3. Rollback playbook: previous artifact version redeploy + verification; test it for real (deploy broken version → auto-health-gate stops → run rollback).
4. `ansible-pull` mode: understand pull-based config (fleets without inbound SSH); note when it wins (roaming/edge).
**VALIDATE:** broken deploy auto-stopped after host 1; rollback restored service; timeline logged.

### EXERCISE 8.7 — Database ops with Ansible [I] [60 min]

**STEPS**

1. community.postgresql: create DBs/users/privileges on the onprem legacy Postgres (idempotent), load seed data.
2. pg_dump backup role with rotation (port your Bash `np-backup-pg` logic into a role — compare maintainability honestly in your log).
3. Patching drill: minor Postgres version upgrade on legacy-dc with pre-backup, maintenance flag (nginx holding page), upgrade, verify, unflag — full change-window orchestration in one playbook.
**VALIDATE:** version upgraded with backup proof and rollback plan documented (pg_dumpall before + restore tested).

### EXERCISE 8.8 — AWX/Tower: Ansible at team scale [B] [60 min]

**STEPS**

1. Install AWX on kind (operator) or skip to theory if RAM-constrained: projects (Git-backed), inventories, credentials, job templates, RBAC, schedules, surveys.
2. Map concepts to governance: who may run which playbook against which inventory with approval gates — the "approved patterns and change windows" from slice SRE-2.
3. Job template for `cis-baseline` with schedule + slack/email notifications.
**VALIDATE:** one scheduled job executed via AWX; or a written design doc if skipped (note tradeoff).

---

## Part C — Ansible × Terraform × the rest of the program

### EXERCISE 8.9 — Monitoring agent rollout everywhere [I] [45 min]

**STEPS**

1. Role `observability-agents`: node-exporter (Prometheus), fluent-bit (logs), otel-collector (Module 11) — deployed to all EC2 (via dynamic inventory + tags) and onprem VMs in one run.
2. Version pinning + upgrade path: change version var → rolling update with handlers; how you'd canary it by tag-scoped runs (`--limit` a canary group first).
3. Idempotency under drift: hand-edit a config on one host, re-run, watch convergence. Contrast with K8s/GitOps auto-sync (Module 10) — who heals drift here? (Only re-runs: schedule Ansible nightly for pet fleets.)
**VALIDATE:** all hosts report metrics to your future Prometheus (Module 11 scrapes them).

### EXERCISE 8.10 — Ansible in pipelines & the orchestration picture [B] [30 min]

**STEPS**

1. Linting: `ansible-lint` clean on the repo (pipeline gate, Module 09).
2. Write ADR-0008: "When Ansible vs user_data vs K8s vs SSM" — one page. Include: SSM Run Command for ad-hoc fleet ops (no SSH at all) — try it: `aws ssm send-command` a shell script to all env=prod instances, collect outputs.
3. Nightly convergence cron via SSM/AWX for the pet fleet; note it in the ops calendar.
**VALIDATE:** SSM run-command executed across tagged fleet; ADR committed.

---

## Module 08 exit gate

- [ ] 6 roles (baseline, nginx, node-exporter/observability, northpay-api, postgres ops, cis-baseline) — lint-clean, idempotent, vaulted.
- [ ] Broken-deploy rollback demonstrated; database patch window orchestrated.
- [ ] On-prem sim fully Ansible-managed (your VMs are "legacy infrastructure as code" — rare and valuable).
- [ ] ADR on config-management boundaries.

**Onward:** `09-CI-CD-Pipelines.md` — every commit becomes a tested, scanned, deployable artifact.
