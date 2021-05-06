---
layout: post
title: Safari-audio-policy
tags: ['2018']
---


[`New <video> Policies for iOS`](https://webkit.org/blog/6784/new-video-policies-for-ios/) 政策限制了音频必须由用户主动触发。
不过如果是自己的 APP 的话就可以通过修改 iOS 配置来禁用这个限制。


![01.jpeg](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210505082612_fdd9496e0c2fc4acc4fda6f3a8728d70.jpeg)

![02.jpeg](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210505082619_b9ee84fed5bf0ef4319eefab9a9b46c5.jpeg)

如图所示，通过配置 [mediaTypesRequiringUserActionForPlayback](https://developer.apple.com/documentation/webkit/wkwebviewconfiguration/1851524-mediatypesrequiringuseractionfor)，自己的 APP 中，Web 播放音频就不需要人为来触发了。

**注意：需要判断 iOS 10 以下，使用旧的 API。**
