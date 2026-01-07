---
title: 从手动到全自动：我的博客 GitHub Actions 部署方案
date: 2026-01-07
summary: 记录博客从本地手动上传到 GitHub Actions 自动部署的演进过程，分享如何利用 CI/CD 实现 Push 即发布的顺滑体验。
category: 学习
tags: [GitHub Actions, VPS, 自动化, CI-CD]
---

最开始写博客只是想记录点学习里遇到的一些问题，结果光是**怎么把网页放上去**这件事就让我折腾了好久。

回顾了一下，这期间我换了三种方案，算是一个从**纯体力活**到**全自动化**的过程。

## 1. 最早的做法：本地编译，手动上传

刚开始最简单：

- 在 Windows 本地跑完 `pnpm build`
- 然后打开 `FinalShell`
- 把生成的 `dist` 文件夹拖进服务器。

这种办法问题也很明显：

- 每次改点东西都要重新编译，重新上传
- `dist` 目录下的静态文件数量多且非常零碎
- `FinalShell` 这类图形化工具在上传大量小文件时效率极低

> 其实正常来说，细碎小文件很多的时候，应该打包上传才是最优解，但是我比较懒 OvO

## 2. 稍微进阶点：在 VPS 上直接编译

后来觉得本地传文件太慢，我就直接在 VPS 上把源码拉下来编译。

我这台 VPS 只有 1C1G 的配置。虽然 `Node.js` 跑起来不至于断连，但 CPU 和内存占用会瞬时拉满。不影响使用，只是看着不舒服

## 3. 现在的方案：交给 GitHub Actions 自动干活

![GitHub Actions 构建成功截图](/blog-cicd-guide/github-action.png)

现在我把所有的编译工作都扔给了 GitHub。我在本地写完内容之后直接 `git push`，剩下的安装、构建、传输全都全自动搞定。

**这是我目前正在使用的完整配置文件：**

在项目根目录下创建 **.github/workflows/deploy.yml**，复制以下内容（注意根据你的实际情况修改标注部分）：

```yaml
name: Build and Deploy Astro

on:
  push:
    branches: [main] # 监听 main 分支的推送

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: 检出代码
        uses: actions/checkout@v4

      - name: 安装 pnpm # 建议先于 Node 安装，方便后续 setup-node 成功挂载 pnpm 缓存
        uses: pnpm/action-setup@v3
        with:
          version: 10.27.0

      - name: 设置 Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 24.12.0
          cache: 'pnpm' # 开启缓存，大幅提升后续部署速度

      - name: 安装依赖并构建
        run: | # 这里的管道符表示下面是一个“命令块”，可以像在终端一样换行写多条命令
          pnpm install
          pnpm build

      - name: 部署到 VPS
        uses: appleboy/scp-action@v0.1.7
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}

          # --- 方案 A：如果你习惯用 SSH 密钥（推荐） ---
          key: ${{ secrets.SSH_KEY }}
          # --- 方案 B：如果你习惯用 VPS 初始密码 ---
          # password: ${{ secrets.SSH_PASSWORD }}

          source: 'dist/*' # 如果只写 dist，会把文件夹本身传过去；加了 /* 则只传内部文件
          target: '/var/www/reavalon.com/dist' # 你的 VPS 网页存放路径
          strip_components: 1 # 剥离 dist/ 前缀，否则 VPS 路径会变成 .../dist/dist/index.html
          overwrite: true
          rm: true # 上传前清空 target 目录，确保服务器文件与仓库完全同步，不留旧残余
```

## Secrets 配置说明

`${{ secrets.XXX }}` 是加密变量

你需要去你的 GitHub 项目页面点击:

**Settings -> Secrets and variables -> Actions -> New repository secret** 填写：

- **SSH_HOST**: 服务器 IP。
- **SSH_USER**: 用户名（通常是 `root`）
- **SSH_KEY**: 你的 SSH 私钥（`id_rsa` 里的全部文本）。
- **SSH_PASSWORD**: 如果你不用密钥，那就新建这个变量填入你的 VPS 登录密码，并在脚本里把 **key** 换成 **password。**

---

## 关于 `rm: true`

`rm: true` 会在上传前**直接清空 `target` 目录，这是一个不可逆的物理删除操作**。

在开启之前，请务必确认：

- target 路径是否完全正确
- 不要写成 /、/var/www 等危险路径

强烈建议第一次运行前，先手动到服务器上确认一遍目录结构。

---

## 一点感想

折腾完这一整套流程之后，我只需要写内容、推送代码，一切自动完成 —— 很适合我这种懒狗 🤣
