---
name: dna-log-api
version: "1.4.0"
description: "Query, analyse and AI-diagnose DnA platform logs via log-api (cluster c82p401). Covers CI/CD pipeline log search by run_id, Kubernetes app log search by instance name, discovering all instances, parallel multi-instance error scanning, diagnosing build/deploy/runtime failures, API key setup on macOS/Windows. AI diagnosis: Maven/npm/Vite build errors, Docker/K8s deploy errors (DEPLOYMENT_FAILED, CrashLoopBackOff, OOMKilled), Spring Boot startup failures, GCExcel license, ALICE/IAM auth, JDBC errors. USE FOR: check deployment logs, why did build fail, CI/CD pipeline failure, find run_id, msmcore/onecm/compas/cashup/csc logs, frontend build error, npm install fail, helm deploy failure, kubernetes app logs, DEPLOYMENT_FAILED, BUILD_FAILED, log-api, JDBC error, Alice entitlement error, Redis error, Azure AD error, crash loop, pod crash, healthz 404, frontend blank page, SPA routing, diagnose deploy error, maven error, vite error, springboot crash, vault injection, gcexcel license, alice 403, root cause analysis."
roles: [developer, platform-engineer]
domain: infrastructure
---

# DnA Log API — Search & Analysis Guide

The **log-api** is a FastAPI service deployed on cluster `c82p401` that wraps the internal
OpenSearch Dashboards at `logs.dna-prod.app.corpintra.net`. It supports two log sources:

| Source | Index | Key filter | Timestamp field |
|--------|-------|------------|-----------------|
| CI/CD pipeline | `ci-cd-*` | `run_id` (GitHub Actions run ID) | `@timestamp` (epoch ms) |
| Kubernetes app | `kubernetes_*` | `instance` label (`app.kubernetes.io/instance`) | `@flb-timestamp` (ISO) |

**Base URL:** `http://log-api.c82p401.c82.cloud.corpintra.net`  
**Swagger UI:** `http://log-api.c82p401.c82.cloud.corpintra.net/docs`  
**Auth header:** `X-API-Key: <key>` (all endpoints except `/health`)

> ⚠️ **IMPORTANT — DNA CodeSpaces deployment logs**: Never check GitHub Actions UI/API for DNA CodeSpaces build or deployment failures. Always query the `ci-cd-*` OpenSearch index using the `run_id` found in `cs-be` pod logs (`prod-dna-cs-workspaces` namespace). Look for `updateGitJobRunId called for wsId=..., gitJobRunId=<ID>` in `cs-be` logs to find the run ID.

---

## 1. API Key Setup

The API key is **not hardcoded** here. Retrieve it from the OS credential store.

### macOS — Keychain

```bash
# Save (one-time setup)
security add-generic-password \
  -s "log_api_key" \
  -a "MUYYANG" \
  -w "<YOUR_API_KEY>"

# Retrieve in shell scripts / Copilot sessions
API_KEY=$(security find-generic-password -s "log_api_key" -a "MUYYANG" -w)
```

### Windows — Credential Manager (PowerShell)

```powershell
# Save (one-time setup — requires module)
Install-Module -Name CredentialManager -Scope CurrentUser -Force

Import-Module CredentialManager
New-StoredCredential -Target "log_api_key" -UserName "MUYYANG" -Password "<YOUR_API_KEY>" -Persist LocalMachine

# Retrieve
$cred = Get-StoredCredential -Target "log_api_key"
$API_KEY = $cred.GetNetworkCredential().Password
```

### Verify the key works

```bash
# Health (no key needed)
curl http://log-api.c82p401.c82.cloud.corpintra.net/health

# Authenticated test
API_KEY=$(security find-generic-password -s "log_api_key" -a "MUYYANG" -w)
curl -s -H "X-API-Key: $API_KEY" \
  "http://log-api.c82p401.c82.cloud.corpintra.net/indices"
```

---

## 2. Instance Naming Convention (Kubernetes app logs)

The `instance` parameter follows this pattern from the DNA CodeSpaces platform:

```
<codespace-name>-int    ← integration / dev environment
<codespace-name>-prod   ← production environment
```

Examples:
- `fc-onepc-msmcore-frontend-dev-int` — MSMCORE frontend, int env
- `onecm-prod-backend-int` — OneCM backend, int env
- `fc-onepc-csc-frontend-azuresql-prod` — CSC frontend, prod env

> The codespace name is the **lowercased GHE repo name** with hyphens. Check the k8s pod label
> `app.kubernetes.io/instance` if unsure.

---

## 3. Querying CI/CD Logs (by run_id)

Use when you know the GitHub Actions run ID.

```bash
API_KEY=$(security find-generic-password -s "log_api_key" -a "MUYYANG" -w)
BASE="http://log-api.c82p401.c82.cloud.corpintra.net"

# Last 6 days, ascending (good for reading a build log top-to-bottom)
curl -s -H "X-API-Key: $API_KEY" \
  "$BASE/cicd/logs?run_id=57088257&from_expr=now-6d&size=5000&sort_order=asc" \
  | python3 -m json.tool

# Compact: just the log lines
curl -s -H "X-API-Key: $API_KEY" \
  "$BASE/cicd/logs?run_id=57088257&from_expr=now-6d&size=5000&sort_order=asc" \
  | python3 -c "
import sys, json, re
d = json.load(sys.stdin)
print('total:', d['total'])
for l in d['logs']:
    clean = re.sub(r'^\d{4}-\d{2}-\d{2}T[\d:.Z]+\s+', '', l.get('msg','').strip())
    print(clean[:200])
"
```

