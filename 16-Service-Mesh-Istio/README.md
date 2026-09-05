# Module 16 — Service Mesh with Istio (Week 23)

> **JD coverage:** "Implement and operate service mesh (Istio) for traffic control, security policies, and service-level observability" (slice SRE-3); "Prior exposure to service mesh architectures (Istio or similar)" — the only mesh named across all six JDs.
> **Lab:** kind (k3d if RAM tight) or EKS dev. Principle: adopt a mesh when its *specific* features solve real problems — mTLS everywhere, fine-grained traffic control, L7 observability — not because it exists. You'll quantify that value (and its cost) yourself.

---

## Part A — Mesh fundamentals

### EXERCISE 16.1 — Istio install & data-plane anatomy [B] [75 min]

**STEPS**

1. Install via `istioctl install --set profile=default` (then contrast: Helm install, and the modern **ambient mode** — install both profiles in separate clusters eventually; ambient = no sidecars, node-level ztunnel + waypoint proxies; know why it emerged: sidecar resource tax + app-team friction).
2. Sidecar injection: label ns `istio-injection=enabled`, rollout-restart the northpay stack; `istioctl analyze`; find the new `istio-proxy` containers (your Module 04 multi-container pod lesson, weaponized).
3. Anatomy deep-dive: exec into a sidecar; `istioctl proxy-config clusters/listeners/routes <pod>` — read how iptables (init container) redirects traffic through Envoy (`sudo iptables -t nat -L` in the pod netns — Module 02 skills again).
4. Control plane: istiod (pilot+citadel+galley merged); how config propagates (xDS); `istioctl proxy-status` for sync state debugging.
5. Overhead measurement: p50/p95 latency and memory before/after sidecars under gen_load (your data, not blog numbers — expect ~2–5ms + ~50MB/proxy; record it).
**VALIDATE:** proxy-config reading demonstrated; overhead numbers logged; ambient-vs-sidecar one-pager written.

### EXERCISE 16.2 — Traffic management: VirtualService & DestinationRule [I] [2 hrs]

**STEPS**

1. Baseline: Gateway (northpay-gateway, ingress traffic — replaces/integrates with ingress; choose Gateway + ALB-in-front design, justify) + VirtualService routing `api.northpay...` → api service.
2. Subsets & weighted routing: deploy `api-v2` (labeled version=v2); VirtualService weights 90/10; watch split via response header; shift 50/50; back. (You did this with Rollouts in 10.7 — now understand mesh-native vs rollout-controller approaches; they compose: Rollouts can *drive* Istio weights — assets show the integration.)
3. Header-based canary: `x-beta-tester: true` → v2 only. Test with curl; then a real beta-tester story (internal employees dogfood first).
4. DestinationRule knobs that save prod:

- **connection pooling** (max connections/pending — protect the DB tier),
- **outlierDetection** (eject a pod erroring 5xxs for 30s — Envoy-level health vs k8s probes; faster, traffic-aware),
- **loadBalancer** algorithms (ROUND_ROBIN vs LEAST_REQUEST — test under skew),
- TLS mode (simple/mutual — Part B).

5. Resilience knobs: `retries` (attempts: 2, perTryTimeout, retryOn: 5xx — **retry amplification warning**: mesh retries × app retries × client retries = multiplicative storm; set the org rule: retries at ONE layer only), `fault injection` (delay 5s@10% and abort 503@5% — chaos without Chaos Mesh!), `timeout` budgets per route.
6. Mirroring (shadow traffic): mirror 10% of prod traffic to v3-dark; observe without serving — the "test with real traffic, zero risk" pattern.
**VALIDATE:** weighted + header canary demonstrated; outlier ejection observed under injected errors; fault-injection latency visible on dashboards; retry-policy rule committed.

### EXERCISE 16.3 — mTLS & security policies [I] [2 hrs]

**STEPS**

1. Enable mTLS: `PeerAuthentication` PERMISSIVE first (observe both plaintext+TLS accepted), then STRICT namespace-wide, then mesh-wide. Watch for breakage (non-meshed callers — redis from outside? health probes from kubelet — istio handles probe rewrite; know the edge cases).
2. Verify: `istioctl x authz check`, Kiali/cert inspection; the certs are SPIFFE identities (`spiffe://cluster.local/ns/prod/sa/api-sa`) — workload identity without app code (fintech compliance gold: encryption-in-transit-everywhere, with identity).
3. AuthorizationPolicy (L4/L7 firewall in-cluster):

