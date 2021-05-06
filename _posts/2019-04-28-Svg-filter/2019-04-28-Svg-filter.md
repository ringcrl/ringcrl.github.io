---
layout: post
title: Svg-filter
tags: ['2019']
---

- filter 的原理与参数
- 如何实现一个积木拖起后的加边框加阴影效果

![01.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210504205929_729bdf190c1dec071cfd72d3be01ff98.png)


# filter 原理与参数

## 原理

- 使用了 filter 的 svg们不会将图案直接渲染为最终图形，会渲染图案的像素到临时位图中
- 由 filter 指定的操作会被应用到该临时区域，其结果会被渲染为最终图形
- filter 标记之间就是执行我们想要操作的滤镜基元，每个基元有一个或多个输入，但只有一个输出
- 基元的输入可以是原始图形(SourceGraphic)、图形的不透明度通道(SourceAlpha)、前一个滤镜基元的输出，只有对图形的形状感兴趣而不管颜色时，不透明度通道是有用的，不透明度通道会和颜色相互作用

```html
<filter id="drop-shadow">
  <!-- 这里是滤镜操作 -->
</filter>
<g id="spring-flower" style="url(#drop-shadow);">
  <!-- 这里绘制图形 -->
</g>
```

## 滤镜边界

- 描述该滤镜的剪裁区域
- 默认值 `x="-10%" y="-10%" width="120%" height="120%"`，为滤镜提供了额外的空间，这样造成的投影，产生的输出就会比输入大
- 这些属性是按照滤镜对象的边界框来计算的，即 filterUnits 的 默认值是 objectBoundingBox
- 可以用 primitiveUnits 属性为用于滤镜基元中的单元指定单位，默认值是 userSpaceOnUse，如果设置为 objectBoundingBox，就可以按照图形尺寸的百分比来表示单位

## feGaussianBlur

- in 指定输入源，`SourceAlpha` 指的是原图
- `stdDeviation` 指定模糊度，数值越大越模糊，如果提供两个由空格分割的数字，则分别代表 x 方向、y 方向的模糊度

```xml
<defs>
  <filter id="drop-shadow">
    <feGaussianBlur in="SourceAlpha" stdDeviation="2">
  </filter>
</defs>
<g id="flower" filter="url(#drop-shadow)">
  <!-- 这里绘制花朵 -->
</g>
```

## 存储、连接、合并滤镜

- result 属性指定当前基元的结果，稍后可以通过 `blur` 引用，给定的名称是一个局部名称，只在包含该基元的 `<filter>` 中有效
- `feOffset` 基元接受它的输入，这里就是 Gaussian blur 的返回结果 blur，它的偏移由 dx 和 dy 值指定，将结果位图存储在 `offsetBlur` 名字下面
- `<feMerge>` 基元包裹一个 `<feMergeNode>` 元素列表，其中每个元素都指定一个输入，这些输入按照出现的顺序一个堆叠在另一个上面，这里我们希望 offsetBlur 在原始值 SourceGraphic 下面

```html
<filter id="drop-shadow">
  <feGaussianBlur in="SourceAlpha" stdDeviation="2" result="blur"/>
  <feOffset in="blur" dx="4" dy="4" result="offsetBlur"/>
  <feMerge>
    <feMergeNode in="offsetBlur"/>
    <feMergeNode in="SourceGraphic"/>
  </feMerge>
</filter>
```

## feColorMatrix

- 允许修改任意像素点的颜色或者阿尔法值
- type 属性为 matrix 的时候，values 4 行 5 列，表四行代表计算 R、G、B、A
- 每行中的数字分别乘以输入像素的 R、G、B、A和常量 1（按照列的顺序），得到输出值
- 若不指定 result，表示用作下一个基元的隐形输入

```html
<filter id="glow">
  <feColorMatrix type="matrix"
    values=
        "0 0 0 0   0
          0 0 0 0.9 0 
          0 0 0 0.9 0 
          0 0 0 1   0"/>
  <feGaussianBlur stdDeviation="2.5"
    result="coloredBlur"/>
  <feMerge>
    <feMergeNode in="coloredBlur"/>
    <feMergeNode in="SourceGraphic"/>
  </feMerge>
</filter>
```

## feComponentTransfer

- 提供一种更方便、更灵活的方式来单独操作每个颜色分量
    - 可以让蓝色更亮，也可以通过增加绿色和红色级别让它没那么强烈
