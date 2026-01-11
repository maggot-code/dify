# Dify混合部署计划 📋

## 📚 文档目录

本目录包含针对您的特定环境和需求的完整部署计划。

### 文件清单

| 文件 | 大小 | 用途 | 何时阅读 |
|------|------|------|---------|
| **dify-hybrid-deployment-plan.md** | 23KB | 📖 完整、详细的部署指南 | 首次部署或需要深入理解 |
| **quick-reference.md** | 8KB | ⚡ 快速查询和命令参考 | 日常开发和快速查找 |
| **README.md** | 本文件 | 🗺️ 计划导航和概述 | 了解整体结构 |

---

## 🎯 您的部署场景

```
┌─ 环境配置 ─────────────────────────────────────┐
│  • OS: macOS                                    │
│  • 虚拟化: OrbStack (Docker)                    │
│  • Python管理: pyenv                            │
│  • 包管理: uv                                   │
│  • 数据卷位置: /Users/codemaggot/dev/dify      │
└────────────────────────────────────────────────┘

┌─ 部署架构 ─────────────────────────────────────┐
│  本地源代码启动:                                │
│  • Flask后端 (HTTP 5001)                       │
│  • Celery Worker (异步任务)                     │
│  • Celery Beat (定时任务)                       │
│                                                 │
│  容器化中间件:                                  │
│  • PostgreSQL 15 (Port 5432)                   │
│  • Redis 6 (Port 6379)                         │
│  • Weaviate (Port 8080)                        │
│                                                 │
│  暂不部署:                                      │
│  • 前端 (Next.js)                              │
│  • Nginx反向代理                                │
└────────────────────────────────────────────────┘
```

---

## 📖 快速导航

### 🚀 快速开始（5分钟）
👉 **文件**: `quick-reference.md` → "⚡ 快速启动（5分钟版）"

包含:
- 前提条件检查清单
- 中间件启动步骤
- 后端启动步骤
- 验证命令

### 📚 完整部署指南（首次部署）
👉 **文件**: `dify-hybrid-deployment-plan.md`

分为四个主要阶段:
1. **环境验证与准备** (15分钟)
   - Python版本管理
   - uv包管理器
   - OrbStack和Docker
   - 项目结构验证

2. **中间件容器部署** (10分钟)
   - Docker Compose配置
   - 数据卷设置
   - 容器启动和验证

3. **本地后端源代码启动** (20分钟)
   - 环境文件配置
   - 依赖安装
   - 数据库迁移
   - 服务启动

4. **前端和其他服务** (可选)
   - 容器化前端部署
   - 本地前端开发

### 🆘 故障排除
👉 **文件**: `dify-hybrid-deployment-plan.md` → "🔧 故障排除指南"

涵盖最常见的5个问题:
- PostgreSQL连接失败
- Redis连接失败
- Celery Worker无法启动
- 数据库迁移失败
- 端口被占用

### 📋 日常快速命令
👉 **文件**: `quick-reference.md` → "🔄 日常操作"

常用操作:
- 启动/停止服务
- 查看服务状态
- 常见问题速查
- 开发技巧

---

## ✅ 验证清单

部署完成后，检查以下项目：

- [ ] 所有前提条件已满足（Python 3.11+, uv, Docker, OrbStack）
- [ ] Docker中间件容器运行中（PostgreSQL, Redis, Weaviate）
- [ ] 后端.env文件已配置
- [ ] 数据库迁移完成
- [ ] Flask开发服务器运行在 http://localhost:5001
- [ ] 连接测试通过（Redis, PostgreSQL, Weaviate）

---

## 🔑 关键要点

### 1. 环境变量同步 💥
**中间件配置** (`middleware.env`) 和 **后端配置** (`api/.env`) 中以下变量**必须相同**:
- `DB_PASSWORD`
- `DB_USERNAME`
- `DB_DATABASE`
- `REDIS_PASSWORD`

### 2. 主机名配置 🌐
在 `api/.env` 中，数据库和Redis的主机名**必须使用** `localhost`:
```
DB_HOST=localhost      # 不要用 127.0.0.1 或容器名
REDIS_HOST=localhost   # 不要用 127.0.0.1 或容器名
```

### 3. 数据卷位置 💾
所有数据卷都映射到 `/Users/codemaggot/dev/dify/`:
```
/Users/codemaggot/dev/dify/
├── postgresql/    ← PostgreSQL数据
├── redis/         ← Redis数据
└── weaviate/      ← Weaviate数据
```

### 4. 安全考虑 🔐
- 使用强密码（建议20+字符）
- 不要将`.env`文件提交到版本控制
- 定期备份数据卷目录

---

## 📊 部署时间预估

| 阶段 | 首次 | 后续 |
|------|------|------|
| 环境验证 | 15分钟 | 2分钟 |
| 中间件部署 | 10分钟 | 1分钟 |
| 后端启动 | 20分钟 | 2分钟 |
| **总计** | **45分钟** | **5分钟** |

---

## 🚀 推荐工作流

### 第一次部署
1. 阅读 `dify-hybrid-deployment-plan.md` 的前两个阶段（30分钟）
2. 按步骤执行环境验证和中间件部署
3. 按步骤执行后端启动
4. 使用验证清单确认所有功能正常

### 日常开发
1. 参考 `quick-reference.md` 的"日常操作"部分
2. 在3个终端中启动：中间件、Flask、Worker（如需要）
3. 使用"快速命令参考"查询常用命令

### 遇到问题
1. 查看 `quick-reference.md` 的"常见问题速查"
2. 查看 `dify-hybrid-deployment-plan.md` 的"故障排除指南"
3. 查看日志：Flask控制台输出 + Docker日志

---

## 🔗 外部资源

- 📖 **官方文档**: https://docs.dify.ai
- 🐛 **Issue Tracker**: https://github.com/langgenius/dify/issues
- 💬 **Discord社区**: https://discord.gg/FngNHpbcY7
- ❓ **FAQ**: https://docs.dify.ai/getting-started/install-self-hosted/faqs

---

## 📝 文档维护

**版本**: 1.0  
**创建日期**: 2026年1月11日  
**最后更新**: 2026年1月11日  
**维护者**: 开发团队

### 何时需要更新此文档
- Dify版本升级（检查breaking changes）
- Docker Compose配置变更
- 依赖版本更新
- 发现新的常见问题

---

## 💡 提示

### 对于首次使用者
- 从 `quick-reference.md` 开始，快速了解概况
- 然后阅读 `dify-hybrid-deployment-plan.md` 的相关章节
- 边读边做，不要跳过任何步骤

### 对于经验丰富的开发者
- 直接使用 `quick-reference.md`
- 需要详细信息时参考主计划文档
- 利用故障排除指南快速解决问题

### 保存此文档
- 在您的IDE中添加书签
- 打印快速参考卡供离线使用
- 在第一次部署后，您可能不需要经常查阅

---

祝您部署顺利！ 🎉

如有任何问题或建议，欢迎提出。
