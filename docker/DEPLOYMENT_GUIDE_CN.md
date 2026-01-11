# Dify 私有化部署快速指南

> 本指南基于真实部署经验编写，适合快速启动和运行 Dify 私有化部署。

## 📋 概述

这是一份针对 Dify v1.11.2 的 Docker Compose 私有化部署指南，包含必要的配置步骤和常见问题排查。按照本指南配置，可以在 10-15 分钟内快速启动完整的 Dify 系统。

---

## 🚀 快速启动 (5 分钟)

### 前置条件
- Docker & Docker Compose 已安装
- 足够的磁盘空间 (~10GB)
- 宿主机开放端口：80、443、5001

### 3 个步骤启动

#### 第 1 步：准备环境文件

```bash
cd docker/
cp .env.example .env
```

#### 第 2 步：修改关键配置

编辑 `.env` 文件，替换以下变量（用你的实际域名或 IP）：

**localhost 本地部署：**
```bash
CONSOLE_API_URL=http://localhost:5001
CONSOLE_WEB_URL=http://localhost
APP_API_URL=http://localhost:5001
FILES_URL=http://localhost:5001
INTERNAL_FILES_URL=http://api:5001
SECRET_KEY=生成一个随机密钥（见下文）
COOKIE_DOMAIN=localhost
```

**远程服务器部署（示例：example.com）：**
```bash
CONSOLE_API_URL=https://api.example.com
CONSOLE_WEB_URL=https://example.com
APP_API_URL=https://api.example.com
FILES_URL=https://api.example.com
INTERNAL_FILES_URL=http://api:5001  # 保持不变（容器内部通信）
SECRET_KEY=生成一个随机密钥（见下文）
COOKIE_DOMAIN=example.com
```

#### 第 3 步：启动服务

```bash
docker compose up -d
```

等待 30-60 秒，所有容器启动完成。验证：

```bash
docker compose ps
# 应显示 11 个 "Up" 状态的容器
```

访问：`http://localhost` 或 `https://example.com`

---

## 🔑 生成 SECRET_KEY

替换你的 SECRET_KEY 用于加密会话和敏感数据：

```bash
openssl rand -base64 42
```

复制生成的字符串，粘贴到 `.env` 中的 `SECRET_KEY=` 后面。

示例（勿直接使用）：
```bash
SECRET_KEY=/Iu5VKtjOwqqBqjAxCtzIGMdZI69Nejnq8xSvEwzMd7RAZZGB1VAb365
```

---

## ⚙️ 关键配置详解

### 必填配置项

| 变量 | 说明 | 示例 | 何时修改 |
|------|------|------|--------|
| **CONSOLE_API_URL** | 前端访问 API 的地址（必须是宿主机可达）| `http://localhost:5001` | 域名变更 |
| **CONSOLE_WEB_URL** | 前端网站地址 | `http://localhost` | 域名变更 |
| **APP_API_URL** | 应用 API 地址 | `http://localhost:5001` | 域名变更 |
| **SECRET_KEY** | 会话加密密钥 | `openssl rand -base64 42` | 首次部署 |
| **FILES_URL** | 文件下载/预览 URL | `http://localhost:5001` | 启用文件处理时 |
| **INTERNAL_FILES_URL** | 容器内部文件 URL | `http://api:5001` | **不要修改** |
| **COOKIE_DOMAIN** | Cookie 作用域 | `localhost` | 支持跨子域时 |

### 常见场景配置

#### 场景 1：本地开发 (localhost)
```bash
CONSOLE_API_URL=http://localhost:5001
CONSOLE_WEB_URL=http://localhost
APP_API_URL=http://localhost:5001
FILES_URL=http://localhost:5001
COOKIE_DOMAIN=localhost
```

#### 场景 2：单一域名 (example.com)
```bash
CONSOLE_API_URL=https://api.example.com
CONSOLE_WEB_URL=https://example.com
APP_API_URL=https://api.example.com
FILES_URL=https://api.example.com
COOKIE_DOMAIN=example.com
```

需要配置 Nginx 反向代理（见下文）。

#### 场景 3：子域名分离 (console.example.com 与 api.example.com)
```bash
CONSOLE_API_URL=https://api.example.com
CONSOLE_WEB_URL=https://console.example.com
APP_API_URL=https://api.example.com
FILES_URL=https://api.example.com
COOKIE_DOMAIN=.example.com  # 注意前缀点
NEXT_PUBLIC_COOKIE_DOMAIN=1
```

---

## 🐳 Docker Compose 关键修改

