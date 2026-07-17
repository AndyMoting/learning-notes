# 47-Docker 基础
> 课时：75 min | 难度：★★★

## 学习目标
- 理解 Docker 核心概念：镜像、容器、仓库、卷、网络
- 掌握 Dockerfile 编写规范与常用指令
- 熟练使用 Docker CLI 执行容器生命周期管理
- 能使用 Docker Compose 编排多容器应用

## 核心概念

### Docker 架构

```
┌─────────────────────────────────────────────┐
│              Docker Client                   │
│              (docker CLI)                    │
└──────────────────┬──────────────────────────┘
                   │ REST API
                   ▼
┌─────────────────────────────────────────────┐
│              Docker Daemon                   │
│            (dockerd 守护进程)                 │
│  ┌─────────┐ ┌─────────┐ ┌──────────┐      │
│  │ Images  │ │Containers│ │ Volumes  │      │
│  │  镜像    │ │  容器     │ │  数据卷   │      │
│  └─────────┘ └─────────┘ └──────────┘      │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────┐
│           Container Registry                │
│    (Docker Hub / GHCR / 私有仓库)            │
└─────────────────────────────────────────────┘
```

### 核心概念关系

| 概念 | 说明 | 类比 |
|------|------|------|
| **Image（镜像）** | 只读模板，包含应用运行所需的一切 | 类的定义 |
| **Container（容器）** | 镜像的运行实例，可读写 | 类的实例 |
| **Registry（仓库）** | 镜像存储与分发服务 | Maven 中央仓库 |
| **Volume（数据卷）** | 容器持久化数据存储机制 | 外挂硬盘 |
| **Network（网络）** | 容器间通信的虚拟网络 | 虚拟交换机 |
| **Dockerfile** | 定义镜像构建步骤的文本文件 | POM 文件 |

### Dockerfile 指令详解

```dockerfile
# ===== 基础镜像 =====
FROM python:3.11-slim AS builder    # 指定基础镜像，AS 定义构建阶段别名

# ===== 元数据 =====
LABEL maintainer="team@example.com"
LABEL version="1.0"
LABEL description="ShopTest 测试服务"

# ===== 环境变量 =====
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1

# ===== 工作目录 =====
WORKDIR /app                         # 设置后续命令的工作目录

# ===== 复制依赖文件（利用缓存层）=====
COPY requirements.txt .

# ===== 安装依赖 =====
RUN pip install --no-cache-dir -r requirements.txt

# ===== 复制应用代码 =====
COPY src/ ./src/
COPY tests/ ./tests/

# ===== 暴露端口 =====
EXPOSE 8000                          # 声明容器监听端口（不自动映射）

# ===== 健康检查 =====
HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

# ===== 启动命令 =====
CMD ["python", "-m", "uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**Dockerfile 指令速查：**

| 指令 | 说明 | 使用注意 |
|------|------|----------|
| `FROM` | 指定基础镜像 | 优先选择 `alpine`/`slim` 变体减小体积 |
| `WORKDIR` | 设置工作目录 | 替代 `RUN cd /path`，后续命令基于此目录 |
| `COPY` | 从构建上下文复制文件 | 不支持构建上下文之外的文件 |
| `ADD` | 复制文件（支持 URL/自动解压） | 除非需要解压/URL，否则优先用 COPY |
| `RUN` | 构建时执行命令 | 多命令用 `&&` 连接减少层数 |
| `ENV` | 设置环境变量 | 容器运行时也可通过 `-e` 覆盖 |
| `ARG` | 构建时参数 | 仅在构建阶段有效，运行时不保留 |
| `EXPOSE` | 声明监听端口 | 实际映射需通过 `-p` 参数 |
| `CMD` | 容器启动默认命令 | 可被 `docker run` 后的参数覆盖 |
| `ENTRYPOINT` | 容器入口点 | 不可被覆盖，与 CMD 配合使用 |
| `VOLUME` | 声明数据卷挂载点 | 避免在容器内存储持久化数据 |
| `USER` | 指定运行用户 | 生产环境必须用非 root 用户 |

**CMD vs ENTRYPOINT 区别：**
```dockerfile
# CMD：可被 docker run 参数覆盖
CMD ["echo", "hello"]
# docker run image echo world → 输出 world

# ENTRYPOINT：不可被覆盖，CMD 作为默认参数
ENTRYPOINT ["echo"]
CMD ["hello"]
# docker run image → 输出 hello
# docker run image world → 输出 world
```

### Docker CLI 常用命令

**镜像操作：**
```bash
docker build -t myapp:1.0 .                    # 构建镜像
docker build -t myapp:1.0 -f Dockerfile.test .  # 指定 Dockerfile
docker images                                   # 查看本地镜像
docker pull python:3.11-slim                    # 拉取镜像
docker push myrepo/myapp:1.0                    # 推送镜像
docker tag myapp:1.0 myrepo/myapp:latest        # 打标签
docker rmi myapp:1.0                            # 删除镜像
docker image prune -a                           # 清理悬空镜像
docker history myapp:1.0                        # 查看镜像构建历史
docker inspect myapp:1.0                        # 查看镜像详细信息
```

**容器操作：**
```bash
docker run -d --name myapp -p 8000:8000 myapp:1.0     # 后台运行
docker run -it --rm myapp:1.0 bash                     # 交互式运行（退出后删除）
docker run -d --name myapp --restart=always myapp:1.0  # 自动重启
docker run -d -e DATABASE_URL=xxx myapp:1.0           # 传入环境变量
docker run -d -v /host/data:/app/data myapp:1.0        # 挂载数据卷
docker run -d --network=my-net myapp:1.0              # 指定网络

docker ps                               # 查看运行中容器
docker ps -a                            # 查看包含已停止的容器
docker stop myapp                       # 停止容器
docker start myapp                      # 启动容器
docker restart myapp                    # 重启容器
docker rm myapp                         # 删除容器
docker rm -f myapp                      # 强制删除运行中容器
docker logs myapp                       # 查看日志
docker logs -f myapp                    # 实时跟踪日志
docker logs --tail=100 myapp            # 查看最近 100 行
docker exec -it myapp bash              # 进入运行中容器
docker exec myapp ls /app                # 在容器内执行命令
docker top myapp                        # 查看容器内进程
docker stats                            # 查看资源使用统计
docker container prune                  # 清理已停止容器
```

**数据卷操作：**
```bash
docker volume create my-data             # 创建数据卷
docker volume ls                         # 列出数据卷
docker volume inspect my-data            # 查看卷详情
docker volume rm my-data                 # 删除数据卷
docker volume prune                      # 清理未使用卷

# 挂载方式
docker run -v my-data:/app/data          # 命名卷挂载
docker run -v /host/path:/app/data       # 绑定挂载（宿主机路径）
docker run -v /app/data                  # 匿名卷挂载
```

**网络操作：**
```bash
docker network create my-net             # 创建网络
docker network ls                        # 列出网络
docker network inspect my-net            # 查看网络详情
docker network connect my-net myapp     # 连接容器到网络
docker network disconnect my-net myapp  # 断开网络
docker network rm my-net                 # 删除网络
```

### Docker Compose

**docker-compose.yml 结构：**
```yaml
version: "3.9"

services:                    # 服务定义
  web:                       # 服务名称
    build:                   # 构建配置
      context: .
      dockerfile: Dockerfile
    image: myapp:1.0         # 或使用预构建镜像
    container_name: my-app   # 容器名称
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/app_db
    env_file:
      - .env                 # 从文件加载环境变量
    volumes:
      - ./src:/app/src       # 开发时挂载源码
      - app-data:/app/data   # 持久化数据卷
    depends_on:
      db:
        condition: service_healthy  # 等待 db 健康后再启动
    restart: unless-stopped
    networks:
      - app-network

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: app_db
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - db-data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user -d app_db"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network

volumes:                     # 数据卷声明
  app-data:
  db-data:

networks:                    # 网络声明
  app-network:
    driver: bridge
```

**Docker Compose CLI：**
```bash
docker-compose up -d                    # 启动所有服务（后台）
docker-compose up --build -d            # 重新构建并启动
docker-compose down                     # 停止并删除容器
docker-compose down -v                  # 同时删除数据卷
docker-compose logs -f                  # 查看日志
docker-compose logs -f web              # 查看指定服务日志
docker-compose ps                       # 查看服务状态
docker-compose exec web bash            # 进入服务容器
docker-compose run --rm test pytest     # 运行一次性任务
docker-compose build --no-cache         # 无缓存构建
docker-compose restart web              # 重启服务
docker-compose pull                     # 拉取最新镜像
```

## 动手实操

### 实操 1：构建 Python 测试项目镜像

```bash
# 项目结构
shop-test/
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
├── src/
│   └── __init__.py
└── tests/
    └── __init__.py
