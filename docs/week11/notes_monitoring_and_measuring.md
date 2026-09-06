# Monitoring and Measuring — Detailed Notes



## Learning objectives

By the end of this topic, you should be able to:

- Explain the main ideas covered in **Monitoring and Measuring — Detailed Notes**.
- Connect these ideas to modern web application development.
- Recognize the patterns, terminology, and trade-offs used in practice.

> **Parent Topic:** Scaling
> **Scope:** How to observe, measure, and understand what a production web application is doing in real time — server logs, metrics, and live monitoring stacks.

---

## 1. Why Monitoring Matters

> **"You cannot manage what you cannot measure."** — Peter Drucker

In production, things go wrong in ways that are impossible to anticipate during development. Monitoring is the discipline of **continuously collecting data about system behaviour** so that:

1. **Problems are detected** before users report them (or before too many users are affected).
2. **Root causes are diagnosed** quickly using historical data.
3. **Capacity is planned** proactively — you know when you're approaching limits before you hit them.
4. **Performance regressions** from new deployments are caught immediately.
5. **SLAs (Service Level Agreements)** can be measured and enforced.

### The Observability Triangle

Modern observability is built on three pillars:

```
                    ┌─────────────────────┐
                    │   OBSERVABILITY     │
                    └──────────┬──────────┘
                               │
          ┌────────────────────┼────────────────────┐
          ▼                    ▼                    ▼
   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
   │    LOGS     │     │   METRICS   │     │   TRACES    │
   │             │     │             │     │             │
   │ What        │     │ How much /  │     │ How a       │
   │ happened?   │     │ How fast?   │     │ request     │
   │ (events)    │     │ (numbers)   │     │ travelled   │
   └─────────────┘     └─────────────┘     └─────────────┘
   ELK Stack            Prometheus          Jaeger / Zipkin
   Loki                 Grafana             AWS X-Ray
```

| Pillar | What It Is | Example |
|---|---|---|
| **Logs** | Timestamped records of discrete events | "User 4521 logged in at 10:32:04" |
| **Metrics** | Numeric measurements over time | "CPU: 73%, RPS: 420, p99 latency: 340ms" |
| **Traces** | End-to-end journey of a single request through a system | "Request spent 200ms in DB, 10ms in app, 5ms in serialization" |

---

## 2. Server Logs

Logs are the **most fundamental** observability tool — every request and significant event is recorded as a line of text with a timestamp.

### 2.1 Types of Server Logs

#### Access Log
Records every HTTP request the server receives.

**Nginx default access log format:**
```
$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent"
```

**Example entries:**
```
203.0.113.42 - - [03/Sep/2026:10:32:04 +0530] "GET /api/products HTTP/2.0" 200 1842 "-" "Mozilla/5.0 (iPhone)"
198.51.100.7 - - [03/Sep/2026:10:32:05 +0530] "POST /api/checkout HTTP/2.0" 201 342 "https://shop.com/cart" "Chrome/120"
10.0.0.5     - - [03/Sep/2026:10:32:06 +0530] "GET /admin/users HTTP/1.1" 403 128 "-" "curl/7.88"
```

**What you can derive from access logs:**
- Request volume over time (throughput)
- Error rate (count of 4xx/5xx status codes)
- Most visited URLs
- User agents / device types
- Referrers (where traffic is coming from)
- Response sizes (large responses = bandwidth cost)

#### Error Log
Records errors, warnings, and exceptions from the server and application.

```
[03/Sep/2026 10:35:22] [error] connect() to unix:/tmp/flask.sock failed (111: Connection refused)
[03/Sep/2026 10:35:45] [warn]  upstream response time 4.312 > 3s threshold for /api/search
[03/Sep/2026 10:36:01] [crit]  worker process 12345 exited with code 1
```

#### Application Log
Custom log entries emitted by your application code. The most information-rich log type.