### API 端口映射（必需）

确保 `docker-compose.yaml` 中的 `api` 服务包含以下配置：

```yaml
api:
  image: langgenius/dify-api:1.11.2
  restart: always
  ports:
    - "0.0.0.0:5001:5001"  # ← 关键：暴露 API 端口到宿主机
  environment:
    <<: *shared-api-worker-env
    MODE: api
    ...
```

**为什么需要？** 
- 前端（浏览器）需要直接访问 API 服务
- 如果不映射此端口，前端会出现 `Connection refused` 错误

### 检查映射是否生效

```bash
docker compose ps | grep api
# 应显示：docker-api-1  0.0.0.0:5001->5001/tcp
```

---

## 🌐 Nginx 反向代理配置（远程部署必需）

如果使用域名而非 localhost，需要配置 Nginx 反向代理。

修改 `docker/nginx/conf.d/default.conf`：

```nginx
# API 反向代理
server {
    listen 80;
    server_name api.example.com;
    
    location / {
        proxy_pass http://api:5001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

# 前端
server {
    listen 80;
    server_name example.com www.example.com;
    
    location / {
        proxy_pass http://web:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

然后重启 Nginx：

```bash
docker compose restart nginx
```

---

## ✅ 部署验证清单

部署完成后，逐项验证：

- [ ] **容器状态**
  ```bash
  docker compose ps
  # 所有容器应显示 "Up"
  ```

- [ ] **API 可达性**
  ```bash
  curl http://localhost:5001/console/api/workspaces
  # 应返回 401（认证错误为正常，说明 API 已响应）
  ```

- [ ] **前端加载**
  ```bash
  # 浏览器访问
  http://localhost/install  # 或你的域名
  # 页面应正常加载（不是白屏或连接拒绝）
  ```

- [ ] **安装向导**
  - 访问 `/install`
  - 创建管理员账户
  - 完成初始化设置

- [ ] **登录**
  - 用创建的账户登录
  - 验证能否访问应用列表 `/apps`

---

## 🔧 常见问题排查

### 问题 1：前端无法加载（白屏或加载中）

**原因：** API 端口未映射或 API 不可达

**排查步骤：**
```bash
# 1. 检查 API 容器是否运行
docker compose ps | grep api
# 应显示 "Up"

# 2. 检查端口映射
docker compose ps | grep api | grep 5001
# 应显示 "0.0.0.0:5001->5001/tcp"

# 3. 测试 API 连接
curl -i http://localhost:5001/console/api/workspaces
# 应返回 HTTP 401（而不是 "Connection refused"）

# 4. 如果端口映射缺失，修改 docker-compose.yaml：
# 在 api: 服务下添加：
# ports:
#   - "0.0.0.0:5001:5001"
# 然后重启：
docker compose down && docker compose up -d
```

### 问题 2：登录后 401 认证错误

**原因：** SECRET_KEY 不一致或 Cookie 配置错误

**解决：**
```bash
# 1. 检查 SECRET_KEY 是否已设置
grep "SECRET_KEY=" .env

# 2. 如果为空，生成新的：
SECRET_KEY=$(openssl rand -base64 42)
sed -i "s/^SECRET_KEY=.*/SECRET_KEY=$SECRET_KEY/" .env

# 3. 重启服务
docker compose down && docker compose up -d

# 4. 清空浏览器 Cookie 并重新登录
```

### 问题 3：CORS 错误

**原因：** 前端和后端域名不匹配

**排查：**
```bash
# 检查 CORS 配置
grep "WEB_API_CORS_ALLOW_ORIGINS\|CONSOLE_CORS_ALLOW_ORIGINS" .env

# 应包含你的前端域名，例如：
WEB_API_CORS_ALLOW_ORIGINS=http://localhost
CONSOLE_CORS_ALLOW_ORIGINS=http://localhost
```

### 问题 4：容器反复重启

**原因：** 数据库连接失败或初始化问题

**排查：**
```bash
# 查看容器日志
docker compose logs api --tail=50

# 检查数据库是否就绪
docker compose logs db_postgres | grep "ready to accept"

# 如果数据库未就绪，等待并重试
sleep 30 && docker compose logs db_postgres
```

### 问题 5：文件上传失败

**原因：** FILES_URL 配置不正确

**检查：**
```bash
# 确保 FILES_URL 与 CONSOLE_API_URL 域名相同
grep "FILES_URL\|CONSOLE_API_URL" .env

