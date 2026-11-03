# QuantDinger 快速启动指南 (Quick Start)

> 本文档记录本项目的启动方式与访问方式，供日常开发与部署参考。
> 详细配置说明见 [DEVELOPMENT.md](DEVELOPMENT.md) 与 [README.md](README.md)。

## 一、启动方式

### 前置条件

| 工具 | 版本要求 |
|------|----------|
| Docker & Docker Compose | Docker 20+（本项目验证于 Docker 29.x / Compose v5.x） |
| Python | 3.10+（仅首次生成 `.env` 时需要） |

### 1. 首次启动（生成环境配置）

```bash
# 生成 .env 与随机本地凭据（若根目录已存在 .env 可跳过）
python scripts/bootstrap_env.py
```

然后编辑根目录 `.env`，确认以下关键项：

- `SECRET_KEY` — JWT 签名密钥（**不能**保留默认值，否则容器拒绝启动）
- `ADMIN_USER` / `ADMIN_PASSWORD` — 初始管理员账号密码
- `POSTGRES_PASSWORD` — 数据库密码（bootstrap 脚本已自动生成）

### 2. 启动全栈

```bash
# 校验 compose 配置（可选，推荐）
docker compose config --quiet

# 构建并后台启动全部服务（首次运行需构建后端镜像，耗时几分钟）
docker compose up -d --build
```

### 3. 查看状态

```bash
# 查看容器运行状态（核心服务应显示 healthy）
docker compose ps

# 查看某个服务的日志（例如后端）
docker compose logs -f backend

# 跟踪全部日志
docker compose logs -f
```

服务全部就绪的标志：`quantdinger-frontend`、`quantdinger-backend` 及各 worker 显示 `(healthy)`；
一次性任务 `quantdinger-migration`、`quantdinger-kafka-init`、`quantdinger-env-sync` 正常退出（Exited (0)）。

### 4. 停止 / 重启

```bash
# 停止全部服务（保留数据卷）
docker compose down

# 停止并删除数据卷（⚠️ 会清空数据库等持久化数据，慎用）
docker compose down -v

# 重启单个服务
docker compose restart backend

# 拉取并更新前端/移动端镜像（二者默认从 GHCR 拉取预构建镜像）
docker compose pull frontend mobile
docker compose up -d frontend mobile
```

## 二、访问方式

启动完成后，通过以下地址访问（默认全部绑定在 `127.0.0.1` 回环地址，仅本机可访问）：

| 入口 | 地址 | 说明 |
|------|------|------|
| 💻 Web 控制台 | http://localhost:8888 | 主前端（Vue SPA，Nginx 托管） |
| 📱 移动端 H5 | http://localhost:8889 | 触屏优化的移动 Web 客户端 |
| 🔌 后端 API | http://localhost:5000 | Flask API（gunicorn） |
| ❤️ 健康检查 | http://localhost:5000/api/health | 返回 `{"status":"healthy"}` 即正常 |

**登录凭据**：使用 `.env` 中 `ADMIN_USER` / `ADMIN_PASSWORD` 配置的管理员账号登录 Web 控制台。

### 基础设施端口（调试用）

| 服务 | 地址 | 说明 |
|------|------|------|
| PostgreSQL | `127.0.0.1:5432` | 持久化主库（库名/用户/密码见 `.env`） |
| Redis（缓存） | `127.0.0.1:6379` | 可淘汰缓存层 |
| Redis（任务队列） | 仅容器网络内 | Celery broker，未映射到宿主机 |
| Kafka | `127.0.0.1:29092` | 事件总线宿主机入口（容器内为 `kafka:9092`） |

### 快速验证

```bash
# 后端健康检查
curl http://localhost:5000/api/health

# 前端页面
curl -I http://localhost:8888/

# 移动端页面
curl -I http://localhost:8889/
```

## 三、服务清单

| 服务 | 容器名 | 角色 |
|------|--------|------|
| frontend | quantdinger-frontend | Web 前端 (Nginx, :8888) |
| mobile | quantdinger-mobile | 移动端 H5 (Nginx, :8889) |
| backend | quantdinger-backend | Flask API (:5000) |
| postgres | quantdinger-db | PostgreSQL 18 主库 |
| redis | quantdinger-redis | 缓存 Redis |
| redis-jobs | quantdinger-redis-jobs | Celery 任务队列 Redis（持久化） |
| kafka | quantdinger-kafka | Kafka 事件总线 |
| migration | quantdinger-migration | 一次性：数据库迁移（启动时执行） |
| kafka-init | quantdinger-kafka-init | 一次性：创建 Kafka topic |
| env-sync | quantdinger-env-sync | 一次性：同步环境配置 |
| trading-worker | quantdinger-trading-worker-1 | 交易执行 worker |
| strategy-dispatcher-worker | quantdinger-strategy-dispatcher-worker-1 | 策略分发 worker |
| strategy-evaluator-worker | quantdinger-strategy-evaluator-worker-1 | 策略评估 worker |
| scheduler-worker | quantdinger-scheduler-worker | 定时调度 worker |
| kafka-audit-worker | quantdinger-kafka-audit-worker | Kafka 审计 worker |
| celery-worker / celery-beat | quantdinger-celery-worker / -beat | Celery 异步任务 |

> 可选 MCP 网关默认未启用，启用方式见 [docker-compose.yml](docker-compose.yml) 中 `mcp` 服务的注释。
