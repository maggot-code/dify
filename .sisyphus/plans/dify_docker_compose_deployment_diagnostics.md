# Dify Docker Compose 私有化部署 - 环境变量配置诊断与修复指南

**文档版本**: v1.0  
**创建日期**: 2025-01-11  
**场景**: Docker Compose 私有化部署中遇到 401 认证错误

---

## 📌 问题描述

在使用 `docker compose up -d` 启动 Dify 多个服务后：
- ✅ 可以正常访问平台界面
- ❌ 登录时遇到 **401 认证错误**
- 💥 不同服务（API、Web、中间件）间通信存在问题

**根本原因**: 环境变量配置不完整或不一致，导致跨服务通信和认证失败。

---

## 🔍 问题根源分析

### 1. 跨服务通信地址配置错误
- **症状**: Web 前端无法正确连接到 API 后端
- **原因**: `CONSOLE_API_URL`、`APP_API_URL` 为空或配置错误
- **影响**: API 请求失败，认证无法进行
- **解决方案**: 根据部署环境设置正确的 API 地址

### 2. CORS 跨域资源共享配置缺失
- **症状**: 浏览器 CORS 预检请求被拒绝
- **原因**: `WEB_API_CORS_ALLOW_ORIGINS` 和 `CONSOLE_CORS_ALLOW_ORIGINS` 未配置或为通配符
- **影响**: 浏览器端请求被阻止，无法发送认证信息
- **解决方案**: 明确配置允许的请求来源

### 3. Cookie 和会话配置不一致
- **症状**: 登录成功后立即退出，或浏览器无法保存 Cookie
- **原因**: `COOKIE_DOMAIN` 和 `NEXT_PUBLIC_COOKIE_DOMAIN` 不一致或为空
- **影响**: 会话 Cookie 无法正确保存和验证
- **解决方案**: 统一设置 Cookie 域名

### 4. SECRET_KEY 使用默认值
- **症状**: 多实例部署时认证失败，Token 验证不通过
- **原因**: 未生成个人 SECRET_KEY，使用框架默认值
- **影响**: JWT Token 签名验证失败，会话加密不安全
- **解决方案**: 使用 `openssl rand -base64 42` 生成强密钥

### 5. Web 端 API 前缀配置缺失
- **症状**: Web 前端控制台和公开 API 地址不明确
- **原因**: `NEXT_PUBLIC_API_PREFIX` 和 `NEXT_PUBLIC_PUBLIC_API_PREFIX` 未配置
- **影响**: 前端无法构建正确的 API 请求 URL
- **解决方案**: 在 `.env` 中明确配置这两个变量

### 6. 内部通信地址配置错误
- **症状**: Plugin Daemon、Sandbox 等服务间通信失败
- **原因**: `INTERNAL_FILES_URL`、`PLUGIN_DIFY_INNER_API_URL` 指向错误地址
- **影响**: 文件处理、插件执行等功能异常
- **解决方案**: 配置正确的内部服务地址

---

## 📐 完整的环境变量配置框架

### 关键配置文件清单

| 文件路径 | 用途 | 服务范围 | 优先级 |
|---------|------|---------|--------|
| `.env` | 主配置文件 | API/Web/中间件 | 🔴 **必须** |
| `middleware.env` | 中间件配置（开发用） | 中间件服务 | 🟡 可选 |
| `docker-compose.yaml` | 服务编排 | 所有服务 | 🔴 **必须** |
| `docker-compose.middleware.yaml` | 中间件编排（开发用） | 中间件 | 🟡 可选 |

### 环境变量优先级

```
🔴 紧急（必须立即配置）
  ├─ SECRET_KEY (API 认证密钥)
  ├─ CONSOLE_API_URL (后端 API 地址)
  ├─ NEXT_PUBLIC_API_PREFIX (前端 API 前缀)
  └─ COOKIE_DOMAIN (Cookie 域名)

🟠 高优先级（直接影响功能）
  ├─ APP_API_URL (App 端 API 地址)
  ├─ CONSOLE_WEB_URL (前端 Web 地址)
  ├─ WEB_API_CORS_ALLOW_ORIGINS (CORS 白名单)
  ├─ NEXT_PUBLIC_COOKIE_DOMAIN (前端 Cookie 配置)
  └─ NEXT_PUBLIC_PUBLIC_API_PREFIX (公开 API 前缀)

🟡 中优先级（影响特定功能）
  ├─ INTERNAL_FILES_URL (内部文件服务地址)
  ├─ FILES_URL (文件下载地址)
  ├─ PLUGIN_DIFY_INNER_API_URL (Plugin 内部 API)
  └─ CONSOLE_CORS_ALLOW_ORIGINS (Console CORS)

🟢 低优先级（一般情况默认值即可）
  ├─ DEPLOY_ENV (部署环境)
  ├─ LOG_LEVEL (日志级别)
  └─ 其他数据库、存储配置
```

---

## 🔧 需要修改的核心环境变量

### 第一层：跨域与通信配置（最关键）

| 变量类别 | 变量名 | 默认值 | 需要修改为 | 优先级 | 说明 |
|---------|--------|--------|-----------|--------|------|
| **API地址** | `CONSOLE_API_URL` | *(空)* | `http://localhost:5001` | 🔴 必须 | 后端 API 地址（console 控制台用） |
| **API地址** | `APP_API_URL` | *(空)* | `http://localhost:5001` | 🔴 必须 | Web App API 地址 |
| **Web地址** | `CONSOLE_WEB_URL` | *(空)* | `http://localhost` | 🟠 高 | 前端 Console Web 地址 |
| **CORS** | `WEB_API_CORS_ALLOW_ORIGINS` | `*` | `http://localhost` | 🔴 必须 | 允许的请求来源（不要用 `*` 生产环境） |
| **CORS** | `CONSOLE_CORS_ALLOW_ORIGINS` | `*` | `http://localhost` | 🟠 高 | Console API CORS 配置 |
| **文件服务** | `FILES_URL` | *(空)* | `http://localhost:5001` | 🟡 中 | 文件下载基础 URL |
| **内部文件** | `INTERNAL_FILES_URL` | *(空)* | `http://api:5001` | 🟡 中 | 容器内文件访问地址 |

### 第二层：前端 API 连接配置

| 变量名 | 默认值 | 需要修改为 | 优先级 | 说明 |
|--------|--------|-----------|--------|------|
| `NEXT_PUBLIC_API_PREFIX` | *(空)* | `http://localhost:5001/console/api` | 🔴 必须 | Web 前端 Console API 前缀 |
| `NEXT_PUBLIC_PUBLIC_API_PREFIX` | *(空)* | `http://localhost:5001/api` | 🟠 高 | Web 前端公开 API 前缀 |

### 第三层：Cookie 和会话配置

| 变量名 | 默认值 | 需要修改为 | 优先级 | 说明 |
|--------|--------|-----------|--------|------|
| `COOKIE_DOMAIN` | *(空)* | `localhost` 或 `.example.com` | 🔴 必须 | API 端 Cookie 域名 |
| `NEXT_PUBLIC_COOKIE_DOMAIN` | *(空)* | *(与 COOKIE_DOMAIN 相同)* | 🔴 必须 | Web 端 Cookie 域名标记 |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | `60` | *(无需改)* | 🟢 低 | 访问令牌过期时间（分钟） |
| `REFRESH_TOKEN_EXPIRE_DAYS` | `30` | *(无需改)* | 🟢 低 | 刷新令牌过期时间（天） |

### 第四层：安全密钥配置

