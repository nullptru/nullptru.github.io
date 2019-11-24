---
title: 基于 Symbol.toPrimitive 构建 sum 的柯里化函数
date: 2019-04-27 22:22:52
tags: 
  - 函数式
  - JavaScript
categories:
	- 技术文档
---

在 js 中，如何编写一个可无限调用的柯里化 sum 函数？（`sum(1,2,3)(4)()(5)`）。在下文中，就让我们一起实现它。而在实现前，我们需要引入两个知识点：柯里化与 `Symbol.toPrimitive`。

## 柯里化

首先，什么是柯里化？柯里化是函数式编程中的一个重要特性，是把接受多个参数的函数变换成接受部分参数的函数，并且返回接受余下的参数而且返回结果的新函数的技术。

例如： `func(a, b, c, d) => func(a)(b)(c)(d)` 就是一个函数的柯里化应用。柯里化函数的优点是可以很方便的基于现有的通用函数构建出一定特性的新函数。我们举一个简单的例子

```javascript
const multiply = x => y => x * y;

// 需求是获取计算6的倍数的函数
const multiply6 = multiply(6)

multiply6(10); // 60
```

### Symbol.toPrimitive
在 js 中，基本数据类型有 `string`, `number`,`boolean`,`null`,`undefined`,`symbol` 这六种。`Symbol.toPrimitive` 简写为 `@@toPrimitive`。

在 [tc39 规范](https://tc39.github.io/ecma262/#sec-toprimitive)中，`@@toPrimitive` 使用 `ToPrimitive ( input [ , PreferredType ] )` 抽象操作实现。操作接收一个 `input` 参数与一个额外的 `PreferredType` 参数。当一个对象能够转换为多个原始值的时候，将根据 `PrefferedType` 进行如下算法转换：

>
>  1. 校验 `input` 值符合 ECMAScript 规范
>  2. 如果 `type(input)` 不为 `object` 则直接返回
>  3. 如果 `type(input)` 为 `object`
>    1. 如果未定义 `PreferredType`，则设置 `hint` 为默认 `default`
>    2. 如果定义 `PreferredType` 为 hint String，则设置 `hint` 为 `string`
>    3. 如果定义 `PreferredType` 为 hint number, 则设置 `hint` 为 `number`
>    4. 如果 `input` 有定义 `@@toPrimitive` 方法
>      1. 调用方法 `@@toPrimitive(input, hint)` 获取结果 `result`
>      2. 如果 `type(result)` 不为 `object`, 返回最终结果
>      4. 如果 `type(result)` 为 `object`，抛出 `TypeError` 异常
>    5. 如果 `hint` 为 `default`，设置 `hint` 为 `number`
>    6. 调用原生 `OrdinaryToPrimitive` 方法求值
>     1. 如果 `hint` 为 `number`, 则依照 `valueOf`, `toString` 的顺序调用函数
>     2. 如果 `hint` 为 `string`, 则依照 `toString`，`valueOf` 的顺序调用函数
>    3. 若函数存在且结果类型不为 `object`，则返回最终结果
>   4. 否则抛出 `TypeError` 异常
>

通过 `@@toPrimitive` 方法，我们可以自定义函数的原始值 （ps. js 里函数也为对象

由此，我们可以写出如下 `sum` 函数

```javascript
const sum = (...xs) => {
  const theSum = (args) => args.reduce((a, b) => a+b, 0)
  const go = (acc) => {
    const result = (...ys) => go(theSum([acc, ...ys]));
    // 定义 @@toPrimitive 方法
    result[Symbol.toPrimitive] = hint =>
      hint === 'string' ? string(acc) : acc;
    return result
  }
  return go(theSum(xs));
}

console.log(sum(1, 2, 3) + 1); // 7
console.log(sum(1)(2)(3) + 1); // 7
console.log(sum(1)(2)(3) + '1'); // 61
```
因为定义了 `Symbol.toPrimitive` 方法，`result` 便具有了函数和作为 `string` 或 `number` 的多重身份，从而实现了我们的目标。
