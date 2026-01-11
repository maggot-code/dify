# Dify 私有化部署文档索引

> **新用户快速导航**——按需选择阅读，5 分钟快速部署

## 🚀 按需要选择阅读

### 场景 1：我想在 5 分钟内快速启动（推荐新手）

**必读**：按顺序阅读

1. **[QUICK_CONFIG.md](./QUICK_CONFIG.md)** - 5 分钟快速指南
   - 包含 3 个场景的配置模板
   - 关键配置项说明
   - 一键部署脚本

2. **操作步骤**
   ```bash
   # 1. 创建 .env（选择场景 A、B 或 C）
   cp .env.template .env
   nano .env  # 修改 5 个关键配置项
   
   # 2. 启动
   docker compose up -d
   
   # 3. 访问
   open http://localhost/install
   ```

**预计用时**：5-10 分钟

---

### 场景 2：遇到问题，需要快速排查（遇到报错）

**必读**：[PROBLEM_SOLUTION_SUMMARY_CN.md](./PROBLEM_SOLUTION_SUMMARY_CN.md)

包含：
- ✅ 问题根本原因分析
- ✅ 关键解决方案（API 端口映射）
- ✅ 常见问题快速检查表
- ✅ 环境变量对应关系

**预计用时**：3-5 分钟定位问题，10-15 分钟解决

---

### 场景 3：需要完整的部署指南（想深入了解）

**必读**：[DEPLOYMENT_GUIDE_CN.md](./DEPLOYMENT_GUIDE_CN.md)

包含：
- 📋 概述和前置条件
- 🚀 快速启动 3 步骤
- ⚙️ 关键配置详解
- 🌐 Nginx 反向代理配置
- 🧪 完整验证清单
- 🔧 常见问题排查（详细版）
- 🎯 场景化部署示例
- 📝 配置文件清单

**预计用时**：30-60 分钟精读，或快速浏览后需要时查阅

---

### 场景 4：想要配置模板（直接套用）

**文件**：[.env.template](./.env.template)

包含：
- 3 个部署场景的完整配置块
- 所有必填项的注释说明
- 可直接复制使用的配置

**使用方法**
```bash
cp .env.example .env       # 创建基础配置
cat .env.template >> .env  # 或参考模板修改
nano .env                  # 编辑关键配置项
```

---

## 📊 文档结构对应表

| 文档 | 大小 | 适合人群 | 阅读时间 | 用途 |
|------|------|--------|--------|------|
| **QUICK_CONFIG.md** | 7.9K | 新手、想快速开始 | 5-10 分钟 | 快速部署、常见错误修复 |
| **PROBLEM_SOLUTION_SUMMARY_CN.md** | 11K | 遇到问题的用户 | 10-20 分钟 | 问题诊断、根本原因分析 |
| **DEPLOYMENT_GUIDE_CN.md** | 11K | 想了解细节的用户 | 30-60 分钟 | 完整指南、深入理解、生产部署 |
| **.env.template** | 5.7K | 所有用户 | 参考使用 | 配置模板、现成样例 |

---

## ⚡ 超快速开始（3 个命令）

适合有 Docker 经验的用户：

```bash
# 1. 创建配置（修改 5 个关键项）
cp .env.example .env && nano .env

# 2. 启动（修改好配置后）
docker compose up -d

# 3. 访问
open http://localhost/install
```

**必须修改的 5 项**：
- `CONSOLE_API_URL`
- `CONSOLE_WEB_URL`
- `APP_API_URL`
- `FILES_URL`
- `SECRET_KEY`

---

## 🎯 按问题类型查阅

