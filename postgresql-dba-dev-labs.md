# PostgreSQL 17 — Senior DBA + Developer Lab Curriculum (AlmaLinux 9)

Hands-on labs to deploy, break, test, and master. Native installs only (systemd / RPM / binaries) — **no Docker**. Every lab is a concrete deployable exercise with a success criterion.

## Count Summary

| Track | Domain blocks | Labs |
|---|---|---|
| DBA | 12 | 78 |
| Developer | 6 | 42 |
| Cross-cutting (chaos / cloud / automation) | 3 | 18 |
| Migration (PG upgrades + Oracle/MySQL/MSSQL → PG) | 5 | 34 |
| Production readiness (enterprise / senior gaps) | 7 | 31 |
| Deep internals & scale architecture | 4 | 19 |
| **Total** | **37** | **222** |

**Suggested rig:** 3 AlmaLinux 9 VMs (`pg-primary`, `pg-standby1`, `pg-standby2`) + 1 `pg-client`/monitoring box. 4 GB RAM each minimum; 8 GB for tuning/benchmark labs. Snapshot before destructive labs.

---

# TRACK A — DBA (78 labs)

## A1. Installation & Cluster Provisioning (8)

1. Install PostgreSQL 17 from the PGDG RPM repo; verify `/usr/pgsql-17/bin` on PATH.
2. `initdb` a fresh cluster with a non-default data dir, locale, and checksum enabled (`--data-checksums`).
3. Run two clusters on one host (ports 5432/5433) as separate systemd units.
4. Relocate the data directory to a dedicated mounted volume; fix SELinux context (`semanage fcontext`).
5. Initialize a cluster with `initdb -k` and prove checksums are on via `pg_controldata`.
6. Set up peer vs md5 vs scram-sha-256 auth in `pg_hba.conf`; test each from `pg-client`.
7. Build PostgreSQL 17 from source with a custom `--prefix`; run alongside the RPM install.
8. Write a systemd override to raise `LimitNOFILE` and set `OOMScoreAdjust`; verify with `systemctl show`.

## A2. Configuration & Tuning (7)

9. Tune `shared_buffers`, `effective_cache_size`, `work_mem`, `maintenance_work_mem` for the VM's RAM; measure before/after with `pgbench`.
10. Configure `huge_pages`; confirm allocation via `/proc/meminfo` and server log.
11. Set kernel params (`vm.overcommit_memory=2`, `kernel.shmmax`) in `/etc/sysctl.conf`; reboot-test persistence.
12. Compare `ALTER SYSTEM` vs editing `postgresql.conf`; inspect `postgresql.auto.conf` and `pg_settings`.
13. Reload vs restart: change a SIGHUP param and a restart-only param; observe which takes effect on `reload`.
14. Configure logging: `logging_collector`, `log_line_prefix`, `log_min_duration_statement`, rotation.
15. Benchmark `wal_sync_method` options with `pg_test_fsync`; pick the fastest safe one.

## A3. Backup & Recovery (10)

16. `pg_dump` in all four formats (plain/custom/directory/tar); restore each with `pg_restore`.
17. `pg_dumpall -g` to capture globals; rebuild roles+tablespaces on a fresh cluster.
18. Parallel dump/restore (`-j`) of a large DB; time it vs single-threaded.
19. Take a physical base backup with `pg_basebackup` (plain + tar, with `-P`).
20. Enable WAL archiving (`archive_mode`, `archive_command`); verify segments land in the archive dir.
21. **Point-in-Time Recovery:** archive WAL, drop a table "by accident," recover to just before the drop using `recovery_target_time`.
22. Recover to a named restore point (`pg_create_restore_point` + `recovery_target_name`).
23. Verify a base backup with `pg_verifybackup`; corrupt a file and watch it fail.
24. Incremental backup chain (PG17): `pg_basebackup --incremental` + `pg_combinebackup` to reconstruct.
25. Install and drive pgBackRest: configure a repo, full + differential + incremental, then restore.

