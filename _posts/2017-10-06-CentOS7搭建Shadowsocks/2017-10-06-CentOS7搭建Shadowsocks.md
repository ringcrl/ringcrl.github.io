---
layout: post
title: CentOS7搭建Shadowsocks
tags: ["2017"]
---

一觉睡醒，发现 Shadowsocks 所有节点都超时了，只好在自己的 vultr 服务器上面搭一个了。

# 安装 Python 相关的工具

```
sudo yum install python-pip
pip install --upgrade pip

sudo yum install python-devel
sudo yum install openssl-devel
```

# 使用 pip 安装 ss、加密依赖包

```
sudo pip install shadowsocks
sudo pip install M2Crypto
```

# 配置 /etc/shadowsocks.json 文件

```
{
    "server": "45.76.182.255",
    "server_port": 15160,
    "local_address": "127.0.0.1",
    "local_port": 1080,
    "password": "my_password",
    "timeout": 300,
    "method": "aes-256-cfb"
}
```

服务器端口号可以随意（大于 1024），本地端口默认 1080（socks5 的国际惯例）。加密方式也可以选其他的，aes256 安全性比较高，但可能因此在加解密上花更多时间，增加延迟。

# 系统防火墙设置

防火墙打开服务器端口，这样 ss 才能工作。

```
firewall-cmd --zone=public --add-port=15160/tcp --permanent
```

# ss 启动与停止

```
sudo ssserver -c /etc/shadowsocks.json -d start
sudo ssserver -c /etc/shadowsocks.json -d stop
```
