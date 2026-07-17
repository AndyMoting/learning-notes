# 48-Docker 测试环境编排
> 课时：90 min | 难度：★★★★

## 学习目标
- 掌握使用 docker-compose 编排完整的多容器测试环境
- 理解容器间网络通信与数据持久化配置
- 能将 Docker 测试环境集成到 CI/CD 流水线
- 能独立搭建 ShopTest 项目的完整测试环境

## 核心概念

### 测试环境架构

```
┌─────────────────────────────────────────────────────────────┐
│                     docker-compose.yml                       │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐     │
│  │   app        │  │   db         │  │  redis        │     │
│  │   应用服务    │  │  PostgreSQL  │  │  缓存服务      │     │
│  │   :8000      │  │  :5432       │  │  :6379        │     │
│  └──────┬───────┘  └──────┬───────┘  └──────┬────────┘     │
│         │                 │                  │               │
│         └─────────────────┼──────────────────┘               │
│                           │                                  │
│                    ┌──────┴───────┐                          │
│                    │ test-runner  │                          │
│                    │ 测试执行容器  │                          │
│                    └──────────────┘                          │
│                                                              │
│  Volumes:  db-data, redis-data, test-reports                │
│  Networks: test-network                                     │
└─────────────────────────────────────────────────────────────┘
```

### 测试环境设计原则

1. **环境一致性**：测试环境与生产环境使用相同的基础镜像与配置
2. **一键启停**：单条命令启动/停止整个测试环境
3. **数据隔离**：测试数据与生产数据完全分离
4. **可重复构建**：每次测试从干净环境开始，避免状态污染
5. **网络隔离**：测试网络独立于开发/生产网络

### 容器间通信

```yaml
# 同一网络内的容器可通过服务名互相访问
services:
  app:
    networks:
      - test-net
    # 访问数据库：使用服务名 db 作为主机名
    environment:
      DATABASE_URL: postgresql://user:pass@db:5432/test_db
      #               ↑ 服务名  ↑ 容器内端口

  db:
    networks:
      - test-net

networks:
  test-net:
    driver: bridge
```

### 数据持久化策略

| 策略 | 适用场景 | 配置方式 |
|------|----------|----------|
| 命名卷 (Named Volume) | 数据库数据持久化 | `volumes: - db-data:/var/lib/postgresql/data` |
| 绑定挂载 (Bind Mount) | 开发时源码同步 | `volumes: - ./src:/app/src` |
| 临时卷 (tmpfs) | 敏感数据不落盘 | `tmpfs: /tmp` |

## 动手实操

### 实操 1：ShopTest 完整测试环境

```yaml
# docker-compose.test.yml
version: "3.9"

services:
  # ========== 数据库 ==========
  postgres:
    image: postgres:15-alpine
    container_name: shoptest-postgres
    environment:
      POSTGRES_DB: shoptest_db
      POSTGRES_USER: test_user
      POSTGRES_PASSWORD: test_password
      POSTGRES_INITDB_ARGS: "--encoding=UTF-8"
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./sql/init.sql:/docker-entrypoint-initdb.d/01-init.sql
      - ./sql/seed.sql:/docker-entrypoint-initdb.d/02-seed.sql
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U test_user -d shoptest_db"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s
    networks:
      - shoptest-net
    restart: unless-stopped

  # ========== Redis 缓存 ==========
  redis:
    image: redis:7-alpine
    container_name: shoptest-redis
    command: redis-server --appendonly yes --requirepass redis_pass
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "redis_pass", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
    networks:
      - shoptest-net
    restart: unless-stopped

  # ========== 应用服务 ==========
  app:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: shoptest-app
    environment:
      DATABASE_URL: postgresql://test_user:test_password@postgres:5432/shoptest_db
      REDIS_URL: redis://:redis_pass@redis:6379/0
      SECRET_KEY: test-secret-key-not-for-production
      LOG_LEVEL: DEBUG
      ENVIRONMENT: testing
    ports:
      - "8000:8000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 15s
      timeout: 10s
      retries: 3
      start_period: 20s
    networks:
      - shoptest-net
    restart: unless-stopped

  # ========== 测试执行器 ==========
  test-runner:
    build:
      context: .
      dockerfile: Dockerfile.test
    container_name: shoptest-runner
    environment:
      BASE_URL: http://app:8000
      DATABASE_URL: postgresql://test_user:test_password@postgres:5432/shoptest_db
      REDIS_URL: redis://:redis_pass@redis:6379/0
      TEST_ENV: docker
      PYTEST_ADDOPTS: "-v --tb=short"
    volumes:
      - ./tests:/app/tests
      - ./reports:/app/reports
      - ./test_data:/app/test_data
    depends_on:
      app:
        condition: service_healthy
    networks:
      - shoptest-net
    # 测试完成后自动退出
    command: >
      pytest tests/
        --junitxml=reports/junit.xml
        --html=reports/report.html
        --self-contained-html
        --cov=src
        --cov-report=xml:reports/coverage.xml
        --cov-report=html:reports/htmlcov

volumes:
  postgres-data:
    driver: local
  redis-data:
    driver: local

networks:
  shoptest-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.28.0.0/16
```

