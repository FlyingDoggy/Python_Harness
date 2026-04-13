---
name: db-migration-ops
version: "1.0.0"
description: "Alembic database migration operations for DNA CodeSpaces deployments. Covers: creating migrations, Helm init container auto-migration, stamp-on-failure self-healing, diagnosing migration failures via log-api, Azure SQL + Vault auth, common error patterns (42S01 table exists, 42S02 table missing, alembic_version empty). USE FOR: alembic migration, database schema change, add column, create table, migration failed, 42S01, 42S02, table already exists, alembic_version empty, alembic stamp, init container crash, helm deploy timeout, context deadline exceeded, DEPLOYMENT_FAILED due to migration, DB schema out of sync, create_all vs alembic, how to add a new migration."
roles: [developer, platform-engineer]
domain: database
---

# Database Migration Operations — Alembic + DNA CodeSpaces

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    DNA K8s Pod Startup                       │
│                                                             │
│  1. Vault init ─→ inject /vault/secrets/envvar              │
│  2. db-migrate init ─→ alembic upgrade head (auto-stamp)    │
│  3. backend container ─→ uvicorn (+ create_all fallback)    │
└─────────────────────────────────────────────────────────────┘
```

**两套建表机制**（必须理解）：

| 机制 | 触发时机 | 行为 | 管理新列？ |
|------|---------|------|-----------|
| **Alembic 迁移** | init container 或手动 `alembic upgrade head` | 按版本链顺序执行 DDL | ✅ 是 |
| **`Base.metadata.create_all()`** | `main.py` 启动时 | 创建不存在的表（从 ORM 模型） | ❌ 只管新表，不改已有表 |

> ⚠️ `create_all` 只能创建全新的表。如果表已存在但缺列，它不会 ALTER TABLE。必须用 Alembic 迁移。

---

## 1. 创建新迁移

### 1.1 添加新列到已有表

```bash
cd backend

# 1. 先修改 ORM 模型（app/models/*.py）
# 2. 生成迁移文件
alembic revision -m "描述信息"

# 3. 编辑生成的迁移文件
```

迁移文件模板（添加列）：

```python
"""020 add wiki import source fields"""
from alembic import op
import sqlalchemy as sa

revision = "020"
down_revision = "019"
branch_labels = None
depends_on = None

def upgrade() -> None:
    op.add_column("wiki_pages", sa.Column("source_file_path", sa.String(500), nullable=True))
    op.add_column("wiki_pages", sa.Column("import_source", sa.String(300), nullable=True))

def downgrade() -> None:
    op.drop_column("wiki_pages", "import_source")
    op.drop_column("wiki_pages", "source_file_path")
```

### 1.2 创建新表

```python
def upgrade() -> None:
    op.create_table(
        "wiki_pages",
        sa.Column("id", sa.Uuid(), primary_key=True),
        sa.Column("product_id", sa.String(50), sa.ForeignKey("knowledge_products.id"), nullable=False),
        sa.Column("title", sa.String(500), nullable=False),
        # ... 其他列
    )
    op.create_index("ix_wiki_pages_product_id", "wiki_pages", ["product_id"])
```

### 1.3 版本号规则

- 文件名：`{NNN}_{描述}.py`（如 `020_add_wiki_import_source_fields.py`）
- `revision = "020"`
- `down_revision = "019"`（指向上一个版本）
- 查看当前 head：`alembic heads`

---

## 2. Alembic 连接配置

`backend/alembic/env.py` 的 `_build_sync_url()` 按优先级构建连接：

| 优先级 | 来源 | 用途 |
|--------|------|------|
| 1 | `DB_SERVER` + `DB_CLIENT_ID` + `DB_CLIENT_SECRET` | Azure SQL SP 认证（DNA 部署） |
| 2 | `DATABASE_URL` 环境变量（`aioodbc` → `pyodbc` 自动转换） | 本地开发 / Docker |
| 3 | `alembic.ini` 的 `sqlalchemy.url` | 兜底默认值 |

DNA Vault 注入的环境变量通过 `/vault/secrets/envvar` 文件提供，init container 用 `set -a && . /vault/secrets/envvar && set +a` 加载。

---

## 3. Helm Init Container 自动迁移

### 3.1 部署模板位置

`deploy/helm/templates/deployment.yaml` → `initContainers[0]: db-migrate`

### 3.2 自愈逻辑

```
alembic upgrade head
  ├── 成功 → 直接通过 ✅
  │   （正常情况：alembic_version 已有记录，从当前版本升级）
  │
  └── 失败 → 42S01 "table already exists"
        │   （首次部署：表由 create_all 创建，alembic_version 为空）
        │
        ├── alembic stamp head（标记当前版本到 alembic_version）
        └── alembic upgrade head（重试，这次从 head 开始，无迁移可跑）
            → 通过 ✅
```

### 3.3 为什么需要 stamp

当 DB 的表是通过 `Base.metadata.create_all()` 创建的（而非 Alembic），`alembic_version` 表为空。Alembic 会从 001 开始重跑所有迁移，撞上已存在的表就崩溃。

`alembic stamp head` 只在 `alembic_version` 表中写入当前版本号，不执行任何 DDL。之后 `upgrade head` 发现已在最新版本，直接跳过。

### 3.4 Vault + Init Container 启动顺序

```
Pod 创建
  ↓
Vault agent init container → 注入 /vault/secrets/envvar
  ↓  (annotation: vault.hashicorp.com/agent-init-first: 'true')