| 变量名 | 默认值 | 需要修改为 | 优先级 | 说明 |
|--------|--------|-----------|--------|------|
| `SECRET_KEY` | `sk-9f73s3lj...` | **生成新密钥** | 🔴 必须 | Flask 会话密钥、JWT 签名密钥 |
| `INIT_PASSWORD` | *(空)* | *(可选，初始密码)* | 🟡 中 | 第一次初始化时的管理员密码 |

### 第五层：Plugin Daemon 内部通信

| 变量名 | 默认值 | 需要修改为 | 优先级 | 说明 |
|--------|--------|-----------|--------|------|
| `PLUGIN_DIFY_INNER_API_URL` | `http://api:5001` | *(无需改)* | 🟠 高 | Plugin 访问 API 的内部地址 |
| `PLUGIN_DIFY_INNER_API_KEY` | `QaHbTe77...` | *(保持一致)* | 🟠 高 | Plugin 与 API 通信的密钥 |
| `INTERNAL_FILES_URL` | *(空)* | `http://api:5001` | 🟡 中 | Plugin 访问文件的内部地址 |

---

## 📊 部署场景与具体配置

### 场景 1：本地单机部署（localhost）

**使用场景**: 开发测试、本机验证

**访问方式**: `http://localhost`

```bash
# === 第一层：跨域与通信（必须配置）===
CONSOLE_API_URL=http://localhost:5001
APP_API_URL=http://localhost:5001
CONSOLE_WEB_URL=http://localhost
WEB_API_CORS_ALLOW_ORIGINS=http://localhost
CONSOLE_CORS_ALLOW_ORIGINS=http://localhost
FILES_URL=http://localhost:5001

# === 第二层：前端 API 连接（必须配置）===
NEXT_PUBLIC_API_PREFIX=http://localhost:5001/console/api
NEXT_PUBLIC_PUBLIC_API_PREFIX=http://localhost:5001/api

# === 第三层：Cookie 和会话（必须配置）===
COOKIE_DOMAIN=localhost
NEXT_PUBLIC_COOKIE_DOMAIN=

# === 第四层：安全密钥（必须配置）===
SECRET_KEY=<执行 openssl rand -base64 42 生成>

# === 第五层：Plugin Daemon（保持默认）===
INTERNAL_FILES_URL=http://api:5001
PLUGIN_DIFY_INNER_API_URL=http://api:5001
```

### 场景 2：局域网服务器部署

**使用场景**: 办公网络内部署

**访问方式**: `http://192.168.1.100`（假设服务器 IP）

```bash
# 将 localhost 替换为服务器 IP
CONSOLE_API_URL=http://192.168.1.100:5001
APP_API_URL=http://192.168.1.100:5001
CONSOLE_WEB_URL=http://192.168.1.100
WEB_API_CORS_ALLOW_ORIGINS=http://192.168.1.100
CONSOLE_CORS_ALLOW_ORIGINS=http://192.168.1.100
FILES_URL=http://192.168.1.100:5001

NEXT_PUBLIC_API_PREFIX=http://192.168.1.100:5001/console/api
NEXT_PUBLIC_PUBLIC_API_PREFIX=http://192.168.1.100:5001/api

COOKIE_DOMAIN=192.168.1.100
NEXT_PUBLIC_COOKIE_DOMAIN=

SECRET_KEY=<执行 openssl rand -base64 42 生成>

INTERNAL_FILES_URL=http://api:5001
PLUGIN_DIFY_INNER_API_URL=http://api:5001
```

### 场景 3：域名部署（生产环境推荐）

**使用场景**: 生产服务器、正式上线

**访问方式**: `https://dify.example.com`

