# Module 02 — Linux, Git, Bash & Python for SRE (Weeks 1–2)

> slice SRE-2's JD is explicit: "comfortable reasoning through a hung process, disk or inode exhaustion, a failing systemd unit, or a tricky networking issue" and "Bash required; Python strongly preferred." Multiple JDs add Go/Python for tooling. As a network engineer your OSI-layer instincts are strong — this module turns them into host-level debugging depth. Every exercise runs on your `legacy-dc` VM or an EC2 instance; break things freely.

Legend: **[B]**eginner / **[I]**ntermediate / **[A]**dvanced / **[E]**xpert

---

## Part A — Linux internals & production debugging

### EXERCISE 2.1 — Process lifecycle deep dive [B] [60 min]

**GOAL:** own `ps`, signals, states, /proc.
**STEPS**

1. Launch three background sleep loops: `sleep 1000 & sleep 2000 & sleep 3000 &`. Note PIDs via `jobs -l`.
2. Explore `/proc/<pid>/`: `cat /proc/<pid>/status`, `ls /proc/<pid>/fd`, `cat /proc/<pid>/limits`, `cat /proc/<pid>/cmdline`. Write down what each tells an SRE during an incident.
3. Signals: `kill -STOP <pid>` (observe `T` state in `ps`), `kill -CONT <pid>`, then `kill -9`. Explain why SIGKILL can't be caught and why that's both useful and dangerous (no cleanup → orphaned temp files, stale locks, corrupt writes).
4. `ps aux --sort=-%cpu | head`, then re-sort by RSS. Learn `pstree -p <pid>`.
5. Run a fork-bomb-safe experiment in a cgroup-limited shell: `systemd-run --user --scope -p MemoryMax=50M bash -c 'while true; do x=$((x+1)); done'` — watch systemd kill it. This is the same mechanism K8s uses for limits (cgroups) — connect those dots now.
**VALIDATE:** you can explain process states R/S/D/T/Z and name one real cause of each.
**TRADEOFFS:** SIGTERM-then-SIGKILL grace periods — why K8s `terminationGracePeriodSeconds` defaults to 30s and when you'd shorten (stateless web) vs lengthen (DB checkpointing, queue drains).

### EXERCISE 2.2 — The D-state mystery [I] [45 min]

**GOAL:** diagnose uninterruptible sleep (the classic "load is 40 but CPU is idle").
**STEPS**

1. Simulate stuck I/O: mount an NFS export from your `branch-office` VM, then firewall NFS mid-read (`sudo iptables -A OUTPUT -p tcp --dport 2049 -j DROP` on the client while `dd if=/mnt/nfs/bigfile of=/dev/null` runs).
2. Observe: `ps aux | awk '$8 ~ /D/'`, `cat /proc/loadavg`, `iostat -x 2` (install sysstat), `dmesg | tail`.
3. Try `kill -9` on the D-state process — note it does nothing. Understand *why* (kernel waiting on I/O completion).
4. Recover: remove the iptables rule; watch the process unstick. Then force the issue: `sudo umount -l` (lazy unmount) as the runbook step when the NFS server is truly dead.
5. Write runbook entry `runbooks/nfs-stale-mount.md`: symptoms → checks → fix → escalation.
**VALIDATE:** you captured a D-state process in `ps` output into your log with the evidence chain.
**TRADEOFFS/WHY:** In interviews this appears as "load average high, CPU low — what do you check?" Answer skeleton: `iostat` for device saturation → `ps` D-state count → NFS/EBS/network storage health → kernel logs. Mention that on EKS this pattern usually means a stuck volume attach or a throttled EBS burst balance.

### EXERCISE 2.3 — Disk full vs inode full [B] [45 min]

**GOAL:** the two different "disk full" failures and their fixes.
**STEPS**

