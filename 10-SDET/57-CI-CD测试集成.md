# 57-CI-CD测试集成

> 课时：50 min | 难度：★★★★☆

## 学习目标

- 理解 CI/CD 流水线中测试的集成方式
- 能够编写 GitHub Actions / Jenkins 流水线
- 掌握测试并行执行和失败重试策略
- 理解测试门禁（Quality Gate）的设计

## 核心概念

### CI/CD 中的测试分层

```
代码提交 → 构建 → 单元测试 → 集成测试 → 部署测试环境 → E2E 测试 → 生产部署
  ↓         ↓        ↓          ↓             ↓             ↓          ↓
Webhook   Compile  pytest     pytest+DB     Smoke Test    Full Suite   Canary
触发      打包     (快速反馈)  (中等耗时)    (部署验证)    (全面验证)   (灰度测试)
          ↓        ↓          ↓             ↓             ↓
        < 1min   1~3min     5~10min       3~5min       15~60min
```

### 测试门禁策略

| 门禁类型 | 阻断条件 | 阶段 |
|---------|---------|------|
| 单元测试门禁 | 通过率 < 100% | 代码合并前 |
| 覆盖率门禁 | 覆盖率下降 > 2% | 代码合并前 |
| E2E 门禁 | 通过率 < 95% | 部署前 |
| 性能门禁 | P99 延迟超过 SLA | 部署前 |
| 安全门禁 | 高危漏洞 > 0 | 部署前 |

## 动手实操

### 实操1：GitHub Actions 流水线

```yaml
# .github/workflows/test-pipeline.yml
name: Test Automation Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1-5'  # 工作日凌晨2点

env:
  PYTHON_VERSION: '3.11'
  ALLURE_VERSION: '2.27.0'

jobs:
  # ============ 阶段1: 代码质量检查 ============
  lint:
    name: Code Quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          
      - name: Cache pip
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('requirements.txt') }}
          
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install flake8 black isort mypy
          
      - name: Run black (format check)
        run: black --check --diff .
        
      - name: Run isort (import order)
        run: isort --check-only --diff .
        
      - name: Run flake8 (lint)
        run: flake8 . --count --statistics
        
      - name: Run mypy (type check)
        run: mypy src/ --ignore-missing-imports

  # ============ 阶段2: 单元测试 ============
  unit-tests:
    name: Unit Tests
    needs: lint
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.10', '3.11', '3.12']
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
          
      - name: Install dependencies
        run: pip install -r requirements.txt
        
      - name: Run unit tests
        run: |
          pytest tests/unit/ \
            -v \
            --cov=src \
            --cov-report=xml \
            --cov-report=html \
            --cov-fail-under=80 \
            -n auto \
            --junitxml=reports/unit-results.xml
            
      - name: Upload coverage
        uses: actions/upload-artifact@v4
        with:
          name: coverage-${{ matrix.python-version }}
          path: reports/
          
      - name: Upload to Codecov
        uses: codecov/codecov-action@v4
        with:
          file: reports/coverage.xml
          flags: unit

  # ============ 阶段3: 集成测试 ============
  integration-tests:
    name: Integration Tests
    needs: unit-tests
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: test_db
          POSTGRES_USER: test_user
          POSTGRES_PASSWORD: test_pass
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
          
      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          
      - name: Install dependencies
        run: pip install -r requirements.txt
        
      - name: Run integration tests
        env:
          DB_HOST: localhost
          DB_PORT: 5432
          DB_NAME: test_db
          DB_USER: test_user
          DB_PASSWORD: test_pass
          REDIS_URL: redis://localhost:6379
        run: |
          pytest tests/integration/ \
            -v \
            --tb=short \
            --junitxml=reports/integration-results.xml
            
      - name: Upload results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: integration-results
          path: reports/

  # ============ 阶段4: E2E 测试 ============
  e2e-tests:
    name: E2E Tests
    needs: integration-tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PYTHON_VERSION }}
          
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          
      - name: Start application
        run: |
          python -m app &
          npx wait-on http://localhost:8000 --timeout 60000
          
      - name: Run E2E tests
        run: |
          pytest tests/e2e/ \
            -v \
            --alluredir=reports/allure-results \
            --tb=short \
            -n 4 \
            --reruns 2 \
            --reruns-delay 5
            
      - name: Generate Allure Report
        if: always()
        run: |
          allure generate reports/allure-results -o reports/allure-report --clean
          
      - name: Upload Allure Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: allure-report
          path: reports/allure-report

  # ============ 阶段5: 通知 ============
  notify:
    name: Notify Results
    needs: [lint, unit-tests, integration-tests, e2e-tests]
    runs-on: ubuntu-latest
    if: always()
    steps:
      - name: Check all jobs status
        run: |
          if [[ "${{ needs.lint.result }}" == "failure" ]] || \
             [[ "${{ needs.unit-tests.result }}" == "failure" ]] || \
             [[ "${{ needs.integration-tests.result }}" == "failure" ]] || \
             [[ "${{ needs.e2e-tests.result }}" == "failure" ]]; then
            echo "Pipeline failed"
            exit 1
          fi
          echo "All tests passed"
          
      - name: Slack Notification
        if: always()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          fields: repo,message,commit,author,action,eventName,ref,workflow
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

### 实操2：Jenkins Declarative Pipeline

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    environment {
        PYTHON_VERSION = '3.11'
        ALLURE_RESULTS = 'reports/allure-results'
        ALLURE_REPORT = 'reports/allure-report'
    }
    
    options {
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Setup') {
            steps {
                sh '''
                    python -m venv .venv
                    . .venv/bin/activate
                    pip install -r requirements.txt
                    pip install -r requirements-dev.txt
                '''
            }
        }
        
        stage('Code Quality') {
            parallel {
                stage('Lint') {
                    steps {
                        sh '''
                            . .venv/bin/activate
                            flake8 src/ tests/ --format=pylint > reports/flake8.txt || true
                        '''
                    }
                }
                stage('Type Check') {
                    steps {
                        sh '''
                            . .venv/bin/activate
                            mypy src/ --ignore-missing-imports --json-file > reports/mypy.json || true
                        '''
                    }
                }
            }
        }
        
        stage('Unit Tests') {
            steps {
                sh '''
                    . .venv/bin/activate
                    pytest tests/unit/ \
                        --cov=src \
                        --cov-report=xml:reports/coverage.xml \
                        --cov-report=html:reports/coverage_html \
                        --cov-fail-under=80 \
                        --junitxml=reports/unit-junit.xml \
                        -n auto
                '''
            }
            post {
                always {
                    junit 'reports/unit-junit.xml'
                    publishHTML([
                        reportDir: 'reports/coverage_html',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }
        
        stage('Integration Tests') {
            steps {
                sh '''
                    . .venv/bin/activate
                    pytest tests/integration/ \
                        --junitxml=reports/integration-junit.xml \
                        -v
                '''
            }
            post {
                always {
                    junit 'reports/integration-junit.xml'
                }
            }
        }
        
        stage('E2E Tests') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'main'
                }
            }
            steps {
                sh '''
                    . .venv/bin/activate
                    pytest tests/e2e/ \
                        --alluredir=${ALLURE_RESULTS} \
                        --junitxml=reports/e2e-junit.xml \
                        -n 4 \
                        --reruns 2
                '''
            }
            post {
                always {
                    junit 'reports/e2e-junit.xml'
                    allure([
                        includeProperties: false,
                        jdk: '',
                        results: [[path: "${ALLURE_RESULTS}"]],
                        properties: [],
                        reportBuildPolicy: 'ALWAYS'
                    ])
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                    // 解析测试结果
                    def testResults = junit testResults: 'reports/*-junit.xml'
                    
                    if (testResults.failCount > 0) {
                        error("Quality Gate Failed: ${testResults.failCount} test(s) failed")
                    }
                    
                    echo "Quality Gate Passed: ${testResults.passCount} passed, ${testResults.skipCount} skipped"
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully'
        }
        failure {
            echo 'Pipeline failed - check test reports'
        }
    }
}
```