## A4. Replication & High Availability (11)

26. Streaming replication: build one hot standby with `pg_basebackup -R`; confirm `pg_stat_replication`.
27. Add a replication slot; prove WAL is retained when the standby is offline.
28. Cascading replication: standby-of-a-standby.
29. Synchronous replication: set `synchronous_standby_names`; test commit latency and standby-down behavior.
30. Promote a standby (`pg_ctl promote` / `pg_promote()`); repoint the app.
31. `pg_rewind` a former primary back into the cluster after a failover without a full rebuild.
32. Delayed replica (`recovery_min_apply_delay`) as a logical-corruption safety net.
33. Logical replication: `CREATE PUBLICATION`/`SUBSCRIPTION`, replicate a subset of tables.
34. Column-list + row-filter logical replication (PG15+ features).
35. `pg_createsubscriber` (PG17): convert a physical standby into a logical subscriber.
36. Deploy Patroni + etcd for automated failover; kill the primary and watch leader election.

## A5. Connection Management & Pooling (4)

37. Install and configure PgBouncer (transaction pooling); route the app through 6432.
38. Compare session vs transaction vs statement pooling modes under `pgbench -c`.
39. Configure `max_connections` vs pooler sizing; find the point where the raw DB thrashes.
40. Set per-role/per-db connection limits (`ALTER ROLE ... CONNECTION LIMIT`); test enforcement.

## A6. Security & Access Control (9)

41. Build a least-privilege role hierarchy (group roles + `GRANT`/`REVOKE`, `NOINHERIT`).
42. `ALTER DEFAULT PRIVILEGES` so new tables auto-grant correctly to an app role.
43. Row-Level Security: multi-tenant table, `CREATE POLICY`, prove tenant isolation.
44. Column-level privileges; grant SELECT on some columns only.
45. Configure TLS: generate certs, `ssl=on`, force `hostssl` in `pg_hba.conf`, verify with `\conninfo`.
46. Client certificate authentication (`clientcert=verify-full`).
47. `SET ROLE` / `SECURITY DEFINER` functions; demonstrate privilege escalation risk and the `search_path` fix.
48. Audit logging with `pgaudit`: log DDL and specific role activity.
49. `REASSIGN OWNED` + `DROP OWNED` to safely retire a role that owns objects.

## A7. Monitoring & Observability (8)

50. Read the cumulative statistics views: `pg_stat_activity`, `pg_stat_database`, `pg_stat_user_tables`.
51. Install `pg_stat_statements`; find the top queries by total and mean time.
52. Detect and diagnose lock waits via `pg_locks` + `pg_stat_activity` join; build a blocking-tree query.
53. Track table/index bloat with a bloat-estimate query; confirm against `pgstattuple`.
54. Monitor replication lag (bytes + seconds) from both primary and standby.
55. Set up Prometheus `postgres_exporter` + Grafana dashboard.
56. Configure `log_min_duration_statement` + pgBadger; generate an HTML report from real logs.
57. Watch checkpoint behavior (`pg_stat_bgwriter` / `pg_stat_checkpointer` in PG17); tune `checkpoint_*`.

## A8. Maintenance & Vacuum (7)

58. Force bloat, then reclaim with `VACUUM` vs `VACUUM FULL`; measure size + lock difference.
59. Tune autovacuum globally and per-table (`autovacuum_vacuum_scale_factor`, cost limits).
60. Simulate and detect transaction-ID wraparound risk; read `datfrozenxid`/`age()`.
61. `REINDEX CONCURRENTLY` a bloated index with zero downtime.
62. `pg_repack` to remove bloat online; compare to `VACUUM FULL`.
63. Schedule maintenance with `vacuumdb --all --analyze --jobs`; wire it into a systemd timer.
64. Detect corruption with `pg_amcheck`; recover using a good replica.

## A9. Partitioning & Large Data (5)

