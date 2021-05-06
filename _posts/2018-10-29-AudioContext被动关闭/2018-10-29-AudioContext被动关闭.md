---
layout: post
title: AudioContext被动关闭
tags: ["2018"]
---

使用 [howler.js](https://github.com/goldfire/howler.js) 的时候发现，设备被“打断”的场景下，声音就再也播不出来了。甚至在合作方的硬件设备上静置一段时间也出现同样的情况。

看 howler 代码的时候发现，`AudioContext` 只会被 new 一次，并且存在 `Holwer.ctx` 上：

```js
Howler.ctx = new AudioContext();
```

再次读取 `Howler.ctx` 的时候，如果已经有实例了，就使用之前那个实例。

这样做有个问题，就是遇到一些情况例如：来电话，或者系统为了节能做了一些优化，很容易出现之前的声音实例已经没办法再使设备发声了。

这时候需要为 Howler 加一个 dispose 方法，即可根据自己的业务需求复活声音了：

```js
// Because some devices seem to break the audioContext instance from hardware after being stationary for a while,
// the previous audioContext instance needs to be closed and reinitialized when used again
// Add a Howler.dispose to close the audioContext instance
Howler.ctx = undefined;
Howler.dispose = function () {
  if (Howler.ctx) {
    Howler.ctx.close();
    Howler.ctx = undefined;
  }
};
```