```python
# Python (Flask/Django) — using Python's logging module
import logging

logger = logging.getLogger(__name__)

def process_order(order_id, user_id):
    logger.info(f"Order processing started", extra={"order_id": order_id, "user_id": user_id})
    try:
        result = payment_gateway.charge(order_id)
        logger.info(f"Payment successful", extra={"order_id": order_id, "amount": result.amount})
    except PaymentError as e:
        logger.error(f"Payment failed", extra={"order_id": order_id, "error": str(e)})
        raise
```

**Structured output (JSON — machine-readable):**
```json
{"timestamp": "2026-09-03T10:36:22Z", "level": "INFO", "msg": "Payment successful", "order_id": "ORD-8821", "amount": 1299.00}
{"timestamp": "2026-09-03T10:36:23Z", "level": "ERROR", "msg": "Payment failed", "order_id": "ORD-8822", "error": "Card declined"}
```

**Log Levels (in order of severity):**

| Level | Use For |
|---|---|
| `DEBUG` | Detailed diagnostic info — only in development |
| `INFO` | Normal operational events — user login, order placed |
| `WARNING` | Something unexpected but handled — retry attempt, deprecated API used |
| `ERROR` | A failure that needs attention — exception caught, service unavailable |
| `CRITICAL` | System-level failure — database down, disk full, app cannot start |

#### Slow Query Log
Logs database queries that exceed a time threshold. Invaluable for finding N+1 problems and missing indexes.

```sql
-- PostgreSQL: log queries slower than 1 second
-- In postgresql.conf:
log_min_duration_statement = 1000  -- milliseconds

-- Example slow query log output:
-- 2026-09-03 10:40:11 IST [12345] LOG: duration: 4213.082 ms statement:
--   SELECT u.*, o.*, p.*
--   FROM users u
--   JOIN orders o ON u.id = o.user_id
--   JOIN products p ON o.product_id = p.id
--   WHERE u.created_at > '2026-01-01'
```

---

### 2.2 Log Management Principles

#### Centralized Logging
In a multi-server environment, logs are spread across many machines. You need to **aggregate them into one place** for searching and analysis.

```
App Server 1 logs  ──┐
App Server 2 logs  ──┤──▶  Log Aggregator  ──▶  Central Log Store  ──▶  UI / Alerts
App Server 3 logs  ──┤     (Logstash/Fluentd)   (Elasticsearch/Loki)    (Kibana/Grafana)
DB Server logs     ──┘
Nginx logs         ──┘
```

#### Log Rotation
Logs grow indefinitely. Log rotation automatically:
- Archives old logs (e.g., compress logs older than 1 day with gzip)
- Deletes very old logs (e.g., delete logs older than 30 days)
- Prevents disk from filling up and crashing the server

```bash
# logrotate config (/etc/logrotate.d/nginx)
/var/log/nginx/*.log {
    daily           # Rotate daily
    missingok       # Don't error if log file is missing
    rotate 14       # Keep 14 days of logs
    compress        # Compress rotated logs with gzip
    delaycompress   # Don't compress yesterday's log yet
    notifempty      # Don't rotate if empty
    postrotate
        nginx -s reopen  # Tell nginx to open new log file after rotation
    endscript
}
```

#### Structured Logging
Write logs as **JSON** (or another machine-parseable format) rather than free-form text. This makes logs **searchable and filterable** by fields.

```
# Unstructured (hard to query):
"User 4521 checked out order ORD-8821 for $1299 at 10:36:22"

# Structured (easy to query: filter by user_id, order_id, amount):
{"timestamp": "2026-09-03T10:36:22Z", "event": "checkout", "user_id": 4521, "order_id": "ORD-8821", "amount": 1299}
```

---

### 2.3 Analysing Logs with Command-Line Tools

For quick analysis without a full monitoring stack:

```bash
# Count requests per status code
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
# Output:
#  42301  200
#   1832  304
#    203  404
#     12  500

# Find the 10 most requested URLs
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10

# Find all 5xx errors in the last hour
grep "$(date -d '1 hour ago' '+%d/%b/%Y:%H')" /var/log/nginx/access.log | grep '" 5'

# Calculate average response size
awk '{sum += $10; count++} END {print sum/count " bytes avg"}' /var/log/nginx/access.log

# Find slowest requests (if response time is logged)
sort -t'"' -k5 -rn /var/log/nginx/access.log | head -20

# Stream live log and highlight errors
tail -f /var/log/nginx/access.log | grep --color=always -E '(50[0-9]|$)'
```

---

## 3. Live Monitoring Tools

Logs tell you what happened. **Live monitoring tools** tell you what is happening **right now** — and alert you when something is wrong before you notice it manually.

### The Two Main Stacks

| Stack | Purpose | Components |
|---|---|---|
| **ELK Stack** | Log aggregation, search, and visualization | Elasticsearch + Logstash/Beats + Kibana |
| **Prometheus + Grafana** | Metrics collection and visualization | Prometheus + Grafana + Alertmanager |

These are often used **together** — ELK for logs, Prometheus/Grafana for metrics.

---

## 4. ELK Stack

**ELK** stands for **Elasticsearch, Logstash, Kibana**. Together they form a complete pipeline for collecting, storing, searching, and visualizing log data.

Modern versions include **Beats** (lightweight shippers) and are sometimes called the **Elastic Stack**.

### 4.1 Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                          ELK STACK                                   │
│                                                                      │
│  ┌──────────────────┐                                                │
│  │   DATA SOURCES   │                                                │
│  │  App Server logs │                                                │
│  │  Nginx logs      │                                                │
│  │  DB logs         │                                                │
│  │  System logs     │                                                │
│  └────────┬─────────┘                                                │
│           │                                                          │
│           ▼                                                          │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │              COLLECTION & SHIPPING                       │        │
│  │                                                          │        │
│  │  Filebeat        ─── lightweight, reads log files        │        │
│  │  Metricbeat      ─── ships system/service metrics        │        │
│  │  Logstash        ─── heavy-duty: parse, filter, enrich  │        │
│  └──────────────────────────────┬───────────────────────────┘        │
│                                 │                                    │
│                                 ▼                                    │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │              ELASTICSEARCH                               │        │
│  │                                                          │        │
│  │  Distributed search & analytics engine                   │        │
│  │  Stores logs as JSON documents                           │        │
│  │  Indexes every field — search millions of logs in <1s   │        │
│  │  Horizontally scalable (add nodes for more capacity)    │        │
│  └──────────────────────────────┬───────────────────────────┘        │
│                                 │                                    │
│                                 ▼                                    │
│  ┌──────────────────────────────────────────────────────────┐        │
│  │              KIBANA                                      │        │
│  │                                                          │        │
│  │  Web UI for Elasticsearch                                │        │
│  │  ├── Discover: Search and explore raw logs              │        │
│  │  ├── Visualize: Bar charts, line graphs, pie charts     │        │
│  │  ├── Dashboard: Combine visualizations                  │        │
│  │  ├── Alerts: Notify when conditions are met             │        │
│  │  └── APM: Application performance monitoring            │        │
│  └──────────────────────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────────────────────┘
```

---

### 4.2 Elasticsearch

**What it is:** A distributed, full-text search and analytics engine built on Apache Lucene.

**Key concepts:**

| Concept | SQL Equivalent | Description |
|---|---|---|
| **Index** | Database / Table | A collection of documents (e.g., `nginx-logs-2026-09`) |
| **Document** | Row | A single log entry stored as JSON |
| **Field** | Column | A key in the JSON document (e.g., `status_code`, `response_time`) |
| **Shard** | — | A piece of an index distributed across nodes |
| **Node** | — | A single Elasticsearch server instance |
| **Cluster** | — | A group of nodes working together |

**Querying with Elasticsearch (Kibana Query Language):**
```
# Find all 500 errors in the last 15 minutes
status_code: 500 AND @timestamp > now-15m

