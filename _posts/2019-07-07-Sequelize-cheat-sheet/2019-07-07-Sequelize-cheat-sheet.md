---
layout: post
title: Sequelize-cheat-sheet
tags: ['2019']
---

最近需要用到 Mysql 来存一些前端 FusionData（前端定义的数据，存成字符串，自己 parse 后理解），选型就是 Sequelize 这个 Node.js ORM（对象关系映射）

文档：http://docs.sequelizejs.com/

# 安装

```sh
yarn add sequelize
yarn add mysql2
```

# 数据库连接

```js
const Sequelize = require('sequelize');

const sequelize = new Sequelize(
  'exe', // 数据库名称
  'root', // 用户名
  '', // 密码
  {
    host: 'localhost',
    dialect: 'mysql', // 'mysql'|'sqlite'|'postgres'|'mssql'
    operatorsAliases: false,

    pool: {
      max: 5,
      min: 0,
      acquire: 30000,
      idle: 10000
    }
  }
);

sequelize
  .authenticate()
  .then(() => {
    console.log('Connection has been established successfully.');
  })
  .catch(err => {
    console.error('Unable to connect to the database:', err);
  });
```

# 定义模型

```js
sequelize.define(
  'tableName',
  {
    firstName: {
      type: Sequelize.STRING
    },
    lastName: {
      type: Sequelize.STRING
    }
  },
  { timestamps: false }
);
```

# 增删改查

## 增

```js
const user = await UserModel.create({
  firstName: "Sue",
  lastName: "Smith"
});
console.log(user.get({'plain': true}));
```

## 查

### 查询单条

```js
// id 查询
const user = await UserModel.findById(1);

// 条件查询
const user = await UserModel.findOne({
  where: { firstName: 'Sue' }
});
```

### 查询多条

```js
// 条件查询
const users = await UserModel.findAll({
  where: {
    id: [1, 2],
    firstName: 'John'
  }
});

// AND 条件
const Op = Sequelize.Op;
const users = await UserModel.findAll({
  where: {
    [Op.and]: [
      { id: [1, 2] },
      { firstName: 'John' }
    ]
  }
});

// OR 条件
const Op = Sequelize.Op;
const users = await UserModel.findAll({
  where: {
    [Op.or]: [
      { id: [1, 2] },
      { firstName: 'John' }
    ]
  }
});

// NOT 条件
const Op = Sequelize.Op;
const users = await UserModel.findAll({
  where: {
    [Op.not]: [
      { id: [1, 2] }
    ]
  }
});
```

### 排序与分页

```js
// 排序
const users = await UserModel.findAll({
  order: [
    ['id', 'DESC']
  ]
});

// 分页
let countPerPage = 2, currentPage = 1;

const users = await UserModel.findAll({
  limit: countPerPage, // 每页多少条
  offset: countPerPage * (currentPage - 1) // 跳过多少条
});
```

## 改

```js
const user = await UserModel.findOne({
  where: { firstName: 'Sue' }
});

const affectedRows = await user.update({ firstName: 'King' });
```

## 删

```js
const user = await UserModel.findOne({
  where: { firstName: 'Sue' }
});

const affectedRows = await user.destroy();
```

## 复合操作

```js
// 查 + 改
const Op = Sequelize.Op;
const affectedRows = await UserModel.update(
  { firstName: 'King' },
  {
    where: {
      [Op.not]: { firstName: null }
    }
  }
);
console.log(affectedRows);

// 查 + 删
const affectedRows = await UserModel.destroy({
  where: { firstName: 'King' }
});
```