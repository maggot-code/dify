# Dify混合部署计划 (本地后端 + 容器中间件)

**部署日期**: 2026年1月
**环境**: macOS + OrbStack (Docker容器化)
**目标**: 仅启动后端服务，前端和中间件通过容器化部署

## 📋 执行概览

本计划将分为三个主要阶段：
1. **环境验证与准备** (预计15分钟)
2. **中间件容器部署** (预计10分钟)
3. **本地后端源代码启动** (预计20分钟)

---

## 第一阶段：环境验证与准备

### 1.1 验证Python版本管理器

**操作步骤**:
```bash
# 验证pyenv已安装
pyenv versions

# 验证当前使用Python版本（需要 >= 3.11, < 3.13）
python --version

# 如果版本不符，使用pyenv安装正确版本
pyenv install 3.12.0  # 推荐版本
pyenv local 3.12.0    # 为项目设置本地Python版本
```

**检查点**:
- ✓ Python版本为 3.11 或 3.12
- ✓ pyenv 已正确配置
- ✓ `.python-version` 文件已在项目根目录创建

### 1.2 验证uv包管理器

**操作步骤**:
```bash
# 验证uv已安装
uv --version

# 如未安装，通过brew安装（推荐）
brew install uv

# 或通过pip安装
pip install uv
```

**检查点**:
- ✓ uv 版本为最新稳定版
- ✓ `uv` 命令可直接在shell中访问

### 1.3 验证OrbStack和Docker

**操作步骤**:
```bash
# 验证OrbStack运行中
orbctl status

# 验证Docker命令可用
docker --version
docker compose --version

# 验证OrbStack虚拟化存储位置
ls -la /Users/codemaggot/dev
```

**检查点**:
- ✓ OrbStack 已启动
- ✓ Docker 和 Docker Compose 命令可用
- ✓ `/Users/codemaggot/dev` 目录存在且可写

### 1.4 验证Dify项目结构

**操作步骤**:
```bash
cd /Users/codemaggot/github.com/maggot-code/dify

# 验证主要目录结构
ls -la | grep -E '^d.*\s(api|web|docker)$'

# 验证关键文件存在
ls -l api/pyproject.toml docker/docker-compose.middleware.yaml docker/.env.example
```

**检查点**:
- ✓ `/api` 后端目录存在
- ✓ `/web` 前端目录存在
- ✓ `/docker` 目录存在
- ✓ `api/pyproject.toml` 包含Python依赖定义
- ✓ `docker/docker-compose.middleware.yaml` 存在

---

## 第二阶段：中间件容器部署

### 2.1 准备Docker Compose环境文件

**操作步骤**:

```bash
cd /Users/codemaggot/github.com/maggot-code/dify/docker

# 复制中间件配置文件
cp middleware.env.example middleware.env

# 编辑middleware.env文件，设置以下关键变量（使用你的编辑器）
# 需要修改的变量：
# - DB_TYPE=postgresql (保持默认)
# - DB_PASSWORD=<设置强密码>
# - REDIS_PASSWORD=<设置强密码>
# - PGDATA_HOST_VOLUME=/Users/codemaggot/dev/dify/postgresql
# - REDIS_HOST_VOLUME=/Users/codemaggot/dev/dify/redis
```

**关键环境变量**:

| 变量 | 建议值 | 说明 |
|------|--------|------|
| `DB_TYPE` | `postgresql` | 使用PostgreSQL作为主数据库 |
| `DB_USERNAME` | `postgres` | 数据库用户名 |
| `DB_PASSWORD` | `<强密码>` | 需要修改为强密码 |
| `DB_DATABASE` | `dify` | 数据库名称 |
| `POSTGRES_MAX_CONNECTIONS` | `100` | 最大连接数 |
| `REDIS_PASSWORD` | `<强密码>` | Redis密码 |
| `PGDATA_HOST_VOLUME` | `/Users/codemaggot/dev/dify/postgresql` | PostgreSQL数据卷位置 |
| `REDIS_HOST_VOLUME` | `/Users/codemaggot/dev/dify/redis` | Redis数据卷位置 |
| `EXPOSE_POSTGRES_PORT` | `5432` | 暴露的PostgreSQL端口 |
| `EXPOSE_REDIS_PORT` | `6379` | 暴露的Redis端口 |

