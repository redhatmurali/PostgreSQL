# PostgreSQL 17 DBA Command Cheatsheet (Linux / AlmaLinux 9)

Scoped to a working PostgreSQL 17 DBA on RHEL-family Linux. Four command layers.

## Count Summary

| Layer | What | Enumerated | Full set |
|---|---|---|---|
| 1 | PostgreSQL Linux binaries | 35 core (+2 contrib) | 35 |
| 2 | Linux/systemd & OS commands | ~25 | n/a |
| 3 | psql meta-commands (DBA subset) | ~70 | ~130 via `\?` |
| 4 | SQL DBA statements | ~90 | ~180 via SQL index |

**Grand total, DBA-relevant enumerated: ~220 commands.**

Paths on AlmaLinux 9: binaries in `/usr/pgsql-17/bin/` · data dir `/var/lib/pgsql/17/data` · service unit `postgresql-17`.

---

## 1. PostgreSQL Linux Binaries (35 core)

### Server / cluster admin (19)

| Binary | Purpose |
|---|---|
| `postgres` | The database server itself |
| `initdb` | Create a new database cluster |
| `pg_ctl` | Start/stop/restart/reload/status a cluster directly |
| `pg_controldata` | Show cluster control info (WAL, checkpoint, version) |
| `pg_resetwal` | Reset the WAL (last-resort recovery) |
| `pg_rewind` | Resync a diverged standby without full base backup |
| `pg_upgrade` | Major-version in-place upgrade |
| `pg_checksums` | Enable/disable/verify data checksums (offline) |
| `pg_archivecleanup` | Remove archived WAL no longer needed |
| `pg_waldump` | Decode/inspect WAL segment contents |
| `pg_walsummary` † | Inspect WAL summary files (incremental backup) |
| `pg_test_fsync` | Benchmark fsync methods for `wal_sync_method` tuning |
| `pg_test_timing` | Measure timing overhead |
| `pg_basebackup` | Take a physical base backup / clone standby |
| `pg_receivewal` | Stream WAL to disk (archiving) |
| `pg_recvlogical` | Consume a logical replication/decoding stream |
| `pg_verifybackup` | Verify integrity of a base backup |
| `pg_combinebackup` † | Reconstruct full backup from incremental chain |
| `pg_createsubscriber` † | Convert a physical standby into a logical subscriber |

### Client applications (16)

| Binary | Purpose |
|---|---|
| `psql` | Interactive terminal / script runner |
| `pg_dump` | Dump a single database |
| `pg_dumpall` | Dump the whole cluster + globals (roles, tablespaces) |
| `pg_restore` | Restore from custom/directory/tar dump |
| `createdb` / `dropdb` | Create / drop a database (CLI wrappers) |
| `createuser` / `dropuser` | Create / drop a role (CLI wrappers) |
| `clusterdb` | CLUSTER all tables in a DB |
| `reindexdb` | REINDEX a DB from CLI |
| `vacuumdb` | VACUUM/ANALYZE from CLI (supports parallel/all-DB) |
| `pg_isready` | Check if server is accepting connections |
| `pgbench` | Benchmarking / load generation |
| `pg_config` | Show build/install configuration paths |
| `pg_amcheck` | Verify heap/btree corruption |
| `ecpg` | Embedded-SQL C preprocessor |

† New in PostgreSQL 17.

**Contrib (+2, from `postgresql17-contrib`):** `oid2name`, `vacuumlo`.

---

## 2. Linux / systemd + OS Commands (~25)

### Service control (service = `postgresql-17`)

```bash
systemctl start   postgresql-17
systemctl stop    postgresql-17
systemctl restart postgresql-17
systemctl reload  postgresql-17          # SIGHUP: reload config, no downtime
systemctl status  postgresql-17
systemctl enable  postgresql-17          # start on boot
systemctl disable postgresql-17
/usr/pgsql-17/bin/postgresql-17-setup initdb    # one-time cluster init
```

### Logs / inspection

```bash
journalctl -u postgresql-17 -f
tail -f /var/lib/pgsql/17/data/log/*.log
ps -ef | grep postgres
ss -tlnp | grep 5432                      # or: netstat -tlnp
```

### Access / privilege

```bash
su - postgres
sudo -u postgres psql
```

### Resources / tuning

```bash
df -h /var/lib/pgsql/17/data              # disk free
du -sh /var/lib/pgsql/17/data/base        # DB size on disk
free -m ; top ; htop
sysctl -w kernel.shmmax=...  vm.overcommit_memory=2   # persist in /etc/sysctl.conf
ulimit -n                                 # open-file limit
getenforce                                # SELinux state (matters on AlmaLinux)
```

**Config files** (all in the data dir): `postgresql.conf`, `pg_hba.conf`, `pg_ident.conf`.

---

## 3. psql Meta-Commands — DBA Subset (~70)

### Connection / session (14)

```
\c  \connect        connect to another DB/host/user
\conninfo           show current connection details
\q                  quit
\password [role]    set/change a role password securely
\!                  run a shell command
\cd                 change psql working directory
\setenv             set an env var for subprocesses
\timing             toggle statement timing
\watch [sec]        re-run last query on an interval
\prompt             prompt for a variable
\echo  \qecho  \warn   print to stdout / query output / stderr
```

### Object listing — `\d` family (30)

