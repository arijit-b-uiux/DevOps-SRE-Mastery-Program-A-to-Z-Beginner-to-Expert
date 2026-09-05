# Module 03 — AWS Foundations (Weeks 2–4)

> **JD coverage:** "Strong hands-on experience with AWS (EC2, ASG, ALB/NLB, IAM, VPC, networking)" (slice SRE-3); "EC2, S3, IAM, VPC, RDS/Aurora, load balancing, multi-account setup" (slice SRE-2); "EKS, S3, Postgres, Mongo, SES, SNS, SQS" (Deel). This module deliberately starts in the **console** (to build mental models) and ends with a rule: *from Module 07 onward, console = read-only.* Tag every resource per your Day-1 standard.

**NorthPay story:** you migrate the demo app from your laptop to a real 3-tier, multi-AZ AWS deployment. Everything you build here manually, you will rebuild in Terraform in Module 07 — and feel the difference.

---

## Part A — IAM & account structure

### EXERCISE 3.1 — IAM fundamentals: users, roles, policies [B] [60 min]

**STEPS**

1. Console → IAM: create role `np-ec2-app-role` with trust policy for EC2; attach `AmazonSSMManagedInstanceCore` (for Session Manager — no SSH keys on instances from now on) and a custom policy `np-app-s3-read` allowing `s3:GetObject` on `arn:aws:s3:::northpay-app-assets-*/*` only.
2. Write that custom policy by hand in JSON (no visual editor). Understand: `Version`, `Statement`, `Effect`, `Action`, `Resource`, `Condition`.
3. Policy Simulator: test whether the role can `s3:PutObject` on the bucket (it can't — verify).
4. Create IAM user `ci-bot` with NO console access, no access keys yet (Module 09 replaces keys with OIDC — write that in the log as the plan).
5. `aws sts get-caller-identity` from CLI; then `aws sts assume-role` into `np-ec2-app-role` with session name, observe temporary credentials.
**VALIDATE:** policy simulator screenshot-level evidence in your log (text is fine); assume-role works.
**TRADEOFFS:** roles vs users (roles = temporary creds, auditable assumers; users = long-lived keys = leak risk). Why every compute identity should be a role: EC2 instance profiles, IRSA in EKS (Module 05/07), Lambda exec roles.

### EXERCISE 3.2 — Least privilege in practice: permission boundaries + SCP preview [I] [75 min]

**STEPS**

1. Create a "developer" IAM policy: broad EC2/RDS read, but write only to resources tagged `env=dev` (use `aws:ResourceTag` conditions) and require `aws:RequestedRegion = ap-south-1`.
2. Attach a permission boundary to a test user limiting max actions to a whitelist; show the *intersection* logic: effective = identity policy ∩ boundary (∩ SCP at org level — Module 13 turns this on for real).
3. Access Advisor + credential report: generate, read "last used" columns. Runbook: quarterly access review process (fintech compliance artifact).
4. IAM Access Analyzer: run it; interpret an "external access" finding by creating a test bucket policy with a wildcard principal, watch it flag, fix it.
**VALIDATE:** test user with boundary cannot `iam:*` even when identity policy allows it — prove with CLI error.
**TRADEOFFS:** RBAC-via-IAM-policies vs ABAC (tag-based). ABAC scales to many teams without policy explosion; requires tagging discipline (your Day-1 standard pays off). Interviewers at slice ask "how do you keep least privilege at 50-engineer scale?" — this is the answer skeleton.

### EXERCISE 3.3 — Session Manager replaces SSH [B] [30 min]

**STEPS**

1. Launch a t3.micro with the `np-ec2-app-role` instance profile, no key pair, no open port 22 in the SG.
2. Connect via Session Manager (console and `aws ssm start-session --target i-...`).
3. Enable session logging to S3 (audit trail — bank-grade). Run commands, then read your own session log in S3.
**VALIDATE:** no SSH key exists, yet you have a shell; session log object exists in S3.
**WHY-WHEN-WHAT:** eliminates key distribution, gives central audit, works through restrictive networks (port 443 out only). Tradeoff: requires SSM agent + IAM; some debugging tools (tcpdump streaming) are clunkier. slice SRE-2's "just-in-time access" = this + Identity Center.

---

## Part B — VPC & networking (your home turf — go deep)

### EXERCISE 3.4 — Build the NorthPay VPC by hand [B] [90 min]

**STEPS** (console first, CLI second time — do it twice)

1. VPC `np-vpc` `10.10.0.0/16`, tenancy default. Enable DNS hostnames + resolution (explain what each does for RDS/EKS later).
2. Subnets across 3 AZs (ap-south-1a/b/c):

- public: 10.10.0.0/22, 10.10.4.0/22, 10.10.8.0/22
- private-app: 10.10.16.0/20, 10.10.32.0/20, 10.10.48.0/20
- private-data: 10.10.64.0/22, 10.10.68.0/22, 10.10.72.0/22
Justify in your log: why /16, why non-overlapping blocks with room to grow, why separate data subnets (RDS subnet groups, different NACL posture).

3. IGW attached; public route table `0.0.0.0/0 → igw`; associate public subnets.
4. NAT: one NAT gateway in AZ-a public subnet. Private route tables → NAT. **Note the cost line (\$0.045/hr + data) and the single-AZ weakness — Exercise 3.6 fixes it.**
5. Security groups: `np-alb-sg` (80/443 from 0.0.0.0/0), `np-app-sg` (3000 from np-alb-sg — SG-to-SG referencing, explain why this beats CIDR), `np-db-sg` (5432 from np-app-sg). Egress: app → 443 to 0.0.0.0/0 (patched images, APIs), db → nowhere but app SG.
6. NACLs: leave default-allow on app subnets, but write a restrictive NACL for data subnets (5432 in from app subnets only, ephemeral ports back). Explain NACL statelessness vs SG statefulness to yourself in writing — classic interview question.
7. VPC Flow Logs → CloudWatch Logs group `np-vpc-flowlogs`. Generate accepted + rejected traffic; query with Logs Insights: top rejected dest ports.
**VALIDATE:** a t3.micro in public-a reaches the internet; one in private-app-a reaches internet via NAT; one in private-data-a does NOT reach the internet (no route) — all three proven.
**TRADEOFFS:** This is where you shine. Write 10 lines: CIDR planning for future peering/TGW (no overlap with 192.168.100/200 on-prem sim or 10.20.x nonprod), SG-referencing for micro-segmentation, NACLs as coarse guardrails. Compare with Cisco ACLs/VRFs/VSANs in an ADR — your network background is a weapon here.

### EXERCISE 3.5 — VPC endpoints: private connectivity to AWS services [I] [60 min]

**STEPS**

1. Gateway endpoint for S3 (free) attached to private route tables. Before/after test: from a private instance, `aws s3 ls` — watch flow logs (no more NAT path for S3) and NAT data processing drop.
2. Interface endpoints (cost \$): `ssm`, `ssmmessages`, `ec2messages` (so private instances can Session-Manager without NAT), `secretsmanager`. Security group on endpoints: 443 from app SG.
3. Private DNS: understand the endpoint's private DNS name overriding the public one; test `dig ssm.ap-south-1.amazonaws.com` from private instance → resolves to ENI IPs.
4. Endpoint policy on the S3 gateway endpoint: restrict to `northpay-*` buckets only.
**VALIDATE:** private-data subnet instance (no internet route) uses SSM via endpoints.
**TRADEOFFS:** interface endpoints cost ~\$7/mo each per AZ + data — at what NAT data volume do endpoints win financially? Do the math in your log (NAT \$0.045/GB vs endpoint \$0.01/GB + hourly). Multi-AZ endpoint placement vs cost. This exact calculation appears in cost-optimization interviews.

### EXERCISE 3.6 — Multi-AZ NAT + failure drill [I] [45 min]

**STEPS**

1. Add NAT gateways in AZ-b and AZ-c; per-AZ private route tables → local NAT. Cost note: 3× hourly — acceptable for prod, kill NAT-b/c after the drill.
2. Failure drill: disable the AZ-a NAT gateway's route (point to a blackhole), watch private-a instances lose internet; then fix routing to fail over via TGW-less design (there isn't one — that's the point: NAT gateways are per-AZ, routes must be per-AZ).
3. Write runbook `nat-gateway-failure.md` incl. detection (CloudWatch `ErrorPortAllocation`, `PacketsDropCount` metrics).
**VALIDATE:** evidence of failover reasoning; runbook committed.
**TRADEOFFS:** 1 NAT (cheap, single point) vs 3 NAT (resilient, 3× cost) vs NAT instances (cheaper, you manage them, worse throughput/HA — but as a network engineer, deploy one NAT *instance* briefly to feel it: `iptables masquerade` on EC2 = your Module 02 netns NAT skill at cloud scale).

---

## Part C — Compute: EC2, ASG, ELB

### EXERCISE 3.7 — EC2 lifecycle + metadata + user-data [B] [60 min]

**STEPS**

1. Launch t3.micro Amazon Linux 2023 in private-app-a, SG `np-app-sg`, instance profile `np-ec2-app-role`, user-data script installing Node 20 + pulling app from a zip in S3 (upload your app zip to `northpay-app-assets-<suffix>` first) + starting via systemd unit (reuse Module 02's unit file!).
2. IMDSv2: enforce `HttpTokens=required`; from the instance get a token (`curl -X PUT ... -H "X-aws-ec2-metadata-token-ttl-seconds: 300"`), query `instance-id`, AZ, IAM role creds. Explain the SSRF angle (why v2's PUT+token defeats naive SSRF — security interview favorite).
3. `stress-ng` on the box → watch CloudWatch CPU alarm you create fire an SNS email. Detailed monitoring vs basic — cost & granularity tradeoff.
4. Stop/start vs reboot vs terminate — what happens to the instance-store and public IP each time? Prove each.
5. Create an AMI from the configured instance; launch a second instance from it — feel golden images. (Module 15 does AMI rotation at scale; note the idea.)
**VALIDATE:** app reachable from another instance in app SG on :3000; IMDSv1 call fails, v2 succeeds.

### EXERCISE 3.8 — ALB + ASG: the standard 3-tier pattern [I] [2 hrs]

**STEPS**

1. Target group `np-api-tg`: port 3000, health check `/healthz`, interval 15s, healthy threshold 2, unhealthy 3 — explain each knob's effect on failover speed vs false positives.
2. ALB `np-api-alb` in 3 public subnets, SG `np-alb-sg`, listener 80 → forward to TG. (HTTPS comes in 3.9.)
3. Launch template `np-api-lt`: your AMI, instance profile, SG, user-data for app version injection.
4. ASG `np-api-asg`: private-app subnets (all 3 AZ), min 2 / desired 3 / max 6, attach TG. Watch instances register and go healthy.
5. `curl` the ALB DNS repeatedly; kill one instance (`aws ec2 terminate-instances`) — watch ASG replace it and ALB drain connections. Note recovery time in your log (it's your first measured RTO!).
6. Scaling policies: target tracking on CPU 50%; then scheduled scaling (scale to 6 at 09:00, 2 at 21:00 — "business hours batch load" story). Generate load with `assets/scripts/gen_load.sh` from a bastion; watch scale-out; check cooldown behavior; then watch scale-IN protection and connection draining.
7. ALB access logs → S3; analyze with your awk skills from Module 02 (top paths, 5xx).
**VALIDATE:** zero-downtime instance replacement demonstrated; scale-out event under load captured with timestamps; ALB logs analyzed.
**TRADEOFFS:** ALB (L7, path/host routing, WAF attach, WebSockets) vs NLB (L4, ultra-low latency, static IPs, TLS passthrough, preserves client IP) vs CLB (legacy — know it exists, never choose it). As a network engineer, map: ALB ≈ L7 reverse proxy farm; NLB ≈ L4 DSR-ish forwarding. When fintech: NLB in front of ALB for static IP whitelisting by partners — a real pattern you'll build in Module 13.

### EXERCISE 3.9 — TLS, ACM, and end-to-end encryption [I] [60 min]

**STEPS**

1. Register a cheap domain (or use a free subdomain provider) → Route53 hosted zone.
2. ACM: request public cert for `api.northpay.<yourdomain>` with DNS validation (CNAME into Route53 — one click in console; do it once via CLI too).
3. ALB HTTPS listener 443 with the cert → forward to TG; HTTP→HTTPS redirect rule on 80.
4. Route53 alias record `api.northpay...` → ALB (why alias beats CNAME at zone apex; alias = free + health-checkable).
5. mTLS/end-to-end: re-encrypt ALB→instance with a self-signed cert on nginx sidecar? For now: understand the option and its operational cost (cert rotation on instances) — Module 16 (Istio mTLS) solves it properly.
6. Test: `curl -v https://api...` — inspect the handshake, cert chain, TLS version. Enforce TLS 1.2+ via listener security policy; test with `openssl s_client -tls1_1` (should fail) vs `-tls1_3`.
**VALIDATE:** A+ style TLS posture (TLS1.1 refused); redirect works; DNS resolves.
**TRADEOFFS:** ACM (free, auto-renew, AWS-only, can't export) vs Let's Encrypt (portable, you own renewal automation) vs enterprise CA. Fintech answer: ACM for AWS-facing, but know cert-manager (Module 05) for in-cluster.

---

## Part D — S3 & storage

### EXERCISE 3.10 — S3 deep: buckets, policies, lifecycle, replication [B→I] [90 min]

**STEPS**

1. Buckets: `northpay-app-assets-<sfx>`, `northpay-logs-<sfx>`, `northpay-backups-<sfx>`. Default encryption SSE-S3; block all public access (account level too).
2. Versioning on backups bucket: upload object, overwrite, delete — recover via version list. MFA-delete concept (know it; skip enabling — root-only pain).
3. Lifecycle policies: logs bucket → Standard-IA after 30d, Glacier Instant after 90d, expire after 365d. Estimate cost delta for 1 TB/mo in your log (FinOps muscle).
4. Bucket policy lab: allow read from your VPC endpoint only (`aws:SourceVpce` condition); test from private instance via endpoint (works) vs your laptop (denied).
5. Presigned URLs: generate GET and PUT URLs via CLI/SDK; upload from laptop without AWS creds. Expiry tradeoffs (7 days max for SigV4).
6. S3 replication (CRR): backups bucket → second bucket in ap-southeast-1 with a replication role. Verify object appears. **This is your first cross-region DR primitive — flag it for Module 15.**
7. S3 Storage Lens dashboard (free tier metrics) + `s3api list-multipart-uploads` cleanup of orphans (cost leak classic).
8. Static website hosting briefly — then explain why CloudFront+OAC is the prod pattern (Module 13 builds it).
**VALIDATE:** CRR object replicated; endpoint-only policy enforced both ways; lifecycle rules visible via CLI.
**TRADEOFFS:** storage classes matrix (access pattern vs retrieval cost — Glacier Flexible's retrieval fee can exceed storage savings for frequently-read data); S3 consistency (strong read-after-write since 2020 — older interview answers say otherwise, correct them).

### EXERCISE 3.11 — EBS & EFS: block vs file in AWS [I] [45 min]

**STEPS**

1. EBS: attach a 20 GB gp3 volume to an instance, `mkfs.xfs`, mount, add to fstab **by UUID** (device names lie after reboots — prove it). Baseline `fio` numbers; then change IOPS/throughput live (`modify-volume` — no downtime!) and re-bench.
2. Snapshot it; create a volume in another AZ from the snapshot (cross-AZ restore = migration primitive); enable fast snapshot restore? (No — cost; know when it's worth it: DR runbooks.)
3. Detach while mounted → observe stuck I/O (your D-state friend from Module 02 returns), then clean handling: `umount` first. 
4. EFS: create file system, mount targets in app subnets, mount on 2 instances concurrently (NFSv4.1), write from both. Use case: shared assets; note latency vs EBS and when EFS beats S3 (POSIX semantics needed) and loses (cost/scale).
5. Resize2fs/xfs_growfs after volume grow — the full "grow disk online" runbook.
**VALIDATE:** live IOPS change measured; EFS concurrent write demo.
**TRADEOFFS:** gp3 vs io2 (when do you *actually* need io2's 64k IOPS?), EBS single-AZ reality vs EFS multi-AZ. Interview: "EBS volume degraded — what does AWS do?" (auto-replication within AZ; you replace volume from snapshot.)

---

## Part E — DNS: Route53

### EXERCISE 3.12 — Route53 routing policies as traffic engineering [I] [75 min]

**STEPS** (use your domain from 3.9)

1. Hosted zone deep dive: SOA/NS records, query logging to CloudWatch.
2. Build and test each policy with real records:

- **Weighted:** `api` → 90% ALB-a, 10% ALB-b (two ALBs or two regions later); verify with repeated dig from multiple resolvers.
- **Latency-based:** records for ap-south-1 and ap-southeast-1 endpoints; test from your laptop vs a Singapore VM (on-prem sim can proxy) — see different answers.
- **Failover:** primary/secondary with health checks; break primary (stop instances) and watch DNS failover timing (TTL vs health check interval math — measure actual failover time!).
- **Geolocation/geoproximity:** concept + when fintech uses it (data residency — RBI angle).

3. Private hosted zone `internal.northpay` associated with np-vpc: `db.internal.northpay` → RDS endpoint (next exercise); understand split-horizon: same name resolves differently inside/outside VPC. **Module 13 extends this to hybrid DNS with your on-prem sim.**
4. Health checks: TCP vs HTTP vs calculated; string matching; why health-checkers need SG/firewall allowance from Route53 health-checker IP ranges (there's a published JSON — your firewall background shows).
**VALIDATE:** measured DNS failover time recorded; split-horizon dig outputs captured.
**TRADEOFFS:** DNS failover is minutes-scale (TTL + check intervals) — never your only HA mechanism for RTO < 5 min; pair with ALB/multi-AZ for seconds-scale. When DNS *is* the right tool: region evacuation, active-passive DR (Module 15).

---

## Part F — Messaging & email: SQS, SNS, SES

### EXERCISE 3.13 — SQS: queues, DLQs, visibility timeout [I] [75 min]

**STEPS**

1. Standard queue `np-settlements`: producer script (Python/boto3, from assets) sends 1000 settlement messages; the NorthPay `worker` container (locally via docker-compose, env pointed at AWS) consumes.
2. Visibility timeout experiment: set 5s, make worker sleep 10s — watch duplicate processing. Fix: timeout > p99 processing + idempotent consumer (worker code has idempotency key — read it). **This is THE SQS interview question.**
3. DLQ: redrive policy maxReceiveCount 3; poison-pill a message (worker crashes on `"amount": -1`); watch it land in DLQ after 3 tries. Then console/CLI redrive back after fixing the worker. Alarm on DLQ depth > 0.
4. FIFO queue `np-payments.fifo`: dedup ID + message group IDs; show ordering within a group, parallelism across groups, 300 TPS limit (3k with batching) — when standard's ~unlimited TPS wins.
5. Long polling vs short: measure empty-receive cost/latency difference.
**VALIDATE:** duplicates demonstrated then eliminated; DLQ round trip; FIFO ordering proof in log.
**TRADEOFFS:** SQS vs SNS vs EventBridge vs Kinesis vs Kafka/MSK — write the comparison table now (fan-out, ordering, replay, throughput, cost). slice/Deel both run queue-heavy payroll systems; expect "design a payment retry system" interviews.

### EXERCISE 3.14 — SNS fan-out + SES basics [B] [45 min]

**STEPS**

1. Topic `np-alerts`: subscribe your email; then fan-out pattern — topic with 2 SQS subscriptions (raw delivery on/off — difference?).
2. Message filtering: attributes on messages (`severity=critical|info`), subscription filter policies; prove info goes only to the info queue.
3. SES: verify your domain (DKIM records into Route53), sandbox-mode send test email via CLI; note production-access request process (out of sandbox) — you'll skip waiting for it but document it; bounce/complaint handling via SNS topic (compliance requirement at payroll companies!).
4. Wire it: CloudWatch alarm → SNS `np-alerts` → email. This is your alerting backbone until Module 11.
**VALIDATE:** filtered fan-out proven; test email received with DKIM pass.

---

## Part G — RDS & relational databases (foundations; deep dive is Module 12)

### EXERCISE 3.15 — RDS PostgreSQL: provision, secure, connect [B] [75 min]

**STEPS**

1. DB subnet group from your private-data subnets. Parameter group `np-pg16`: `log_statement='all'` (lab only!), `shared_preload_libraries='pg_stat_statements'`.
2. Instance `np-postgres`: db.t3.micro, Postgres 16, Multi-AZ **off** (cost — you enable it in 3.17), storage autoscaling on, encryption at rest with default KMS key, deletion protection ON, automated backups 7 days, backup window 03:00-04:00 (why off-peak), enhanced monitoring 15s, performance insights ON.
3. SG `np-db-sg` allows 5432 from `np-app-sg` only. No public access. Ever.
4. Secrets: master password in Secrets Manager (RDS-managed rotation — enable 30-day rotation). Fetch it from an app instance via CLI with the instance role (add a least-privilege secrets policy).
5. Connect from app instance: `psql`, run `assets/seed-data/postgres_seed.sql` (schema + 50k customers/orders via the generator). Update app `.env` to point at RDS; restart; `/readyz` green.
6. IAM database authentication: enable, create `rds_iam` grant, connect with token instead of password. When is this better? (No secrets at rest; short-lived tokens; per-human auditing.)
**VALIDATE:** app serves orders from RDS; secret rotation executes; IAM-auth psql session works.

### EXERCISE 3.16 — RDS operations: snapshots, PITR, read replica [I] [60 min]

**STEPS**

1. Manual snapshot; restore to new instance `np-postgres-restore` (note: restore = new endpoint — connection-string management matters; this is why `db.internal.northpay` private DNS exists — repoint it).
2. Point-in-time recovery drill: note time T; `DELETE FROM orders WHERE ...` (oops); restore to T-1min as another instance; verify row is back; document exact steps + elapsed time (measured RTO for DB PITR). Template: `assets/scripts/pitr_drill.sh`.
3. Read replica in ap-south-1b; run reporting query on replica; observe replication lag metric. Promote? Not cross-region yet (Module 15 does cross-region replica + promotion runbook).
4. CloudWatch alarms: CPU>80, FreeStorageSpace<20%, DatabaseConnections>80, ReplicaLag>60s → SNS `np-alerts`.
**VALIDATE:** PITR recovered the row with evidence; replica lag metric visible.

### EXERCISE 3.17 — HA & failover drill [I] [45 min]

**STEPS**

1. Modify to Multi-AZ (downtime? — modification applies at maintenance window or with brief disruption; read the docs and plan it like a change request, using your `CHANGE-REQUEST.md` template).
2. Reboot with failover; watch the endpoint DNS flip to standby AZ (TTL trickery — the CNAME stays, the A record behind it changes); measure app-visible downtime from your `np-deploy-check` script's log.
3. Aurora preview (theory + console tour): storage architecture (6 copies, 3 AZs, self-healing), reader endpoints, faster failover vs RDS Multi-AZ, cost delta. Module 12 migrates np-postgres → Aurora properly.
**VALIDATE:** failover downtime measured (expect 30–90 s); change-request doc filled.
**TRADEOFFS:** Multi-AZ (sync standby, no read scaling) vs read replicas (async, scale reads, usable for DR with promotion) vs Aurora (both, better, pricier, Postgres-compatible but not Postgres — extension caveats).

---

## Part H — CloudWatch & first observability (until Module 11 takes over)

### EXERCISE 3.18 — CloudWatch agent + custom metrics + dashboards [I] [60 min]

**STEPS**

1. Install CloudWatch agent on app instances via user-data (config: collectd-style metrics — mem, disk, plus `statsd` app metrics). IAM: `CloudWatchAgentServerPolicy` on the role.
2. Custom metric from the app: settlement batch duration (`PutMetricData` via cron'd script); alarm on p90 > threshold.
3. Build dashboard `np-overview`: ALB 5xx, ASG CPU, RDS connections/lag, DLQ depth, custom metric. Share as your daily standup screen from now on.
4. Logs: app logs → CloudWatch Logs via agent; metric filter on `ERROR` → alarm; Logs Insights query practice (top error signatures — reuse Module 02 triage skills, cloud edition).
5. Composite alarms: "page-worthy" = (ALB 5xx>2% for 5m) AND (healthy hosts >= 1) — reduce noise; explain alarm fatigue.
**VALIDATE:** dashboard live; one alarm fires and emails during a stress test.
**TRADEOFFS:** CloudWatch vs Prometheus/Grafana (Module 11): CW = zero-ops, AWS-native, per-metric pricing pain at scale, weak high-cardinality; Prom = standard, rich query, you operate it. Real shops run both (CW for AWS-service truth, Prom for app/K8s).

### EXERCISE 3.19 — EventBridge: the automation fabric [I] [45 min]

**STEPS**

1. Rule: EC2 instance-state change → SNS (know immediately when anything dies).
2. Scheduled rule: nightly `np_stop_after_hours` (Lambda-free — target SSM Automation `AWS-StopEC2Instance` or call your script on a bastion).
3. Event pattern practice: match `source=aws.rds` failover events → post to SNS with input transformer (format a human message).
4. Cross-account event bus: concept + when (org-wide audit events) — Module 13 implements.
**VALIDATE:** stop an instance → email in < 1 min; nightly stop schedule proven next morning in your cost script.

---

## Part I — ECR + the bridge to containers

### EXERCISE 3.20 — ECR: registry operations [B] [30 min]

**STEPS**

1. Repos `northpay/api`, `northpay/worker`, `northpay/web` with scan-on-push, immutable tags, lifecycle policy (keep last 10 images, expire untagged after 7d).
2. Build the api image locally (Module 04 preview): `docker build`, `aws ecr get-login-password | docker login`, tag `:0.1.0` + `:latest` (immutability blocks latest overwrite? — realize the conflict, drop `:latest`, learn why immutable SHA-tagged releases are the GitOps way).
3. View scan findings (CVEs in the base image — you'll fix bases in Module 04).
4. Cross-account pull permissions: write the policy (Module 13 uses it: prod account pulls from shared ECR).
**VALIDATE:** image pushed, scan report read, immutable-tag rejection experienced.

---

## Module 03 exit gate (interview-tested claims you can now make)

- [ ] Full 3-tier NorthPay app on AWS: Route53 → ACM TLS → ALB → ASG (3 AZ) → RDS Postgres, app talking to SQS/SNS, logs+metrics in CloudWatch, **all tagged**, all SGs least-privilege.
- [ ] Measured numbers in your log: ASG replacement time, ALB drain behavior, RDS failover downtime, DNS failover time, PITR RTO.
- [ ] VPC endpoints + flow logs + private DNS understood and demonstrated.
- [ ] Queue failure modes (duplicates, poison, ordering) reproduced and fixed.
- [ ] 10+ mini-postmortems; runbooks for NAT failure, PITR, disk grow, lockout recovery.

**Teardown note:** this stack stays running for Module 04–05 tickets (it's your target environment); stop RDS overnight via your script if cost pinches; NAT gateways b/c deleted after 3.6 (keep one).

**Onward:** `04-Containers-Docker.md`.
