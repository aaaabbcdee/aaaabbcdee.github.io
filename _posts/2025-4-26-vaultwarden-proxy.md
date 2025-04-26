---
layout:     post
title:      "通过反向代理配置vaultwarden项目实现多端密码存储"
subtitle:   "学习记录"
date:       2025-04-26 15:42:00
author:     "bel"
header-img: "img/post-bg-2015.jpg"
tags:
    - study
---

Vaultwarden 是一个轻量级的密码管理服务，支持自托管。

---

### **1. 准备工作**
	需要准备一台服务器（2核2G即可），一个域名以及对应的ssl证书
#### **1.1 更新系统**
确保系统是最新的：
```bash
sudo apt update && sudo apt upgrade -y
```

#### **1.2 安装 Docker**
Vaultwarden 可以通过 Docker 快速部署。首先安装 Docker 和 Docker Compose。

- 安装 Docker：
```bash
sudo apt install -y docker.io
```

- 启动并设置 Docker 开机自启：
```bash
sudo systemctl start docker
sudo systemctl enable docker
```

- 验证 Docker 是否安装成功：
```bash
docker --version
```

- 安装 Docker Compose：
```bash
sudo apt install -y docker-compose
```

---

### **2. 创建 Vaultwarden 数据目录**

创建一个目录来存储 Vaultwarden 的数据：
```bash
sudo mkdir -p /opt/vaultwarden/data
sudo chmod 750 /opt/vaultwarden/data
```

---

### **3. 使用 Docker 部署 Vaultwarden**

#### **3.1 创建 `docker-compose.yml` 文件**
在 `/opt/vaultwarden/` 目录下创建 `docker-compose.yml` 文件：
```bash
sudo nano /opt/vaultwarden/docker-compose.yml
```

将以下内容粘贴到文件中：
```yaml
version: '3'

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    environment:
      - WEBSOCKET_ENABLED=true  # 启用 WebSocket 支持（可选）
      - ADMIN_TOKEN=your_admin_token  # 设置管理员令牌（用于管理界面）
    ports:
      - "8080:80"  # 将容器的 8080 端口映射到主机的 80 端口
    volumes:
      - /opt/vaultwarden/data:/data  # 挂载数据目录
```

> **注意**：将 `your_admin_token` 替换为你自己的管理员令牌（可以使用随机字符串生成器生成，需要自己记住）。

#### **3.2 启动容器**
在 `/opt/vaultwarden/` 目录下运行以下命令启动 Vaultwarden：
```bash
sudo docker-compose up -d
```

- `-d` 参数表示以后台模式运行。

#### **3.3 检查容器状态**
运行以下命令查看容器是否正常运行：
```bash
sudo docker ps
```

你应该能看到类似以下输出：
```plaintext
CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS         PORTS                NAMES
abc123456789   vaultwarden/server     "/usr/bin/dumb-init …"   2 minutes ago   Up 2 minutes   0.0.0.0:80->80/tcp   vaultwarden
```

---

### **4. 配置 HTTPS（可选但推荐）**
#### **4.1 安装 ssl证书**

为了安全访问你的 Vaultwarden 实例，建议配置 HTTPS。

如果你已经有一个 SSL 证书，可以通过以下步骤配置到 Nginx 中，使其支持 HTTPS。

 **1. 确保 SSL 证书文件可用**
 
你的 SSL 证书通常包含以下两个文件：
- **证书文件**（如 `fullchain.pem` 或 `certificate.crt`）：这是服务器的公钥证书。
- **私钥文件**（如 `privkey.pem` 或 `private.key`）：这是服务器的私钥。

确保你已经将这两个文件上传到服务器上的某个目录。例如：
```bash
/etc/nginx/ssl/
```

如果没有这个目录，可以创建它：
```bash
sudo mkdir -p /etc/nginx/ssl
```

然后将证书和私钥文件复制到该目录中：
```bash
sudo cp /path/to/your/fullchain.pem /etc/nginx/ssl/
sudo cp /path/to/your/privkey.pem /etc/nginx/ssl/
```

**2. 修改 Nginx 配置以支持 HTTPS**

编辑你的 Nginx 配置文件（例如 `/etc/nginx/sites-available/vaultwarden`）：
```bash
sudo nano /etc/nginx/sites-available/vaultwarden
```

在现有配置的基础上，添加 HTTPS 相关的设置。以下是完整的示例配置：

```nginx
server {
    listen 80;
    server_name mindmosaic.xyz;

    # 强制将 HTTP 请求重定向到 HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;  # 启用 HTTPS 和 HTTP/2
    server_name mindmosaic.xyz;

    # SSL 证书和私钥路径
    ssl_certificate /etc/nginx/ssl/fullchain.pem;  # 替换为你的证书文件路径
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;  # 替换为你的私钥文件路径

    # SSL 配置优化（可选）
    ssl_protocols TLSv1.2 TLSv1.3;  # 使用现代加密协议
    ssl_ciphers HIGH:!aNULL:!MD5;   # 使用强加密套件
    ssl_prefer_server_ciphers on;   # 优先使用服务器端加密套件

    # 反向代理配置
    location / {
        proxy_pass http://127.0.0.1:8080;  # 转发到 Vaultwarden 的端口
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

 **3. 测试 Nginx 配置**
 
保存文件后，测试 Nginx 配置是否正确：
```bash
sudo nginx -t
```

如果输出显示 `syntax is ok` 和 `test is successful`，说明配置正确。
 **4. 重新加载 Nginx**
 
重新加载 Nginx 以应用新的配置：
```bash
sudo systemctl reload nginx
```
 **5. 验证 HTTPS 配置**
 
1. 打开浏览器，访问你的域名（如 `https://mindmosaic.xyz`）。
2. 如果配置正确，你应该能够通过 HTTPS 访问 Vaultwarden，并且浏览器会显示一个安全锁标志。
 **6. 自动重定向 HTTP 到 HTTPS**
 
在上面的配置中，我们已经添加了以下内容，用于将 HTTP 请求自动重定向到 HTTPS：
```nginx
server {
    listen 80;
    server_name mindmosaic.xyz;

    return 301 https://$host$request_uri;
}
```

这确保了所有通过 HTTP 的请求都会被重定向到 HTTPS。

---
**总结**
1. 将 SSL 证书和私钥文件上传到服务器（如 `/etc/nginx/ssl/`）。
2. 修改 Nginx 配置文件，添加 `ssl_certificate` 和 `ssl_certificate_key` 路径。
3. 测试并重新加载 Nginx 配置。
4. 验证 HTTPS 是否正常工作。
**注意**
	如果使用了cloudflare来管理域名或出现其它问题，可以参考：
	[[cloudflare中SSL_TLS 模式设置问题]]
#### **4.2 安装 Nginx**
如果尚未安装 Nginx，可以运行以下命令安装：
```bash
sudo apt install -y nginx
```

#### **4.3 配置 Nginx 反向代理**
编辑 Nginx 配置文件（例如 `/etc/nginx/sites-available/vaultwarden`）：
```bash
sudo nano /etc/nginx/sites-available/vaultwarden
```

添加以下内容：
```nginx
server {
    listen 80;
    server_name your-domain.com;  # 替换为你的域名

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

启用配置：
```bash
sudo ln -s /etc/nginx/sites-available/vaultwarden /etc/nginx/sites-enabled/
sudo nginx -t  # 测试配置文件
sudo systemctl reload nginx
```

---

### **5. 访问 Vaultwarden**

#### **5.1 访问 Web 界面**
打开浏览器并访问你的服务器地址或域名：
- 如果未配置 HTTPS：`http://your-server-ip`
- 如果已配置 HTTPS：`https://your-domain.com`

#### **5.2 登录或注册**
首次访问时，你需要注册一个新的账户。此账户将用于管理你的密码库。

---

### **6. 管理 Vaultwarden**

#### **6.1 访问管理界面**
如果你在 `docker-compose.yml` 中设置了 `ADMIN_TOKEN`，可以通过以下 URL 访问管理界面：
```
https://your-domain.com/admin
```

需要输入管理员令牌（即 `ADMIN_TOKEN` 的值）才能访问。

#### **6.2 更新 Vaultwarden**
要更新 Vaultwarden 到最新版本，只需拉取最新的镜像并重启容器：
```bash
sudo docker-compose pull
sudo docker-compose up -d
```

---

### **7. 备份与恢复**

#### **7.1 备份数据**
定期备份 `/opt/vaultwarden/data` 目录，其中包含所有用户数据和配置。

#### **7.2 恢复数据**
将备份的数据复制回 `/opt/vaultwarden/data`，然后重新启动容器即可。

---
通过以上步骤，你就可以成功在 Ubuntu 22.04 上部署 Vaultwarden，并通过 HTTPS 安全地访问你的密码管理服务！
