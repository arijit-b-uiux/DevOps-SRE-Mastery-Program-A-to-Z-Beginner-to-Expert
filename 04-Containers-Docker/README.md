# Module 04 — Containers & Docker (Week 5)

> **JD coverage:** "Strong working experience with Docker and Kubernetes" (Deel). Docker is where your Module 02 Linux knowledge crystallizes — containers *are* namespaces + cgroups + layered filesystems, and you'll prove that to yourself before touching a Dockerfile. Interviewers can smell the difference between "writes Dockerfiles" and "understands containers."

---

## Part A — What a container actually is

### EXERCISE 4.1 — Build a container by hand (no Docker) [A] [90 min]

**GOAL:** construct an isolated "container" using only Linux primitives.
**STEPS**

1. Rootfs: `debootstrap stable /tmp/myroot http://deb.debian.org/debian` (or export alpine via `docker export` once). This is your "image."
2. Isolate mounts + chroot: `sudo unshare --mount --uts --ipc --net --pid --fork --mount-proc chroot /tmp/myroot /bin/bash`.
3. Inside: `hostname container1`, `mount -t proc proc /proc`, `ps aux` (PID 1 is your shell — feel it), `ip a` (empty netns).
4. Resource limits with cgroups v2: create `/sys/fs/cgroup/np-demo`, set `memory.max=64M`, add your shell's PID; then run a memory hog inside and watch it OOM at 64 MB — **this is exactly what K8s `resources.limits.memory` does.**
5. Networking: attach the netns to your Module 02 bridge/veth setup — give the "container" an IP, reach the app.
6. Overlay filesystem: `mount -t overlay overlay -o lowerdir=base,upperdir=changes,workdir=work merged/` — write a file in `merged`, see it land in `changes`. That's image layers.
**VALIDATE:** write the mapping table in your log: namespace → docker flag → k8s equivalent (e.g., pid ns → container; memory cgroup → limits.memory; overlay → image layers).
**WHY-WHEN-WHAT:** When a pod is "stuck," an expert asks: is it the cgroup limit (OOM), the namespace (network unreachable), or the overlay (image pull)? Juniors restart and pray. This exercise is the difference.

### EXERCISE 4.2 — Docker internals & storage drivers [I] [45 min]

**STEPS**

1. `docker info` — read every line; identify storage driver (overlay2), cgroup version, logging driver.
2. `docker run --rm -it alpine sh` then from host: find its namespaces (`lsns`, `readlink /proc/<pid>/ns/*`), its cgroup, its overlay mount (`mount | grep overlay`).
3. `docker inspect` an image: see Layers array; find them under `/var/lib/docker/overlay2`. Change one file, `docker diff` the container.
4. Copy-on-write cost demo: time writing 1 GB inside a container vs in a mounted volume — feel why volumes exist for data.
5. Prune discipline: `docker system df`, `docker system prune -a`, build cache mechanics.
**VALIDATE:** you can point at the on-disk layer for a given image layer hash.

---

## Part B — Dockerfiles done right

### EXERCISE 4.3 — Anatomy: layers, caching, build context [B] [45 min]

**STEPS**

1. Build `assets/app/api/Dockerfile.naive` (provided — deliberately bad). Note image size (~1 GB), build time, layer count.
2. `.dockerignore` experiment: without it, `node_modules` and `.git` bloat the context; add one, watch context size drop (`docker build` first line).
3. Cache invalidation: change one source line, rebuild — which layers rebuilt? Reorder Dockerfile (package.json+install BEFORE copying source) — now rebuilds are seconds.
4. `docker history` reading; `dive` tool (install it) for per-layer waste analysis.
**VALIDATE:** rebuild time before/after recorded; image diff understood via dive.

### EXERCISE 4.4 — Multi-stage builds & minimal bases [I] [60 min]

**STEPS**

1. Rewrite api Dockerfile multi-stage: `node:20-bookworm` build stage (npm ci, tests) → `node:20-bookworm-slim` runtime with only `dist/` + prod node_modules. Measure size drop.
2. Go further: `distroless` base (`gcr.io/distroless/nodejs20-debian12`) — no shell, no package manager. Note debug tradeoff (no `docker exec sh`! solution: ephemeral debug containers, Module 05).
3. Alpine vs Debian-slim vs distroless: build all three, compare size/CVE count (trivy scan, next exercise)/musl-vs-glibc quirks (DNS resolution differences, `node` native modules). **musl DNS quirks are a real K8s incident class.**
4. Non-root: `USER node`, read-only rootfs compatible? (app must not write outside /tmp — fix it), drop capabilities at runtime (`--cap-drop ALL`).
5. Buildkit: `DOCKER_BUILDKIT=1`, `--mount=type=cache` for npm cache, secret mounts for private npm tokens (`--mount=type=secret`) — never `ARG` for secrets (they persist in history; prove it with `docker history`).
**VALIDATE:** final image < 150 MB, runs as non-root, trivy shows near-zero fixable CRITICALs on distroless.
**TRADEOFFS:** distroless (security, size) vs debuggability; slim (balance). Fintech default: distroless or slim+non-root, enforced by policy in CI (Module 09) and admission control (Module 13).

### EXERCISE 4.5 — Security scanning & signing [I] [45 min]

**STEPS**

