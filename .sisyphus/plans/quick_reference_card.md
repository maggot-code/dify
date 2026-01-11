# Dify Docker Compose 环境变量配置 - 快速参考卡

**创建日期**: 2025-01-11  
**用途**: 快速查找和复制配置代码片段

---

## 🎯 30 秒快速配置

### 1. 生成 SECRET_KEY
```bash
openssl rand -base64 42
# 复制输出结果
```

### 2. 选择部署场景，复制对应配置

#### localhost 部署
```bash
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
SECRET_KEY=<粘贴上面生成的密钥>
```

#### IP 地址部署（例：192.168.1.100）
```bash
CONSOLE_API_URL=http://192.168.1.100:5001
APP_API_URL=http://192.168.1.100:5001
CONSOLE_WEB_URL=http://192.168.1.100
NEXT_PUBLIC_API_PREFIX=http://192.168.1.100:5001/console/api
NEXT_PUBLIC_PUBLIC_API_PREFIX=http://192.168.1.100:5001/api
COOKIE_DOMAIN=192.168.1.100
NEXT_PUBLIC_COOKIE_DOMAIN=
WEB_API_CORS_ALLOW_ORIGINS=http://192.168.1.100
CONSOLE_CORS_ALLOW_ORIGINS=http://192.168.1.100
FILES_URL=http://192.168.1.100:5001
INTERNAL_FILES_URL=http://api:5001
SECRET_KEY=<粘贴上面生成的密钥>
```

#### 域名部署（例：dify.example.com）
```bash
CONSOLE_API_URL=https://dify.example.com
APP_API_URL=https://dify.example.com
CONSOLE_WEB_URL=https://dify.example.com
NEXT_PUBLIC_API_PREFIX=https://dify.example.com/console/api
NEXT_PUBLIC_PUBLIC_API_PREFIX=https://dify.example.com/api
COOKIE_DOMAIN=.example.com
NEXT_PUBLIC_COOKIE_DOMAIN=1
WEB_API_CORS_ALLOW_ORIGINS=https://dify.example.com
CONSOLE_CORS_ALLOW_ORIGINS=https://dify.example.com
FILES_URL=https://dify.example.com
INTERNAL_FILES_URL=http://api:5001
SECRET_KEY=<粘贴上面生成的密钥>
```

### 3. 重启服务
```bash
cd /path/to/dify/docker
docker compose down
docker compose up -d
```

### 4. 验证
```bash
# 检查服务状态
docker compose ps

# 查看日志（出错时查看）
docker compose logs api
docker compose logs web

# 尝试登录：打开浏览器访问配置的地址
```

---

## 🔍 快速诊断命令

```bash
# 查看当前关键配置
grep -E "CONSOLE_API_URL|APP_API_URL|SECRET_KEY|COOKIE_DOMAIN|NEXT_PUBLIC" .env

# 检查服务运行状态
docker compose ps

# 查看 API 认证相关日志
docker compose logs api | grep -iE "401|unauthorized|token|cors|cookie" | tail -20

# 查看 Web 连接错误
docker compose logs web | grep -iE "error|failed|api|cors" | tail -20

# 测试 API 端点
curl http://localhost:5001/health

# 测试容器间通信
docker compose exec api ping web
docker compose exec web ping api

# 查看 SECRET_KEY 是否正确设置
docker compose exec api printenv SECRET_KEY

# 手动测试登录
curl -X POST http://localhost:5001/console/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"password"}'
```

---

## 📋 常见错误速查表

| 错误 | 原因 | 解决方案 |
|------|------|--------|
| **401 Unauthorized** | SECRET_KEY 错误 或 用户不存在 | 更新 SECRET_KEY，重启服务 |
| **登录后立即退出** | COOKIE_DOMAIN 不一致 | 确保 COOKIE_DOMAIN 和 NEXT_PUBLIC_COOKIE_DOMAIN 对应 |
| **CORS error** | WEB_API_CORS_ALLOW_ORIGINS 不正确 | 改为具体地址，不要用 `*` |
| **无法连接 API** | NEXT_PUBLIC_API_PREFIX 为空 | 配置 NEXT_PUBLIC_API_PREFIX |
| **网络请求超时** | CONSOLE_API_URL 指向错误地址 | 检查 CONSOLE_API_URL 是否可访问 |
| **Plugin 错误** | INTERNAL_FILES_URL 不正确 | 设置为 `http://api:5001` |