**检查点**:
- ✓ `middleware.env` 文件已复制并编辑
- ✓ 所有密码已更改为强密码
- ✓ 数据卷路径已设置为 `/Users/codemaggot/dev/dify/*`

### 2.2 创建数据卷目录

**操作步骤**:

```bash
# 创建所需的数据目录
mkdir -p /Users/codemaggot/dev/dify/{postgresql,redis,weaviate}

# 验证目录已创建
ls -la /Users/codemaggot/dev/dify/
```

**检查点**:
- ✓ `/Users/codemaggot/dev/dify/postgresql` 目录存在
- ✓ `/Users/codemaggot/dev/dify/redis` 目录存在
- ✓ `/Users/codemaggot/dev/dify/weaviate` 目录存在

### 2.3 启动中间件服务

**操作步骤**:

```bash
cd /Users/codemaggot/github.com/maggot-code/dify/docker

# 使用middleware.env启动中间件容器（PostgreSQL + Redis + Weaviate）
docker compose --env-file middleware.env \
  -f docker-compose.middleware.yaml \
  -p dify \
  up -d

# 验证所有容器已启动
docker compose --env-file middleware.env \
  -f docker-compose.middleware.yaml \
  -p dify \
  ps

# 等待容器健康检查通过（大约30秒）
sleep 30

# 检查PostgreSQL连接
docker compose --env-file middleware.env \
  -f docker-compose.middleware.yaml \
  -p dify \
  exec db_postgres pg_isready -h localhost

# 检查Redis连接
docker compose --env-file middleware.env \
  -f docker-compose.middleware.yaml \
  -p dify \
  exec redis redis-cli ping
```

**关键服务信息**:

| 服务 | 容器端口 | 本地主机端口 | 健康检查 |
|------|----------|-------------|----------|
| PostgreSQL | 5432 | 5432 | `pg_isready` |
| Redis | 6379 | 6379 | Redis PING |
| Weaviate | 8080 | 8080 | HTTP Health endpoint |

**检查点**:
- ✓ `docker compose ps` 显示所有容器状态为 `Up`
- ✓ PostgreSQL 返回 `accepting connections`
- ✓ Redis 返回 `PONG`
- ✓ Weaviate 可通过 `curl http://localhost:8080/v1/.well-known/ready` 访问

### 2.4 验证容器网络连接

**操作步骤**:

```bash
# 查看Docker网络
docker network ls | grep dify

# 验证容器间通信
docker compose --env-file middleware.env \
  -f docker-compose.middleware.yaml \
  -p dify \
  exec api curl http://db_postgres:5432  # 应返回错误但证明连接存在
```

**检查点**:
- ✓ 存在 `dify_default` 或类似的Docker网络
- ✓ 容器可通过服务名相互通信

---

## 第三阶段：本地后端源代码启动

### 3.1 准备后端环境

**操作步骤**:

```bash
cd /Users/codemaggot/github.com/maggot-code/dify/api

# 复制环境文件
cp .env.example .env

# 生成SECRET_KEY（macOS）
secret_key=$(openssl rand -base64 42)
sed -i '' "/^SECRET_KEY=/c\\
SECRET_KEY=${secret_key}" .env
```

**编辑 `.env` 文件中的关键变量**:

```bash
# 编辑以下变量（使用nano、vim或编辑器打开.env）

# API URLs
CONSOLE_API_URL=http://localhost:5001
CONSOLE_WEB_URL=http://localhost:3000
SERVICE_API_URL=http://localhost:5001
APP_WEB_URL=http://localhost:3000
FILES_URL=http://localhost:5001
INTERNAL_FILES_URL=http://127.0.0.1:5001
TRIGGER_URL=http://localhost:5001

# Redis配置（与middleware.env一致）
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=<与middleware.env中设置的相同>
REDIS_DB=0

# Celery配置
CELERY_BROKER_URL=redis://:<REDIS_PASSWORD>@localhost:6379/1
CELERY_BACKEND=redis

# 数据库配置（与middleware.env一致）
DB_TYPE=postgresql
DB_USERNAME=postgres
DB_PASSWORD=<与middleware.env中设置的相同>
DB_HOST=localhost
DB_PORT=5432
DB_DATABASE=dify

# 存储配置（本地开发使用local存储）
STORAGE_TYPE=opendal
OPENDAL_SCHEME=fs
OPENDAL_FS_ROOT=storage

# 向量数据库配置
VECTOR_STORE=weaviate
WEAVIATE_ENDPOINT=http://localhost:8080

# 日志和调试
LOG_LEVEL=INFO
FLASK_DEBUG=true
DEBUG=false
```

