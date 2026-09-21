# Lab 77 — Create a Tablespace on a Separate Mount; Move a Table/Index onto It; Verify Placement

> **Track A · DBA · A12 Storage & Tablespaces · Lab 1 of 2 (Lab 77/222)**
> Teaching package: concept → diagram → steps → verify → quick-ref → self-test → video script.
> **Depends on:** Lab 04 (volumes/SELinux), Lab 17 (tablespaces in dumps). Opens the storage track.

---

## 0. Lab Card (at a glance)

| Field | Value |
|---|---|
| **Objective** | Create a tablespace on a separate mount (correctly owned/labeled), move a table and index onto it, and verify their physical placement. |
| **Success criterion** | The tablespace exists on the target device; a moved table/index reports the new tablespace; `pg_tablespace_location()` confirms the path. |
| **Scope boundary** | Creating + placing on a tablespace. Moving a whole database's default tablespace is Lab 78. |
| **Prereqs** | Lab 04; a spare mount/directory; SELinux labeling |
| **Time** | 20–30 min |
| **Difficulty** | ★★★☆☆ |
| **Risk** | Low–Medium — moving an object takes an exclusive lock; do on a scratch table. |

---

## 1. Learning Objectives

1. **What a tablespace is** — a logical name for a physical location.
2. **Why use one** — put hot/cold data on different storage.
3. **Prep the directory** — ownership, empty, SELinux label.
4. **Place objects** — `CREATE`/`ALTER … SET TABLESPACE`, defaults.
5. **Verify + caveats** — placement, lock, backup implications.

---

## 2. Concept Primer — the "why"

**A tablespace maps a name to a directory.** By default all data lives under the data directory (`pg_default` tablespace). A **tablespace** lets you place specific tables and indexes in a **different physical location** — typically a **different disk/volume**. Uses: put **hot** tables/indexes on fast **SSD/NVMe** and **cold/archive** data on cheaper/larger storage, spread I/O across devices, or dedicate a volume to a heavy table.

**Prepare the directory first (Lab 04 rules apply):** the tablespace location must be a directory that is **owned by `postgres`**, **empty**, and — under Enforcing SELinux — labeled **`postgresql_db_t`** (`semanage fcontext` + `restorecon`), or the `CREATE TABLESPACE` (or the server) can't use it. PostgreSQL creates a **version-keyed subdirectory** inside it (so, as in Lab 17, two same-version clusters can't share one location).

**Create and place:**
```sql
CREATE TABLESPACE fast LOCATION '/mnt/fast/pgdata';
CREATE TABLE hot (...) TABLESPACE fast;          -- new object on the tablespace
ALTER TABLE existing SET TABLESPACE fast;         -- move an existing table
ALTER INDEX existing_idx SET TABLESPACE fast;     -- move an index (independently!)
```
A table and its indexes are placed **independently** — moving a table does **not** move its indexes; move each with its own `ALTER … SET TABLESPACE`.

**Defaults:**
- `default_tablespace` (per session/role/db) — where new objects go when none is specified.
- `temp_tablespaces` — where **temp** objects and sort/hash **spill** files go; point this at a fast volume to speed heavy sorts (and offload temp I/O from the data dir).

**Verify placement:** the table's `tablespace` (via `\d+` / `pg_class.reltablespace`), and the tablespace's path via **`pg_tablespace_location(oid)`**.

**Caveats — know these:**
- **Moving an object takes an `ACCESS EXCLUSIVE` lock** and **rewrites** it to the new location — blocking access for the duration (like `VACUUM FULL`, Lab 58). Plan a window for large tables.
- **Tablespaces complicate backups.** Physical backups (`pg_basebackup`, Lab 19) and `pg_dumpall -g` (Lab 17) must account for tablespace locations; on restore the location must exist (and may need remapping — `--tablespace-mapping`).
- A tablespace can't be dropped while it holds objects (move/drop them first).
- Tablespaces are **cluster-wide** but map to a **host path** — on a standby/replica the same path must exist (physical replication requires matching tablespace paths).

---

## 3. Diagrams

### 3.1 Create + move + verify flow

```mermaid
flowchart TD
    A["prepare dir on the mount: owned postgres · empty · SELinux postgresql_db_t"] --> B["CREATE TABLESPACE fast LOCATION '/mnt/fast/pgdata'"]
    B --> C["new object: CREATE TABLE hot (...) TABLESPACE fast"]
    B --> D["move existing: ALTER TABLE t SET TABLESPACE fast (ACCESS EXCLUSIVE)"]
    D --> E["move its index separately: ALTER INDEX t_idx SET TABLESPACE fast"]
    C & E --> F{verify}
    F -->|\\d+ / pg_class.reltablespace| G["object → tablespace 'fast'"]
    F -->|pg_tablespace_location(oid)| H["path on the mount"]
    G & H --> I([✔ placement confirmed])
```

