# Module 12 — Databases for SREs (Weeks 16–17)

> **JD coverage:** "Postgres, DocumentDB" + "Mongo" (Deel DevOps); "PostgreSQL, DynamoDB, ElasticSearch... enough to triage alongside our DBAs" (slice SRE-2); "RDS/Aurora" (slice SRE-2); databases are named in four of six JDs. You are not becoming a DBA — you're becoming the SRE who keeps stateful systems alive, restorable, and performant, and who can *talk* to DBAs in their language.
> Seed data: `assets/seed-data/` (postgres_seed.sql, mongo_seed.js, elasticsearch_bulk.ndjson, dynamodb_seed.py — generators included; run them to create volume).

**Universal drill rules (apply to every engine below):** (1) backups are fiction until you've restored; (2) measure every restore's RTO; (3) know the RPO the tool actually gives you vs what the business thinks it gets.

---

## Part A — PostgreSQL (RDS + self-hosted + Aurora)

### EXERCISE 12.1 — Load & explore the NorthPay schema [B] [45 min]

**STEPS**

1. On your RDS instance (Module 03): `psql -f assets/seed-data/postgres_seed.sql` — creates `customers`, `accounts`, `orders`, `settlements`, `audit_log` with FKs, indexes, then a generator inserting 50k customers / 500k orders (`generate_series` based — inspect the SQL first; know what you're loading).
2. Verify cardinalities; `\dt+`, `\di+` sizes; `pg_stat_statements` reset.
3. Point the api at it (already done in 3.15); run gen_load; watch `pg_stat_activity` live (`SELECT pid, state, query_start, wait_event_type, query FROM pg_stat_activity WHERE state <> 'idle'`).
**VALIDATE:** 500k orders; you can see live queries during load.

### EXERCISE 12.2 — Query performance: EXPLAIN literacy [I] [90 min]

**STEPS**

1. `EXPLAIN (ANALYZE, BUFFERS)` on the api's top 5 queries (get them from pg_stat_statements by total time).
2. Force a seq scan problem: query `orders WHERE upper(customer_email)=...` (no index supports it) → see Seq Scan on 500k rows; fix with expression index; measure before/after.
3. B-tree vs GIN vs partial vs covering indexes: build one of each on this schema; know when (JSONB containment → GIN; "recent unsettled" → partial WHERE; covering INCLUDE to hit index-only scans).
4. `work_mem` spill: force a big sort, watch `Sort Method: external merge Disk` — tune per-query first, not globally.
5. N+1 demo: the api's `/api/customers/:id/orders-detailed` does N+1 (check the code); prove via query count in logs; fix with a join; measure latency delta.
6. Lock forensics: open txn A holding a row lock (`BEGIN; UPDATE orders ...`), txn B blocks → query `pg_locks` + `pg_blocking_pids()`; kill the blocker (`pg_terminate_backend`) — and write when that's safe vs dangerous (long transactions, idle-in-transaction — find those: `state = 'idle in transaction'` aged > 5 min, the classic pool-exhaustion cause).
7. Autovacuum: watch it; understand bloat (`pg_stat_user_tables.n_dead_tup`); when manual VACUUM/REINDEX; why wraparound prevention is sacred.
**VALIDATE:** N+1 fixed with evidence; blocking-lock incident simulated and resolved via catalog queries; index added cut p95 by >10× on the target query.

### EXERCISE 12.3 — Backup, PITR, and restore drills (the real thing) [I] [2 hrs]

**STEPS**

1. **Self-hosted first** (legacy-dc Postgres — full control): base backup with `pg_basebackup` + WAL archiving to a local dir ("S3 stand-in"), then PITR: delete rows at T, restore to T-1min on a standby data dir (`recovery_target_time`), verify. **Doing PITR by hand once teaches you what RDS automates — and what can still go wrong (WAL gaps!).**
2. Logical backup: `pg_dump -Fc` + `pg_restore` into a *different* DB name; table-level restore (`-t orders`); parallel restore `-j`.
3. RDS PITR drill (repeat of 3.16 but timed + scripted): use `assets/scripts/pitr_drill.sh`; target: full drill < 45 min including verification. Record actual minutes — that's your measured RTO.
4. Snapshot vs PITR vs logical: RPO/RTO/cost/scope table (write it): snapshot (crash-consistent whole-instance), PITR (5-min granularity-ish, new instance), logical (table-level, slow, portable across versions).
5. Cross-region automated backups (RDS feature) + encrypted snapshot copy — preview of Module 15 DR.
6. Restore verification automation: post-restore script checking row counts + canary query + app connectivity — "a restore you haven't verified is a hope."
**VALIDATE:** hand-rolled PITR succeeded on self-hosted; RDS drill timed; verification script green.

### EXERCISE 12.4 — Replication, failover & connection management [A] [2 hrs]

**STEPS**

1. Streaming replication on self-hosted: primary on legacy-dc, physical standby on branch-office VM (`pg_basebackup -R`), `pg_stat_replication`, lag monitoring. Controlled promotion (`pg_promote`) + timeline concept; then a *hostile* drill: kill -9 the primary mid-write, promote standby, measure data loss window (RPO ≈ replication lag).
2. Synchronous vs async replication: flip `synchronous_commit=remote_apply`, measure write latency penalty vs RPO=0. **The core durability-vs-latency tradeoff — know the numbers from your own lab.**
3. RDS Multi-AZ failover re-drill (Module 03 did it) — now with pgBouncer-style thinking: how do apps survive the CNAME flip? (DNS TTL, driver retry, pooler.) Test app behavior during failover: how many requests failed? Tune pool `max`, `idleTimeoutMillis`.
4. Connection exhaustion lab: set api pool to 500 conns → RDS `max_connections` (t3.micro ≈ 90-ish, check `SHOW max_connections`) → errors; fix with **RDS Proxy** (create it, point app, watch DatabaseConnections flatten). When Proxy pays for itself: Lambda/bursty/many-pod fleets (your EKS api at 20 replicas × 20 pool = 400 conns — do the math).
5. Logical replication intro: `wal2json` concept or native pub/sub (`CREATE PUBLICATION`) → replicate one table to another PG — the foundation of zero-downtime migrations and CDC (Debezium mention).
**VALIDATE:** standby promotion executed both friendly and hostile with measured RPO; app survives RDS failover with < N failed requests (record N); RDS Proxy flattens connection spikes.

### EXERCISE 12.5 — Aurora migration & operations [A] [2 hrs] [COST: run same-day, then decide]

**STEPS**

1. Migrate RDS Postgres → Aurora (method: create Aurora read replica of the RDS instance, promote — the lowest-downtime native path; note DMS as the heterogeneous option, Module 14 uses DMS cross-cloud).
2. Aurora anatomy: writer/reader endpoints, cluster storage (auto-scaling, 6 copies/3AZ), `aurora_replica_status()`, reader lag vs physical replication lag.
3. Failover drill: `aws rds failover-db-cluster`; measure endpoint flip time (typically < 30s — compare with your RDS Multi-AZ numbers from 3.17).
4. Aurora-specific ops: backtrack (concept — know it exists, its limits), cloning (fast copy-on-write clones — **perfect for "give QA a prod-like DB" and for risky migration rehearsals**; create a clone, run a destructive migration test on it, delete it).
5. Cost review: Aurora vs RDS pricing for your footprint; when plain RDS wins (small, steady) vs Aurora (HA-critical, read-scaling, storage growth).
**VALIDATE:** migration completed with downtime measured; clone-test-destroy cycle done; cost comparison written.

### EXERCISE 12.6 — Zero-downtime schema migrations (expand/contract) [A] [90 min]

*(The pattern behind every "how do you change schema without downtime" interview answer.)*
**STEPS**

1. Scenario: split `customers.full_name` → `first_name`+`last_name` while the api keeps deploying.
2. Execute the pattern: **expand** (add new columns nullable + dual-write in app v1) → **backfill** (batched UPDATE, 10k rows/batch, throttled — write the migration job as a k8s Job with progress logging) → **verify** (consistency check query) → **contract** (app v2 reads new columns; drop old column in a *separate later* release).
3. Failure modes to name: adding column with DEFAULT on huge table (table rewrite on old PG; fast on PG11+ — know version behavior), adding index without CONCURRENTLY (locks writes — reproduce the lock on a busy table, then redo `CREATE INDEX CONCURRENTLY`; note its own failure mode: invalid index cleanup).
4. Migration tooling: embed migrations in CI (Flyway/Liquibase/Node migrate — sample uses `node-pg-migrate`); rule: migrations run as ArgoCD PreSync hooks (Module 10 wired this), must be backward-compatible by design.
**VALIDATE:** full expand-contract cycle with app serving traffic throughout (gen_load running, zero 5xx); the non-concurrent index lock incident documented.

---

## Part B — MongoDB & DocumentDB

### EXERCISE 12.7 — MongoDB replica set operations [I] [2 hrs]

**STEPS**

1. Self-hosted 3-node replica set in docker-compose (`assets/seed-data/mongo-compose.yml`): rs.initiate, priorities, one delayed member concept (know why: fat-finger protection window).
2. Load `mongo_seed.js` (customers activity feed, KYC docs metadata — 200k docs). Understand the sample schema's embedding choices (orders embedded in activity vs referenced — debate both).
3. CRUD + indexes: explain("executionStats") on the api's queries; covered queries; TTL index for session-like data (auto-expiry — useful pattern); partial indexes; the ESR rule (Equality, Sort, Range) for compound index order.
4. **Failover drill:** kill the primary container mid-write; watch election (time it — RTO), write concern `majority` vs `w:1` behavior during the gap (prove w:1 writes can roll back after failover — `ROLLBACK` in logs; that's RPO reality).
5. Backup/restore: `mongodump --oplog` (point-in-time-ish via oplog replay) + `mongorestore`; filesystem snapshot alternative; verify by restore into a scratch set.
6. Read preference routing: send analytics reads to secondaries (`readPreference=secondaryPreferred`); measure impact on primary during a heavy report.
**VALIDATE:** election time measured; rollback-under-w:1 reproduced and explained; restore verified.

### EXERCISE 12.8 — DocumentDB on AWS [I] [90 min]

**STEPS**

1. Provision a small DocumentDB cluster (db.t3.medium, 1 instance — cheapest viable) in data subnets; TLS in-transit (rds-combined CA bundle — app config change; do it), SG from app only.
2. Migrate: `mongodump` from self-hosted → `mongorestore` into DocumentDB (note the gotchas: no `$where`, limited aggregation stages, different index behaviors — hit one incompatibility on purpose using the seed's TTL index options; document).
3. Failover + replica scaling similar to Aurora (cluster endpoints, reader endpoint).
4. When DocumentDB vs self-managed Mongo on EC2 vs Atlas on AWS (write the matrix: ops burden, compatibility, cost, features). Deel's JD names DocumentDB — you can now discuss it credibly.
5. Change streams → worker (cache invalidation use case): implement the sample (`assets/app/worker/change-stream.js`) watching orders → invalidating redis keys.
**VALIDATE:** app runs against DocumentDB; incompatibility documented with workaround; change-stream flow works.

---

## Part C — Elasticsearch / OpenSearch

### EXERCISE 12.9 — Elasticsearch cluster fundamentals [I] [2 hrs]

**STEPS**

1. 3-node ES 8 cluster via compose (assets) — node roles (master/data/ingest), shards & replicas (set 3 shards/1 replica on an index; watch allocation: `_cat/shards`), yellow/red status meanings (unassigned shards — force one by killing a node, then `_cluster/allocation/explain`).
2. Load `elasticsearch_bulk.ndjson` (500k transaction-search docs) via bulk API from file (`curl -H Content-Type:application/x-ndjson --data-binary @...`); bulk sizing (5–15 MB batches) and rejection handling (429s = queue full → backoff).
3. Mapping matters: dynamic mapping creates text+keyword duals; define an explicit mapping (date formats, keyword for IDs, `text` with analyzer for narrations); reindex into the mapped index (`_reindex` API) — your first zero-downtime reindex (alias swap pattern: write alias `transactions` → v1; build v2; atomically swap alias; delete v1 later).
4. Query lab: match vs term, bool/filter context (filter = cached, no scoring — perf), aggregations (daily settlement volume, per-customer histograms) powering a "search" endpoint in the web app.
5. ILM: hot→warm→delete policy (hot 7d on SSD, warm 30d, delete 60d — compliance retention story); rollover via alias; `_ilm/explain`.
6. Snapshot lifecycle: repository on S3 (install repository-s3 plugin; IRSA/keystore), SLM policy nightly; **restore drill**: delete index, restore from snapshot, alias back. Timed.
7. Performance & safety: circuit breakers, heap sizing (50% RAM rule), slow logs, refresh_interval tuning for bulk loads, forcemerge on read-only warm indices.
**VALIDATE:** alias-swap reindex with zero query downtime; ILM rolls indices; snapshot restore verified; red-status recovery via allocation explain documented.

### EXERCISE 12.10 — OpenSearch on AWS + ELK-vs-Loki positioning [I] [90 min]

**STEPS**

1. Amazon OpenSearch Service: create a small domain (t3.small, 1 node, fine-grained access control + master user in Secrets Manager); SG/IAM access policy (both exist — understand the split: SG = network, FGAC/IAM = authz).
2. Ingest options: direct bulk from app vs OpenSearch Ingestion vs Fluent Bit output from your k8s logs (wire fluent-bit → OpenSearch for the `audit` log namespace only — per ADR-0011, OpenSearch = security/audit search).
3. OpenSearch Dashboards: build an audit-events dashboard (who changed what, when — slice SRE-2 compliance artifact).
4. ISM (OpenSearch's ILM); UltraWarm/cold storage concepts (cost lever).
5. ADR-0012: Elasticsearch self-managed vs OpenSearch Service vs Elastic Cloud vs "just Loki" — cost, ops, features (vector/k-NN, anomaly detection), license history (SSPL fork story — know it in one paragraph).
**VALIDATE:** audit logs searchable with dashboards; ADR committed.

---

## Part D — DynamoDB & ElastiCache (breadth)

### EXERCISE 12.11 — DynamoDB: the NoSQL at scale model [I] [90 min]

**STEPS**

1. Table `np-sessions`: PK customer_id, SK session_ts; on-demand vs provisioned+autoscaling (cost math both modes for your load profile); GSI on `status` (sparse index power); LSI limits.
2. Load with `dynamodb_seed.py` (boto3 batch_writer, 100k items); observe WCU behavior; hot-partition demo: query pattern hammering one PK → throttling (provisioned mode, low WCU) → fix by design (write sharding suffixes).
3. Consistency: eventually vs strongly consistent reads (cost×2); transactions API (TransactWriteItems) — when NoSQL joins die and transactions are the escape hatch (settlement ledger use case).
4. TTL attribute for session expiry; Streams → Lambda-less consumer (read stream via CLI/SDK) → CDC into OpenSearch pattern (the classic "DDB streams → search index").
5. Backup: PITR enable (35-day), on-demand backup, restore-to-new-table; point-in-time drill: delete items, restore table, compare.
6. DAX (cache) awareness — when ElastiCache-in-front beats DAX (multi-purpose caching) vs DAX simplicity.
**VALIDATE:** hot-partition throttling reproduced + design fix written; PITR restore verified; stream consumer reading changes.

### EXERCISE 12.12 — ElastiCache Redis: caching patterns & failure modes [I] [75 min]

**STEPS**

1. Cluster-mode-disabled redis (single + replica) in data subnets; app integration: `/api/orders/:id` cached (TTL 60s) — cache-aside pattern in the sample code; measure latency p95 before/after cache (and stampede when TTL expires — then add request coalescing/jittered TTL; name the thundering-herd problem).
2. Patterns implemented: cache-aside, write-through (concept), distributed lock (`SET NX PX` + unlock Lua — then read about Redlock debate; write both sides), rate limiter (INCR+EXPIRE sliding window — used by your API gateway later).
3. Eviction policies: maxmemory 100MB + allkeys-lru vs volatile-ttl — fill it, watch evictions metric, understand `OOM command not allowed` errors with noeviction.
4. Failover drill: trigger failover; app behavior during DNS/endpoint flip (errors? retries?); Multi-AZ with automatic failover timing.
5. Persistence: AOF vs RDB on self-hosted comparison; on ElastiCache: snapshots + why persistence ≠ durability guarantee (cache is rebuildable by design — if you need durability, it's not a cache).
6. Redis in K8s: your dev redis runs in-cluster (Helm) — contrast HA story vs ElastiCache; when each.
**VALIDATE:** stampede reproduced then fixed (evidence: redis `keyspace_misses` + DB query count graphs); failover survived with retry logic.

---

## Module 12 exit gate

- [ ] Every engine loaded with generated data, secured (SG/TLS/secrets), monitored (key metrics on your Grafana: connections, replication lag, slow queries, ES red/yellow, DDB throttles, redis evictions).
- [ ] Restore drills executed + timed for: PG (PITR self-hosted + RDS), Mongo (dump+oplog), ES (snapshot), DDB (PITR). Restore-verification scripts exist for each.
- [ ] Failover drills: PG standby promotion + RDS/Aurora failover + Mongo election + redis failover — each with measured RTO/RPO and app-impact notes.
- [ ] Zero-downtime migration (expand/contract) executed under load.
- [ ] ADRs: ES-vs-OpenSearch-vs-Loki; DocumentDB positioning; cache durability line.

**Onward:** `13-AWS-Advanced-Networking-and-Security.md` — multi-account, Transit Gateway, PrivateLink, hybrid DNS, WAF/Cloudflare, compliance-as-code. Your networking degree pays off here.