**检查点**:
- ✓ `.env` 文件已复制
- ✓ `SECRET_KEY` 已生成为强随机值
- ✓ 所有数据库凭证与middleware.env一致
- ✓ Redis和PostgreSQL的主机名为 `localhost`

### 3.2 安装Python依赖

**操作步骤**:

```bash
cd /Users/codemaggot/github.com/maggot-code/dify/api

# 使用uv安装依赖（包括开发工具）
uv sync --dev

# 验证安装完成
uv pip list | head -20
```

**依赖说明**:

- 主要依赖通过 `pyproject.toml` 中 `[project]dependencies` 安装
- 开发工具通过 `[dependency-groups]dev` 安装
- 可选依赖组（storage、tools、vdb）通过 `[tool.uv]default-groups` 自动安装

**检查点**:
- ✓ 所有依赖安装完成（无ERROR信息）
- ✓ 关键包已安装：flask、sqlalchemy、celery、redis等
- ✓ Python虚拟环境已激活

### 3.3 初始化数据库

**操作步骤**:

```bash
cd /Users/codemaggot/github.com/maggot-code/dify/api

# 运行数据库迁移
uv run flask db upgrade

# 验证迁移成功
uv run flask db current  # 应显示当前迁移版本
```

**迁移说明**:

- Flask-Migrate 根据 `migrations/` 目录中的脚本执行数据库初始化
- 这会创建所有必要的表结构
- 迁移是幂等的（多次运行安全）

**检查点**:
- ✓ 迁移完成，未报告ERROR
- ✓ 数据库中已创建表结构
- ✓ `flask db current` 返回有效的版本信息

### 3.4 启动后端服务

**操作步骤**:

```bash
cd /Users/codemaggot/github.com/maggot-code/dify/api

# 方式一：开发模式启动（带调试）
uv run flask run --host 0.0.0.0 --port 5001 --debug

# 方式二：在另一个终端启动Celery Worker（用于异步任务）
# （在一个新的终端窗口中执行）
cd /Users/codemaggot/github.com/maggot-code/dify/api
uv run celery -A app.celery worker -P threads -c 2 --loglevel INFO \
  -Q dataset,priority_dataset,priority_pipeline,pipeline,mail,ops_trace,app_deletion,plugin,workflow_storage,conversation,workflow,schedule_poller,schedule_executor,triggered_workflow_dispatcher,trigger_refresh_executor,retention

# 方式三：启动Celery Beat（用于定时任务，可选）
# （在第三个终端窗口中执行）
cd /Users/codemaggot/github.com/maggot-code/dify/api
uv run celery -A app.celery beat
```

**启动日志示例**:

```
 * Serving Flask app 'app'
 * Debug mode: on
 * Running on http://0.0.0.0:5001
```

**检查点**:
- ✓ Flask开发服务器启动，监听 `0.0.0.0:5001`
- ✓ 日志中无ERROR或CRITICAL信息
- ✓ 连接到Redis成功
- ✓ 连接到PostgreSQL成功
- ✓ Celery Worker（如果启动）显示 `Ready to accept tasks`

### 3.5 验证后端连接性

**操作步骤**:

```bash
# 在新的终端中测试后端
curl http://localhost:5001/health  # 检查健康状态
curl http://localhost:5001/api/users  # 如果需要，检查API响应

# 检查Redis连接
redis-cli -h localhost -p 6379 ping

# 检查PostgreSQL连接
psql -h localhost -U postgres -d dify -c "SELECT 1;"
```

