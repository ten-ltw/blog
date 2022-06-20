---
title: "Angular 变更检测——它到底是如何工作的？"
date: 2021-03-23T19:41:26+08:00
categories: ["Frontend"]
tags: ["Angular"]
hiddenFromHomePage: true
---

原文：[Angular Change Detection - How Does It Really Work?](https://blog.angular-university.io/how-does-angular-2-change-detection-really-work/)

与 AngularJs 中的等效方法相比，Angular 变更检测机制更加透明且易于推理。但是，在某些情况下（例如进行性能优化时），我们的确需要了解底层做了什么。因此，让我们通过研究以下主题来更深入地研究变更检测：

- 变更检测是如何实现的？
- Angular 变更检测器是什么样的，我能看到它吗？
- 默认变更检测机制如何工作
- 打开 / 关闭变更检测，并手动触发
- 避免变更检测循环：生产与开发模式
- `OnPush` 变更检测模式到底做了什么？
- 使用 Immutable.js 简化 Angular 应用程序的构建
- 总结

如果你在寻找更多关于 `OnPush` 变更检测的信息，看一下这篇文章 [Angular OnPush Change Detection and Component Design - Avoid Common Pitfalls](https://blog.angular-university.io/onpush-change-detection-how-it-works)。

## 变更检测是如何实现的

Angular 可以检测组件数据何时变更，然后自动重新渲染视图以反映该变更。但是，如何在诸如单击按钮之类的低级事件发生之后做到这一点，这可以在页面任何地方发生吗？

要了解它是如何工作的，我们首先需要意识到在 JavaScript 中整个运行时可以被设计重写。如果需要，我们可以重写 `String` 或 `Number` 中的函数。

### 重写浏览器默认机制

Angular 在启动时会修补几个低级浏览器 API，例如 `addEventListener`，这是用于注册所有浏览器事件（包括单击处理程序）的浏览器功能。Angular 会用下面等效的新版本替换 `addEventListener`：

```javascript
// this is the new version of addEventListener
function addEventListener(eventName, callback) {
     // call the real addEventListener
     callRealAddEventListener(eventName, function() {
        // first call the original callback
        callback(...);     
        // and then run Angular-specific functionality
        var changed = angular.runChangeDetection();
         if (changed) {
             angular.reRenderUIPart();
         }
     });
}
```

新版本的 `addEventListener` 为任何事件处理程序增加了更多功能：其不仅调用已注册的回调，而且 Angular 还可以运行变更检测并更新 UI。

### 低级运行时补丁是如何工作的？

浏览器 API 的低级补丁是由 Angular 的 [Zone.js](https://github.com/angular/zone.js/) 库完成的。了解什么是 zone 很重要。

zone 无非是一个执行上下文，它可以在多个 JavaScript VM [^1]执行回合后幸存下来。这是一种通用机制，我们可以用来向浏览器添加额外的功能。Angular 用 Zone 来触发变更检测，但另一种可能的用途是进行应用程序性能分析，或跟踪跨多个 VM 轮次运行的长堆栈跟踪。

### 支持浏览器异步 API

修补以下常用的浏览器机制，以支持变更检测：

- 全部浏览器事件（click、mouseover、keyup 等）
- `setTimeout()`  和 `setInterval()`
- Ajax HTTP 请求

实际上，Zone.js 修补了许多其他浏览器 API，以明了地触发 Angular 变更检测，例如 Websockets。看看[ Zone.js 测试说明](https://github.com/angular/zone.js/tree/master/test/patch)来了解当前支持的功能。

这种机制的局限性之一在于，如果由于某些原因 Zone.js 不支持异步浏览器 API，那么将不会触发变更检测。例如，IndexedDB 回调就是这种情况。

这就解释了变更检测是如何触发的，但是一旦触发，它实际上如何工作？

### 变更检测树

每个 Angular 组件都有一个关联的创建于应用程序启动时的变更检测器。例如下面 `TodoItem` 组件：

```javascript
@Component({
    selector: 'todo-item',
    template: `<span class="todo noselect" 
       (click)="onToggle()">{{todo.owner.firstname}} - {{todo.description}}
       - completed: {{todo.completed}}</span>`
})
export class TodoItem {
    @Input()
    todo:Todo;

    @Output()
    toggle = new EventEmitter<Object>();

    onToggle() {
        this.toggle.emit(this.todo);
    }
}
```

如果切换 Todo 状态，则此组件将接收 Todo 对象作为输入并发生事件。为了使示例更有趣，[Todo 类](https://github.com/jhades/blog.angular-university.io/blob/master/ng2-change-detection/src/todo.ts#L11)包含嵌套对象：

```javascript
export class Todo {
    constructor(public id: number, 
        public description: string, 
        public completed: boolean, 
        public owner: Owner) {
    }
}
```

我们可以看到 Todo 有一个属性 `owner`，它本身是一个具有两个属性的对象：firstname 和 lastname。

### Todo Item 变更检测器是什么样子的？

我们实际上可以在运行时看到变更检测器的外观！ 可以通过访问某个属性查看它，只需在 Todo 类中添加一些代码以[触发断点](https://github.com/jhades/blog.angular-university.io/blob/master/ng2-change-detection/src/todo .ts＃L11)。

执行到断点时，我们可以遍历堆栈跟踪并查看实际的变更检测：

![What does an Angular change detector look like](https://raw.githubusercontent.com/jhades/blog.angular-university.io/master/ng2-change-detection/images/change.jpg)

不用担心，你将永远不必调试此代码！也不涉及任何魔术，它只是在应用程序启动时构建的普通 JavaScript 方法。但是它是做什么的呢？

## 默认变更检测机制如何工作？

乍一看，此方法可能看起来很奇怪，并且其全部变量命名奇怪。但是，通过更深入地研究它，我们发现它的操作非常简单：对于模板中使用的每个表达式，它会将表达式中使用的属性的当前值与该属性的先前值进行比较。

如果前后的属性值不同，它将把 isChanged 设置为 true，就是这样！差不多了，它是通过使用名为 `looseNotIdentical()` 的方法进行值比较，使用 `===` 比较但指定 NaN 相等逻辑（请参阅 [here](https://github.com/angular/angular/blob/50548fb5655bca742d1056ea91217a3b8460db08/modules/ angular2 / src / facade / lang.ts＃L367)）。

### 嵌套对象的 `owner` 如何处理？

我们可以在变更检测器代码中看到 `owner` 嵌套对象的属性同样被检测差异。但是只比较 firstname 属性，不比较 lastname 属性。

这是因为组件模板中未使用 lastname （请参阅 [here](https://github.com/jhades/blog.angular-university.io/blob/master/ng2-change-detection/src/todo_item.ts#L7)）！ 同样，出于相同原因，未比较 Todo 的顶级 id 属性。这样，我们可以放心地说：

> 默认情况下，Angular 变更检测通过检测模板表达式的值是否已更改来工作。这是针对所有组件都做的处理。

我们还可以得出以下结论：

> 默认情况下，Angular 不会进行深度对象比较来检测变更，它仅考虑模板使用的属性。

### 为什么默认情况下变更检测如此工作？

Angular 的主要目标之一是更加透明和易于使用，从而使框架用户不必费劲地调试框架并了解内部机制，就能够有效地使用它。

如果你熟悉 AngularJs，想象一下 `$digest()` 和 `$apply()` 以及何时使用它们或不使用它们的所有陷阱。Angular 的主要目标之一就是避免这种情况。

### 引用如何比较?

事实上 Javascript 对象是可变的，Angular 希望为这些对象提供开箱即用的全面支持。

想象一下，如果 Angular 默认变更检测机制将基于组件输入的引用比较而不是默认机制，那将会是什么样子？ 即使是像 TODO 应用程序这样简单的东西，构建起来也很棘手：开发人员必须非常小心地创建新的 Todo，而不是简单地更新属性。

但是，正如我们将看到的，如果我们确实需要的话，仍然可以自定义 Angular 变更检测。

### 性能如何？

请注意，todo list 组件的变更检测器如何显式引用 `todos` 属性。

做到这一点的另一种方法是在组件的属性之间动态循环，从而使代码通用而不是特定于组件。这样，我们就不必在启动时为每个组件构建一个变更检测器！ 那么这是怎样一回事呢？

### 快速浏览虚拟机内部

所有这些都与 Javascript 虚拟机的工作方式有关。尽管通常 VM 即时编译器无法轻易地将用于动态比较属性的代码优化为本机代码。

这与变更检测器的特定代码不同，变更检测器确实显式访问每个组件输入属性。该代码非常类似于我们手工编写的代码，并且很容易被虚拟机转换为本地代码。

使用生成的显式的检测器的最终结果是一种变化检测机制，该机制非常快（比 AngularJs 更快），可预测且易于推理。

但是，如果遇到性能困境，是否有一种方法可以优化变更检测？

## OnPush 变更检测模式

如果我们的 Todo list 真的很大，我们可以配置 [TodoList](https://github.com/jhades/blog.angular-university.io/blob/master/ng2-change-detection/src/todo_list.ts) 组件仅在 Todo list 变更时才进行更新。这可以通过将组件变更检测策略改为 `OnPush` 来完成：

```javascript
@Component({
    selector: 'todo-list',
    changeDetection: ChangeDetectionStrategy.OnPush,
    template: ...
})
export class TodoList {
    ...
}
```

现在让我们给应用程序添加一对按钮：一个按钮通过直接对列表的第一条进行改变来切换，另一个按钮将 Todo 添加到整个列表中。代码如下：

```javascript
@Component({
    selector: 'app',
    template: `<div>
                    <todo-list [todos]="todos"></todo-list>
               </div>
               <button (click)="toggleFirst()">Toggle First Item</button>
               <button (click)="addTodo()">Add Todo to List</button>`
})
export class App {
    todos:Array = initialData;

    constructor() {
    }

    toggleFirst() {
        this.todos[0].completed = ! this.todos[0].completed;
    }

    addTodo() {
        let newTodos = this.todos.slice(0);
        newTodos.push( new Todo(1, "TODO 4", 
            false, new Owner("John", "Doe")));
        this.todos = newTodos;
    }
}
```

现在让我们看看两个新按钮的行为：

- 第一个按钮“Toggle First Item”不起作用！ 这是因为 `toggleFirst()` 方法直接使列表的元素发生改变。
  `TodoList` 无法检测到它，因为它的输入引用 `todos` 没有改变。
- 第二个按钮确实起作用！ 请注意，方法 `addTodo()` 创建了 todo list 的副本，然后向该副本添加了一个项目，最后用复制的 list 替换了 todos 成员变量。这会触发变更检测，因为该组件在其输入中检测到引用变更：它收到了一个新列表！
- 在第二个按钮中，直接修改 todo list 将不起作用！我们需要一个新列表。

### `OnPush` 真的只是通过引用比较比较输入吗?

事实并非如此，如果你尝试通过单击来切换 todo，它仍然可以使用！即使你将 [TodoItem](https://github.com/jhades/blog.angular-university.io/blob/master/ng2-change-detection/src/todo_item.ts) 也切换为 `OnPush`。这是因为 `OnPush` 不仅仅检查组件输入中的更改：如果组件发生事件，该事件也将触发变更检测。

根据[引用](http://victorsavkin.com/post/110170125256/change-detection-in-angular-2)自 Victor Savkin 的博客：

> 当使用 OnPush 检测器时，框架将在其任何输入属性发生更改时，当它触发事件或当 Observable 触发事件时检查 OnPush 组件。

尽管可以提供更好的性能，但如果与可变对象一起使用，则使用 `OnPush` 的代价是很高的复杂性。它可能会引入难以推理和重现的错误。但是有一种方法可以使 `OnPush` 的使用成为可行的。

### 使用 Immutable.js 简化 Angular 应用程序的构建

如果我们仅使用不可变对象和不可变列表构建应用程序，则可以在所有地方使用 `OnPush`，而不会陷入变更检测错误的风险。这是因为对于不可变对象，修改数据的唯一方法是创建一个新的不可变对象并替换先前的对象。对于不可变的对象，我们可以保证：

- 一个新的不可变对象将始终触发 `OnPush` 变更检测
- 我们不能忘记创建对象的新副本而意外地产生错误，因为修改数据的唯一方法是创建新对象

对于不可变的一个不错的选择是使用 [Immutable.js](https://facebook.github.io/immutable-js/) 库。该库提供了用于构建应用程序的不可变基元，例如不可变对象（Map）和不可变列表。

该库也可以以类型安全的方式使用，阅读[之前的文章](https://blog.angular-university.io/angular-2-application-architecture-building-flux-like-apps-using-redux-and-immutable-js-js/)，以获取有关如何执行此操作的示例。

## 避免变更检测循环：生产与开发模式

Angular 变更检测的重要属性之一是，它与 AngularJs 不同，它强制执行单向数据流：当更新控制器类上的数据时，变更检测将运行并更新视图。

但是视图的更新本身并不会触发进一步的更改，而这些更改又会触发视图的进一步更新，从而在 AngularJs 中创建了 digest cycle。

### 如何在Angular中触发变更检测循环？

一种方法是使用生命周期回调。例如，在 [TodoList](https://github.com/jhades/blog.angular-university.io/blob/master/ng2-change-detection/src/todo_list.ts#L46) 组件中，我们可以触发回调到另一个组件改变其中一个绑定：

```javascript
ngAfterViewChecked() {
    if (this.callback && this.clicked) {
        console.log("changing status ...");
        this.callback(Math.random());
    }
}
```

错误消息将显示在控制台中：

```
EXCEPTION: Expression '{{message}} in App@3:20' has changed after it was checked
```

仅当我们在开发模式下运行 Angular 时，才会引发此错误消息。如果启用生产模式会怎样？

```javascript
@NgModule({
    declarations: [App],
    imports: [BrowserModule],
    bootstrap: [App]
})
export class AppModule {}
```

在生产模式下，不会引发该错误，并且不会发现问题。

### 变更检测问题经常发生吗？

我们的确必须竭尽全力触发变更检测循环，但以防万一最好在开发阶段始终使用开发模式，因为这样可以避免问题。

这种保证是以 Angular 始终运行变更检测两次为代价的，这是第二次检测此类情况。在生产模式下，变更检测仅运行一次。

## 打开 / 关闭变更检测并手动触发

在某些特殊情况下，我们确实希望关闭变更检测。想象一下一种情况，其中大量数据通过 Websocket 从后端到达。我们可能只想每 5 秒更新一次 UI 的特定部分。为此，我们首先将变更检测器注入到组件中：

```javascript
constructor(private ref: ChangeDetectorRef) {
    ref.detach();
    setInterval(() => {
      this.ref.detectChanges();
    }, 5000);
  }
```

如我们所见，我们只是拆下了变更检测器，这有效地关闭了变更检测。然后，我们只需调用 `detectChanges()` 每 5 秒手动触发一次。

现在，让我们快速总结一下有关 Angular 变更检测所需的所有知识：它是什么，它如何工作以及变更检测主要的可用类型是什么。

## 总结

Angular 变更检测是内置的框架功能，可确保组件数据与其 HTML 模板视图之间的自动同步。

变更检测通过检测常见的浏览器事件（例如鼠标单击，HTTP 请求和其他类型的事件），并确定是否需要更新每个组件的视图来起作用。

变更检测有两种类型：

- 默认变更检测：Angular 通过比较事件发生之前和之后的所有模板表达式值（对于组件树的所有组件）来决定是否需要更新视图。
- OnPush 变更检测：这是通过检测一些新数据是否已通过组件输入或使用异步管道订阅的 Observable 显式推送到组件中而起作用的。（还有触发事件）

Angular的**默认**变更检测机制实际上与 AngularJs 相似：它比较浏览器事件前后的模板表达式的值，以查看是否有所更改。**所有**组件都这样做。但是也有一些重要的区别：

其一，没有变更检测循环，也没有像 AngularJs 中的 digest cycle。这允许仅通过查看其模板和其控制器就可以推断出每个组件。

另一个区别是，由于构建了变更检测器，因此组件中检测变更的机制要快得多。

最后，与 AngularJs 不同，变更检测机制是可以自定义的。

[^1]: JavaScript VM，JavaScript 虚拟机，即 JavaScript 引擎。