### 实操 2：测试专用 Dockerfile

```dockerfile
# Dockerfile.test
FROM python:3.11-slim

WORKDIR /app

# 安装系统依赖
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# 安装 Python 依赖
COPY requirements.txt requirements-test.txt ./
RUN pip install --no-cache-dir -r requirements.txt -r requirements-test.txt

# 复制源码与测试代码
COPY src/ ./src/
COPY tests/ ./tests/
COPY test_data/ ./test_data/

# 创建报告目录
RUN mkdir -p reports

# 默认命令
CMD ["pytest", "tests/", "-v"]
```

### 实操 3：初始化 SQL 脚本

```sql
-- sql/init.sql
CREATE SCHEMA IF NOT EXISTS test_schema;

CREATE TABLE IF NOT EXISTS test_schema.users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) DEFAULT 'user',
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS test_schema.products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    stock INTEGER DEFAULT 0,
    category VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS test_schema.orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES test_schema.users(id),
    total_amount DECIMAL(10, 2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

```sql
-- sql/seed.sql
INSERT INTO test_schema.users (username, email, password_hash, role) VALUES
('admin', 'admin@shoptest.com', '$2b$12$hashed_admin_password', 'admin'),
('testuser1', 'user1@shoptest.com', '$2b$12$hashed_user1_password', 'user'),
('testuser2', 'user2@shoptest.com', '$2b$12$hashed_user2_password', 'user');

INSERT INTO test_schema.products (name, description, price, stock, category) VALUES
('测试商品A', '用于测试的商品A', 99.99, 100, '电子产品'),
('测试商品B', '用于测试的商品B', 199.50, 50, '电子产品'),
('测试商品C', '用于测试的商品C', 29.90, 200, '日用品'),
('测试商品D', '用于测试的商品D', 599.00, 10, '奢侈品');
```

### 实操 4：环境管理脚本

```bash
#!/bin/bash
# scripts/test-env.sh

set -e

COMPOSE_FILE="docker-compose.test.yml"
PROJECT_NAME="shoptest"

case "$1" in
  start)
    echo "启动测试环境..."
    docker-compose -f $COMPOSE_FILE -p $PROJECT_NAME up -d --build
    echo "等待服务就绪..."
    sleep 10
    echo "测试环境已启动"
    echo "  应用: http://localhost:8000"
    echo "  数据库: localhost:5432"
    echo "  Redis: localhost:6379"
    ;;

  stop)
    echo "停止测试环境..."
    docker-compose -f $COMPOSE_FILE -p $PROJECT_NAME down
    ;;

  restart)
    $0 stop
    $0 start
    ;;

  clean)
    echo "清理测试环境（含数据卷）..."
    docker-compose -f $COMPOSE_FILE -p $PROJECT_NAME down -v
    ;;

  test)
    echo "运行测试..."
    docker-compose -f $COMPOSE_FILE -p $PROJECT_NAME run --rm test-runner
    ;;

  logs)
    docker-compose -f $COMPOSE_FILE -p $PROJECT_NAME logs -f "${2:-}"
    ;;

  status)
    docker-compose -f $COMPOSE_FILE -p $PROJECT_NAME ps
    ;;

  shell)
    docker-compose -f $COMPOSE_FILE -p $PROJECT_NAME exec "${2:-app}" bash
    ;;

  *)
    echo "用法: $0 {start|stop|restart|clean|test|logs|status|shell}"
    exit 1
    ;;