**检查点**:
- ✓ `/health` 端点返回200状态码或类似响应
- ✓ Redis PING 返回 `PONG`
- ✓ PostgreSQL 查询返回 `1`
- ✓ 后端日志显示请求已被处理

---

## 第四阶段：前端和其他服务（可选）

### 4.1 前端部署（容器化方式）

如果您稍后需要启动前端，可以选择：

**选项A：容器化前端（推荐）**

```bash
cd /Users/codemaggot/github.com/maggot-code/dify/docker

# 编辑.env文件，设置：
# CONSOLE_API_URL=http://localhost:5001
# SERVICE_API_URL=http://localhost:5001

# 启动前端容器
docker compose -f docker-compose.yaml -p dify up -d nginx
```

**选项B：本地前端开发**

```bash
cd /Users/codemaggot/github.com/maggot-code/dify/web

# 安装依赖
pnpm install

# 启动开发服务器
pnpm run dev  # 访问 http://localhost:3000
```

---

## 🔧 故障排除指南

### 问题1：PostgreSQL连接失败

**症状**: `psycopg2.OperationalError: could not connect to server`

**排查步骤**:
```bash
# 1. 检查容器状态
docker ps | grep db_postgres

# 2. 查看容器日志
docker logs dify-db_postgres-1

# 3. 验证环境变量
grep DB_ /Users/codemaggot/github.com/maggot-code/dify/api/.env

# 4. 验证主机和端口
telnet localhost 5432
```

**可能原因**:
- PostgreSQL容器未启动
- 主机/端口配置错误
- 防火墙阻止连接

**解决方案**:
- 重启中间件容器：`docker compose -f docker-compose.middleware.yaml -p dify restart db_postgres`
- 检查`.env`中的`DB_HOST`应为 `localhost` 而不是 `127.0.0.1`

---

### 问题2：Redis连接失败

**症状**: `redis.ConnectionError: Error -2 connecting to localhost:6379`

**排查步骤**:
```bash
# 1. 检查Redis容器
docker logs dify-redis-1

# 2. 测试连接
redis-cli -h localhost -p 6379 ping

# 3. 检查Redis密码
grep REDIS_PASSWORD /Users/codemaggot/github.com/maggot-code/dify/api/.env
```

**可能原因**:
- Redis密码不匹配
- Redis容器未启动
- 防火墙阻止连接

**解决方案**:
- 重启Redis：`docker compose -f docker-compose.middleware.yaml -p dify restart redis`
- 确保`.env`中的`REDIS_PASSWORD`与`middleware.env`一致

---

### 问题3：Celery Worker无法启动

**症状**: `ImportError` 或 `ModuleNotFoundError`

**排查步骤**:
```bash
# 1. 检查虚拟环境
which python
python --version

# 2. 重新安装依赖
uv sync --dev

# 3. 检查app.celery配置
grep -r "celery = " /Users/codemaggot/github.com/maggot-code/dify/api/app*.py
```

**可能原因**:
- Python版本不正确
- 依赖未完全安装
- Celery配置错误

**解决方案**:
- 使用 `pyenv local 3.12.0` 确保正确版本
- 运行 `uv sync --dev --refresh` 重新安装
- 检查 `CELERY_BROKER_URL` 是否正确

---

### 问题4：迁移失败

**症状**: `flask db upgrade` 返回错误

**排查步骤**:
```bash
# 1. 检查迁移状态
uv run flask db current

# 2. 查看可用迁移
uv run flask db history

# 3. 降级到上一个版本（如果需要）
uv run flask db downgrade
```

**可能原因**:
- 数据库已被污染
- 迁移脚本有错误
- 数据库权限不足

**解决方案**:
- 删除旧的数据卷并重新创建：
  ```bash
  docker compose -f docker-compose.middleware.yaml -p dify down -v
  rm -rf /Users/codemaggot/dev/dify/postgresql
  ```
- 重新启动中间件并运行迁移

---

### 问题5：端口被占用

**症状**: `OSError: [Errno 48] Address already in use`

**排查步骤**:
```bash
# 1. 查看占用的端口
lsof -i :5001
lsof -i :6379
lsof -i :5432

# 2. 查看Docker容器端口映射
docker ps --format "table {{.Names}}\t{{.Ports}}"
```

