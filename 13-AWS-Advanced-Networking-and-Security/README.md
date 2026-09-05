# Module 13 — AWS at Scale: Multi-Account, Advanced Networking & Security/Compliance (Week 18)

> **JD coverage:** "Architect and manage VPCs, routing, security groups, NACLs, ALB/NLB, and hybrid or private connectivity patterns" (slice SRE-3); "multi-account setup", "least-privilege and just-in-time access", "audit trails", "compliance (RBI, NPCI, PCI) as guardrails" (slice SRE-2); "defense-in-depth using Cloudflare, network firewalls, IAM" (slice SRE-3); "GitOps workflows for Terraform" and cost governance (slice Staff).
> This is your networking-engineer advantage module. Everything is Terraform-from-here-on.

---

## Part A — Multi-account architecture (AWS Organizations)

### EXERCISE 13.1 — Organizations: accounts, OUs, SCPs [I] [2 hrs]

**STEPS**

1. Convert your account into an Organization (management account = current; note best practice is a bare management account — document the exception).
2. Create OUs: `Security`, `Infrastructure`, `Workloads` (Prod/NonProd under it), `Sandbox`. Create member accounts via `aws organizations create-account` (or TF `aws_organizations_account`): `northpay-security` (log archive + audit), `northpay-network` (TGW, shared DNS — Part B), `northpay-prod`, `northpay-nonprod`. Assume `OrganizationAccountAccessRole` into each.
3. **Service Control Policies** (the guardrails the JDs mean):