**Parameters:**

| Param | Default | Notes |
|-------|---------|-------|
| `run_id` | required | GitHub Actions workflow run ID |
| `from_expr` | `now-15m` | OpenSearch time expression |
| `to_expr` | `now` | OpenSearch time expression |
| `size` | 500 | Max rows (up to 10 000) |
| `sort_order` | `asc` | `asc` for chronological, `desc` for latest-first |

---

## 4. Finding a run_id Without Prior Knowledge

When the user doesn't have the run_id, query OpenSearch directly using a text search across the
`ci-cd-*` index (bypasses the run_id requirement):

```python
import httpx, json, warnings
warnings.filterwarnings("ignore")

BASE = "https://logs.dna-prod.app.corpintra.net"

# Get anonymous session cookie
r = httpx.get(f"{BASE}/auth/anonymous", follow_redirects=True, verify=False, timeout=15)
cookie = r.cookies.get("security_authentication")

# Free-text search — replace "msmcore" with any keyword (project name, error text, etc.)
query = {
    "size": 50,
    "query": {
        "bool": {
            "filter": [
                {"query_string": {"query": "msmcore", "default_field": "*"}},
                {"range": {"@timestamp": {"gte": "now-6d", "lte": "now"}}}
            ]
        }
    },
    "sort": [{"@timestamp": {"order": "desc"}}],
    "_source": ["run_id", "msg", "job_name", "workflow_name", "@timestamp"]
}

headers = {
    "Content-Type": "application/json",
    "osd-xsrf": "true",
    "osd-version": "2.17.1",
    "securitytenant": "anonymous",
    "Cookie": f"security_authentication={cookie}"
}

resp = httpx.post(
    f"{BASE}/internal/search/opensearch",
    json={"params": {"index": "ci-cd-*", "body": query}},
    headers=headers, verify=False, timeout=30
)

hits = resp.json().get("rawResponse", {}).get("hits", {}).get("hits", [])

# Deduplicate run_ids
seen = {}
for h in hits:
    src = h["_source"]
    rid = src.get("run_id")
    if rid not in seen:
        seen[rid] = {"job": src.get("job_name",""), "ts": src.get("@timestamp","")}
        print(f"run_id={rid}  job={seen[rid]['job']}  ts={seen[rid]['ts']}")
```

---

## 5. Querying Kubernetes App Logs

```bash
API_KEY=$(security find-generic-password -s "log_api_key" -a "MUYYANG" -w)
BASE="http://log-api.c82p401.c82.cloud.corpintra.net"

# Error logs, last 6 days
curl -s -H "X-API-Key: $API_KEY" \
  "$BASE/app/logs?instance=fc-onepc-msmcore-frontend-dev-int&from_expr=now-6d&size=200&level=ERROR" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
print('total:', d['total'])
for l in d['logs']:
    print(f\"[{l.get('@flb-timestamp','')}] {l.get('log','')[:150]}\")
"

# All levels — useful for startup/init failures
curl -s -H "X-API-Key: $API_KEY" \
  "$BASE/app/logs?instance=fc-onepc-msmcore-frontend-dev-int&from_expr=now-24h&size=500&sort_order=asc"
```

**Parameters:**

| Param | Default | Notes |
|-------|---------|-------|
| `instance` | required | Full `app.kubernetes.io/instance` label value |
| `from_expr` | `now-15m` | Time range start |
| `size` | 500 | Max rows |
| `level` | (none) | Filter: `ERROR`, `WARN`, `INFO` |
| `sort_order` | `desc` | `asc` for startup sequence, `desc` for latest errors |

```bash
# List available fields in kubernetes_* index
curl -s -H "X-API-Key: $API_KEY" "$BASE/app/logs/fields" | python3 -m json.tool
```

---

## 6. Analysing Build/Deploy Failures

### Workflow

1. **Find the run_id** — from GHE Actions UI, or via the free-text search in §4
2. **List jobs in the run** — shows the pipeline structure
3. **Filter for errors** — search for `error`, `fail`, `##[error]`, `Process completed with exit code`

```python
import subprocess, json, re

API_KEY = subprocess.check_output(
    ["security", "find-generic-password", "-s", "log_api_key", "-a", "MUYYANG", "-w"]
).decode().strip()

import urllib.request

url = f"http://log-api.c82p401.c82.cloud.corpintra.net/cicd/logs?run_id=RUN_ID&from_expr=now-6d&size=5000&sort_order=asc"
req = urllib.request.Request(url, headers={"X-API-Key": API_KEY})
with urllib.request.urlopen(req) as r:
    d = json.load(r)

logs = d["logs"]
# List unique jobs
print("Jobs:", sorted(set(l.get("job_name","") for l in logs)))

# Filter error/key lines
keywords = ["error","Error","ERROR","fail","Fail","FAIL","##[error]",
            "Process completed","exit code","DEPLOYMENT_FAILED","BUILD_FAILED"]
for l in logs:
    msg = re.sub(r"^\d{4}-\d{2}-\d{2}T[\d:.Z]+\s+", "", l.get("msg","").strip())
    if any(k in msg for k in keywords):
        print(msg[:200])
```

### Common Failure Patterns

