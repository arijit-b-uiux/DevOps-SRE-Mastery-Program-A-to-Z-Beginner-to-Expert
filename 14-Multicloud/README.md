# Module 14 — Multicloud: GCP & Azure from Zero to Production-Grade (Weeks 19–20)

> **JD coverage:** "Multicloud exposure, especially with major cloud providers" (slice SRE-2 good-to-have); "Drive adoption of cloud-native and distributed systems practices (Kubernetes, AWS, multi-region architectures)" (slice Staff). Your own goal: "build everything from scratch to the max level" with "multicloud networking" as the differentiator.
> **Method:** not re-learning — *mapping*. Every exercise pairs the AWS concept you own with its GCP/Azure counterpart, then builds it. Free tiers: GCP \$300/90-day credits + always-free tier; Azure \$200/30-day credits. Budget alarms in each cloud on day one.
> **Story:** NorthPay expands — GCP for a data/ML-adjacent workload + DR presence; Azure because an enterprise customer mandates it. You must make them one platform, not three silos.

---

## Part A — GCP

### EXERCISE 14.1 — GCP foundations & the resource hierarchy [B] [75 min]

**STEPS**

1. Create account (credits), create org-less project structure: folders (`northpay-prod`, `northpay-nonprod`) + projects (`np-prod-app`, `np-shared-net`). Map to AWS: Organization≈Resource Manager (folders≈OUs, projects≈accounts — but resources live in projects, and projects are cheap/numerous).
2. IAM: members/roles/bindings; basic vs predefined vs custom roles; **deny policies + policy conditions** (GCP's SCP-ish levers — compare with SCPs in your log); service accounts (≈IAM roles) + **Workload Identity Federation** (≈IRSA — GKE uses it natively).
3. gcloud CLI: `gcloud init`, `--project` discipline, `gcloud auth application-default` for SDK/Terraform; config configurations (≈AWS profiles).
4. Billing: budget + alerts (50/90/100%) to email; label/tag strategy parity with AWS.
5. Quotas: check `compute` quotas for your region (GCP quota-requests are a rite of passage; GPUs/IPs need requests).
6. API enablement model: `gcloud services enable compute.googleapis.com container.googleapis.com ...` — unlike AWS, APIs are opt-in per project (Terraform `google_project_service` resource).
**VALIDATE:** budget armed; hierarchy created; `gcloud compute instances list` works from CLI.

### EXERCISE 14.2 — GCP networking: VPC as a global fabric [I] [90 min]

**STEPS**

1. VPC is **global** in GCP (regional subnets inside) — create custom-mode VPC `np-gcp-vpc` with subnets `asia-south1` (10.30.0.0/20) + `asia-southeast1` (10.30.16.0/20). No IGW needed (global routing built-in); no NAT gateways — instead **Cloud NAT** (regional, router-attached); Cloud Router = managed BGP speaker (remember it for Part C).
2. Firewall rules = VPC-wide, tag/service-account-based (not subnet-bound SGs): allow 443 to `sa-app`, internal-any between subnets, deny-all else implied. Compare with SG/NACL in your log — one page ("GCP firewalls ≈ SG semantics at global scope; hierarchical firewall policies ≈ SCP-for-network").
3. Private Google Access (reach Google APIs without public IPs — ≈VPC endpoints concept) + Private Service Connect (≈PrivateLink; note the producer/consumer direction differences).
4. LB family map: Global External Application LB (anycast single IP, ≈CloudFront+ALB fusion), Regional NLB (≈NLB), internal LBs; serverless NEGs concept.
5. Cloud DNS: private zones + forwarding (≈Route53 PHZ + Resolver); DNS peering between VPCs.
**VALIDATE:** VM in each subnet reaches internet via Cloud NAT only; firewall tag rules proven; one-page AWS↔GCP networking map in your notes.

### EXERCISE 14.3 — GKE: the other managed Kubernetes [I] [2 hrs]

**STEPS**

1. GKE Standard cluster (Autopilot noted — mode tradeoff: Autopilot = node-less billing/less control, Standard = node pools you shape; NorthPay picks Standard for control parity with EKS) in the VPC, private nodes + private endpoint with authorized networks.
2. Node pools (≈managed node groups), Workload Identity for the worker SA → Pub/Sub access (≈IRSA → SQS).
3. VPC-native clusters (alias IPs — pods get VPC IPs like AWS VPC CNI; compare capacity math), network policies (Calico-based) — port one Module 05 NetworkPolicy verbatim (it works! CNIs differ, API doesn't — write that sentence).
4. Deploy northpay via your same Helm chart + same ArgoCD (register GKE cluster in ArgoCD — Module 10 multi-cluster pays off immediately).
5. GKE extras worth one line each: Binary Authorization (≈image-signing admission), GKE Enterprise/Fleet (multi-cluster management), Config Sync (Google's GitOps — vs ArgoCD, one paragraph).
6. Operations: release channels (≈EKS upgrade cadence but auto), node auto-upgrade/auto-repair, surge upgrades.
**VALIDATE:** northpay-api on GKE reachable via GCP LB; ArgoCD deploys to EKS and GKE from the same gitops repo (different values overlays); Workload Identity proven (deny without SA binding).

### EXERCISE 14.4 — GCP data & messaging parity [I] [90 min]

**STEPS**

1. Cloud SQL Postgres (private IP, ≈RDS): HA configuration, PITR (`gcloud sql backups` + point-in-time flags), read replicas; connect from GKE via private IP + Auth Proxy concept.
2. Firestore/Bigtable awareness (when each; vs DynamoDB mental model — document vs wide-column).
3. Pub/Sub (≈SNS+SQS fused): topic + subscription (push/pull), exactly-once option, dead-letter topics, message ordering keys; port the worker to a Pub/Sub variant (feature-flagged in code: `QUEUE_DRIVER=sqs|pubsub` — the portability seam exercise below).
4. Memorystore (≈ElastiCache); GCS (≈S3: buckets are globally named, uniform bucket-level access ≈ block-public+policy, lifecycle rules, dual-region for DR).
5. Cloud Operations (Stackdriver): dashboards + uptime checks + alert policies — compare with your Grafana stack honestly (strength: zero setup; weakness: portability/cost at scale).
**VALIDATE:** worker processes from Pub/Sub with DLQ; Cloud SQL PITR drill executed (same discipline as Module 12); GCS lifecycle + versioning set.

### EXERCISE 14.5 — The portability seam: making NorthPay cloud-portable [A] [2 hrs]

**GOAL:** the real multicloud skill — architecture that doesn't care.
**STEPS**

1. Inventory every AWS touchpoint in the app (SQS, S3, Secrets Manager, RDS). Abstract behind interfaces already present in the sample app (`QUEUE_DRIVER`, `STORAGE_DRIVER`, `SECRETS_DRIVER` env flags); implement GCP drivers (pubsub, GCS, Secret Manager).
2. Terraform: `modules/appstack-aws` vs `modules/appstack-gcp` exposing identical outputs (queue URL, bucket, db host); environment composition per cloud. The contract = outputs; implementations differ.
3. K8s layer unchanged: same chart, same ArgoCD — values differ (endpoints, SA annotations vs workload-identity bindings).
4. Observability unchanged: OTel → same LGTM (single pane!) — ship GKE cluster telemetry to your Mimir/Loki/Tempo (cross-cloud remote-write — label `cloud=gcp`).
5. Write ADR-0013: "portability strategy: containers + OTel + Terraform modules + driver seams; accepted leak points (IAM wiring, LB flavors, managed-DB feature gaps)." **Being able to say where portability *ends* is the senior answer.**
**VALIDATE:** same app image runs on EKS (SQS-backed) and GKE (PubSub-backed) switching only env config; single Grafana shows both clouds' metrics side by side.

---

## Part B — Azure (faster lap — the concepts are mapped already)

### EXERCISE 14.6 — Azure foundations [B] [60 min]

**STEPS**

1. Subscription + management groups (≈org/OUs); resource groups (≈nothing in AWS — a lifecycle/scoping box; learn it); Entra ID (tenant ≈ Identity Center directory; RBAC roles Owner/Contributor/Reader vs IAM policies; PIM = just-in-time elevation — **slice SRE-2's JIT maps perfectly, demo it**).
2. Azure Policy + initiatives (≈SCP+Config): assign "allowed locations" + "require encryption" policies; deny effect test.
3. az CLI + cloud shell; budget alert on the subscription; tags.
4. ARM/Bicep awareness: write one Bicep file (storage account) to feel it; then decide Terraform everywhere (ADR-0014 one paragraph — multicloud mandates a portable IaC).
**VALIDATE:** policy deny captured; PIM activation flow executed.

### EXERCISE 14.7 — AKS + Azure networking [I] [2 hrs]

**STEPS**

1. VNet (regional, unlike GCP global) + subnets + NSGs (≈SG+NACL hybrid); route tables (UDRs — more explicit than AWS, less than Cisco).
2. AKS private cluster, Azure CNI (pod IPs from VNet — same VPC-native idea), managed identity + **Workload Identity** (yes, same name as GCP — OIDC everywhere now), AAD/Entra RBAC integration.
3. Deploy northpay to AKS via the same ArgoCD + chart (3rd cluster registered — `cloud=azure` label).
4. Azure LB/App Gateway/Front Door mapping (Front Door ≈ CloudFront+GA fusion; App Gateway ≈ ALB with WAF built in).
5. Azure Database for PostgreSQL Flexible Server (≈RDS) + Service Bus (≈SQS) awareness + Blob Storage; implement the `azure` drivers if time allows, else design notes.
6. Monitor/Log Analytics (≈CloudWatch) + export to your LGTM via OTel collector (daemonset on AKS) — one-pane rule holds.
**VALIDATE:** app healthy on AKS via ArgoCD; metrics flowing to your Grafana with `cloud=azure`; one NSG misconfiguration debugged on purpose.

---

## Part C — Multicloud networking (the crown jewel)

### EXERCISE 14.8 — AWS ↔ GCP: HA VPN with BGP [E] [3 hrs] ← *your unfair advantage*

**STEPS**

1. GCP side: HA VPN gateway (2 interfaces), Cloud Router (ASN 64513), two tunnels to AWS TGW VPN (or VGW VPN) — AWS CGW entries = GCP gateway IPs.
2. BGP: Cloud Router ↔ AWS (ASN 64512); advertise GCP subnet ranges; receive AWS 10.10/10.20 ranges. `gcloud compute routers get-status` + AWS tunnel status. **You are now running eBGP between two hyperscalers — savor it, then document it.**
3. Verify: pod on GKE → pod on EKS via private IPs (mind the SNAT: pod→VPC IP at egress; CNI specifics per cloud — trace it hop by hop with tcpdump on a VM in each cloud).
4. Failover: kill tunnel 1, measure reconvergence (BGP timers); enable BFD for sub-second detection (Cloud Router supports BFD — configure; AWS side tunnel BFD? check current support; if unavailable, tune keepalive/hold timers and measure).
5. Route hygiene: no overlapping CIDRs across all three clouds + on-prem (your Day-1 CIDR plan pays off — show the master allocation table); summarization at boundaries; AS-path filters to prevent transit (GCP must never become transit between AWS and Azure — write the filter, explain the risk).
6. Throughput/latency baseline: iperf3 across the tunnel; note per-tunnel limits (~3 Gbps class) and when you need Dedicated/Partner Interconnect or DX (real money talk).
**VALIDATE:** BGP session outputs both sides; failover measured; cross-cloud pod-to-pod private-IP connectivity proven; as-path filter committed.

### EXERCISE 14.9 — AWS ↔ Azure VPN + the three-cloud ring [A] [90 min]

**STEPS**

1. Azure side: Virtual Network Gateway (route-based, BGP-enabled, ASN 65515), two Local Network Gateways → AWS tunnels; BGP over it. (Cost note: VNG hourly is real money — build, validate, destroy; Terraform makes rebuild trivial.)
2. Full mesh check: AWS ↔ GCP, AWS ↔ Azure, and (optionally) GCP ↔ Azure directly vs via-AWS-transit (TGW can't transit third-party... it can route to VPN attachments — design both, pick hub=AWS with spoke filtering).
3. Consolidate: master route table document — every CIDR, ASN, tunnel, BFD/timer setting in the org, committed to `runbooks/multicloud-network.md`. This document is a senior artifact.
**VALIDATE:** three-cloud reachability matrix (9 cells) all green or documented-blocked-by-design.

### EXERCISE 14.10 — Multicloud service discovery, DNS & traffic steering [A] [2 hrs]

**STEPS**

1. DNS: one public zone strategy — Cloudflare as the neutral edge (from 13.7): `api.northpay.example` → proxied to AWS ALB + GCP GLB origins; Cloudflare Load Balancer (or Route53 multivalue/GCP Cloud DNS equivalents — compare) with health checks steering traffic AWS↔GCP 50/50.
2. Active-passive variant: GCP as warm-standby DR (weights 100/0; failover on health check — measured failover time; tie into Module 15 DR runbook).
3. Split-horizon across clouds: `*.internal.northpay` resolution in all three clouds — Route53 PHZ ↔ Cloud DNS peering/forwarding ↔ Azure Private DNS + resolvers; or a simpler pattern: each cloud's internal zone + cross-cloud forwarding rules over the VPNs. Implement AWS↔GCP forwarding both ways (inbound/outbound endpoints meet Cloud DNS forwarding).
4. Data gravity: the app can fail over, the *data* can't teleport — Cloud SQL read replica cross-cloud? No — native replication stays in-cloud; cross-cloud DR = backup/restore or logical replication (pglogical/Debezium) or DMS. Write the honest assessment: **multicloud active-active databases are where architectures go to die; choose one writer region/cloud, replicate async, accept RPO.**
5. Egress cost reality: price 1 TB cross-cloud egress each direction; watch it reshape your traffic-steering decisions (this is why "multicloud active-active everything" fails CFO review — put numbers in the ADR).
**VALIDATE:** DNS-steered failover AWS→GCP executed with downtime measured; cross-cloud internal DNS working; egress-cost table in ADR-0015.

### EXERCISE 14.11 — Multicloud operations: one pane, one pipeline [A] [90 min]

**STEPS**

1. Unified observability (done if 14.5/14.7 flowed): single Grafana, `cloud` label everywhere; dashboards comparing p95 across clouds for the same service — performance parity data for steering decisions.
2. Unified CI/CD: one pipeline builds the image once → pushed to ECR + GAR (or one neutral registry — tradeoff note); ArgoCD ApplicationSet rolling the same release to all clusters with per-cloud waves (dev clouds first).
3. Unified secrets: same app secrets in SM/GCP-SM/Key Vault via a small sync pattern or ESO multi-store; consistency checks in CI.
4. Unified IAM story (the hardest): humans via Identity Center + Entra + Google identity — federation options (Entra↔Google, SSO everywhere); write the identity architecture page; audit trail aggregation (CloudTrail + GCP Audit Logs + Azure Activity Log → one S3/OpenSearch — the compliance win).
5. FinOps multicloud: CUR + GCP billing export + Azure cost export → one S3 → Athena/queries → weekly cost report by cloud/service/env. (slice Staff: "cost reporting and governance" — you now do it across three clouds.)
**VALIDATE:** one deploy PR rolls all clouds in waves; one dashboard answers "is the app healthy everywhere"; one cost report covers three clouds.

---

## Module 14 exit gate

- [ ] NorthPay runs on EKS + GKE (+ AKS validated), deployed by one ArgoCD, observed by one LGTM, built by one pipeline.
- [ ] Site-to-site eBGP between clouds with measured failover; route-policy discipline (no accidental transit); master CIDR/ASN registry.
- [ ] DNS-steered cross-cloud failover executed and timed; internal DNS spans clouds.
- [ ] Portability ADRs with honest leak-points; egress economics table; multicloud FinOps report.
- [ ] Interview weapon: "I've run BGP between AWS and GCP with HA VPN, deployed the same app to EKS and GKE from one Git repo, and here's my per-cloud cost report" — rehearse saying it plainly.

**Onward:** `15-SRE-Reliability-Engineering.md` — the discipline layer: SLOs, incidents, DR, chaos, capacity, cost.