- default-deny in prod ns;
- allow web→api on /api/* GET/POST only;
- allow api→worker only from `serviceAccount: api-sa` (identity-based, not IP-based — IPs lie in k8s; identity doesn't);
- deny-all + explicit list = your NetworkPolicy at L7 (write the comparison: NetworkPolicy L3/4 vs AuthorizationPolicy L7 with identity — you run both).

4. JWT end-user auth at the mesh: RequestAuthentication with a test JWKS (self-signed keys) — reject unsigned calls at the proxy, before app code (defense layering).
5. Egress control: ServiceEntry for external deps (AWS endpoints), egress gateway option, `REGISTRY_ONLY` outbound policy — data-exfil guardrail.
6. Audit: policy changes through GitOps only (AuthorizationPolicies in your gitops repo; ArgoCD selfHeal reverting hand-edits — compliance story complete).
**VALIDATE:** STRICT mTLS mesh-wide with zero app changes; unauthorized service-to-service call blocked with identity evidence; audit trail via Git.

### EXERCISE 16.4 — Mesh observability [I] [90 min]

**STEPS**

1. Automatic telemetry: sidecars emit RED metrics per workload (no app instrumentation!) — `istio_requests_total{source_workload, destination_workload, response_code}` — the service map from data. Dashboard: per-edge latency/errors.
2. Kiali: topology graph, config validation, traffic animation under load; find the misconfigured route via Kiali once.
3. Tracing integration: Istio propagates/generates spans → your OTel collector → Tempo (Module 11 stack absorbs mesh spans; sampling config in mesh config).
4. Access logs from Envoy → Loki (JSON formatted with trace_id — the correlation triangle closes at L7).
5. SLOs at the mesh layer: availability SLI from `istio_requests_total` works *without app metrics* — SLOs for third-party/legacy services you can't instrument (write this insight; it's a Staff-level pattern).
**VALIDATE:** per-edge RED dashboard; mesh spans visible in Tempo traces; one SLO defined purely on mesh metrics.

---

## Part B — Production engineering with the mesh

### EXERCISE 16.5 — Upgrades, canary control-plane, and failure modes [A] [2 hrs]

**STEPS**

1. Control-plane upgrade: canary revision install (`istioctl install --set revision=1-22` alongside), migrate ns labels gradually, verify, remove old — **never in-place for prod** (write why: data-plane restarts per revision move = controlled).
2. Data-plane restart coordination: rollout restart waves; PDB interplay; Karpenter/consolidation with sidecars (memory accounting changes — recalc requests).
3. Failure drills:

- istiod down: data plane keeps serving (config frozen) — prove it; what breaks (no new config, no cert rotation → time limit).
- Sidecar crash loop in one pod: symptoms (pod unready? traffic?), debug via envoy admin interface (`localhost:15000/stats`, `/config_dump`).
- Config push storm: 500 VirtualServices churn — control-plane CPU; `pilot_push_rejects` metrics.

4. Multi-cluster mesh (design doc + optional build): primary-remote vs multi-primary on EKS+GKE from Module 14 — east-west gateways, flat-pod-network requirement vs gateway-based, trust federation. This is the mesh answer to multicloud service discovery (compare with your DNS-based 14.10 approach — honest tradeoffs).
5. Cost accounting: sidecar RAM/CPU × fleet, istiod sizing, xDS churn; vs ambient mode savings (revisit your 16.1 numbers; write the "when ambient" memo).
**VALIDATE:** revision-canary upgrade executed; istiod-down behavior documented; multi-cluster design doc with the DNS-vs-mesh comparison.

### EXERCISE 16.6 — The "should we adopt a mesh?" decision exercise [E] [60 min]

**STEPS**

1. Write NorthPay's mesh ADR covering: problems solved (mTLS compliance, L7 traffic control, uniform telemetry) vs costs (ops complexity, latency tax, debugging surface, expertise) vs alternatives (NetworkPolicy+PSP-ish, ingress-level canary, OTel-only).
2. Decision framework (generalize): team size < 10 or < 20 services → probably skip; compliance-mandated encryption/identity → strong pull; heavy multi-cluster → mesh or careful DNS design; platform team exists → mesh affordable.
3. Alternatives survey one-pager: Linkerd (lighter, Rust proxy), Consul Connect (VM-friendly), Cilium mesh (sidecarless eBPF), AWS VPC Lattice (managed service-mesh-ish — know it, it's the "no-ops" answer).
4. Present (record yourself, 10 min): the ADR as if to eng leadership. Staff interviews are this meeting.
**VALIDATE:** ADR + framework + recorded presentation in your portfolio folder.

---

## Module 16 exit gate

- [ ] STRICT mTLS + L7 authorization policies across prod ns, all via GitOps.
- [ ] Weighted + header canaries, outlier ejection, fault injection, mirroring — all demonstrated under load.
- [ ] Mesh telemetry absorbed into your single Grafana/Tempo/Loki stack; mesh-based SLO for a black-box service.
- [ ] Revision-canary control-plane upgrade done; failure modes documented.
- [ ] The mesh decision ADR + presentation recorded.

**Onward:** `17-Daily-Job-Simulation-90-Days.md` is already running in parallel; now `18-SRE-Interview-Prep.md` — convert all of this into offers.
