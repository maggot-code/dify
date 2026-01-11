# Dify 私有化部署 - 问题解决总结

## 🎯 核心问题总结

本文档记录了 Dify Docker Compose 部署过程中遇到的问题、根本原因和解决方案。

---

## 📌 问题描述

**现象**：部署 Dify v1.11.2 后，前端无法连接到后端 API，出现以下错误：
- 浏览器报告 `Connection refused` 或 `Network error`
- 前端页面无法加载数据
- 尝试访问 `/install` 页面时白屏或加载中

**根本原因**（在这个部署中）：
1. API 服务在 Docker 容器内运行，监听 `0.0.0.0:5001`
2. 但 `docker-compose.yaml` 中的 API 服务**缺少端口映射** (`ports` 段)
3. 导致容器内的 5001 端口未暴露到宿主机
4. 浏览器（运行在宿主机）无法访问容器内的 API

---

## 🔧 关键解决方案

### 解决方案 1：添加 API 端口映射（推荐）✅

**修改文件**：`docker-compose.yaml`

**修改位置**：API 服务定义（约第 705-710 行）

**修改前**：
```yaml
api:
  image: langgenius/dify-api:1.11.2
  restart: always
  environment:
    # ...
```

**修改后**：
```yaml
api:
  image: langgenius/dify-api:1.11.2
  restart: always
  ports:
    - "0.0.0.0:5001:5001"  # ← 添加这两行
  environment:
    # ...
```

**重启生效**：
```bash
docker compose down
docker compose up -d
```

**验证**：
```bash
docker compose ps | grep api
# 应显示：0.0.0.0:5001->5001/tcp
```

---

## 🌐 关键环境变量配置

### 最小化配置（仅需修改 5 个）

| 变量名 | 说明 | 本地值 | 远程值 |
|--------|------|--------|--------|
| **CONSOLE_API_URL** | 前端调用 API 的地址<br/>（必须是宿主机可达） | `http://localhost:5001` | `https://api.example.com` |
| **CONSOLE_WEB_URL** | 前端网站 URL<br/>（用于 CORS 和重定向） | `http://localhost` | `https://example.com` |
| **APP_API_URL** | 应用 API 地址<br/>（与 CONSOLE_API_URL 相同） | `http://localhost:5001` | `https://api.example.com` |
| **FILES_URL** | 文件下载 URL<br/>（与 CONSOLE_API_URL 相同） | `http://localhost:5001` | `https://api.example.com` |
| **SECRET_KEY** | 会话加密密钥<br/>（必须生成新值） | `openssl rand -base64 42` | `openssl rand -base64 42` |

### 内部通信配置（保持不变）

```bash
INTERNAL_FILES_URL=http://api:5001  # 容器内部通信，不要改
DB_HOST=db_postgres                 # 容器名称，不要改
REDIS_HOST=redis                    # 容器名称，不要改
```

### 跨域配置（与 CONSOLE_WEB_URL 匹配）

```bash
WEB_API_CORS_ALLOW_ORIGINS=http://localhost         # 或你的域名
CONSOLE_CORS_ALLOW_ORIGINS=http://localhost         # 或你的域名
COOKIE_DOMAIN=localhost                             # 或你的域名
```

---

## 🔑 生成 SECRET_KEY

**重要**：不能使用默认值或示例值，必须生成新的。

```bash
# 执行此命令
openssl rand -base64 42

# 输出示例（每次不同）
/Iu5VKtjOwqqBqjAxCtzIGMdZI69Nejnq8xSvEwzMd7RAZZGB1VAb365

# 复制输出，粘贴到 .env
SECRET_KEY=/Iu5VKtjOwqqBqjAxCtzIGMdZI69Nejnq8xSvEwzMd7RAZZGB1VAb365
```

---

## 🚀 完整部署步骤

### 第 1 步：初始化配置（5 分钟）

```bash
cd docker/

# 从示例创建 .env
cp .env.example .env

# 根据部署环境修改 5 个关键配置
# （使用 QUICK_CONFIG.md 中的模板）
nano .env
```