1. Byte-full: `fallocate -l 90%OF_DISK /var/tmp/filler` (compute size from `df -h`). Verify `df -h` shows ~100%.
2. Notice: `df -i` still fine. Now clean up. This failure mode: big files (logs, core dumps, WAL).
3. Inode-full: on a spare loop filesystem (`dd if=/dev/zero of=/tmp/fs.img bs=1M count=200 && mkfs.ext4 /tmp/fs.img && sudo mount -o loop /tmp/fs.img /mnt/t`), run `for i in $(seq 1 200000); do touch /mnt/t/f$i; done` until `touch: No space left on device` while `df -h` shows free space. `df -i` shows IUse%=100%.
4. Practice detection commands: `df -h` vs `df -i`, find the culprit dir: `for d in /var/*; do echo "$d $(find $d -xdev 2>/dev/null | wc -l)"; done | sort -k2 -n | tail`.
5. Fix patterns: logrotate for bytes (`man logrotate`, force with `logrotate -f`), inode-full → delete millions of small files efficiently: `rsync -a --delete empty/ target/` or `find -delete`, and discuss why `rm -rf` on 1M files is slow (per-file syscalls).
**VALIDATE:** you reproduced both failures and can differentiate in < 60 seconds with two commands.
**TRADEOFFS:** inode exhaustion is the classic "small-files" workload failure (CI caches, container layers, mail spools). In K8s it surfaces as node pressure eviction; connect to `nodefs`/`imagefs` eviction thresholds.

### EXERCISE 2.4 — systemd as a service manager (and a debugger's friend) [I] [60 min]

**GOAL:** write, break, and fix systemd units — exactly what the JD names.
**STEPS**

1. Write `/etc/systemd/system/northpay-api.service` running the Node api directly (`ExecStart=/usr/bin/node /opt/app/server.js`), with `Restart=on-failure`, `RestartSec=3`, `MemoryMax=300M`, `EnvironmentFile=/etc/northpay/api.env`, `After=network-online.target`.
2. `systemctl daemon-reload && systemctl enable --now northpay-api`. Verify: `systemctl status`, `journalctl -u northpay-api -f`.
3. Break it three ways and diagnose each:
a. Wrong path in ExecStart → status `203/EXEC`. Learn `systemd-analyze verify` and reading exit codes.
b. Port already in use (start a dummy listener) → app exits, watch Restart loop; `systemctl show northpay-api -p NRestarts`, then hit `StartLimitBurst` and the unit enters `failed` permanently. Learn `systemctl reset-failed`.
c. MemoryMax exceeded → cgroup OOM kill; find it in `journalctl -k` and `systemctl status` (`oom-kill`). Difference vs container OOMKill: same kernel mechanism.
4. Add hardening directives you'd actually want in prod: `NoNewPrivileges=true`, `ProtectSystem=strict`, `ReadWritePaths=/var/lib/northpay`, `PrivateTmp=true`. Explain each in your log.
5. Learn `systemd-analyze blame`, `systemd-analyze critical-chain` for boot-time questions.
**VALIDATE:** all three failures diagnosed from journal evidence alone (write the journal lines into your log).
**TRADEOFFS:** why modern containers replaced many systemd-managed services — but note node-level agents (Datadog, Fluent Bit, node-exporter) still run as systemd units even in K8s shops; that's why JDs test it.

### EXERCISE 2.5 — Networking from the host's perspective [I] [75 min]

**GOAL:** translate your router/switch knowledge into Linux host networking — the exact skills K8s/CNI debugging needs.
**STEPS**

1. Interfaces & addresses the modern way: `ip -br a`, `ip -br l`, `ip r`, `ip neigh`. Compare with legacy `ifconfig`/`route -n` outputs (know both — older prod systems exist).
2. Namespaces (this is the foundation of containers): create two netns connected by a veth pair:

```bash
   sudo ip netns add red; sudo ip netns add blue
   sudo ip link add veth-red type veth peer name veth-blue
   sudo ip link set veth-red netns red; sudo ip link set veth-blue netns blue
   sudo ip netns exec red ip addr add 10.0.0.1/24 dev veth-red
   sudo ip netns exec blue ip addr add 10.0.0.2/24 dev veth-blue
   sudo ip netns exec red ip link set veth-red up; sudo ip netns exec blue ip link set veth-blue up
   sudo ip netns exec red ping 10.0.0.2
```

You have just built the same primitive Docker/K8s pod networking uses. Write that sentence in your log.