```bash
# 使用完整域名和 HTTPS
CONSOLE_API_URL=https://dify.example.com
APP_API_URL=https://dify.example.com
CONSOLE_WEB_URL=https://dify.example.com
WEB_API_CORS_ALLOW_ORIGINS=https://dify.example.com
CONSOLE_CORS_ALLOW_ORIGINS=https://dify.example.com
FILES_URL=https://dify.example.com

NEXT_PUBLIC_API_PREFIX=https://dify.example.com/console/api
NEXT_PUBLIC_PUBLIC_API_PREFIX=https://dify.example.com/api

# Cookie 域名使用顶级域名（前缀点号可选）
COOKIE_DOMAIN=.example.com
NEXT_PUBLIC_COOKIE_DOMAIN=1

SECRET_KEY=<执行 openssl rand -base64 42 生成>

INTERNAL_FILES_URL=http://api:5001
PLUGIN_DIFY_INNER_API_URL=http://api:5001
```

### 场景 4：反向代理部署（Nginx + Docker）

**使用场景**: 多应用共用反向代理

**访问方式**: `https://dify.company.com`（通过 Nginx）

```bash
# Nginx 处理 HTTPS，内部使用 HTTP
CONSOLE_API_URL=https://dify.company.com
APP_API_URL=https://dify.company.com
CONSOLE_WEB_URL=https://dify.company.com
WEB_API_CORS_ALLOW_ORIGINS=https://dify.company.com
CONSOLE_CORS_ALLOW_ORIGINS=https://dify.company.com
FILES_URL=https://dify.company.com

NEXT_PUBLIC_API_PREFIX=https://dify.company.com/console/api
NEXT_PUBLIC_PUBLIC_API_PREFIX=https://dify.company.com/api

COOKIE_DOMAIN=.company.com
NEXT_PUBLIC_COOKIE_DOMAIN=1

SECRET_KEY=<执行 openssl rand -base64 42 生成>

INTERNAL_FILES_URL=http://api:5001
PLUGIN_DIFY_INNER_API_URL=http://api:5001
```

---

## 🔐 生成安全的 SECRET_KEY

### 方法 1：使用 OpenSSL（推荐）

```bash
openssl rand -base64 42
```

**示例输出**:
```
ABCdeFGHiJKLmNoPQRSTUVWXYZ0123456789+/==
```

### 方法 2：使用 Python

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
```

### 方法 3：使用 Docker 容器

```bash
docker run --rm alpine/openssl rand -base64 42
```

---

## 📋 实施步骤（分阶段）

### 阶段 1：诊断当前配置（10 分钟）

```bash
# 1. 检查 .env 文件是否存在
ls -la /path/to/dify/docker/.env

# 2. 查看当前配置的关键变量
grep -E "CONSOLE_API_URL|APP_API_URL|SECRET_KEY|COOKIE_DOMAIN|NEXT_PUBLIC" .env

# 3. 检查 Docker Compose 服务状态
docker compose ps

# 4. 查看 API 容器日志（检查认证错误）
docker compose logs api | grep -i "unauthorized\|401\|cors\|cookie"

# 5. 查看 Web 容器日志
docker compose logs web | grep -i "api\|error\|cors"

# 6. 检查容器间网络通信
docker compose exec api ping web
docker compose exec web ping api
```

### 阶段 2：生成必要的密钥（5 分钟）

```bash
# 1. 生成新的 SECRET_KEY
openssl rand -base64 42
# 复制输出结果，将其设置为 SECRET_KEY 值

# 2. 验证 PLUGIN_DAEMON_KEY 和 PLUGIN_DIFY_INNER_API_KEY
# 这两个值应该在所有配置中保持一致
grep "PLUGIN_DAEMON_KEY\|PLUGIN_DIFY_INNER_API_KEY" .env
```

### 阶段 3：更新 .env 文件（15 分钟）

```bash
# 1. 备份当前配置
cp .env .env.backup.$(date +%Y%m%d_%H%M%S)

# 2. 编辑 .env 文件，根据部署场景更新以下变量：
# - CONSOLE_API_URL
# - APP_API_URL
# - CONSOLE_WEB_URL
# - WEB_API_CORS_ALLOW_ORIGINS
# - NEXT_PUBLIC_API_PREFIX
# - NEXT_PUBLIC_PUBLIC_API_PREFIX
# - COOKIE_DOMAIN
# - NEXT_PUBLIC_COOKIE_DOMAIN
# - SECRET_KEY
# - FILES_URL