**可能原因**:
- 另一个Flask进程仍在运行
- 容器端口映射冲突

**解决方案**:
- 杀死占用端口的进程：`kill -9 <PID>`
- 更改Flask端口：`uv run flask run --port 5002`
- 重新启动所有Docker容器

---

## 📊 架构图

```
┌─────────────────────────────────────────────────────────────┐
│                     本地开发环境 (macOS)                      │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Flask后端 (源代码启动)                              │    │
│  │  ┌──────────────────────────────────────────┐       │    │
│  │  │ 运行: uv run flask run --port 5001       │       │    │
│  │  │ • HTTP Server: 0.0.0.0:5001              │       │    │
│  │  │ • Debug: 启用                             │       │    │
│  │  │ • 依赖: Python 3.11+ via uv              │       │    │
│  │  └──────────────────────────────────────────┘       │    │
│  │                       ↕️ TCP                         │    │
│  │  ┌──────────────────────────────────────────┐       │    │
│  │  │ Celery Worker (可选)                      │       │    │
│  │  │ 运行: uv run celery -A app.celery worker │       │    │
│  │  │ • 异步任务处理                            │       │    │
│  │  │ • 数据集导入、索引等                      │       │    │
│  │  └──────────────────────────────────────────┘       │    │
│  │                       ↕️ TCP                         │    │
│  │  ┌──────────────────────────────────────────┐       │    │
│  │  │ Celery Beat (可选)                        │       │    │
│  │  │ 运行: uv run celery -A app.celery beat   │       │    │
│  │  │ • 定时任务调度                            │       │    │
│  │  └──────────────────────────────────────────┘       │    │
│  └─────────────────────────────────────────────────────┘    │
│                          ↕️ Network                           │
├─────────────────────────────────────────────────────────────┤
│              OrbStack Docker容器 (虚拟化)                     │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────────────────┐  ┌──────────────────────────┐ │
│  │ PostgreSQL 15 容器        │  │ Redis 6 容器              │ │
│  │ • Port: 5432             │  │ • Port: 6379             │ │
│  │ • 数据库: dify           │  │ • 密码: 受保护            │ │
│  │ • 数据卷: /Users/...dev/ │  │ • 数据卷: /Users/...dev/ │ │
│  │  postgresql              │  │  redis                   │ │
│  └──────────────────────────┘  └──────────────────────────┘ │
│                                                               │
│  ┌──────────────────────────────────────────┐               │
│  │ Weaviate 向量数据库 (可选)                │               │
│  │ • Port: 8080                             │               │
│  │ • HTTP API: /v1/*                        │               │
│  │ • 数据卷: /Users/...dev/weaviate        │               │
│  └──────────────────────────────────────────┘               │
│                                                               │
└─────────────────────────────────────────────────────────────┘
                      Docker Network: dify_default
```

---

## 🚀 启动脚本（快速参考）

创建一个启动脚本 `start-dify-dev.sh` 来简化启动流程：

```bash
#!/bin/bash
set -e

PROJECT_DIR="/Users/codemaggot/github.com/maggot-code/dify"
API_DIR="$PROJECT_DIR/api"
DOCKER_DIR="$PROJECT_DIR/docker"

echo "📦 启动Dify本地开发环境..."

# 第1步：启动中间件
echo "🐳 步骤 1: 启动Docker中间件..."
cd "$DOCKER_DIR"
docker compose --env-file middleware.env \
  -f docker-compose.middleware.yaml \
  -p dify \
  up -d

echo "⏳ 等待中间件就绪..."
sleep 10

# 第2步：启动后端
echo "🚀 步骤 2: 启动后端服务..."
cd "$API_DIR"
uv run flask run --host 0.0.0.0 --port 5001 --debug &
FLASK_PID=$!

echo ""
echo "✅ Dify开发环境已启动！"
echo ""
echo "📊 服务信息:"
echo "  • 后端 API:     http://localhost:5001"
echo "  • PostgreSQL:   localhost:5432"
echo "  • Redis:        localhost:6379"
echo "  • Weaviate:     http://localhost:8080"
echo ""
echo "💡 后续步骤:"
echo "  1. 启动前端:     cd web && pnpm install && pnpm run dev"
echo "  2. 启动Worker:   cd api && uv run celery -A app.celery worker"
echo "  3. 启动Beat:     cd api && uv run celery -A app.celery beat"
echo ""
echo "按 Ctrl+C 停止后端服务"

wait $FLASK_PID
```