| Error in logs | Likely cause | Fix |
|---------------|-------------|-----|
| `sh: 1: cross-env: not found` / `exit code: 127` | `npm install` missing or failed before `npm run build` | Add `RUN npm ci` before `RUN npm run build` in Dockerfile |
| `npm error Exit handler never called!` after 72s | Corporate Artifactory `.npmrc` credentials not available inside Docker build | Pass `NPM_TOKEN` as build arg: `--build-arg NPM_TOKEN=${{ secrets.NPM_TOKEN }}` and configure `.npmrc` inside Dockerfile |
| `DEPLOYMENT_FAILED` in Update Status job | Helm deploy failed after image build | Check `helm upgrade` step output; look for K8s events |
| `unauthorized` / `denied` on `docker push` | Registry credentials expired or missing | Check `REGISTRY_DNA_USER` / `robot_dnaplatform+cicdbot` credentials |
| `Image pull backoff` in K8s app logs | New image tag not pushed; pod can't pull | Verify `docker push` completed successfully in CI/CD logs |
| `vault agent` errors in pod init | Vault secret path missing or role not configured | Check `vault_hashicorp_com/agent-inject-secret-*` annotation and Vault path |

---

## 7. Index Retention & Limits

| | CI/CD (`ci-cd-*`) | Kubernetes (`kubernetes_*`) |
|---|---|---|
| Retention | ~6 days | ~6 days |
| Total docs | ~4M | ~45M+ |
| Max query size | 10 000 rows | 10 000 rows |
| Pagination | ❌ Not supported | ❌ Not supported |

> Use `from_expr=now-6d` to cover maximum available history.

---

## 8. Full Analysis Example (Python one-shot)

Analyse a failed build end-to-end — find run_id then dump errors:

```python
import httpx, json, re, subprocess, warnings
warnings.filterwarnings("ignore")

API_KEY = subprocess.check_output(
    ["security", "find-generic-password", "-s", "log_api_key", "-a", "MUYYANG", "-w"]
).decode().strip()

BASE_API = "http://log-api.c82p401.c82.cloud.corpintra.net"
OS_BASE  = "https://logs.dna-prod.app.corpintra.net"

# Step 1: find run_id via keyword search
r = httpx.get(f"{OS_BASE}/auth/anonymous", follow_redirects=True, verify=False, timeout=15)
cookie = r.cookies.get("security_authentication")

q = {
    "size": 10,
    "query": {"bool": {"filter": [
        {"query_string": {"query": "msmcore", "default_field": "*"}},
        {"range": {"@timestamp": {"gte": "now-6d"}}}
    ]}},
    "sort": [{"@timestamp": {"order": "desc"}}],
    "_source": ["run_id"]
}
resp = httpx.post(f"{OS_BASE}/internal/search/opensearch",
    json={"params": {"index": "ci-cd-*", "body": q}},
    headers={"Content-Type":"application/json","osd-xsrf":"true",
             "osd-version":"2.17.1","securitytenant":"anonymous",
             "Cookie": f"security_authentication={cookie}"},
    verify=False, timeout=30)
run_id = resp.json()["rawResponse"]["hits"]["hits"][0]["_source"]["run_id"]
print(f"Found run_id: {run_id}")

# Step 2: fetch all logs and show errors
resp2 = httpx.get(f"{BASE_API}/cicd/logs",
    params={"run_id": run_id, "from_expr": "now-6d", "size": 5000, "sort_order": "asc"},
    headers={"X-API-Key": API_KEY}, timeout=60)
logs = resp2.json()["logs"]

errors = [k for k in ["error","Error","ERROR","##[error]","exit code","BUILD_FAILED","DEPLOYMENT_FAILED","not found"]]
print(f"\n=== Errors in run {run_id} ===")
for l in logs:
    msg = re.sub(r"^\d{4}-\d{2}-\d{2}T[\d:.Z]+\s+", "", l.get("msg","").strip())
    if any(e in msg for e in errors):
        print(msg[:220])
```

---

## 9. Discover All Active Instances (Aggregation)

Use this **before** querying logs when you don't know the exact instance name. Aggregations return
counts per instance without fetching all log rows — very efficient.

```python
import httpx, json, warnings
warnings.filterwarnings("ignore")

BASE = "https://logs.dna-prod.app.corpintra.net"
r = httpx.get(f"{BASE}/auth/anonymous", follow_redirects=True, verify=False, timeout=15)
cookie = r.cookies.get("security_authentication")

headers = {
    "Content-Type": "application/json",
    "osd-xsrf": "true", "osd-version": "2.17.1",
    "securitytenant": "anonymous",
    "Cookie": f"security_authentication={cookie}"
}

def discover_instances(keyword: str, hours: int = 24) -> list[dict]:
    """Find all k8s instances whose name contains keyword, sorted by log volume."""
    q = {
        "size": 0,
        "query": {"bool": {"filter": [
            {"range": {"@flb-timestamp": {"gte": f"now-{hours}h", "lte": "now"}}},
            {"query_string": {
                "query": keyword,
                "default_field": "k8s_resource_labels.app_kubernetes_io/instance"
            }}
        ]}},
        "aggs": {
            "instances": {
                "terms": {
                    "field": "k8s_resource_labels.app_kubernetes_io/instance.keyword",
                    "size": 50
                }
            }
        }
    }
    resp = httpx.post(f"{BASE}/internal/search/opensearch",
        json={"params": {"index": "kubernetes_*", "body": q}},
        headers=headers, verify=False, timeout=30)
    return resp.json().get("rawResponse", {}).get("aggregations", {}).get("instances", {}).get("buckets", [])

# Example: find all OneCM + CoMPaS instances
for bucket in discover_instances("onecm OR compas OR com-pas"):
    print(f"  {bucket['key']:50s}  {bucket['doc_count']:>8,} docs/24h")
```

