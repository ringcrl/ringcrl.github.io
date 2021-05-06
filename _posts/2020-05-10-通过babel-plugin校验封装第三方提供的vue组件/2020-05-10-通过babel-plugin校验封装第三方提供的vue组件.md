---
layout: post
title: 通过babel-plugin校验封装第三方提供的vue组件
tags: ['2020']
---

之前就了解过 babel AST 相关的知识，刚好一个业务需求需要校验与修改第三方提供的 vue 组件，用 babel-plugin 来做这件事再合适不过了。

# 需求背景

- 三方提供互动的 vue 组件，如：https://m.v.qq.com/txi/dev
- 三方只编写基本的样式逻辑，最后需要在 vue 组件中**注入默认 props、注入 mixins 提供与引擎通信的能力、并将 vue 组件挂在到 window.\_interactComps 下**

![01.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210430081018_2942f3eb8745a37e3795db46ea579e84.png)

# 预期结果

## 转换前的 vue 代码

Bubble.vue

```vue
<style>
.bubble {
  position: absolute;
  display: table;
}
</style>

<template>
  <div
    class="bubble"
    v-show="visible"
    @click="onClick"
    :style="{
      top: y + '%',
      left: x + '%',
      width: width + '%',
      height: height + '%',
    }"
  ></div>
</template>

<script>
export default {
  data() {
    return {};
  },
  props: {
    action: {
      type: Object,
      default: function _default() {
        return {};
      },
    },
  },
  mounted() {
    this.bindListener();
  },
  methods: {
    bindListener() {
      this.bridge.on("videoTimeUpdate", (time) => {
        if (time >= this.startTime) {
          this.show();
        } else {
          this.hide();
        }
      });
    },
    onClick() {
      console.log("on bubble click!");
    },
  },
};
</script>
```

## 转换后的 vue 代码

截取部分 JS

```js
// 注入 mixin
const mixin = {
  created() {
    /* eslint-disable-next-line */
    this.bridge = iBridge.getBridge(this._uid);
  },
  data() {
    return {
      visible: false,
    };
  },
  methods: {
    show() {
      if (this.visible) return;
      this.visible = true;
      if (this.root === "true") {
        this.bridge.showInteract();
        this.bridge.report({
          type: "show",
          compid: this.nodeid,
          tplid: this.tplid,
        });
      }
    },
    proxy(codeHash, func) {
      const context = this;
      return function (e) {
        func(e);
        let touch = e.touches && e.touches[0];
        if (!touch) {
          touch = e.changedTouches && e.changedTouches[0];
        }
        if (!touch) {
          touch = e;
        }

        let widgetId = codeHash;
        let { target } = e;
        while (target && target.dataset) {
          const { report } = target.dataset;
          if (report && typeof report === "string") {
            widgetId = report;
            break;
          }
          target = target.parentNode;
        }
        const pPos = context.bridge.px2Percent({
          x: touch.pageX || 0,
          y: touch.pageY || 0,
        });
        context.bridge.report({
          type: e.type,
          x: pPos.x,
          y: pPos.y,
          compid: context.nodeid,
          tplid: context.tplid,
          widgetid: widgetId,
        });
      };
    },
  },
  // ...
};

const VBubble = {
  mixins: [mixin],
  props: {
    action: {
      type: Object,
      default: function _default() {
        return {};
      },
    },
    // 注入 default props
    x: {
      type: Number,
      default: 0,
    },
    y: {
      type: Number,
      default: 0,
    },
    width: {
      type: Number,
      default: 10,
    },
    height: {
      type: Number,
      default: 5,
    },
    startTime: {
      type: Number,
      default: 0,
    },
    endTime: {
      type: Number,
      default: 0,
    },
  },
  // ...
};

// vue 组件注入到全局，提供后面互动 SDK，Vue.component('xxx', comp) 的形式加载组件
window._interactComps = Object.assign(window._interactComps || {}, { VBubble });

export default VBubble;
```

对比转换前与转换后的代码可以看到，主要的改动是：**注入默认 props、注入 mixins 提供与引擎通信的能力、并将 vue 组件挂在到 window.\_interactComps 下**

# babel-plugin 编写

## webpack 配置

提供给第三方一个 babel-plugin，他们本地正常开发 vue 组件逻辑，不用关心 defaultProps、mixins、全局变量注入，在打包时帮他们处理好即可。

```js
module.exports = (env) => {
  return {
    // ...
    module: {
      rules: [
        {
          test: /\.vue$/,
          loader: "vue-loader",
        },
        // 它会应用到普通的 `.js` 文件
        // 以及 `.vue` 文件中的 `<script>` 块
        {
          test: /\.js$/,
          use: {
            loader: "babel-loader",
            options: {
              // 引入 babel-plugin-ivcomponent 插件
              plugins: ["./lib/babel-plugin-ivcomponent.js"],
              presets: ["@babel/preset-env"],
            },
          },
        },
      ],
    },
  };
};
```

## plugin 功能拆解

### 入口

```js
// babel-plugin-ivcomponent.js
const visitor = {
  ExportDefaultDeclaration(path) {
    // 检测到 export default {} 代码会进入到这个方法
  },
};

module.exports = () => ({
  visitor,
});
```

### 添加 props