# Find slow requests to the checkout API
request_path: "/api/checkout" AND response_time_ms > 1000

# Find all errors for a specific user
user_id: 4521 AND level: ERROR

# Count errors by endpoint
status_code: 5* | stats count by request_path
```

**Why Elasticsearch is fast:**
- **Inverted index:** Every field value is indexed — like a book's back-index. Finding all documents where `status_code = 500` is instant.
- **Distributed:** Queries run in parallel across shards on multiple nodes.
- **Near real-time:** Documents are searchable within ~1 second of ingestion.

---

### 4.3 Logstash

**What it is:** A server-side data processing pipeline. It **ingests** data from many sources, **transforms** it, and **ships** it to a destination (usually Elasticsearch).

**Logstash Pipeline:**
```
INPUT ──▶ FILTER ──▶ OUTPUT

Example:
┌──────────────────────────────────────────────────────────┐
│ INPUT: Read Nginx access log                              │
│   203.0.113.42 - [03/Sep/2026:10:32:04] "GET /api" 200   │
└──────────────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│ FILTER: Parse with grok (regex pattern)                   │
│   ip:        203.0.113.42                                 │
│   timestamp: 2026-09-03T10:32:04                          │
│   method:    GET                                          │
│   path:      /api                                         │
│   status:    200                                          │
│   + GeoIP lookup: country=US, city=New York               │
│   + UA parse: browser=Chrome, OS=Windows, device=Desktop  │
└──────────────────────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────┐
│ OUTPUT: Send to Elasticsearch as structured JSON          │
│   { "ip": "203.0.113.42", "status": 200,                 │
│     "path": "/api", "country": "US", ... }               │
└──────────────────────────────────────────────────────────┘
```

**Logstash config example:**
```ruby
input {
  file {
    path => "/var/log/nginx/access.log"
    start_position => "beginning"
  }
}

filter {
  grok {
    match => { "message" => "%{COMBINEDAPACHELOG}" }
  }
  date {
    match => ["timestamp", "dd/MMM/yyyy:HH:mm:ss Z"]
    target => "@timestamp"
  }
  geoip {
    source => "clientip"    # Add country/city from IP
  }
  mutate {
    convert => { "response" => "integer" }  # status code as int
    convert => { "bytes" => "integer" }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "nginx-logs-%{+YYYY.MM.dd}"  # Daily index rotation
  }
}
```

**Filebeat vs Logstash:**

| | Filebeat | Logstash |
|---|---|---|
| **Resource use** | Very lightweight (Go binary, ~50MB RAM) | Heavy (JVM, ~500MB–1GB RAM) |
| **Processing** | Minimal — just ships logs | Full ETL pipeline — parse, filter, enrich |
| **Deploy on** | Every server (agent) | Dedicated server |
| **Typical use** | Ship logs from many servers to central store | Central parsing/enrichment pipeline |

Common pattern: **Filebeat on every server → Logstash central server → Elasticsearch**

---

### 4.4 Kibana

**What it is:** A web-based UI for exploring and visualizing data in Elasticsearch.

**Key features:**

#### Discover
- Browse raw log entries with full-text search.
- Filter by time range, field values, and search terms.
- See individual log lines with all parsed fields.
```
Time: 2026-09-03 10:32:04    Status: 500    Path: /api/checkout
  ip: 203.0.113.42    method: POST    response_time: 4213ms
  error: "Database connection pool exhausted"
