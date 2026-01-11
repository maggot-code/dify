# 🚀 Dify 私有化部署 - 从这里开始

> **⏱️ 5分钟快速部署 | 新手友好 | 中文文档完整**

## 📋 三个快速选择

### 我想快速启动（5-10 分钟）
👉 **阅读** → [QUICK_CONFIG.md](./QUICK_CONFIG.md)

包含：配置模板、快速命令、常见错误修复

```bash
# 3 个命令快速启动
cp .env.example .env
# 编辑 .env（参考 QUICK_CONFIG.md 的模板修改 5 个关键项）
docker compose up -d
```

---

### 遇到问题需要排查
👉 **阅读** → [PROBLEM_SOLUTION_SUMMARY_CN.md](./PROBLEM_SOLUTION_SUMMARY_CN.md)

包含：问题根本原因、快速修复方案、排查清单

常见问题：
- ❌ 前端白屏 → API 端口未映射
- ❌ 401 错误 → SECRET_KEY 问题
- ❌ CORS 错误 → 域名配置不一致

---

### 想要完整了解
👉 **阅读** → [DEPLOYMENT_GUIDE_CN.md](./DEPLOYMENT_GUIDE_CN.md)

包含：完整指南、Nginx 反代、所有配置说明、场景化示例

---

## ⚡ 超快速 3 步启动

假设你有 Docker 经验，最快只需：

### 第 1 步：创建配置
```bash
cd docker/
cp .env.example .env
```

### 第 2 步：修改 5 个关键项（参考下表）
```bash
nano .env

# 修改这 5 个：
CONSOLE_API_URL=http://localhost:5001
CONSOLE_WEB_URL=http://localhost
APP_API_URL=http://localhost:5001
FILES_URL=http://localhost:5001
SECRET_KEY=<执行: openssl rand -base64 42>
```

### 第 3 步：启动
```bash
docker compose up -d
sleep 30
docker compose ps  # 验证所有容器 "Up"

# 访问
open http://localhost/install
```

**完成！** 创建管理员账户后，访问 `http://localhost/apps`

---

## 🔑 5 个必填配置项速查表

| 项目 | 本地值 | 说明 |
|------|--------|------|
| **CONSOLE_API_URL** | `http://localhost:5001` | 前端调用 API 的地址 |
| **CONSOLE_WEB_URL** | `http://localhost` | 前端网站地址 |
| **APP_API_URL** | `http://localhost:5001` | 应用 API 地址 |
| **FILES_URL** | `http://localhost:5001` | 文件服务地址 |
| **SECRET_KEY** | `openssl rand -base64 42` | 加密密钥（**必须生成新值**） |

---

## ❓ 常见问题快速解决

### Q1：前端白屏？
**A:** API 端口未映射
```bash
# 检查
docker compose ps | grep api
# 应显示：0.0.0.0:5001->5001/tcp

# 如果缺失，修改 docker-compose.yaml，在 api: 下添加：
ports:
  - "0.0.0.0:5001:5001"

# 然后重启
docker compose down && docker compose up -d
```

### Q2：401 认证错误？
**A:** SECRET_KEY 问题
```bash
# 生成新 KEY
openssl rand -base64 42

# 更新 .env
SECRET_KEY=<新生成的值>

# 重启
docker compose down && docker compose up -d
```

### Q3：容器无法启动？
**A:** 查看日志
```bash
docker compose logs api --tail=50
# 找出具体错误信息
```

---

## 📁 文档导航

| 文档 | 用途 | 阅读时间 |
|------|------|---------|
| **START_HERE.md** | 你在这里 👈 | 2 分钟 |
| **README_CN.md** | 文档索引和导航 | 3 分钟 |
| **QUICK_CONFIG.md** | ⭐ 快速启动（新手必读） | 5 分钟 |
| **PROBLEM_SOLUTION_SUMMARY_CN.md** | ⭐ 问题诊断和排查 | 10 分钟 |
| **DEPLOYMENT_GUIDE_CN.md** | 完整部署指南 | 30 分钟 |
| **.env.template** | 配置参考模板 | 参考使用 |

---

## ✅ 部署成功标志

```bash
# 1. 容器都在运行
docker compose ps | grep Up
# 应显示多个 Up

# 2. API 可访问
curl -i http://localhost:5001/console/api/workspaces
# 应返回 401（说明 API 正常）

# 3. 前端加载成功
open http://localhost/install
# 页面应能正常加载（不是白屏）

# 4. 能登录
# 创建管理员账户后，访问 http://localhost/apps
```

---

## 🎯 下一步

### 路线 A：快速体验
1. 按照"超快速 3 步启动"操作
2. 访问 http://localhost/install 创建账户
3. 体验 Dify 的功能

### 路线 B：遇到问题
1. 查阅"常见问题快速解决"
2. 查阅 [PROBLEM_SOLUTION_SUMMARY_CN.md](./PROBLEM_SOLUTION_SUMMARY_CN.md)
3. 按照步骤修复

### 路线 C：深入学习
1. 阅读 [README_CN.md](./README_CN.md) 了解文档体系
2. 精读 [DEPLOYMENT_GUIDE_CN.md](./DEPLOYMENT_GUIDE_CN.md)
3. 理解 Dify 的完整部署架构

---

## 💡 关键概念 30 秒速成

**Docker 端口映射是什么？**
```
浏览器访问：localhost:5001
         ↓
  Docker 映射：0.0.0.0:5001 → 5001（容器内）
         ↓
  API 服务：内部运行在容器的 5001 端口
```

没有映射 → 浏览器无法访问 → 白屏

**SECRET_KEY 是什么？**

用于加密用户会话和敏感数据。不生成新值会导致认证失败。

**INTERNAL_FILES_URL 为什么不要改？**

这是容器内部通信地址。Docker 会自动将 `api` 解析到容器名，改掉会导致容器间无法通信。

---

## 🔗 快速链接

- 📖 [QUICK_CONFIG.md](./QUICK_CONFIG.md) - 快速配置指南
- 🔍 [PROBLEM_SOLUTION_SUMMARY_CN.md](./PROBLEM_SOLUTION_SUMMARY_CN.md) - 问题诊断
- 📚 [DEPLOYMENT_GUIDE_CN.md](./DEPLOYMENT_GUIDE_CN.md) - 完整指南
- 📋 [README_CN.md](./README_CN.md) - 文档导航
- 🎁 [.env.template](./.env.template) - 配置模板

---

## 🎓 再需要什么？

| 需要 | 查阅 |
|------|------|
| 配置参考 | .env.template |
| 部署步骤 | QUICK_CONFIG.md |
| 问题排查 | PROBLEM_SOLUTION_SUMMARY_CN.md |
| 深入理解 | DEPLOYMENT_GUIDE_CN.md |
| 文档导航 | README_CN.md |

---

**准备好了？ → 打开 [QUICK_CONFIG.md](./QUICK_CONFIG.md) 开始部署！** 🚀

或者直接执行快速命令：
```bash
cd docker/ && cp .env.example .env && nano .env
# （修改 5 个关键项，参考上面的表格）
docker compose up -d && sleep 30 && docker compose ps
```

---

*需要帮助？遇到问题？查看 [PROBLEM_SOLUTION_SUMMARY_CN.md](./PROBLEM_SOLUTION_SUMMARY_CN.md)*

**祝部署顺利！** 🎉