使用方式：
```bash
chmod +x start-dify-dev.sh
./start-dify-dev.sh
```

---

## ✅ 验证清单

部署完成后，使用此清单验证所有组件正常运行：

- [ ] Python版本为 3.11 或 3.12
- [ ] uv 已安装且可用
- [ ] OrbStack 已启动
- [ ] PostgreSQL容器运行中，端口5432可访问
- [ ] Redis容器运行中，端口6379可访问
- [ ] Weaviate容器运行中，端口8080可访问
- [ ] 后端 `.env` 文件已创建和配置
- [ ] 数据库迁移完成（`flask db current` 返回版本）
- [ ] Flask开发服务器启动在 `http://localhost:5001`
- [ ] `curl http://localhost:5001/health` 返回成功响应
- [ ] 后端日志中无ERROR或CRITICAL信息
- [ ] Redis连接正常（`redis-cli ping` 返回PONG）
- [ ] PostgreSQL连接正常（`psql` 命令成功）

---

## 📝 重要提示和最佳实践

### 1. 数据持久化
- 所有数据卷都映射到 `/Users/codemaggot/dev/dify/` 目录
- 这确保了即使容器重启，数据也不会丢失
- 定期备份此目录以防数据损失

### 2. 环境变量安全
- 不要将 `.env` 文件提交到版本控制系统
- 使用强密码（建议20个字符以上）
- 对于生产环境，使用密钥管理服务

### 3. 开发效率
- 在单独的终端窗口中运行Flask、Worker和Beat
- 使用 `FLASK_DEBUG=true` 启用自动重载
- 在修改代码后，Flask会自动重启

### 4. 调试技巧
- 设置 `LOG_LEVEL=DEBUG` 以获取详细日志
- 使用 `uv run flask shell` 进入交互式Python shell
- 使用 `redis-cli` 检查缓存状态

### 5. 清理和重置
```bash
# 完全清理（删除所有数据）
cd /Users/codemaggot/github.com/maggot-code/dify/docker
docker compose -f docker-compose.middleware.yaml -p dify down -v
rm -rf /Users/codemaggot/dev/dify

# 仅重启容器（保留数据）
docker compose -f docker-compose.middleware.yaml -p dify restart

# 查看日志
docker compose -f docker-compose.middleware.yaml -p dify logs -f <service>
```

---

## 🆘 获取帮助

遇到问题时：

1. **查看官方文档**: https://docs.dify.ai/getting-started/install-self-hosted/local-source-code
2. **检查日志**:
   - 后端: 在Flask启动终端中查看
   - 容器: `docker logs dify-<service>-1`
   - 应用: `/Users/codemaggot/github.com/maggot-code/dify/api/logs/`

3. **社区支持**:
   - GitHub Issues: https://github.com/langgenius/dify/issues
   - Discord: https://discord.gg/FngNHpbcY7
   - 官方FAQ: https://docs.dify.ai/getting-started/install-self-hosted/faqs

---

## 📌 快速命令参考

```bash
# 查看后端服务状态
curl -s http://localhost:5001/health | jq

# 查看所有Docker容器
docker ps -a

# 查看容器日志（实时跟踪）
docker logs -f dify-db_postgres-1

# 进入容器shell
docker exec -it dify-db_postgres-1 /bin/bash

# 停止所有中间件
cd /Users/codemaggot/github.com/maggot-code/dify/docker
docker compose -f docker-compose.middleware.yaml -p dify stop

# 启动所有中间件
docker compose --env-file middleware.env -f docker-compose.middleware.yaml -p dify up -d

# 重启后端（终止进程并重新启动）
pkill -f "flask run" && cd /Users/codemaggot/github.com/maggot-code/dify/api && uv run flask run --port 5001

# 查看端口占用
lsof -i :5001
```

---

**计划版本**: v1.0
**最后更新**: 2026年1月
**维护者**: 开发团队
