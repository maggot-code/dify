# Dify Docker Compose 配置快速参考

## 📝 必须修改的 5 个配置项

复制此模板到 `.env` 文件中，根据部署环境修改：

### 本地开发环境 (localhost)

```bash
# ============================================
# 1. API 与前端 URL 配置
# ============================================
CONSOLE_API_URL=http://localhost:5001
CONSOLE_WEB_URL=http://localhost
APP_API_URL=http://localhost:5001
FILES_URL=http://localhost:5001
INTERNAL_FILES_URL=http://api:5001

# ============================================
# 2. Cookie 与跨域配置
# ============================================
COOKIE_DOMAIN=localhost
WEB_API_CORS_ALLOW_ORIGINS=http://localhost
CONSOLE_CORS_ALLOW_ORIGINS=http://localhost

# ============================================
# 3. 加密密钥（必须生成新值！）
# ============================================
# 执行：openssl rand -base64 42
# 然后复制结果到这里：
SECRET_KEY=生成一个新的密钥粘贴到这里

# ============================================
# 4. 数据库配置（保持默认即可）
# ============================================
DB_USERNAME=postgres
DB_PASSWORD=difyai123456
DB_HOST=db_postgres
DB_PORT=5432
DB_DATABASE=dify

# ============================================
# 5. Redis 配置（保持默认即可）
# ============================================
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=
```

---

### 远程服务器部署 (example.com)

```bash
# ============================================
# 1. API 与前端 URL 配置（替换 example.com）
# ============================================
CONSOLE_API_URL=https://api.example.com
CONSOLE_WEB_URL=https://example.com
APP_API_URL=https://api.example.com
FILES_URL=https://api.example.com
INTERNAL_FILES_URL=http://api:5001

# ============================================
# 2. Cookie 与跨域配置
# ============================================
COOKIE_DOMAIN=example.com
WEB_API_CORS_ALLOW_ORIGINS=https://example.com,https://api.example.com
CONSOLE_CORS_ALLOW_ORIGINS=https://example.com,https://api.example.com

# ============================================
# 3. 加密密钥（必须生成新值！）
# ============================================
SECRET_KEY=生成一个新的密钥粘贴到这里

# ============================================
# 4. 数据库配置（保持默认即可）
# ============================================
DB_USERNAME=postgres
DB_PASSWORD=difyai123456
DB_HOST=db_postgres
DB_PORT=5432
DB_DATABASE=dify

# ============================================
# 5. Redis 配置（保持默认即可）
# ============================================
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=
```

---

## 🔍 配置验证清单

完成上述配置后，执行以下命令验证：

```bash
# 1. 确认 SECRET_KEY 已设置
grep "^SECRET_KEY=" .env

# 2. 确认 API URL 设置正确
grep "CONSOLE_API_URL\|APP_API_URL" .env

# 3. 确认 COOKIE_DOMAIN 设置正确
grep "COOKIE_DOMAIN" .env

# 4. 启动服务
docker compose up -d

# 5. 等待 30-60 秒，检查容器状态
docker compose ps
# 应显示所有容器为 "Up"

# 6. 测试 API 连接
curl -i http://localhost:5001/console/api/workspaces
# 应返回 HTTP 401（说明 API 正常运行）

# 7. 访问前端
# http://localhost/install（本地）或
# https://example.com/install（远程）
```

---

## ⚡ 一键部署脚本

保存为 `deploy.sh`，运行 `bash deploy.sh`：

