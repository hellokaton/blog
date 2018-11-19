---
layout: post
title: 在 Ubuntu 上安装 Rocket.Chat
tags: ["ubuntu", "vps"]
---

[Rocket.Chat](https://github.com/RocketChat/Rocket.Chat){:target="_blank"} 是一种类似 Slack 的开源聊天软件，当然你可能没用过 Slack，毕竟它在国内不流行，这名字听起来像是 “火箭聊天”，非常霸气啊！不过光开源这一项就很吸引我了，同道中人同道中人。

> 观看 [视频版](https://www.youtube.com/watch?v=iaAot5K2sps){:target="_blank"} 教程。

# Rocket.Chat 特性

- 群组聊天
- 直接通信
- 私聊群
- 桌面通知
- 媒体嵌入
- 链接预览
- 文件上传
- 语音/视频聊天
- 截图
- 支持你目前使用的任何平台

这篇文章中我将在 `Ubuntu 18.04 LTS` 上安装 Rocket.Chat，使用 Nginx 作为反向代理，同时还会配置 SSL 证书。

# 准备环境

你需要有以下环境：

- 一台 Ubuntu 的服务器
- 一个域名（非必须，最好提供）
- 你的双手

> 如果有域名在你的网站域名商处添加一条 DNS 的 A 记录指向服务器。

# 安装 Rocket.Chat

开始前我们先更新下操作系统

```shell
sudo apt update && sudo apt upgrade
```

安装 `Rocket.Chat` 最快的方法是使用它的 `Snap`。`Snap` 是 `Linux` 系统上一种软件包管理的方式。它类似一个容器拥有一个应用程序所有的文件和库，各个应用程序之间完全独立。所以使用snap包的好处就是它解决了应用程序之间的依赖问题，使应用程序之间更容易管理。在 `Ubuntu 16.04 LTS` 以上版本的系统都内置了。

1. 安装 Rocket.Chat

```shell
sudo snap install rocketchat-server
```

2. 安装后，Rocket.Chat 服务会自动启动，检查一下是否在运行：

```shell
sudo service snap.rocketchat-server.rocketchat-server status
```

你可以访问 [Rocket.Chat snap](https://rocket.chat/docs/installation/manual-installation/ubuntu/snaps/){:target="_blank"} 查看一些其他命令。

# 使用 Nginx 反向代理

## 安装 Nginx

```shell
sudo apt install -y nginx
```

**启动 nginx**

```shell
sudo systemctl start nginx
# 开机自动启动
sudo systemctl enable nginx
```

## 设置反向代理

**禁用默认欢迎页**

默认的欢迎页配置文件位置在 `/etc/nginx/sites-enabled/default`。实际上真正的位置是 `/etc/nginx/sites-available/`，只不过用了软连接。

```shell
sudo ls -l /etc/nginx/sites-enabled
total 0
lrwxrwxrwx 1 root root 34 Aug 16 18:20 default -> /etc/nginx/sites-available/default
```

删除欢迎页

```shell
sudo rm /etc/nginx/sites-enabled/default
```

**创建反向代理配置**

默认可能没安装 `vim`，可以安装 `sudo apt-get install -y vim`，然后创建自己的配置文件：

```shell
sudo vim /etc/nginx/sites-available/rocketchat.conf
```

内容如下：

```conf
server {
    listen 80;

    server_name example.com;

    location / {
        proxy_pass http://localhost:3000/;
    }
}
```

> 将这里的 `example.com` 修改为你自己的域名。

通过以下方式创建指向它的链接来启用新配置 `/etc/nginx/sites-available/`：

```shell
sudo ln -s /etc/nginx/sites-available/rocketchat.conf /etc/nginx/sites-enabled/
```

测试配置是否有误

```shell
sudo nginx -t
```

重新加载 nginx 配置

```shell
sudo nginx -s reload
```

## 配置 SSL 证书

申请证书的方式很多，这里我们使用的是免费的 [Let's Encrypt](https://letsencrypt.org/){:target="_blank"} 的 SSL 证书。有人写了一个名为 [Certbot](https://certbot.eff.org/){:target="_blank"} 的工具让我们可以轻松的获得证书。

**安装 Certbot**

```shell
sudo apt-get install -y software-properties-common
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install -y python-certbot-nginx
sudo certbot --nginx
```

Certbot 会询问有关该网站的信息。在执行 `sudo apt-get install -y python-certbot-nginx` 的时候会询问位置信息，我们选择 亚洲（也就是 `6. Asia`），时区选择 `69. Shanghai` 即可。

在执行 `sudo certbot --nginx` 的时候会询问你的邮箱，填写和你注册域名时相同的邮箱，其他询问同意即可。

**开启证书自动续约**

我们的证书有效期是 3 个月，不过 certbot 很聪明，有一个自动续约的功能，执行以下命令即可：

```shell
sudo certbot renew --dry-run
```

# 开始使用 Rocket.Chat

第一次打开会看到这个界面

![Rocket.Chat 设置]({{ "/public/images/2018/11/rocketchat01.png" | prepend: site.cdnurl }} "Rocket.Chat 设置")

这里需要设置一下管理员信息，成功后就可以登录啦！

![Rocket.Chat 登录]({{ "/public/images/2018/11/rocketchat02.png" | prepend: site.cdnurl }} "Rocket.Chat 登录")

好了，到这里已经安装完毕了，Rocket.Chat 的更多有趣的玩法需要你自己体验，起飞吧少年！

# 参考

- [Rocket.Chat in Ubuntu](https://rocket.chat/docs/installation/manual-installation/ubuntu/){:target="_blank"}
- [Nginx on Ubuntu](https://certbot.eff.org/lets-encrypt/ubuntuxenial-nginx.html){:target="_blank"}