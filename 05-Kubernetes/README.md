# Module 05 — Kubernetes: Core to Day-2 Operations (Weeks 5–6)

> **JD coverage:** "Strong working experience with Docker and Kubernetes" (Deel); "you can debug a CrashLoopBackOff, OOMKill, failed rollout, or in-cluster DNS issue, not only deploy manifests" (slice SRE-2 — quoted verbatim because it's the module's design spec); "Own Kubernetes (EKS) platforms... upgrades, capacity planning, operational stability" (slice SRE-3).
> **Lab:** local `kind` for iteration (free, fast); EKS for the parts that need it (end of module, then continuously). Manifests: `assets/k8s/`.

---

## Part A — Architecture & mental model

### EXERCISE 5.1 — Cluster anatomy on kind [B] [60 min]

**STEPS**

1. `kind create cluster --config assets/k8s/kind-3node.yaml` (1 control-plane + 2 workers).
2. Control-plane tour: `docker exec` into the control-plane node; find static-pod manifests in `/etc/kubernetes/manifests` (apiserver, etcd, scheduler, controller-manager). Understand: K8s runs K8s via kubelet + static pods.
3. etcd: snapshot it (`etcdctl snapshot save` with the right endpoints/certs — hunt them in the manifests). Realize: **etcd is the entire cluster state; everything else is replaceable.**
4. The reconcile loop: run `kubectl get pods -w` in one pane; delete a pod from a Deployment in another; watch DesiredState → controller → new pod. Write the controller pattern in your words (observe → diff → act).
5. API tour: `kubectl api-resources | head -30`, `kubectl explain deployment.spec.strategy`, `--v=8` on a get to see raw API calls.
6. k9s daily driver: navigate pods/logs/describe without typing manifests.
**VALIDATE:** etcd snapshot taken and restored to a scratch dir (not applied — just prove you could).
**TRADEOFFS:** managed control plane (EKS: AWS runs etcd/apiserver HA, you never see them — and can't break them) vs self-managed. Fintech reality: EKS/GKE everywhere; know self-managed internals anyway for debugging depth.

### EXERCISE 5.2 — Pods: the atomic unit [B] [60 min]

**STEPS**

1. Raw pod from `assets/k8s/01-pod.yaml`: api container. Apply, exec, logs, port-forward, delete.
2. Multi-container pod: add a `log-shipper` sidecar sharing an `emptyDir` volume. Understand: shared netns (localhost between containers) + shared volumes, isolated filesystems. **Sidecar pattern = the Istio/telemetry foundation (Modules 11, 16).**
3. Lifecycle: `postStart`/`preStop` hooks; termination grace; why your app's SIGTERM handler (Module 04) matters now: K8s sends TERM → waits grace → KILL.
4. Probes (do it wrong first): no probes + broken DB → traffic hits dead pods. Then add `readinessProbe` (/readyz) vs `livenessProbe` (/healthz) vs `startupProbe` (slow-starting apps). **Kill rule: liveness failing = restart; readiness failing = remove from Service endpoints.** Misconfigured liveness = crashloop storms; prove it with an aggressive probe.
5. Init containers: `wait-for-db` init that blocks until pg_isready succeeds — ordering without depends_on.
6. Static pods + DaemonSets concept; node placement preview: nodeSelector, taints/tolerations on kind workers.
**VALIDATE:** pod with broken DB never receives traffic (endpoints object proves it: `kubectl get endpoints`); slow-start app survives via startupProbe.
**TRADEOFFS:** probe tuning is SLO work in disguise: too lax = users hit errors; too strict = cascading restarts under load. This framing reappears in Module 15 error budgets.

### EXERCISE 5.3 — ReplicaSets, Deployments, rollout mechanics [B] [60 min]

**STEPS**

1. Deployment `api` (3 replicas, image :0.1.0). `kubectl rollout history/status/pause/resume`.
2. Update to :0.2.0 with `strategy: rollingUpdate maxSurge=1 maxUnavailable=0` — watch in slow motion (`--record`, `rollout status -w`). Why maxUnavailable=0 for prod?
3. Rollback: `rollout undo` to previous; understand ReplicaSet retention (`revisionHistoryLimit`).
4. **Failed rollout drill:** deploy :0.3.0-broken (image tag typo → ImagePullBackOff). Rollout stalls; old pods stay (that's the design!); `rollout undo`. Then a subtler one: image exists but app crashes → `progressDeadlineSeconds` exceeded → `Progressing=False` condition. Read it with `kubectl describe`.
5. Recreate strategy: when acceptable (dev, stateful singletons) vs never (prod payments).
6. HPA preview: scale to 6 by hand (`kubectl scale`), then delete HPA-worthy scaling decisions in your log: "scale on what metric, to what max, why" — Module 05 Part E does it for real.
**VALIDATE:** failed rollouts detected via status conditions (script it: `kubectl rollout status --timeout=60s || kubectl rollout undo` — this one-liner is a CI/CD deploy gate, Module 09 uses it).

### EXERCISE 5.4 — Services & cluster networking (network-engineer home game) [I] [90 min]

**STEPS**

1. ClusterIP: `svc/api` round-robins across pods (prove with alternating pod names in responses). Inspect: kube-proxy iptables rules on a kind node (`docker exec <node> iptables -t nat -L KUBE-SVC-XXX -n`).
2. DNS: `kubernetes.default`, `api.default.svc.cluster.local` FQDN anatomy; `nslookup` from a debug pod; `ndots:5` — understand why `curl api` triggers 5 search-domain lookups before resolving (performance + NXDOMAIN surprises; fix with trailing dot or FQDN).
3. NodePort vs LoadBalancer (kind: use extraPortMappings); ExternalName (CNAME-like) and when it lies.
4. Headless service (`clusterIP: None`): DNS returns pod IPs directly — needed for StatefulSet member discovery (Module 12 Mongo/PG on k8s) and client-side LB.
5. sessionAffinity: ClientIP — when you need stickiness (websockets, non-shared sessions) and why to avoid it (stateless design beats sticky design).
6. **Network debugging drill:** from pod A: `curl svc-b` fails. Systematic ladder: pod→pod direct IP? service IP? DNS? endpoints populated? NetworkPolicy? Write the ladder as runbook `k8s-svc-debug.md`. (Tools: `kubectl exec`, ephemeral debug container `kubectl debug -it --image=nicolaka/netshoot`.)
7. kube-proxy modes: iptables (default) vs IPVS — switch kind to IPVS, diff the rules, recall Module 02's conntrack lesson; note IPVS for >1k services.
**VALIDATE:** the debug ladder runbook exists and was used to solve a planted failure (friend/script breaks endpoints by label mismatch).
**TRADEOFFS:** As the network engineer, write a page comparing K8s networking to what you know: Service VIP ≈ SLB VIP; kube-proxy iptables ≈ stateless NAT rules; CNI overlay ≈ VXLAN EVPN; NetworkPolicy ≈ distributed ACLs. This page will make your interviews unfair (in your favor).

### EXERCISE 5.5 — Ingress & L7 traffic management [I] [75 min]

**STEPS**

1. Install ingress-nginx on kind (`assets/k8s/ingress/kind-patch.yaml` + helm). Ingress rules: `web.northpay.local` → web svc; `api.northpay.local` → api svc; path-based `/api/*` variant. `/etc/hosts` entries; test.
2. TLS at ingress: self-signed via openssl now (cert-manager in Part F); redirect annotations; rate-limit annotation demo (429s under gen_load).
3. Read the ingress controller's generated nginx.conf (`kubectl exec` into controller pod) — as someone who knows proxies, map Ingress resources → nginx directives. This demystifies "magic."
4. ALB comparison (EKS later): AWS Load Balancer Controller turns Ingress → ALB with target groups per service (ip mode vs instance mode — ip mode = pods registered directly, fewer hops, needs CNI support; you'll choose ip mode and justify).
5. Multiple ingress controllers: ingressClass field; why teams run internal + external controllers separately.
**VALIDATE:** host + path routing working; rate limiting observed; generated config inspected.
**TRADEOFFS:** ingress-nginx vs ALB controller vs Traefik vs Gateway API. Know Gateway API exists as the successor (role-oriented: infra owns Gateway, devs own HTTPRoutes) — one paragraph in your ADR; Module 16 revisits with Istio.

### EXERCISE 5.6 — ConfigMaps & Secrets [B] [45 min]

**STEPS**

1. ConfigMap from file + envFrom into api; update CM → pods DON'T reload (prove it) → rollout restart needed, or use `Reloader`/checksum-annotation trick (Helm does checksums — Module 06).
2. Secrets: base64 ≠ encryption. Prove: `kubectl get secret -o jsonpath` decodes it. → Therefore: RBAC on secrets, etcd encryption-at-rest (EKS: KMS envelope encryption — enable in Part G), external secrets.
3. External Secrets Operator pattern (concept + install on EKS later): secrets live in AWS Secrets Manager, synced to K8s secrets. Why: one secret store, rotation without redeploys, audit in one place. Fintech-grade answer.
4. Immutability: `immutable: true` on versioned CMs — safety + kubelet load reduction at scale.
**VALIDATE:** secret rotated in AWS SM → synced → app picks up on restart; you can explain why plaintext etcd = breach.

### EXERCISE 5.7 — StatefulSets & storage [I] [60 min]

**STEPS**

1. kind + local-path provisioner (default in kind). PVC → PV lifecycle; `Retain` vs `Delete` reclaim policies (delete a PVC by accident on Retain → data survives; learn the rescue).
2. StatefulSet postgres (1 replica): stable name `pg-0`, headless svc, volumeClaimTemplates. Delete pod → same name+volume reattach. Scale to understand ordinals.
3. Compare with Deployment+PVC: why Deployments can't safely share RWO volumes across nodes (attach errors — reproduce the Multi-Attach error by forcing reschedule to another node).
4. Snapshots: VolumeSnapshot CRDs concept; on EKS later, EBS CSI snapshots.
5. The big tradeoff table (write it): DB in K8s (operators, control, cost) vs RDS/Aurora (managed backups/patching/HA). **NorthPay decision: databases on RDS/Aurora/DocumentDB; K8s runs stateless + queue workers + ephemeral state (redis).** Most fintech JDs agree; SRE-3 interviews probe whether you can argue both sides.
**VALIDATE:** pod deletion + rescheduling preserves data; Multi-Attach error reproduced and resolved via node drain + force-delete (documented!).

---

## Part B — Scheduling, resources, autoscaling

### EXERCISE 5.8 — Requests/limits & QoS: the OOMKill mastery lab [I] [90 min]

**STEPS**

1. QoS classes: Guaranteed (req==lim), Burstable, BestEffort — assign deliberately; understand eviction priority under node pressure.
2. **OOMKill lab:** api with `limits.memory=150Mi`, hit `/api/leak`; watch `OOMKilled` reason, exit 137, `kubectl get events`, node dmesg. Then requests-only (no limit): node memory pressure → kubelet eviction order → BestEffort dies first. Write the kernel-vs-kubelet OOM distinction (cgroup OOM = container dies; node pressure = eviction by QoS+usage).
3. CPU throttling: limits.cpu=100m + load → `container_cpu_cfs_throttled_periods_total` (Module 11 graphs it); understand why many teams set requests-only for CPU (limiting CPU = latency jitter) but always limit memory. **Current best practice debate — know both sides.**
4. VPA (vertical pod autoscaler) on kind: recommendations mode for right-sizing requests from real usage.
5. ResourceQuotas + LimitRanges per namespace: protect the cluster from one team's greed; defaults injected.
**VALIDATE:** you predicted eviction order correctly; throttling evidence captured; right-sized api requests from VPA recs.

### EXERCISE 5.9 — HPA, KEDA, and cluster autoscaling [I] [90 min]

**STEPS**

1. metrics-server on kind; HPA v2 on api: cpu 60%, min 2 max 8. Load-test → watch scale events; tune `behavior` (scale-down stabilization 300s — why? flapping).
2. Custom metric HPA: prometheus-adapter on a /metrics rate (`http_requests_per_second`) — scale on traffic, not CPU. When is CPU a *bad* scaling signal? (I/O-bound, queue-backlog workloads.)
3. KEDA: ScaledObject on SQS queue depth (points at your real AWS queue — kind → AWS works with IAM keys for lab; IRSA is the EKS answer) — scale worker 0→10 on backlog, 0 when empty (cost!). **This pattern (scale-to-zero workers) is a fintech cost interview favorite.**
4. Cluster autoscaling: kind can't; note for EKS part (Karpenter vs cluster-autoscaler — Part G).
**VALIDATE:** queue-driven scaling proven end-to-end with timestamps: messages in → pods out → backlog drained → pods back to 0.

### EXERCISE 5.10 — Placement: affinity, taints, spread [A] [60 min]

**STEPS**

1. `topologySpreadConstraints` across AZs + hostnames — prove even spread; why `DoNotSchedule` vs `ScheduleAnyway` when maxSkew violated.
2. podAntiAffinity: api replicas never share a node (required) — then try to schedule 3 replicas on 2 nodes: Pending pods. Read the scheduler events. (Interview classic: "pods pending, cluster has capacity — why?")
3. Taints on a kind worker (`dedicated=batch:NoSchedule`) + toleration on worker deployment; nodeSelector vs nodeAffinity soft/hard.
4. PDBs (PodDisruptionBudgets): maxUnavailable=1 on api; try `kubectl drain` → drain respects PDB and stalls on the 2nd pod; voluntary vs involuntary disruptions; PDBs are why node upgrades don't outage you (Part G upgrade exercise uses this).
5. PriorityClasses: `system-critical` for observability agents — preemption behavior under resource crunch.
**VALIDATE:** drain blocked by PDB until capacity added; pending-pod diagnosis written into runbook.

---

## Part C — Security & multi-tenancy

### EXERCISE 5.11 — RBAC from zero [I] [75 min]

**STEPS**

1. kind cluster, create ServiceAccount `api-sa`; Role allowing get/list on configmaps in `default` only; RoleBinding. Token → kubeconfig → `kubectl --kubeconfig=sa.conf get secrets` (denied) vs `get configmaps` (allowed). Capture the 403 anatomy (`User "system:serviceaccount:..." cannot ...`).
2. ClusterRole vs Role; aggregation; `admin/edit/view` defaults.
3. The developer persona: namespace-scoped `edit` for team-alpha in ns `alpha`, nothing elsewhere. Impersonation testing: `kubectl auth can-i --as=system:serviceaccount:alpha:dev-sa delete pods -n prod` → no.
4. Break-glass: emergency ClusterRole + a documented, audited process (audit log query for its use — preview of CIS/compliance, Module 13).
5. EKS reality preview: aws-auth ConfigMap / access entries mapping IAM↔K8s RBAC (Part G).
**VALIDATE:** can-i matrix committed; break-glass usage visible in API audit logs (kind audit policy config).

### EXERCISE 5.12 — Pod security & policy enforcement [I] [60 min]

**STEPS**

1. Pod Security Standards: label ns `restricted` — try privileged pod (rejected), hostPath (rejected), runAsRoot (rejected). Levels: privileged/baseline/restricted; enforce vs audit vs warn rollout strategy.
2. securityContext deep: runAsNonRoot, readOnlyRootFilesystem (+ emptyDir /tmp), drop ALL caps, seccomp RuntimeDefault. Apply to api — fix the breakage (app writing tmp) properly.
3. NetworkPolicy: default-deny ingress+egress in ns; allow api←web, api→postgres:5432, api→kube-dns:53, worker→internet:443 only. Test each allowed/denied path with netshoot pods. **Your firewall skills transfer 1:1 — write policies like ACLs with a default-deny stance.**
4. Admission control with Kyverno (install): policies — require non-root, require resources.requests, disallow `:latest`, require team label. Audit mode → report → enforce. (OPA/Gatekeeper alternative — know the name, Kyverno's YAML-friendliness is why we use it.)
**VALIDATE:** policy violation blocked with clear message; NetworkPolicy matrix proven both directions.
**TRADEOFFS:** policy-as-code at admission vs pipeline-time scanning (Module 09) — answer: both (defense in depth); admission is the last gate, CI is the fast feedback.

### EXERCISE 5.13 — The day-2 debug gauntlet [A] [3–4 hrs] ← *the slice SRE-2 JD, literally*

Use `assets/k8s/broken/` — 10 scenarios, each a separate namespace. Rules: no reading the YAML answer keys first; for each, write symptom → hypothesis → evidence → fix → prevention (that's a postmortem).

1. **CrashLoopBackOff** (bad env var → app exit 1)
2. **OOMKilled loop** (limit too low on memory-hungry batch job)
3. **ImagePullBackOff** (wrong tag; then private-registry missing imagePullSecret variant)
4. **Pending pods** (unschedulable: resource request too big; PVC unbound variant)
5. **Failed rollout** (progressDeadline exceeded; liveness probe kills new pods mid-rollout)
6. **In-cluster DNS failure** (CoreDNS scaled to 0 → everything NXDOMAIN; then ndots/search-domain latency case)
7. **Service has no endpoints** (label selector mismatch)
8. **NetworkPolicy blackhole** (fresh default-deny forgot DNS egress)
9. **Node NotReady** (kind: stop kubelet on a worker via docker exec — watch pod eviction, node leases)
10. **Certificate expiry** (ingress TLS secret expired → browser/curl failure; detect via blackbox probe)
**VALIDATE:** 10 postmortems; then re-run the gauntlet timed (< 10 min per scenario = interview-ready).
**TRADEOFFS/WHY:** this gauntlet *is* the hands-on portion of SRE-2 interviews. Speed comes from the ladder: object status → events → logs → node → network → control plane. Never skip to kubectl delete.

---

## Part D — cert-manager, jobs, operators (production peripherals)

### EXERCISE 5.14 — cert-manager: TLS as a K8s native [B] [45 min]

**STEPS**

1. Install cert-manager (helm). SelfSigned ClusterIssuer → internal certs for services.
2. Let's Encrypt staging issuer (HTTP01 via ingress) for `web.northpay.<yourdomain>` when on EKS; local kind: use self-signed + understand the flow (Order → Challenge → Certificate → Secret).
3. Auto-renewal proof: issue a 1h-duration cert, watch renewal in events.
4. Debug a stuck Order (challenge pending — wrong ingress class): `kubectl get challenges`, `cert-manager logs`. Add to gauntlet runbook.
**VALIDATE:** renewed cert without intervention; stuck-challenge debug documented.

### EXERCISE 5.15 — Jobs, CronJobs, and batch patterns [B] [45 min]

**STEPS**

1. Job: one-off DB migration (`assets/k8s/jobs/migrate.yaml` — runs the seed SQL). backoffLimit, activeDeadlineSeconds, ttlSecondsAfterFinished.
2. CronJob `settlement-report` nightly: concurrencyPolicy Forbid (why — overlapping settlements = double payments; the fintech answer), startingDeadlineSeconds (missed window behavior).
3. Parallelism/completions modes: indexed job for fan-out batch (preview of queue-based autoscaling from 5.9).
4. Failure drill: make the cronjob fail 3 nights; alert design: "cronjob missed its schedule" — dead man's switch pattern (Module 11 implements with Prometheus).
**VALIDATE:** missed-schedule detection logic written down; Forbid demonstrated.

### EXERCISE 5.16 — Operators & CRDs: extending K8s [I] [45 min]

**STEPS**

1. CRD anatomy: install a simple operator (e.g., postgres-operator or mongodb community operator on kind) — watch CRs become StatefulSets+Services+Secrets.
2. `kubectl get crd`, describe the controller's reconcile via logs.
3. When operators vs Helm (Module 06) vs managed services: operator = day-2 automation baked in (backups, failover), at the cost of trusting someone's controller with prod data. NorthPay: operators for dev-cluster datastores; prod data on RDS/DocumentDB (ADR-0002).
**VALIDATE:** ADR-0002 committed with the decision matrix.

---

## Part E — EKS: the real thing

### EXERCISE 5.17 — Provision EKS (console once, then never again) [I] [2 hrs] [COST: ~$3–5 if torn down same day]

**STEPS**

1. eksctl cluster create: `np-eks-dev`, k8s 1.31+, managed nodegroup `ng-general` (2× t3.medium, private-app subnets of your Module 03 VPC), OIDC provider enabled (critical for IRSA next).
2. aws-auth/access entries: your SSO admin → cluster-admin; create a read-only IAM role → view-only group; test both kubeconfigs.
3. VPC CNI deep-dive (networking gold): each pod gets a real VPC IP (that's why subnet sizing matters — /20 app subnets, remember?). `kubectl get pods -o wide` ↔ AWS console ENIs. Prefix delegation (ENABLE_PREFIX_DELEGATION) → more pods per node; calculate max pods before/after.
4. CoreDNS on EKS: managed addon; scale it; check its PDB.
5. Storage: EBS CSI driver addon (IRSA role); gp3 StorageClass (encrypted, WaitForFirstConsumer — why: volume follows pod's AZ).
6. AWS Load Balancer Controller (helm + IRSA): Ingress → ALB, Service type LoadBalancer → NLB. Expose api via ALB with ACM cert + WAF later (Module 13).
7. Cluster autoscaler → then **Karpenter** (install both, compare): consolidation, spot handling, interruption queue (SQS + EventBridge — your Module 03 services return). Karpenter is the 2026 default answer; know CA for legacy fleets.
8. Control plane logs → CloudWatch (api, audit, authenticator); audit log query for a forbidden action.
**VALIDATE:** pod with VPC IP reachable from an EC2 in the same VPC (no NAT games — routed fabric!); Karpenter provisions a spot node under load and consolidates when idle (capture the events).
**TRADEOFFS:** EKS vs ECS vs self-managed k8s on EC2 — one page: operational surface, ecosystem (operators/CRDs), portability, cost (EKS $0.10/hr control plane). Also EKS Auto Mode (know it exists; tradeoffs: less control, faster start).

### EXERCISE 5.18 — IRSA: pod-level AWS permissions [I] [60 min]

**STEPS**

1. Problem first: worker needs SQS. Bad way — node role with SQS (every pod on the node inherits it; prove it with a test pod reading the queue). 
2. IRSA right way: IAM role `np-worker-sqs` with trust on OIDC provider + serviceaccount `default:worker-sa`; annotate SA; new pod → env vars + projected token; only worker pods can read the queue.
3. Trace the JWT: `cat /var/run/secrets/eks.amazonaws.com/serviceaccount/token`, decode payload (aud, sub). Understand sts:AssumeRoleWithWebIdentity.
4. Scope it tighter: condition on queue ARN; deny everything else; Access Analyzer validation.
**VALIDATE:** two pods, one SA each: allowed vs denied captured. Node role stripped of SQS afterward (least privilege achieved).

### EXERCISE 5.19 — EKS upgrade drill (the "operational stability" JD line) [A] [2 hrs]

**STEPS**

1. Current cluster at N-1 minor version (recreate if needed). Read the EKS upgrade calendar + k8s deprecation notes (e.g., removed APIs — `kubectl convert`/pluto scan your manifests for deprecated API versions; run pluto across assets/k8s).
2. Upgrade control plane via console; observe: workloads untouched, API available throughout.
3. Upgrade addons (coredns, kube-proxy, vpc-cni, ebs-csi) with compat matrix checks.
4. Upgrade nodegroup: managed nodegroup rolling update respects PDBs (your 5.10 PDB proves its worth — capture drain events); compare with Karpenter node expiry/drift.
5. Failure injection: upgrade with PDB maxUnavailable=0 and 1-node capacity → upgrade stalls; interpret events; fix capacity; complete. **This stall is a real on-call page; write the runbook.**
6. Rollback reality: you can't downgrade control plane — prevention = staging-first upgrades + deprecation scanning + version support policy. Write NorthPay's upgrade policy (N-1 support, quarterly cadence).
**VALIDATE:** full N-1→N upgrade with zero app downtime measured via continuous `np-deploy-check`; stall runbook committed.

### EXERCISE 5.20 — EKS troubleshooting at AWS depth [A] [90 min]

**STEPS**

1. CNI IP exhaustion: shrink a nodegroup's pod subnet mentally — simulate with small subnets in a throwaway VPC: pods Pending with `Insufficient cilium/aws-ip` style events; fix via prefix delegation or secondary CIDRs (100.64.0.0/10 CGNAT range — VPC secondary CIDR feature; explain why CGNAT space).
2. Node join failures: instance can't reach API (private endpoint + missing NAT), wrong cluster SG, IAM instance profile missing — reproduce one, read node logs via SSM (`/var/log/messages`, `aws ec2 get-console-output`).
3. ALB controller silent failure: target group created but targets unhealthy — SG missing rule from node SG to pod IPs (ip mode). Security-group-per-pod concept (when pod-level SGs are worth it: RDS access from specific pods only).
4. etcd/control-plane pressure symptoms via CW: API latency, throttling (`apiserver_request_total` 429s) — request-size/count limits, when to contact AWS support.
**VALIDATE:** each failure reproduced with evidence + fix + detection (which metric/log catches it earliest).

---

## Module 05 exit gate

- [ ] Full app (web/api/worker + redis) running on kind via raw manifests, probes/resources/PDBs/NetworkPolicies set; then on EKS with ALB+ACM+IRSA+Karpenter.
- [ ] 10-scenario debug gauntlet done twice (second pass timed).
- [ ] Upgrade drill executed with zero downtime + stall runbook.
- [ ] You can draw: request path internet→ALB→node→pod (with SG/CNI hops), and control-plane reconcile loop, from memory.

**Onward:** `06-Helm-and-Kustomize.md` (package it), then `07-Terraform-IaC.md` (never click again).
