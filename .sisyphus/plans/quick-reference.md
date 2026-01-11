# Dify部署快速参考卡

## 🎯 一句话总结
在本地用源代码启动Flask后端，通过OrbStack容器化运行PostgreSQL、Redis、Weaviate中间件。

---

## ⚡ 快速启动（5分钟版）

### 前提条件检查
```bash
# ✅ 全部返回有效结果才能继续
python --version          # 应为 3.11 或 3.12
uv --version              # 应为最新版本
docker --version          # 应为最新版本
ls /Users/codemaggot/dev  # 目录应存在
```

### 中间件启动（第一次运行）
```bash
cd /Users/codemaggot/github.com/maggot-code/dify/docker

# 复制和编辑配置
cp middleware.env.example middleware.env
# 编辑middleware.env: 修改密码，设置卷目录为 /Users/codemaggot/dev/dify/*

# 创建数据目录
mkdir -p /Users/codemaggot/dev/dify/{postgresql,redis,weaviate}

# 启动容器
docker compose --env-file middleware.env -f docker-compose.middleware.yaml -p dify up -d

# 验证启动
docker compose -f docker-compose.middleware.yaml -p dify ps
```

### 后端启动（第一次运行）
```bash
cd /Users/codemaggot/github.com/maggot-code/dify/api

# 配置环境
cp .env.example .env
# 编辑.env: 生成SECRET_KEY，同步middleware.env中的密码

# 安装依赖
uv sync --dev

# 初始化数据库
uv run flask db upgrade

# 启动服务
uv run flask run --host 0.0.0.0 --port 5001 --debug
```

**启动完成！** 打开 http://localhost:5001

---

## 📋 环境变量映射表

**middleware.env** ↔️ **api/.env**

| 中间件配置 | 后端配置 | 说明 |
|-----------|---------|------|
| `DB_PASSWORD` | `DB_PASSWORD` | 💥 必须相同 |
| `DB_USERNAME` | `DB_USERNAME` | 💥 必须相同 |
| `DB_DATABASE` | `DB_DATABASE` | 💥 必须相同 |
| `REDIS_PASSWORD` | `REDIS_PASSWORD` | 💥 必须相同 |
| `PGDATA_HOST_VOLUME` | - | 数据库数据卷位置 |
| `REDIS_HOST_VOLUME` | - | Redis数据卷位置 |
| - | `DB_HOST=localhost` | 本地连接 |
| - | `REDIS_HOST=localhost` | 本地连接 |

---

## 🔄 日常操作

### 启动已有环境
```bash
# 终端1: 启动中间件
cd docker && docker compose --env-file middleware.env -f docker-compose.middleware.yaml -p dify up -d

# 终端2: 启动后端
cd api && uv run flask run --port 5001 --debug

# 终端3: 启动Worker（可选，用于异步任务）
cd api && uv run celery -A app.celery worker -P threads -c 2 --loglevel INFO -Q dataset,priority_dataset,priority_pipeline,pipeline,mail,ops_trace,app_deletion,plugin,workflow_storage,conversation,workflow,schedule_poller,schedule_executor,triggered_workflow_dispatcher,trigger_refresh_executor,retention
```

### 停止所有服务
```bash
# 停止Flask（Ctrl+C 在对应终端）
# 停止Celery（Ctrl+C 在对应终端）
# 停止中间件
cd docker && docker compose -f docker-compose.middleware.yaml -p dify stop
```

### 查看状态
```bash
# 中间件状态
docker compose -f docker-compose.middleware.yaml -p dify ps

# 验证连接
redis-cli ping                         # 应返回PONG
psql -h localhost -U postgres -d dify  # 应连接成功
curl http://localhost:5001/health      # 应返回200或类似

# 查看日志
docker logs -f dify-db_postgres-1
docker logs -f dify-redis-1
```

---

## 🆘 常见问题速查

| 问题 | 症状 | 快速解决 |
|------|------|---------|
| **连接数据库失败** | `psycopg2.OperationalError` | `docker compose -f docker-compose.middleware.yaml -p dify restart db_postgres` |
| **连接Redis失败** | `redis.ConnectionError` | 检查密码匹配，重启: `docker compose restart redis` |
| **端口被占用** | `Address already in use` | `lsof -i :5001` 找到PID，`kill -9 <PID>` |
| **数据库表不存在** | `Table does not exist` | `cd api && uv run flask db upgrade` |
| **导入包失败** | `ModuleNotFoundError` | `uv sync --dev --refresh` |
| **Worker启动失败** | `ImportError` | 检查Python版本: `python --version` |

