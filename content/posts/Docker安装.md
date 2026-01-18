---
title: Debian 安装 Docker
date: 2020-10-17
tags: 
- Docker
- Debian
---

## 安装 Docker

1. 安装必要依赖

```sh
sudo apt update
sudo apt install ca-certificates curl
```

2. 安装GPG证书

```sh
sudo install -m 0755 -d /etc/apt/keyrings

# 阿里云镜像
sudo curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/debian/gpg -o /etc/apt/keyrings/docker.asc

# 腾讯云镜像
sudo curl -fsSL https://mirrors.cloud.tencent.com/docker-ce/linux/debian/gpg -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

3. 写入软件源信息

```sh
# 阿里云镜像
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://mirrors.aliyun.com/docker-ce/linux/debian/
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

```sh
# 腾讯云镜像
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://mirrors.cloud.tencent.com/docker-ce/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

4. 安装 Docker Engine

```sh
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## Docker 非 root 用户运行

Docker 运行需要 root 权限，如果你不想使用 root 账户运行 Docker，可以把非 root 用户添加到 docker 用户组当中，即可使用非 root 用户运行 Docker。

1. 创建 `docker` 用户组.

```sh
sudo groupadd docker
```

2. 把非 root 用户添加到 `docker` 用户组.

```sh
sudo usermod -aG docker $USER
```

3. 将用户添加到docker组后，注销并重新登录，使更改生效。或者，运行以下命令，对组进行更改：

```sh
newgrp docker
```

4. 测试 Docker 是否安装正确

```shell
docker run hello-world
```

## 镜像加速

由于国内网络的原因，使用官方的镜像源下载 Docker 镜像的速度非常慢，使用加速器可以提升获取 Docker 官方镜像的速度。本文介绍使用阿里云和腾讯云提供的镜像源来加速拉取镜像的速度。

### 阿里云镜像配置

阿里云提供的镜像服务需要我们登录阿里云的账号，我们首先登录阿里云，而后到[以下链接](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)获取加速器地址

在该页面可以获取到你的加速器地址，然后可以通过修改 `/etc/docker/daemon.json` 配置文件来使用加速器，修改后的 `daemon.json` 文件内容如下

```shell
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://xxxxx.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 腾讯云镜像配置

1. 执行以下命令，打开 `/etc/docker/daemon.json` 配置文件。

```sh
sudo vim /etc/docker/daemon.json
```

2. 添加以下内容，并保存

```json
{
   "registry-mirrors": [
       "https://mirror.ccs.tencentyun.com"
  ]
}
```

3. 执行以下命令，重启 Docker 即可。

```sh
sudo systemctl restart docker
```
## 参考资料

- [Install Docker Engine on Debian](https://docs.docker.com/engine/install/debian/)
