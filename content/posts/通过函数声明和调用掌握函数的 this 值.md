---
title: "通过函数声明和调用掌握函数的 this 值"
date: 2021-01-16T10:15:20+08:00
categories: ["Frontend"]
tags: ["JavaScript"]
---

# 通过函数声明和调用掌握函数的 this 值
函数的 this 对于基础薄弱的前端程序员一直是一个比较头疼的问题，在对 ES6 的模糊和对 ES6 之前的函数作用域不理解双重打击下，往往无法区分 this 值到底是什么，因而在编写程序时常常需要通过测试当前作用域的 this 是什么，再去使用，甚至是某些公司严格规定哪种方式去实现。

细细的看完了权威指南的第七版受益颇多，但是作者并没有拿出一个章节专门将相关内容进行汇总，这里就通过函数的声明和调用来总结下函数 this 值的几种情况。
## 函数定义
函数的定义有多种方式来适应不同的使用场景：
- 函数声明语句
- 函数表达式
- 箭头函数
- 嵌套函数
- 应用函数的构造函数 Function()

### 函数声明语句
与大多数声明语句一样，以关键字 + 名称的形式来进行声明，函数用关键字 function 来声明：
```js
function testOutPut(){
    console.log('Hello world!');        
}
```
函数声明语句是函数名变成一个变量，这个变量的值是函数本身。这里实际上是声明了一个 testOutPut 变量，并讲函数对象赋值给 testOutPut。在 JavaScript 的运行时中，函数对象创建于该函数所在作用域的代码开始执行之前，也就是说，在同一个上下文中，可以先执行函数调用再进行函数语句声明。只是第一个函数作为值的概念的应用场景。
### 函数表达式
函数表达式是一个赋值语句，将一个函数赋值给一个变量或常量。感觉函数表达式跟函数声明语句的运行时一样，但是如果先进行函数调用再进行函数表达式来定义函数将会异常。因为函数表达式在没有执行前并没有将函数赋值给变量，所以它也就不能被引用。
```js
beforeDefined();
let beforeDefined = function(){console.log(1);}
// Uncaught ReferenceError: beforeDefined is not defined
```
函数表达式的函数名称是可选项，如果给函数表达式添加一个函数名，那这个函数的*局部函数作用域内*会包含该属性的函数名的对象，其值绑定的是该函数：
```js
const f = function fact(x) {
    if (x <= 1) return 1;
    else return x*fact(x-1);
};
```
只有在函数局部作用域内会有这个对象，有点闭包的感觉。
### 箭头函数
箭头函数是 ES6 的新特性，在实际生产中非常常用。它支持很多简洁语法：
```js
const sum = (x, y) => { return x + y; };
const sum = (x, y) => x + y;
const polynomial = x => x*x + 2*x + 3;
const constantFunc = () => 42;
```
注意参数和箭头之间不能换行，否则会引起歧义。
除此之外，它最大的特性就是它继承定义它环境的 this 值，我理解添加箭头函数的目的就是为了解决方法的嵌套函数 this 值为全局对象（严格模式下是 undefined）的缺陷：
```js
let o = {
    a:1,
    m:function(){
        let self = this;
        console.log(this === o);
        f();
        function f(){
            console.log(this === o);
            console.log(self === o);
            console.log(this);
        }
    }
}
o.m();
// true
// flase
// true
// Window{...}
```
在 ES6 之前，我们需要通过闭包将 this 值保存在变量中使用，而有了箭头函数，可以直接继承 m 方法的 this 值：
```js
o = {
    a:1,
    m:function(){
        let self = this;
        console.log(this === o);
        const f = () => {
            console.log(this === o);
            console.log(self === o);
            console.log(this);
        }
        f();
    }
}
o.m();
// true
// true
// true
// {a: 1, m: ƒ}
```
### 嵌套函数
函数可以嵌套在其他函数内。嵌套函数可以访问包含它们函数的变量和参数（闭包）：
```js
function first(){
    let strFirst = "first";
    second();
    function second(){
        let strSecond = "second";
        third();
        function third(){
            console.log(strFirst);
            console.log(strSecond);
        }
    }
}
first();
// first
// second
```
### Function() 构造函数创建函数
Function() 函数的实参都是字符串，最后一个实参是函数体。
```js
const f = new Function("x", "y", "return x*y;");
```
这里想到一个情况，当函数名相同时是否会报错：
```js
const f1 = new Function("x", "x", "return x;");
f1(1,2);
// 2
```
实际情况并没有，再次测试一下直接函数声明语句使用两个相同的参数，也没有报错：
```js
function f2(x,x,y){return x*y};
f2(1,2,2);
// 4
```
这样看实际上跟函数外部用 var 声明的变量在函数中再次声明一个效果：
```js
var x = 1;
function test(){let x = 2;console.log(x)};
console.log(x);
test();
// 1
// 2
```
函数变量声明多个相相同的变量名时，传递的实参值将反复覆盖同名参数。
Function() 构造函数非常重要的一点，就是它所创建的函数并不是使用词法作用域，相反，函数体代码的编译类似顶层函数，看书上的一个例子：
```js
let scope = "global";
function constructFunction() {
    let scope = "local";
    return new Function("return scope");  // Doesn't capture local scope!
}
constructFunction()(); 
// "global"
```
## 函数调用
函数调用和方法调用很常见就不再赘述，这里有三种很少使用的调用方式：
- 构造函数调用
- 间接调用
- 隐式调用

