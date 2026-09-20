# PostgreSQL 17 — Senior DBA + Developer Lab Curriculum (AlmaLinux 9)

Hands-on labs to deploy, break, test, and master. Native installs only (systemd / RPM / binaries) — **no Docker**. Every lab is a concrete deployable exercise with a success criterion.

## Count Summary

| Track | Domain blocks | Labs |
|---|---|---|
| DBA | 12 | 78 |
| Developer | 6 | 42 |
| Cross-cutting (chaos / cloud / automation) | 3 | 18 |
| **Total** | **21** | **138** |

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

## How to Work Through This

- **Order:** A1→A4 first (you can't do anything else without install + backup + replication), then B-track in parallel, C-track last.
- **Discipline:** snapshot the VM, do the destructive step, prove recovery, snapshot again. A lab isn't done until you've *broken* it and recovered.
- **Evidence:** keep a `lab-NN/` dir per lab with the commands run, the `EXPLAIN`/log output, and a one-line "what I learned." That folder is your NETAPORT source material.

Want a companion `lab-tracker.csv` (lab #, track, title, status, notes) to check these off, or should I expand any single block into a full step-by-step runbook?
