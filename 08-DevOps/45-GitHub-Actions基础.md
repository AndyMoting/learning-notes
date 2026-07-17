# 45-GitHub Actions 基础
> 课时：60 min | 难度：★★★

## 学习目标
- 理解 CI/CD 核心概念与 GitHub Actions 的定位
- 掌握 Workflow 文件的结构与语法规则
- 能编写可运行的测试自动化工作流
- 掌握环境变量、密钥管理与 Actions 复用机制

## 核心概念

### CI/CD 概念

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Code    │ →  │  Build   │ →  │   Test   │ →  │ Deploy   │
│  Commit  │    │  编译构建  │    │  运行测试  │    │  部署上线  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
     │                │               │               │
     └──── CI ────────┘               └──── CD ───────┘
     Continuous Integration         Continuous Delivery/Deployment
     （持续集成：代码合并+验证）        （持续交付/部署：自动发布）
```

**GitHub Actions 定位：**
GitHub 原生提供的 CI/CD 平台，通过 YAML 文件定义工作流，在事件触发时自动执行。与 GitHub 仓库深度集成，无需额外服务器（使用 GitHub-hosted runner）。

### Workflow 文件结构

```yaml
# 文件位置：.github/workflows/<name>.yml

name: Workflow 名称

on:                    # 触发条件
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:                  # 任务集合
  job-name:            # 任务 ID
    runs-on: ubuntu-latest    # 运行环境
    steps:             # 执行步骤
      - name: 步骤名称
        uses: action@v1         # 引用 Action
        with:                   # 传入参数
          key: value
      - name: 执行命令
        run: |
          command1
          command2
```

### 触发器 (on)

```yaml
# Push 触发
on:
  push:
    branches:
      - main
      - 'feature/**'       # 通配符匹配
    paths:
      - 'src/**'           # 仅当 src/ 目录有变更时触发
      - '!src/docs/**'     # 排除 docs

# PR 触发
on:
  pull_request:
    types: [opened, synchronize, reopened]

# 定时触发（Cron 表达式）
on:
  schedule:
    - cron: '0 2 * * 1-5'  # 每工作日凌晨 2 点

# 手动触发
on:
  workflow_dispatch:
    inputs:
      env:
        description: '部署环境'
        required: true
        type: choice
        options: [staging, production]
```

### Runner 类型

| 类型 | 规格 | 适用场景 |
|------|------|----------|
| `ubuntu-latest` | Linux，免费额度充足 | 大多数项目 |
| `windows-latest` | Windows Server | .NET/Windows 项目 |
| `macOS-latest` | macOS | iOS/macOS 项目 |
| `self-hosted` | 自建服务器 | 需要特殊硬件/内网访问 |

### 核心 Actions 库

| Action | 用途 | 示例 |
|--------|------|------|
| `actions/checkout` | 检出代码 | `uses: actions/checkout@v4` |
| `actions/setup-python` | 配置 Python 环境 | `uses: actions/setup-python@v5` |
| `actions/setup-node` | 配置 Node.js 环境 | `uses: actions/setup-node@v4` |
| `actions/cache` | 缓存依赖 | `uses: actions/cache@v4` |
| `actions/upload-artifact` | 上传产物 | `uses: actions/upload-artifact@v4` |
| `actions/download-artifact` | 下载产物 | `uses: actions/download-artifact@v4` |

### 环境变量与 Secrets

```yaml
env:                          # 工作流级别变量
  PYTHON_VERSION: '3.11'
  TEST_ENV: 'ci'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: 使用变量
        run: echo "Python 版本 $PYTHON_VERSION"

      - name: 使用 Secrets（在仓库 Settings → Secrets 配置）
        env:
          API_KEY: ${{ secrets.API_KEY }}
          DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
        run: |
          pytest --api-key="$API_KEY"
```

**Secrets 配置位置：**
GitHub 仓库 → Settings → Secrets and variables → Actions → New repository secret

## 动手实操

### 实操 1：基础测试工作流

```yaml
# .github/workflows/test.yml

