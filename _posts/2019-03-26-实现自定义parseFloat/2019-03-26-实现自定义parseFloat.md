
---
layout: post
title: 实现自定义parseFloat
tags: ['2019']
---

一个很不常见的需求

- 支持超出 17 位的数字（Number.MAX_SAFE_INTEGER）的展示
- 大数不转为科学计数法
- 各种正负号、000、科学计数法输入、数字加字符、字符加数字的 edge case 处理
- 全部使用正则处理，待进行 Benchmark 后才能确定能否用于生产
- 完全基于 TDD 开发

![01.png](https://cdn-1257430323.cos.ap-guangzhou.myqcloud.com/assets/imgs/20210505081213_9e5b2a68a79ef5feea30161a1d58c05b.png)

```js
/**
 * 测试用例，待完善
 */
const testCase = [
  '-0.123',
  '-.123',
  '--123',
  '44444444444444444444',
  '-5555555555555555555',
  '.1234567891234567891',
  '12345678910',
  '12345678910123456789100',
  '0.123456789',
  '123456789123456789123456789123456789123456789123456789',
  '5e+3',
  '5e-5',
  '5e55',
  '123456felix',
  '123456felix123456yang123456',
  'felix123456',
  'felix123456yang123456is123456poor',
  '+123',
  '123 123',
  '123.123.123',
  '0000123',
  '-0000123',
  '-0000.123',
  '+0000123',
  '+0000.123',
  '-.123',
];
const expect = [
  '-0.123',
  '-0.123',
  undefined,
  '44444444444444444444',
  '-5555555555555555555',
  '0.1234567891234567891',
  '12345678910',
  '12345678910123456789100',
  '0.123456789',
  '123456789123456789123456789123456789123456789123456789',
  '5000',
  '0.00005',
  '5e+55',
  '123456',
  '123456',
  undefined,
  undefined,
  '123',
  '123',
  '123.123',
  '123',
  '-123',
  '-0.123',
  '123',
  '0.123',
  '-0.123',
];

function test() {
  let failIndex = 0;
  let successCount = 0;
  let failResult;
  const passed = testCase.every((item, index) => {
    if (numberHandler(item) === expect[index]) {
      successCount++;
      return true;
    }
    failIndex  = index;
    failResult = numberHandler(item);
    return false;
  });
  if (passed) {
    console.log(`测试通过，通过 ${successCount} 条测试用例`);
  } else {
    console.log(`
      第 ${failIndex + 1} 条用例失败
      输入：${testCase[failIndex]}
      预期：${expect[failIndex]}
      结果：${failResult}
    `);
  }
}

test();

/**
 * 
 * @param {string} numStr 
 */
function numberHandler(numStr) {
  // 如果不理解下面为什么这样处理
  // 可以尝试去掉，看看哪个用例过不了

  if (isInvalid(numStr)) {
    return undefined;
  }

  if (isPureNumber(numStr)) {
    return numStr.charAt(0) === '-' ?
      numStr.replace(/^-0+/, '-') :
      numStr.replace(/^0+/, '');
  }

  if (isPlusNumber(numStr)) {
    return numStr.slice(1).replace(/^0+/, '');
  }

  if (isStartWithDot(numStr)) {
    return numStr.charAt(0) === '-' ?
      `-0${numStr.slice(1)}`:
      `0${numStr}`;
  }

  if (isDecimal(numStr)) {
    return numStr.replace(/^([+-]?)(0+)(\.\d+)/, (matcher, flag, prev, next) => {
      return flag === '+' ?
        `${prev.replace(/0+/, '0')}${next}` :
        `${flag}${prev.replace(/0+/, '0')}${next}`;
    });
  }

  if (isScientificNotation(numStr)) {
    return parseFloat(numStr) + '';
  }

  if (isNumberPlusNotNumber(numStr)) {
    return numStr.match(/^(\d+)/g)[0];
  }

  if (isMultiDots(numStr)) {
    return numStr.match(/^(\d+\.\d+)/g)[0];
  }

  return undefined;
}

/**
 * 不存在数字
 */
function isInvalid(numStr) {
  return !/\d/.test(numStr);
}

/**
 * 纯数字，包含负数
 */
function isPureNumber(numStr) {
  return /^-?\d+$/.test(numStr);
}

/**
 * 数字前面带有 + 号
 */
function isPlusNumber(numStr) {
  return /^\+\d+$/.test(numStr);
}

/**
 * 以 . 开头
 */
function isStartWithDot(numStr) {
  return /^[+-]?\.\d+$/.test(numStr);
}

/**
 * 格式正确的小数
 */
function isDecimal(numStr) {
  return /^[+-]?\d+\.\d+$/.test(numStr);
}

/**
 * 输入科学计数法
 */
function isScientificNotation(numStr) {
  return /^[+-]?\d+e[+-]?\d+$/.test(numStr);
}

/**
 * 输入【数字+非数字的情况】
 */
function isNumberPlusNotNumber(numStr) {
  return /^\d+[^\d\.]+/.test(numStr);
}

/**
 * 多个 . 的情况
 */
function isMultiDots(numStr) {
  return /^(\d+\.\d+)/.test(numStr);
}
```
