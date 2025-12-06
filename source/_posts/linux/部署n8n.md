---
title: 部署n8n
toc: true
description: more
tags:
  - linux
categories:
  - linux
date: 2025-11-30 09:59:31
---



# 1. DNS 设置



![image-20251130100249790](E:\111\MyBlog\source\images\n8n\dns.png)



#  2. 创建一个`.env`文件

创建一个项目目录来存储您的 n8n 环境配置和 Docker Compose 文件，并进入该目录：

```bash
mkdir n8n-compose
cd n8n-compose
```

在 n8n-compose 目录下，创建一个 .env 文件来自定义 n8n 实例的详细信息。请将其修改为与您自己的信息相符：

| **.env file**                                                |
| :----------------------------------------------------------- |
| # DOMAIN_NAME and SUBDOMAIN together determine where n8n will be reachable from<br/># The top level domain to serve from<br/>DOMAIN_NAME=example.com<br/><br/># The subdomain to serve from<br/>SUBDOMAIN=n8n<br/><br/># The above example serve n8n at: https://n8n.example.com<br/><br/># Optional timezone to set which gets used by Cron and other scheduling nodes<br/># New York is the default value if not set<br/>GENERIC_TIMEZONE=Europe/Berlin<br/><br/># The email address to use for the TLS/SSL certificate creation<br/>SSL_EMAIL=user@example.com |

# 3. 创建一个本地文件夹

在项目目录中，创建一个名为 local-files 的目录，用于在 n8n 实例和主机系统之间共享文件（例如，使用“从磁盘读取/写入文件”节点）：

```bash
mkdir local-files
```

下面的 Docker Compose 文件可以自动创建此目录，但手动创建可确保以正确的所有权和权限创建该目录。



# 4. 创建 Docker Compose 文件

创建一个名为 compose.yaml 的文件。将以下内容粘贴到该文件中：

```BASH
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_HOST=${SUBDOMAIN}.${DOMAIN_NAME}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - N8N_RUNNERS_ENABLED=true
      - NODE_ENV=production
      - WEBHOOK_URL=https://${SUBDOMAIN}.${DOMAIN_NAME}/
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - TZ=${GENERIC_TIMEZONE}
    volumes:
      - ./n8n_data:/home/node/.n8n
      - ./local-files:/files
    restart: always

```

启动：

```
docker-compose up -d
```

此时 n8n 已经跑在：

```
http://你的服务器IP:5678
```

# 5. 配置 Nginx 反代 + SSL

安装 nginx：

```
sudo apt install nginx -y
```

启用 HTTPS：

你可以选择：

### 方法 1：Let’s Encrypt 自动证书（最常见）

```
sudo apt install certbot python3-certbot-nginx -y
```

执行：

```
sudo certbot --nginx -d n8n.yourdomain.com
```

会自动生成 HTTPS 配置。

然后修改 Nginx 配置 `/etc/nginx/sites-enabled/n8n`：

```
server {
    listen 443 ssl;
    server_name n8n.yourdomain.com;

    location / {
        proxy_pass http://localhost:5678/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

重启 nginx：

```
sudo nginx -s reload
```

现在可以直接访问：

```
https://n8n.yourdomain.com
```