65. Range-partition a time-series table; attach/detach partitions.
66. List/hash partitioning; verify partition pruning in `EXPLAIN`.
67. Automate partition creation with `pg_partman` + a maintenance timer.
68. Sub-partitioning (range → list); test constraint exclusion.
69. Bulk-load 100M rows with `COPY`; compare load time indexed vs load-then-index.

## A10. Extensions (4)

70. Install core contrib extensions (`pg_stat_statements`, `pgcrypto`, `pg_trgm`, `hstore`); enable each.
71. `postgres_fdw`: query a remote PostgreSQL; `IMPORT FOREIGN SCHEMA`.
72. `file_fdw`: expose a CSV on disk as a table.
73. Deploy TimescaleDB (or Citus) on the native install; convert a table to a hypertable/distributed table.

## A11. Upgrade & Migration (3)

74. Minor-version upgrade (17.x → 17.y) via RPM; verify with `SELECT version()`.
75. Major-version upgrade with `pg_upgrade` (`--link` and copy modes); validate + `analyze` after.
76. Zero-downtime major upgrade using logical replication between old and new clusters.

## A12. Storage & Tablespaces (2)

77. Create a tablespace on a separate mount; move a table/index onto it; verify placement.
78. Move an entire database's default tablespace; confirm files relocated and old dir clean.

---

# TRACK B — DEVELOPER (42 labs)

## B1. Schema Design & Data Modeling (8)

79. Model a normalized OLTP schema (3NF) with PK/FK/unique/check constraints; test constraint violations.
80. Design surrogate vs natural keys; benchmark `bigint` identity vs `uuid` (v4 vs v7) as PK.
81. Deferrable constraints + `SET CONSTRAINTS`; insert a temporarily-inconsistent graph in one txn.
82. Exclusion constraints (`EXCLUDE USING gist`) to prevent overlapping bookings/ranges.
83. Generated columns (stored) + expression-based logic; verify recomputation on update.
84. Domains + composite types + enums; enforce a value set at the type level.
85. Model a hierarchy three ways (adjacency list, `ltree`, closure table); query each.
86. Temporal/versioned table design (valid-time ranges with `tstzrange` + GiST).

## B2. SQL Mastery (8)

87. Window functions: running totals, `LAG`/`LEAD`, `rank/dense_rank`, `ntile`, per-group top-N.
88. Recursive CTE: walk an org chart and a graph with cycle detection.
89. `GROUPING SETS`, `ROLLUP`, `CUBE` for multi-level aggregation.
90. `LATERAL` joins for per-row subqueries and top-N-per-group.
91. `MERGE` (PG15+) for upsert-with-delete; compare to `INSERT ... ON CONFLICT`.
92. `DISTINCT ON` vs window-function dedup; benchmark both.
93. Set operations (`UNION`/`INTERSECT`/`EXCEPT`) and `FILTER` clauses on aggregates.
94. `RETURNING` on INSERT/UPDATE/DELETE to avoid round-trips.

## B3. Indexing & Query Performance (8)

95. Read `EXPLAIN (ANALYZE, BUFFERS)`; identify seq scan vs index scan vs bitmap heap scan.
96. B-tree composite index column-order lab; prove leftmost-prefix rule.
97. Partial indexes for a hot subset (e.g. `WHERE status='active'`).
98. Expression index (e.g. `lower(email)`); make a query use it.
99. Covering index with `INCLUDE`; achieve an index-only scan (check `heap fetches`).
100. GIN index for `jsonb` and array containment; measure the speedup.
101. GiST/SP-GiST for geometric/range queries; BRIN for huge append-only tables.
102. Diagnose a bad plan from stale stats; fix with `ANALYZE` and tuned `default_statistics_target`.

## B4. Transactions & Concurrency (7)