---

## 📁 关键文件位置

```
/Users/codemaggot/github.com/maggot-code/dify/
├── docker/
│   ├── .env                    ← 容器配置（自动生成，不提交）
│   ├── .env.example            ← 容器配置示例
│   ├── middleware.env          ← 中间件配置（自动生成，不提交）
│   ├── middleware.env.example  ← 中间件配置示例
│   ├── docker-compose.middleware.yaml  ← 中间件编排文件
│   └── docker-compose.yaml     ← 完整编排文件（如需前端）
│
├── api/
│   ├── .env                    ← 后端配置（自动生成，不提交）
│   ├── .env.example            ← 后端配置示例
│   ├── pyproject.toml          ← Python依赖定义
│   ├── app.py                  ← Flask主应用
│   ├── migrations/             ← 数据库迁移脚本
│   └── requirements.txt         ← 生成的依赖文件
│
├── web/
│   ├── .env.example            ← 前端配置示例
│   ├── package.json            ← Node.js依赖定义
│   └── ...
│
└── .sisyphus/
    └── plans/
        ├── dify-hybrid-deployment-plan.md ← 完整部署计划（本文件）
        └── quick-reference.md              ← 快速参考（本文件）

/Users/codemaggot/dev/dify/        ← 数据卷目录（由容器创建和使用）
├── postgresql/                     ← PostgreSQL数据文件
├── redis/                          ← Redis持久化文件
└── weaviate/                       ← Weaviate数据文件
```

---

## 🔑 重要密码和凭证

生成强密码的方式：
```bash
# 生成强随机密码（32个字符）
openssl rand -base64 32

# 生成可读强密码（混合字符）
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
```

**安全提示**:
- 不要在版本控制中提交 `.env` 文件
- 不要在日志中打印密码
- 定期更换密码
- 为不同环境使用不同密码

---

## 📊 服务端口映射

| 服务 | 端口 | 用途 | 检查命令 |
|------|------|------|---------|
| Flask API | 5001 | 后端REST API | `curl http://localhost:5001/health` |
| PostgreSQL | 5432 | 主数据库 | `psql -h localhost -U postgres` |
| Redis | 6379 | 缓存/消息队列 | `redis-cli ping` |
| Weaviate | 8080 | 向量数据库 | `curl http://localhost:8080/health` |

---

## 🚨 重置/清理命令

```bash
# ⚠️ 危险操作 - 删除所有数据！

# 仅重启容器（保留数据）
cd docker && docker compose -f docker-compose.middleware.yaml -p dify restart

# 停止并删除容器（保留数据卷）
cd docker && docker compose -f docker-compose.middleware.yaml -p dify down

# 完全删除（包括所有数据！）
cd docker && docker compose -f docker-compose.middleware.yaml -p dify down -v
rm -rf /Users/codemaggot/dev/dify/*

# 重置后端（删除迁移历史）
cd api && rm -rf migrations/__pycache__ && uv run flask db stamp head
```

---

## 💡 开发技巧

### 热重载
Flask 在 `--debug` 模式下会自动重载代码：
```bash
uv run flask run --debug
```

### 交互式Shell
进入Python REPL环境，可以直接执行代码：
```bash
cd api && uv run flask shell
```

### 查看数据库
```bash
# 通过psql连接
psql -h localhost -U postgres -d dify

# 列出所有表
\dt

# 查看表结构
\d table_name
```

### 检查Redis缓存
```bash
redis-cli -h localhost -p 6379
# 列出所有键
KEYS *

# 查看某个键的值和类型
TYPE key_name
GET key_name
```

---

## 📞 获取帮助

遇到问题的优先级排序：

1. **官方文档** → https://docs.dify.ai
2. **本计划** → dify-hybrid-deployment-plan.md (完整版) 
3. **日志检查** → 查看Flask/Docker日志找错误信息
4. **FAQ** → https://docs.dify.ai/getting-started/install-self-hosted/faqs
5. **社区** → GitHub Issues / Discord

---

**版本**: v1.0  
**最后更新**: 2026年1月  
**针对环境**: macOS + OrbStack + pyenv + uv
