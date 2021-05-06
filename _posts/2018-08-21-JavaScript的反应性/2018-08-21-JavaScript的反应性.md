---
layout: post
title: JavaScript的反应性
tags: ["2018"]
---

反应性并不是 JavaScript 编程通常的运行方式，Vue 的反应性是如何实现的？

原文地址：<https://mp.weixin.qq.com/s/Wm5-3hsqre7ft_f0YBnoeg>

# 程序基本运行逻辑

```js
let price = 5;
let quantity = 2;
let total = price * quantity;
price = 20;
console.log(total); // 是 10，然而我们希望得到新的值 40
```

# 依赖类

```js
let data = {
  price: 5,
  quantity: 2,
};
let target = null;

// Dependence 类
class Dep {
  constructor() {
    this.subscribers = [];
  }
  depend() {
    if (target && !this.subscribers.includes(target)) {
      this.subscribers.push(target);
    }
  }
  notify() {
    this.subscribers.forEach((sub) => sub());
  }
}

// 将数据变成可观察的
Object.keys(data).forEach((key) => {
  let internalValue = data[key];

  const dep = new Dep();

  Object.defineProperty(data, key, {
    get() {
      dep.depend();
      return internalValue;
    },
    set(newVal) {
      internalValue = newVal;
      dep.depend();
    },
  });
});

function watcher(myFunc) {
  target = myFunc;
  target();
  target = null;
}

watcher(() => {
  data.total = data.price * data.quantity;
});
```

结果：

```js
data.total; // 10
data.price = 20;
data.total; // 40
data.quantity = 3;
data.total; // 60
```
