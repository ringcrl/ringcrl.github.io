---
layout: post
title: KeyEvents与输入抖动问题
tags: ['2019']
---

- 输入框的长度会随着已输入字符长度变化，为什么会出现抖动？
- 为什么 scratch 的键盘事件要绑成这样子？

![03.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210504210759_addcb22e4f302f7fcfef99dcb26f77e4.png)


# 输入抖动问题

在积木中输入框会根据已输入字符的长度更新输入框的长度，输入框其实是一个 input，盖在了 SVG 之上，并根据 SVG 的长度计算最新长度并更新：

![04.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210504210823_3305f9209ccc669447ac912c8f06e6ec.png)

![05.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210504210830_bfa64b6d75dc8ccd4f19ee51c280c282.png)

![06.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210504210837_6e2f9be0158b593ea32834ab94fd868a.png)

但最近发现个问题，由于 input 的 CSS 属性是 `text-align: center;`，值改变之后再更新长度居中，视觉上会出现先往左，再居中的“抖一下”效果。

通过观察样式更新的方法，发现更改宽度的方法比值更改的方法慢了 `100ms` 左右的时间触发，查看了一下更改宽度的方法绑在了 `keyup` 事件之上，而 `keyup` 事件的触发比真正改变输入框值的方法要慢很多，远远大于 `16ms` 的人眼无法察觉的视觉变化时间，那输入框改变值之后触发的第一个方法是什么呢？

# key-evnets

W3C 中对键盘事件的【定义】：https://w3c.github.io/uievents/#event-type-keydown

做个小 DEMO 来实验一下，对 input 框绑定了 `keydown`、`keyup`、`keypress`、`input` 事件

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Key-Events</title>
</head>
<body>
  <input id="input" type="text">
  <script>
    const input = document.querySelector('input');
    input.addEventListener('keydown', (e) => {
      console.log('keydown', e.target.value);
    });
    input.addEventListener('keyup', (e) => {
      console.log('keyup', e.target.value);
    });
    input.addEventListener('keypress', (e) => {
      console.log('keypress', e.target.value);
    });
    input.addEventListener('input', (e) => {
      console.log('input', e.target.value);
    });
  </script>
</body>
</html>
```

## 短按一下

![02.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210504210847_ecd28bc4bed0b29e929f45267a6635eb.png)

- 触发的顺序是：keydown => keypress => input => keyup
- 注意：keydown、keypress 钩子触发的时候，还是没有赋值的，真正赋值是在 input 之后


## 长按

![01.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210504210853_57c566a389e26f6678a8a059b1136e65.png)

- 触发循序是：keydown => keypress => input => keydown(n) => keyup
- 注意：长按的表现是在 input 之后插入 n 个 keydown 直至手放开后触发 keyup

## 不同键盘的表现

测试了不同的键盘包括 MBP 的自带键盘，机械键盘，长按的表现还是不一样的，不过和本问题无关，先留意这个点，日后有相关的问题查阅 W3C 的【规定】进行解决。

# 总结

- 对于实时性要求高的输入事件例如根据输入改变输入框宽度的场景，把事件绑定在 input 上，因为 keyup 的延时时间会让人有抖动的感觉
- 对于键盘事件要了解短按、长按的不同表现区别，最好的方法是去 W3C 查阅【规定】，一般浏览器厂商或是硬件厂商都会尽可能的遵守这个【规定】