**Known OneCM/CoMPaS instances (as of 2026-03-18):**

| Instance | Docs/24h | Notes |
|---|---|---|
| `onecm-authorization-int` | 701 K | Auth service — highest volume |
| `onecm-prod-backend-prod` | 223 K | Main backend prod |
| `onecm-prod-backend-int` | 128 K | Main backend int |
| `onecm-user-frontend-prod` | 37 K | User frontend prod |
| `onecm-user-frontend-int` | 36 K | User frontend int |
| `fc-onecm-mec-frontend-dev-int` | 35 K | MEC frontend dev |
| `fc-onecm-compas-frontend-dev-int` | 35 K | CoMPaS frontend dev |
| `onecm-java-backend-int` | 30 K | Java backend int |
| `onecm-java-backend-prod` | 15 K | Java backend prod |
| `fc-onecm-mec-backend-dev-int` | 10 K | MEC backend dev |
| `fc-onecm-compas-backend-dev-int` | 3 K | CoMPaS backend dev |

> ⚠️ **CMPlan** is Azure SQL–based and does **not** appear in `kubernetes_*` logs.

---

## 10. Parallel Multi-Instance Error Scan

When checking the health of multiple apps at once, query them in parallel:

```bash
API_KEY=$(security find-generic-password -s "log_api_key" -a "MUYYANG" -w)
BASE="http://log-api.c82p401.c82.cloud.corpintra.net"

INSTANCES=(
  "onecm-prod-backend-int"
  "onecm-prod-backend-prod"
  "onecm-java-backend-int"
  "onecm-java-backend-prod"
  "fc-onecm-mec-backend-dev-int"
  "fc-onecm-compas-backend-dev-int"
  "onecm-authorization-int"
)

for instance in "${INSTANCES[@]}"; do
  curl -s -H "X-API-Key: $API_KEY" \
    "$BASE/app/logs?instance=$instance&from_expr=now-24h&size=200&level=ERROR&sort_order=desc" \
    | python3 -c "
import sys, json
d = json.load(sys.stdin)
total = d['total']
if total > 0:
    print(f'=== $instance: {total} errors ===')
    for l in d['logs'][:3]:
        print(f'  [{l.get(\"@flb-timestamp\",\"\")[:19]}] {l.get(\"log\",\"\")[:120]}')
" 2>/dev/null &
done
wait
```

**Python version with error categorization:**

```python
import httpx, re, subprocess
from collections import Counter

API_KEY = subprocess.check_output(
    ["security", "find-generic-password", "-s", "log_api_key", "-a", "MUYYANG", "-w"]
).decode().strip()
BASE = "http://log-api.c82p401.c82.cloud.corpintra.net"
HEADERS = {"X-API-Key": API_KEY}

INSTANCES = [
    "onecm-prod-backend-int", "onecm-prod-backend-prod",
    "onecm-java-backend-int", "onecm-java-backend-prod",
    "onecm-authorization-int",
]

for instance in INSTANCES:
    r = httpx.get(f"{BASE}/app/logs",
        params={"instance": instance, "from_expr": "now-24h", "size": 500, "level": "ERROR"},
        headers=HEADERS, timeout=30)
    logs = r.json()["logs"]
    total = r.json()["total"]
    if total == 0:
        continue
    print(f"\n=== {instance}: {total} total errors ===")
    patterns = Counter()
    samples = {}
    for l in logs:
        log = l.get("log", "")
        m = re.search(
            r"(CannotGetJdbc|JDBC|Redis|DnaAliceApiServiceImpl|CallPipelineService"
            r"|AADSTS|401|400|GlobalException|timeout|refused)", log)
        if m:
            key = m.group(1)
            patterns[key] += 1
            if key not in samples:
                samples[key] = log[:200]
    for key, count in patterns.most_common(5):
        print(f"  [{count}x] {key}")
        print(f"         {samples[key][:150]}")
```

---

## 11. Known Runtime Error Patterns (OneCM / CoMPaS)

| Error pattern | Instance(s) | Meaning | Action |
|---|---|---|---|
| `CannotGetJdbcConnectionException: Failed to obtain JDBC Connection` + `createElapseMillis` in billions | `onecm-authorization-int` | Druid connection pool stuck — one zombie connection is blocking all new connections. `active 0, maxActive 120, creating 1` means the pool is waiting on a hung create. | Restart the pod; investigate DB-side slow connection (network timeout, DB overload). Check if Druid `connection-timeout` is configured. |
| `DnaAliceApiServiceImpl: Getting all entitlements for user: <USERNAME>` + response with entitlements | All backend instances | Logged at ERROR level but is **not a real error** — it's the Alice authorization service returning entitlements for a user who lacks the required role. The app is correctly denying access. | Reduce log level for this case to WARN or INFO in the app config. |
| `CallPipelineServiceImpl: 获取令牌失败，状态码: 400` + `AADSTS90002: Tenant '...' not found` | `onecm-prod-backend-prod` | Azure AD tenant ID in the app config is wrong or the service principal was moved to a different tenant. | Check `spring.security.oauth2` or equivalent config — update the tenant ID. Verify the service principal still exists in the target tenant. |
| `CallPipelineServiceImpl: API调用失败，状态码: 401` + `InvalidToken: Access token is invalid` | `onecm-prod-backend-prod` | The cached OAuth token is expired or the service principal client secret has rotated/expired. | Rotate the client secret in Azure AD and update the K8s secret / Vault path. |
| `ApplicationAuthenticationFilter: Unable to connect to Redis` | `onecm-java-backend-int` | Redis cache is unreachable — auth filter can't validate sessions. Transient or Redis pod restart. | Check the Redis pod status in the same namespace. If recurring, check Redis memory/eviction config. |
| `GlobalExceptionHandler: error:` (no detail) | `fc-onecm-mec-backend-dev-int` | Unhandled exception caught by global handler. Details not printed in this log line. | Query surrounding log lines (±30s) using `from_expr` / `to_expr` window to see the full stack trace. |

