---
title: "GitHub Actions 自动部署 Hugo 博客到云服务器"
date: 2022-11-25T20:38:01+08:00
tags:
- 博客
- GitHub Actions
---

之前本博客的源码是存放在 GitHub，托管在 Vercel 中。我只需在本地编写文章，编写完毕后将文章提交到 GitHub 仓库中。Vercel 可以监控 GitHub 仓库的变化，当仓库有更新时，Vercel 会读取仓库内容，自动执行编译操作，并更新博客网站。

之前在腾讯云购买了三年的轻量应用服务器，为了把这台服务器利用起来，我准备将博客迁移到腾讯云服务器。使用国内的服务器最大的优点是，国内访问的速度非常快。但是，国内的服务器建站需要对域名备案，否则无法通过 HTTPS 访问网站。

迁移到腾讯云后，当博客有更行时，只能手动编译上传到服务器，这种方式非常麻烦，于是想到利用 GitHub Actions 自动部署博客，当更新博客时，由 GitHub Actions 编译同步到服务器。


## 启用 GitHub Actions

> 在开始前，先确保你本地已安装好 Hugo 博客系统。

首先，在博客源码的根目录创建一个 `.github\workflows` 目录，然后在此目录中创建一个 Actions 的配置文件，这里我命名为 `deploy.yml`。文件的内容如下：

```yml
name: CodeClub Deloyment

on:
  push:
    branches: master
  pull_request:
    branches: master

jobs:
  deployment:
    runs-on: ubuntu-latest
    environment:
      name: Dev
      url: https://${{ secrets.HOSTNAME }}
    steps:
      - uses: actions/checkout@v3
        with:
          submodules: 'recursive'
      - name: Setting SSH key
        uses: webfactory/ssh-agent@v0.5.4
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
      - name: Scan public keys
        run: ssh-keyscan ${{ secrets.HOSTNAME }} >> ~/.ssh/known_hosts
      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with: 
          hugo-version: ${{ vars.HUGO_VERSION }}
          extended: true
      - name: Build
        run: hugo --gc --minify -b https://${{ secrets.HOSTNAME }}
      - name: Deploy
        run: rsync -avz --delete public/ ${{ secrets.USERNAME }}@${{ secrets.HOSTNAME }}:/var/www/html
```
- `name`: Workflow 的名称，这里可以按个人喜好定义。
- `on`: 定义何时触发 Workflow，这里我定义当 `push` 或 `PR` 到仓库的 `master` 分支，执行此 Workflow。
- `jobs`: Jobs 是由一系列的 `steps` 组成，这些 `steps` 在同一个运行环境中按顺序运行。每一个 `step` 要么执行一条 shell 命令，要么执行一个 Actions。


```
- uses: actions/checkout@v3
  with:
    submodules: 'recursive'
```
为了将源码部署到服务器上，第一步是从我们存储的仓库 checkout 出需要部署的网站源码。‘

```
- name: Setting SSH key
  uses: webfactory/ssh-agent@v0.5.4
  with:
    ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}
```
这一步是将我们的私钥添加到 Actions Runner 的 ssh-agent 中去。因为我们在使用 ssh 连接到服务器上时，需要使用密钥，但是每次运行 GitHub Actions 的主机是随机分配的，其公私钥并不是固定的。

```
- name: Scan public keys
  run: ssh-keyscan ${{ secrets.HOSTNAME }} >> ~/.ssh/known_hosts
```
通常在使用 ssh 连接到一台新主机上时，会提示是否保存相应主机的指纹信息，自动化部署中，我们是无法响应此请求的，所以我们需要将需要连接的主机的公钥存储在 Actions Runner 的 known_hosts 中去，这里使用 ssh-keyscan 命令将要部署主机的公钥添加到 known_hosts 中。这样连接到服务器就不会提示保存主机指纹信息。

```
- name: Setup Hugo
  uses: peaceiris/actions-hugo@v2
```
安装 Hugo 环境。


```
- name: Build
  run: hugo --gc --minify -b https://${{ secrets.HOSTNAME }}
```
编译博客源码生成 HTML 文件。

```
- name: Deploy
  run: rsync -avz --delete public/ ${{ secrets.USERNAME }}@${{ secrets.HOSTNAME }}:/var/www/html
```
使用 `rsync` 将编译的 HTML 文件同步到服务器的 `/var/www/html` 目录下。

## 设置 Secrets和环境变量

在上面的 workflows 文件中，有几个 `${{}}` 包裹的变量，其中`secrets`开头的变量是为了避免泄露隐私信息，例如私钥、登录用户名等新信息，因为 GitHub 公有仓库的文件任何人都可以查看。

这里使用 GitHub 提供的 Secrets 来隐藏隐私信息。在仓库的 Settings 中可以添加相应的值，运行时会替换 workflow 中的变量。

添加方法如下：在仓库的 Settings 页面选择 Secrets -> Actions，点击右上角的 `New repository secret` 添加 `Repository secrets`。在 Environments -> {ENV_NAME}，点击底部的 `Add secret` 添加 `Environment secrets`。

![GitHub Actions Repository Secrets.png](https://s2.loli.net/2022/12/06/jZiyarn6WeLfd8T.png)

![GitHub Actions Enviroments Secrets.png](https://s2.loli.net/2022/12/06/hDQnjSJGNg6iy9Y.png)

> 注意：Secrets 根据作用域不同有两种，分别是 `Repository secrets` 和 `Environment secrets`。`Repository secrets` 在不同环境中都生效，`Environment secrets` 只在指定的环境中生效，使用 `Environment secrets` 可以将不同分支的代码部署到不同的环境中。


而 `vars` 开头的变量是为了存储非敏感的配置数据。只有在此环境的上下文中，GitHub Actions 才能通过变量上下文访问这些变量。

## 服务器配置

部分Linux发行版默认没有安装`rsync`，提前安装好`rsync`

```shell
sudo apt update
sudo apt install rsync
```

出于安全考虑，通常会单独创建一个用于GitHub Action部署用的Linux用户。

```shell
# 创建用户 github，并设置家目录
sudo useradd -m -s /bin/bash github

# 设置密码（可选）
sudo passwd github

# 或者使用更安全的方式，不设置密码，仅通过SSH密钥访问
sudo passwd -l github
```

配置可以用于登录的SSH密钥

```shell
# 登录服务器，切换到 root 或具有 sudo 权限的用户
sudo su -

# 确保 authorized_keys 文件权限正确
sudo chown github:github /home/github/.ssh/authorized_keys
sudo chmod 600 /home/github/.ssh/authorized_keys
sudo chmod 700 /home/github/.ssh

echo YOUR_SSH_PUBLIC_KEY >> /home/github/.ssh/authorized_keys
```

然后给该用户授权用于部署静态页面的目录读写权限

```bash
sudo mkdir -p /var/www/html
sudo chmod -R 755 /var/www/html/
sudo chown -R github:github /var/www/html/
```

## 遇到的问题

- GitHub Actions 部署时，因为 GitHub Actions 的服务器在境外，腾讯云会发送异地登录警告邮件，无法自动部署。需要在腾讯云[主机安全控制台](https://console.cloud.tencent.com/cwp/manage/loginLog/loginwhitelist)中为用于部署的用户创建白名单。