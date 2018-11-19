---
layout: post
title: 在 Google Cloud 上安装 Gogs 
tags: ["git", "vps"]
---

为啥选 Google Cloud 呢？主要原因是他们家有香港和台湾的服务器，速度和价格来讲都比较好，但是他们的 Web 面板操作真心复杂。下面来看看如何操作吧！

如果你没有域名，请跳过安装 Nginx 和域名配置的选项，请确保防火墙配置正常，需开放如下端口：

- 22
- 80、443
- 3000（测试用）

# 允许 SSH 登录

默认情况在 GCP（Google Cloud Platform）开的实例只能在 Web 端点击界面上的 `SSH` 登录（主要是为了安全），有些人习惯在本地使用 SSH 登陆就需要做一些配置，这里我开了一台 CentOS7 的机器。

允许 SSH 登陆有两种方式，网上有一种说法是设置 ROOT 密码然后修改 SSH 配置，让我们可以实现本地登陆，这种方式相对而言不安全，在 GCP 中支持 SSH 公钥登陆的方式，我来介绍下。

首先在 `Compute Engine -> 元数据` 选项里添加自己的 SSH 公钥。

<img src='{{ "/public/images/2018/11/gcp_meta_data.png" | prepend: site.cdnurl }}'  alt="gcp meta data" width="400px"/>

![gcp 添加公钥]({{ "/public/images/2018/11/gcp_add_rsa_pub.png" | prepend: site.cdnurl }} "gcp 添加公钥")

将自己的公钥帖进去保存即可：

```shell
cat ~/.ssh/id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAAvTjxXeOkDBmYl9BCcdKqYA1Cw6EjxKr58MVJNIIjzPGc0dCpt2CDe9qLzw+v+/KOhDd+6t0mwVeVyDLJjgCgxw== hello@example.com
```