---

## 12. Performance Analysis — Stored Procedure Slowness (CashUp US)

### Overview

`cashup-us-report-backend-pg-prod` is the report-generation backend for CashUp US.
It logs performance via **【性能监控】** (performance monitor) tags. Use these to profile DB latency.

### Instance naming

| Service | Instance name |
|---------|--------------|
| Upload/submit backend | `cashup-us-backend-pg-prod` |
| Report generation backend | `cashup-us-report-backend-pg-prod` |
| INT/staging | `cashup-us-backend-int` |

> **Note**: `cashup-us-backend-pg-prod` logs at WARN level only — it will appear silent
> when no file uploads are occurring. Absence of logs ≠ pod is crashed.

### Step 1 — Fetch timing logs

```python
# Only fetch perf-monitor and SP logs (much smaller result set)
r = httpx.get(f"{BASE}/app/logs",
    params={"instance": "cashup-us-report-backend-pg-prod",
            "from_expr": "now-24h", "size": 5000,
            "keyword": "性能监控 OR 存储过程 OR 耗时 OR 保存"},
    headers=HEADERS, timeout=60)
```

### Step 2 — Parse SP timings

```python
import re
from collections import defaultdict

sp_times = defaultdict(list)
for l in logs:
    log = l.get("log", "")
    ms_m = re.search(r'耗时[=: ]+(\d+)\s*ms', log)
    if not ms_m: continue
    ms = int(ms_m.group(1))
    sp_m = re.search(r'EXEC (cashup_us\.\w+)', log)
    key = sp_m.group(1) if sp_m else re.search(r'(generate\w+|update\w+|insert\w+)', log, re.I)
    if key:
        sp_times[key if isinstance(key,str) else key.group(1)].append(ms)

for op, times in sorted(sp_times.items(), key=lambda x: -sum(x[1])/len(x[1])):
    avg = sum(times)/len(times)
    print(f"{op}: calls={len(times)} avg={avg:.0f}ms max={max(times):,}ms")
```

### Confirmed performance profile (CashUp US, 2026-03-17)

| Stored procedure | Avg | Max | Calls/day |
|---|---|---|---|
| `cashup_us.generate_forecast_report` | **5,365 ms** | 5,918 ms | 5 |
| `generateReportMonthlyTotal` | **4,313 ms** | 6,156 ms | 5 |
| `cashup_us.insert_into_ads_liquidity_proc_data` | **3,025 ms** | 3,596 ms | 8 |
| `updateLiquidityData` (per entity) | **2,784 ms** | 3,785 ms | 14 |
| `generateUnpublishedDataWithLock` (total) | **2,259 ms** | 4,779 ms | 13 |
| `cashup_us.generate_ihb_adj_data` | 1,266 ms | 1,385 ms | 5 |
| `cashup_us.generate_liquidity_adj_data` | 598 ms | 900 ms | 8 |
| DB batch insert (~3000 rows) | 2,500 ms (cold) → 500 ms (warm) | 5,679 ms | varies |

### Root cause analysis

1. **Azure SQL stored procedures are the primary bottleneck** — `generate_forecast_report` alone
   takes 5-6 seconds per call. Each report page chains 2-4 stored procs → **15-25s total latency**.

2. **Parameter sniffing / cold plan cache** — DB inserts go from 5.7s (first call) → 1.6s → 0.5s.
   Stored procs likely suffer the same: first execution per param set generates a slow plan.

3. **Lock serialization** — `generateUnpublishedDataWithLock` uses a distributed lock.
   Concurrent users wait for each other, compounding perceived slowness.

4. **SQL dialect translation overhead** — app converts PostgreSQL syntax to MSSQL T-SQL on-the-fly
   (`extract(YEAR FROM ...)` → `YEAR(CAST(GETDATE() AS DATE))`). Minor overhead, not the main cause.

### Recommended fixes

| Fix | Target | Expected impact |
|-----|--------|-----------------|
| `UPDATE STATISTICS` on all `cashup_us.*` tables | Azure SQL DBA | Refresh query plans, likely 50-80% speedup |
| Add `OPTION (OPTIMIZE FOR UNKNOWN)` to heavy SPs | Dev | Eliminate parameter sniffing |
| Cache report output per date+entity (Redis / in-memory) | Dev | Avoid repeated SP calls for same params |
| Profile SP with `SET STATISTICS TIME ON` in SSMS | Dev/DBA | Identify table scan vs index seek bottleneck |
| Move `generateUnpublishedDataWithLock` to async/background job | Dev | Remove user-visible wait |

### Known false-positive ERROR patterns in CashUp