---

## 🔧 环境变量速查（按优先级）

### 🔴 紧急（5个）

```bash
SECRET_KEY                          # JWT 签名密钥 - 必须更改为自定义值
CONSOLE_API_URL                     # API 地址 - 后端服务地址
NEXT_PUBLIC_API_PREFIX              # Web 前端 API 前缀
COOKIE_DOMAIN                       # Cookie 域名
NEXT_PUBLIC_COOKIE_DOMAIN           # Web 端 Cookie 标记
```

### 🟠 高优先级（5个）

```bash
APP_API_URL                         # App API 地址
CONSOLE_WEB_URL                     # Web 前端地址
WEB_API_CORS_ALLOW_ORIGINS          # CORS 白名单
NEXT_PUBLIC_PUBLIC_API_PREFIX       # 公开 API 前缀
CONSOLE_CORS_ALLOW_ORIGINS          # Console CORS
```

### 🟡 中优先级（4个）

```bash
FILES_URL                           # 文件下载地址
INTERNAL_FILES_URL                  # 容器内文件地址（应为 http://api:5001）
PLUGIN_DIFY_INNER_API_URL           # Plugin 内部 API 地址
PLUGIN_DIFY_INNER_API_KEY           # Plugin 内部 API 密钥
```

---

## ✅ 部署检查清单

**配置阶段**
- [ ] 生成新的 SECRET_KEY
- [ ] 确定访问地址（localhost / IP / 域名）
- [ ] 更新 .env 文件中的 10+ 个关键变量
- [ ] 备份原配置：`cp .env .env.backup`

**启动阶段**
- [ ] 停止旧服务：`docker compose down`
- [ ] 启动新服务：`docker compose up -d`
- [ ] 验证服务状态：`docker compose ps`（所有服务 running）

**测试阶段**
- [ ] API 健康检查：`curl http://localhost:5001/health` → 200
- [ ] 浏览器访问：打开配置的地址
- [ ] 登录测试：输入用户名密码
- [ ] Cookie 验证：F12 → Application → Cookies （看 access_token）
- [ ] Network 检查：F12 → Network → 所有 API 请求都是 200（非 401）

**验证成功标志**
- ✅ 能登录
- ✅ 登录后页面正常加载
- ✅ 不会立即退出
- ✅ 没有 CORS 错误
- ✅ 没有 401 错误

---

## 🔐 密钥管理

### 生成 SECRET_KEY

```bash
# 推荐：OpenSSL（最安全）
openssl rand -base64 42

# 备选：Python
python3 -c "import secrets; print(secrets.token_urlsafe(32))"

# 备选：Docker
docker run --rm alpine/openssl rand -base64 42
```

### 验证密钥

```bash
# 查看容器中的密钥
docker compose exec api printenv SECRET_KEY

# 查看 .env 文件中的密钥
grep "SECRET_KEY" .env

# 验证密钥长度（应该 > 32 字符）
openssl rand -base64 42 | wc -c
```

---

## 📞 常见问题速查

### Q: 登录时显示 401 Unauthorized
**A**: 
1. 检查 SECRET_KEY 是否已更新：`grep SECRET_KEY .env`
2. 检查用户是否存在
3. 查看日志：`docker compose logs api | grep 401`
4. 重启服务：`docker compose restart api`

### Q: 登录后立即退出登录
**A**:
1. 检查 COOKIE_DOMAIN：`grep COOKIE_DOMAIN .env`
2. 检查浏览器 Cookie：F12 → Application → Cookies
3. 验证 NEXT_PUBLIC_COOKIE_DOMAIN 是否正确设置

### Q: 浏览器出现 CORS 错误
**A**:
1. 检查 WEB_API_CORS_ALLOW_ORIGINS：`grep WEB_API_CORS .env`
2. 改为具体地址（不要用 `*`）：`WEB_API_CORS_ALLOW_ORIGINS=http://localhost`
3. 重启 API 服务：`docker compose restart api`

