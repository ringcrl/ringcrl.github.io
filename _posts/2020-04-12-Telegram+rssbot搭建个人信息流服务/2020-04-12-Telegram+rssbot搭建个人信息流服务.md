---
layout: post
title: Telegram+rssbot搭建个人信息流服务
tags: ['2020']
---

【即刻】走丢了很久，怀念她。想有一个渠道可以收敛日常生活中的所有通知，发现 Telegram 机器人配合 RSS 就很好的兼顾了阅读体验以及拓展支持，打开机器人对话框就能呈现一个完美的 Timeline。

方案：Telegram + 谷歌云 + flowerss-bot + RSSHub

![05.jpeg](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210504202236_535bd7187eeb7cb2053054a832088f4d.jpeg)

# Telegram 机器人申请

在 Telegram 搜索 @BotFather，发送 `/newbot` 新建一个你的专属机器人，得到 token 后面会用到

![01.jpeg](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210504202303_535bd7187eeb7cb2053054a832088f4d.jpeg)


# 申请谷歌云

因为网络通信原因，找了个可以白嫖的国外云服务厂商，首年免费，使用下来体验真的挺不错

https://console.cloud.google.com/?hl=zh-CN

![02.jpeg](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210504202317_cd3c1c59311d5b436c31e38f3f20ae61.jpeg)

![03.jpeg](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210504203723_d619e6ca494ff94aa28a89fab7a2c231.jpeg)

Compute Engine => 虚拟机实例 => 新建一个虚拟机，然后点击 ssh 登录上去

# rssbot 服务启动（弃用）

使用的是 debian 系统，使用以下操作启动 rssbot 服务

```sh
# 安装 wget、zip 等必要的包
sudo apt-get install wget
sudo apt-get install zip

# 下载 rssbot release 包
wget https://github.com/iovxw/rssbot/releases/download/v1.4.4/rssbot-v1.4.4-linux.zip

# 解压
unzip rssbot-v1.4.4-linux.zip

# 后台启动服务，这里填入上面生成的 token
nohup ./rssbot DATAFILE TELEGRAM-BOT-TOKEN &
```

下面就可以在 Telegram 机器人发送下面的指令了

```sh
/rss # 显示当前订阅的 RSS 列表
/sub # 订阅一个 RSS: /sub http://example.com/feed.xml
/unsub # 退订一个 RSS: /unsub http://example.com/feed.xml
```

# flowerss-bot 服务启动

https://github.com/indes/flowerss-bot 是一个更好用的机器人

- 在网页 ssh 登录上谷歌云实例后，进行下面操作
- Install Docker Engine on Debian：https://docs.docker.com/engine/install/debian/
- 按照 Docker 部署教程进行部署 https://github.com/indes/flowerss-bot

# RSSHub

https://docs.rsshub.app/，万物皆可 RSS

按照网页上的订阅方式就可以一波操作订阅各种源了

![04.jpeg](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210504203734_a023c01660c93b1a920d02bed72cf817.jpeg)