```
\d [name]           describe table/view/index/sequence
\d+                 …with sizes, storage, comments
\dt \dt+            list tables
\di                 list indexes
\ds                 list sequences
\dv                 list views
\dm                 list materialized views
\df                 list functions
\dp  \z             list access privileges
\dn                 list schemas
\du  \dg            list roles
\dx                 list installed extensions
\db                 list tablespaces
\dl                 list large objects
\dD                 list domains
\dc                 list conversions
\dC                 list casts
\dT                 list data types
\do                 list operators
\dO                 list collations
\da                 list aggregates
\dF                 list text-search configs
\dy                 list event triggers
\dRp                list publications
\dRs                list subscriptions
\l  \l+             list databases
```

### Source / edit (5)

```
\sf   show function source
\sv   show view source
\e    edit query buffer in $EDITOR
\ef   edit a function
\ev   edit a view
```

### Query buffer / execution (10)

```
\g    execute (optionally to file/command)
\gx   execute with expanded output
\gexec  run each result row as SQL
\gset   store result into psql variables
\p    show buffer
\r    reset buffer
\w    write buffer to file
\i    run SQL file
\ir   run SQL file relative to current script
\o    send query output to file/pipe
```

### Output formatting (11)

```
\x    toggle expanded display
\a    toggle aligned/unaligned
\H    toggle HTML output
\t    toggle tuples-only (no header/footer)
\pset control any output option
\f    field separator
\C    table title
\s    command history
\set \unset   psql variables
\copy  client-side COPY (no server file access needed)
```

Run `\?` for the full list (~130), `\h` for SQL syntax help.

---

## 4. SQL DBA Statements (~90)

### Roles / privileges / security (13)

```sql
CREATE ROLE / ALTER ROLE / DROP ROLE
GRANT / REVOKE
SET ROLE / RESET ROLE
ALTER DEFAULT PRIVILEGES
CREATE POLICY / ALTER POLICY / DROP POLICY   -- row-level security
SECURITY LABEL
REASSIGN OWNED / DROP OWNED                   -- before dropping a role
```

### Databases / schemas / tablespaces (9)

```sql
CREATE DATABASE / ALTER DATABASE / DROP DATABASE
CREATE SCHEMA / ALTER SCHEMA / DROP SCHEMA
CREATE TABLESPACE / ALTER TABLESPACE / DROP TABLESPACE
```

### Configuration / cluster control (9)

```sql
ALTER SYSTEM ...          -- writes postgresql.auto.conf
SET / RESET / SHOW        -- session GUCs
LOAD                      -- load a shared library
DISCARD                   -- reset session state
CHECKPOINT
SELECT pg_reload_conf();
SELECT pg_terminate_backend(pid);   -- and pg_cancel_backend(pid)
```

### Maintenance (6)

```sql
VACUUM
VACUUM FULL
ANALYZE
REINDEX
CLUSTER
TRUNCATE
```

### Backup / WAL / replication (12)

```sql
CREATE PUBLICATION / ALTER PUBLICATION / DROP PUBLICATION
CREATE SUBSCRIPTION / ALTER SUBSCRIPTION / DROP SUBSCRIPTION
SELECT pg_switch_wal();
SELECT pg_backup_start('label');  SELECT pg_backup_stop();
SELECT pg_create_physical_replication_slot('slot');
SELECT pg_create_logical_replication_slot('slot','pgoutput');
SELECT pg_promote();
SELECT pg_wal_replay_pause();  SELECT pg_wal_replay_resume();
```

### Extensions / foreign data (9)

```sql
CREATE EXTENSION / ALTER EXTENSION / DROP EXTENSION
CREATE FOREIGN DATA WRAPPER / ALTER ... / DROP ...
CREATE SERVER
CREATE USER MAPPING
IMPORT FOREIGN SCHEMA
```

### Transactions / locking / async (10)

```sql
BEGIN / START TRANSACTION
COMMIT / ROLLBACK
SAVEPOINT
SET TRANSACTION
LOCK
LISTEN / NOTIFY / UNLISTEN
```

### Core DDL (12)

`CREATE / ALTER / DROP` applied to:
`TABLE`, `INDEX`, `VIEW`, `MATERIALIZED VIEW`, `SEQUENCE`, `FUNCTION`, `PROCEDURE`, `TRIGGER`, `TYPE`, `DOMAIN`
plus `COMMENT` and `COPY`.

In psql, run `\h <COMMAND>` for exact syntax of any statement. The full SQL reference index lists ~180 statements (many are query-side, not DBA).

---

## Quick DBA Daily Reference

```bash
# Is it up?
pg_isready -h localhost -p 5432
sudo -u postgres psql -c "SELECT version();"

# Who's connected / what's running?
sudo -u postgres psql -c "SELECT pid, usename, state, query FROM pg_stat_activity WHERE state <> 'idle';"

# Database sizes
sudo -u postgres psql -c "SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database ORDER BY 2 DESC;"

# Backup one DB / whole cluster
pg_dump -Fc -d mydb -f /backup/mydb.dump
pg_dumpall -g -f /backup/globals.sql          # roles + tablespaces only

# Reload config after editing postgresql.conf / pg_hba.conf
systemctl reload postgresql-17

# Manual vacuum+analyze all DBs, parallel
sudo -u postgres vacuumdb --all --analyze --jobs=4
```