### 第 2 步：启动服务（2 分钟）

```bash
# 启动所有容器
docker compose up -d

# 等待 30-60 秒
sleep 30

# 验证容器状态
docker compose ps

# 应输出：
# docker-api-1                  Up  0.0.0.0:5001->5001/tcp
# docker-nginx-1                Up  0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp
# docker-web-1                  Up
# docker-db_postgres-1          Up  (healthy)
# docker-redis-1                Up  (healthy)
# ... 其他容器
```

### 第 3 步：验证连接（3 分钟）

```bash
# 3.1 测试 API 直接连接
curl -i http://localhost:5001/console/api/workspaces

# 应返回：HTTP/1.1 401 UNAUTHORIZED
# （401 是正常的，说明 API 正在运行并要求认证）

# 3.2 访问前端
# 打开浏览器访问：http://localhost/install
# 或：https://example.com/install

# 3.3 检查前端 HTML 配置
curl http://localhost/install | grep "data-api-prefix"

# 应包含：data-api-prefix="http://localhost:5001/console/api"
```

### 第 4 步：完成初始化（5 分钟）

1. 打开 `http://localhost/install` 或你的部署 URL
2. 创建管理员账户（设置邮箱和密码）
3. 完成初始化设置
4. 登录后访问 `http://localhost/apps`

---

## 🧪 常见问题快速检查表

### 问题：前端白屏或"正在加载中"

**原因排查顺序**：

1. **API 容器运行状态**
   ```bash
   docker compose ps | grep api
   # 应显示 "Up"
   ```

2. **API 端口映射**
   ```bash
   docker compose ps | grep api | grep 5001
   # 应显示 "0.0.0.0:5001->5001/tcp"
   ```

3. **API 可达性**
   ```bash
   curl -i http://localhost:5001/console/api/workspaces
   # 应返回 401（而不是 Connection refused）
   ```

4. **前端配置**
   ```bash
   curl http://localhost/install | grep "data-api-prefix"
   # 应包含正确的 API URL
   ```

**修复步骤**：
- 如果 API 未运行：`docker compose restart api`
- 如果端口未映射：修改 `docker-compose.yaml`，添加 `ports: ["0.0.0.0:5001:5001"]`
- 如果 API 无响应：查看日志 `docker compose logs api --tail=50`

---

### 问题：401 认证错误

**可能原因**：
- SECRET_KEY 为空或使用默认值
- Cookie 配置不正确
- 浏览器 Cookie 已过期

**修复步骤**：
```bash
# 1. 检查 SECRET_KEY
grep "^SECRET_KEY=" .env

# 2. 如果为空或默认值，生成新的
SECRET_KEY=$(openssl rand -base64 42)
sed -i "s/^SECRET_KEY=.*/SECRET_KEY=$SECRET_KEY/" .env

# 3. 重启服务
docker compose down && docker compose up -d

# 4. 清空浏览器 Cookie 和本地存储
# 开发者工具 → Application → 删除 localhost 的 Cookie
```

---

### 问题：CORS 错误

**原因**：前端和后端域名配置不一致

**检查和修复**：
```bash
# 查看当前配置
grep -E "CONSOLE_API_URL|CONSOLE_WEB_URL|WEB_API_CORS" .env

# 修改示例（需要一致）：
CONSOLE_API_URL=https://api.example.com
CONSOLE_WEB_URL=https://example.com
WEB_API_CORS_ALLOW_ORIGINS=https://example.com,https://api.example.com

# 重启生效
docker compose restart
```

---

## 📊 配置对应关系图