| Pattern | Class | Reality |
|---------|-------|---------|
| Financial data values logged per category (category :Closing JPM Bank Balance, value: 370) | `VersionServiceImpl` | Debug data dump at ERROR level — not a real error, should be DEBUG |
| `生成工作日：261 个，从 2026-01-01 到 2026-12-31` | `LiquidityDataService` | Informational logged at ERROR — not a real error |
| `未找到类别 <NAME> 的去年期末余额，使用默认值0` | `LiquidityDataService` | WARN: missing prior year-end balance, falls back to 0 — data quality issue, not service failure |

---

## 13. DNA Codespaces Deployment Debug Loop — Crash Analysis & Iterative Fix

Use this workflow whenever a DNA Codespaces deployment shows `DEPLOYMENT_FAILED` or the pod is
crash-looping. The loop is: **check logs → classify failure → fix via GHE API → rebuild → repeat**.

### Overview

```
User reports DEPLOYMENT_FAILED
        │
        ▼
GET /app/logs?instance=<name>-int&from_expr=now-1h&size=500
        │
        ├─ No new pod? ─────────────────────────────────────────▶ Build step failed (check CI/CD logs)
        │                                                          GET /cicd/logs?run_id=<id>
        │
        ├─ Pod exists, 0 or 1 log lines, then dies ────────────▶ Import-time crash (Python/Node)
        │                                                          Read traceback, fix in source
        │
        ├─ Pod running but probes → 404 / 5xx ─────────────────▶ Missing health endpoint
        │                                                          Add /healthz route
        │
        └─ Pod running, probe passes but app broken ────────────▶ Runtime config error
                                                                   Check env vars, DB URL, secrets
```

---

### Step 1 — Pod Lifecycle Analysis (identify which build version is failing)

```python
import json, subprocess, urllib.request

API_KEY = subprocess.check_output(
    ["security", "find-generic-password", "-s", "log_api_key", "-w"],
    stderr=subprocess.DEVNULL
).decode().strip()

INSTANCE = "fcos-cdt-portfolio-management-int"  # adjust to your codespace

req = urllib.request.Request(
    f"http://log-api.c82p401.c82.cloud.corpintra.net/app/logs"
    f"?instance={INSTANCE}&from_expr=now-2h&size=500",
    headers={"X-API-Key": API_KEY}
)
with urllib.request.urlopen(req) as r:
    data = json.load(r)

pods = {}
for l in data["logs"]:
    pod  = l["kubernetes"]["pod_name"]
    img  = l["kubernetes"]["container_image"]   # e.g. :int-v4
    t    = l["time"]
    if pod not in pods:
        pods[pod] = {"first": t, "last": t, "image": img, "lines": 0}
    if t < pods[pod]["first"]: pods[pod]["first"] = t
    if t > pods[pod]["last"]:  pods[pod]["last"]  = t
    pods[pod]["lines"] += 1

for pod, v in sorted(pods.items(), key=lambda x: x[1]["first"]):
    tag = v["image"].split(":")[-1]   # e.g. int-v4
    print(f"{tag:8s}  lines={v['lines']:4d}  {v['first'][11:19]} → {v['last'][11:19]}  {pod[-10:]}")
```

**Interpreting output:**

| Lines | Duration | Meaning |
|-------|----------|---------|
| 1–5 | < 1s | Import-time crash — app died before first health check |
| 5–30 | 1–60s | Startup crash or immediate probe failure |
| 30–200 | 1–10 min | Probe failure loop (app running, but probes 404/5xx) |
| Stable, still ticking | Ongoing | Running pod (old healthy version still serving) |

---

### Step 2 — Get Startup Logs for the Failing Pod

```python
# Get startup sequence (asc = chronological) for a specific pod
import json, subprocess, urllib.request

API_KEY = subprocess.check_output(
    ["security", "find-generic-password", "-s", "log_api_key", "-w"],
    stderr=subprocess.DEVNULL
).decode().strip()

INSTANCE = "fcos-cdt-portfolio-management-int"
FAILING_POD_SUFFIX = "wgz9n"   # last 5 chars of pod hash

req = urllib.request.Request(
    f"http://log-api.c82p401.c82.cloud.corpintra.net/app/logs"
    f"?instance={INSTANCE}&from_expr=now-2h&size=500&sort_order=asc",
    headers={"X-API-Key": API_KEY}
)
with urllib.request.urlopen(req) as r:
    data = json.load(r)

print(f"=== Startup logs for pod ending in {FAILING_POD_SUFFIX} ===")
for l in data["logs"]:
    if FAILING_POD_SUFFIX in l["kubernetes"]["pod_name"]:
        print(l["time"][11:19], l.get("log", "").strip()[:250])
```

---

### Step 3 — Classify Failure Type & Common Patterns

#### Type A: Import-time crash (Python)
**Signal:** Pod has 1–30 log lines all at the same timestamp, all from `<frozen importlib>`.
```
AttributeError: FieldChange
File "/code/app/crud/epic.py", line 114, in <module>
) -> dict[str, EpicImportResult.FieldChange]:
```
**Cause:** Pydantic v2 evaluates type annotations at import time (`__getattr__` raises for unknown
nested model attributes). Wrong nested type → `AttributeError` before `uvicorn` can serve requests.  
**Fix:** Replace `OuterModel.InnerClass` with the correct top-level model that is already imported.

