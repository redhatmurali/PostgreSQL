# Lab 55 — Set Up Prometheus `postgres_exporter` + Grafana Dashboard

> **Track A · DBA · A7 Monitoring & Observability · Lab 6 of 8 (Lab 55/222)**
> Teaching package: concept → diagram → steps → verify → quick-ref → self-test → video script.
> **Depends on:** Labs 50–54 (the stats you'll now graph). Turns ad-hoc queries into continuous dashboards + alerts.

---

## 0. Lab Card (at a glance)

| Field | Value |
|---|---|
| **Objective** | Deploy `postgres_exporter`, scrape it with Prometheus, and visualize/alert in Grafana — with a least-privilege monitoring role and custom queries for the metrics from prior labs. |
| **Success criterion** | Prometheus scrapes PostgreSQL metrics; a Grafana dashboard shows connections, cache hit ratio, TPS, and replication lag; an alert fires on a threshold. |
| **Scope boundary** | The monitoring pipeline. Log analysis is Lab 56. |
| **Prereqs** | Labs 50–54; a running cluster (+ standby for lag); network between components |
| **Time** | 45–60 min |
| **Difficulty** | ★★★★☆ |
| **Risk** | Low — read-only monitoring; use a non-superuser exporter role. |

---

## 1. Learning Objectives

1. **The pipeline** — exporter → Prometheus → Grafana.
2. **`postgres_exporter`** — how it turns stats views into metrics.
3. **Least-privilege monitoring role** — not superuser.
4. **Custom queries** — expose the exact metrics from Labs 50–54.
5. **Dashboard + alert** — visualize and threshold.

---

## 2. Concept Primer — the "why"

**Ad-hoc queries don't scale to operations.** Labs 50–54 taught you to *query* stats — but you can't watch `pg_stat_activity` by hand at 3 a.m. A monitoring pipeline turns those views into **continuous time series** you can graph, correlate, and **alert** on.

**The three components:**
1. **`postgres_exporter`** — a small agent that connects to PostgreSQL, runs queries against the statistics views, and exposes the results as Prometheus metrics at an HTTP `/metrics` endpoint (default `:9187`). It ships **built-in** metrics (`pg_stat_database`, `pg_stat_replication`, connections, locks…) and supports **custom queries** for anything else.
2. **Prometheus** — a time-series database that **scrapes** the exporter's `/metrics` on an interval (e.g. every 15 s) and stores the series. You query it with **PromQL** and define **alerting rules**.
3. **Grafana** — connects to Prometheus as a data source and renders **dashboards**; it also evaluates alerts and routes notifications.

Flow: **PostgreSQL → postgres_exporter (/metrics) → Prometheus (scrape + store + rules) → Grafana (dashboards + alerts).**

**Least-privilege monitoring role (security, not an afterthought).** The exporter needs only to **read stats** — never superuser. Create a role with `pg_monitor` (a built-in role that grants read access to the monitoring views: `pg_read_all_stats`, `pg_read_all_settings`, `pg_stat_scan_tables`). `GRANT pg_monitor TO exporter;` gives full visibility with no write or admin power. Restrict its pg_hba line to the exporter host.

**Custom queries close the gap.** The built-ins cover a lot, but the exact derived metrics from prior labs — **cache hit ratio** (Lab 50), **replication byte/time lag** (Lab 54), **bloat estimates** (Lab 53), **blocking counts** (Lab 52) — are added via a custom-queries file (`queries.yaml`): each entry is a SQL query plus metric/label definitions, and the exporter exposes the results.

**Dashboards + alerts.** Grafana panels visualize PromQL over these series (connections, commits/sec, cache hit %, lag). Alerts threshold them (e.g. `pg_replication_lag_seconds > 30`, `cache_hit_ratio < 0.99`, `pg_stat_activity idle-in-transaction > N`), combining conditions to avoid false positives (Lab 54's idle-lag trap).

---

## 3. Diagrams

### 3.1 Pipeline + setup flow

```mermaid
flowchart TD
    A["create least-priv role: exporter + GRANT pg_monitor"] --> B["run postgres_exporter (DATA_SOURCE_NAME) → /metrics :9187"]
    B --> C["Prometheus scrapes :9187 every 15s → stores time series"]
    C --> D["Grafana ← Prometheus data source"]
    D --> E["dashboard: connections · cache hit % · TPS · replication lag"]
    E --> F["alert rules: lag>30s · cache<99% · idle-in-txn>N"]
    B --> G["queries.yaml: custom metrics (cache ratio, lag, bloat from Labs 50-54)"]
    G --> C
    F --> H([✔ continuous, alertable observability])
```

### 3.2 Components

```mermaid
flowchart LR
    PG[(PostgreSQL<br/>stat views)] -->|read as pg_monitor| EXP["postgres_exporter<br/>/metrics :9187"]
    EXP -->|scrape 15s| PROM["Prometheus<br/>store + PromQL + rules"]
    PROM -->|data source| GRAF["Grafana<br/>dashboards + alerts"]
    GRAF --> NOTIFY([alerts → email/Slack/PagerDuty])
    note["exporter = read-only pg_monitor role · custom queries for derived metrics · alert on combined conditions"]
```

---

## 4. Prerequisites

```bash
# a least-privilege monitoring role:
sudo -u postgres psql -c "CREATE ROLE exporter LOGIN PASSWORD 'ExpPass!1';"
sudo -u postgres psql -c "GRANT pg_monitor TO exporter;"       # read-only monitoring access
# allow it from the exporter host (localhost here):
HBA=$(sudo -u postgres psql -tAc "SHOW hba_file;")
echo "host all exporter 127.0.0.1/32 scram-sha-256" | sudo tee -a "$HBA"; sudo systemctl reload postgresql-17
```

---

## 5. Step-by-Step

### Step 1 — Install and run postgres_exporter

```bash
# download the release binary (adjust version), install as a service user:
cd /tmp
VER=0.15.0
curl -sL -o pgexp.tar.gz "https://github.com/prometheus-community/postgres_exporter/releases/download/v${VER}/postgres_exporter-${VER}.linux-amd64.tar.gz"
tar xzf pgexp.tar.gz && sudo cp postgres_exporter-*/postgres_exporter /usr/local/bin/
sudo useradd -r -s /sbin/nologin pgexporter 2>/dev/null || true

# systemd unit with the connection string + custom queries:
sudo tee /etc/systemd/system/postgres_exporter.service >/dev/null <<'EOF'
[Unit]
Description=Prometheus PostgreSQL exporter
After=network-online.target
[Service]
User=pgexporter
Environment=DATA_SOURCE_NAME=postgresql://exporter:ExpPass!1@127.0.0.1:5432/postgres?sslmode=disable
ExecStart=/usr/local/bin/postgres_exporter --extend.query-path=/etc/postgres_exporter/queries.yaml
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

### Step 2 — Custom queries for the Labs 50–54 metrics

```bash
sudo mkdir -p /etc/postgres_exporter
sudo tee /etc/postgres_exporter/queries.yaml >/dev/null <<'EOF'
pg_cache_hit:
  query: "SELECT datname, blks_hit::float/nullif(blks_hit+blks_read,0) AS ratio FROM pg_stat_database WHERE datname IS NOT NULL"
  metrics:
    - datname: {usage: "LABEL"}
    - ratio:   {usage: "GAUGE", description: "Cache hit ratio"}
pg_replication_lag:
  query: "SELECT COALESCE(EXTRACT(EPOCH FROM replay_lag),0) AS seconds, application_name FROM pg_stat_replication"
  metrics:
    - application_name: {usage: "LABEL"}
    - seconds: {usage: "GAUGE", description: "Replay lag seconds"}
pg_idle_in_txn:
  query: "SELECT count(*) AS c FROM pg_stat_activity WHERE state='idle in transaction'"
  metrics:
    - c: {usage: "GAUGE", description: "Idle-in-transaction sessions"}
EOF
sudo systemctl daemon-reload && sudo systemctl enable --now postgres_exporter
curl -s http://127.0.0.1:9187/metrics | grep -E "pg_up|pg_cache_hit|pg_replication_lag" | head
```

### Step 3 — Install and configure Prometheus

```bash
sudo dnf install -y prometheus2 2>/dev/null || echo "install prometheus from repo/binary"
sudo tee -a /etc/prometheus/prometheus.yml >/dev/null <<'EOF'
  - job_name: 'postgresql'
    static_configs:
      - targets: ['127.0.0.1:9187']
EOF
sudo systemctl enable --now prometheus
# confirm the target is UP:
curl -s http://127.0.0.1:9090/api/v1/targets | grep -o '"health":"up"' | head -1
```

### Step 4 — Install Grafana + add Prometheus data source

```bash
sudo tee /etc/yum.repos.d/grafana.repo >/dev/null <<'EOF'
[grafana]
name=grafana
baseurl=https://rpm.grafana.com
gpgcheck=1
gpgkey=https://rpm.grafana.com/gpg.key
EOF
sudo dnf install -y grafana && sudo systemctl enable --now grafana-server
# Grafana UI at http://<host>:3000 (admin/admin). Add data source: Prometheus → http://127.0.0.1:9090
```

### Step 5 — Import a PostgreSQL dashboard

```bash
# In Grafana UI: Dashboards → Import → use a community dashboard ID for postgres_exporter
#   (e.g., the prometheus-community PostgreSQL dashboard), select the Prometheus data source.
# Panels appear: connections, TPS (rate(pg_stat_database_xact_commit[5m])), cache hit %, replication lag.
```

### Step 6 — Add an alert

```bash
# In Grafana (or Prometheus rules): alert when replication lag exceeds 30s AND bytes behind:
#   expr: pg_replication_lag_seconds > 30
# generate lag to test (from Lab 54), watch the alert transition to Firing.
sudo -u postgres pgbench -c 8 -T 20 benchdb >/dev/null 2>&1 &
```

---

## 6. Verification Checklist

- [ ] `exporter` role has `pg_monitor` (not superuser)
- [ ] `curl :9187/metrics` returns `pg_up 1` and custom metrics
- [ ] Prometheus target `postgresql` is **UP**
- [ ] Grafana has the Prometheus data source
- [ ] Dashboard shows connections, cache hit %, TPS, replication lag
- [ ] An alert rule evaluates (and fires under a threshold breach)
- [ ] Custom queries expose Labs 50–54 metrics

---

## 7. Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `pg_up 0` | Exporter can't connect | Check `DATA_SOURCE_NAME`, pg_hba for `exporter`, password |
| Metrics missing | Role lacks stats access | `GRANT pg_monitor TO exporter` |
| Custom metrics absent | queries.yaml path/syntax | `--extend.query-path`; validate YAML; check exporter logs |
| Prometheus target DOWN | Wrong target/port/firewall | Fix `targets`; open 9187; `journalctl -u prometheus` |
| Grafana no data | Data source URL wrong | Point at `http://…:9090`; test the data source |
| Exporter is superuser | Over-privileged | Use `pg_monitor` role only |
| Alert never fires | Threshold/expr wrong | Test PromQL in Prometheus; verify the metric name |

---

## 8. Quick Reference Card (paste-ready)

```bash
# least-priv monitoring role
sudo -u postgres psql -c "CREATE ROLE exporter LOGIN PASSWORD 'ExpPass!1'; GRANT pg_monitor TO exporter;"

# exporter (systemd): DATA_SOURCE_NAME=postgresql://exporter:pw@127.0.0.1:5432/postgres?sslmode=disable
#   --extend.query-path=/etc/postgres_exporter/queries.yaml   → /metrics on :9187
curl -s http://127.0.0.1:9187/metrics | grep pg_up

# prometheus scrape job:
#   - job_name: postgresql
#     static_configs: [{ targets: ['127.0.0.1:9187'] }]

# grafana: add Prometheus data source (http://127.0.0.1:9090) → import a postgres_exporter dashboard
# alerts: rate(pg_stat_database_xact_commit[5m]) TPS · pg_replication_lag_seconds>30 · cache_hit<0.99
# pipeline: PostgreSQL → exporter(/metrics) → Prometheus(scrape+rules) → Grafana(dashboards+alerts)
# exporter role = pg_monitor (NOT superuser) · custom queries.yaml for derived metrics
```

---

## 9. Self-Check

1. What are the three components of the pipeline and their roles?
2. What does `postgres_exporter` do?
3. What privilege should the exporter's role have, and why not superuser?
4. How do you expose a derived metric (like cache hit ratio) that isn't built in?
5. What is the default exporter metrics port?
6. Why combine conditions in a lag alert (recall Lab 54)?

<details>
<summary>Answers</summary>

1. **postgres_exporter** (reads stats → `/metrics`), **Prometheus** (scrapes + stores + alert rules), **Grafana** (dashboards + alerts).
2. Connects to PostgreSQL, runs queries against the statistics views, and exposes the results as Prometheus metrics over HTTP.
3. `pg_monitor` (read-only monitoring access) — superuser is unnecessary and a security risk.
4. Add it to a `queries.yaml` custom-queries file (`--extend.query-path`) with SQL + metric/label definitions.
5. `:9187`.
6. To avoid false positives — e.g. time-based lag misleads on an idle primary (Lab 54); combine with byte lag.
</details>

---

## 10. Video / Teaching Script

| # | On screen | Say (narration) |
|---|---|---|
| 1 | Title: "From queries to dashboards" | "You've learned to query every stat by hand. Now let's make them continuous — graphed, stored, and alerting while you sleep." |
| 2 | pipeline diagram | "Three pieces: an exporter reads PostgreSQL's stats, Prometheus scrapes and stores them, Grafana draws and alerts." |
| 3 | pg_monitor role | "Security first: the exporter gets a read-only monitoring role — pg_monitor. Never superuser to watch metrics." |
| 4 | exporter /metrics | "Point it at the database, and there's a metrics endpoint — pg_up equals one means we're connected." |
| 5 | custom queries | "The built-ins are good, but our derived metrics — cache hit ratio, replication lag — come from a custom query file." |
| 6 | Grafana dashboard | "Wire Grafana to Prometheus, import a dashboard, and there it is: connections, throughput, cache hits, lag — live." |
| 7 | alert | "Then the payoff: an alert. Lag over thirty seconds, and someone gets paged — combined with byte lag so we don't cry wolf on idle." |
| 8 | Outro | "Continuous observability, no 3am guesswork. Next: turning the logs into reports with pgBadger." |

---

## 11. Glossary

- **postgres_exporter** — agent exposing PostgreSQL stats as Prometheus metrics.
- **Prometheus** — time-series DB that scrapes, stores, and alerts (PromQL).
- **Grafana** — dashboards and alerting over data sources.
- **`pg_monitor`** — built-in read-only monitoring role.
- **`queries.yaml`** — custom-query file for derived metrics.
- **`/metrics` :9187** — the exporter's HTTP endpoint.
- **Scrape / alert rule** — periodic pull / threshold condition.

---
*NETAPORT lab series · PostgreSQL 17 on AlmaLinux 9 · Lab 55/222 · A7 Monitoring & Observability*