3. Add a bridge (`ip link add br0 type bridge`), attach both veths to it, re-ping. You've built the docker0/cni0 pattern.
4. NAT/masquerade by hand: `iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -j MASQUERADE` on the host with a default route out; verify a netns can reach the internet. This is exactly how pods reach external services.
5. Conntrack: `conntrack -L | head`; exhaust it: set `net.netfilter.nf_conntrack_max` low via sysctl, generate many short connections (`for i in $(seq 1 5000); do curl -s --max-time 1 localhost:3000/healthz; done` from a loop), watch `nf_conntrack: table full, dropping packet` in dmesg. Fix and restore. **This exact failure appears in K8s node interviews.**
6. ss mastery: `ss -tlnp` (who listens), `ss -tnp state established`, `ss -s` (summary — TIME_WAIT floods), `ss -ti` (TCP internals: cwnd, rtt).
7. tcpdump: capture a TLS handshake `tcpdump -i any -nn 'tcp port 443' -c 20`; identify SYN, SYN-ACK, ACK, then TLS ClientHello. Filter practice: by host, port, net, flags (`'tcp[tcpflags] & tcp-syn != 0'`).
**VALIDATE:** netns ping works; masquerade verified; conntrack exhaustion reproduced and fixed; you can read a TCP handshake in tcpdump.
**TRADEOFFS/WHY-WHEN-WHAT:** iptables vs nftables vs ipvs (kube-proxy modes) — you'll meet this in Module 05; plant the question now: "kube-proxy iptables mode does O(n) rule traversal per service; at 5k services this matters" — remember it.

### EXERCISE 2.6 — Performance triage toolkit: the first 60 seconds [A] [90 min]

**GOAL:** internalize a USE-method triage (Utilization, Saturation, Errors) so deep it's muscle memory.
**STEPS**

1. Install: `sysstat`, `htop`, `iotop`, `nethogs`, `perf` (`linux-tools`), `bpftrace` if available, `strace`, `lsof`.
2. Run each, learn 3 things it shows that others don't: `uptime`, `dmesg -T | tail`, `vmstat 1`, `mpstat -P ALL 1`, `pidstat 1`, `iostat -xz 1`, `free -m`, `sar -n DEV 1`, `top -H` (threads!).
3. Generate controlled load, one at a time, and capture the tool evidence of each:

- CPU: `stress-ng --cpu 4 --timeout 120s` → which tools show it? (`mpstat` per-core, `pidstat` per-process)
- Memory: `stress-ng --vm 2 --vm-bytes 2G` → watch `free`, `vmstat` (si/so swap), OOM in dmesg
- Disk: `fio --name=randrw --rw=randrw --size=1G --directory=/tmp` → `iostat -x` await/%util
- Network: `iperf3` between VMs → `sar -n DEV`, `nethogs`

4. strace a slow command: `strace -c -f curl localhost:3000/api/orders` — see where syscalls burn time. Then `strace -e trace=network -p <pid>` on the running API.
5. lsof forensics: `lsof -i :3000`, `lsof +L1` (deleted-but-open files — the classic "deleted the log but disk still full" case; reproduce it: `node server.js > big.log &`, fill big.log, `rm big.log`, disk still full, find with lsof, fix with `: > /proc/<pid>/fd/<fd>` or restart).
**VALIDATE:** for each of the 4 load types you have a note: "tool X, metric Y, threshold Z = problem."
**TRADEOFFS:** Brendan Gregg's USE method vs RED (Rate/Errors/Duration, service-level) vs Four Golden Signals (Google SRE: latency, traffic, errors, saturation). Know all three vocabularies — interviewers mix them; Module 11 maps them onto dashboards.

### EXERCISE 2.7 — Users, permissions, sudo, PAM & SSH at production depth [I] [60 min]

**STEPS**

1. Users/groups: `useradd -m -s /bin/bash deploy`, group strategy (`northpay` group, shared dir with setgid `chmod 2770`), `umask` reasoning.
2. sudoers via `visudo`: grant `deploy` passwordless restart of one unit only: `deploy ALL=(root) NOPASSWD: /bin/systemctl restart northpay-api`. Test abuse paths (can they edit the unit file? Why is that a privesc hole? → unit files are root-owned; but `/etc/systemd/system` write = root. Lock it.)
3. SSH hardening on `legacy-dc`: `/etc/ssh/sshd_config` — `PasswordAuthentication no`, `PermitRootLogin no`, `AllowUsers ops deploy`, `MaxAuthTries 3`, `ClientAliveInterval 300`. Restart sshd safely: keep a second session open while testing (write the recovery procedure in your runbook — locking yourself out of EC2 via sshd misconfig is a real interview scenario).
4. SSH deep: `ssh -vvv` read the handshake; agent forwarding (`-A`) and why it's risky on jump hosts (socket hijack) → prefer `ProxyJump`. Set up: branch-office → legacy-dc via ProxyJump.
5. File permissions tripwires: `chmod u+s` (SUID), `getcap`/`setcap` (why `setcap cap_net_bind_service=+ep /usr/bin/node` beats running as root for port 80), ACLs with `getfacl/setfacl`.
6. Audit trail: enable `auditd`, watch a file: `auditctl -w /etc/northpay/api.env -p wa -k secrets`; cat it, then `ausearch -k secrets`. This is the Linux side of "audit trails that come with a bank" (slice SRE-2).
**VALIDATE:** failed-abuse tests documented (e.g., deploy user cannot write unit file); audit event captured for the watched file.