# 例如：
CONSOLE_API_URL=http://localhost:5001
FILES_URL=http://localhost:5001  # 应相同
```

---

## 🛠️ 常用操作命令

### 查看日志

```bash
# 查看所有服务日志
docker compose logs -f

# 查看特定服务日志（如 API）
docker compose logs -f api

# 查看最后 100 行
docker compose logs api --tail=100
```

### 重启服务

```bash
# 重启所有服务
docker compose restart

# 重启特定服务（如 API）
docker compose restart api

# 完全重建（推荐在配置大改动后）
docker compose down && docker compose up -d
```

### 清理数据

```bash
# 停止并删除所有容器（数据保留在 volumes/）
docker compose down

# 完全清理（包括数据，谨慎使用）
docker compose down -v
```

### 更新配置

```bash
# 修改 .env 后，重启使配置生效
docker compose down && docker compose up -d

# 验证新配置已加载
docker compose exec api env | grep CONSOLE_API_URL
```

---

## 📊 系统架构概览

```
浏览器 (localhost:80/443)
   ↓
Nginx 反向代理
   ├→ http://web:3000 (前端 Next.js)
   └→ http://api:5001 (后端 API)
   
API 连接链路：
浏览器 → Nginx (localhost) → API 容器 (5001)
         ↓ 映射
      宿主机:5001 → 容器:5001
```

---

## 🎯 场景化部署示例

### 示例 1：本地开发部署

```bash
# 1. 创建 .env
cp .env.example .env

# 2. 修改关键变量（保持默认 localhost）
CONSOLE_API_URL=http://localhost:5001
CONSOLE_WEB_URL=http://localhost
SECRET_KEY=$(openssl rand -base64 42)

# 3. 启动
docker compose up -d

# 4. 访问
# http://localhost/install
```

### 示例 2：阿里云 ECS 远程部署

```bash
# 1. 创建 .env
cp .env.example .env

# 2. 修改配置（替换 your-domain.com）
CONSOLE_API_URL=https://api.your-domain.com
CONSOLE_WEB_URL=https://your-domain.com
APP_API_URL=https://api.your-domain.com
FILES_URL=https://api.your-domain.com
SECRET_KEY=$(openssl rand -base64 42)
COOKIE_DOMAIN=your-domain.com

# 3. 配置 Nginx（修改 docker/nginx/conf.d/default.conf）
# 指向你的域名

# 4. 启动
docker compose up -d

# 5. 配置 SSL（可选，推荐使用 Let's Encrypt）
# 修改 certbot 配置或手动上传证书

# 6. 访问
# https://your-domain.com/install
```

---

## 📝 配置文件清单

| 文件 | 用途 | 修改频率 |
|------|------|--------|
| `.env` | 主环境配置 | 初始部署 + 域名变更 |
| `docker-compose.yaml` | 容器编排 | 仅需确保 API 端口映射存在 |
| `nginx/conf.d/default.conf` | Nginx 反向代理 | 使用域名部署时 |
| `certbot/` | SSL 证书管理 | 需要 HTTPS 时 |

---

## ✨ 最佳实践

### 1. 安全建议
- ✅ 生成强 SECRET_KEY（使用 `openssl rand -base64 42`）
- ✅ 定期备份 `volumes/` 目录（数据库和文件）
- ✅ 限制宿主机防火墙，仅允许需要的端口
- ✅ 使用 HTTPS 生产环境（启用 Certbot）

### 2. 性能优化
- 分配足够内存（推荐 4GB+）
- 使用 SSD 存储（PostgreSQL 性能关键）
- 定期清理容器日志：`docker system prune`

### 3. 监控与维护
- 监听容器状态：`docker compose ps`
- 定期查看日志：`docker compose logs -f`
- 建立备份策略：`docker compose exec db_postgres pg_dump ...`

---

## 📞 获取帮助

遇到问题？按以下步骤排查：

1. **查看日志**
   ```bash
   docker compose logs api --tail=100
   docker compose logs web --tail=100
   docker compose logs nginx --tail=100
   ```

2. **测试连接**
   ```bash
   curl http://localhost:5001/console/api/workspaces
   ```

3. **检查配置**
   ```bash
   grep -E "CONSOLE_API_URL|APP_API_URL|SECRET_KEY" .env
   ```

4. **重新启动**
   ```bash
   docker compose down && docker compose up -d
   ```

---

## 📄 版本信息

- **Dify 版本**：v1.11.2
- **Docker 版本**：推荐 20.10+
- **Docker Compose 版本**：推荐 2.0+

---

**祝你部署顺利！🚀**