1. `trivy image` on your naive vs hardened images; understand CVSS vs fixability vs exploitability; generate SBOM (`trivy image --format cyclonedx`).
2. `.trivyignore` governance: when is ignoring a CVE acceptable (no fix available + not exploitable in context) and who approves — write the policy (you'll enforce in pipeline, Module 09).
3. Sign images with cosign keyless (OIDC) or a local key; verify. (Supply-chain security — SLSA concepts; GitHub Actions will do keyless signing in Module 09.)
**VALIDATE:** scan reports + SBOM committed to repo; verification command works.

### EXERCISE 4.6 — Runtime knobs that matter in prod [I] [60 min]

**STEPS**

1. PID 1 problem: run node as PID 1 (`docker run node server.js`) — signals behave oddly (node doesn't forward SIGTERM by default as PID 1); fix with `tini` (`--init`) or proper signal handlers in code. Test: `docker stop` grace period behavior before/after.
2. Logging: `--log-driver json-file --log-opt max-size=10m --log-opt max-file=3` vs `fluentd` driver; K8s expectation: stdout/stderr only, rotation handled by kubelet — align.
3. Restart policies: no/always/unless-stopped/on-failure — why K8s replaces these.
4. Resource limits locally: `--memory=128m` → induce OOM, read `docker inspect` `OOMKilled:true`, find it in dmesg. CPU: `--cpus=0.5` + stress → throttle stats in `docker stats`.
5. Healthchecks: Dockerfile `HEALTHCHECK` with curl; watch `unhealthy` status on a broken dependency (stop the db container).
6. Networking modes: bridge/host/none + port publishing; user-defined bridge with DNS by container name (compose does this) — inspect iptables rules docker adds (`iptables -t nat -L DOCKER`).
**VALIDATE:** OOMKill reproduced and explained; signal handling fixed and demonstrated with graceful shutdown log lines.

### EXERCISE 4.7 — Compose as the dev-loop standard [B] [45 min]

**STEPS**

1. Dissect `assets/app/docker-compose.yml`: services (api×2, worker, web, postgres, mongo, redis, elasticsearch), healthcheck dependencies (`depends_on: condition: service_healthy`), named volumes, override files.
2. `docker compose up -d --scale api=3`; put nginx (web) in front as LB — round-robin across apis; observe sticky-session absence issues (note for Module 05 session affinity).
3. Profiles: add `tools` profile (adminer, mongo-express) that only starts with `--profile tools`.
4. `docker compose watch` or bind mounts for hot-reload dev loop.
5. Environment parity: `.env` files per env; `compose.yml` + `compose.prod.yml` layering — understand the merge rules.
**VALIDATE:** scaled api behind nginx LB serves; worker processes an order end-to-end (API → SQS-or-local queue → worker → Mongo write path; verify in Mongo).

---

## Part C — Registries & distribution

### EXERCISE 4.8 — ECR workflows (extends 3.20) [B] [30 min]

**STEPS**

1. Build → tag with git SHA (`git rev-parse --short HEAD`) → push all three images.
2. Pull-through cache concept + ECR replication config (cross-region replication for DR — enable to ap-southeast-1).
3. Image retention lifecycle rules via CLI JSON; dry-run the rule evaluation.
**VALIDATE:** SHA-tagged images in both regions.

### EXERCISE 4.9 — Local registry + offline workflows [I] [30 min]

**STEPS**

1. Run `registry:2` container; retag/push/pull — understand the wire protocol (`/v2/_catalog`).
2. Air-gap simulation: `docker save | gzip` images, import on `legacy-dc` (no internet registry) with `docker load`. When does this matter? Regulated/edge environments — one-line interview answer.
**VALIDATE:** app runs on legacy-dc from loaded tarballs.

---

## Part D — Container debugging gauntlet (pre-K8s warm-up)

### EXERCISE 4.10 — Broken containers clinic [A] [90 min]

Debug each scenario on the compose stack; document root cause + fix + detection method for each (this format = your future postmortems):

1. **Crashloop:** api env points at wrong DB host → container exits. Tools: `docker ps -a`, `docker logs --tail 50`, exit code reading (1 app, 137 SIGKILL/OOM, 143 SIGTERM).
2. **Silent hang:** worker connected to a blackholed queue endpoint (iptables DROP) — logs look fine, nothing processes. Tools: `docker exec` + `ss -tnp`, `strace -p`.
3. **Memory leak:** run `api` with `--memory=200m` and hit `/api/leak` endpoint (provided in sample app — allocates per request without free); watch `docker stats`, get OOMKilled, confirm 137.
4. **DNS weirdness:** add `dns: [8.8.8.8]` override breaking internal name resolution; symptoms vs fix; why containers should use the embedded resolver for service discovery.
5. **Permission denied:** volume mounted with wrong uid (root-owned) into non-root container; `fsGroup`-equivalent fix via chown init container pattern (preview of K8s).
6. **Time bomb:** image with `apt-get update` unpinned at build breaks 3 weeks later when a transitive dep changes — reproduce with an old cached layer; fix: lockfiles (`package-lock.json` + `npm ci`), base image digests (`node:20@sha256:...`).
**VALIDATE:** six mini-postmortems in `postmortems/` using the template.
**WHY-WHEN-WHAT:** these six are 80% of container incidents you'll ever see. The other 20% are the K8s versions of the same — next module.

---

## Module 04 exit gate

- [ ] You can explain containers via namespaces/cgroups/overlay from building one.
- [ ] All three app images: multi-stage, non-root, < 150 MB, scanned, SHA-tagged in ECR (both regions).
- [ ] Compose stack with healthchecks, scaling, profiles is your daily dev loop.
- [ ] Six-incident debug clinic documented.

**Onward:** `05-Kubernetes.md` — same app, real orchestrator.
