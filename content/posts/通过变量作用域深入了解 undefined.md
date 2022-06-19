---
title: "通过变量作用域深入了解 undefined"
date: 2020-12-24T21:57:44+08:00
categories: ["Frontend"]
---

# 通过变量作用域深入了解 undefined
undefined 是一个变量，它有可能被篡改，所以使用void（0）来代替他，但是在做实验的时候发现虽然给它赋值不报错，取值时候 undefined 却并没有改变。
```JavaScript
    undefined = 0
    // 0
    undefined
    // undefined
```
这回学习到了变量作用域，再结合之前学的对象的属性，深入了解下 undefined 到底什么时候会被篡改。
## 函数作用域和声明提前
别的语言一般为块级作用域，JavaScript 取而代之使用了函数作用域：变量在声明它们的函数体以及这个函数体嵌套的任意函数体内都是有定义的。
```JavaScript
    var a = function test(o){
        var i = 0;                          // i在整个函数体内均是有定义的
        if(typeof o == "object"){
            var j = 4;                      // j在函数体内是有定义的，不仅仅是在这个代码段内
            for(var k=0; k < 3; k++){       // k在函数体内是有定义的，不仅仅是在循环内
                console.log(k);             // 输出数字0～2
            }
            console.log(k);                 // k已经定义了，输出3
        }
        console.log(k);                     // k已经定义了，输出3
        console.log(j);                     // j已经定义了，输出4
    };
```
JavaScript 的这个特性被非正式地称为声明提前（hoisting），即 JavaScript 函数里声明的所有变量（但不涉及赋值）都被“提前”至函数体的顶部。
```JavaScript
    var a = '1';
    function f(){
        console.log(a);
        var a = '2';
        console.log(a);
    }
```
由于函数作用域的特性，变量 a 的声明被提前到了整个函数体最前面，声明了但是没赋值，所以第一次 console 是 undefined。执行到 var 语句时才被赋值，这应该也是为什么可以一个变量多次 var 声明的原因，后面每一次 var 只是起到了赋值的作用。
## ES6 块作用域
ES6 弥补了这这个概念，追加了 let 和 const，也就有了 {} 块作用域的概念，const 用来声明常量，这个就不提了，用上面的例子感受一下 let 和 var 的区别。
```JavaScript
    var a = function test(o){
        var i = 0;                          // i在整个函数体内均是有定义的
        if(typeof o == "object"){
            let j = 4;                      // j在if块内是有定义的
            for(var k=0; k < 3; k++){       // k在函数体内是有定义的，不仅仅是在循环内
                console.log(k);             // 输出数字0～2
                console.log(j);             // j已经定义了，输出4
            }
            console.log(k);                 // k已经定义了，输出3
        }
        console.log(k);                     // k已经定义了，输出3
        console.log(j);                     // j在块外，抛出异常Uncaught ReferenceError: j is not defined
    };
```
let 和 const 只有在块内是被声明的。
## 全局变量的本质
当声明一个 JavaScript 全局变量时，实际上是定义了全局对象的一个属性。当使用 var 声明一个变量时，创建的这个属性是不可配置的，无法通过 delete 运算符删除。
```JavaScript
    test =1
    // 1
    window.test
    // 1
    delete test
    // true
    window.test
    // undefined
    var test = 1
    // undefined
    window.test
    // 1
    delete test
    // false
    window.test
    // 1
```
可以看到当定义一个全局变量的时候他变成了 window 的属性。
JavaScript 可以允许使用 this 关键字来引用全局对象，但是不能引用局部变量中存放的对象。
```JavaScript
    this.test
    // 1
```
## 深入理解 undefined 的本质
undefined 明明是个变量为什么赋值后无法使用呢。任何地方都可以直接使用 undefined，那他是一个全局变量一个全局的属性，事实也果然如此。
```JavaScript
    window.undefined == undefined
    // true
```
而属性分两类，他们的特征如下：
* 数据属性
    - value：就是属性的值。
    - writable：决定属性是否能被赋值。
    - enumerable：决定 for in 能否枚举该属性。
    - configurable：决定该属性能否被删除或者改变特征值。
* 访问器属性（getter/setter）
    - getter：函数或 undefined，在取属性值时被调用。
    - setter：函数或 undefined，在设置属性值时被调用。
    - enumerable：决定 for in 能否枚举该属性。
    - configurable：决定该属性能否被删除或者改变特征值。
查看下全局属性 undefined 的属性特征。

```JavaScript
    console.log(Object.getOwnPropertyDescriptor(window,'undefined'));
    // configurable: false
    // enumerable: false
    // value: undefined
    // writable: false
```

writable：false，就是这个原因 undefined 虽然是个全局变量，但是我们赋值之后他并没有被改变。那为什么还需要用 void(0) 来代替 undefined 呢？结合上面的函数作用域再做一个实验。

```JavaScript
    function test() {
        var undefined = 100; 
        console.log(undefined);
        var obj = {a : undefined}; 
        console.log(Object.getOwnPropertyDescriptor(obj,'a'));
    }
    test();
    // 100
    // configurable: true
    // enumerable: true
    // value: 100
    // writable: true
```

在函数作用域中，undefined 被从声明并赋值为 100，无论是直接 console 它自己还是去查看 undefined 属性特性都能看出，undefined 变量被修改了。这也就说在函数作用域内 undefined 可以被修改。
```JavaScript
    function test() {
        const undefined = 100; 
        console.log(undefined);
        const obj = {a : undefined}; 
        console.log(Object.getOwnPropertyDescriptor(obj,'a'));
        test();
    }
    // 100
    // configurable: true
    // enumerable: true
    // value: 100
    // writable: true
```
块作用域内也一样可以被修改，得出结论 undefined 是一个类似 var 声明一样可以被重复声明的全局属性，而他的属性特性 writable 是 false，所以不能被修改，但是在非全局作用域内，他是可以被篡改的，这时候就要注意用 void (0) 来替代 undefined。
## 作用域链^1
以书中的角度用对象属性去看作用域链：
* JavaScript最顶层的代码，其作用域链只有一个全局对象。

![](../media/16088182648987/16556418348507.jpg)

* 不包含嵌套的函数，其作用域两个对象：函数自身的变量对象和全局对象。

![](../media/16088182648987/16556418584659.jpg)


> 活动对象(activation object)，该对象包含了函数的所有局部变量、命名参数、参数集合以及this，然后此对象会被推入作用域链的前端

* 对于包含了嵌套函数的函数，其作用域包含至少三个对象：自身的变量对象，外层的变量对象（将自身嵌套的函数），全局变量对象。

![](../media/16088182648987/16556418835132.jpg)

作用域链给我的感觉类似原型链，将函数的上下文构成了链表的形式，而方法执行时一层一层往外找，所以为了方便链表从上到下查找属性，所以有了声明提前，链表最顶端为嵌套函数的最里层，链表最底端则是全局变量对象。

```JavaScript
    function test(){
        var fun1,fun2;
        for(var i=0;i<2;i++){
            if(i === 0){
                fun1 = function(){console.log('fun1:i='+i)};
            }
            if(i === 1){
                fun2 = function(){console.log('fun2:i='+i)};
            }
        };
        fun1();
        fun2();
    }∫
    test();
    // fun1:i=2
    // fun2:i=2
```

## 注
1. 在看 JavaScript 权威指南第七版第八章时发现作者将原来的作用域链的“链”字都删除了，变成了作用域，但是还没有看前面作用域相关内容，应该有新的理解，在完成翻译后再继续补全。