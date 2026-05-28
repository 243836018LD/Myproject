# 📚 博客系统CI/CD项目 完整部署文档+踩坑手册

## 一、项目概述
本项目为企业级全容器化博客系统，实现了从代码提交到生产上线的完整自动化CI/CD流程，具备生产级环境一致性、可运维、可扩展能力，可直接用于团队协作与多环境部署。

### 核心技术栈
| 模块 | 技术选型 |
|------|----------|
| 容器编排 | Docker + Docker Compose |
| Web服务 | Nginx 反向代理 |
| 应用服务 | WordPress 博客系统 |
| 数据库 | MySQL |
| 监控体系 | Prometheus + Grafana |
| 私有仓库 | Harbor |
| CI/CD工具 | GitHub Actions |

---

## 二、完整部署步骤
### 1. 虚拟机基础环境准备
```bash
# 更新系统
sudo apt update && sudo apt upgrade -y

# 安装依赖
sudo apt install git vim curl net-tools -y

# 安装Docker
curl -fsSL https://get.docker.com | bash
sudo systemctl enable docker
sudo systemctl start docker

# 安装Docker Compose
sudo apt install docker-compose-plugin -y

# 验证安装
docker --version
docker compose version
```

### 2. 项目初始化与Git配置
```bash
# 进入项目目录
cd /opt/myproject

# 配置Git用户信息
git config --global user.name "243836018LD"
git config --global user.email "2438376018@qq.com"

# 生成SSH密钥，配置GitHub免密登录
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519 -C "vm-ubuntu"
cat ~/.ssh/id_ed25519.pub
# 将公钥添加到GitHub Settings → SSH and GPG keys

# 测试SSH连通性
ssh -T git@github.com

# 初始化Git仓库
git init
git remote add origin git@github.com:243836018LD/Myproject.git
git add .
git commit -m "init: 项目初始提交"
git push -u origin main
```

### 3. Harbor私有仓库部署
```bash
# 下载Harbor离线安装包
wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-online-installer-v2.10.0.tgz
tar -zxvf harbor-online-installer-v2.10.0.tgz
cd harbor

# 配置Harbor
cp harbor.yml.tmpl harbor.yml
# 修改harbor.yml中的hostname为192.168.2.126:8080，关闭https
./install.sh --with-clair --with-trivy

# 启动Harbor
docker compose up -d

# 访问Harbor
# 地址：http://192.168.2.126:8080
# 默认账号：admin
# 默认密码：Harbor12345
```

### 4. 项目核心配置文件
#### docker-compose.yml
```yaml
version: '3'
services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - wordpress
    restart: always

  wordpress:
    image: wordpress:latest
    ports:
      - "8080:80"
    environment:
      WORDPRESS_DB_HOST: mysql:3306
      WORDPRESS_DB_USER: wordpress
      WORDPRESS_DB_PASSWORD: wordpress@123
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wordpress_data:/var/www/html
    depends_on:
      - mysql
    restart: always

  mysql:
    image: mysql:5.7
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root@123
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wordpress
      MYSQL_PASSWORD: wordpress@123
    volumes:
      - mysql_data:/var/lib/mysql
    restart: always

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    restart: always

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    restart: always

volumes:
  wordpress_data:
  mysql_data:
  prometheus_data:
  grafana_data:
```

#### nginx.conf
```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://wordpress:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

#### prometheus.yml
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```

### 5. GitHub Actions CI/CD流水线配置
文件路径：`.github/workflows/deploy.yml`
```yaml
name: 博客项目CI/CD（构建→推Harbor→自动部署）

on:
  push:
    branches: [ main ]

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: 拉取代码
        uses: actions/checkout@v4

      - name: 登录Harbor私有仓库
        uses: docker/login-action@v3
        with:
          registry: 192.168.2.126:8080
          username: ${{ secrets.HARBOR_USER }}
          password: ${{ secrets.HARBOR_PWD }}

      - name: 构建并推送镜像到Harbor
        run: |
          docker build -t 192.168.2.126:8080/lnmp/blog:latest .
          docker push 192.168.2.126:8080/lnmp/blog:latest

      - name: 远程部署到虚拟机
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: root
          password: ${{ secrets.SERVER_PWD }}
          script: |
            cd /opt/myproject
            docker compose pull
            docker compose up -d
```

### 6. 启动项目
```bash
cd /opt/myproject
docker compose up -d
```

访问服务：
- 博客系统：http://192.168.2.126
- Prometheus监控：http://192.168.2.126:9090
- Grafana可视化：http://192.168.2.126:3000
- Harbor仓库：http://192.168.2.126:8080

---

## 三、踩坑手册（全流程问题解决方案）
### 1. Git相关问题
| 报错现象 | 根因 | 解决方案 |
|----------|------|----------|
| `fatal: 不是 git 仓库（或者任何父目录）：.git` | .git目录损坏/未初始化 | `rm -rf .git && git init` 重新初始化 |
| `Recv failure: 连接被对方重置` | HTTPS协议被网络阻断 | 切换为SSH协议，生成SSH密钥并添加到GitHub |
| `Key is invalid. You must supply a key in OpenSSH public key format` | 公钥格式错误/复制不全 | 重新生成ed25519密钥，完整复制一整行公钥，无换行无空格 |
| `ERROR: Repository not found` | 仓库名/用户名拼写错误 | 核对仓库地址，确保用户名和仓库名完全匹配，先在GitHub网页创建空仓库 |
| `Everything up-to-date` | 本地与远程代码一致，无新提交 | 正常提示，无需处理，修改文件后重新提交即可 |

### 2. Docker相关问题
| 报错现象 | 根因 | 解决方案 |
|----------|------|----------|
| Docker命令无权限 | 当前用户不在docker组 | `sudo usermod -aG docker $USER`，重启终端生效 |
| 容器启动失败，端口冲突 | 宿主机端口被占用 | `netstat -tulpn` 查看占用端口，修改docker-compose.yml中的端口映射 |
| Harbor登录失败，私有仓库拉取镜像报错 | Docker不信任私有仓库 | 修改 `/etc/docker/daemon.json`，添加 `"insecure-registries": ["192.168.2.126:8080"]`，重启Docker |

### 3. CI/CD流水线问题
| 报错现象 | 根因 | 解决方案 |
|----------|------|----------|
| GitHub Actions登录Harbor失败 | 密钥配置错误 | 核对GitHub仓库的Secrets配置，确保HARBOR_USER、HARBOR_PWD完全正确 |
| 远程部署到虚拟机失败 | SSH连接失败 | 确认虚拟机SSH服务开启，防火墙允许22端口，密钥配置正确 |
| 镜像推送Harbor失败 | Harbor项目不存在 | 先在Harbor网页创建lnmp项目，再推送镜像 |

### 4. 其他常见问题
1.  **Vim交换文件 `.swp` 问题**：编辑文件异常退出生成的垃圾文件，用 `rm -f .*.swp` 删除，配置 `set noswapfile` 永久禁用
2.  **Nginx 502错误**：反向代理配置错误，确认wordpress容器正常启动，nginx.conf配置正确
3.  **MySQL连接失败**：数据库容器未启动，核对数据库账号密码，确认docker-compose.yml中的环境变量配置正确