### 3.2 Placement model

```mermaid
flowchart LR
    subgraph DEFAULT [pg_default (data dir)]
      T1["most tables/indexes"]
    end
    subgraph FAST [tablespace 'fast' → /mnt/fast (SSD)]
      T2["hot table + its index (moved independently)"]
    end
    subgraph TEMP [temp_tablespaces → /mnt/fast]
      T3["sort/hash spills + temp objects"]
    end
    note["dir: postgres-owned · empty · postgresql_db_t · move = ACCESS EXCLUSIVE rewrite · standby needs same path"]
```

---

## 4. Prerequisites

```bash
# a separate mount (Lab 04). Example: /mnt/fast (its own volume). Prepare the tablespace dir:
sudo mkdir -p /mnt/fast/pgdata
sudo chown -R postgres:postgres /mnt/fast/pgdata && sudo chmod 0700 /mnt/fast/pgdata
sudo semanage fcontext -a -t postgresql_db_t "/mnt/fast/pgdata(/.*)?" 2>/dev/null || true
sudo restorecon -Rv /mnt/fast/pgdata
df -h /mnt/fast     # confirm it's the intended device
```

---

## 5. Step-by-Step

### Step 1 — Create the tablespace

```bash
sudo -u postgres psql -c "CREATE TABLESPACE fast LOCATION '/mnt/fast/pgdata';"
sudo -u postgres psql -c "SELECT spcname, pg_tablespace_location(oid) AS location FROM pg_tablespace;"
```

### Step 2 — New object directly on the tablespace

```bash
sudo -u postgres psql -d benchdb <<'SQL'
CREATE TABLE hot_data (id bigserial PRIMARY KEY, v text) TABLESPACE fast;
CREATE INDEX hot_data_v_idx ON hot_data(v) TABLESPACE fast;
INSERT INTO hot_data (v) SELECT md5(g::text) FROM generate_series(1,10000) g;
SQL
```

### Step 3 — Move an EXISTING table (and its index) onto it

```bash
sudo -u postgres psql -d benchdb <<'SQL'
CREATE TABLE IF NOT EXISTS to_move AS SELECT g id, md5(g::text) v FROM generate_series(1,50000) g;
CREATE INDEX to_move_id ON to_move(id);
-- ACCESS EXCLUSIVE lock while it rewrites:
ALTER TABLE to_move SET TABLESPACE fast;
ALTER INDEX to_move_id SET TABLESPACE fast;     -- index moves INDEPENDENTLY
SQL
```

### Step 4 — Verify placement

```bash
sudo -u postgres psql -d benchdb -c "
SELECT c.relname, c.relkind,
       COALESCE(t.spcname, 'pg_default') AS tablespace
FROM pg_class c LEFT JOIN pg_tablespace t ON c.reltablespace = t.oid
WHERE c.relname IN ('hot_data','hot_data_v_idx','to_move','to_move_id') ORDER BY relname;"
# and the physical path:
sudo -u postgres psql -c "SELECT spcname, pg_tablespace_location(oid) FROM pg_tablespace WHERE spcname='fast';"
sudo -u postgres psql -d benchdb -c "\d+ hot_data" | grep -i tablespace
```

### Step 5 — (Optional) point temp spills at the fast volume

```bash
sudo -u postgres psql -c "ALTER SYSTEM SET temp_tablespaces = 'fast'; SELECT pg_reload_conf();"
sudo -u postgres psql -c "SHOW temp_tablespaces;"
# now sort/hash spills and temp objects use the fast tablespace
```

### Step 6 — See the files land on the device

```bash
sudo ls -l /mnt/fast/pgdata/                        # a PG_<ver>_<catver> subdir
sudo du -sh /mnt/fast/pgdata/*                       # data now on the separate mount
```

---

## 6. Verification Checklist

- [ ] Tablespace directory: postgres-owned, empty, `postgresql_db_t`
- [ ] `CREATE TABLESPACE` succeeded; `pg_tablespace_location` shows the mount
- [ ] A new table/index created directly on the tablespace
- [ ] An existing table **and its index** moved (index moved separately)
- [ ] `pg_class.reltablespace` / `\d+` confirm placement
- [ ] Files present on the separate device (`/mnt/fast`)
- [ ] (Optional) `temp_tablespaces` set to the fast volume