```bash
#!/bin/bash

# 定颜色输出
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m' # No Color

echo -e "${YELLOW}========== Dify 部署工具 ==========${NC}"

# 1. 检查文件
if [ ! -f ".env.example" ]; then
    echo -e "${RED}❌ 错误：未找到 .env.example 文件${NC}"
    exit 1
fi

# 2. 创建 .env 如果不存在
if [ ! -f ".env" ]; then
    echo -e "${YELLOW}📝 创建 .env 文件...${NC}"
    cp .env.example .env
    echo -e "${GREEN}✅ .env 已创建${NC}"
else
    echo -e "${YELLOW}ℹ️  .env 文件已存在${NC}"
fi

# 3. 检查 SECRET_KEY
SECRET_KEY=$(grep "^SECRET_KEY=" .env | cut -d'=' -f2)
if [ -z "$SECRET_KEY" ] || [ "$SECRET_KEY" = "sk-9f73s3ljTXVcMT3Blb3ljTqtsKiGHXVcMT3BlbkFJLK7U" ]; then
    echo -e "${YELLOW}🔑 生成 SECRET_KEY...${NC}"
    NEW_SECRET=$(openssl rand -base64 42)
    sed -i.bak "s/^SECRET_KEY=.*/SECRET_KEY=$NEW_SECRET/" .env
    echo -e "${GREEN}✅ SECRET_KEY 已生成: $NEW_SECRET${NC}"
fi

# 4. 提示用户修改配置
echo -e "${YELLOW}📋 请修改 .env 文件中的以下变量：${NC}"
echo -e "   - CONSOLE_API_URL"
echo -e "   - CONSOLE_WEB_URL"
echo -e "   - APP_API_URL"
echo -e "   - FILES_URL"
echo -e "   - COOKIE_DOMAIN"
read -p "修改完成？(y/n) " -n 1 -r
echo
if [[ ! $REPLY =~ ^[Yy]$ ]]; then
    echo -e "${RED}❌ 已取消${NC}"
    exit 1
fi

# 5. 启动服务
echo -e "${YELLOW}🚀 启动 Docker 服务...${NC}"
docker compose down 2>/dev/null || true
docker compose up -d

# 6. 等待服务启动
echo -e "${YELLOW}⏳ 等待服务启动（30 秒）...${NC}"
sleep 30

# 7. 验证
echo -e "${YELLOW}🔍 验证服务状态...${NC}"
if docker compose ps | grep -q "Up"; then
    echo -e "${GREEN}✅ 容器已启动${NC}"
else
    echo -e "${RED}❌ 容器启动失败，查看日志：${NC}"
    docker compose logs --tail=20
    exit 1
fi

# 8. 测试 API
echo -e "${YELLOW}🧪 测试 API 连接...${NC}"
if curl -s http://localhost:5001/console/api/workspaces | grep -q "401\|unauthorized"; then
    echo -e "${GREEN}✅ API 运行正常${NC}"
else
    echo -e "${YELLOW}⚠️  API 可能未就绪，请稍后重试${NC}"
fi

# 9. 完成
echo -e "${GREEN}========== 部署完成！==========${NC}"
echo -e "访问地址："
echo -e "  - 安装向导: http://localhost/install"
echo -e "  - 应用首页: http://localhost/apps"
echo -e "${YELLOW}首次访问时，需要创建管理员账户${NC}"
```

---

## 🔧 docker-compose.yaml 关键检查

确保 `api` 服务中包含以下端口映射：

```yaml
api:
  image: langgenius/dify-api:1.11.2
  restart: always
  ports:
    - "0.0.0.0:5001:5001"  # ← 必须存在
  environment:
    <<: *shared-api-worker-env
    MODE: api
  # ... 其他配置
```

如果缺失 `ports` 段，需要手动添加，然后运行：

```bash
docker compose down && docker compose up -d
```

---

## 📋 配置对应关系表

| 部署场景 | CONSOLE_API_URL | CONSOLE_WEB_URL | COOKIE_DOMAIN | 备注 |
|---------|-----------------|-----------------|---------------|------|
| 本地 (localhost) | `http://localhost:5001` | `http://localhost` | `localhost` | 开发环境 |
| 单一域名 | `https://api.example.com` | `https://example.com` | `example.com` | 需 Nginx 反代 |
| 子域分离 | `https://api.example.com` | `https://console.example.com` | `.example.com` | 需 Nginx 反代 |

---

## ❌ 常见错误及修复

| 错误症状 | 原因 | 修复方法 |
|---------|------|--------|
| 前端白屏 | API 未映射端口 5001 | 检查 docker-compose.yaml，添加 `ports: ["0.0.0.0:5001:5001"]` |
| Connection refused | API 容器未运行 | 运行 `docker compose ps` 检查，`docker compose logs api` 查看错误 |
| 401 认证失败 | SECRET_KEY 为空或默认值 | 生成新 `SECRET_KEY=openssl rand -base64 42` |
| CORS 错误 | 域名配置不一致 | 检查 `CONSOLE_API_URL` 与 `CONSOLE_WEB_URL` 是否匹配 |
| 容器反复重启 | 数据库未就绪 | 等待 60 秒，运行 `docker compose restart` |

---

## ✅ 部署后验证清单

- [ ] `docker compose ps` 显示所有容器 "Up"
- [ ] `curl http://localhost:5001/console/api/workspaces` 返回 401
- [ ] 访问 `http://localhost/install` 页面加载成功
- [ ] 创建管理员账户无错误
- [ ] 登录后能访问 `/apps` 页面
- [ ] 前端可正常创建应用

---

**快速开始：3 个命令**

```bash
cp .env.example .env                    # 1. 创建配置
# 编辑 .env，修改 5 个关键配置项
docker compose up -d                    # 2. 启动服务
# 等待 30-60 秒
open http://localhost/install           # 3. 访问页面
```