```

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安装系统依赖
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# 安装 Python 依赖（利用缓存层）
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 复制项目代码
COPY . .

# 非 root 用户运行
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

CMD ["pytest", "tests/", "-v"]
```

```bash
# 构建
docker build -t shop-test:1.0 .

# 运行测试
docker run --rm shop-test:1.0

# 传环境变量运行
docker run --rm -e TEST_ENV=staging shop-test:1.0

# 挂载本地代码（开发时实时更新）
docker run --rm -v $(pwd)/src:/app/src -v $(pwd)/tests:/app/tests shop-test:1.0
```

### 实操 2：多阶段构建（减小镜像体积）

```dockerfile
# 阶段 1：构建测试依赖
FROM python:3.11-slim AS builder

WORKDIR /build
COPY requirements.txt .
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# 阶段 2：运行环境
FROM python:3.11-slim

WORKDIR /app

# 从 builder 阶段复制已安装的依赖
COPY --from=builder /install /usr/local

COPY . .

USER nobody
CMD ["pytest", "tests/", "-v"]
```

### 实操 3：开发环境热重载

```yaml
# docker-compose.dev.yml
version: "3.9"

services:
  app:
    build: .
    volumes:
      - ./src:/app/src
      - ./tests:/app/tests
    environment:
      - DEBUG=1
      - LOG_LEVEL=debug
    command: ["pytest", "-f", "tests/", "-v"]  # -f 持续监听文件变化
    ports:
      - "8000:8000"
```

```bash
docker-compose -f docker-compose.dev.yml up
```

### 实操 4：调试容器

```bash
# 查看运行中容器的详细信息
docker inspect my-container

# 查看容器日志
docker logs --tail=50 -f my-container

# 进入容器排查问题
docker exec -it my-container bash

# 查看容器资源使用
docker stats my-container

# 查看容器端口映射
docker port my-container

# 从容器内复制文件到宿主机
docker cp my-container:/app/logs/error.log ./error.log

# 查看容器间网络连通性
docker exec my-container ping db
```

## 常见坑

- `docker build` 的构建上下文（末尾的 `.`）过大会导致构建缓慢，使用 `.dockerignore` 排除不必要文件
- Alpine 镜像体积小但部分 Python 包（如 `pandas`、`numpy`）需编译，构建变慢；权衡选择 `slim` 或 `alpine`
- 容器内不要存储持久化数据，数据卷或外部存储是唯一可靠方案
- `docker-compose down` 不会删除命名卷，需加 `-v` 参数；生产环境慎用 `-v`
- `depends_on` 仅保证容器启动顺序，不保证服务就绪；数据库场景必须配合 `healthcheck`
- 开发环境挂载源码到容器时，需注意宿主机与容器内的文件权限问题
- `docker system prune` 会清理所有未使用的镜像、容器、网络，执行前务必确认

## 自测清单

- [ ] 能解释镜像、容器、数据卷、网络四个核心概念
- [ ] 能编写包含 FROM/WORKDIR/COPY/RUN/CMD 的 Dockerfile
- [ ] 能使用 docker build/run/ps/logs/exec 完成容器生命周期管理
- [ ] 理解 CMD 与 ENTRYPOINT 的区别及组合用法
- [ ] 能编写包含多服务的 docker-compose.yml
- [ ] 能使用数据卷实现容器数据持久化
- [ ] 理解多阶段构建的目的与实现方式

## 延伸阅读

- Docker 官方文档：https://docs.docker.com/
- Dockerfile 最佳实践：https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
- Docker Compose 文档：https://docs.docker.com/compose/
- .dockerignore 参考：https://docs.docker.com/build/concepts/context/#dockerignore-files
- Docker 安全扫描：https://docs.docker.com/docker-hub/vulnerability-scanning/
- Kubernetes 入门（容器编排进阶）：https://kubernetes.io/docs/tutorials/kubernetes-basics/
