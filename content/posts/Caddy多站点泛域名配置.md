---
title: "Caddy 自动申请泛域名证书，多站点配置"
date: 2023-01-09T14:01:51+08:00
tags:
- Caddy
- 泛域名证书
- Caddy多站点
slug: caddy-wildcard-certificate
---

目前比较流行的 Web 服务器主要有：Nginx、Apache、Caddy 等，其中 Nginx 以高性能著称，但是 Nginx 的配置文件比较复杂冗长，非专业人员想要玩转 Nginx 还是有一定难度的。

最初我也是使用 Nginx 作为 Web 服务器，使用免费的 `Let’s Encrypt` 证书，申请的证书每隔90天就要重新续约一次，非常麻烦。偶然的机会，了解到了 Caddy 这个使用 Golang 开发的 Web 服务器，被其简洁的配置文件、自动 HTTPS 的特性吸引了，于是在重装服务器后，毅然决定投入 Caddy 的怀抱。

## Caddy 安装

因为我们要使用 DNS API 来自动申请 HTTPS 证书，我们要安装包含对应 DNS 提供商插件的二进制文件。这里以 Tencent Cloud DNS (DNSPod) 为例，在[下载页面](https://caddyserver.com/download)搜索选择 `caddy-dns/tencentcloud`，然后点击 `Download` 下载包含 DNSPod 插件的 Caddy 二进制文件，如下图所示。

![下载Caddy](https://s2.loli.net/2026/01/18/mcFEVautxObCyTh.png)

然后需要将 Caddy 配置为系统服务，确保 Caddy 一直运行。将下载的二进制文件移动到 `$PATH` 下，例如：

```bash
sudo mv caddy /usr/bin/
```

测试是否正常工作：

```bash
caddy version
```

创建一个名为 `caddy` 用户组：

```bash
sudo groupadd --system caddy
```

创建一个名为 `caddy` 的用户，同时创建具有可读权限的主目录

```bash
sudo useradd --system \
    --gid caddy \
    --create-home \
    --home-dir /var/lib/caddy \
    --shell /usr/sbin/nologin \
    --comment "Caddy web server" \
    caddy
```

创建Caddy Server配置文件

```bash
sudo mkdir /etc/caddy
sudo touch /etc/caddy/Caddyfile
```

然后，我们将使用 `systemd` 来创建系统服务，在 `/etc/systemd/system/` 目录下创建 `caddy.service` 文件

```bash
sudo vim /etc/systemd/system/caddy.service
```

文件内容如下：

```ini
[Unit]
Description=Caddy
Documentation=https://caddyserver.com/docs/
After=network.target network-online.target
Requires=network-online.target

[Service]
Type=notify
User=caddy
Group=caddy
ExecStart=/usr/bin/caddy run --environ --config /etc/caddy/Caddyfile
ExecReload=/usr/bin/caddy reload --config /etc/caddy/Caddyfile --force
TimeoutStopSec=5s
LimitNOFILE=1048576
LimitNPROC=512
PrivateDevices=yes
PrivateTmp=true
ProtectSystem=full
AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

在保存服务配置文件后，使用 `systemctl` 命令启动服务：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now caddy
```

验证是否正在运行

```bash
sudo systemctl status caddy
```

按照上面的步骤如果 `/etc/caddy/Caddyfile` 中没有正确配置，这里会启动失败，暂时先忽略，后文会介绍如何配置 `Caddy` 并给出一个模板，正确配置 `Caddyfile` 后再重启服务即可。

## 自动 HTTPS 配置

获得公众信任的 TLS 证书需要来自公众信任的第三方机构的验证。Caddy 使用 ACME 协议自动验证，Caddy 支持三种方式：`HTTP challenge`、`TLS-ALPN challenge` 和 `DNS challenge`。使用泛域名证书，必须要用 `DNS challenge` 来验证。

> 关于 ACME Challenge 可以查看[官方文档](https://caddyserver.com/docs/automatic-https#acme-challenges)

这里我以我使用的 Tencent Cloud DNS (DNSPod) 为例，在下载页面点击 Tencent Cloud DNS (DNSPod) 插件的名称，进入到插件的 GitHub 页面，查看相应的配置文档。Tencent Cloud DNS (DNSPod) 插件需要提供 `TENCENTCLOUD_SECRET_ID` 和 `TENCENTCLOUD_SECRET_KEY`。

进入 TENCENTCLOUD 控制台，打开 `API 密钥管理` 页面，创建新的密钥。

![创建密钥](https://s2.loli.net/2026/01/18/ClRVQIHhpjGe86N.png)

最后在需要配置的站点的 TLS 配置中配置 DNS 插件，例如：

```Caddyfile
tls {
    dns tencentcloud {
        secret_id {env.TENCENTCLOUD_SECRET_ID}
        secret_key {env.TENCENTCLOUD_SECRET_KEY}
    }
    protocols tls1.2 tls1.3

    ciphers TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256 TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384 TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256 TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256
}
```

这里我还指定了支持 TLS 协议使用的协议版本和加密算法，使用的配置为 [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/) 的 `Intermediate` 配置。

完成以上配置后，Caddy 就可以自动完成 TLS 证书的申请和配置，显然比配置 Nginx 要简单太多。

## 泛域名多站点配置

泛域名证书又名通配符证书，泛域名证书是 SSL 证书的一种形式，它是指可以保护一个域名所有子域名的 SSL 证书，例如 `*.example.com` 可以保护包括 `a.example.com` 、`b.example.com` 在内的等二级域名。

通常情况下，我们一个域名下会有多个子域名，并且有可能这多个子域名的站点运行在同一台服务器上。Caddy 可以很方便的自动申请并配置泛域名证书，这里我给出一个完整的 `Caddyfile` 为例：

```Caddyfile
{
    admin off
    email example@mail.com
}

*.example.com example.com {
    header / Strict-Transport-Security "max-age=63072000"

    encode gzip

    tls {
        dns tencentcloud {
            secret_id {env.TENCENTCLOUD_SECRET_ID}
            secret_key {env.TENCENTCLOUD_SECRET_KEY}
        }
        protocols tls1.2 tls1.3

        ciphers TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256 TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256 TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384 TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256 TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305_SHA256
    }

    @root host example.com
    handle @root {
        redir https://www.example.com{uri}
    }

    @www host www.example.com
    handle @www {
        root * /var/www/html

        file_server
    }

    @a host a.example.com
    handle @a {
        reverse_proxy 127.0.0.1:8080
    }

    @b host b.example.com
    handle @b {
        reverse_proxy 127.0.0.1:8081
    }

    handle {
        abort
    }
}
```

在这个例子中，我一共有3个子域名 `www`，`a` 和 `b`，裸域名将 `www` 子域名下，`a` 子域名和 `b` 子域名将反向代理本机的 8080 和 8081 端口。

> 全局代码块中，email 是用于申请证书的邮箱，注意替换成自己的邮箱地址。

> 站点代码块的括号外的 `*.example.com example.com`，注意一定要添加 `example.com` 否则裸域名无法使用。

有关泛域名的相关具体配置，可以参考 [Caddy Documentation](https://caddyserver.com/docs/caddyfile/patterns#wildcard-certificates) 中的相关部分。

## 验证站点证书配置

保存 Caddy 配置文件，重新启动 Caddy Server

```bash
sudo systemctl restart caddy.service
```

在浏览器中访问你的站点，点击地址栏中的 URL 左边的锁标志，选择 `连接是安全的`，然后点击`证书有效` 查看站点的证书是否正确配置，如下图我的博客已经使用了泛域名证书

![Wildcard certificate](https://s2.loli.net/2023/01/13/vqIihLE9zYjUyQu.png)

然后可以使用 [SSL Labs](https://www.ssllabs.com/ssltest/index.html) 来查看我们的服务器 SSL 设置兼容性如何，我这里使用的[Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/) 的 `Intermediate` 配置在 `SSL Labs` 可以获得 `A+` 的评分，基本可以兼容绝大多数设备。

![SSL Labs Summary](https://s2.loli.net/2023/01/13/9nLpgNtJOBWZ1XY.png)

## 更新 Caddy Server

下载最新版本的 Caddy Server，然后停止正在运行的 Caddy Server，并删除 Caddy Server。

```sh
sudo systemctl disable caddy
sudo systemctl stop caddy.service
sudo rm /usr/bin/caddy
```

将下载的 Caddy Server 移动到 `/usr/bin` 目录下，修改可执行文件的所有者并修改权限。

```sh
sudo mv caddy /usr/bin/caddy
sudo chown root:root /usr/bin/caddy
sudo chmod 755 /usr/bin/caddy
```

重新启动 Caddy Server。

```sh
sudo systemctl enable caddy
sudo systemctl start caddy
```