- 通过内置 feFuncR、 feFuncG、feFuncB、feFuncA 调整红绿蓝和阿尔法的级别
    - 每个元素可以指定一个 type 说明如何修改该通道，所有结果大于 1.0 被减少为 1.0，小于 0 被调整为 0
        - linear：把当前颜色值分量放到公式 `slope * C + intercept` 中，intercept 为结果提供一个基准值，slope 是一个简单的比例因子。`<feFuncB type="linear" slope="3" intercept="0.2">`
        - table：将颜色值划分为一系列相等的间隔，每个间隙中的值都相应地扩大，类似于最小的四分之一颜色范围的值加倍，下一个四分之一都塞入一个十分之一的范围，保持第三个四分之一的范围不变，最后一个四分之一的值塞入剩下的 15% 的颜色范围中 `<feFuncG type="table" tableValues="0.0, 0.5, 0.6, 0.85, 1.0" />`

## feComposite

- 接受两个输入源，分别指定在 in 和 in2 属性中
- operator 属性用于设置如何合并这两个输入源
    - over：`<feComposite operator="over" in="A" in2="B" />` 生成的结果 A 层叠在 B 上面，`<feMergeNode>` 仅仅是制定 over 操作的 `feComposite` 元素的一种便利的快捷方式
    - in：`<feComposite operator="in" in="A" in2="B" />` 结果是 A 的一部分重叠在 B 的不透明区域，类似于蒙版效果，但这个蒙版仅仅基于 B 的阿尔法通道，而不是它的颜色亮度
    - out：`<feComposite operator="out" in="A" in2="B" />`，结果是 A 的一部分位于 B 的不透明区域的外部
    - atop：`feComposite operator="atop" in="A" in2="B" />`，结果是 A 的一部分位于 B 里面，B 的一部分在 A 外面
    - xor：`<feComposite operator="xor" in="A" in2="B" />`，结果包含位于 B 的外面的 A 的部分和位于 A 的外面的 B 的部分

## feBlend

- 接受两个输入源，分别指定在 in 和 in2 属性中
- mode 属性用于设置如何混合输入源
    - normal：只有 B
    - multiply：对于每个颜色通道，将 A 的值和 B 的值想成，由于颜色值在 0~1 之间，相乘会让它们更小，这会加深颜色，如果某个颜色是白色则没有效果
    - screen：把每个通道的颜色值加载一起，然后减去它们乘积，明亮颜色或者浅色往往回避暗色占优势，但相似亮度的颜色会被合并
    - darken：取 A 和 B 的每个通道的最小值，颜色较暗
    - lighten：提取 A 和 B 的每个通道的最大值，颜色较亮

## feFlood 和 feTile

- feFlood 提供一个纯色区域用于组合或者合并，提供 flood-color 和 flood-opacity 属性
- feTile 提取输入信息作为团，然后横向和纵向平铺填充滤镜指定的区域

## filter 动画

- 通过 attributeName 指定变化的属性，from、to、begin、dur 指定变化的值
- 详见 [SVG 动画部分](https://static.chenng.cn/#/%E5%9F%BA%E7%A1%80-%E5%89%8D%E7%AB%AF/SVG/SVG)

```html
<feGaussianBlur result="outShadowAnimate" in="outColor" stdDeviation="3">
  <animate id="increase" attributeType="XML" attributeName="stdDeviation" from="3" to="10" begin="0s;decrease.end" dur="0.4s" />
  <animate id="decrease" attributeType="XML" attributeName="stdDeviation" from="10" to="3" begin="increase.end" dur="0.4s" />
</feGaussianBlur>
```

# 实践

## 描边 + 阴影

<iframe height="265" style="width: 100%;" scrolling="no" title="stroke + shadow" src="//codepen.io/ringcrl/embed/ZZPLLY/?height=265&theme-id=0&default-tab=html,result" frameborder="no" allowtransparency="true" allowfullscreen="true">
  See the Pen <a href='https://codepen.io/ringcrl/pen/ZZPLLY/'>stroke + shadow</a> by ringcrl
  (<a href='https://codepen.io/ringcrl'>@ringcrl</a>) on <a href='https://codepen.io'>CodePen</a>.
</iframe>

## 呼吸动画

<iframe height="265" style="width: 100%;" scrolling="no" title="Animate Filter" src="//codepen.io/ringcrl/embed/QPoKBp/?height=265&theme-id=0&default-tab=html,result" frameborder="no" allowtransparency="true" allowfullscreen="true">
  See the Pen <a href='https://codepen.io/ringcrl/pen/QPoKBp/'>Animate Filter</a> by ringcrl
  (<a href='https://codepen.io/ringcrl'>@ringcrl</a>) on <a href='https://codepen.io'>CodePen</a>.
</iframe>