### EXERCISE 2.8 — Logs, journald, rsyslog, and time [B] [45 min]

**STEPS**

1. journald: persistent storage (`Storage=persistent`), `journalctl --disk-usage`, vacuuming, `-p err`, `--since "1 hour ago"`, `-o json-pretty` (this JSON is what log shippers parse).
2. rsyslog forwarding: ship `legacy-dc` logs to `branch-office` (`*.* @@192.168.200.10:514` TCP — why TCP not UDP for security/compliance?), verify with logger + receiver side.
3. Time: `timedatectl`, chrony sync status (`chronyc sources -v`). Then break time: `sudo timedatectl set-ntp false; sudo date -s "-10 minutes"` and observe TLS failures (`curl` to an HTTPS endpoint — cert not-yet-valid errors). Restore. Interview gold: "API calls failing with cert errors, certs are valid — what do you check?"
4. Write `runbooks/clock-skew.md` — symptoms (TLS errors, token expiry weirdness, metrics gaps) → `timedatectl` → fix.
**VALIDATE:** remote log line received; TLS failure reproduced from skew and explained precisely.

### EXERCISE 2.9 — Package management & reproducibility [B] [30 min]

**STEPS**

1. `apt` end-to-end: pin a version (`apt-mark hold`), list package files (`dpkg -L`), find owning package (`dpkg -S`), inspect scripts (`apt-get download` + `dpkg-deb -e`).
2. Snap vs deb vs source build — install nginx all three ways on different VMs/ports; compare config locations, service management, update behavior.
3. Concept check in your log: why do immutable infrastructure + containers make package management *less* central, yet `apt` knowledge still matters for AMIs, node images, and break-glass fixes?

---

## Part B — Git at team scale

### EXERCISE 2.10 — Git internals that save you in incidents [I] [60 min]

**STEPS**

1. `.git` tour: `git hash-object -w`, `cat-file -p`, see blobs/trees/commits. Understand why Git is a content-addressable store.
2. reflog rescue: `git reset --hard HEAD~3` on a scratch repo with commits, recover via `git reflog`. Runbook entry: "I force-pushed/reset and lost commits" — the #1 junior panic you'll fix for teammates.
3. `git bisect` for real: script a bug (`git bisect run bash -c 'grep -q BUG src/config.txt'`) over 20 commits where commit 11 injected the bug. Time it. Interviewers love bisect stories.
4. Interactive rebase: clean a messy branch (`rebase -i` squash/reword/drop), then `push --force-with-lease` (why `--force-with-lease` over `--force`).
5. cherry-pick a hotfix across release branches — the standard prod hotfix flow: `main` → `release/1.4` cherry-pick → tag `v1.4.1`.
6. Submodules vs subtrees vs "just separate repos": add `northpay-runbooks` as a submodule of a `northpay-meta` repo; experience the friction; remove it. Now you have an opinion with evidence.
**VALIDATE:** bisect found the bad commit in ≤ 5 steps; reflog recovery demo written in runbook.

### EXERCISE 2.11 — Team workflows: trunk-based vs GitFlow [I] [45 min]

**STEPS**

1. Simulate trunk-based: short-lived feature branches → PR → squash merge → `main` always releasable; tag releases from main. Do 3 cycles in `northpay-app`.
2. Simulate GitFlow: `develop`, `feature/*`, `release/*`, `hotfix/*`. Do one full release cycle.
3. In an ADR, choose one for NorthPay with justification (hint: high-frequency deployment + GitOps → trunk-based; GitFlow for tightly regulated release trains). Name the tradeoffs explicitly: merge complexity vs release isolation.
4. Commit conventions: adopt Conventional Commits (`feat:`, `fix:`, `chore:`) — Module 09 will auto-generate changelogs from them.

