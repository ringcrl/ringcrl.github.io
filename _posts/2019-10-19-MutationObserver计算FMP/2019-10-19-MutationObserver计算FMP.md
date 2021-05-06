---
layout: post
title: MutationObserver计算FMP
tags: ['2019']
---

- FMP 即 First Meaningful Paint，有意义的首屏时间
- 通过 MutationObserver 观察 DOM 结构变化以及视口范围计算实现


## 实现方案

```js
const details = [];
const ignoreEleList = ['script', 'style', 'link', 'br'];
let observeDom;
let firstScreenTiming;

// 查看当前元素的祖先元素是否在数组中
function isEleInArray(target, arr) {
  if (!target || target === document.documentElement) {
    return false;
  } else if (arr.indexOf(target) !== -1) {
    return true;
  } else {
    return isEleInArray(target.parentElement, arr);
  }
}

function isInFirstScreen(target) {
  if (!target || !target.getBoundingClientRect) return false;

  const rect = target.getBoundingClientRect();
  const screenHeight = window.innerHeight;
  const screenWidth = window.innerWidth;
  return rect.left >= 0
    && rect.left < screenWidth
    && rect.top >= 0
    && rect.top < screenHeight;
}

function updateTiming() {
  if (observeDom) {
    observeDom.disconnect();
  }
  for (let i = 0; i < details.length; i++) {
    const detail = details[i]
    for (let j = 0; j < detail.roots.length; j++) {
      if (isInFirstScreen(detail.roots[j])) {
        firstScreenTiming = detail.time;
        break;
      }
    }
    if (typeof firstScreenTiming === 'number') {
      break;
    }
  }
  console.log('ccc firstScreenTiming', firstScreenTiming);
}

if (window.MutationObserver) {
  observeDom = new MutationObserver((mutations => {
    if (!mutations || !mutations.forEach) return;
    const detail = {
      time: performance.now(),
      roots: [],
    };

    mutations.forEach(mutation => {
      if (!mutation || !mutation.addedNodes || !mutation.addedNodes.forEach) return;

      mutation.addedNodes.forEach(ele => {
        if (ele.nodeType === 1 && ignoreEleList.indexOf(ele.nodeName.toLocaleLowerCase()) === -1) {
          if (!isEleInArray(ele, detail.roots)) {
            detail.roots.push(ele);
          }
        }
      });
    });

    if (detail.roots.length) {
      details.push(detail);
    }
  }));

  observeDom.observe(document, {
    childList: true,
    subtree: true,
  });
}

window.addEventListener('load', function () {
  updateTiming()
});
```

## 效果

![01.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210504203859_b566df8480d7aa8b145575a8ef083f16.png)

由图可见，页面是 Node 直出，在页面 DOM 元素开始渲染的时候会触发 MutationObserver 事件，即可获取到渲染事件。