# 3. 验证 INTERNAL_FILES_URL 是否正确设置
# 应该是容器内部地址：http://api:5001
```

### 阶段 4：重启服务（10 分钟）

```bash
# 1. 停止所有服务
docker compose down

# 2. 确认所有容器已停止
docker compose ps

# 3. 清理旧容器和网络（可选但推荐）
docker compose down -v

# 4. 使用新配置启动服务
docker compose up -d

# 5. 验证服务状态
docker compose ps

# 6. 检查启动日志
docker compose logs -f api web
```

### 阶段 5：测试与验证（15 分钟）

```bash
# 1. 检查 API 服务是否运行正常
curl http://localhost:5001/health

# 2. 查看 API 日志确保没有错误
docker compose logs api | tail -50

# 3. 打开浏览器访问
# 进入 http://localhost 或配置的地址

# 4. 尝试登录
# - 输入用户名/密码
# - 观察浏览器控制台（F12 → Network）
# - 查看请求是否成功（200 OK 而不是 401）
# - 检查 Cookie 是否被保存

# 5. 验证请求详情
# 在浏览器 Network 标签中：
# - POST /console/api/auth/login 应返回 200
# - Response Headers 应包含 Set-Cookie
# - 后续请求的 Cookie 中应包含 access_token

# 6. 查看容器日志确保认证通过
docker compose logs api | grep -i "login\|token\|auth"
```

---

## ✅ 验证检查清单

完成配置后，逐一检查以下项目：

### 配置验证

- [ ] `.env` 文件中 `CONSOLE_API_URL` 已配置（非空）
- [ ] `.env` 文件中 `APP_API_URL` 已配置（非空）
- [ ] `.env` 文件中 `NEXT_PUBLIC_API_PREFIX` 已配置（非空）
- [ ] `.env` 文件中 `NEXT_PUBLIC_PUBLIC_API_PREFIX` 已配置（非空）
- [ ] `.env` 文件中 `SECRET_KEY` 已更换为自定义值（非默认）
- [ ] `.env` 文件中 `COOKIE_DOMAIN` 已配置
- [ ] `.env` 文件中 `NEXT_PUBLIC_COOKIE_DOMAIN` 已配置（与 COOKIE_DOMAIN 对应）
- [ ] `.env` 文件中 `WEB_API_CORS_ALLOW_ORIGINS` 已配置为具体地址（非通配符）
- [ ] `.env` 文件中 `INTERNAL_FILES_URL` 已配置为 `http://api:5001`

### 服务验证

- [ ] 所有 Docker 容器已启动：`docker compose ps` 显示所有服务 running
- [ ] API 服务响应正常：`curl http://localhost:5001/health` 返回 200
- [ ] 无启动错误：`docker compose logs` 中无 error 关键字

### 功能验证

- [ ] 能够访问 Web 界面：浏览器打开配置的地址
- [ ] 能够成功登录：输入用户名密码并登录
- [ ] 浏览器 Cookie 已保存：F12 → Application → Cookies 中看到 `access_token`
- [ ] 登录后不会立即退出：登录成功后页面能正常加载
- [ ] 网络请求正常：F12 → Network 中 API 请求返回 200 而非 401
- [ ] 没有 CORS 错误：F12 → Console 中无 CORS-related 错误信息

### API 请求验证

```bash
# 测试 API 认证端点
curl -X POST http://localhost:5001/console/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@example.com",
    "password": "your_password"
  }'

# 成功响应应包含：
# {
#   "result": "success",
#   "data": {
#     "access_token": "...",
#     "refresh_token": "...",
#     ...
#   }
# }
```

---

## 🚨 常见问题排查