---

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `CREATE TABLESPACE`: permission denied / not empty | Dir perms/SELinux/not empty | postgres-own, `chmod 0700`, empty, `postgresql_db_t` (Lab 04) |
| Index still on `pg_default` after moving table | Indexes move independently | `ALTER INDEX … SET TABLESPACE` too |
| Move blocks the table | `ACCESS EXCLUSIVE` rewrite | Plan a window for large tables |
| Can't drop the tablespace | It still holds objects | Move/drop objects first |
| Standby fails / path missing | Tablespace path must exist on replica | Create matching path on the standby |
| Backup/restore fails on tablespace | Location must exist / remap | `--tablespace-mapping` on restore (Lab 17) |
| Server won't start after mount gone | Tablespace mount not present | Ensure the mount is available at boot (`RequiresMountsFor`, Lab 04) |

---

## 8. Quick Reference Card (paste-ready)

```bash
# prepare dir (Lab 04): postgres-owned · empty · SELinux postgresql_db_t
sudo mkdir -p /mnt/fast/pgdata && sudo chown -R postgres:postgres /mnt/fast/pgdata && sudo chmod 0700 /mnt/fast/pgdata
sudo semanage fcontext -a -t postgresql_db_t "/mnt/fast/pgdata(/.*)?" && sudo restorecon -Rv /mnt/fast/pgdata
```
```sql
CREATE TABLESPACE fast LOCATION '/mnt/fast/pgdata';
CREATE TABLE hot (...) TABLESPACE fast;              -- new object here
ALTER TABLE t   SET TABLESPACE fast;                 -- move existing (ACCESS EXCLUSIVE)
ALTER INDEX t_i SET TABLESPACE fast;                 -- indexes move SEPARATELY
-- defaults: SET default_tablespace='fast';  ALTER SYSTEM SET temp_tablespaces='fast';

-- verify: SELECT spcname, pg_tablespace_location(oid) FROM pg_tablespace;
--         SELECT relname, reltablespace::regclass FROM pg_class WHERE relname='t';   (\d+ t)

-- caveats: dir postgres-owned/empty/labeled · move = exclusive rewrite · standby needs same path · backups must remap
```

---

## 9. Self-Check

1. What is a tablespace, and why use one?
2. What must be true of the tablespace directory before `CREATE TABLESPACE`?
3. When you move a table with `SET TABLESPACE`, do its indexes move too?
4. What lock does moving an object take, and what's the implication?
5. How do you verify an object's tablespace and its physical path?
6. Name a backup/replication caveat of tablespaces.

<details>
<summary>Answers</summary>

1. A named mapping to a physical directory, letting you place tables/indexes on different storage (e.g. hot data on SSD).
2. Owned by `postgres`, **empty**, and (Enforcing SELinux) labeled `postgresql_db_t`.
3. **No** — indexes move independently; `ALTER INDEX … SET TABLESPACE` each.
4. `ACCESS EXCLUSIVE` — it rewrites the object, blocking access for the duration (plan a window).
5. `pg_class.reltablespace` (or `\d+`) for the object; `pg_tablespace_location(oid)` for the path.
6. Physical backups/`pg_dumpall -g` record tablespace locations; on restore the path must exist or be remapped (`--tablespace-mapping`), and a standby needs the same path.
</details>

---

## 10. Video / Teaching Script

| # | On screen | Say (narration) |
|---|---|---|
| 1 | Title: "Put your hot data on faster disk" | "By default everything lives in one directory. Tablespaces let you put specific tables on a different, faster disk." |
| 2 | prepare dir | "Same rules as any Postgres directory: owned by postgres, empty, and SELinux-labeled — or it won't work." |
| 3 | CREATE TABLESPACE + place | "Create the tablespace, then create a table right on it. Hot data on the SSD." |
| 4 | move existing + index | "Move an existing table too — but note: indexes don't follow. Move each one yourself." |
| 5 | verify | "Confirm placement — the catalog shows the tablespace, and this function shows the actual path on disk." |
| 6 | caveats | "Two warnings: moving a table locks it exclusively while it rewrites, and tablespaces complicate backups and standbys — the path has to exist everywhere." |
| 7 | Outro | "Data placed by design. Next: moving a whole database's default tablespace." |

---

## 11. Glossary

- **Tablespace** — a named mapping to a physical directory.
- **`pg_default`** — the default tablespace (data dir).
- **`CREATE TABLESPACE … LOCATION`** — define one on a path.
- **`SET TABLESPACE`** — place/move a table or index (exclusive lock).
- **`default_tablespace` / `temp_tablespaces`** — new-object / temp-spill placement.
- **`pg_tablespace_location()`** — the directory a tablespace maps to.
- **Version-keyed subdir** — PostgreSQL's per-version folder inside the location.

---
*NETAPORT lab series · PostgreSQL 17 on AlmaLinux 9 · Lab 77/222 · A12 Storage & Tablespaces*
