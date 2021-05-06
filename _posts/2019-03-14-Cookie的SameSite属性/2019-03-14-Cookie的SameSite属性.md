---
layout: post
title: Cookie的SameSite属性
tags: ['2019']
---


翻译自：<https://medium.com/compass-security/samesite-cookie-attribute-33b3bfeaeb95>

- 17年的文章，是对 CSRF 很好的防御手段
- 理解了这个属性也能避免很多开发上的麻烦，例如 iFrame 中的 Cookie 处理

# 介绍

过去十年，我教我的学生五个 cookie 属性:“path, domain, expire, HttpOnly, Secure”。但是现在我们有了一个新属性 -SameSite。你知道新引入的 SameSite cookie 属性的细节吗？一种新的防止跨站点请求伪造（cross site request forgery）的 http 安全特性。该值可以设置为 `Strict` 或 `Lax`。

> SameSite示例:
> Set-Cookie: jsessionid=oIZEL75SLnw; HttpOnly; Secure; SameSite=Strict

下面会有 DEMO 演示，请确保您使用 FireFox 或 Chrome 等浏览器进行演示、比较结果。使用开发人员工具来观察网络堆栈。

# OWASP 的定义

SameSite 阻止浏览器将 cookie 与跨站点请求一起发送。主要目标是降低跨来源信息泄露的风险。它还提供了一些针对跨站点请求伪造攻击的保护。该标志的可能值为 `Lax` 或 `Strict`。

# 用例说明

假设您有一个电子银行网站，并且交易表单容易出现跨站点请求伪造漏洞。假设电子银行客户(受害者)已经通过身份验证(cookie)，攻击者让受害者点击 xsrf 链接，这将在创建非法交易。这种 xsrf 链接可以通过电子邮件、Twitter、Facebook、公共博客或任何其他基于超文本标记语言的系统进行传播。现代安全的电子银行解决方案可以使用随机 xsrf token 来抵御这种类型的攻击。但是为了这篇文章，让我们假设电子银行是脆弱的，并且不包含这样一个随机的xsrf令牌。

在 SameSite 之前，经过身份验证的受害者点击准备好的 xsrf 页面，然后进行交易。这是因为浏览器具有与电子银行的会话 cookie，因此会将会话 cookie 添加到电子银行交易请求中。但是如果浏览器不添加会话 cookie 呢？这正是 SameSite 所做的。如果链接来自外部站点，浏览器不会将cookie 添加到已通过身份验证的网站。在 `SameSite=Strict` 的情况下，浏览器一般不会添加 cookie。如果 `SameSite=Lax`，则如果用户单击顶级网址，浏览器将发送 cookie。做下面的演示，了解 `Strict` 和 `Lax` 的区别。

# Demo 页面

> 翻了墙也无法访问，网站已经坏掉了

为了 SameSite cookie 属性，我创建了一个简短的演示页面。它由两个网站组成。[第一页](http://www.bambut.ch/samesite.html)是设置一些 cookies，[第二页](http://www.glocken-emil.ch/samesitetest.html)是从第一个站点请求一个网址。因此，您现在可以用 Firefox、Chrome 或者 Safari 测试 cookies 是否被发送到[第一页](http://www.bambut.ch/)。

![01.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210505081501_3f7def04c81801d84d4c1e190b69945f.png)

# Chrome 测试

## 访问 demo1 页面

首先，您需要加载第一个演示页面，它设置了四个不同的 cookies。

![02.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210505081508_2785505b564f0972ba60dc859cd600aa.png)

请检查 cookies 是否已在 Chrome 中设置。使用“Application”选项卡中的内置开发工具。请看下图。

![03.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210505081514_627ac4b2c0d1c98efd5e489dc0d0b257.png)

> Cookie MyBamBut 没有设置 SameSite 属性 (过去 10 年前的设置)
> Cookie MyBamButNone 设置 SameSite=None
> Cookie MyBamButStrict 设置 SameSite=Strict
> Cookie MyBamButLax 设置 SameSite=Lax

Chrome 接受了四个 cookies 中的三个。SameSite 设置为“无”的 cookie 被拒绝。

## 查看第二个演示页面的 HTML

![04.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210505081521_5ff5059e5c084f5aafb4432d81c65978.png)

如您所见，第二页有一个顶级网址(href)，并从第一页的域名加载一个 `<img src>`。

## 访问第二个演示页面

使用与第 1 步和第 2 步相同的 Chrome 浏览器访问第二个演示页面。打开开发工具，点击“Network
”并重新加载页面。点击“printheaders.php”，检查哪些 cookies 是由 Chrome 发送的。分析请求标题。

![05.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210505081528_5f425175921a99eade8e8185d31e230e.png)

正如您在上面的图片中看到的，Chrome 只是添加了没有设置 SameSite 属性的 cookie。`SameSite=Strict` 和 `SameSite=Lax` 未发送到第一个演示页面。Cool — 这就是你想要的，一个很酷的 xsrf 防御手段。

## 访问第二个演示页面中的链接

现在让我们来看看如果你点击顶级链接，Chrome 是如何表现的。请点击第二个演示页面上的 `PrintHeaders` 链接，并按照步骤 3 再次分析请求头。

![06.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210505081535_0593ffd2d4325ef7d58139a659370c8e.png)

在下图中，您可以看到点击顶级网址时，第一个演示页面发送了哪些 Cookies。

![07.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210505081541_cb55e160fb6cdb15406a6d6298ccb3ad.png)

Chrome 将 MyBamButLax 添加到第一个演示页面的 HTTP 请求中。但是 MyBamButStrict 从未被发送到第一个演示页面。因此，设置`SameSite=Strict` 实际上是拒绝 `xsrf` 攻击。

# 结论

- SameSite cookie 属性对于防止跨站点请求伪造是一个很大的帮助。如果链接来自与源站点不同的地方，该值设置为“Strict”将阻止(较新的)浏览器添加 cookie，这在电子银行环境中非常酷。
- 此外，SameSite 属性尚未在所有主要浏览器版本中实现。截至 2017 年 11 月，相同的站点属性在 Chrome、Firefox 中实现

# 补充

原作者还漏了一种场景，在 iFrame 中使用的坑，把如下 `b.com` 页面当成 iFrame 嵌入 `a.com` 页面

```html
<body>
  <script>
    // 情况一：不设置
    document.cookie="b-cookie=b;domain=.b.com;path=/"
    document.cookie="b-cookie1=b1;domain=.b.com;path=/"
    console.log(document.cookie); // b-cookie=b;b-cookie1=b1;

    // 情况二：设置 SameSite
    document.cookie="b-cookie=b;domain=.b.com;path=/;SameSite=lax"
    document.cookie="b-cookie1=b1;domain=.b.com;path=/"
    console.log(document.cookie); // b-cookie1=b1;
  </script>
</body>
```
