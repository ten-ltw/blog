---
title: "TypeScript 补漏"
date: 2021-05-07T18:06:39+08:00
categories: ["Frontend"]
tags: ["TypeScript"]
hiddenFromHomePage: true
---

## TypeScript 基础

### 函数签名和重载 Signature Overloading

```typescript
function contactPeople(method: "email", ...people: HasEmail[]): void;
function contactPeople(method: "phone", ...people: HasPhoneNumber[]): void;
function contactPeople(method: "email" | "phone", ...people: (HasEmail | HasPhoneNumber)[]): void {
}
```

### 词法作用域 Lexical Scope

#### TODO

## Interfaces 和 type aliases

#### TODO

## 泛型 Generics

### 泛型使用

```typescript
interface WrappedValue<T> {
  value: T;
}
let val: WrappedValue<string[]> = { value: [] };
val.value;
```

鼠标悬停在 `val.value` 的 `value` 上可以看到：

> (property) WrappedValue<string[]>.value: string[]

在各种类库的描述文件中也经常能够看到泛型的应用：

```typescript
interface MapLike<T> {
    [index: string]: T;
}
interface SortedReadonlyArray<T> extends ReadonlyArray<T> {
    " __sortedArrayBrand": any;
}
interface SortedArray<T> extends Array<T> {
    " __sortedArrayBrand": any;
}
```

### 泛型参数

上述方式在使用 WrappedValue 时必须添加泛型参数，如果不添加使用会报错：

> Generic type 'WrappedValue<T>' requires 1 type argument(s).

这里可以看到 WrappedValue 需要一个实参，那么我们也可以为其设置参数默认值，就可以不提供类型参数也能使用 WrappedValue：

``` typescript
interface WrappedValue<T = any> {
  value: T;
}
let val: WrappedValue = { value: [] };
val.value;
```

下面是一个常用的泛型做参数的情况：

```typescript
function resolveOrTimeout<T>(promise: Promise<T>, timeout: number): Promise<T> { // => 1
  return new Promise<T>((resolve, reject) => { // => 2
    // start the timeout, reject when it triggers
    const task = setTimeout(() => reject("time up!"), timeout); // => 3

    promise.then(val => {
      // cancel the timeout
      clearTimeout(task); // => 4                                    

      // resolve with the value // => 5
      resolve(val);                                            
    });
  });
}
```

1. 泛型 T 设定后，会传递一个类型为 Promise\<T> 名为 promise 的参数。
2. 这个方法返回一个类型为 Promise\<T> 的新 Promise。
3. 根据 timeout 时间设置定时，如果时间到了，执行 callback，也就是新创建的 Promise 将 rejected。
4. 如果 timeout 设定的时间没终止，传入的参数 promise 提前 fulfilled，那么直接清除延时。
5. 将 fulfilled 的值 resolve 到新创建的 Promise 中。

这是一个典型的 Http 请求超时的解决方案。

### 约束和作用域 Constraints Scope

可以通过 extends 来约束泛型：

```typescript
function arrayToDict<T extends { id: string }>(array: T[]): { [k: string]: T } {
  const out: { [k: string]: T } = {};
  array.forEach(val => {
    out[val.id] = val;
  });
  return out;
}
```

如果用以下代码将不带有 id 属性对象的数组作为 arrayToDict 的参数：

```typescript
const myDict = arrayToDict([
  { foo: "foo" }
]);
```

就会出现以下错误：

> Type '{ foo: string; }' is not assignable to type '{ id: string; }'.
>   Object literal may only specify known properties, and 'foo' does not exist in type '{ id: string; }'.

这里限制了泛型 T 内必须有 string 类型的 id 属性，所以我们可以使用 `val.id`。如果将约束 `extends { id: string }` 删除，就会看到 `val.id` 出现下面的错误：

> Property 'id' does not exist on type 'T'.

泛型类似参数，同样享有函数参数同样的作用域，也可以应用在闭包中：