### 问题 1：登录返回 401 Unauthorized

**可能原因**:
1. `SECRET_KEY` 未配置或不一致
2. API 端点地址错误
3. 数据库连接失败，无法验证用户
4. Token 过期或签名不正确

**排查步骤**:
```bash
# 检查 API 日志
docker compose logs api | grep -i "401\|unauthorized\|token"

# 检查 SECRET_KEY 是否正确设置
docker compose exec api printenv SECRET_KEY

# 重新生成 SECRET_KEY
openssl rand -base64 42
# 更新 .env 并重启
```

### 问题 2：登录成功后立即退出或被强制登出

**可能原因**:
1. `COOKIE_DOMAIN` 配置错误
2. `NEXT_PUBLIC_COOKIE_DOMAIN` 与 `COOKIE_DOMAIN` 不一致
3. 浏览器 Cookie 设置（SameSite 属性）
4. 会话超时设置过短

**排查步骤**:
```bash
# 检查 Cookie 设置
# 在浏览器 F12 → Application → Cookies 中查看：
# - 域名是否正确
# - Path 是否为 /
# - SameSite 属性
# - Secure 属性（HTTPS 需要）

# 检查配置
grep "COOKIE_DOMAIN\|ACCESS_TOKEN" .env

# 增加 Token 过期时间进行测试
# ACCESS_TOKEN_EXPIRE_MINUTES=1440  # 改为 24 小时
```

### 问题 3：浏览器出现 CORS 错误

**可能原因**:
1. `WEB_API_CORS_ALLOW_ORIGINS` 未包含前端地址
2. `CONSOLE_CORS_ALLOW_ORIGINS` 配置不当
3. API 未启用 CORS 支持

**排查步骤**:
```bash
# 在浏览器控制台查看 CORS 错误信息
# 例如：Access to XMLHttpRequest at 'http://api:5001' from 'http://localhost'
#       has been blocked by CORS policy

# 检查 CORS 配置
grep "CORS_ALLOW" .env

# 如果使用通配符 * 会导致问题，改为具体地址
# WEB_API_CORS_ALLOW_ORIGINS=http://localhost
# CONSOLE_CORS_ALLOW_ORIGINS=http://localhost
```

### 问题 4：Web 前端无法连接到 API

**可能原因**:
1. `NEXT_PUBLIC_API_PREFIX` 未配置
2. `NEXT_PUBLIC_API_PREFIX` 指向错误地址
3. API 服务未正常运行

**排查步骤**:
```bash
# 在浏览器 F12 → Network 中检查请求 URL
# 应该是 http://localhost:5001/console/api/...

# 检查配置
grep "NEXT_PUBLIC_API_PREFIX" .env

# 验证 API 服务状态
docker compose ps | grep api
docker compose logs api | tail -20

# 手动测试 API 连接
curl http://localhost:5001/console/api/info
```

### 问题 5：容器间网络通信失败

**可能原因**:
1. Docker 网络配置不正确
2. 防火墙阻止
3. 容器 DNS 解析失败

**排查步骤**:
```bash
# 测试容器间通信
docker compose exec api ping web
docker compose exec web ping api

# 检查容器网络
docker network ls
docker network inspect <network_name>

# 查看容器网络配置
docker compose exec api cat /etc/hosts

# 在 API 容器中测试 Web 服务
docker compose exec api curl http://web:3000

# 在 Web 容器中测试 API 服务
docker compose exec web curl http://api:5001/health
```

---

## 📝 配置修改示例

### 修改前（问题配置）

```bash
# .env 文件（修改前）
CONSOLE_API_URL=
APP_API_URL=
CONSOLE_WEB_URL=
NEXT_PUBLIC_API_PREFIX=
NEXT_PUBLIC_PUBLIC_API_PREFIX=
COOKIE_DOMAIN=
NEXT_PUBLIC_COOKIE_DOMAIN=
SECRET_KEY=sk-9f73s3ljTXVcMT3Blb3ljTqtsKiGHXVcMT3BlbkFJLK7U
WEB_API_CORS_ALLOW_ORIGINS=*
CONSOLE_CORS_ALLOW_ORIGINS=*
```

