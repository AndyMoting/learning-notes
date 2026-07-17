# 06-Git-GitHub
> 课时：40 min | 难度：★★

## 安装步骤

### 安装 Git

**Windows：**

1. 访问 <https://git-scm.com/download/win>
2. 下载 64-bit Git for Windows Setup
3. 安装选项（推荐设置）：
   - Select Components: 默认勾选
   - Default editor: 选择 "Use Visual Studio Code as Git's default editor" 或 "Nano"
   - PATH environment: 选择 "Git from the command line and also from 3rd-party software"
   - HTTPS transport backend: 使用 OpenSSL
   - Line ending conversions: 选择 "Checkout Windows-style, commit Unix-style line endings"
4. 完成安装

**macOS：**

```bash
brew install git
```

或通过 Xcode Command Line Tools：`xcode-select --install`

**Linux（Debian/Ubuntu）：**

```bash
sudo apt-get install git
```

验证安装：

```bash
git --version
```

**预期输出：**

```
git version 2.42.x
```

### 配置 Git 用户信息

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
git config --global core.autocrlf true   # Windows
git config --global core.autocrlf input  # macOS/Linux
```

验证配置：

```bash
git config --list
```

### 生成 SSH 密钥并配置 GitHub

1. 生成密钥对：

```bash
ssh-keygen -t ed25519 -C "your.email@example.com"
```

按 Enter 接受默认路径，设置密码（可为空）。

2. 复制公钥内容：

**Windows：**

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | Set-Clipboard
```

**macOS：**

```bash
pbcopy < ~/.ssh/id_ed25519.pub
```

**Linux：**

```bash
cat ~/.ssh/id_ed25519.pub
```

3. 访问 <https://github.com/settings/keys>
4. 点击 "New SSH key"
5. Title 填写 "Work-Laptop"，Key 粘贴公钥内容
6. 点击 "Add SSH key"

7. 验证连接：

```bash
ssh -T git@github.com
```

**预期输出：**

```
Hi username! You've successfully authenticated, but GitHub does not provide shell access.
```

### 创建第一个仓库

```bash
mkdir test-project
cd test-project
git init
echo "# Test Project" > README.md
git add README.md
git commit -m "Initial commit"
git branch -M main
git remote add origin git@github.com:username/test-project.git
git push -u origin main
```

### 创建 .gitignore（Python 测试项目）

在项目根目录创建 `.gitignore`：

```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
.venv/
venv/
ENV/
env/
*.egg-info/
dist/
build/

# Testing
.pytest_cache/
.coverage
htmlcov/
.tox/
allure-results/
allure-report/
*.log

# IDE
.idea/
.vscode/
*.swp
*.swo

# OS
.DS_Store
Thumbs.db
```

### 克隆已有仓库

```bash
git clone git@github.com:username/repository.git
cd repository
```

## 验证安装

```bash
git --version
ssh -T git@github.com
```

**预期输出：**

```
git version 2.42.x
Hi username! You've successfully authenticated, but GitHub does not provide shell access.
```

## 常见坑

1. **SSH 连接超时** — 检查网络或防火墙是否屏蔽 22 端口；可改用 HTTPS 方式：`git remote set-url origin https://github.com/username/repo.git`
2. **每次 push 要求输入密码** — SSH 密钥未加载，执行 `ssh-add ~/.ssh/id_ed25519`（Windows：`Get-Service ssh-agent | Set-Service -StartupType Automatic; Start-Service ssh-agent; ssh-add $env:USERPROFILE\.ssh\id_ed25519`）
3. **换行符警告** — Windows 设置 `git config --global core.autocrlf true`，macOS/Linux 设置 `input`
4. **.gitignore 不生效** — 文件已被跟踪时需先移除缓存：`git rm -r --cached . && git add . && git commit -m "Update gitignore"`
5. **commit 时提示设置 user.email** — 执行 `git config --global user.email "your.email@example.com"`

## 延伸阅读

- <https://git-scm.com/download/win>
- <https://docs.github.com/en/authentication/connecting-to-github-with-ssh>
- <https://git-scm.com/book/zh/v2>
- <https://www.atlassian.com/git/tutorials/setting-up-a-repository>