103. Demonstrate all four isolation levels; reproduce dirty-read absence, non-repeatable read, phantom.
104. Force and observe a deadlock; read the deadlock report; fix with consistent lock ordering.
105. `SELECT ... FOR UPDATE` vs `FOR NO KEY UPDATE` vs `SKIP LOCKED` (queue pattern).
106. Serializable isolation: trigger a serialization failure and implement retry logic.
107. Advisory locks (`pg_advisory_lock`) for app-level mutexes.
108. Savepoints + partial rollback inside one transaction.
109. Long-transaction impact: hold one open and watch vacuum/bloat and `xmin` horizon stall.

## B5. Server-Side Programming (6)

110. PL/pgSQL functions: control flow, `RAISE`, exception blocks, `RETURNS TABLE`.
111. Triggers: `BEFORE`/`AFTER`, row vs statement, an audit-trail trigger.
112. `INSTEAD OF` triggers on a view to make it writable.
113. Stored procedures with in-procedure `COMMIT`/`ROLLBACK` (batch processing).
114. `LISTEN`/`NOTIFY` pub-sub; consume events from a client.
115. Set-returning functions + `LATERAL`; write a PL/Python or PL/pgSQL generator.

## B6. Modern Data Types & Search (5)

116. `jsonb` deep-dive: operators, `jsonb_path_query`, indexing, partial updates with `jsonb_set`.
117. Full-text search: `tsvector`/`tsquery`, ranking, a generated `tsvector` column + GIN index.
118. Fuzzy search with `pg_trgm` similarity + trigram index.
119. Vector search with `pgvector`: store embeddings, HNSW/IVFFlat index, ANN query (aligns with your PharmaGraph/pgvector work).
120. Range types + arrays: containment, overlap, `unnest`, aggregation.

---

# TRACK C — CROSS-CUTTING (18 labs)

## C1. Chaos & Failure Drills (7)

121. Kill `-9` the postmaster mid-write; confirm crash recovery replays WAL cleanly.
122. Fill the data disk to 100%; observe behavior, recover gracefully.
123. Corrupt a heap page on disk; detect via checksums; restore the block from a replica/backup.
124. Sever the primary↔standby network; measure lag, reconnection, slot WAL retention.
125. Exhaust connections (`max_connections`); prove the reserved superuser slots still let you in.
126. Simulate wraparound emergency (aggressive xid consumption in a test DB); resolve it.
127. Runaway query OOM: force a huge sort, watch the OOM killer, then fix with `work_mem`/limits.

## C2. Automation & IaC (6)

128. Ansible playbook: install + `initdb` + configure a cluster end-to-end, idempotently.
129. Bash provisioning script (your house style) to stand up primary+standby with slots.
130. systemd timers for backup, vacuum, and log-rotation jobs (no cron).
131. n8n workflow: alert on replication lag / failed backup / disk threshold.
132. Config drift check: script that diffs live `pg_settings` against a golden `postgresql.conf`.
133. Automated restore-test: nightly restore of last backup into a scratch cluster + `pg_verifybackup`.

## C3. Cloud & Scale (5)

134. Deploy PostgreSQL on an EC2/Azure VM with a separate EBS/managed-disk data volume.
135. Migrate a self-managed DB into RDS/Azure Database via logical replication (near-zero downtime).
136. Compare self-managed tuning vs a managed instance's defaults; document the gaps.
137. Read replica across regions; measure cross-region lag.
138. Cost/benchmark: `pgbench` TPS on self-hosted vs managed at matched vCPU/RAM.

---

# TRACK D — MIGRATION (34 labs)

