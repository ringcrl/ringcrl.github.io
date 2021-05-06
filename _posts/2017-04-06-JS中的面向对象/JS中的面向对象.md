<!--JS-OO-->

面向对象并不是：封装、继承、多态，而是写代码的套路问题（定势思维）。

<!--more-->

# 面向对象解决的问题

- 封装：隐藏细节
- 继承：复用代码
- 多态：灵活，div 元素可以用 `console.dir` 来看所继承的

`HTMLDivElement => HTMLElement => Element => Node => EventTarget`

面向对象解决的问题：遇到任何问题都用一套思维去解决。

# 原型链（对象与对象之间的关系）

## 数据类型

1. 基本数据类型
    - number
    - string
    - boolean
    - undefined
    - null
    - symbol
2. 复杂数据类型
    - object
        - 普通对象
        - 数组
        - 函数

## 理解 `__proto__`

把 `__proto__` 理解为中文的【共】字

```js
const obj = {};
obj.__proto__ === Object.prototype;

const arr = [];
arr.__proto__ === Array.prototype;

const fn = function() {};
fn.__proto__ === Function.prototype;
```

# this（对象与函数之间的关系）

## 确定 this 是什么

`this` 是函数的第一个参数，`arguments` 是函数的其他参数。

- 参数的值只有在传参的时候才确定
- this 是函数第一个参数

this 是参数！！！this 的值只有在传参的时候才确定！！！

确定 this 的方法：

- 查文档
- 控制台

## 绑定 this 的方法

千万别用 _this、that 之类无聊的写法了

```js
button.onclick = function() {
  this.disabled = true;
  const btn = this;
  $.get('url', function() {
    btn.disabled = false;
  })
}
```

## 箭头函数中的 this

```js
const fn = (x) => {
  console.log(this);
  console.log(arguments);
  console.log(x);
}

fn.call(1, 2); // 1 被吃了，不存在的
// window
// undefined
// 2
```

## 通过 bind 绑定 this

bind 就是 call 的一种形式啊，连续用 call 一个月，不吃语法糖，就知道各种调用的原理啦！

```js
const obj = {
  name: 'Chenng'
}

function fn() {
  console.log(this);
}

const fnBindThis = fn.bind(obj);

// 等价于
function fnBindThis() {
  fn.call(obj);
}
```

# new 的用法

[JS 的 new 到底是干什么的？](https://zhuanlan.zhihu.com/p/23987456)

new 的出现为了批量创建对象。

## 对象是内存的关系

对象是存在内存中的，是没有名字的。

## 构造函数

返回一个新创造对象的函数，就叫做构造函数。

## 不使用 new 来创建函数与对象的关联

```js
function createSoldier(i) {
  const obj = {
    id: i,
    healthPoint: 42
  };
  obj.__proto__ = createSoldier.commonProperty;
  return obj;
}
createSoldier.commonProperty = {
  property: 'property',
  fn: function() {},
}
```

## 使用 new 来创建函数与对象的关联

```js
function Soldier(i) {
  this.id = i;
  this.healthPoint = 42;
}

// Soldier.prototype = {
//   constructor: Soldier, // 一个函数的 prototype 属性上面【默认】有个 constructor 属性
//   property: 'property',
//   fn: function() {},
// }

// 下面这种写法代替上面的注释内容更好
Soldier.prototype.property = 'property';
Soldier.prototype.fn = function() {};
```

## 构造函数习俗

- 构造函数首字母大写
- 构造函数可以省略 Create，因为它本身就是做 Create 的事情
- 构造函数不传参数，可以省略()

# 继承的写法

## 士兵与人类的构造函数

```js
function Human(options) {
  this.name = options.name;
  this.skin_color = options.skin_color;
}
Human.prototype.eat = function() {};
Human.prototype.sleep = function() {};

function Soldier(options) {
  Human.call(options);
  this.ID = options.ID;
  this.health_point = options.health_point;
}
Soldier.prototype.fight = function() {};
Soldier.prototype.defense = function() {};
```

## `__.proto__` 不能直接用

### Human 构造函数内部实现

```js
function Human() {
  // this = {}
  // this.__proto__ = Human.prototype
  // return this
}
```

### 士兵继承人类

```js
// 因为不能用 __proto__，但是实现的效果如下
Soldier.prototype.__proto__ = Human.prototype;

// 所以 Soldier 继承 Human 可以这么写
Soldier.prototype = new Human();
// Solider.prototype.__proto__ === this.__proto__【new Human() 返回 this】 === Human.prototype
```

### A.prototype = new B() 存在问题

```js
function Human(options) {
  this.name = options.name;
  this.skin_color = options.skin_color;
}
Human.prototype.eat = function() {};
Human.prototype.sleep = function() {};

function Soldier(options) {
  Human.call(this, options);
  this.ID = options.ID;
  this.health_point = options.health_point;
}
Soldier.prototype = new Human({name: '', skin_color: '', ID: '', health_point: ''});
Soldier.prototype.fight = function() {};
Soldier.prototype.defense = function() {};

var s = new Soldier({name: '', skin_color: '', ID: '', health_point: ''})
// s 上面有 name 和 skin_color 的特有属性，但是 s.__proto__ 上也有 name 和 skin_color 这两个特有属性
```

### 四行行圣杯方法

```js
function FakeHuman() {};
FakeHuman.prototype = Human.prototype;
Soldier.prototype = new FakeHuman();
Soldier.prototype.constructor = Soldier;
// 真正的 Soldier.prototype.__proto__ === Human.prototype
```

完整代码

```js
function Human(options) {
  this.name = options.name;
  this.skin_color = options.skin_color;
}
Human.prototype.eat = function() {};
Human.prototype.sleep = function() {};

function Soldier(options) {
  Human.call(this, options);
  this.ID = options.ID;
  this.health_point = options.health_point;
}
function FakeHuman() {};
FakeHuman.prototype = Human.prototype;
Soldier.prototype = new FakeHuman();
Soldier.prototype.constructor = Soldier; // prototype 上自带的 constructor 属性手动加回来
Soldier.prototype.fight = function() {};
Soldier.prototype.defense = function() {};

var s = new Soldier({name: '', skin_color: '', ID: '', health_point: ''})
```

### Object.create 实现圣杯方法

```js
Soldier.prototype = Object.create(Human.prototype);
// 真正的 Soldier.prototype.__proto__ === Human.prototype
```

## class 真正的类

```js
class Human {
  constructor(options) {
    this.name = options.name;
    this.skin_color = options.skin_color;
  }
  eat() {};
  sleep() {};
}

class Soldier extends Human {
  constructor(options) {
    super(options); // 相当于 Human，但是 Human 是个类，不能当函数用
    this.ID = options.ID;
    this.health_point = options.health_point;
  }
  fight() {};
  defense() {};
}

const s = new Soldier({name: '', skin_color: '', ID: '', health_point: ''});
```
