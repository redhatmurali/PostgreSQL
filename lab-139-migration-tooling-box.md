# Lab 139 — Stand Up the Tooling Box: `ora2pg`, AWS SCT, `pgloader`, `db_migrator`; Confirm Each Connects to a Source

> **Track D · Migration · D1 Tooling & Assessment · Lab 1 of 8 (Lab 139/222)**
> Teaching package: concept → diagram → steps → verify → quick-ref → self-test → video script.
> **Depends on:** Lab 01 (install), Lab 71 (FDW). Opens the migration track.

---

## 0. Lab Card (at a glance)

| Field | Value |
|---|---|
| **Objective** | Install ora2pg, AWS SCT, pgloader, and db_migrator on an AlmaLinux 9 tooling box, and confirm each can connect to a source database. |
| **Success criterion** | All four tools install; each connects to a test source (Oracle/MySQL/MSSQL) and returns a version/handshake. |
| **Scope boundary** | Tooling setup + connectivity. Actual migrations start Lab 140. |
| **Prereqs** | AlmaLinux 9 host; access to a source DB; Oracle Instant Client (for Oracle) |
| **Time** | 40–60 min |
| **Difficulty** | ★★★★☆ |
| **Risk** | Low — installs + read-only connects. |

---

## 1. Learning Objectives

1. **The four tools** and their strengths.
2. **Common prereqs** — Instant Client, JDBC, FDW.
3. **Install** each on AlmaLinux 9.
4. **Connect-test** each to a source.
5. **Choose** the right tool per source/goal.

---

## 2. Concept Primer — the "why"

**A heterogeneous migration needs a *toolbox*, not one tool** — different sources and goals call for different tools. Four cover the field:

- **`ora2pg`** — the **Oracle → PostgreSQL** gold standard (also MySQL). **Perl**-based; converts **schema, data, and PL/SQL → PL/pgSQL** (procedures, packages, triggers, views), and produces an **assessment report** (migration cost/complexity). Connects via **`DBD::Oracle`** (needs **Oracle Instant Client**) or `DBD::mysql`, configured in `ora2pg.conf`.
- **`pgloader`** — a fast **data-loading + migration** tool (Common **Lisp**). Best for **MySQL / SQLite / MSSQL / CSV → PostgreSQL** — schema + data + type mapping in **one command**. Simpler than ora2pg for MySQL; lighter code conversion. Connects via connection strings (`mysql://`, `mssql://`).
- **AWS Schema Conversion Tool (SCT)** — **Java** desktop/CLI tool for **multi-source schema + code conversion** with a **complexity assessment** (flags manual-conversion items); pairs with **AWS DMS** for the data. Connects via **JDBC** (Oracle/MSSQL/MySQL drivers). *(AWS is succeeding SCT with the managed **DMS Schema Conversion** — but SCT is still widely used; note it's a **GUI** app, run on a workstation or via CLI.)*
- **`db_migrator`** — a **PostgreSQL extension** (Cybertec) that runs the migration **inside PostgreSQL**, pulling from the source through a **foreign data wrapper** (`oracle_fdw` for Oracle, `mysql_fdw`, `tds_fdw` for MSSQL). Good for **FDW-based, incremental** migration.

**The common prereqs:**
- **Oracle sources** → **Oracle Instant Client** (basic + devel), required by **`DBD::Oracle`** (ora2pg) *and* **`oracle_fdw`** (db_migrator). Set `ORACLE_HOME`/`LD_LIBRARY_PATH`.
- **SCT** → **Java (JDK)** + source **JDBC drivers** (ojdbc, mssql-jdbc, connector-j) placed in SCT's drivers directory.
- **db_migrator** → the matching **FDW** extension + its client library (Instant Client / FreeTDS for MSSQL).
- **pgloader** → from **EPEL** or built with **SBCL**; MSSQL needs **FreeTDS**.

**Choosing:**
- **Oracle, comprehensive (schema + PL/SQL + assessment)** → **ora2pg**.
- **MySQL/CSV, fast data + simple schema** → **pgloader**.
- **Multi-source assessment + AWS/DMS pipeline** → **AWS SCT** (+ DMS).
- **Oracle via FDW / incremental inside PostgreSQL** → **db_migrator**.
Often you use **more than one** — SCT/ora2pg to *assess and convert schema/code*, pgloader/DMS/db_migrator to *move data*.

---

## 3. Diagrams

### 3.1 Tooling-box setup flow

```mermaid
flowchart TD
    A["AlmaLinux 9 tooling box"] --> B["prereqs: EPEL · Perl · Java(JDK) · Oracle Instant Client · FDW/FreeTDS · JDBC drivers"]
    B --> C["ora2pg: Perl + DBD::Oracle/DBD::mysql + ora2pg"]
    B --> D["pgloader: EPEL/SBCL"]
    B --> E["AWS SCT: Java + SCT + JDBC drivers"]
    B --> F["db_migrator: PG extension + oracle_fdw/mysql_fdw/tds_fdw"]
    C --> G["connect-test: ora2pg -t SHOW_VERSION"]
    D --> H["connect-test: pgloader dry-run / --version"]
    E --> I["connect-test: SCT connect via JDBC"]
    F --> J["connect-test: query source through the FDW"]
    G & H & I & J --> K([✔ toolbox ready])
```

### 3.2 Tool selection

```mermaid
flowchart LR
    subgraph TOOLS [migration toolbox]
      O["ora2pg: Oracle/MySQL · schema+PL/SQL+data · assessment (Perl, Instant Client)"]
      P["pgloader: MySQL/CSV/MSSQL · fast data + simple schema (Lisp)"]
      S["AWS SCT: multi-source schema+code + assessment (Java/JDBC, +DMS)"]
      D2["db_migrator: Oracle→PG via FDW · inside PostgreSQL (extension)"]
    end
    note["Oracle → Instant Client (DBD::Oracle + oracle_fdw) · SCT → JDBC · db_migrator → FDW · often use several together"]
```

---

## 4. Prerequisites — base packages

```bash
sudo dnf install -y epel-release
sudo dnf install -y perl perl-DBI perl-DBD-MySQL perl-Time-HiRes make gcc gcc-c++ \
                    java-17-openjdk freetds unzip wget
# Oracle Instant Client (download from Oracle — free, accept license): basic + devel + sqlplus
#   sudo dnf install -y oracle-instantclient-basic oracle-instantclient-devel oracle-instantclient-sqlplus
export ORACLE_HOME=/usr/lib/oracle/21/client64
export LD_LIBRARY_PATH=$ORACLE_HOME/lib
echo "$ORACLE_HOME/lib" | sudo tee /etc/ld.so.conf.d/oracle.conf && sudo ldconfig
```

---

## 5. Step-by-Step

### Step 1 — ora2pg (Perl + DBD::Oracle)

```bash
# DBD::Oracle (needs Instant Client + env above):
sudo -E cpan -T DBD::Oracle 2>/dev/null || echo "install DBD::Oracle via cpan with ORACLE_HOME set"
# ora2pg:
sudo dnf install -y ora2pg 2>/dev/null || {
  wget -q https://github.com/darold/ora2pg/archive/refs/tags/v24.3.tar.gz -O ora2pg.tgz
  tar xf ora2pg.tgz && cd ora2pg-* && perl Makefile.PL && make && sudo make install && cd - ; }
ora2pg --version
```

### Step 2 — pgloader

```bash
sudo dnf install -y pgloader 2>/dev/null || {
  echo "not in repo → build with SBCL: dnf install -y sbcl freetds-devel; make from source"; }
pgloader --version
```

### Step 3 — AWS SCT (Java + JDBC drivers)

```bash
java -version    # JDK present
# download SCT + JDBC drivers from AWS, then:
#   sudo dnf install -y ./aws-schema-conversion-tool-*.rpm
# place source JDBC drivers where SCT expects them (Settings → Drivers):
#   ojdbc (Oracle), mssql-jdbc (MSSQL), mysql-connector-j (MySQL)
echo "SCT is a GUI app (run on a workstation/X) — or use AWS DMS Schema Conversion (managed). CLI mode exists."
```

### Step 4 — db_migrator (PG extension + oracle_fdw)

```bash
# on the tooling box's PostgreSQL 17:
sudo dnf install -y oracle_fdw_17 2>/dev/null || echo "install/build oracle_fdw (needs Instant Client)"
# db_migrator (from source):
wget -q https://github.com/cybertec-postgresql/db_migrator/archive/refs/heads/master.tar.gz -O dbm.tgz
tar xf dbm.tgz && cd db_migrator-* && sudo make install && cd -
sudo -u postgres psql -c "CREATE EXTENSION IF NOT EXISTS oracle_fdw; CREATE EXTENSION IF NOT EXISTS db_migrator;"
sudo -u postgres psql -c "\dx" | grep -E "oracle_fdw|db_migrator"
```

### Step 5 — Connect-test each tool to a source

```bash
# ora2pg → Oracle (configure ora2pg.conf: ORACLE_DSN/USER/PWD), then:
cat > /tmp/ora2pg.conf <<'EOF'
ORACLE_DSN  dbi:Oracle:host=ORA_HOST;sid=ORCL;port=1521
ORACLE_USER miguser
ORACLE_PWD  migpass
EOF
ora2pg -t SHOW_VERSION -c /tmp/ora2pg.conf     # → prints the Oracle version = connected

# pgloader → MySQL (dry-run parses the connection):
pgloader --dry-run mysql://miguser:migpass@MYSQL_HOST/sourcedb postgresql://postgres@localhost/benchdb 2>&1 | tail -3

# db_migrator → Oracle via oracle_fdw:
sudo -u postgres psql -d benchdb <<'SQL'
CREATE SERVER IF NOT EXISTS ora FOREIGN DATA WRAPPER oracle_fdw OPTIONS (dbserver '//ORA_HOST:1521/ORCL');
CREATE USER MAPPING IF NOT EXISTS FOR postgres SERVER ora OPTIONS (user 'miguser', password 'migpass');
SELECT oracle_diag('ora');    -- oracle_fdw diagnostic → confirms the connection
SQL
# AWS SCT → connect via JDBC in the GUI/CLI to the source (Oracle/MSSQL) and run an assessment.
```

### Step 6 — Record the toolbox inventory

```bash
{ echo "ora2pg:     $(ora2pg --version 2>&1)"
  echo "pgloader:   $(pgloader --version 2>&1 | head -1)"
  echo "java(SCT):  $(java -version 2>&1 | head -1)"
  echo "db_migrator/oracle_fdw:"; sudo -u postgres psql -tAc "SELECT extname FROM pg_extension WHERE extname IN ('db_migrator','oracle_fdw');"
} | tee /tmp/migration-toolbox.txt
```

---

## 6. Verification Checklist

- [ ] Oracle Instant Client installed; `ldconfig` set (for Oracle sources)
- [ ] `ora2pg --version` works; `SHOW_VERSION` connects to Oracle
- [ ] `pgloader --version` works; dry-run parses a source
- [ ] SCT (Java + JDBC drivers) installed / plan documented
- [ ] `db_migrator` + `oracle_fdw` extensions created; FDW connects
- [ ] Each tool connected to a source (version/handshake)
- [ ] Toolbox inventory recorded

---

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `DBD::Oracle` build fails | Instant Client / env missing | Install basic+devel; set `ORACLE_HOME`/`LD_LIBRARY_PATH`; `ldconfig` |
| ora2pg can't connect | DSN/creds/listener | Verify `ora2pg.conf`, Oracle listener, network |
| `pgloader` not in repo | Not packaged | EPEL, or build with SBCL |
| SCT needs a GUI | Desktop app | Run on a workstation, or use DMS Schema Conversion; drivers in the right dir |
| `oracle_fdw` load error | Instant Client not found | Install + `ldconfig` |
| `db_migrator` extension missing | Not built | `make install` from source |
| MSSQL connect fails | No FreeTDS/tds_fdw | Install FreeTDS; `tds_fdw` / mssql JDBC / `mssql://` |

---

## 8. Quick Reference Card (paste-ready)

```bash
# TOOLBOX (AlmaLinux 9): common prereqs = EPEL · Perl · JDK · Oracle Instant Client · FreeTDS · JDBC drivers · FDWs
# ora2pg    (Oracle/MySQL · schema+PL/SQL+data · assessment):  perl + DBD::Oracle + ora2pg   → ora2pg -t SHOW_VERSION -c conf
# pgloader  (MySQL/CSV/MSSQL · fast data):                     dnf/EPEL or SBCL              → pgloader --dry-run mysql://... postgresql://...
# AWS SCT   (multi-source schema+code assessment, +DMS):       JDK + SCT rpm + JDBC drivers  → connect via JDBC (GUI/CLI)
# db_migrator (Oracle→PG via FDW, inside PG):                  oracle_fdw + db_migrator ext  → oracle_diag('server') via oracle_fdw

# choose: Oracle+code → ora2pg · MySQL/CSV data → pgloader · assessment+DMS → SCT · FDW/incremental → db_migrator
# Oracle sources ALWAYS need Oracle Instant Client (DBD::Oracle + oracle_fdw)
```

---

## 9. Self-Check

1. What are the four tools and their strengths?
2. What's the common prereq for Oracle sources?
3. How does ora2pg connect, and how do you test it?
4. How does db_migrator connect to the source?
5. What does SCT require, and its caveat?
6. When would you pick pgloader over ora2pg?

<details>
<summary>Answers</summary>

1. **ora2pg** (Oracle/MySQL — schema+PL/SQL+data+assessment, Perl); **pgloader** (MySQL/CSV/MSSQL — fast data + simple schema, Lisp); **AWS SCT** (multi-source schema+code + assessment, Java/JDBC, + DMS); **db_migrator** (Oracle→PG via FDW, a PG extension).
2. **Oracle Instant Client** — needed by both `DBD::Oracle` (ora2pg) and `oracle_fdw` (db_migrator).
3. Via Perl `DBD::Oracle` configured in `ora2pg.conf` (`ORACLE_DSN`/`USER`/`PWD`); test with `ora2pg -t SHOW_VERSION -c conf`.
4. Through a **foreign data wrapper** (`oracle_fdw`/`mysql_fdw`/`tds_fdw`) — it runs inside PostgreSQL.
5. **Java (JDK) + source JDBC drivers**; it's a **GUI** app (run on a workstation or CLI) and AWS is succeeding it with **DMS Schema Conversion**.
6. For **MySQL/CSV** with fast data loading and simple schema; ora2pg is for Oracle + comprehensive schema/PL-SQL-code conversion + assessment.
</details>

---

## 10. Video / Teaching Script

| # | On screen | Say (narration) |
|---|---|---|
| 1 | Title: "The migration toolbox" | "Moving Oracle, MySQL, or SQL Server to Postgres? No single tool does it all. Let's build the box with four — each for a different job." |
| 2 | ora2pg | "ora2pg is the Oracle heavyweight — schema, data, and PL/SQL turned into PL/pgSQL, plus an honest assessment of how hard it'll be." |
| 3 | pgloader | "pgloader is the sprinter — point it at MySQL and it loads schema and data in one command." |
| 4 | SCT | "AWS SCT assesses and converts across sources, and hands data off to DMS. Java, JDBC drivers, a proper conversion report." |
| 5 | db_migrator | "And db_migrator lives *inside* Postgres — it reaches into Oracle through a foreign data wrapper and pulls." |
| 6 | Instant Client | "The one thing every Oracle path needs: the Instant Client. Get that right first, or nothing connects." |
| 7 | Outro | "Toolbox ready, all connected. Next: assess an Oracle schema and estimate the effort." |

---

## 11. Glossary

- **ora2pg** — Perl Oracle/MySQL→PG migrator (schema+code+data).
- **pgloader** — Lisp fast data-loader/migrator.
- **AWS SCT** — Java schema-conversion + assessment tool.
- **db_migrator** — PG extension migrating via FDW.
- **Oracle Instant Client** — Oracle client libs (DBD::Oracle/oracle_fdw).
- **JDBC / FDW** — SCT's connectivity / db_migrator's connectivity.
- **DMS** — AWS Database Migration Service (data + succeeds SCT).

---
*NETAPORT lab series · PostgreSQL 17 on AlmaLinux 9 · Lab 139/222 · D1 Tooling & Assessment*