``` typescript
function firstTuple<T>(firstElement: T) {
  return function finishTuple<U>(b: U) {
    return [a, b] as [T, U];
  };
}
const myTuple = startTuple(["first"])(42);
```

鼠标悬停在 myTuple 上可以看到该元组的类型：

> **const** myTuple: [string[], number]

### 泛型的使用

泛型只有在 input 和 output 同时需要时才使用，下面是一个没有必要用泛型的例子：

``` typescript
interface Shape {
  draw();
}

interface Circle extends Shape {
  radius: number;
}

function drawShapes1<S extends Shape>(shapes: S[]) {
  shapes.forEach(s => s.draw());
}

function drawShapes2(shapes: Shape[]) {
  // this is simpler. Above type param is not necessary
  shapes.forEach(s => s.draw());
}
```

方法 2 和方法 1 起到了同样的效果，但是输出不再需要泛型。

但是如果有这样一个需求，Shape 中不但有 draw 方法，还有是否已经绘制的 Flag，并且我要在绘制之后将 Shape 对象返回，就可以使用泛型：

``` typescript
interface Shape {
  draw();
  isDrawn: boolean;
}

interface Circle extends Shape {
  radius: number;
}

function drawShapes<S extends Shape>(shapes: S[]): S[] {
  return shapes.map(s => {
    s.draw;
    s.isDrawn = true;
    return s;
  })
}
```

## 顶层类型和底层类型

### Top Types

- any
- unknow

顶层类型可以接收任何类型的值。

unknow 在不确定类型时无法获取它的值。

```typescript
let myUnknown: unknown = "hello, unknown";

if (typeof myUnknown === "string") {
  // in here, myUnknown is of type string
  myUnknown.split(", "); // 
}
if (myUnknown instanceof Promise) {
  // in here, myUnknown is of type Promise<any>
  myUnknown.then(x => console.log(x));
}
```

> typeof 在判断 null、array、object 时都是 object，需要用 instanceof 来判断原型链。

### Type Guards

TypeScript 的类型判断自上而下进行，无法反向推断类型，比如函数的参数，如果没有定义类型，即使在后面赋值 string，参数类型仍然是 any。

可以用以下方式确认复杂类型：

```typescript
function isHasEmail(x: any): x is HasEmail {
  return typeof x.name === "string" && typeof x.email === "string";
}
if (isHasEmail(myUnknown)) {
  // In here, myUnknown is of type HasEmail
  console.log(myUnknown.name, myUnknown.email);
}
```

同理利用泛型有了下面这种通用的类型守卫：

```typescript
function isDefined<T>(arg: T | undefined): arg is T {
  return typeof arg !== "undefined";
}
```

### Branded Types

#### TODO

### Bottom Types

never 个人感觉没啥用。

## 高级类型

### keyof 和 typeof

typeof 是 JavaScript 就有的特性。

keyof 可以直接获取接口的属性名。

接口也可以用属性表达式来获取对应属性的类型。

```typescript
interface CommunicationMethods {
  email: HasEmail;
  phone: HasPhoneNumber;
  fax: { fax: number };
}

function contact<K extends keyof CommunicationMethods>(
  method: K,
  contact: CommunicationMethods[K] // 💡turning key into value -- a *mapped type*
) {
  //...
}
type AllType = keyof CommunicationMethods;
let x: AllType = 'email';
type Email = CommunicationMethods['email'];
```

### Built-In 类型

#### Partial

```typescript
/**
 * Make all properties in T optional
 */
type Partial<T> = {
    [P in keyof T]?: T[P];
};
```

返回新的类型，并且所有的参数都是可选项。

#### Pick

```typescript
type HasThen<T> = Pick<Promise<T>, "then" | "catch">;

let hasThen: HasThen<number> = Promise.resolve(4);
hasThen.then;
```

返回一个类型，指定允许访问的属性。

#### Extract

#### Exclude

#### Record