- Deny regions outside allowlist (`ap-south-1`, `ap-southeast-1`) on all OUs.
- Deny disabling CloudTrail/Config anywhere.
- Deny unencrypted S3/RDS creation (condition-based).
- Tag-enforcement strategy note (SCPs can't fully enforce tags — pair with OPA/Config rules).
- Sandbox OU: allow *except* expensive services (deny `ec2:RunInstances` above t3.medium via condition? — not directly possible; use instance-type condition keys on ec2:RunInstances — yes it exists: `ec2:InstanceType`).

4. Test each SCP: try forbidden action as admin in member account → `explicit deny` error anatomy (learn to read "with an explicit deny in a service control policy" — the message that tells you instantly it's an SCP).
5. Centralized CloudTrail → security account bucket (org trail); Config aggregator; GuardDuty org enablement (delegate admin to security account).
6. Identity Center multi-account: permission sets `Admin` (prod break-glass only), `PowerUser` (nonprod), `ReadOnly` (everywhere, default). **This is least-privilege at org scale + your JIT story.**
7. Cost: org-level CUR (Cost & Usage Report) to S3 + Athena query (top 10 services by account) — slice Staff's "cost baselines and reporting" starts here.
**VALIDATE:** SCP denies captured (screenshot-worthy errors); org trail aggregating; permission-set access matrix committed; CUR query runs.

### EXERCISE 13.2 — Cross-account patterns [I] [90 min]

**STEPS**

1. Cross-account IAM roles: `np-prod-deployer` in prod account trusting the CI role in a shared `northpay-tools`/mgmt context; `sts:AssumeRole` chains (feel role chaining; note 1-hour session cap when chaining).
2. Cross-account ECR pull (prod pulls images from tools account repo policy) — wire it, since prod EKS nodes pull images.
3. RAM (Resource Access Manager): share the prod VPC's subnets with... actually don't (design note: shared VPCs vs per-account VPCs — write the tradeoff; NorthPay: per-account VPCs + TGW, shared services account for endpoints).
4. Cross-account S3/KMS: security account's log bucket policy + KMS key policy both needed (the classic "I allowed the bucket but KMS still denies" — reproduce and fix; **kms:ViaService condition**).
5. Audit drill: as an auditor with ReadOnly, answer "who can touch prod RDS?" using only Access Analyzer + IAM policy evaluation logic (`aws iam simulate-principal-policy`).
**VALIDATE:** ECR cross-account pull works; KMS double-policy issue reproduced & fixed with the explanation written.

---

## Part B — Advanced networking (your crown jewel)

### EXERCISE 13.3 — Transit Gateway hub-and-spoke [A] [3 hrs]

**STEPS**

1. Terraform: TGW in network account; share to org via RAM; attach prod + nonprod + network-account VPCs (3 attachments); appliance-mode considerations noted.
2. Route domains: two TGW route tables — `prod-rt` (isolated: prod VPCs see only each other + shared services), `shared-rt` (nonprod + shared services). Implement segmented routing = security zones at the routing layer (like VRFs — say that in interviews, because it is exactly that).
3. VPC route tables updated; SGs referencing across TGW (security group referencing works across TGW-attached VPCs in same region — prove it; over peering it also works same-region).
4. Centralized egress: route all spoke internet egress via network-account egress VPC (NAT fleet there); insert inspection later. Tradeoff paragraph: central egress (control, NAT cost concentration, blast radius) vs distributed NAT (simple, scattered).
5. VPC peering vs TGW: build one peering (nonprod↔tools) and feel route-table non-transitivity — then articulate: peering = point-to-point, non-transitive, fine ≤ 10-ish VPCs; TGW = transitive hub, route domains, scales to hundreds, costs per-attachment + per-GB.
6. Multicast/IGMP note (skip implementing; know TGW supports it — niche interview trivia for finance).
**VALIDATE:** prod↔nonprod CANNOT ping (route domain isolation proven), both CAN reach shared services; egress flows via central NAT (flow log evidence).

### EXERCISE 13.4 — Hybrid connectivity: Site-to-Site VPN + BGP with your on-prem sim [A] [3 hrs] ← *the exercise most candidates can't do*

**STEPS**

1. Customer gateway = your `legacy-dc` strongSwan public IP (or NAT-t behind your router with UDP500/4500 forwarded — home-lab reality; document the CGW config with your actual IP; dynamic IP handling note: DDNS or re-create).
2. Terraform: VGW (or TGW as VPN attachment — do TGW: `aws_vpn_connection` with `transit_gateway_id`), CGW, VPN connection with **two tunnels** (AWS always provisions two; you'll see why).
3. strongSwan on legacy-dc: `ipsec.conf` from the AWS-generated config file (download per tunnel — choose strongSwan template), BGP via FRRouting (FRR `bgpd`: ASN 65000 side vs AWS side 64512; advertise 192.168.100.0/24; **you know BGP cold — this is where you fly**).
4. Verify: tunnels UP, BGP established, routes propagated into TGW route table; ping prod private instance from legacy-dc; traceroute shows the path. Then kill tunnel 1 → traffic continues via tunnel 2 (measure failover seconds; BGP timers vs route propagation).
5. Route engineering drills: AS-path prepend on your advertisements to influence return path; MED usage on AWS side (site-to-site VPN prefers... test it); summarization at TGW.
6. Private VIF/Direct Connect: theory deep-dive (physical circuit, LAG, private/public VIFs, Direct Connect Gateway → TGW for multi-region) — write the "when DX vs VPN" decision (consistent latency/bandwidth/egress-cost at scale vs $0 and instant).
7. MTU/MSS on tunnels: TCP MSS clamping on strongSwan (`iptables TCPMSS`) — prove a hang without it using a large TLS handshake; the classic VPN silent-killer.
**VALIDATE:** BGP session output (`show ip bgp summary`) committed; tunnel failover timed; on-prem VM reaches prod RDS through the tunnel (SG allows 192.168.100.0/24 — least privilege!).

### EXERCISE 13.5 — Hybrid DNS: Route53 Resolver endpoints [I] [90 min]

**STEPS**

1. Problem statement (write it): on-prem VMs must resolve `db.internal.northpay` (private hosted zone); AWS instances must resolve `legacy.northpay.corp` (your dnsmasq zone).
2. Inbound endpoint (on-prem → AWS): resolver ENIs in network account; point legacy-dc dnsmasq: `server=/internal.northpay/<inbound-ENI-IP>`.
3. Outbound endpoint + forwarding rule: AWS → on-prem for `northpay.corp` → dnsmasq IP; conditional forwarding across the VPN.
4. TTL & failure behaviors: resolver endpoint ENI down (kill one — they're per-AZ; two for HA); split-horizon correctness tests from both sides.
5. Alternatives table: resolver endpoints vs simple conditional forwarders vs running your own BIND in AWS; cost of endpoints (~$0.125/hr/ENI adds up — note).
**VALIDATE:** cross-resolution working both directions over the VPN; one-ENI failure tolerated.

### EXERCISE 13.6 — PrivateLink & service-oriented connectivity [I] [75 min]

**STEPS**

1. Expose the legacy payroll service *from* prod account *to* nonprod via PrivateLink: NLB → endpoint service → interface endpoint in consumer VPC. Prove: consumer reaches it via local ENI IP; no routing between VPC CIDRs (that's the point — overlapping CIDRs OK! Create an overlap deliberately to feel why PrivateLink saves mergers/ acquisitions/ partner integrations).
2. Endpoint policies: restrict which principals can connect; acceptance workflow.
3. vs TGW vs peering decision matrix (write it): PrivateLink = service-level, one-directional, overlap-safe, per-service exposure — the SaaS/partner pattern; TGW = network-level full mesh within trust boundary.
4. Reverse use: your fintech partners consume *your* settlement API via PrivateLink with fixed ENI IPs they can whitelist — fintech-grade answer for partner connectivity.
**VALIDATE:** overlapping-CIDR communication via PrivateLink works; endpoint policy rejection demonstrated.

### EXERCISE 13.7 — Edge: CloudFront, WAF, Cloudflare, Global Accelerator [I] [2 hrs]

**STEPS**

1. CloudFront in front of the ALB (OAC pattern for S3 assets + ALB origin for api): behaviors (static vs api paths), cache policies (TTL, key composition), TLS, custom error pages. Measure latency from your location before/after for static assets.
2. WAF on the ALB + CloudFront: AWS managed rule groups (Core, SQLi, KnownBadInputs), rate-based rule (2k/5min/IP — test with gen_load, watch blocks), geo-blocking rule (fintech: allow IN only for settlement endpoints — RBI-flavored requirement), logging to S3 + Athena query of blocks.
3. **Cloudflare in front** (Deel/slice SRE-3 name it): free plan, point your domain's NS there — proxy (orange-cloud) the web hostname: TLS modes (Flexible vs Full-Strict — always Strict; prove Flexible's MITM hole conceptually), caching rules, firewall rules (rate limiting, bot fight), Workers concept (edge compute), and origin-lockdown (only allow Cloudflare IP ranges to reach your ALB — AWS-managed prefix list exists for this! Implement the SG rule). Failover: Cloudflare LB concept vs Route53.
4. Global Accelerator: create one for the ALB (static anycast IPs) — when GA beats CloudFront (non-HTTP, TCP/UDP, static IP whitelisting for partners) — fintech partners demand static IPs; now you know the tool.
5. mTLS at the edge: Cloudflare API Shield / ALB mutual TLS (mTLS on ALB exists — verify client certs) — for partner APIs.
**VALIDATE:** rate-limit blocks observed in WAF logs; origin only reachable via Cloudflare ranges (direct ALB hostname access fails — lockdown proof); TLS-Strict enforced.

---

## Part C — Security & compliance as code

### EXERCISE 13.8 — Secrets, keys, and encryption strategy [I] [75 min]

**STEPS**

1. KMS: symmetric CMKs per domain (`np-data`, `np-logs`); key policies vs grants; rotation (automatic yearly); envelope encryption explanation (data keys vs master keys — draw it once).
2. Encrypt everything audit: Config managed rules (`encrypted-volumes`, `rds-storage-encrypted`, `s3-bucket-server-side-encryption-enabled`) via conformance pack — org-wide, auto-remediation (SSM documents) for dev OU only (blast-radius thinking on auto-remediation).
3. Secrets Manager vs SSM Parameter Store vs Vault (the matrix: rotation built-in? cost/secret? cross-account? k8s integration?) — NorthPay: SM for app secrets (ESO), SSM PS for config, Vault evaluated-not-adopted (document why).
4. Rotation drill: rotate RDS secret (30-day auto already on from Module 03) — force rotation now, verify app still connects (it re-reads? connection pool refresh window — find the gap and document it; this gap is a real incident class).
5. Certificate lifecycle: ACM private CA (cost note — $400/mo; use it in design docs, not in lab) vs cert-manager CA for internal mTLS (Module 16).
**VALIDATE:** conformance pack deployed org-wide; one auto-remediation observed; rotation gap documented with mitigation.

### EXERCISE 13.9 — Detection & response: GuardDuty, Security Hub, Macie [I] [60 min]

**STEPS**

1. GuardDuty org-wide (delegated admin) — generate findings (port-scan your own instance from a throwaway IP; `aws guardduty create-sample-findings`), finding → EventBridge → SNS paging path.
2. Security Hub: enable FSBP standard + CIS AWS Foundations; treat it as your compliance dashboard; suppress-with-reason workflow for accepted risks.
3. Macie discovery job on your logs bucket (find planted fake PAN data in a test file — PCI angle: "where does cardholder data actually live?"); data classification thinking.
4. Detective/Inspector: ECR continuous scanning + Inspector for EC2 — agent story vs your trivy gates in CI (defense in depth again: build-time + run-time).
5. Incident response runbook: finding types → triage steps → containment (isolate SG, snapshot before terminate — forensics-first mindset: **preserve evidence, then remediate**).
**VALIDATE:** sample finding paged you; Macie found the planted PANs; IR runbook committed.

### EXERCISE 13.10 — Compliance-as-code program (RBI/PCI-style) [A] [2 hrs]

**STEPS**

1. Write NorthPay's control baseline: pick 15 controls (encryption at rest/in transit, MFA, logging retention 90d+/1y cold, no public DBs, SG 0.0.0.0/0 restrictions, backup existence + tested restore, access review cadence, change management via PR-only, secrets in SM, network segmentation prod/nonprod, vulnerability scanning cadence, IR runbooks, DR drills, data residency ap-south-1, audit trail integrity).
2. Implement each control as: Config rule / SCP / OPA policy / pipeline gate / scheduled script — with an evidence artifact (query, report) per control. Build `compliance/` folder in infra repo: control → implementation → evidence link.
3. Evidence automation: weekly EventBridge → Lambda-free approach: Step Functions or a cron script generating `compliance-report.md` (Config rule status, failed items, open risks) committed to Git — auditor-ready history.
4. Access review drill: quarterly process — script listing all human access (Identity Center assignments), stale (>90d unused) credentials, wildcard policies; run it, file findings as tickets.
5. Data residency enforcement: SCP region allowlist (done in 13.1) + Config rule for cross-region replication settings review; write the RBI-flavored rationale paragraph.
**VALIDATE:** 15 controls each mapped to a technical enforcement + evidence; compliance report auto-generated; access review executed once for real.

---

## Module 13 exit gate

- [ ] 5-account org with OU guardrails, org CloudTrail, delegated security services, permission-set model.
- [ ] TGW hub-and-spoke with route-domain isolation + central egress; BGP VPN to your on-prem sim with measured tunnel failover; hybrid DNS both directions; PrivateLink with overlapping CIDRs.
- [ ] Edge stack: CloudFront+WAF (rate/geo/managed rules) and Cloudflare strict-TLS with origin lockdown; GA understood.
- [ ] Compliance-as-code: 15 controls enforced with weekly evidence reports.
- [ ] You can whiteboard NorthPay's full network: on-prem ↔ VPN/BGP ↔ TGW ↔ segmented VPCs ↔ PrivateLink services ↔ edge. (Practice this whiteboard — it wins Staff interviews.)

**Onward:** `14-Multicloud.md` — same muscles, GCP + Azure.
