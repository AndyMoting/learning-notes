# 44-Git 工作流
> 课时：75 min | 难度：★★

## 学习目标
- 理解 Git 三棵树模型（工作区/暂存区/仓库）与分支机制
- 掌握标准 Git Flow 分支模型及其适用场景
- 熟练使用常用 Git 命令管理测试资产
- 掌握 Pull Request 工作流与 Code Review 协作模式

## 核心概念

### Git 三棵树模型

```
工作目录 (Working Directory)
    │
    │ git add
    ▼
暂存区 (Staging Index)
    │
    │ git commit
    ▼
本地仓库 (Local Repository)
    │
    │ git push
    ▼
远程仓库 (Remote Repository)
```

**状态流转：**
```
Untracked → git add → Staged → git commit → Unmodified
Unmodified → 编辑 → Modified → git add → Staged
Unmodified → git rm → Deleted (Untracked)
```

### Git Flow 分支模型

```
main (生产分支) ─────────────────────────────────────────
    ↑                                                    │
    │ 合并hotfix                                        │
develop (开发分支) ──────────────────────────────────┐   │
    ↑         ↑          ↑                           │   │
    │         │          │ merge feature             │   │
feature/    feature/    feature/                     │   │
login       payment     search                       │   │
                                                    │   │
release/1.0 ──────────────── merge ──────────────────┘   │
    │                                                    │
    │ merge                                              │
hotfix/bug-001 ────────────────────────────────────────-┘
```

**分支职责：**
| 分支 | 用途 | 生命周期 | 合并目标 |
|------|------|----------|----------|
| main | 生产可部署版本 | 永久 | 无 |
| develop | 功能集成与测试 | 永久 | 无 |
| feature/xxx | 新功能开发 | 临时 | develop |
| release/xxx | 版本发布准备 | 临时 | develop + main |
| hotfix/xxx | 紧急生产修复 | 临时 | develop + main |

### 常用 Git 命令

**仓库操作：**
```bash
git init                              # 初始化本地仓库
git clone <url>                       # 克隆远程仓库
git clone -b develop <url>            # 克隆指定分支
git remote add origin <url>           # 添加远程仓库
git remote -v                         # 查看远程仓库地址
```

**日常操作：**
```bash
git status                            # 查看工作区状态
git add <file>                        # 添加到暂存区
git add -A                            # 添加所有变更
git add -p                            # 交互式选择部分修改
git commit -m "feat: 新增登录测试用例"  # 提交到本地仓库
git commit --amend                    # 修改最近一次提交
git log --oneline -10                 # 查看最近10条提交
git log --graph --all --oneline       # 图形化查看分支历史
```

**分支操作：**
```bash
git branch                            # 查看本地分支
git branch -a                         # 查看所有分支（含远程）
git branch <name>                     # 创建分支
git branch -d <name>                  # 删除已合并分支
git branch -D <name>                  # 强制删除分支
git checkout <branch>                 # 切换分支
git checkout -b <branch>              # 创建并切换分支
git switch <branch>                   # 切换分支（新语法）
git switch -c <branch>                # 创建并切换分支（新语法）
```

**合并操作：**
```bash
git merge <branch>                    # 合并指定分支到当前分支
git merge --no-ff <branch>            # 禁用快进合并，保留分支痕迹
git rebase <branch>                   # 变基操作
git rebase -i HEAD~3                  # 交互式变基（修改最近3次提交）
```

**远程操作：**
```bash
git push origin <branch>              # 推送分支到远程
git push -u origin <branch>           # 推送并设置上游分支
git push origin --delete <branch>     # 删除远程分支
git pull                              # 拉取并合并
git fetch                             # 仅拉取不合并
git fetch --prune                     # 拉取并清理已删除的远程分支
```

**撤销操作：**
```bash
git restore <file>                    # 丢弃工作区修改
git restore --staged <file>           # 取消暂存（保留工作区修改）
git reset --soft HEAD~1               # 回退提交（保留暂存区和工作区）
git reset --mixed HEAD~1              # 回退提交（保留工作区，清空暂存区）
git reset --hard HEAD~1               # 回退提交（丢弃所有修改）
git revert <commit>                   # 创建反向提交（安全撤销）
```

### .gitignore 模板（测试项目）

```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
venv/
env/
.env
.venv
pip-log.txt
pip-delete-this-directory.txt

# pytest
.pytest_cache/
htmlcov/
.coverage
coverage*.xml
*.cover

# Allure
allure-results/
allure-report/

# IDE
.idea/
.vscode/
*.swp
*.swo
*~

# OS
.DS_Store
Thumbs.db

# 测试数据（敏感）
test_data/secrets/
*.pem
*.key

# 构建产物
dist/
build/
*.egg-info/
```

### 提交信息规范（Conventional Commits）

```
<type>(<scope>): <description>

type 类型：
- feat:     新功能
- fix:      修复 Bug
- docs:     文档变更
- style:    代码格式（不影响功能）
- refactor: 重构
- perf:     性能优化
- test:     测试相关
- chore:    构建/工具变更

示例：
feat(auth): 新增登录接口测试用例
fix(order): 修复订单创建测试的断言逻辑
test(payment): 增加支付宝支付场景测试
chore(ci): 更新 GitHub Actions 工作流配置
```