### 前端连接问题
- 现象：白屏、加载中、Connection refused
- **查阅**：[QUICK_CONFIG.md - 常见错误](./QUICK_CONFIG.md#-常见错误及修复) 或 [PROBLEM_SOLUTION_SUMMARY_CN.md - 问题排查](./PROBLEM_SOLUTION_SUMMARY_CN.md#-常见问题快速检查表)
- **解决方案**：添加 API 端口映射到 `docker-compose.yaml`

### 认证/登录问题
- 现象：401 错误、无法登录
- **查阅**：[QUICK_CONFIG.md - 常见错误](./QUICK_CONFIG.md#-常见错误及修复) 中的 401 错误
- **解决方案**：生成新 SECRET_KEY，清空 Cookie 重试

### 域名/CORS 问题
- 现象：CORS 错误、跨域访问失败
- **查阅**：[PROBLEM_SOLUTION_SUMMARY_CN.md - CORS 错误](./PROBLEM_SOLUTION_SUMMARY_CN.md#问题cors-错误)
- **解决方案**：确保 `CONSOLE_API_URL` 和 `CONSOLE_WEB_URL` 一致

### 容器启动问题
- 现象：容器反复重启、无法启动
- **查阅**：[DEPLOYMENT_GUIDE_CN.md - 常见问题排查](./DEPLOYMENT_GUIDE_CN.md#常见问题排查)
- **解决方案**：查看 `docker compose logs api`，等待数据库就绪

---

## 🔑 5 个必填配置项说明

无论哪个部署场景，都必须修改这 5 项：

| 配置项 | 说明 | 本地示例 | 远程示例 |
|--------|------|--------|--------|
| **CONSOLE_API_URL** | 前端访问 API 的地址 | `http://localhost:5001` | `https://api.example.com` |
| **CONSOLE_WEB_URL** | 前端网站地址 | `http://localhost` | `https://example.com` |
| **APP_API_URL** | 应用 API 地址（同上） | `http://localhost:5001` | `https://api.example.com` |
| **FILES_URL** | 文件服务 URL（同上） | `http://localhost:5001` | `https://api.example.com` |
| **SECRET_KEY** | 加密密钥（必须生成） | `openssl rand -base64 42` | `openssl rand -base64 42` |

---

## 🔐 SECRET_KEY 生成

```bash
# 执行此命令
openssl rand -base64 42

# 复制输出到 .env 中
SECRET_KEY=生成的64个字符
```

**注意**：不要使用示例值或默认值！

---

## ✅ 部署完成验证

```bash
# 1. 容器都在运行
docker compose ps

# 2. API 可访问
curl -i http://localhost:5001/console/api/workspaces

# 3. 前端加载成功
open http://localhost/install

# 4. 创建管理员账户
# （在浏览器中完成）

# 5. 登录并访问应用
open http://localhost/apps
```

---

## 🛠️ 常用命令速查

```bash
# 启动所有服务
docker compose up -d

# 停止所有服务
docker compose down

# 查看容器状态
docker compose ps

# 查看 API 日志
docker compose logs -f api

# 重启 API 服务
docker compose restart api

# 完全重建（清除所有数据）
docker compose down -v && docker compose up -d

# 进入 API 容器
docker compose exec api bash

# 查看 SECRET_KEY 是否设置
grep "^SECRET_KEY=" .env
```

---

## 📁 关键文件位置

```
docker/
├── .env                           # 实际配置文件（自动生成，修改此文件）
├── .env.example                   # 官方示例（参考用）
├── .env.template                  # 快速模板（推荐复制用）
├── docker-compose.yaml            # 容器编排（检查 API 端口映射）
├── QUICK_CONFIG.md                # ⭐ 快速配置指南
├── PROBLEM_SOLUTION_SUMMARY_CN.md # ⭐ 问题解决总结
├── DEPLOYMENT_GUIDE_CN.md         # ⭐ 完整部署指南
├── nginx/conf.d/default.conf      # Nginx 配置（域名部署时修改）
├── certbot/                       # SSL 证书（HTTPS 时使用）
└── volumes/                       # 数据存储（数据库和文件）
```

---

## 🎯 推荐阅读路径

### 路径 A：想要快速启动（5-10 分钟）
```
开始
  ↓
阅读：QUICK_CONFIG.md（5 分钟）
  ↓
操作：修改 .env，运行 docker compose up -d（5 分钟）
  ↓
验证：访问 http://localhost/install
  ↓
完成！
```

### 路径 B：遇到问题需要排查（10-20 分钟）
```
遇到问题
  ↓
阅读：PROBLEM_SOLUTION_SUMMARY_CN.md（10 分钟）
  ↓
对号入座找对应问题
  ↓
按解决步骤修改和重启
  ↓
问题解决！
```

### 路径 C：想要完全理解（30-60 分钟）
```
想深入了解
  ↓
快速读：QUICK_CONFIG.md（5 分钟）
  ↓
精读：DEPLOYMENT_GUIDE_CN.md（25 分钟）
  ↓
参考：.env.template（10 分钟）
  ↓
实践：按步骤部署（10 分钟）
  ↓
完全掌握！
```

---

## 💡 核心概念

### Docker 端口映射

```
浏览器
  ↓
localhost:5001 (宿主机)
  ↓
0.0.0.0:5001->5001/tcp (映射)
  ↓
容器内 API:5001
```

**没有映射** = 浏览器无法访问容器内的 API

### 环境变量配置

```
CONSOLE_API_URL
  ↓
浏览器读取 → 发起 API 请求到这个地址
  ↓
必须指向 API 容器的可访问地址
```

**不一致** = CORS 错误或无法连接

---

## 📞 快速排查决策树

```
启动 Dify 后遇到问题？

├─ 前端无法加载（白屏）？
│  └─ 查看：QUICK_CONFIG.md → 常见错误 → 前端白屏
│     └─ 解决：检查 API 端口映射
│
├─ 登录失败（401）？
│  └─ 查看：QUICK_CONFIG.md → 常见错误 → 401 认证失败
│     └─ 解决：重新生成 SECRET_KEY
│
├─ CORS 错误？
│  └─ 查看：PROBLEM_SOLUTION_SUMMARY_CN.md → CORS 错误
│     └─ 解决：检查域名配置
│
├─ 容器反复重启？
│  └─ 查看：DEPLOYMENT_GUIDE_CN.md → 容器反复重启
│     └─ 解决：检查 docker compose logs
│
└─ 其他问题？
   └─ 查看：DEPLOYMENT_GUIDE_CN.md → 常见问题排查
      └─ 或 PROBLEM_SOLUTION_SUMMARY_CN.md
```

---

## 🚨 最常见的 3 个问题

### 问题 1：前端白屏（最常见）
**原因**：API 端口未映射  
**查阅**：[PROBLEM_SOLUTION_SUMMARY_CN.md - 第一个问题](./PROBLEM_SOLUTION_SUMMARY_CN.md#问题前端白屏或正在加载中)  
**解决**：修改 docker-compose.yaml，添加 `ports: ["0.0.0.0:5001:5001"]`

### 问题 2：SECRET_KEY 问题
**原因**：未生成新 KEY 或使用默认值  
**查阅**：[QUICK_CONFIG.md - 设置 SECRET_KEY](./QUICK_CONFIG.md#-第-2-步设置-secret_key必须修改)  
**解决**：`openssl rand -base64 42` 生成新 KEY

### 问题 3：域名配置不一致
**原因**：CONSOLE_API_URL 和 CONSOLE_WEB_URL 不匹配  
**查阅**：[PROBLEM_SOLUTION_SUMMARY_CN.md - CORS 错误](./PROBLEM_SOLUTION_SUMMARY_CN.md#问题cors-错误)  
**解决**：确保两者域名相同

---

## ✨ 最佳实践

✅ **做**：
- 修改 `SECRET_KEY`（使用 `openssl rand -base64 42` 生成）
- 根据部署环境修改 5 个关键配置项
- 先在本地测试（localhost），再在生产环境部署
- 定期备份 `volumes/` 目录

❌ **不要**：
- 使用示例或默认的 SECRET_KEY
- 跳过环境变量配置
- 直接在生产环境操作
- 删除 `docker-compose.yaml` 中的 `ports` 映射

---

## 📞 需要帮助？

1. **快速查阅**：本文档的索引表
2. **问题诊断**：查阅对应文档中的"常见问题"章节
3. **配置参考**：查看 `.env.template` 的对应场景
4. **深入理解**：阅读 `DEPLOYMENT_GUIDE_CN.md` 的对应章节

---

## 📋 文件对应速查表

| 我想... | 查阅文件 | 章节 |
|--------|---------|------|
| 快速部署 | QUICK_CONFIG.md | 开头的 3 个命令 |
| 了解问题根因 | PROBLEM_SOLUTION_SUMMARY_CN.md | 问题描述与根本原因 |
| 修复前端白屏 | QUICK_CONFIG.md | 常见错误及修复 |
| 配置 SECRET_KEY | PROBLEM_SOLUTION_SUMMARY_CN.md 或 QUICK_CONFIG.md | 对应章节 |
| 设置 Nginx 反代 | DEPLOYMENT_GUIDE_CN.md | Nginx 反向代理配置 |
| 了解架构 | PROBLEM_SOLUTION_SUMMARY_CN.md | 配置对应关系图 |
| 看配置示例 | .env.template | 3 个场景的完整配置 |

---

**现在你可以开始部署了！🚀**

选择你的路径：
- 🚀 [快速启动 → QUICK_CONFIG.md](./QUICK_CONFIG.md)
- 🔧 [遇到问题 → PROBLEM_SOLUTION_SUMMARY_CN.md](./PROBLEM_SOLUTION_SUMMARY_CN.md)
- 📚 [深入了解 → DEPLOYMENT_GUIDE_CN.md](./DEPLOYMENT_GUIDE_CN.md)
- 📝 [配置模板 → .env.template](./.env.template)

---

*最后更新：2025-01-11*  
*Dify v1.11.2*
