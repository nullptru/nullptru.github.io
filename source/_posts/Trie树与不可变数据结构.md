---
title: Trie 树与不可变数据结构
date: 2019-07-05 23:07:24
tags:
  - 前端
  - 数据结构
categories:
	- 技术文档
---

> 本文首发于[知乎专栏：饿了么前端](https://zhuanlan.zhihu.com/p/63207283)

## 不可变对象

### 什么是不可变对象

不可变对象是指数据在创建之后它的状态（成员变量、属性等的值）就无法更改，每次的修改实际上是创建了一个新对象，是一种只读不写的数据结构。与之相对的则为可变对象。

让我们通过一个简单的例子认识下不可变对象：

```javascript
'use strict'
let immutableObj = Object.freeze({ a: 1 })
immutableObj.a = 2; // throw TypeError: Cannot assign to read only property 'a' of object '
console.log(immutableObj.a); // 1

let commonObj = { a: 1 }
commonObj.a = 2;
console.log(commonObj.a); // 2
```
JS 中我们可以通过 [`Object.freeze`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze) 函数简单的将一个对象转换为不可变对象。

### 不可变对象的优点

不可变对象是函数式编程语言的构建基础，正是不可变才使纯函数成为可能。因此我们来看看不可变对象能给我们带来哪些便利

1. 因为对象不可变，因此数据便于构建与测试，便于进行时间旅行调试。
2. 不可变对象始终是线程安全的，因此在并发并行操作上有良好的表现。
3. 不可变对象的使用过程不会产生副作用。
4. 不可变对象不会被上下文修改影响产生难以追溯的 bug

## 不可变对象的变更

在可变对象的场景下，数据变更即为对原始数据的状态进行修改：

![可变对象的变更](https://pic2.zhimg.com/80/v2-c65dc5593c8f0c77902513fae6c0b76e_hd.png)

而对于不可变的对象，我们需要创建一个新的对象进行存储：

![不可变对象的变更](https://pic2.zhimg.com/80/v2-a58b6f4fee7e95e7fff0dab27b8e004f_hd.png)

要实现变更，我们第一想到的就是创建一个新数组 `A'`，将数组 `A` 的元素逐一 Copy，最后在新数组上实现 `insert` 操作。与可变对象的变更相比，势必带来大幅的性能问题。为解决这一问题，笔者将通过这篇文章介绍一种称为「结构共享」的方式对不可变对象变更性能进行优化， [`Immutable.js`](https://facebook.github.io/immutable-js/docs/#/) 与 `clojure` 正是采用这种方式实现。

### Trie 树（前缀树/字典树）

在介绍结构共享前，我们先引入一个称之为 Trie 树的数据结构。 Trie 树与查找树不同，键不是直接保存在节点中，而是由节点在树中的位置决定。一般情况下，不是所有的节点都有对应的值，只有叶子节点和部分内部节点所对应的键才有相关的值。在 Trie 树中，根结点不保存值，此外的每一个节点的子树都含有相同的前缀，也就是节点所对应键。一般而言键为字符串，但也可以是其他数据结构。例如，bitwise trie 中的键是一串比特，可以用于表示整数或者内存地址。在该例子中，我们便采用 bit 值作为树的键值。

![前缀树](https://pic2.zhimg.com/80/v2-be590af80a6d836c357fccc2e8492415_hd.png)

### 使用 Trie 树构建数组

我们以上面例子中的数组 `A` 为例构建 Trie 树。

![](https://pic4.zhimg.com/80/v2-7ffe48c9cb08bad233c93825761776ba_hd.png)

根据元素在数组中的位置，我们转化为对应二进制索引。例如，`001` 为二进制 `2`，代表数组中第二个元素，在 Trie 树中的位置为：左 -> 左 -> 右。由此数组的 Trie 树构建完毕。

### 基于 Trie 树的结构共享

同样的 `insert` 操作，在 Trie 树中我们如何通过结构共享实现呢。因为不可修改数据节点，因此在 `insert` 操作时，我们首先创建个新的 Head 节点，然后根据索引查找元素插入的位置。查找过程中，对于无影响的子树，我们复用原树的路径索引，对于产生影响的子树，我们进行节点的复制，直至找到节点位置，执行 `insert` 操作，具体如下图所示。

![](https://pic3.zhimg.com/80/v2-55040fbeaf434d09a04eaf1d36265bdd_hd.png)

由此一来，在一次数据变更操作中，我们最多需要创建的数据节点数即为树的深度，避免了整个数组数据的变动，而达到了性能优化的目的。

需要注意的是，再执行多次同一操作时，即使产生的结果相同，但我们也并未复用其根节点。例如：

```
let a = [1,2,3,4,5,6,7]
b = push(a, 8)
c = push(a, 8)
```
产生的结果如下：

![](https://pic3.zhimg.com/80/v2-4d44202b44a6eb750c6b8816ffa23114_hd.png)

除插入操作外，我们也看看其他变更操作的展现。

```
let a = [1,2,3,4,5,6,7]
update(a[4], 8)
```

![](https://pic2.zhimg.com/80/v2-a02e2c467095c10ae31d3fc82dc6cc02_hd.png)

```
let a = [1,2,3,4,5]
remove(a, 5)
```

![](https://pic1.zhimg.com/80/v2-3fe36e88f13d852c5bb0eab9e063296e_hd.png)

这里的删除操作，我们选了一个特例，即子树仅有一个节点的情况，根据我们之前描述的操作，在完成删除操作时，会产生一个只有一个子树的空节点，如上图中虚线框所示，此时，若考虑性能的进一步优化，我们可以移除该节点，用其子节点作为新的根节点。

### Trie 树的分支与性能

到目前为止，我们所描述的均为分支为 2 的树，当数据量上升时随之导致树的深度迅速增加。一方面，深度增加意味着每一层节点中可供共享的数据节点数将增加，另一方面也意味着当我们要进行对象变更时所执行的遍历时间和节点复制数将随之增加。因此在实际操作过程中，我们应结合两者的利弊做一个权衡。

## 参考资料
+ [Persistent data structure](https://en.wikipedia.org/wiki/Persistent_data_structure)
+ [Github: Immutable.js](https://github.com/immutable-js/immutable-js/blob/master/src/List.js)
+ [Medium: Immutable.js, persistent data structures and structural sharing](https://medium.com/@dtinth/immutable-js-persistent-data-structures-and-structural-sharing-6d163fbd73d2)
+ [understanding-persistent-vector-pt-1](https://hypirion.com/musings/understanding-persistent-vector-pt-1)