### Q: 网页无法加载（白屏）
**A**:
1. 检查 NEXT_PUBLIC_API_PREFIX：`grep NEXT_PUBLIC_API_PREFIX .env`
2. 打开 F12 查看 Console 和 Network 标签
3. 检查 Web 容器日志：`docker compose logs web`

### Q: 怎样完全重置所有配置？
**A**:
```bash
# 1. 停止并删除所有容器和数据
docker compose down -v

# 2. 从示例重新创建 .env
cp .env.example .env

# 3. 编辑 .env 重新配置
vi .env

# 4. 启动服务
docker compose up -d
```

---

## 🚀 一键快速启动脚本

保存以下内容为 `quick_start.sh`：

```bash
#!/bin/bash

set -e

echo "🔧 Dify Docker Compose 快速启动脚本"
echo ""

# 1. 生成 SECRET_KEY
echo "📝 生成 SECRET_KEY..."
SECRET_KEY=$(openssl rand -base64 42)
echo "生成的密钥: $SECRET_KEY"

# 2. 选择部署场景
echo ""
echo "选择部署场景:"
echo "1) localhost (本地开发)"
echo "2) IP 地址 (局域网)"
echo "3) 域名 (生产环境)"
read -p "请选择 (1-3): " scenario

# 3. 备份原配置
if [ -f .env ]; then
    cp .env .env.backup.$(date +%Y%m%d_%H%M%S)
    echo "✅ 原配置已备份"
fi

# 4. 停止旧服务
echo ""
echo "⏹️  停止旧服务..."
docker compose down

# 5. 启动新服务
echo ""
echo "🚀 启动服务..."
docker compose up -d

# 6. 等待服务启动
echo "⏳ 等待服务启动（30秒）..."
sleep 30

# 7. 验证服务
echo ""
echo "✅ 服务状态:"
docker compose ps

echo ""
echo "✅ 快速启动完成！"
echo ""
echo "后续步骤:"
echo "1. 编辑 .env 文件，更新以下变量："
echo "   - CONSOLE_API_URL"
echo "   - NEXT_PUBLIC_API_PREFIX"
echo "   - COOKIE_DOMAIN"
echo "   - WEB_API_CORS_ALLOW_ORIGINS"
echo "   将 SECRET_KEY 设置为: $SECRET_KEY"
echo ""
echo "2. 重启服务: docker compose restart"
echo "3. 打开浏览器访问配置的地址"
```

使用方法：
```bash
chmod +x quick_start.sh
./quick_start.sh
```

---

## 📊 配置对比表

| 功能点 | localhost | IP 地址 | 域名 |
|--------|-----------|---------|------|
| CONSOLE_API_URL | http://localhost:5001 | http://IP:5001 | https://domain |
| COOKIE_DOMAIN | localhost | IP | .domain |
| HTTPS | ❌ | ❌ | ✅ |
| 生产可用 | ❌ | ⚠️ | ✅ |
| 配置复杂度 | ⭐ | ⭐⭐ | ⭐⭐⭐ |

---

## 🎓 相关概念解释

### SECRET_KEY
- **作用**: Flask 会话加密和 JWT Token 签名
- **重要性**: 🔴 最高 - 决定认证是否有效
- **更新频率**: 首次部署时生成，之后不需要频繁更改

### COOKIE_DOMAIN
- **作用**: 浏览器 Cookie 的有效域名范围
- **例子**: 
  - `localhost` → 只在 localhost 域名下有效
  - `.example.com` → 在 example.com 及其所有子域名下有效
- **关键点**: 必须与访问地址匹配

### NEXT_PUBLIC_API_PREFIX
- **作用**: 前端代码中的 API 请求基础 URL
- **特点**: `NEXT_PUBLIC_*` 前缀表示这是在浏览器中可见的公开变量
- **格式**: 必须是完整 URL（包含 http:// 或 https://）

### CORS（跨域资源共享）
- **目的**: 防止恶意网站访问 API
- **配置**: `WEB_API_CORS_ALLOW_ORIGINS` 指定允许的来源
- **避坑**: 生产环境不要用 `*`，应指定具体地址

---

最后更新: 2025-01-11
