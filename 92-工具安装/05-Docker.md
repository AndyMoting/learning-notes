# 05-Docker
> 课时：50 min | 难度：★★★

## 安装步骤

### 安装 Docker Desktop

**Windows：**

1. 访问 <https://www.docker.com/products/docker-desktop/>
2. 下载 Docker Desktop for Windows
3. 安装前确认：
   - Windows 版本 ≥ 10 22H2 或 Windows 11
   - 已启用 WSL 2：`wsl --install`
   - BIOS 中启用虚拟化（Intel VT-x / AMD-V）
4. 双击 `Docker Desktop Installer.exe`
5. 安装过程中勾选 "Use WSL 2 instead of Hyper-V"
6. 安装完成后重启计算机
7. 启动 Docker Desktop，等待状态栏显示 "Docker Desktop is running"

**macOS：**

1. 下载 Docker Desktop for Mac（Intel 或 Apple Silicon 对应版本）
2. 将 Docker 拖入 `Applications`
3. 启动 Docker Desktop，授予系统扩展权限
4. 等待菜单栏 Docker 图标变为运行状态

**Linux（Debian/Ubuntu）：**

```bash
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

将当前用户加入 docker 组（免 sudo）：

```bash
sudo usermod -aG docker $USER
newgrp docker
```

### 创建 Docker Hub 账号

1. 访问 <https://hub.docker.com/>
2. 点击 "Sign Up"，填写用户名、邮箱、密码
3. 验证邮箱后登录
4. 终端登录：

```bash
docker login
```

输入用户名和密码，显示 "Login Succeeded" 即成功。

### 运行第一个容器

```bash
docker run hello-world
```

**预期输出：**

```
Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```

### 安装 Docker Compose

**Windows/macOS：** Docker Desktop 已内置 Docker Compose，无需单独安装。

**Linux：**

```bash
sudo apt-get install docker-compose-plugin
```

验证：

```bash
docker compose version
```

**预期输出：**

```
Docker Compose version v2.23.x
```

### 构建第一个测试容器

创建 `Dockerfile`：

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD ["pytest", "--tb=short"]
```

创建 `requirements.txt`：

```
pytest==7.4.3
requests==2.31.0
```

创建 `docker-compose.yml`：

```yaml
version: "3.9"
services:
  test-runner:
    build: .
    volumes:
      - .:/app
    command: pytest --tb=short
```

构建并运行：

```bash
docker compose up --build
```

**预期输出：**

```
[+] Building 5.0s (9/9) FINISHED
[+] Running 2/2
 ✔ Container project-test-runner-1  Recreated
Attaching to project-test-runner-1
test-runner-1  | ============================= test session starts ==============================
test-runner-1  | collected 2 items
test-runner-1  | test_demo.py ..                                                      [100%]
test-runner-1  | ============================== 2 passed in 0.05s ==============================
```

## 验证安装

```bash
docker --version
docker compose version
docker run hello-world
```

**预期输出：**

```
Docker version 24.0.x, build xxxxxxx
Docker Compose version v2.23.x
Hello from Docker!
```

## 常见坑

1. **WSL 2 未安装** — Windows 下执行 `wsl --install`，安装后重启；确认 WSL 版本：`wsl --set-default-version 2`
2. **Docker Desktop 启动失败** — 检查虚拟化是否启用（任务管理器 → 性能 → CPU → 虚拟化：已启用）
3. **Linux 下 docker 命令需要 sudo** — 将用户加入 docker 组：`sudo usermod -aG docker $USER`，然后重新登录
4. **镜像下载慢** — 配置镜像加速器，编辑 `/etc/docker/daemon.json`（Linux）或在 Docker Desktop Settings → Docker Engine 中添加 registry-mirrors
5. **端口冲突** — 容器端口与宿主机冲突时，修改映射：`-p 8081:80` 将容器 80 端口映射到宿主机 8081

## 延伸阅读

- <https://www.docker.com/products/docker-desktop/>
- <https://docs.docker.com/get-docker/>
- <https://docs.docker.com/compose/install/>
- <https://docs.docker.com/reference/dockerfile/>
- <https://hub.docker.com/>