#### Type B: Health probe 404 (k8s kills healthy app)
**Signal:** Pod starts, uvicorn logs `Application startup complete.`, then every 10s:
```
GET /fcos-cdt-portfolio-management/int//healthz HTTP/1.1  404 Not Found
```
After 5–10 failures Kubernetes terminates the pod.  
**Cause:** DNA Codespaces platform probes `GET /healthz` on the container port. The app only has
`GET /api/v1/health`.  
**Fix:** Add `/healthz` route alongside the existing health endpoint:
```python
@app.get("/healthz", tags=["health"])
@app.get("/api/v1/health", tags=["health"])
def health():
    return {"status": "ok"}
```

#### Type C: Wrong path resolution in Docker
**Signal:** App starts, DB log shows suspicious path:
```
[database] Using DATABASE_URL: sqlite:////portfolio_ops.db   ← root filesystem!
```
**Cause:** `Path(__file__).resolve().parents[N]` resolves differently in Docker vs local dev.
In Docker with `COPY backend/ /code/`, `database.py` is at `/code/app/database.py`:
- `parents[0]` = `/code/app`
- `parents[1]` = `/code`  ← correct for project root in Docker
- `parents[2]` = `/`     ← filesystem root (wrong!)

**Fix:** Use `parents[1]` when the Docker working dir is `/code` and code is at `/code/app/*.py`.

#### Type D: Missing `deploy/build.Dockerfile`
**Signal:** Build step fails; no new pod appears at all in k8s logs.  
**Cause:** DNA Codespaces CI/CD expects `deploy/build.Dockerfile` at **exactly** that path in the
repo root. The build pipeline fails before producing an image.  
**Fix:** Create `deploy/build.Dockerfile` building from repo root context:
```dockerfile
FROM python:3.12.1-slim
WORKDIR /code
COPY backend/requirements.txt /code/
RUN pip install --no-cache-dir -r requirements.txt
COPY backend/ /code/
EXPOSE 8080
CMD ["sh", "-c", "uvicorn app.main:app --root-path ${ROOT_PATH_NON_API} --host 0.0.0.0 --port 8080"]
```

#### Type E: Frontend with path-prefix routing (single-service platform)
**Signal:** App deploys, frontend loads blank page or navigating to `/dashboard` gives 404.  
**Cause:** DNA Codespaces injects `ROOT_PATH_NON_API=/codespace-name/int/` (with trailing slash).
`BrowserRouter` without a `basename` doesn't know where the app root is, so routes don't match.
Vite asset paths like `/assets/main.js` (absolute) never reach the pod through the proxy.  
**Fix pattern:**
```
deploy/build.Dockerfile  → multi-stage: Node builds frontend, Python serves it via StaticFiles
frontend/vite.config.ts  → base: './'   (relative asset paths survive the path prefix)
frontend/src/App.tsx     → <HashRouter>  (hash routing avoids basename config entirely)
backend/app/main.py      → mount /code/static/assets + catch-all FileResponse(index.html)
backend/requirements.txt → add aiofiles  (required by FastAPI StaticFiles)
```

---

### Step 4 — Fix via GHE API (No git clone required)

Push targeted fixes directly using the GHE contents API. Much faster than clone → edit → push,
especially on corporate VPN. Requires only the GHE PAT from macOS Keychain.

```python
import base64, json, subprocess, urllib.request, urllib.error

TOKEN = subprocess.check_output(
    ["security", "find-internet-password", "-s", "mercedes-benz.ghe.com", "-w"],
    stderr=subprocess.DEVNULL
).decode().strip()

ORG  = "DNA-CodeSpaces"
REPO = "FCOS-CDT-Portfolio-Management"
BASE = f"https://mercedes-benz.ghe.com/api/v3/repos/{ORG}/{REPO}/contents"

def ghe_get_sha(path: str, branch: str = "feature/deployment-config") -> str:
    """Get the current blob SHA — required for updates."""
    req = urllib.request.Request(
        f"{BASE}/{path}?ref={branch}",
        headers={"Authorization": f"token {TOKEN}"}
    )
    with urllib.request.urlopen(req) as r:
        return json.load(r)["sha"]

def ghe_push(path: str, new_content: str, message: str,
             branch: str = "feature/deployment-config") -> str:
    """Create or update a file. Returns the new commit SHA."""
    sha = ghe_get_sha(path, branch)
    payload = json.dumps({
        "message": message + "\n\nCo-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>",
        "content": base64.b64encode(new_content.encode()).decode(),
        "sha": sha,
        "branch": branch
    }).encode()
    req = urllib.request.Request(
        f"{BASE}/{path}", data=payload, method="PUT",
        headers={"Authorization": f"token {TOKEN}", "Content-Type": "application/json"}
    )
    with urllib.request.urlopen(req) as r:
        return json.load(r)["commit"]["sha"][:10]

# Example: fix /healthz missing from main.py
new_main = ghe_get_current_content("backend/app/main.py")   # read existing
new_main = new_main.replace(
    '@app.get("/api/v1/health"',
    '@app.get("/healthz", tags=["health"])\n@app.get("/api/v1/health"'
)
commit_sha = ghe_push("backend/app/main.py", new_main, "fix: add /healthz endpoint")
print(f"pushed: {commit_sha}")
```

**Helper to read current file content:**
```python
def ghe_get_current_content(path: str, branch: str = "feature/deployment-config") -> str:
    req = urllib.request.Request(
        f"{BASE}/{path}?ref={branch}",
        headers={"Authorization": f"token {TOKEN}"}
    )
    with urllib.request.urlopen(req) as r:
        d = json.load(r)
        return base64.b64decode(d["content"]).decode()
```