## 动手实操

### 实操 1：仓库初始化与基础操作

```bash
# 1. 创建测试项目目录
mkdir shop-test && cd shop-test

# 2. 初始化 Git 仓库
git init

# 3. 创建 .gitignore
cat > .gitignore << 'EOF'
__pycache__/
venv/
.env
.pytest_cache/
allure-results/
EOF

# 4. 创建项目结构
mkdir -p tests/{unit,integration,e2e}
mkdir -p test_data/{fixtures,secrets}
mkdir -p reports

# 5. 添加初始文件
cat > tests/__init__.py << 'EOF'
EOF
cat > tests/unit/__init__.py << 'EOF'
EOF
cat > tests/integration/__init__.py << 'EOF'
EOF

# 6. 第一条提交
git add .
git commit -m "chore: 初始化测试项目结构"

# 7. 推送到远程
git remote add origin git@github.com:org/shop-test.git
git push -u origin main
```

### 实操 2：Feature 分支工作流

```bash
# 1. 从 develop 创建功能分支
git checkout develop
git pull origin develop
git checkout -b feature/login-tests

# 2. 在功能分支上开发
cat > tests/unit/test_login.py << 'EOF'
import pytest
from src.auth import login

def test_login_success():
    result = login("admin", "Pass123!")
    assert result["code"] == 200
    assert "token" in result

def test_login_wrong_password():
    result = login("admin", "wrong")
    assert result["code"] == 401
EOF

# 3. 提交
git add tests/unit/test_login.py
git commit -m "feat(login): 新增登录模块单元测试"

# 4. 推送功能分支
git push -u origin feature/login-tests

# 5. 创建 Pull Request（在 GitHub/GitLab 上操作）
# Base: develop ← Compare: feature/login-tests
# 填写 PR 描述，指定 Reviewer

# 6. Review 通过后合并到 develop
git checkout develop
git merge --no-ff feature/login-tests
git push origin develop

# 7. 删除功能分支
git branch -d feature/login-tests
git push origin --delete feature/login-tests
```

### 实操 3：管理测试数据

```bash
# 1. 添加测试数据目录
mkdir -p test_data/fixtures

# 2. 创建测试数据文件
cat > test_data/fixtures/users.json << 'EOF'
[
    {"username": "admin", "role": "admin", "active": true},
    {"username": "user1", "role": "user", "active": true},
    {"username": "user2", "role": "user", "active": false}
]
EOF

# 3. 忽略敏感数据
echo "test_data/secrets/" >> .gitignore

# 4. 创建敏感数据模板（不含真实值）
mkdir -p test_data/secrets
cat > test_data/secrets/config.example.json << 'EOF'
{
    "api_key": "YOUR_API_KEY_HERE",
    "db_password": "YOUR_DB_PASSWORD_HERE"
}
EOF

# 5. 提交
git add test_data/
git commit -m "test(data): 添加测试数据与敏感配置模板"
```

### 实操 4：解决合并冲突

```bash
# 场景：两个分支修改了同一文件的同一位置
# 1. 尝试合并
git checkout develop
git merge feature/payment-tests
# CONFLICT (content): Merge conflict in tests/integration/test_order.py

# 2. 查看冲突文件
git status

# 3. 手动编辑冲突区域
# <<<<<<< HEAD
# develop 分支的内容
# =======
# feature 分支的内容
# >>>>>>> feature/payment-tests

# 4. 解决冲突后标记为已解决
git add tests/integration/test_order.py
git commit -m "fix: 解决测试用例合并冲突"
```

## 常见坑

- 永远不要在 main 分支上直接开发，所有变更通过 PR/MR 合并
- `git reset --hard` 会永久丢失未提交的修改，执行前务必确认
- 不要在 Git 仓库中提交敏感信息（密码、密钥、Token），一旦提交立即轮换
- feature 分支生命周期不宜超过 1-2 周，过长会增加合并冲突概率
- rebase 与 merge 区别：rebase 改写历史保持线性，merge 保留分支结构；已推送的分支禁止 rebase
- 测试报告文件（HTML/JSON）不宜直接提交到 Git，使用 CI 产物存储或 Git LFS

## 自测清单

- [ ] 能解释 Git 三棵树模型中工作区、暂存区、仓库的关系
- [ ] 能描述 Git Flow 中五类分支的职责与合并规则
- [ ] 能熟练使用 add/commit/push/pull/branch/merge 命令
- [ ] 能编写合理的 .gitignore 文件
- [ ] 能遵循 Conventional Commits 规范编写提交信息
- [ ] 能处理合并冲突并正确解决
- [ ] 理解 rebase 与 merge 的区别及各自的适用场景

## 延伸阅读

- Pro Git 官方书籍（免费）：https://git-scm.com/book/zh/v2
- Conventional Commits 规范：https://www.conventionalcommits.org/
- Git Flow 详细说明：https://nvie.com/posts/a-successful-git-branching-model/
- .gitignore 模板集合：https://github.com/github/gitignore
- Git 飞行规则（故障排查）：https://github.com/k88hudson/git-flight-rules
