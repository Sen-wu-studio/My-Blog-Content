---
title: Mizuki 博客基础框架搭建记录
published: 2026-04-22
pinned: true
description: 网站框架更换，使用 Astro + Mizuki 框架。
tags: [Blogging, 杂谈]
category: 网站技术
licenseName: "Unlicensed"
author: 留夏
draft: false
comment: true
pubDate: 2026-04-22
date: 2026-04-20
image: "./cover.png"
permalink: "blog-setup-summary"
---

# Mizuki 博客基础框架搭建记录

> 本文记录一次基于 **Astro + Mizuki + 内容分离 + Docker Nginx + Cloudflare Tunnel + GitHub Webhook + Twikoo** 的个人博客基础框架搭建流程。  
> 目标是实现：多设备写作、内容仓库独立管理、本地服务器自动构建发布、评论系统自托管。

---

## 1. 前言

worldpress 使用了很长一段时间，逐渐发现有诸多不便和弊端，列如备份麻烦、主题迁移麻烦、不支持 markdown 格式文档等等。
拖了很久终于下决心大改博客框架，本文就是记录全新博客框架的方案和实现过程。
我的服务器使用 Ubuntu24.04系统，本文所有操作均在服务器上。

## 2. 最终架构

整体访问链路如下：

```text
GitHub 内容仓库
        ↓ GitHub Webhook
本地 Webhook 服务
        ↓
Mizuki sync-content
        ↓
pnpm build
        ↓
Docker Nginx 发布静态文件
        ↓
Cloudflare Tunnel
        ↓
blog.liveinnow.top
```

评论系统链路如下：

```text
Mizuki 文章页
        ↓
Twikoo 评论组件
        ↓
twikoo.liveinnow.top
        ↓
Twikoo Docker 后端
        ↓
MongoDB Docker 数据库
```

---

## 3. 方案选型

本次选择的方案：

| 模块 | 方案 |
|---|---|
| 博客框架 | Astro |
| 博客主题 | Mizuki |
| 内容管理 | 独立 GitHub 内容仓库 |
| 本地 Web 服务 | Docker Nginx |
| 外网访问 | Cloudflare Tunnel |
| 自动更新 | GitHub Webhook |
| 评论系统 | Twikoo |
| 评论数据库 | MongoDB Docker |

选择 Mizuki 的主要原因：

- 支持内容分离模式；
- 适合个人主页、项目展示、独立页面和博客归档；
- 基于 Astro，前端生态现代；
- 支持 Twikoo 评论；
- 适合后期迁移和维护。

---

## 4. 目录结构

服务器上的主要目录：

```text
/home/usr/myserver/
├── blog/
│   ├── Mizuki/                  # Mizuki 主题项目
│   │   ├── content/             # sync-content 拉取的内容仓库
│   │   ├── dist/                # pnpm build 构建结果
│   │   └── update-mizuki.sh     # 自动更新脚本
│   │
│   └── webhook/                 # GitHub Webhook 服务
│       └── webhook-server.js
│
├── myweb/                       # Docker Nginx 映射的网站根目录
│
└── twikoo/
    ├── docker-compose.yml       # Twikoo + MongoDB
    └── mongo-data/              # MongoDB 数据目录
```

Docker Nginx 映射关系：

```text
宿主机目录：/home/usr/myserver/myweb
容器内目录：/usr/share/nginx/html
```

---

## 5. Mizuki 配置