### 实操3：pytest 并行执行配置

```ini
# pytest.ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts =
    -v
    --strict-markers
    --tb=short
    --reruns 2
    --reruns-delay 1
    -p no:warnings
markers =
    smoke: 冒烟测试（快速验证核心功能）
    regression: 回归测试（全量验证）
    integration: 集成测试
    e2e: 端到端测试
    slow: 慢速测试（>30s）
    flaky: 不稳定测试（需要重试）
    critical: 关键用例（阻断部署）
    skip_env(prod): 生产环境跳过
```

```python
# tests/conftest.py - 并行执行安全配置

import pytest
import logging
import tempfile
import shutil
from pathlib import Path
from typing import Generator

logger = logging.getLogger(__name__)


@pytest.fixture(scope="function")
def isolated_data_dir() -> Generator[Path, None, None]:
    """
    每个测试独立的临时数据目录
    确保并行执行时文件系统隔离
    """
    temp_dir = Path(tempfile.mkdtemp(prefix="test_"))
    try:
        yield temp_dir
    finally:
        shutil.rmtree(temp_dir, ignore_errors=True)


@pytest.fixture(scope="function")
def isolated_config() -> Generator[dict, None, None]:
    """
    测试隔离的配置
    防止并行测试间的配置污染
    """
    import os
    # 保存原始环境变量
    original_env = {
        key: os.environ.get(key)
        for key in ["TEST_ENV", "DB_HOST", "DB_NAME", "LOG_LEVEL"]
    }
    
    # 为当前测试设置独立配置
    test_id = tempfile.mkdtemp(prefix="test_").split("_")[-1]
    os.environ["DB_NAME"] = f"test_db_{test_id}"
    os.environ["LOG_LEVEL"] = "ERROR"
    
    yield os.environ
    
    # 恢复原始环境变量
    for key, value in original_env.items():
        if value is None:
            os.environ.pop(key, None)
        else:
            os.environ[key] = value


def pytest_collection_modifyitems(config, items):
    """自动添加 marker 基于测试名称规则"""
    for item in items:
        if "smoke" in item.nodeid:
            item.add_marker(pytest.mark.smoke)
        if "e2e" in item.nodeid:
            item.add_marker(pytest.mark.e2e)
        if "slow" in item.nodeid:
            item.add_marker(pytest.mark.slow)
```

## 常见坑

1. **串行执行耗时过长**：E2E 测试串行执行超过 1 小时——使用 pytest-xdist 并行
2. **共享数据库冲突**：并行测试操作同一数据库导致数据污染——每个 worker 独立数据库/schema
3. **非确定性测试**：flaky test 在 CI 中随机失败——标记并重试，后续修复
4. **环境差异**：CI 环境与本地行为不一致——使用 Docker 统一环境
5. **流水线无超时**：测试挂起导致资源浪费——设置合理的 timeout

## 自测清单

- [ ] 能编写 GitHub Actions 多阶段测试流水线
- [ ] 能配置 pytest-xdist 并行执行
- [ ] 能设计测试门禁策略
- [ ] 能处理并行测试的数据隔离问题
- [ ] 能配置 Allure 报告和 Slack 通知

## 延伸阅读

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Jenkins Pipeline Tutorial](https://www.jenkins.io/doc/book/pipeline/)
- [pytest-xdist](https://pytest-xdist.readthedocs.io/)
- [Testing in CI/CD - Google Testing Blog](https://testing.googleblog.com/)
