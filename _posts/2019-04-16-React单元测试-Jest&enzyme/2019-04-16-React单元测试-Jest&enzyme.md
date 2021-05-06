---
layout: post
title: React单元测试-Jest&enzyme
tags: ['2019']
---


- 结合一个 Todo-List 的例子进行 React 单元测试
- 模板使用：React、Redux、Webpack@4、Babel@7


# 项目地址

https://github.com/ringcrl/react-todolist-enzyme

# 环境搭建

复习一下 React 环境搭建。

## 相关文档

- [webpack-loader](https://webpack.js.org/loaders)
- [Babel Usage Guide](https://babeljs.io/docs/en/usage)
- [Configuring Jest](https://jestjs.io/docs/en/configuration)
- [Using enzyme with Jest](https://airbnb.io/enzyme/docs/guides/jest.html)

## 依赖安装

```sh
# react
yarn add react react-dom react-redux redux

# babel
yarn add -D @babel/cli @babel/core @babel/polyfill @babel/preset-env @babel/preset-react babel-jest babel-loader

# webpack
yarn add -D webpack webpack-cli webpack-dev-server

# scss
yarn add -D node-sass sass-loader style-loader css-loader

# 测试套件
yarn add -D jest enzyme enzyme-adapter-react-16
```

## 环境配置

### babel.config.js

```js
const presets = [
  [
    '@babel/env',
    {
      targets: {
        edge: '17',
        firefox: '60',
        chrome: '67',
        safari: '11.1'
      }
    }
  ],
  ['@babel/preset-react']
];

module.exports = { presets };
```

### jest.config.js

```js
module.exports = {
  verbose: true,
  modulePaths: ['<rootDir>/__test__/'],
  moduleNameMapper: {
    '.(css|less)$': '<rootDir>/__test__/NullModule.js'
  },
  setupFilesAfterEnv: ['<rootDir>/__test__/setupTests.js'],
  transform: {
    '^.+\\.jsx?$': 'babel-jest'
  }
};
```

### webpack.config.js

```js
const webpack = require('webpack');
const env = process.env.NODE_ENV;
const path = require('path');

module.exports = {
  mode: 'development',
  devtool: 'eval',
  entry: ['./src/index.jsx'],
  output: {
    path: path.resolve(__dirname, 'dist'),
    filename: 'bundle.js',
    publicPath: '/dist'
  },
  module: {
    rules: [
      {
        test: /\.js|jsx$/,
        exclude: /node_modules/,
        use: {
          loader: 'babel-loader'
        }
      },
      {
        test: /\.scss$/,
        use: [
          {
            loader: 'style-loader' // 将 JS 字符串生成为 style 节点
          },
          {
            loader: 'css-loader' // 将 CSS 转化成 CommonJS 模块
          },
          {
            loader: 'sass-loader' // 将 Sass 编译成 CSS
          }
        ]
      }
    ]
  },
  resolve: {
    extensions: ['.jsx', '.js']
  },
  plugins: [
    new webpack.DefinePlugin({
      'process.env.NODE_ENV': JSON.stringify(env)
    })
  ]
};
```

### .eslintrc.js

```js
module.exports = {
  "parser": "babel-eslint",
  "parserOptions": {
      "ecmaVersion": 6,
      "sourceType": "module",
      "ecmaFeatures": {
          "jsx": true,
          "modules": true,
          "experimentalObjectRestSpread": true
      }
  },
  "env": {
    "browser": true,
    "commonjs": true,
    "es6": true,
    "node": true,
    "jest": true,
  },
  "globals": {
    "Raven": true, // 文件中注释 /* global Raven */
  },
  "extends": [
    "eslint:recommended", // https://cn.eslint.org/docs/rules/
    "plugin:react/recommended" // https://github.com/yannickcr/eslint-plugin-react
  ], 
  "parserOptions": {
    "ecmaVersion": 2018,
    "sourceType": "module"
  },
  "rules": {
    "prefer-const": ["error"],
    "no-var": ["error"],
    "linebreak-style": ["error", "unix"],
    "quotes": ["error", "single"],
    "semi": ["error", "always"],
    "no-control-regex": ["off"],
    "no-useless-escape": ["off"],
    "no-console": ["off"],
    "react/prop-types": ["off"],
  }
};
```

### .editorconfig

```js
[*]
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = false
insert_final_newline = true
indent_style = space
indent_size = 2

[{*.yml,*.json}]
indent_style = space
indent_size = 2
```

# Todo list

```
src
├── actions
│   └── index.js
├── components
│   ├── todo-creator
│   │   └── index.jsx
│   └── todo-item
│       └── index.jsx
├── constants
│   └── index.js
├── container
│   ├── index.jsx
│   └── index.scss
├── index.jsx
└── reducers
    ├── index.js
    └── todos.js
```

一个普通的 React、Redux 应用，效果如下

![01.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210505081014_c6d8898f747f95020603b4c73e5b0289.png)

- 回车后添加 Todo Item
- 点击 Todo Item 前面的 Input 删除此 Item

# `__test__`

## 文件概述

```
__test__
├── NullModule.js
├── components
│   ├── todo-creator
│   │   └── index.test.js
│   └── todo-item
│       └── index.test.js
├── reducers
│   └── todos.test.js
└── setupTests.js
```

- `NullModule.js` 导出的是空对象，用于跳过样式文件的检测，与 `jest.config.js` 里面的 `moduleNameMapper` 字段对应
- `setupTests.js` 用于使用 Jest 配置 Enzyme 和适配器，[文档](https://airbnb.io/enzyme/docs/guides/jest.html)

## 测试内容

- TodoCreator(输入框)
    - 正确渲染了 render input 输入框
    - 如果输入框有输入内容，回车后 addTodo reducer 被调用
    - 如果输入框没有输入内容，回车后 addTodo reducer 不被调用
    - addTodo 之后清空输入框
- TodoItem
    - Todo item 被正确渲染
    - input 改变时删除 item
    - 渲染 item 的 title
- reducers
    - 初始 state 正确
    - ADD_TODO 结果正确
    - DELETE_TODO 结果正确

具体逻辑看 `__tests__` 部分代码，主要使用了 enzyme 的类 JQ API，进行 "DOM" 检测、事件触发等操作。

```js
expect(wrapper.find('input').length).toBe(1)

wrapper.find('input').simulate('change');

expect(wrapper.find('.todo-title').text()).toBe(props.todo.text);

const mockEventObj = {
  key: 'Enter',
  target: {
    value: 'TEST'
  }
};
wrapper.find('input').simulate('keydown', mockEventObj);
expect(props.addTodo).toBeCalled();

// ...
```

## 测试结果

![02.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210505081035_98f5f8a4d080a0c1734a2b135ca23625.png)