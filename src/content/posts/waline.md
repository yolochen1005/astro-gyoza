---
title: 在自己的 VPS 上部署 Waline 评论系统
date: 2026-01-02
summary: 记录一次在个人 VPS 上从零部署 Waline 评论系统的完整过程，包括踩过的坑、权限问题以及最终的稳定方案。
category: 学习
tags: [Waline, 评论系统, VPS, Docker]
---

## 前言

今天我在自己的 VPS 上跑通了 Waline 评论系统。整个从博客上线到添加自建评论系统，加上中途休息断断续续，大概半个小时就完成了。过程顺利，但仍有几个容易踩坑的地方，所以写下来作为参考，也方便自己日后回顾。

[Waline](https://github.com/walinejs/waline)是一款轻量、开源、支持自托管的评论系统，安全性和扩展性都比 Valine 更好。我希望评论系统 **完全在自己掌控的服务器上运行**，所以选择了自托管 Docker 方案。

---

## 准备工作

在动手之前，请确保了以下条件：

- 一台可以公网访问的 VPS
- 自备域名，并设置好 A 记录指向 VPS（这里演示以域名 comments.reavalon.com为例）
- Nginx 已经部署并可以正常访问博客
- Docker 和 Docker Compose 已安装
- 基本终端操作能力

---

## 部署步骤概览

1. **创建数据目录**

```bash
mkdir -p /root/waline/data
```

2. **下载官方示例 SQLite 数据库**

也可以使用别的数据库，为了方便仅演示 SQlite

[Waline官方数据库](https://github.com/walinejs/waline/tree/main/assets)

```bash
cd /root/waline/data
wget https://raw.githubusercontent.com/walinejs/waline/main/assets/waline.sqlite
```

3. **准备 docker-compose.yml 文件**

确保当前目录为 **/root/waline**

```bash
cd /root/waline
vim docker-compose.yml
```

```yaml
services:
  waline:
    container_name: waline
    image: lizheming/waline:latest
    restart: always
    ports:
      - 8360:8360
    volumes:
      - ./data:/app/data
    environment:
      TZ: 'Asia/Shanghai'
      SQLITE_PATH: '/app/data/waline.sqlite'
      JWT_TOKEN: '随机生成的安全 token'
      SITE_NAME: 'Avalon'
      SITE_URL: 'https://comments.reavalon.com'
      SECURE_DOMAINS: 'reavalon.com,comments.reavalon.com'
      AUTHOR_EMAIL: '你的邮箱'
```

4. **修改权限，保证 Docker 能访问数据库**

```bash
chmod 755 /root/waline/data
chmod 664 /root/waline/data/waline.sqlite
```

5. **启动容器**

```bash
cd /root/waline
docker compose up -d
```

6. **Nginx 反代配置**

```nginx
server
{
  listen 80;
  server_name comments.reavalon.com;

  location / {
    proxy_pass http://127.0.0.1:8360;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header REMOTE-HOST $remote_addr;
    add_header X-Cache $upstream_cache_status;
    # cache
    add_header Cache-Control no-cache;
    expires 12h;
  }
}
```

重载 nginx

```bash
sudo nginx -t
sudo systemctl reload nginx
```

用 certbot 申请证书

```bash
sudo certbot --nginx -d comments.reavalon.com
```

7. **测试访问**

打开 `https://comments.reavalon.com`，首次访问应该是这个样子

![](/waline/waline.png)

这时候输入 `https://comments.reavalon.com/ui/login`，默认情况首个注册的人会被设定成管理员。

## 常见坑与解决

1. **403 Forbidden**

- 原因：域名校验不通过
- 解决：在 `SECURE_DOMAINS` 中加入博客主域和评论子域，例如 `reavalon.com,comments.reavalon.com`

2. **500 SQLITE_ERROR: no such table**

- 原因：数据库未初始化
- 解决：
  - 指定 `SQLITE_PATH` 到文件，例如 `/app/data/waline.sqlite`
  - 首次访问 Waline 后端自动建表
  - 确认 Docker 数据卷权限

3. **权限问题**

- Docker 容器内要能读写数据库
- chmod 755 数据目录 + 664 数据库文件是最稳的
