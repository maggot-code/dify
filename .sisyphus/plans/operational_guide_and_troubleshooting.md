# Dify Docker Compose 部署 - 实际操作指南与故障排查

**创建日期**: 2025-01-11  
**目的**: 提供分步骤的实际操作流程和完整的故障排查决策树

---

## 📖 实际操作指南（完整步骤）

### 步骤 1：环境检查（5 分钟）

#### 1.1 检查 Docker 环境

```bash
# 检查 Docker 是否安装
docker --version
# 预期输出: Docker version 20.x.x+

# 检查 Docker Compose 版本
docker compose version
# 预期输出: Docker Compose version 2.x.x+

# 检查 Docker 服务是否运行
docker ps
# 如果报错 "Cannot connect to Docker daemon"，则 Docker 未启动
# macOS: 打开 Docker Desktop 应用
# Linux: 运行 systemctl start docker
```

#### 1.2 进入 docker 目录

```bash
cd /Users/codemaggot/github.com/maggot-code/dify/docker

# 验证目录结构
ls -la
# 应该看到:
# - docker-compose.yaml
# - docker-compose.middleware.yaml
# - .env (或 .env.example)
# - volumes/ (数据目录)
```

#### 1.3 检查当前配置

```bash
# 查看 .env 文件是否存在
[ -f .env ] && echo "✅ .env 文件存在" || echo "❌ .env 文件不存在"

# 检查关键变量是否已配置
echo "=== 当前关键配置 ==="
grep -E "^(CONSOLE_API_URL|APP_API_URL|SECRET_KEY|COOKIE_DOMAIN|NEXT_PUBLIC_API_PREFIX)" .env || echo "❌ 关键变量未配置"
```

---

### 步骤 2：创建/更新 .env 文件（10 分钟）

#### 2.1 创建 .env 文件（如果不存在）

```bash
# 如果 .env 不存在，从示例文件创建
if [ ! -f .env ]; then
    cp .env.example .env
    echo "✅ 已从 .env.example 创建 .env 文件"
else
    echo "ℹ️  .env 文件已存在"
fi

# 备份当前配置
cp .env .env.backup.$(date +%Y%m%d_%H%M%S)
echo "✅ 配置已备份到 .env.backup.*"
```

#### 2.2 生成 SECRET_KEY

```bash
# 生成新的 SECRET_KEY
NEW_SECRET_KEY=$(openssl rand -base64 42)
echo "生成的密钥: $NEW_SECRET_KEY"

# 在 macOS 上更新 .env（使用 sed）
sed -i '' "s/^SECRET_KEY=.*/SECRET_KEY=$NEW_SECRET_KEY/" .env

# 在 Linux 上更新 .env（使用 sed）
sed -i "s/^SECRET_KEY=.*/SECRET_KEY=$NEW_SECRET_KEY/" .env

# 验证更新
echo ""
echo "验证密钥更新:"
grep "SECRET_KEY=" .env | head -1
```

#### 2.3 根据部署场景更新其他变量

**选项 A：localhost 部署**

```bash
# 编辑 .env 文件，更新以下变量
cat >> .env << 'EOF'

# === 以下为本次添加的配置（localhost 场景）===

CONSOLE_API_URL=http://localhost:5001
APP_API_URL=http://localhost:5001
CONSOLE_WEB_URL=http://localhost
NEXT_PUBLIC_API_PREFIX=http://localhost:5001/console/api
NEXT_PUBLIC_PUBLIC_API_PREFIX=http://localhost:5001/api
COOKIE_DOMAIN=localhost
NEXT_PUBLIC_COOKIE_DOMAIN=
WEB_API_CORS_ALLOW_ORIGINS=http://localhost
CONSOLE_CORS_ALLOW_ORIGINS=http://localhost
FILES_URL=http://localhost:5001
INTERNAL_FILES_URL=http://api:5001
EOF

echo "✅ localhost 配置已添加到 .env"
```

**选项 B：IP 地址部署**

```bash
# 替换 IP_ADDRESS 为实际 IP（例：192.168.1.100）
IP_ADDRESS="192.168.1.100"

# 更新配置
sed -i '' "s/localhost/${IP_ADDRESS}/g" .env  # macOS
# sed -i "s/localhost/${IP_ADDRESS}/g" .env    # Linux

echo "✅ IP 地址配置已更新: $IP_ADDRESS"
```

**选项 C：域名部署**

```bash
# 替换 DOMAIN 为实际域名（例：dify.example.com）
DOMAIN="dify.example.com"

# 更新配置
cat > /tmp/domain_config.txt << EOF
CONSOLE_API_URL=https://${DOMAIN}
APP_API_URL=https://${DOMAIN}
CONSOLE_WEB_URL=https://${DOMAIN}
NEXT_PUBLIC_API_PREFIX=https://${DOMAIN}/console/api
NEXT_PUBLIC_PUBLIC_API_PREFIX=https://${DOMAIN}/api
COOKIE_DOMAIN=.$(echo ${DOMAIN} | cut -d. -f2-)
NEXT_PUBLIC_COOKIE_DOMAIN=1
WEB_API_CORS_ALLOW_ORIGINS=https://${DOMAIN}
CONSOLE_CORS_ALLOW_ORIGINS=https://${DOMAIN}
FILES_URL=https://${DOMAIN}
INTERNAL_FILES_URL=http://api:5001
EOF

# 附加到 .env
cat /tmp/domain_config.txt >> .env
echo "✅ 域名配置已添加: $DOMAIN"
```

#### 2.4 验证配置

```bash
# 检查关键变量是否已正确配置
echo "=== 验证关键配置 ==="
echo ""
echo "SECRET_KEY:"
grep "^SECRET_KEY=" .env | head -1
echo ""
echo "CONSOLE_API_URL:"
grep "^CONSOLE_API_URL=" .env | head -1
echo ""
echo "NEXT_PUBLIC_API_PREFIX:"
grep "^NEXT_PUBLIC_API_PREFIX=" .env | head -1
echo ""
echo "COOKIE_DOMAIN:"
grep "^COOKIE_DOMAIN=" .env | head -1
echo ""
echo "WEB_API_CORS_ALLOW_ORIGINS:"
grep "^WEB_API_CORS_ALLOW_ORIGINS=" .env | head -1
```

---

### 步骤 3：启动 Docker 服务（10 分钟）

#### 3.1 停止现有服务

```bash
# 检查当前运行的服务
echo "=== 当前服务状态 ==="
docker compose ps

# 停止所有服务
echo ""
echo "⏹️  停止服务..."
docker compose down

# 等待 1-2 秒
sleep 2

# 验证服务已停止
echo "✅ 服务已停止"
docker compose ps
```

#### 3.2 启动新服务

```bash
# 启动所有服务（后台运行）
echo "🚀 启动服务..."
docker compose up -d

# 如果需要查看启动日志（实时输出）
# docker compose up （不加 -d 参数）
# 按 Ctrl+C 退出日志查看

# 等待服务完全启动（30 秒）
echo "⏳ 等待服务启动..."
for i in {30..1}; do
    echo -n "$i "
    sleep 1
done
echo ""

# 验证服务状态
echo ""
echo "=== 服务启动状态 ==="
docker compose ps
```

#### 3.3 检查启动日志

```bash
# 查看 API 服务日志（最近 20 行）
echo "=== API 服务日志 ==="
docker compose logs api | tail -20

# 查看 Web 服务日志
echo ""
echo "=== Web 服务日志 ==="
docker compose logs web | tail -20

# 查看是否有错误
echo ""
echo "检查错误信息..."
docker compose logs | grep -i "error\|failed\|exception" | tail -10 || echo "✅ 没有明显的错误信息"
```

---

### 步骤 4：验证服务连接（5 分钟）

#### 4.1 测试 API 服务

```bash
# 测试 API 健康状态
echo "测试 API 服务..."
curl -s http://localhost:5001/health || echo "❌ API 无响应"

# 如果上面测试失败，尝试检查 API 容器是否正在运行
docker compose ps | grep api

# 查看 API 容器日志（排查问题）
docker compose logs api | tail -50
```

#### 4.2 测试容器间通信

```bash
# 从 API 容器访问 Web 容器
echo "测试 API 到 Web 的连接..."
docker compose exec api ping -c 1 web

# 从 Web 容器访问 API 容器
echo "测试 Web 到 API 的连接..."
docker compose exec web ping -c 1 api

# 测试 API 容器访问自身的内部地址
echo "测试 API 内部地址..."
docker compose exec api curl -s http://api:5001/health
```

#### 4.3 测试数据库连接

```bash
# 检查数据库服务是否运行
echo "数据库服务状态:"
docker compose ps | grep -E "postgres|mysql"

# 检查 API 日志中是否有数据库连接错误
docker compose logs api | grep -iE "database|connect|postgres|mysql" | tail -10
```

---

### 步骤 5：浏览器测试（5 分钟）

#### 5.1 访问 Web 界面

```bash
# 根据配置的地址打开浏览器
# localhost 部署: http://localhost
# IP 部署:       http://192.168.1.100
# 域名部署:      https://dify.example.com

# 注意: 可能需要等待 30-60 秒 Web 容器完全启动
```

#### 5.2 检查浏览器控制台（F12）

打开浏览器开发者工具（F12），检查：

**Console 标签** - 查看是否有错误
```
预期: 没有或很少的 error 信息
警惕: CORS error, Cannot access, 404 等错误
```

**Network 标签** - 检查 API 请求
```
预期: 
  - GET /console/api/info → 200
  - 其他 API 请求 → 200

警惕:
  - CORS blocked 错误
  - 401 Unauthorized
  - 连接超时 (timeout)
```

**Application 标签** - 查看 Cookie
```
预期:
  - Domain: 与配置的 COOKIE_DOMAIN 匹配
  - 登录后出现 access_token cookie
  - 登录后出现 refresh_token cookie

警惕:
  - 没有 domain 设置
  - Cookie 丢失或无法保存
```

#### 5.3 登录测试

```bash
# 使用默认凭证尝试登录（如果未设置 INIT_PASSWORD）
# 邮箱: admin@dify.ai
# 密码: (根据初始化配置)

# 或者创建新账户进行测试

# 观察：
# 1. 点击登录后页面是否有加载动画
# 2. F12 Network 标签中是否看到 POST /console/api/auth/login 请求
# 3. 该请求的状态码是否为 200（而非 401）
# 4. 返回的 Response 是否包含 access_token
# 5. 登录后是否跳转到主页面
# 6. 是否能看到工作区和项目等信息
```

---

### 步骤 6：认证测试（API 级别）

```bash
# 使用 curl 手动测试登录端点
echo "=== 测试 API 认证端点 ==="
echo ""

# 1. 首先获取 info（无需认证）
echo "1. 获取服务信息（无需认证）:"
curl -s http://localhost:5001/console/api/info | python3 -m json.tool | head -20

# 2. 尝试登录（获取 token）
echo ""
echo "2. 尝试登录（获取 access token）:"
curl -s -X POST http://localhost:5001/console/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@dify.ai",
    "password": "your_password"
  }' | python3 -m json.tool

# 3. 如果登录成功，使用返回的 token 进行认证请求
# 需要从上面的响应中提取 access_token 值
```

---

## 🔍 故障排查决策树

```
开始 → 能否访问 Web 界面？
│
├─ NO → 检查 Docker 是否运行
│        ├─ docker ps 有报错？
│        │  YES → 启动 Docker 服务
│        │  NO → 进行步骤 A
│        │
│        └─ 步骤 A: 检查服务日志
│           ├─ docker compose logs | grep error
│           └─ 根据错误信息修复
│
└─ YES → 能否成功登录？
         │
         ├─ NO (401 错误) → 进行故障排查 - 401 错误分支
         │
         ├─ NO (CORS 错误) → 进行故障排查 - CORS 错误分支
         │
         └─ YES → 登录后能保持会话吗？
                  │
                  ├─ NO (登录后立即退出) → 进行故障排查 - Cookie 错误分支
                  │
                  └─ YES → ✅ 配置成功！
```

### 分支 1: 401 错误排查流程

```
401 Unauthorized 错误
│
├─ 第一步：确认 SECRET_KEY 是否已正确配置
│  ├─ grep "SECRET_KEY=" .env 显示默认值？
│  │  YES → 生成新的 SECRET_KEY 并重启
│  │  NO  → 进行第二步
│  │
│  └─ docker compose exec api printenv SECRET_KEY 与 .env 中的一致？
│     NO  → 重启服务: docker compose down && docker compose up -d
│     YES → 进行第二步
│
├─ 第二步：检查用户是否存在于数据库
│  ├─ 查看 API 日志中的用户认证信息
│  │  docker compose logs api | grep -i "login\|auth"
│  │  
│  └─ 如果是新部署，可能需要创建用户或使用默认初始化
│
├─ 第三步：检查数据库连接
│  ├─ docker compose ps | grep -E "postgres|mysql"
│  │  └─ 数据库容器是否在运行？
│  │
│  └─ docker compose logs api | grep -i "database\|connection"
│     └─ 有连接错误吗？
│
└─ 第四步：如果上述步骤都无法解决
   └─ 查看完整的 API 日志
      docker compose logs api > api_logs.txt
      查找 "401" 或 "Unauthorized" 相关的错误堆栈
```

### 分支 2: CORS 错误排查流程

```
CORS 错误（浏览器控制台中的 CORS blocked 信息）
│
├─ 第一步：识别错误中的关键信息
│  └─ 记下错误信息中的 Origin （来源）
│     例如：Access to XMLHttpRequest at 'http://api:5001'
│           from origin 'http://localhost' has been blocked
│
├─ 第二步：检查 CORS 白名单配置
│  └─ grep "CORS_ALLOW" .env
│     CONSOLE_CORS_ALLOW_ORIGINS=?
│     WEB_API_CORS_ALLOW_ORIGINS=?
│
├─ 第三步：验证配置是否正确
│  ├─ 如果使用 * （通配符）
│  │  └─ ⚠️ 这可能导致某些浏览器的 CORS 预检请求失败
│  │     改为具体地址
│  │
│  └─ 如果使用具体地址
│     └─ 检查是否与浏览器访问的地址一致
│        例如：浏览器访问 http://localhost 但 CORS 配置的是 http://127.0.0.1
│
├─ 第四步：更新配置并重启
│  └─ 编辑 .env，更新 CORS 配置
│     docker compose restart api
│
└─ 第五步：清除浏览器 cache 并重新加载
   └─ Ctrl+Shift+Delete (强制刷新)
      或在 Network 标签中禁用缓存后重新加载
```

### 分支 3: Cookie/Session 错误排查流程

```
登录后立即退出或无法保持会话
│
├─ 第一步：检查 Cookie 域名配置
│  └─ grep "COOKIE_DOMAIN" .env
│     COOKIE_DOMAIN=?
│     NEXT_PUBLIC_COOKIE_DOMAIN=?
│
├─ 第二步：检查浏览器中的 Cookie
│  ├─ 打开 F12 → Application → Cookies
│  ├─ 查看 access_token 的 Domain 属性
│  │
│  └─ 验证配置：
│     ├─ 如果访问 http://localhost，Domain 应该是 localhost
│     ├─ 如果访问 http://192.168.1.100，Domain 应该是 192.168.1.100
│     └─ 如果访问 https://dify.example.com，Domain 应该是 .example.com
│
├─ 第三步：检查 Token 过期时间
│  └─ grep "TOKEN_EXPIRE" .env
│     ACCESS_TOKEN_EXPIRE_MINUTES=60
│     └─ 如果是 1 分钟，可能快速过期
│
├─ 第四步：检查 API 日志中的 Cookie 相关信息
│  └─ docker compose logs api | grep -i "cookie\|session"
│
├─ 第五步：更新配置并重启
│  └─ 确保 COOKIE_DOMAIN 和 NEXT_PUBLIC_COOKIE_DOMAIN 一致
│     docker compose down
│     docker compose up -d
│
└─ 第六步：清除浏览器数据重新测试
   ├─ F12 → Application → Cookies → 删除所有 Cookie
   ├─ F12 → Application → Local Storage → 清空
   └─ 重新登录测试
```

---

## 🛠️ 常见命令速查

### 日志查看

```bash
# 查看所有服务日志
docker compose logs

# 查看特定服务日志（API）
docker compose logs api

# 实时查看日志
docker compose logs -f api

# 查看最后 50 行日志
docker compose logs api | tail -50

# 搜索特定关键词
docker compose logs | grep "ERROR\|401"

# 保存日志到文件
docker compose logs > all_logs.txt
docker compose logs api > api_logs.txt
```

### 服务管理

```bash
# 查看服务状态
docker compose ps

# 启动所有服务
docker compose up -d

# 停止所有服务
docker compose down

# 重启特定服务
docker compose restart api

# 停止特定服务
docker compose stop api

# 启动特定服务
docker compose start api

# 删除所有容器和数据（警告：会丢失数据！）
docker compose down -v
```

### 容器交互

```bash
# 进入容器 shell
docker compose exec api /bin/bash
docker compose exec web /bin/bash

# 在容器中执行命令
docker compose exec api printenv SECRET_KEY
docker compose exec api curl http://web:3000

# 查看容器中的文件
docker compose exec api ls -la /app/api

# 查看容器资源使用
docker compose stats
```

### 网络诊断

```bash
# 测试容器间连接
docker compose exec api ping -c 1 web
docker compose exec web ping -c 1 api

# 查看容器网络配置
docker network ls
docker network inspect <network_name>

# 在容器中进行 DNS 查询
docker compose exec api nslookup web

# 在容器中测试 API 连接
docker compose exec api curl http://api:5001/health
```

### 配置验证

```bash
# 检查 .env 文件内容
cat .env

# 查看 docker-compose.yaml 配置
cat docker-compose.yaml

# 验证 docker-compose.yaml 语法
docker compose config

# 检查特定环境变量
grep "CONSOLE_API_URL" .env
grep "SECRET_KEY" .env

# 比较 .env 和 .env.example 的差异
diff .env .env.example
```

---

## 📊 常见问题排查对照表

| 现象 | 可能原因 | 快速诊断 | 解决方案 |
|------|--------|--------|--------|
| **无法访问 Web** | Docker 未启动或服务未运行 | `docker compose ps` | 启动 Docker 和服务 |
| **Web 出现 404 error** | 文件不存在或 nginx 配置错误 | `docker compose logs web` | 检查 nginx 日志 |
| **API 无法连接** | API 服务未启动或端口错误 | `curl http://localhost:5001/health` | 检查 API 服务状态 |
| **401 错误（登录失败）** | SECRET_KEY 错误或用户不存在 | `docker compose logs api \| grep auth` | 更新 SECRET_KEY 或创建用户 |
| **CORS 错误** | CORS 白名单配置不正确 | `grep CORS .env` | 更新 CORS 配置为具体地址 |
| **登录后立即退出** | Cookie Domain 不匹配 | F12 → Cookies 检查 domain | 更新 COOKIE_DOMAIN |
| **容器间无法通信** | 网络隔离或 DNS 失败 | `docker compose exec api ping web` | 检查容器网络配置 |
| **数据库连接失败** | 数据库容器未启动或凭证错误 | `docker compose ps \| grep db` | 启动数据库，验证凭证 |
| **文件上传失败** | 文件存储配置错误 | `docker compose logs api \| grep file` | 检查 STORAGE_TYPE 和权限 |
| **性能缓慢** | 资源不足或配置不优化 | `docker compose stats` | 增加资源限制或优化配置 |

---

最后更新: 2025-01-11