---

## Part C — Bash for operators

### EXERCISE 2.12 — Bash correctness fundamentals [B] [60 min]

**STEPS**

1. Write `~/northpay/scripts/lib/strict.sh`: `set -euo pipefail`, `IFS=$'\n\t'`, `trap` for cleanup, `readonly` for constants. Source it in every script from now on.
2. Quoting lab: intentionally break a script with unquoted vars on filenames with spaces; fix with `"$var"` and arrays.
3. `[[ ]]` vs `[ ]` vs `(( ))`; regex with `=~`.
4. Exit codes: design a health-check script returning 0/1/2 (ok/warn/crit) — Nagios-style; this exact pattern becomes a Prometheus blackbox/Zabbix check in Module 11.
5. Idempotency patterns: `mkdir -p`, `grep -q || echo >> file`, `systemctl is-enabled || systemctl enable`. Rule: running a script twice must be a no-op.
**VALIDATE:** `shellcheck` clean on all scripts (install it; it's your pipeline gate in Module 09).

### EXERCISE 2.13 — Text-processing gauntlet [I] [90 min]

**GOAL:** awk/sed/grep/jq fluency for log and API wrangling.
**STEPS** (dataset: `assets/seed-data/sample-access.log` — 100k nginx lines, and the app API)

1. Top 10 IPs by request count: `awk '{print $1}' log | sort | uniq -c | sort -nr | head`.
2. Error rate per minute: extract 5xx with awk, bucket by minute.
3. p95-ish latency approximation from `$request_time` using awk percentiles (sort + NR*0.95 trick). Explain why awk percentiles ≠ real histograms (you'll meet real ones in Prometheus).
4. sed in-place edits with backup (`sed -i.bak`), multi-pattern deletes, and a config-templating use: generate 3 nginx vhosts from one template + envsubst.
5. jq: pipe `curl localhost:3000/api/orders | jq` — filter orders > 1000 INR, group by customer, `.[] | select(...)`. Then `aws ec2 describe-instances | jq` games: list all instances with tag env=prod and their AZs.
6. xargs parallel: `cat hosts.txt | xargs -P8 -I{} ssh {} uptime`.
**VALIDATE:** all six outputs saved to your log; shellcheck clean.

### EXERCISE 2.14 — Write the NorthPay ops toolkit v1 in Bash [A] [2 hrs]

**STEPS** — build these five scripts for real; they'll be reused in tickets:

1. `np-health`: checks app /healthz, db connectivity, disk>85%, load>nproc; Nagios exit codes; `--verbose` mode.
2. `np-backup-pg`: pg_dump the local Postgres, gzip, timestamped, retain last 7 (`find -mtime +7 -delete`), verify backup by listing archive contents; exit non-zero on any failure.
3. `np-deploy-check`: after a deploy, polls /healthz 30x/2s, then 50 requests measuring latency, prints min/avg/max; non-zero if any failure.
4. `np-log-triage <since>`: given "10 min ago", aggregates journalctl + nginx log errors, dedupes, prints top 5 signatures.
5. `np-cost-today`: AWS CE API (`aws ce get-cost-and-usage`) for today's spend by service, sorted. (Needs Cost Explorer enabled — done in 0.1.)
**VALIDATE:** each script has `--help`, strict mode, shellcheck clean, committed with a test run in the log.
**TRADEOFFS:** when does Bash stop being enough? Your answer by feel: >~200 lines, need data structures, need error *handling* not just exit codes, need to talk to APIs with auth/retries → Python. State this in interviews.

---

## Part D — Python for SRE tooling

### EXERCISE 2.15 — Python setup & the SRE stdlib tour [B] [60 min]

**STEPS**

1. `python3 -m venv ~/.venvs/ops`; pip install: `boto3 requests rich click pydantic pytest`.
2. Stdlib tour with mini-tasks: `os`/`pathlib` (walk /var/log, sum sizes), `subprocess` (run `df -h`, parse), `json`, `datetime`/`zoneinfo` (log timestamps across TZs — fintech pain point), `concurrent.futures` (parallel HTTP checks to 20 URLs), `argparse` → then rewrite with `click` and compare ergonomics, `logging` (structured JSON logs with `python-json-logger`).
3. Error-handling discipline: write `retry()` decorator with exponential backoff + jitter (you'll reuse it for AWS API calls). Explain why jitter (thundering herd).
**VALIDATE:** retry decorator demoed against a flaky local endpoint (make one: flask app failing 50% of the time).

### EXERCISE 2.16 — boto3: automate your own AWS [I] [90 min]

**STEPS**

1. `np_inventory.py`: list all EC2/RDS/S3/ELB across ap-south-1 with tags, output rich table; `--json` flag for piping.
2. `np_stop_after_hours.py`: stop all instances tagged `ttl=daily` (dry-run flag first!). Schedule with cron `0 20 * * *`. This is real toil-reduction — the slice SRE-2 JD's "automates away the repetitive work."
3. `np_s3_audit.py`: find buckets without encryption or with public access; report. (Preview of Module 13 security baseline.)
4. Retry/throttling: provoke `ClientError` throttling with a tight loop of describe calls, handle with your retry decorator + botocore's adaptive mode; log the difference.
5. Pagination: prove you handle it (list S3 objects > 1000 in a test bucket).
**VALIDATE:** cron stops a tagged test instance; audit script flags a deliberately-created public bucket.

### EXERCISE 2.17 — Build a real CLI: `npctl` [A] [3 hrs]

**GOAL:** the "you built a tool" story every platform JD wants.
**STEPS**

1. click-based CLI `npctl` with commands:

- `npctl status` — fleet/app/DB health rollup (reuse 2.16 + HTTP checks)
- `npctl deploy-env <name>` — scaffold a namespace/configs for a new env (file templating with Jinja2)
- `npctl drain <host>` — systemctl stop app, wait for connections to drain, report (precursor to K8s pod eviction)
- `npctl incident <title>` — creates postmortem skeleton from template + timestamped incident log file

2. Config via pydantic-settings (env vars + yaml), structured logging, `npctl --version`.
3. Tests: pytest for retry logic and at least 2 commands with mocked boto3 (`moto` or `botocore.stub.Stubber`).
4. Package it: `pyproject.toml`, `pip install -e .`, entry point works from anywhere.
**VALIDATE:** `npctl status` gives one-screen health of the whole local lab; tests pass in CI (Module 09 will wire this).
**TRADEOFFS:** Python vs Go for ops tooling — Python wins on speed-of-writing + boto3 maturity; Go wins on single-binary distribution (great for CLI tools to fleets), concurrency, and being the language of K8s/Terraform ecosystems (read their source!). Both JDs prefer Go — do this module in Python, and in Module 18 rewrite `npctl status` in Go as the interview-prep exercise.

### EXERCISE 2.18 — Go awareness sprint (for the JDs that prefer Go) [B→I] [2 hrs]

**STEPS**

1. Install Go; write: a concurrent HTTP health-checker reading URLs from a file (goroutines + channels + WaitGroup), printing a status table.
2. Build a static binary: `CGO_ENABLED=0 go build`; scp it to `legacy-dc` and run — feel the distribution advantage vs Python.
3. Read (don't write) a small K8s controller example or `kubectl` plugin skeleton; note in your log how client-go maps to the kubectl commands you know.
**VALIDATE:** static binary runs on the VM with no dependencies.
**TRADEOFFS:** state clearly when you'd pick Go (fleet agents, K8s-adjacent tooling, performance) vs Python (glue, data, speed of iteration). This exact framing impresses Staff-level interviewers.

---

## Module 02 exit gate

- [ ] You reproduced and fixed: D-state hang, disk-full, inode-full, systemd restart loop, cgroup OOM, conntrack table full, clock-skew TLS failure, deleted-but-open file.
- [ ] `np-health`, `np-backup-pg`, `np-deploy-check`, `np-log-triage`, `np-cost-today` committed & cron-wired.
- [ ] `npctl` installed as a real package with tests.
- [ ] git bisect + reflog rescue in your runbook.
- [ ] 8+ runbook entries written; every failure got a mini-postmortem.

**Onward:** `03-AWS-Foundations.md`. Keep doing daily tickets — the board assumes Module 02 skills from Day 4.