The migration playbook is always the same five phases — **assess → schema convert → data load → code/app remediation → cutover & verify** — but each source engine breaks in its own way. Do D1 first (it's engine-agnostic), then the engine you actually face.

## D1. Migration Methodology & Tooling (6)

139. Stand up the tooling box: install `ora2pg`, AWS Schema Conversion Tool (SCT), `pgloader`, and `db_migrator` (Postgres extension); confirm each connects to a source.
140. Assessment lab: run `ora2pg --type SHOW_REPORT --estimate_cost` (and SCT assessment) against a source; read the effort/complexity score and produce a migration difficulty matrix.
141. Build a repeatable migration harness dir: `01-assess/ 02-schema/ 03-data/ 04-code/ 05-verify/` with logs per phase.
142. Row-count + checksum reconciliation lab: write a source-vs-target validator (per-table counts, aggregate checksums, sample-row diffs).
143. CDC concept lab: set up change-data-capture from a source into PostgreSQL for near-zero-downtime cutover (see engine tracks); measure replication lag.
144. Rollback plan lab: define and rehearse the abort-and-revert path before any real cutover.

## D2. PostgreSQL → PostgreSQL (older → newer) (6)

*(Consolidates and expands the old upgrade labs — homogeneous migration.)*

145. In-place `pg_upgrade` from PG13/14/15 → PG17 (`--copy` mode); `analyze` + validate after.
146. `pg_upgrade --link` (hard-link) mode for a huge cluster; understand the no-going-back tradeoff.
147. Logical-replication upgrade: replicate PG14 → PG17 live, then cutover with minimal downtime.
148. Dump/restore across a version gap (`pg_dump` from new binaries against old server); handle deprecated syntax.
149. Cross-architecture / cross-OS move (e.g. CentOS 7 PG11 → AlmaLinux 9 PG17) via logical replication.
150. Extension-version reconciliation: upgrade `postgis`/`pgvector`/etc. as part of the jump; `ALTER EXTENSION ... UPDATE`.

## D3. Oracle → PostgreSQL (8)

151. Seed an Oracle XE source schema (HR/OE sample) to migrate from.
152. `ora2pg` full schema conversion: tables, constraints, sequences, indexes; review the generated DDL.
153. Convert Oracle **PL/SQL packages/procedures/functions** to PL/pgSQL with `ora2pg`; hand-fix what it can't (autonomous txns, `CONNECT BY`, `DECODE`, `NVL`, `ROWNUM`).
154. Data-type mapping lab: `NUMBER`→`numeric`/`bigint`, `DATE`→`timestamp`, `VARCHAR2`, `CLOB`/`BLOB`→`text`/`bytea`; verify precision/edge values.
155. Migrate Oracle sequences + triggers-as-identity to PostgreSQL `GENERATED ... AS IDENTITY`.
156. Data load at scale with `ora2pg` (COPY mode + parallel `-j`); benchmark vs direct-path.
157. Rewrite Oracle-isms in app SQL: hierarchical `CONNECT BY` → recursive CTE, `(+)` outer joins → ANSI, `DUAL`, `MINUS`→`EXCEPT`, `SYSDATE`, analytic-function gaps.
158. Near-zero-downtime Oracle→PG cutover with CDC (Debezium/Oracle LogMiner or GoldenGate concept); reconcile and switch the app.

## D4. MySQL / MariaDB → PostgreSQL (7)

159. Seed a MySQL/MariaDB source (`sakila` or `world` sample DB).
160. One-shot migration with `pgloader`; read its summary report and error casts.
161. Data-type + default remediation: `tinyint(1)`→`boolean`, `datetime`/`ZERO_DATE` (`0000-00-00`) handling, `unsigned` ints, `enum`→ enum/domain, `AUTO_INCREMENT`→ identity.
162. Case-sensitivity & identifier-quoting lab: MySQL backticks and case-folding vs PostgreSQL folding rules; fix broken references.
163. Rewrite MySQL SQL dialect: `LIMIT x,y`→`LIMIT/OFFSET`, `IFNULL`→`COALESCE`, `GROUP_CONCAT`→`string_agg`, backtick removal, `ON UPDATE CURRENT_TIMESTAMP`→ trigger.
164. Engine/semantics gaps: `REPLACE INTO`/`INSERT IGNORE`→`ON CONFLICT`, implicit-commit DDL, storage-engine assumptions, silent truncation vs strict mode.
165. CDC migration with Debezium (MySQL binlog) → PostgreSQL for continuous sync + low-downtime cutover.

## D5. SQL Server (MSSQL) → PostgreSQL (7)

166. Seed a SQL Server source (`AdventureWorks` or `WideWorldImporters`).
167. Schema conversion with the `babelfish`/`db_migrator`/SCT path or `sqlserver2pgsql`; review generated DDL.
168. Deploy **Babelfish for PostgreSQL** (TDS + T-SQL compatibility) and test running the app against PG with minimal SQL changes; understand where it helps vs full conversion.
169. Data-type mapping: `DATETIME2`/`DATETIMEOFFSET`→`timestamptz`, `MONEY`→`numeric`, `UNIQUEIDENTIFIER`→`uuid`, `NVARCHAR`, `BIT`→`boolean`, `IMAGE`/`VARBINARY`→`bytea`.
170. T-SQL → PL/pgSQL rewrite: `IDENTITY`→ identity cols, `TOP n`→`LIMIT`, `ISNULL`→`COALESCE`, `GETDATE()`, `+` string concat→`||`, `[bracketed]` identifiers, temp tables `#t`, `MERGE` nuances.
171. Migrate SQL Server stored procedures, functions, and triggers; handle multi-result-set procs and `OUTPUT` params.
172. Low-downtime MSSQL→PG cutover with CDC (Debezium SQL Server connector); reconcile row counts + checksums, then switch.

---

# TRACK E — PRODUCTION READINESS (31 labs)

The senior/production gap. Everything above proves you can *build and operate* a cluster; this track proves you can *harden, secure, change safely, and survive an incident* on one that real users and auditors depend on. Do these once A–D are solid.

## E1. Enterprise Authentication & Secrets (5)

173. LDAP / Active Directory auth via `pg_hba.conf` (`ldap` method, search+bind mode); log in as a directory user.
174. Kerberos / GSSAPI single sign-on against a KDC/AD; passwordless `psql` with a ticket.
175. PAM and RADIUS auth methods; understand when each is the right enterprise fit.
176. Externalize credentials: HashiCorp Vault (or AWS Secrets Manager) issuing **dynamic, short-lived DB roles**; app pulls creds at runtime, no static password.
177. Password & session policy: raise `scram_iterations`, enforce complexity/expiry with `passwordcheck`/`credcheck`, set `idle_session_timeout` and `idle_in_transaction_session_timeout`.

## E2. Encryption at Rest & Data Protection (4)

178. Encrypt the data volume with LUKS/dm-crypt; measure the throughput overhead vs plaintext (PostgreSQL has no native TDE — this is how you get it).
179. Column-level encryption with `pgcrypto`; design where the key lives and the query-vs-security tradeoff.
180. Encrypted, immutable, offsite backups: pgBackRest → S3 with `--repo-cipher`, a retention policy, and object-lock; then restore from S3.
181. Non-prod data protection: mask/anonymize PII when refreshing a dev/staging DB from prod (custom masking or `anon` extension).

## E3. Zero-Downtime Schema Change Discipline (5)

182. Lock-safe DDL: always set `lock_timeout` + retry; build the "safe vs table-rewriting" matrix for `ALTER TABLE` (add column w/ volatile default, type change, `SET NOT NULL`, etc.).
183. Add a FK/check as `NOT VALID`, then `VALIDATE CONSTRAINT` separately to avoid a long lock.
184. `CREATE INDEX CONCURRENTLY` / `DROP INDEX CONCURRENTLY` under live write load; handle the failed-invalid-index case.
185. Versioned, repeatable migrations with Flyway (or Liquibase / sqitch) run in CI against a scratch cluster.
186. Expand-contract (blue-green schema) pattern + a large backfill done in throttled batches without bloat or replica lag.

## E4. Advanced Performance & Plan Management (5)

187. Wait-event analysis: read `pg_stat_activity.wait_event(_type)`, install `pg_wait_sampling` for an active-session-history view; find what queries actually wait on.
188. Extended statistics: `CREATE STATISTICS` on correlated columns; prove the row-estimate and plan improve.
189. Plan stability: use `pg_hint_plan` to pin a plan; build a regression test that catches a plan flip after a stats/version change.
190. Benchmark at scale with HammerDB (TPROC-C, TPC-C-like); compare findings to `pgbench` and record a repeatable baseline.
191. Autovacuum under stress: reproduce a freeze storm / aggressive-wraparound vacuum during heavy writes; tune cost limits and `vacuum_freeze_*` so it doesn't stall the app.

## E5. AlmaLinux / RHEL Production Landmines (4)

192. **glibc collation version change:** upgrade the OS (or glibc), then detect indexes silently broken by a changed sort order (`pg_collation` version mismatch); fix by `REINDEX`, and migrate to ICU or `C.UTF-8` collation to immunize against it. *This is the classic OS-upgrade data-corruption trap — practice it deliberately.*
193. SELinux in **enforcing** mode with a custom data dir, non-default port, and extra tablespace; write the `semanage`/policy-module fixes so PostgreSQL still starts.
194. Time sync: run `chrony`, then induce clock drift and observe the damage to replication timestamps, `pg_stat` timings, and log correlation.
195. Storage layer: xfs vs ext4, `noatime`, and the `full_page_writes` interaction with your storage's atomicity; benchmark and justify the choice.

## E6. HA Routing, DR & Operational Runbooks (5)

196. Put HAProxy (or pgpool-II) in front of Patroni for automatic read/write split + health-checked routing; kill the primary and confirm the app reconnects to the new leader.
197. Pooler HA: two PgBouncer instances behind keepalived/a VIP so the connection layer itself has no single point of failure.
198. Formal DR drill: documented **RTO/RPO** targets, cross-site failover *and failback*, with a sign-off checklist.
199. Restore-time SLA test: time a full PITR from cold/offsite storage end-to-end — does it actually meet your stated RTO? Most don't on the first try.
200. On-call runbook set: write step-by-step recovery docs for the top 5 incidents (disk full, xid wraparound, replication broken, connection storm, runaway query). These become NETAPORT articles.

## E7. Database Testing & Governance (3)

201. `pgTAP` unit tests for schema, constraints, and functions; run them in CI on every migration.
202. Data-quality audit: scripted checks for orphan rows, dup keys, and NULLs that violate intent; schedule and alert.
203. Access review: a least-privilege audit script (who can do what, which roles are superuser/bypass-RLS) producing compliance evidence — frame it for PCI/HIPAA-style controls.

---

# TRACK F — DEEP INTERNALS & SCALE ARCHITECTURE (19 labs)

What separates "runs it" from "understands why it behaves that way, and scales it." These are the internals and architecture labs a senior is expected to reason about under load.

## F1. Storage & Execution Internals Tuning (6)

204. TOAST deep-dive: storage strategies (`PLAIN`/`MAIN`/`EXTERNAL`/`EXTENDED`), `lz4` vs `pglz` compression (PG14+), and how oversized rows get sliced; measure size + read cost per strategy.
205. `fillfactor` + HOT updates: tune fillfactor on a hot table, prove the HOT-update ratio rises and index churn/bloat drops (`n_tup_hot_upd` in `pg_stat_user_tables`).
206. WAL tuning: `wal_compression`, `max_wal_size`/`min_wal_size`, `wal_buffers`; measure checkpoint frequency and write amplification under `pgbench`.
207. Parallel query: `max_parallel_workers(_per_gather)`, force and inspect parallel plans, then find the OLTP workload where parallelism *hurts* and cap it.
208. JIT compilation: toggle `jit` and its cost thresholds; measure the OLAP benefit vs the OLTP overhead on short queries.
209. Large objects: `lo` storage vs `bytea`, and cleaning orphaned LOs with `vacuumlo`.

## F2. Replica Read-Scaling & Conflicts (4)

210. Route read-only traffic to standbys; implement application-level read/write splitting and verify with `pg_stat_activity`.
211. `hot_standby_feedback` tradeoff: turn it on to stop replica query cancellations, then observe the bloat it causes on the primary — decide per workload.
212. Reproduce a **replica query conflict** (canceled by recovery); tune `max_standby_streaming_delay` and understand the lag-vs-cancellation choice.
213. Lag-aware read load-balancing across multiple replicas (HAProxy/pgpool health checks that pull a lagging replica out of rotation).

## F3. Multi-Tenancy & Horizontal Scale (5)

214. Build all three tenancy models on the same dataset — shared-schema + RLS, schema-per-tenant, database-per-tenant — and compare isolation, connection/ops overhead, and scaling ceiling.
215. Tenant lifecycle automation: provision a new tenant (schema + roles + defaults) and hard-delete one (GDPR-style right-to-erasure) with a single repeatable script.
216. Noisy-neighbor control: per-tenant `statement_timeout`, connection caps, and work_mem limits so one tenant can't starve the rest.
217. Citus: distribute a large table across worker nodes; write co-located joins and a distributed query; observe the shard placement.
218. Cross-tenant reporting/rollups over a sharded or RLS model **without breaking isolation** — the query pattern that most often leaks tenant data if done wrong.

## F4. Scheduling, Views & Traffic Realism (4)

219. `pg_cron`: schedule in-database maintenance/refresh jobs; compare operationally to the systemd-timer approach (labs 63, 130).
220. Materialized views in anger: `REFRESH MATERIALIZED VIEW CONCURRENTLY`, incremental/rollup refresh patterns, and the staleness-vs-cost tradeoff.
221. `pgreplay`: capture real production log traffic and replay it against a candidate config/version/hardware for a *realistic* regression test — far better signal than synthetic `pgbench`.
222. Event-driven pipeline: logical decoding → Kafka (or n8n) so downstream consumers react to row changes; measure end-to-end latency (ties into your automation work).

---

## What's Deliberately Out of Scope

So you know the boundary is a choice, not an oversight — these are real PostgreSQL topics left out because they're domain-specific or app-layer, not core senior-DBA/dev. Pull any in if your work needs it:

- **PostGIS / geospatial** — a full discipline of its own (spatial types, GiST/SP-GiST spatial indexes, tiling). Add a track only if you do mapping/location work.
- **Graph/time-series specializations** beyond the TimescaleDB lab (e.g. Apache AGE for graph).
- **App-layer caching** (Redis read-through, etc.) — architecture, not database.
- **Distributed 2PC/XA transactions** (`max_prepared_transactions`) — only if you run cross-database distributed commits.
- **OpenTelemetry / distributed tracing** into the DB — modern observability, app-integration heavy.

---

- **Order:** A1→A4 first (you can't do anything else without install + backup + replication), then B-track in parallel, C-track next. Do **D-track last** — heterogeneous migration assumes you already know how to run, tune, and recover the target cluster. Within D, always do D1 (methodology) before any engine-specific block. **E-track is the senior filter** — run it against everything you've built; it's what an auditor or a 3 a.m. page will actually test. **F-track is depth** — take it when you need to explain *why*, not just make it work, or when you hit a scale wall.
- **Migration reality check:** ~80% of the pain is never the data — it's stored procedures, dialect SQL, data-type edge cases (dates, nulls, unsigned/precision), and app code. Budget your lab time accordingly: the load runs in minutes, the code remediation takes weeks.
- **Discipline:** snapshot the VM, do the destructive step, prove recovery, snapshot again. A lab isn't done until you've *broken* it and recovered.
- **Evidence:** keep a `lab-NN/` dir per lab with the commands run, the `EXPLAIN`/log output, and a one-line "what I learned." That folder is your NETAPORT source material.

Want a companion `lab-tracker.csv` (lab #, track, title, status, notes) to check these off, or should I expand any single block into a full step-by-step runbook?
