---
layout: post
title: 编写webpack-loader
tags: ["2018"]
---

我们用过很多的 webpack loader 例如：`babel-loader`、`less-loader`、`css-loader` 之类的，如何写一个我们自己的 loader 呢？

# webpack.config.js

```js
const path = require("path");

module.exports = {
  mode: "development",
  resolveLoader: {
    modules: [
      "node_modules",
      // 本地的路径，把所有 xxx-loader.js 丢到这个目录下
      // 如果已经为 loader 创建了独立的包，则使用 yarn link 开发
      path.resolve(__dirname, "loaders"),
    ],
  },
  devtool: "source-map",
  entry: "./index.js",
  output: {
    path: path.resolve(__dirname, "dist"),
    filename: "boundle.js",
  },
  module: {
    rules: [
      // loader 默认从右向左执行、从下到上执行，但是可以通过配置 force 选项强制改变
      {
        test: /\.js$/,
        use: {
          loader: "my-babel-loader",
          options: {
            presets: ["@babel/preset-env"],
          },
        },
      },
    ],
  },
};
```

# my-babel-loader.js

```js
// 一些工具方法，最常用的是获取传递给 loader 的选项
const loaderUtils = require("loader-utils");
// 校验传递参数的格式
const validateOptions = require("schema-utils");

const babel = require("@babel/core");

// 参数骨架：https://www.npmjs.com/package/schema-utils
const schema = {
  type: "object",
  properties: {
    presets: {
      type: "array",
    },
  },
};

/**
 *
 * @param {string} source 包含文件内容的字符串
 * @return {string|buffer}
 */
function loader(source) {
  const options = loaderUtils.getOptions(this);

  validateOptions(schema, options, "my-babel-loader");

  // 对于异步的操作，需要在这个方法中手动阻塞 loader 执行
  // 异步操作结束之后调用 cb 让 loader 继续执行
  const cb = this.async();

  babel.transform(
    source,
    {
      ...options,
      filename: this.resourcePath.split("/").pop(),
      // 调试源码使用，和 webpack.config.js 里面的配置 devtool: 'source-map' 不是一个东西
      sourceMaps: true,
    },
    (err, result) => {
      cb(err, result.code, result.map);
    }
  );
}

module.exports = loader;
```

# 其他 loader 的 demo

## file-loader

```js
const loaderUtils = require("loader-utils");
function loader(source) {
  const filename = loaderUtils.interpolateName(this, "[hash].[ext]", {
    content: source,
  });
  this.emitFile(filename, source); // 把文件发射出去

  return `module.export="${filename}"`;
}
loader.raw = true; // 使用二进制处理

module.exports = loader;
```

## url-loader

这个 loader 有个特点，文件大小在限制范围内就转成 base64。

```js
const loaderUtils = require("loader-utils");
const mime = require("mime");
function loader(source) {
  const { limit } = loaderUtils.getOptions(this);
  if (limit && limit > source.length) {
    return `modele.exports=data:${mime.getType(this.resourcePath)};base64,${
      source.toString * "base64"
    }`;
  } else {
    require("./file-loader").call(this, source);
  }
}
loader.raw = true; // 使用二进制处理

module.exports = loader;
```

## style-loader

```js
function loader(source) {
  let str = `
    let style = document.createElement('style');
    style.innerHTML = ${JSON.stringify(source)};
    document.head.appendChild(style);
  `;
  return str;
}
module.exports = loader;
```