```
┌─────────────────────────────────────────────────────────────┐
│  浏览器 (用户访问的 URL)                                     │
│  ├─ http://localhost     → CONSOLE_WEB_URL                  │
│  └─ http://localhost:5001 → CONSOLE_API_URL                 │
└────────────┬────────────────────────────────────────────────┘
             │
             ↓
┌─────────────────────────────────────────────────────────────┐
│  Nginx 反向代理 (宿主机)                                     │
│  ├─ 监听 80:443                                             │
│  └─ 转发到容器内的 web:3000 和 api:5001                     │
└────────────┬────────────────────────────────────────────────┘
             │
             ↓
┌─────────────────────────────────────────────────────────────┐
│  Docker 容器网络                                             │
│  ├─ web (Next.js 3000)    → CONSOLE_WEB_URL                 │
│  ├─ api (Flask 5001)      → CONSOLE_API_URL                 │
│  ├─ db_postgres (5432)    → DB_HOST                         │
│  └─ redis (6379)          → REDIS_HOST                      │
└─────────────────────────────────────────────────────────────┘

关键：CONSOLE_API_URL 必须指向 API 容器的 5001 端口
      该端口必须通过 docker-compose.yaml 的 ports 段暴露
```

---

## 💾 部署文件清单

本部署包含以下关键文件：

| 文件 | 位置 | 用途 | 新用户需要修改 |
|------|------|------|-------------|
| `.env.template` | `docker/` | 配置模板 | ✅ 复制并修改 |
| `QUICK_CONFIG.md` | `docker/` | 快速配置指南 | ✅ 参考此文件 |
| `DEPLOYMENT_GUIDE_CN.md` | `docker/` | 完整部署指南 | ℹ️ 遇到问题时查看 |
| `docker-compose.yaml` | `docker/` | 容器编排文件 | ⚠️ 检查 API 端口映射 |
| `.env` | `docker/` | 实际配置（自动生成） | ✅ 修改此文件 |

---

## ✅ 部署成功标志

当以下条件都满足时，说明部署成功：

1. ✅ `docker compose ps` 显示所有容器为 "Up"
2. ✅ `curl http://localhost:5001/console/api/workspaces` 返回 401
3. ✅ 访问 `http://localhost/install` 页面加载成功（不是白屏）
4. ✅ 能够创建管理员账户
5. ✅ 能够登录并访问 `/apps` 页面
6. ✅ 能够创建和运行应用

---

## 🔗 相关文档

- **快速配置**：[QUICK_CONFIG.md](./QUICK_CONFIG.md)
- **完整指南**：[DEPLOYMENT_GUIDE_CN.md](./DEPLOYMENT_GUIDE_CN.md)
- **配置模板**：[.env.template](./.env.template)

---

## 🎓 关键学习点

### 为什么需要端口映射？

Docker 容器运行时，内部服务监听的端口（如 API 的 5001）默认是**隔离**的，仅在容器内部和其他容器间可访问。

要让宿主机（或外部客户端）访问容器内的服务，必须通过 `docker-compose.yaml` 的 `ports` 段进行映射：

```yaml
ports:
  - "0.0.0.0:5001:5001"
  #  ↑        ↑      ↑
  #  宿主机   宿主机  容器
  #  接口    端口    端口
```

这样，浏览器访问 `http://localhost:5001` 时，实际上是访问 Docker 容器内的 API。

### 为什么 INTERNAL_FILES_URL 不需要映射？

`INTERNAL_FILES_URL=http://api:5001` 用于**容器间通信**。Docker 提供的 DNS 解析允许容器通过服务名（如 `api`）相互访问，无需额外的端口映射。

这是容器网络的优势——内部通信完全独立于宿主机。

---

## 📞 支持与反馈

遇到问题时的排查顺序：

1. 查看日志：`docker compose logs -f api`
2. 检查配置：`grep -E "CONSOLE_API_URL|SECRET_KEY" .env`
3. 重启服务：`docker compose restart`
4. 完全重建：`docker compose down && docker compose up -d`

如问题持续未解决，收集以下信息：
- `docker compose ps` 的输出
- `docker compose logs api --tail=50` 的输出
- `.env` 中的关键配置（隐藏敏感信息）

---

**祝你部署顺利！🚀**

*最后更新：2025-01-11*