### 构造函数调用
```js
function CreatObj(){
    this['p'] = "property";    
}
o = new CreatObj();
// CreatObj {p: "property"}
```
构造函数创建一个新对象，并将这个对象作为它的上下文，也就是说构造函数中的 this 值是新创建的对象。
构造函数通常没有 return 语句，当显示的用 return 语句返回一个对象时，构造函数创建的对象将变成它 return 的对象，其他情况当没有返回值或返回一个原始值时，返回值都被忽略，返回它新创建的对象。
### 间接调用
JavaScript 万物接对象，对象就有自己的属性和方法。同理，函数也是对象，也有自己的方法和属性，而 call() 和 apply() 方法也可以用来间接调用函数。
#### call() 方法
MDN 中对于 call() 的解释已经很明了了：
> The call() method calls a function with a given this value and arguments provided individually.

```js
f.call(o, 1, 2);
```
这里不过是将 o 对象作为函数 f 调用的上下文，并传入两个实参 1 和 2 作为实参列表，也就是说它等同与下面的代码：
```js
o.m = f;
o.m(1, 2);
delete o.m;
```
这里注意 delete 语句，目的就是说函数在使用 call 方法时只是临时将 o 作为 f 的上下文并调用。而函数还有个不涉及调用的方法 bind() 它可以返回一个新的函数对象，`f.bind(o);`，而这个对象将一直绑定 o 为函数 f 的 this 值。
#### apply() 方法
> The apply() method calls a function with a given this value, and arguments provided as an array (or an array-like object).

不难看出 apply() 和 call() 基本一致，只是实参以数组或者类似数组的对象进行传递。这里对类似数组的对象进行测试：
```js
function test(...args){
    console.log(this);
    for(item in args) console.log(item);
}
let o = {a:1};
let likeArr = {p1:1,p2:2};
likeArr.length = 2;    
let unLikeArr = {p1:1,p2:2};
test.apply(o,likeArr);
// {a: 1}
// 0
// 1
test.apply(o,unLikeArr);
// {a: 1}
```
可见有了 length 的对象就被当作是类似数组的对象。

#### this 值
可见当 call() 和 apply() 调用函数时给函数绑定了一个上下文 this 值，但是箭头函数是个特例，它从它定义的位置的上下文继承 this 值。也就是说当给箭头函数定义的函数使用这两个方法时，第一个参数将被忽略：
```js
let o = {
    a: 1,
    f: function(){
        const oBind = {b:1};
        const test = ()=> console.log(this);
        test.apply(oBind);
    }
}
o.f();
// {a: 1, f: ƒ}
```

### 隐式调用
- getter 和 setter 方法在获取或者设置它的属性时可能被隐式调用。
- 在对象进行字符串或者数值型和 BigInt 类型转换时，会隐式调用 toString() 和 valueOf() 方法。
- 循环可迭代对象的元素时会产生很多方法调用。
- 模板字符串可以调用函数。
- Proxy 对象的任何一个操作都会导致函数调用。

## 关于函数 this 值的总结
函数 this 值有以下几种情况：
- 函数声明语句和函数表达式定义的函数，在非严格模式下 this 值永远是 global，严格模式下是 undefined。
- apply() call() bind() 方法用在非箭头函数时，第一个参数作为函数运行时的 this 值。
- 箭头函数的 this 值永远继承于它所定义的位置的 this 值。
- 对象的方法中 this 值是该对象的引用，嵌套在该对象中的函数不为方法，所以遵循函数的 this 值（global 或者 undefined）。
- 构造函数的 this 值是新创建的对象。

闭包只是参数和变量作用域的引用，与 this 值无关。