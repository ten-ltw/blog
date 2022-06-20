---
title: "JavaScript 深浅拷贝"
date: 2021-02-04T21:39:37+08:00
categories: ["Frontend"]
tags: ["JavaScript"]
---

# JavaScript 深浅拷贝

在数组章节提到数组的展开运算符时，作者举例说明展开运算符可以进行拷贝，原为如下：
 
> The spread operator is a convenient way to create a (shallow) copy of an array:

```js
let original = [1,2,3];
let copy = [...original];
copy[0] = 0;  // Modifying the copy does not change the original
original[0]   // => 1
```

copy 对象为 original 展开后的数组字面量，但是拷贝后的数组修改了序列为 0 的元素的值，但是被拷贝数组中序列为 0 的元素的值并没有被改变，作者称这样的拷贝为浅拷贝，

在我的印象中深浅拷贝是如下形式的：

- 浅拷贝（shallowCopy）只是增加了一个指针指向已存在的内存地址，在修改了拷贝元，拷贝先也会跟着改变。
- 深拷贝（deepCopy）是增加了一个指针并且申请了一个新的内存，使这个增加的指针指向这个新的内存，拷贝元和拷贝之间脱离了关系。

但是这个实例中，作者只测试了拷贝先的修改，我又做了如下测试：

```js
let original = [1,2,3];
let copy = [...original];
original[0] = 10;
copy[0]  // => 1
```

就是说在修改被拷贝数组时，拷贝后的数组也不跟随改变。
所以作者的浅拷贝指的是什么呢？
问题需要最根溯源，从基础来解决问题，下面就先来回溯下数据类型。

## JavaScript 数据类型

最新 ECMAScript 标准定义了九种类型：

- 六个数据类型

```js
typeof undefined === 'undefined';
typeof true === 'boolean';
typeof 123 === 'number';
typeof 'str' === 'string';
typeof 123n ==='bigint';
typeof Symbol('symbol') === 'symbol';
```

- 两个结构类型

```js
typeof {} === 'object';
typeof (() => {}) === 'function';
```
- 结构根

```js
typeof null === 'object';
```

以上九种类型又分为原始值（Primitive Value）和对象（Object）：

- Primitive values
    - Boolean type
    - Null type
    - Undefined type
    - Number type
    - BigInt type
    - String type
    - Symbol type
- Objects

原始值（又名基本类型）其值存放在栈中，它的值无法被修改，当对基本类型进行操作时，会在新的空间内创建一个原始值。

对象也就是引用类型（又名复杂类型），它只在栈中保存其内存地址，指向存在堆中的数据，当对其进行修改时，栈内的地址进行改变，指向堆中另外的数据。

## 问题解决

现在再回想一下深浅拷贝的定义就会发现，其只是针对复杂类型。也就是说在原文中所用示例在用展开运算符对 original 数组进行拷贝，而其拷贝的值是初始数组内的元素，而其元素只是 number 类型的原始值，因此这个实例无法展示其浅拷贝特性。

了解了这些后，针对引用类型做一个实验：

```js
let original = [{a:1},2,3];
let copy = [...original];
original[0].a = 10;
copy[0].a; // => 10
```
这个例子中初始数组中的第一个元素为对象，是一个引用类型数据，拷贝后对其中的 a 属性进行修改，果然拷贝后的数组中第一个元素的 a 属性值也被修改。如此证明了展开运算符的确是浅拷贝。

## 总结

这次的错误理解的根源是对基本概念掌握的不够牢固，这里展开运算符只是将初始数组中的元素展开到一个新的数组字面量中，因此并不是拷贝整个数组，而是拷贝数组中的每个元素，所以深浅拷贝的描述对象也是数组中的元素。