esac
```

```bash
# 使用方式
chmod +x scripts/test-env.sh

./scripts/test-env.sh start      # 启动环境
./scripts/test-env.sh test       # 运行测试
./scripts/test-env.sh logs app   # 查看应用日志
./scripts/test-env.sh shell app  # 进入应用容器
./scripts/test-env.sh status     # 查看状态
./scripts/test-env.sh stop       # 停止环境
./scripts/test-env.sh clean      # 清理环境（含数据）
```

### 实操 5：CI/CD 集成

```yaml
# .github/workflows/docker-test.yml

name: Docker Test Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  docker-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: 设置 Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: 缓存 Docker 层
        uses: actions/cache@v4
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-buildx-

      - name: 构建并启动测试环境
        run: |
          docker-compose -f docker-compose.test.yml up -d --build

      - name: 等待服务就绪
        run: |
          timeout 120 bash -c '
            until curl -s http://localhost:8000/health; do
              echo "等待应用就绪..."
              sleep 5
            done
          '

      - name: 运行测试
        run: |
          docker-compose -f docker-compose.test.yml exec -T test-runner \
            pytest tests/ -v --junitxml=reports/junit.xml

      - name: 复制测试报告
        if: always()
        run: |
          docker cp shoptest-runner:/app/reports ./reports

      - name: 上传测试报告
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: docker-test-reports
          path: reports/

      - name: 发布测试结果到 PR
        if: github.event_name == 'pull_request' && always()
        uses: dorny/test-reporter@v1
        with:
          name: Docker 测试结果
          path: reports/junit.xml
          reporter: java-junit

      - name: 清理环境
        if: always()
        run: |
          docker-compose -f docker-compose.test.yml down -v
```

### 实操 6：多环境配置管理

```yaml
# docker-compose.override.yml（开发环境自动加载）
version: "3.9"

services:
  app:
    volumes:
      - ./src:/app/src    # 开发时挂载源码
    environment:
      DEBUG: "true"
      LOG_LEVEL: debug

  test-runner:
    volumes:
      - ./tests:/app/tests  # 开发时挂载测试代码
```

```yaml
# docker-compose.ci.yml（CI 环境专用）
version: "3.9"

services:
  app:
    environment:
      LOG_LEVEL: warning

  test-runner:
    environment:
      PYTEST_ADDOPTS: "-v --tb=line -q"
```

```bash
# 开发环境（自动加载 override）
docker-compose up -d

# CI 环境（指定多文件）
docker-compose -f docker-compose.test.yml -f docker-compose.ci.yml up -d
```

## 常见坑

- `depends_on` 仅保证启动顺序不保证服务就绪，数据库场景必须配合 `healthcheck` + `condition: service_healthy`
- 容器内应用连接数据库应使用服务名（如 `postgres`）而非 `localhost`，`localhost` 指向容器自身
- 数据卷挂载时注意宿主机路径权限，容器内用户可能无法访问宿主机文件
- `docker-compose down -v` 会删除所有命名卷数据，生产环境绝对禁止
- CI 环境中 Docker 构建建议使用 Buildx 缓存（`--cache-from`/`--cache-to`）加速
- 测试容器退出后默认被删除（`--rm`），如需保留日志需在退出前复制出来
- 多项目并行时注意容器名冲突，使用 `-p` 参数指定不同 project name

## 自测清单

- [ ] 能编写包含数据库、应用、测试执行器的 docker-compose.yml
- [ ] 理解容器间通过服务名通信的原理
- [ ] 能配置 healthcheck 确保服务依赖就绪
- [ ] 能使用命名卷实现数据库数据持久化
- [ ] 能将 Docker 测试环境集成到 GitHub Actions 流水线
- [ ] 能编写环境管理脚本实现一键启停
- [ ] 理解多环境配置（dev/test/ci）的覆盖与组合方式

## 延伸阅读

- Docker Compose 规范：https://docs.docker.com/compose/compose-file/
- Docker 网络详解：https://docs.docker.com/network/
- Docker 数据卷文档：https://docs.docker.com/storage/volumes/
- Docker 健康检查：https://docs.docker.com/engine/reference/builder/#healthcheck
- Testcontainers（代码中启动测试容器）：https://testcontainers.com/
- Docker 安全最佳实践：https://docs.docker.com/engine/security/