### 修改后（正确配置）

```bash
# .env 文件（修改后 - localhost 场景）
CONSOLE_API_URL=http://localhost:5001
APP_API_URL=http://localhost:5001
CONSOLE_WEB_URL=http://localhost
NEXT_PUBLIC_API_PREFIX=http://localhost:5001/console/api
NEXT_PUBLIC_PUBLIC_API_PREFIX=http://localhost:5001/api
COOKIE_DOMAIN=localhost
NEXT_PUBLIC_COOKIE_DOMAIN=
SECRET_KEY=ABCdeFGHiJKLmNoPQRSTUVWXYZ0123456789+/==
WEB_API_CORS_ALLOW_ORIGINS=http://localhost
CONSOLE_CORS_ALLOW_ORIGINS=http://localhost
FILES_URL=http://localhost:5001
INTERNAL_FILES_URL=http://api:5001
```

---

## 🔗 关键代码位置参考

理解以下代码位置可帮助深入理解配置的作用：

### API 认证相关

| 功能 | 文件位置 | 关键变量 |
|------|--------|--------|
| JWT Token 签名 | `api/libs/passport.py` | `SECRET_KEY` |
| Cookie 配置 | `api/libs/token.py` | `COOKIE_DOMAIN`, `CONSOLE_API_URL` |
| CORS 中间件 | `api/extensions/ext_cors.py` | `WEB_API_CORS_ALLOW_ORIGINS`, `CONSOLE_CORS_ALLOW_ORIGINS` |
| 登录端点 | `api/controllers/console/auth/login.py` | 使用上述配置进行认证 |

### Web 前端相关

| 功能 | 文件位置 | 关键变量 |
|------|--------|--------|
| API 请求封装 | `web/lib/http.ts` | `NEXT_PUBLIC_API_PREFIX` |
| 认证请求 | `web/service/auth.ts` | `NEXT_PUBLIC_API_PREFIX` |
| Cookie 处理 | `web/middleware.ts` | `NEXT_PUBLIC_COOKIE_DOMAIN` |

---

## 📚 相关文档链接

- **官方部署指南**: `/docker/README.md`
- **环境变量示例**: `/docker/.env.example`
- **Docker Compose 配置**: `/docker/docker-compose.yaml`
- **中间件配置**: `/docker/docker-compose.middleware.yaml`

---

## 🎯 总结

Dify Docker Compose 部署中的 401 认证错误通常源于以下几个关键配置缺失或不一致：

| 优先级 | 必须配置项 | 影响范围 |
|--------|-----------|--------|
| 🔴 紧急 | `SECRET_KEY` | JWT 认证、会话加密 |
| 🔴 紧急 | `CONSOLE_API_URL` + `NEXT_PUBLIC_API_PREFIX` | API 地址正确性 |
| 🔴 紧急 | `COOKIE_DOMAIN` + `NEXT_PUBLIC_COOKIE_DOMAIN` | Cookie 保存和验证 |
| 🟠 高 | `WEB_API_CORS_ALLOW_ORIGINS` | 跨域请求允许 |
| 🟡 中 | `INTERNAL_FILES_URL` | 内部服务通信 |

**解决步骤**：
1. ✅ 根据部署场景确定访问地址
2. ✅ 生成新的 `SECRET_KEY`
3. ✅ 更新 `.env` 文件中的关键变量
4. ✅ 重启 Docker 服务
5. ✅ 验证登录功能和浏览器请求

---

**最后更新**: 2025-01-11  
**维护者**: Sisyphus Agent  
**反馈**: 遇到问题请查阅对应的常见问题排查部分，或检查 Docker 容器日志