```js
const template = require("@babel/template").default;

const addDefaultProps = (path) => {
  const defaultPropsTmpl = `const defaultProps = {
    x: {
      type: Number,
      default: 0,
    },
    y: {
      type: Number,
      default: 0,
    },
    width: {
      type: Number,
      default: 10,
    },
    height: {
      type: Number,
      default: 5,
    },
    startTime: {
      type: Number,
      default: 0,
    },
    endTime: {
      type: Number,
      default: 0,
    },
  }`;

  // 获取 defaultProps 的 AST properties
  const astDefaultPropsProperties = template.ast(defaultPropsTmpl)
    .declarations[0].init.properties;

  // 获取 vue 组件中的 props 属性，如果 vue 组件中没有写，则手动添加一个 props: {}
  const { properties } = path.node.declaration;
  let propsPerperties = properties.find((item) => item.key.name === "props");
  if (!propsPerperties) {
    // 如果没有定义 props 属性，手动添加 props: {}
    propsPerperties = t.objectProperty(
      t.identifier("props"),
      t.objectExpression([])
    );
    properties.push(propsPerperties);
  }

  // 将默认的 props 添加到 props: { ... }
  propsPerperties.value.properties.push(...astDefaultPropsProperties);
};
```

### 添加 mixins

```js
const addMixin = (path) => {
  const { properties } = path.node.declaration;

  // 禁止第三方使用 mixins 属性
  let mixinsProperty = properties.find((item) => item.key.name === "mixins");
  if (mixinsProperty) {
    throw new ReferenceError("组件不允许使用 mixins 属性");
  }

  mixinsProperty = t.objectProperty(
    t.identifier("mixins"),
    t.arrayExpression([t.identifier("mixin")])
  );

  properties.push(mixinsProperty);

  // 最后插入了 mixins: [mixin]，那 mixin 这个变量从哪里来呢？看下面
};
```

### mixin 变量插入，export default 转换

```js
// 用于把 AST 转换成字符串代码
const generate = require("@babel/generator").default;

const getMixinTmpl = () => {
  return `
    const mixin = {
      ...
    }
  `;
};

const codeTransform = (path) => {
  const VFileName = "VBubble";

  let { code } = generate(path.node);

  // 替换 export default 为 const VBubble = {}
  code = code.replace("export default", `const ${VFileName} =`);

  // 代码上方加入 const mixin = {}
  code = getMixinTmpl() + code;

  // 将 vue 组件注入到全局，供互动 JSSDK 使用
  code += `window._interactComps = Object.assign(window._interactComps || {}, { ${VFileName} });`;
  code += `export default ${VFileName}`;

  // 将字符串代码转回 AST，把当前的 ExportDefaultDeclaration 整个替换掉
  const resultAst = babelParser.parse(code, {
    sourceType: "module",
  });
  path.replaceWithMultiple(resultAst.program.body);
};
```

### 禁止 import 其他 JS

```js
const visitor = {
  ExportDefaultDeclaration(path) {
    // ...
  },
  // 发现 import 声明语句的时候报错
  ImportDeclaration() {
    throw new ReferenceError("组件不允许使用 import 语法");
  },
};

module.exports = () => ({
  visitor,
});
```

## 完整地址

https://github.com/ringcrl/babel-plugin-ivcomponent/blob/master/lib/babel-plugin-ivcomponent.js

# babel-plugin 基础

- [Babel 插件手册](https://github.com/jamiebuilds/babel-handbook/blob/master/translations/zh-Hans/plugin-handbook.md)
- [babel-traverse](https://babeljs.io/docs/en/babel-traverse)
- [babel-template](https://babeljs.io/docs/en/babel-template)
- [babel-types](https://babeljs.io/docs/en/babel-types)
- [babel-generator](https://babeljs.io/docs/en/babel-generator)
- [AST Explorer](https://astexplorer.net/)

## babel-traverse

用于遍历代码，找到需要修改的位置，例如上面的代码：

```js
const visitor = {
  ExportDefaultDeclaration(path) {
    // 代码中包含 export default 声明语句的时候会调用，通过 path.node 拿到具体内容
  },
  ImportDeclaration() {
    // 代码中包含 import 声明语句的时候会调用
  },
};

module.exports = () => ({
  visitor,
});
```

## babel-template

开发非常友好的，写一段代码字符即可拿到转换得到这段代码的 AST，在原 AST 中进行插入替换很方便

```js
const name = "my-module";
const mod = "myModule";

const ast = template.ast`
  var ${mod} = require("${name}");
`;
```

## babel-types

功能主要有两种：

- 一方面可以用它验证 AST 节点的类型，例如使用 isClassMethod 或 assertClassMethod 方法可以判断 AST 节点是否为 class 中的一个 method
- 另一方面可以用它构建 AST 节点，例如调用 classMethod 方法，可生成一个新的 classMethod 类型 AST 节点

例如我要生成一个 `mixins: [mixin]` 的属性表达式插入到对象中

```js
mixinsProperty = t.objectProperty(
  t.identifier("mixins"),
  t.arrayExpression([t.identifier("mixin")])
);

properties.push(mixinsProperty);
```

## babel-generator

有时候直接写 AST 比较麻烦，也可以直接写字符串来生成 AST

```js
import {parse} from '@babel/parser';
import generate from '@babel/generator';

const code = 'class Example {}';
const ast = parse(code);

const output = generate(ast, { /* options */ }, code);
```

## AST Explorer

![02.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210430081045_4925a297ef65d47f4e9a9e30e9866eac.png)

AST Explorer 可以看到代码转 AST 的样子，这时候如果我们需要注入一个 props，查询 babel-types 的方法，就可以知道这样来写：

```js
propsPerperties = t.objectProperty(
  t.identifier("props"),
  t.objectExpression([])
);
properties.push(propsPerperties);
```

# And more!

![03.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210430081101_593c1a926989f1b0e1ff7d78139ea2ac.png)

文档最后有一句话：And more，这真是个充满想象力的工具。