db-migrate init container → source envvar && alembic upgrade head
  ↓
backend container → uvicorn (表已就绪)
```

---

## 4. 诊断迁移失败

### 4.1 部署失败后查看 CI/CD 日志

```bash
# 1. 找到最新 run_id
API_KEY=$(security find-generic-password -s "log_api_key" -a "MUYYANG" -w)
BASE="http://log-api.c82p401.c82.cloud.corpintra.net"

curl -s -H "X-API-Key: $API_KEY" "$BASE/cicd/latest-run?wsid=fc-cdt-ai-ops" | python3 -m json.tool

# 2. 拉取部署日志，过滤错误
curl -s -H "X-API-Key: $API_KEY" \
  "$BASE/cicd/logs?run_id=<RUN_ID>&from_expr=now-6h&size=5000&sort_order=asc" \
  | python3 -c "
import sys, json, re
d = json.load(sys.stdin)
for l in d['logs']:
    msg = re.sub(r'^\d{4}-\d{2}-\d{2}T[\d:.Z]+\s+', '', l.get('msg','').strip())
    job = l.get('job_name', '')
    if any(k in msg for k in ['error','Error','ERROR','FAIL','##[error]','deadline','helm3']):
        print(f'[{job}] {msg[:300]}')
"
```

### 4.2 查看应用日志中的迁移错误

```bash
curl -s -H "X-API-Key: $API_KEY" \
  "$BASE/app/logs?instance=fc-cdt-ai-ops-int&minutes=30&order=asc&size=500" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
for l in d['logs']:
    log = l.get('log', '')
    if 'alembic' in log.lower() or '42S0' in log or 'already an object' in log or 'upgrade' in log.lower():
        print(f'[{l.get(\"@flb-timestamp\",\"\")[:19]}] {log[:250]}')
"
```

---

## 5. 常见错误模式与修复

### 5.1 42S01 — Table already exists

```
ProgrammingError: ('42S01', "There is already an object named 'roles' in the database")
```

**原因**：`alembic_version` 表为空，Alembic 从 001 开始跑，碰到已存在的表。
**修复**：init container 自动 `stamp head`（已实现自愈），或手动：
```bash
kubectl exec -it deployment/fc-cdt-ai-ops-int -n prod-dna-cs-apps -- \
  sh -c 'set -a && . /vault/secrets/envvar && set +a && alembic stamp head'
```

### 5.2 42S02 — Invalid object name (table missing)

```
ProgrammingError: ('42S02', "Invalid object name 'wiki_pages'")
```

**原因**：新表的迁移没有执行。可能是 Alembic 未跑到最新版本，或部署的代码版本包含新模型但迁移未跑。
**修复**：确保 `alembic upgrade head` 执行成功。检查 init container 日志。

### 5.3 Helm context deadline exceeded

```
Error: UPGRADE FAILED: context deadline exceeded
```

**原因**：init container 崩溃循环（CrashLoopBackOff），Pod 无法在 `--timeout=10m0s` 内就绪。
**修复**：通常是 5.1 的下游后果。修复迁移问题后重新部署。

### 5.4 pytds 加密错误

```
pytds.tds_base.Error: Client does not have encryption enabled but it is required by server
```

**原因**：某组件用了 `pytds` 直连 Azure SQL 但未启用 TLS。Azure SQL 要求加密连接。
**修复**：改用 `pyodbc` + `ODBC Driver 18`（默认加密），或者 pytds 连接参数加 `cafile`/`use_mars=True`。

---

## 6. 本地开发迁移操作

### 6.1 Docker Compose（OrbStack）

`docker-compose.local.yml` 的 `db-init` 服务自动执行：
```bash
python -m alembic upgrade head && python -m scripts.init_db
```

### 6.2 手动操作

```bash
cd backend

# 查看当前版本
docker exec aiops_backend alembic current

# 升级到最新
docker exec aiops_backend alembic upgrade head

# 查看 head
docker exec aiops_backend alembic heads

# 回滚一步
docker exec aiops_backend alembic downgrade -1

# 标记版本（不执行 DDL）
docker exec aiops_backend alembic stamp 020
```

### 6.3 验证表结构

```bash
docker exec aiops_backend python -c '
import pyodbc
conn = pyodbc.connect("DRIVER={ODBC Driver 18 for SQL Server};SERVER=mssql,1433;DATABASE=fcosdb_cdt_aiops_dev;UID=sa;PWD=AiOps_Dev_2024;TrustServerCertificate=yes")
cursor = conn.cursor()
cursor.execute("SELECT COLUMN_NAME, DATA_TYPE FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_NAME = '"'"'wiki_pages'"'"' ORDER BY ORDINAL_POSITION")
for r in cursor.fetchall():
    print(r[0], "|", r[1])
conn.close()
'
```

---

## 7. 迁移 Checklist（每次新增迁移时）

- [ ] ORM 模型已更新（`app/models/*.py`）
- [ ] Pydantic schema 已更新（`app/schemas/*.py`，如有新字段）
- [ ] 迁移文件已创建并审核（`alembic/versions/NNN_*.py`）
- [ ] `down_revision` 正确链接到上一个版本
- [ ] 本地 `alembic upgrade head` 通过
- [ ] `DATABASE.md` 已更新表结构和迁移历史
- [ ] Docker rebuild 后表结构验证通过

---

*最后更新: 2026-04-10 | 基于 Phase 11 部署失败排查经验总结*