Mizuki 主题：[Mizuki](https://github.com/LyraVoid/Mizuki)
::github{repo="LyraVoid/Mizuki"}

Mizuki 文档：[Mizuki Next Theme](https://docs.mizuki.mysqil.com/)
Mizuki 主题的文档写的非常详尽，所以我这里只会写关键指令不做多的赘述。

下载 Mizuki主题 

```bash
cd /home/usr/myserver/blog
git clone https://github.com/LyraVoid/Mizuki.git
cd Mizuki
pnpm install
```

启动开发服务器

```bash
pnpm dev
```

可在 http://localhost:4321 访问网页

建立内容仓库
```bash
cd /home/usr/myserver/blog
mkdir My-Content
cd My-Content
git init 
git branch -M main
git add.
git commit -m init
git remote add origin https://github.com/yourname/My-Content.git
git push -u origin main
```

`.env` 配置示例：

```env
ENABLE_CONTENT_SYNC=true
CONTENT_REPO_URL=git@github.com:yourname/My-Content.git
CONTENT_DIR=./content
```

同步内容仓库：

```bash
pnpm run sync-content
```

构建 Mizuki：

```bash
pnpm build
```

部署到 Nginx 网站目录：

```bash
rsync -av --delete dist/ /home/usr/myserver/myweb/
```

---

## 6. GitHub SSH 配置

为了避免 HTTPS 访问 GitHub 出现 TLS 问题，服务器使用 SSH 拉取内容仓库。

生成 SSH key：

```bash
ssh-keygen -t ed25519 -C "mizuki-server"
```

查看公钥：

```bash
cat ~/.ssh/id_ed25519.pub
```

将公钥添加到 GitHub：

```text
GitHub
→ Settings
→ SSH and GPG keys
→ New SSH key
```

测试 SSH：

```bash
ssh -T git@github.com
```

成功后，内容仓库地址使用：

```text
git@github.com:yourname/My-Content.git
```

---

## 7. Docker Nginx 部署

当前服务器中 Nginx 是 Docker 容器，不是宿主机直接安装的 Nginx。

查看容器：

```bash
docker ps
```

示例：

```text
nginx:latest   127.0.0.1:80->80/tcp   my-nginx
```

查看 Nginx 配置：

```bash
docker exec -it my-nginx nginx -T
```

查看网站目录：

```bash
docker inspect my-nginx --format '{{json .Mounts}}' | python3 -m json.tool
```

最终确认：

```text
/home/ws/myserver/myweb → /usr/share/nginx/html
```

因此 Mizuki 构建后只需要同步到：

```bash
/home/usr/myserver/myweb
```

---

## 8. Cloudflare Tunnel 配置

cloudflared systemd 服务：

```bash
systemctl cat cloudflared
```

当前服务配置：

```ini
[Service]
Type=simple
User=usr
ExecStart=/usr/local/bin/cloudflared tunnel run myserver
Restart=on-failure
RestartSec=5s
```

因为服务以 `usr` 用户运行，默认读取配置：

```text
/home/usr/.cloudflared/config.yml
```

配置示例：

```yaml
tunnel: myserver
credentials-file: /home/usr/.cloudflared/your-tunnel-id.json

ingress:
  - hostname: blog.liveinnow.top
    service: http://localhost:80

  - hostname: hook.liveinnow.top
    service: http://localhost:9000

  - hostname: twikoo.liveinnow.top
    service: http://localhost:8082

  - service: http_status:404
```

新增 DNS route：

```bash
cloudflared tunnel route dns myserver blog.liveinnow.top
cloudflared tunnel route dns myserver hook.liveinnow.top
cloudflared tunnel route dns myserver twikoo.liveinnow.top
```

重启 cloudflared：

```bash
sudo systemctl restart cloudflared
```

查看状态：

```bash
sudo systemctl status cloudflared
```

查看日志：

```bash
journalctl -u cloudflared -f
```

---

## 9. 自动更新脚本

更新脚本路径：

```text
/home/usr/myserver/blog/Mizuki/update-mizuki.sh
```

脚本示例：

```bash
#!/usr/bin/env bash
set -e

export PATH="/home/usr/.local/share/pnpm:/usr/local/bin:/usr/bin:/bin:$PATH"

PROJECT_DIR="/home/usr/myserver/blog/Mizuki"
WEB_DIR="/home/usr/myserver/myweb"

cd "$PROJECT_DIR"

echo "[1/3] Sync content..."
pnpm run sync-content

echo "[2/3] Build Mizuki..."
pnpm build

echo "[3/3] Deploy to Docker Nginx..."
rsync -av --delete dist/ "$WEB_DIR/"

echo "Mizuki deployed successfully."
```

注意：

如果脚本由 systemd 服务调用，可能出现：

```text
pnpm: 未找到命令
```

原因是 systemd 环境变量较少，没有加载用户终端的 PATH。

解决方式：

- 在脚本中补充 `PATH`；
- 或者使用 `which pnpm` 查到 pnpm 绝对路径后，在脚本中直接调用绝对路径。

查看 pnpm 路径：

```bash
which pnpm
```

给脚本执行权限：

```bash
chmod +x /home/usr/myserver/blog/Mizuki/update-mizuki.sh
```

手动测试：

```bash
/home/usr/myserver/blog/Mizuki/update-mizuki.sh
```

---

## 10. GitHub Webhook 自动触发

Webhook 服务作用：

```text
GitHub 内容仓库 push
        ↓
GitHub 请求 hook.liveinnow.top/github
        ↓
Cloudflare Tunnel 转发到 localhost:9000
        ↓
本地 Node.js Webhook 服务
        ↓
执行 update-mizuki.sh
```

Webhook 服务文件：

```text
/home/usr/myserver/blog/webhook/webhook-server.js
```

核心逻辑：

- 监听 `127.0.0.1:9000`；
- 只接受 `POST /github`；
- 校验 GitHub Webhook Secret；
- 只处理 `push` 和 `ping` 事件；
- push 时执行 Mizuki 更新脚本。

systemd 服务：

```text
/etc/systemd/system/mizuki-webhook.service
```

服务配置示例：

```ini
[Unit]
Description=Mizuki GitHub Webhook
After=network.target

[Service]
Type=simple
User=usr
WorkingDirectory=/home/usr/myserver/blog/webhook
Environment=GITHUB_WEBHOOK_SECRET=your-secret
ExecStart=/usr/bin/node /home/usr/myserver/blog/webhook/webhook-server.js
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

启动服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now mizuki-webhook
```

查看状态：

```bash
sudo systemctl status mizuki-webhook
```

查看日志：

```bash
journalctl -u mizuki-webhook -f
```

GitHub Webhook 配置位置：

```text
My-Blog-Content
→ Settings
→ Webhooks
→ Add webhook
```

配置内容：

```text
Payload URL: https://hook.liveinnow.top/github
Content type: application/json
Secret: 与 systemd 中 GITHUB_WEBHOOK_SECRET 一致
Event: Just the push event
Active: 勾选
```

---

## 11. Twikoo 评论系统

Twikoo 使用 Docker 部署，数据库使用 MongoDB。

目录：

```bash
mkdir -p /myserver/twikoo
cd /myserver/twikoo
```

使用 docker-compose 安装

`docker-compose.yml`：

```yaml
services:
  mongo:
    image: mongo:6
    container_name: twikoo-mongo
    restart: always
    volumes:
      - ./mongo-data:/data/db
    networks:
      - twikoo-net

  twikoo:
    image: imaegoo/twikoo:latest
    container_name: twikoo
    restart: always
    depends_on:
      - mongo
    ports:
      - "127.0.0.1:8082:8080"
    environment:
      - MONGODB_URI=mongodb://mongo:27017/twikoo
      - TZ=Asia/Shanghai
    networks:
      - twikoo-net

networks:
  twikoo-net:
    driver: bridge
```

启动：

```bash
docker compose up -d
```

查看状态：

```bash
docker ps
```

查看日志：

```bash
docker logs -f twikoo
docker logs -f twikoo-mongo
```

本地测试：

```bash
curl -I http://127.0.0.1:8082
```

Cloudflare Tunnel 暴露：

```yaml
- hostname: twikoo.liveinnow.top
  service: http://localhost:8082
```

Mizuki 中开启评论：

```ts
export const commentConfig = {
  enable: true,
  twikoo: {
    envId: "https://twikoo.liveinnow.top",
    lang: "zh-CN",
  },
}
```

文章 Frontmatter 中开启评论：

```yaml
---
title: 示例文章
published: 2026-04-21
description: 示例文章描述
tags: [Mizuki, Twikoo]
category: Blog
draft: false
comment: true
---
```

---

## 12. 日常写作和更新流程

在任意设备上克隆内容仓库：

```bash
git clone git@github.com:Sen-wu-studio/My-Blog-Content.git
cd My-Blog-Content
```

新增或修改文章：

```text
src/content/posts/
```

提交：

```bash
git add .
git commit -m "update blog content"
git push
```

之后 GitHub Webhook 会自动触发服务器更新。


---

## 13. 常用排查命令

### 查看 Docker 容器

```bash
docker ps
```

### 查看 Nginx 容器配置

```bash
docker exec -it my-nginx nginx -T
```

### 查看 Nginx 网站目录映射

```bash
docker inspect my-nginx --format '{{json .Mounts}}' | python3 -m json.tool
```

### 查看 cloudflared 服务

```bash
sudo systemctl status cloudflared
journalctl -u cloudflared -f
```

### 查看 cloudflared 配置

```bash
cat /home/ws/.cloudflared/config.yml
```

### 查看 Webhook 服务

```bash
sudo systemctl status mizuki-webhook
journalctl -u mizuki-webhook -f
```

### 测试博客访问

```bash
curl -I http://localhost
curl -I https://blog.liveinnow.top
```

### 测试 Webhook 入口

```bash
curl -I https://hook.liveinnow.top/github
```

说明：

`GET /github` 返回 404 是正常的，因为 Webhook 服务只处理 `POST /github`。

### 测试 Twikoo

```bash
curl -I https://twikoo.liveinnow.top
docker logs -f twikoo
```

---

## 14. 总结

这次从 worldpress 更换到现在的框架也是花了不少功夫，下面列举下更新框架的优点。

核心优点：

- 内容主题分离，不易被平台绑死；
- 多设备写作方便；
- GitHub push 自动更新；
- 服务更轻量；
- 可使用 git 管理，更加工程化；
- 安全面更小
- 后续迁移成本较低。

接下来不继续折腾主题了，重点搞内容，内容才是博客的灵魂呀。