---

### Step 5 — Trigger Rebuild & Monitor

After pushing fixes, instruct the user to rebuild in the DNA Codespaces UI
(`feature/deployment-config` → Build → int). Then poll for the new pod:

```python
import time

def wait_for_new_pod(instance: str, known_pods: set, api_key: str, timeout: int = 600):
    """Poll until a pod hash not in known_pods appears in logs."""
    deadline = time.time() + timeout
    while time.time() < deadline:
        req = urllib.request.Request(
            f"http://log-api.c82p401.c82.cloud.corpintra.net/app/logs"
            f"?instance={instance}&from_expr=now-5m&size=50",
            headers={"X-API-Key": api_key}
        )
        with urllib.request.urlopen(req) as r:
            logs = json.load(r)["logs"]
        pods_now = {l["kubernetes"]["pod_name"] for l in logs}
        new_pods = pods_now - known_pods
        if new_pods:
            return new_pods
        print(f"  waiting for new pod... ({int(deadline - time.time())}s left)")
        time.sleep(30)
    return set()
```

---

### DNA Codespaces — Deployment Failure Quick Reference

| Symptom | First check | Likely fix |
|---------|-------------|-----------|
| No new pod after build | CI/CD logs (`/cicd/logs?run_id=<id>`) | Missing `deploy/build.Dockerfile`, npm/pip install error |
| Pod has 1 log line then gone | Startup traceback in that log line | Import-time crash — syntax error, wrong type annotation, missing import |
| Pod restarts every ~60s, probe → 404 | `GET /healthz` 404 in logs | Add `/healthz` route; check probe path in Helm values |
| Pod restarts every ~60s, probe → 500 | App exception on startup in logs | DB connection error, missing env var, wrong path |
| Pod stable but old image still serving | `container_image` tag in logs | New build not deployed; trigger Deploy step separately from Build |
| Frontend blank page | Browser console errors | Asset paths wrong (use `base: './'` in Vite), or `BrowserRouter` needs `basename` (use `HashRouter` instead) |
| Frontend 404 on hard refresh | No SPA catch-all in server | Add `/{full_path:path}` → `FileResponse(index.html)` |


---

## CDT AI Diagnosis Layer

> Absorbed from `cdt-dna-logs` — pattern-matched root cause analysis for FC.OS CDT stack

### Maven Build Failures

| Error Signature | Root Cause | Fix |
|---|---|---|
| `Could not find artifact ... in artifactory` | `settings.xml` missing or PAT expired | Place `settings.xml` in `~/.m2/`; regenerate PAT with SSO for DNA-CodeSpaces |
| `COMPILATION ERROR` + `cannot find symbol` | Wrong Java version or missing import | Verify JDK 17+: `java -version`; check `pom.xml` source/target |
| `Return code is: 401` from Artifactory | PAT not in `settings.xml` or expired | Update `<password>` in `settings.xml` with fresh PAT |

### npm / Vite Build Failures (CDT Frontend)

| Error Signature | Root Cause | Fix |
|---|---|---|
| `VITE_SPREAD_KEY is not defined` | License env var missing | Add `VITE_SPREAD_KEY`, `VITE_SPREAD_DESIGNER_KEY`, `VITE_WIJMO_KEY` to `.env` or CodeSpace secrets |
| `The key is invalid` (SpreadJS console) | Wrong or expired SpreadJS 18.x license | Request current key from team lead |
| `ENOENT: no such file or directory, open 'build/...'` | Output dir mismatch | Ensure `vite.config.ts` sets `outDir: 'build'` (not `dist/`) |
| `npm ERR! code ERESOLVE` | Peer dependency conflict | `npm install --legacy-peer-deps` |

### Kubernetes / DyP-CaaS Deployment Failures

| Error Signature | Root Cause | Fix |
|---|---|---|
| `CrashLoopBackOff` | App exits immediately | Check startup logs for datasource / Vault config errors |
| `ImagePullBackOff` | Docker image not found | Verify image tag exists; check registry credentials |
| `OOMKilled` | Container exceeded memory limit | Increase `resources.limits.memory` in Helm `deploy/helm/values.yaml` |
| `Liveness probe failed` | App not responding on health endpoint | Verify `/actuator/health` returns 200; check port matches Helm config |

### Spring Boot Startup Failures

| Error Signature | Root Cause | Fix |
|---|---|---|
| `Failed to configure a DataSource` | DB connection string missing | Set `spring.datasource.url` in `application.yml` or Vault |
| `Connection refused: <postgres-host>:5432` | Wrong DB host or VPN not active | Verify DB host; confirm VPN active |
| `Error creating bean ... Vault` | Vault unreachable or wrong path | Check `VAULT_ADDR` + `VAULT_PATH` env vars in CodeSpace secrets |

### GCExcel / ALICE Errors

| Error Signature | Root Cause | Fix |
|---|---|---|
| `License key is invalid or expired` (GCExcel) | GCExcel license not initialized | Ensure license init called at startup with `GCEXCEL_LICENSE` env var |
| `403 Forbidden` from app API | ALICE role missing | Check via `dna-alice-auth-data-service` skill: `GET /user/{id}/roles` |
| `vaultInjector: secret not found` | Wrong Vault path in CodeSpace config | Verify `codespaces_kv` path matches actual Vault secret path |