> 如果你还没听过 SSH 公钥登陆，可以看看 [这篇](https://blog.biezhi.me/2017/08/ssh-no-password-login.html) 文章。

这时候就可以在本地登陆了，用户名不是 ROOT，而是你刚才添加 SSH 公钥后左边的这个用户名

![gcp meta data]({{ "/public/images/2018/11/gcp_login_name.png" | prepend: site.cdnurl }} "gcp meta data")

所以我们要这样登陆

```shell
ssh 用户名@123.123.123.123
```

要进入到 ROOT 权限只需要输入 `sudo su` 即可。

# 安装 Nginx

安装 Nginx 我使用的是 [lnmp](https://github.com/lj2007331/lnmp){:target="_blank"} 的一键安装，可以直接配置 SSL 证书以及其他的软件安装。

```shell
yum -y install wget screen python
wget http://mirrors.linuxeye.com/lnmp-full.tar.gz
cd lnmp && screen -S lnmp
./install.sh
```

下面就开始安装了，安装过程中需注意：

- SSH 端口是默认的 22
- 不开启 iptables（这里我们使用 GCP 的 Web 防火墙管理即可）
- WebServer 选择 Nginx
- 其他的组件都不用安装，选择 `N`

安装成功后会提示你重启机器，重启即可！

然后访问 `http://IP` 会看到这个界面：

![gcp meta data]({{ "/public/images/2018/11/lnmp_web.png" | prepend: site.cdnurl }} "gcp meta data")

在左侧的菜单中点击可以看到各个脚本的操作指南。

# 配置域名绑定

首选在域名服务端上配置一条 DNS 记录，如域名 `git.example.com` 指向你的服务器 IP（A 记录）。

然后我们在命令行添加虚拟主机：

```shell
cd lnmp
./vhost.sh
#######################################################################
#       OneinStack for CentOS/RedHat 6+ Debian 7+ and Ubuntu 12+      #
#       For more information please visit https://oneinstack.com      #
#######################################################################

What Are You Doing?
	1. Use HTTP Only
	2. Use your own SSL Certificate and Key
	3. Use Let's Encrypt to Create SSL Certificate and Key
	q. Exit

# 这里我选择 3 生成一个 Let's Encrypt的免费证书
Please input the correct option: 3
```

下面依次输入你的域名信息即可，其他均可选默认。完成后访问你的域名 `https://git.example.com` 

# 安装 Gogs

先创建一个新用户 `git`，我们使用它来操作 `gogs`。

```shell
useradd -m -s /bin/bash git
passwd git
```

给 `git` 用户设置一个登陆密码。

```shell
su - git
cd /home/git/
wget https://dl.gogs.io/0.11.66/gogs_0.11.66_linux_amd64.tar.gz
tar -zxvf gogs_0.11.66_linux_amd64.tar.gz
rm -f gogs_0.11.66_linux_amd64.tar.gz
```

启动 Gogs 尝试

```shell
cd gogs
./gogs web
```

访问 `http://ip:3000` 你应该可以看到这个界面：

![Gogs 安装]({{ "/public/images/2018/11/gogs_install.png" | prepend: site.cdnurl }} "Gogs 安装")

到这里证明我们的 Gogs 运行没问题了，有域名的话这里先别安装。

## 配置 Nginx 反向代理

这里我们需要用域名安装，所以先配置一下域名访问 Gogs，修改 nginx 的配置：

```shell
vi /usr/local/nginx/conf/vhost/git.example.com.conf
```

下面是我的配置，你可以参考：

```conf
server {
  listen 80;
  listen 443 ssl http2;
  ssl_certificate /usr/local/nginx/conf/ssl/git.example.com.crt;
  ssl_certificate_key /usr/local/nginx/conf/ssl/git.example.com.key;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_ciphers EECDH+CHACHA20:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
  ssl_prefer_server_ciphers on;
  ssl_session_timeout 10m;
  ssl_session_cache builtin:1000 shared:SSL:10m;
  ssl_buffer_size 1400;
  add_header Strict-Transport-Security max-age=15768000;
  ssl_stapling on;
  ssl_stapling_verify on;
  server_name git.example.com;
  access_log off;
  index index.html index.htm index.php;
  root /data/wwwroot/git.example.com;
 
  include /usr/local/nginx/conf/rewrite/none.conf;

  location / {
    proxy_pass http://localhost:3000;
  }

}
```

修改后让配置生效：

```shell
service nginx reload
```

现在我们使用 `git` 用户将 Gogs 启动起来，然后访问你的域名就可以安装了！

## 配置 Gogs

**数据库设置**

Gogs 支持多种数据库，我这里偷懒没有装 MySQL 直接用的 SQLite，如果你想使用 MySQL 的话可以先安装然后设置用户名等信息。

**应用基本设置**

这里需要注意的是 _运行系统用户_ 是 `git`，**域名** 填写你自己的域名，**应用 URL** 填写你的域名 `https://git.example.com/`，其他默认即可。

**邮件服务设置**

如果你准备了邮件服务器的话可以配置这里，当然这个配置以后也可以修改的，假设你使用的是 163 的邮箱。

<img src='{{ "/public/images/2018/11/gogs_smtp.png" | prepend: site.cdnurl }}'  alt="Gogs SMTP" width="600px"/>

注册 Gogs 的第一个用户就是管理员！

# 开机启动

现在我们启动 gogs 服务的方式并不方便，使用 `ctrl + c` 后服务就会停掉，所以我们需要将这个服务运行在后台，方式也很多，比如使用 `nohup`、`screen`、`supervisor` 等。

这里我不介绍它们，直接用 systemd 服务来完成，先创建一个新的服务：

```shell
vi /etc/systemd/system/gogs.service
```

```shell
[Unit]
 Description=Gogs
 After=syslog.target
 After=network.target
 After=mariadb.service mysqld.service postgresql.service memcached.service redis.service

 [Service]
 # Modify these two values and uncomment them if you have
 # repos with lots of files and get an HTTP error 500 because
 # of that
 ###
 #LimitMEMLOCK=infinity
 #LimitNOFILE=65535
 Type=simple
 User=git
 Group=git
 WorkingDirectory=/home/git/gogs
 ExecStart=/home/git/gogs/gogs web
 Restart=always
 Environment=USER=git HOME=/home/git

 [Install]
 WantedBy=multi-user.target
```

编辑后保存，然后重新加载 systemd 服务：

```shell
systemctl daemon-reload
```

启动 gogs 服务并使用 systemctl 命令在启动运行它。

```shell
systemctl start gogs
systemctl enable gogs
```

你可以使用如下命令检查：

- `lsof -i :3000`：检查端口是否启动
- `systemctl status gogs`：检查服务是否启动

```shell
[root@gogs ~]# lsof -i :3000
COMMAND PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
gogs    722  git    7u  IPv6  13950      0t0  TCP *:hbci (LISTEN)

[root@gogs ~]# systemctl status gogs
● gogs.service - Gogs
   Loaded: loaded (/etc/systemd/system/gogs.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2018-11-12 11:50:43 CST; 3h 9min ago
 Main PID: 722 (gogs)
   CGroup: /system.slice/gogs.service
           └─722 /home/git/gogs/gogs web

Nov 12 14:57:15 gogs gogs[722]: [Macaron] [Static] Serving /assets/octicons-4.3.0/octicons.woff2
Nov 12 14:57:15 gogs gogs[722]: [Macaron] 2018-11-12 14:57:15: Completed GET /assets/octicons-4.3.0/octicons.woff2?ef21c39f0ca9b1b5116e5eb7…in 78.941µs
Nov 12 14:57:15 gogs gogs[722]: [Macaron] 2018-11-12 14:57:15: Started GET /css/themes/default/assets/fonts/outline-icons.woff2 for 127.0.0.1
Nov 12 14:57:15 gogs gogs[722]: [Macaron] [Static] Serving /css/themes/default/assets/fonts/outline-icons.woff2
Nov 12 14:57:15 gogs gogs[722]: [Macaron] 2018-11-12 14:57:15: Completed GET /css/themes/default/assets/fonts/outline-icons.woff2 304 Not M…n 136.904µs
Nov 12 14:57:15 gogs gogs[722]: [Macaron] 2018-11-12 14:57:15: Started GET /img/favicon.png for [::1]
Nov 12 14:57:15 gogs gogs[722]: [Macaron] [Static] Serving /img/favicon.png
Nov 12 14:57:15 gogs gogs[722]: [Macaron] 2018-11-12 14:57:15: Completed GET /img/favicon.png 200 OK in 996.151µs
Nov 12 14:57:18 gogs gogs[722]: [Macaron] 2018-11-12 14:57:18: Started GET / for 127.0.0.1
Nov 12 14:57:18 gogs gogs[722]: [Macaron] 2018-11-12 14:57:18: Completed GET / 200 OK in 18.146843ms
Hint: Some lines were ellipsized, use -l to show in full.
```

好了，下载 Gogs 已经安装成功，接下来就按它的文档去使用吧！如果你在安装过程中有什么疑问可以留言告诉我。

# 参考

- [gogs](https://gogs.io){:target="_blank"}