```

#### Visualize
Build charts from log data:
- **Line chart:** Error rate over time.
- **Bar chart:** Top 10 slowest endpoints.
- **Pie chart:** Request distribution by status code.
- **Heat map:** Traffic by hour of day × day of week.
- **Data table:** Top 10 IPs by request count.

#### Dashboard
Combine multiple visualizations into a real-time dashboard:
```
┌─────────────────────────────────────────────────────────────┐
│  NGINX MONITORING DASHBOARD                    [Last 1 hour] │
├──────────────────────────┬──────────────────────────────────┤
│  Requests/min            │  Error rate (5xx)                │
│  ████████░░ 420 rps      │  0.24% ↑ from 0.18%             │
├──────────────────────────┼──────────────────────────────────┤
│  Response time (p95)     │  Status code distribution        │
│  [line chart]            │  [pie: 200:87%, 304:10%, 5xx:3%] │
├──────────────────────────┴──────────────────────────────────┤
│  Top 10 slowest endpoints (avg response time)               │
│  /api/search          1842ms  [████████████]                │
│  /api/recommendations  923ms  [██████]                      │
│  /api/checkout         412ms  [███]                         │
└─────────────────────────────────────────────────────────────┘
```

---

## 5. Prometheus + Grafana

While ELK handles **logs** (events), Prometheus + Grafana handles **metrics** (numbers over time). They answer different questions and are highly complementary.

### 5.1 Prometheus

**What it is:** An open-source **time-series database and monitoring system**. It collects numeric metrics from your services and stores them efficiently for querying.

**Pull-Based Architecture:**
Unlike most monitoring tools that receive data pushed to them, Prometheus **pulls** metrics by scraping HTTP endpoints:

```
App Server exposes:  GET /metrics
                     # A Prometheus text format response
                     http_requests_total{method="GET", status="200"} 42301
                     http_requests_total{method="POST", status="500"} 12
                     http_request_duration_seconds{quantile="0.95"} 0.342
                     process_resident_memory_bytes 52428800

Prometheus Server:   Every 15 seconds, scrapes GET /metrics from all targets
                     Stores as time-series data
```

**Why pull-based?**
- Prometheus knows which targets exist (not the other way around).
- If a target goes down, Prometheus detects the scrape failure → alerts.
- Easier to manage in dynamic environments (Kubernetes).

---

### 5.2 Prometheus Data Model

All data in Prometheus is a **time series**: a sequence of `(timestamp, value)` pairs, identified by a **metric name** and **labels**.

```
metric_name{label1="value1", label2="value2"} value @timestamp

Examples:
http_requests_total{method="GET", endpoint="/api/products", status="200"} 42301
http_requests_total{method="POST", endpoint="/api/checkout", status="500"} 12
node_cpu_seconds_total{cpu="0", mode="idle"} 98234.5
redis_connected_clients{} 47
```

**Metric Types:**

| Type | Description | Example |
|---|---|---|
| **Counter** | Only increases (resets to 0 on restart). Tracks total occurrences. | `http_requests_total`, `errors_total` |
| **Gauge** | Can go up or down. Current snapshot of a value. | `memory_usage_bytes`, `active_connections`, `queue_length` |
| **Histogram** | Samples observations into configurable buckets. Tracks distribution. | `http_request_duration_seconds` (buckets: 0.1s, 0.5s, 1s, 5s) |
| **Summary** | Pre-calculated quantiles on the client side. | `request_duration_p50`, `request_duration_p99` |

**Histogram vs Summary:**
```
Histogram stores:
  http_request_duration_seconds_bucket{le="0.1"} 8234   ← requests under 100ms
  http_request_duration_seconds_bucket{le="0.5"} 21043  ← requests under 500ms
  http_request_duration_seconds_bucket{le="1.0"} 29821  ← requests under 1s
  http_request_duration_seconds_bucket{le="+Inf"} 30100 ← all requests
  http_request_duration_seconds_sum 12834.2              ← total seconds
  http_request_duration_seconds_count 30100              ← total requests

→ Prometheus can calculate any percentile from these buckets at query time
→ Can be aggregated across multiple servers (histograms can be summed)
```

---

### 5.3 PromQL — Prometheus Query Language

Prometheus has its own powerful query language for slicing and aggregating metrics.

```promql
# Total request rate (requests per second, averaged over 5 min window)
rate(http_requests_total[5m])

# Error rate (fraction of requests returning 5xx)
rate(http_requests_total{status=~"5.."}[5m])
  /
rate(http_requests_total[5m])

# 95th percentile latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Memory usage in MB
process_resident_memory_bytes / 1024 / 1024

# CPU usage percentage
100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# DB connection pool utilization (% of pool used)
db_pool_connections_active / db_pool_connections_max * 100

# Requests per endpoint (top 5 busiest)
topk(5, sum by (endpoint) (rate(http_requests_total[5m])))
```

---

### 5.4 Alertmanager

Prometheus includes **Alertmanager** — a rules engine that triggers notifications when metric thresholds are breached.

**Alert Rules (defined in YAML):**
```yaml
groups:
  - name: web_app_alerts
    rules:
      # Alert if error rate > 1% for 5 minutes
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.instance }}"
          description: "Error rate is {{ $value | humanizePercentage }}"

      # Alert if p95 latency > 1 second
      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High latency detected"

      # Alert if memory usage > 90%
      - alert: HighMemoryUsage
        expr: process_resident_memory_bytes / node_memory_MemTotal_bytes > 0.9
        for: 5m
        labels:
          severity: critical
```

**Alertmanager routes to:**
- 📧 Email
- 💬 Slack / Teams / Discord
- 📟 PagerDuty / OpsGenie (on-call escalation)
- 🪝 Webhook (any custom integration)

---

### 5.5 Grafana

**What it is:** A multi-source data visualization and dashboarding platform. Grafana does not store data — it **queries** data sources (Prometheus, Elasticsearch, PostgreSQL, Loki, etc.) and visualizes them.

**Grafana Dashboard Example:**
```
┌─────────────────────────────────────────────────────────────────────┐
│  PRODUCTION OVERVIEW                                  [Last 1 hour] │
├────────────────┬────────────────┬────────────────┬──────────────────┤
│  RPS           │  Error Rate    │  p95 Latency   │  Active Servers  │
│  420 req/s     │  0.24%  ▲      │  342ms         │  6/6  ✅          │
├────────────────┴────────────────┴────────────────┴──────────────────┤
│  Request Rate (per endpoint)          │  CPU Utilization (per server)│
│  [stacked area chart]                 │  [line chart, 6 lines]      │
│                                       │                             │
│  /api/products ████████               │  Server 1: 72%              │
│  /api/search   █████                  │  Server 2: 68%              │
│  /api/checkout ██                     │  Server 3: 71%              │
├───────────────────────────────────────┴─────────────────────────────┤
│  Error Rate Over Time                  │  DB Query Duration (p95)   │
│  [line chart — spikes visible]         │  [line chart — 150ms avg]  │
├───────────────────────────────────────┴─────────────────────────────┤
│  Active Alerts                                                       │
│  ⚠️  HighLatency on /api/search — 2.1s p95 (threshold: 1s) — 3m ago │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Grafana Panel Types:**

| Panel | Best For |
|---|---|
| **Time series** | Request rate, latency, CPU over time |
| **Gauge** | Current value vs threshold (CPU%, memory%) |
| **Stat** | Single important number (total errors, uptime) |
| **Bar chart** | Comparing values across categories (requests per endpoint) |
| **Heatmap** | Latency distribution over time (histogram data) |
| **Table** | Tabular data with sortable columns |
| **Logs** | Log stream panel (via Loki) |
| **Alert list** | Show all firing/pending alerts |

**Grafana Data Sources (can query all of these):**
- Prometheus (metrics)
- Elasticsearch / Loki (logs)
- PostgreSQL / MySQL (directly query your DB)
- CloudWatch (AWS metrics)
- InfluxDB (time-series data)
- Jaeger / Tempo (distributed traces)

---

### 5.6 What to Monitor — Key Metrics

#### The "Four Golden Signals" (Google SRE Book)

| Signal | What To Track | Example Metric |
|---|---|---|
| **Latency** | How long requests take | p50, p95, p99 response time |
| **Traffic** | How much demand | Requests per second |
| **Errors** | Rate of failing requests | 5xx rate, exception count |
| **Saturation** | How full your system is | CPU%, memory%, DB pool% |

#### Per-Component Metrics to Monitor

**Web Server (Nginx):**
```
http_requests_total          — request rate by endpoint + status
http_request_duration_ms     — latency (p50, p95, p99)
http_connections_active      — concurrent connections
nginx_upstream_response_time — how long backend takes
```

**Application Server:**
```
process_cpu_seconds_total    — CPU usage
process_resident_memory_bytes — memory usage
app_request_queue_length     — are requests building up?
app_errors_total             — application-level errors
app_cache_hit_rate           — cache effectiveness
```

**Database:**
```
db_query_duration_ms         — slow query detection
db_connections_active        — connection pool usage
db_replication_lag_seconds   — how far replicas are behind
db_lock_wait_time_ms         — lock contention
db_rows_read_total           — read volume
db_rows_written_total        — write volume
```

**System:**
```
node_cpu_seconds_total       — CPU by core and mode
node_memory_MemAvailable_bytes — free memory
node_disk_io_time_seconds    — disk I/O utilization
node_network_transmit_bytes_total — network bandwidth
node_filesystem_avail_bytes  — disk space remaining
```

---

## 6. Full Monitoring Stack — Putting It Together

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE MONITORING STACK                             │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │  APPLICATION LAYER                                              │    │
│  │  Flask/Django app  ─── exposes /metrics ──▶ Prometheus         │    │
│  │                    ─── writes logs ──────▶ Filebeat/Logstash   │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌──────────────────┐          ┌────────────────────────────────────┐   │
│  │   METRICS STACK  │          │           LOGS STACK               │   │
│  │                  │          │                                    │   │
│  │   Prometheus     │          │   Filebeat ──▶ Logstash            │   │
│  │   (scrape & store)│         │               ──▶ Elasticsearch    │   │
│  │        │         │          │                       │            │   │
│  │        ▼         │          │                       ▼            │   │
│  │   Alertmanager   │          │                   Kibana           │   │
│  │   (alerts)       │          │                   (log search)     │   │
│  │        │         │          │                                    │   │
│  │        ▼         │          └────────────────────────────────────┘   │
│  │   Grafana ◀──────┼──────── also queries Elasticsearch for logs       │
│  │   (dashboards)   │                                                    │
│  └──────────────────┘                                                    │
│                                                                          │
│  ALERTS ROUTE TO:                                                        │
│  Slack ── PagerDuty ── Email ── Webhook                                  │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Key Takeaways

- **Observability has three pillars:** Logs (what happened), Metrics (how much/how fast), Traces (how a request travelled). A mature system uses all three.
- **Server logs** are the most fundamental tool — access logs, error logs, application logs, and slow query logs each reveal different problems.
- **Structured logging** (JSON) makes logs machine-searchable and dramatically more useful in an ELK stack.
- **Log rotation** is essential to prevent disk exhaustion — configure it on every server.
- **ELK Stack** = collect logs (Filebeat/Logstash) → index them (Elasticsearch) → visualize and search (Kibana). Ideal for log aggregation and search.
- **Prometheus** uses a pull model — it scrapes a `/metrics` endpoint from services. Stores numeric time-series data.
- **The 4 metric types** in Prometheus: Counter (total counts), Gauge (current values), Histogram (distributions), Summary (pre-calculated quantiles).
- **The Four Golden Signals** (Google SRE): Latency, Traffic, Errors, Saturation — monitor all four for every critical service.
- **Grafana** visualizes data from any source (Prometheus, Elasticsearch, PostgreSQL) in unified dashboards with alerting.
- **Alertmanager** routes alerts to Slack/PagerDuty/email when thresholds are breached — essential for on-call response.
- Monitor at **every layer**: web server, app server, database, and system resources — the bottleneck can be anywhere.

## Further practice

- Revisit the examples in this topic and explain each step in your own words.
- Identify one place where the concept could be used in a web application.
- Test a small variation and note how the behavior changes.