name: Python Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11', '3.12']

    steps:
      - name: 检出代码
        uses: actions/checkout@v4

      - name: 设置 Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: 缓存 pip 依赖
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-

      - name: 安装依赖
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: 运行测试
        run: |
          pytest tests/ -v --tb=short --junitxml=reports/junit.xml

      - name: 上传测试报告
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-reports-py${{ matrix.python-version }}
          path: reports/
          retention-days: 30
```

### 实操 2：多任务依赖工作流

```yaml
# .github/workflows/ci.yml

name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - name: 安装检查工具
        run: pip install flake8 black isort
      - name: 运行代码检查
        run: |
          flake8 src/ tests/
          black --check src/ tests/
          isort --check src/ tests/

  unit-test:
    needs: lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - name: 安装依赖
        run: pip install -r requirements.txt
      - name: 运行单元测试
        run: pytest tests/unit/ -v --cov=src --cov-report=xml

  integration-test:
    needs: unit-test
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: test_db
          POSTGRES_PASSWORD: test_pass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    env:
      DATABASE_URL: postgresql://postgres:test_pass@localhost:5432/test_db
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - name: 安装依赖
        run: pip install -r requirements.txt
      - name: 运行集成测试
        run: pytest tests/integration/ -v
```

### 实操 3：定时测试与通知

```yaml
# .github/workflows/scheduled-test.yml

name: Nightly Regression

on:
  schedule:
    - cron: '0 2 * * *'  # 每天凌晨 2 点
  workflow_dispatch:       # 支持手动触发

jobs:
  regression:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - name: 安装依赖
        run: pip install -r requirements.txt
      - name: 运行回归测试
        run: pytest tests/ -m regression -v --tb=short
        continue-on-error: true
        id: test

      - name: 发送通知
        if: failure()
        run: |
          curl -X POST "${{ secrets.DINGTALK_WEBHOOK }}" \
            -H 'Content-Type: application/json' \
            -d '{
              "msgtype": "text",
              "text": {
                "content": "回归测试失败：${{ github.repository }}\n分支：${{ github.ref }}\n链接：${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
              }
            }'
        env:
          DINGTALK_WEBHOOK: ${{ secrets.DINGTALK_WEBHOOK }}
```

### 实操 4：Matrix 策略测试多版本

```yaml
# .github/workflows/matrix-test.yml

name: Compatibility Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.9', '3.10', '3.11']
        exclude:
          - os: macos-latest
            python-version: '3.9'  # 排除不常用组合

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - name: 安装依赖
        run: pip install -r requirements.txt
      - name: 运行测试
        run: pytest tests/ -v
      - name: 上传报告
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: report-${{ matrix.os }}-py${{ matrix.python-version }}
          path: reports/
```

## 常见坑

- Workflow 文件必须放在 `.github/workflows/` 目录下，否则不会生效
- `run` 命令中的 `|` 需注意 YAML 缩进，每行命令对齐到同一层级
- Secrets 在工作流日志中会自动脱敏，但不要在 `echo` 中直接打印 Secrets 值
- `needs` 指定的依赖任务失败时，当前任务默认被跳过（不是失败）
- GitHub-hosted runner 的 `ubuntu-latest` 指向最新的 LTS 版本，关注官方公告避免版本变更导致测试失败
- Artifact 上传有时限：免费账户存储 90 天，单文件最大 10GB
- Cron 定时触发仅对默认分支生效，其他分支的 workflow 不会被定时触发

## 自测清单

- [ ] 能区分 CI 与 CD 的概念及各自的目标
- [ ] 能描述 Workflow 文件中 name/on/jobs/steps 的作用
- [ ] 能编写包含 checkout/setup/run 的基础工作流
- [ ] 理解 Runner 类型及其选择依据
- [ ] 能配置 Secrets 并在 Workflow 中安全引用
- [ ] 能使用 strategy/matrix 实现多版本并行测试
- [ ] 能使用 needs 定义任务间的依赖关系

## 延伸阅读

- GitHub Actions 官方文档：https://docs.github.com/en/actions
- GitHub Actions 语法参考：https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions
- Awesome Actions（工具集）：https://github.com/sdras/awesome-actions
- GitHub Actions Marketplace：https://github.com/marketplace?type=actions
- GitHub-hosted runner 规格：https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners
