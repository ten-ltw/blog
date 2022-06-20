---
title: "ng-fundamentals 基础内容查缺补漏"
date: 2021-04-05T10:51:21+08:00
categories: ["Frontend"]
tags: ["Angular"]
---

# ng-fundamentals 基础内容查缺补漏

## 组件

```javascript
@Component({
  selector: 'events-app',
  template: `
    <h2>Hello world</h2>
    <img src="/assets/images/basic-shield.png"/>
  `
})
```

资源路径相对于 index.html，不能使用静态路径访问文件。

webpack 需要在编译 app 时知道哪一个文件要被捆绑，在 angular.json 中设置：

```json
"options": {
    "outputPath": "dist/ng-fundamentals",
    "index": "src/index.html",
    "main": "src/main.ts",
    "polyfills": "src/polyfills.ts",
    "tsConfig": "tsconfig.app.json",
    "aot": true,
    "assets": [
        "src/favicon.ico",
        "src/assets"
    ],
```

组件需要在模块上注册 angular 才知道去哪里获取组件资源。

组件有一个 class 包含所有组件逻辑和内容。

## Angular 模板语法

### 模板变量

`#标识符` 模板变量可以在模板的任何位置访问：

```html
<event-thumbnail #thumbnnail [event]="events" (eventClick)="handleEventClicked($event)"></event-thumbnail>
<button class="btn btn-primary" (click)="thumbnnail.logFoo()">Log me some foo</button>
```

可以在 HTML 中通过组件的模板变量访问子组件中的成员变量和成员方法。

### 结构指令

\* 星号代表结构指令

结构指令改变 DOM 的形状，它不是隐藏 DOM 而是添加或删除 DOM。

> safe navigation operator ?
>
> ?. 以这种操作符绑定数据，预防 undefined

*ngIf 在画面屡次显示隐藏 DOM 时会消耗过大，这时候应该使用 hidden 属性，画面初始化后不变选用 *ngIf。

### ng 绑定 class

#### class binding

```html
<span [class.green]="event?.time === '8:00 am'">{{event?.time}}</span>
```

#### ngClass

```html
<span [ngClass]="{green: event?.time === '8:00 am', bold: event?.time === '8:00 am'}">{{event?.time}}</span>
```

ngClass 参数可以是函数，函数返回一个对象 {classKey：boolean,…} 或空格分隔字符串 'green bold' 或者 class 数组 ['green' , 'bold']。

### ng 绑定 style

#### style binding

```html
<span [style.color]="event?.time === '8:00 am' ? '#003300' : '#bbb'">{{event?.time}}</span>
```

#### ngStyle

[ngStyle] 后面接对象 {三元运算，三元运算}，也可以使用函数返回样式对象。

### 模板表达式约束

以下内容不允许使用：

- Assignment（=,+=,==,etc）
- new Keyword
- Expression Chaining With ;
- Global Namespace

总结来说就是不允许产生副作用，不允许使用 global 命名空间内的成员。

## 服务

### 服务

```typescript
@Injectable()
constructor(private eventService: EventService) {    
}
```

这里的实际上是个简写，全文：

```typescript
@Injectable()
private eventService: EventService;
constructor(private _eventService: EventService) {
    this.eventService = _eventService;
}
```

不要在组件构造函数中添加过多逻辑，应该将逻辑放到生命周期方法中。

`implements OnInit` 是 typeScript，只是语法上严谨，并不是必须的。

### 引入第三方库

引入 toastr 时，在上面声明了一个 toastr ：

```typescript
declare let toastr: any;
```

实际上，在 angular.json 中已经添加了 toastr 的 Js 包，使其在 window 对象中添加了 toaster 变量，我们已经知道了 toastr object 是什么，我们可以直接使用这个对象中的接口，但是 typeScript 编译器不知道，所以只需要声明一个变量，让编译器通过编译即可。

在使用时，不要在使用的组件中直接声明 toastr 对象去使用，这样不容易 mock，不方便单元测试，取而代之的时提供一个可复用的服务。

目前，已经有了 ngx-toastr 库提供了 ToastrServie 接口供 angular 使用。

### 记录状态

service 还有个功能是记录状态。例如身份认证。

## 路由

组件都有 selector 属性，是因为其会被用在 html 的内部，当组件被作为路由使用时，则不需要 selector 属性。

```typescript
import { Routes } from "@angular/router";
```

路由只是一个**数组**配置（它是有序的，从上往下查找匹配路由）,这里导入 Routes 也同样只是为了 typeScript 的静态类型 check。

真正让这个数组奏效的是在模块中通过路由模块的根路由对该数组的载入：

```type
RouterModule.forRoot(appRoutes)
```

```typescript
{ path: '', redirectTo: 'events', pathMatch: 'full' }
```

注意添加默认路由。

pathMatch：path-matching 策略，默认 prefix，还有一个选项是 full

- prefix 以指定的 path 开始的 URL
- full 完全符合指定 path 的 URL

### 路由的步骤

1. 添加一个 router-outlet
2. 定义路由（一个常量数组）包含一个默认路由
3. 在 module 中使用 RouterModule 载入路由
4. \<base href="/">

### 路由参数

#### 定义参数

冒号提示 Angular 这是一个 URL 参数。

#### 获取参数

```typescript
import { ActivatedRoute } from "@angular/router";
```

注入 ActivateRoute service 获取参数。

```typescript
+this.route.snapshot.params['id']
```

这里获取到的参数是字符串，需要转成 number

### HTML 链接路由

```typescript
[routerLink]="['/events', event.id]"
```

[routerLink] 的参数是数组，第一个元素是字符串路由地址，第二个元素是参数。

### Code 导航页面

```typescript
Router.navigate(['/events', event.id])
```

参数同上

### Route Guards

守卫即可以使用方法，也可以使用服务。

它既可以组织进入某个页面，也可以阻止离开某个页面。

#### canActivate

通过服务实现 canActivate

```typescript
{ path: 'events/:id', component: EventDetailsComponent, canActivate: [EventRouteActivator] },
```

```typescript
@Injectable()
export class EventRouteActivator implements CanActivate {
    constructor(private eventService: EventService, private router: Router) {
    }
    
    canActivate(route: ActivatedRouteSnapshot) {
        const eventExists = !!this.eventService.getEvent(+route.params['id']);
        if (!eventExists)
            this.router.navigate(['/404']);
        return eventExists;
    }
}
```

canActivate 最后返回 boolean 值。

#### canDeactivate

通过方法实现 canDeactivate

```typescript
{ path: 'events/new', component: CreateEventComponent, canDeactivate: ['canDeactivateCreateEvent'] },
```

方法需要注入 providers，且不能使用缩写:

```typescript
  providers: [
    {
      provide: 'canDeactivateCreateEvent',
      useValue: checkDirytState
    }
  ]
```

useValue 是方法，方法可以放在任何文件里：

```typescript
export function checkDirytState(component: CreateEventComponent) {
  if (component.isDirty) {
    return window.confirm('You have not saved this event, do you really want to cancel?')
  }
  return true;
}
```

### active link

```html
<a [routerLink]="['/events']" routerLinkActive="active" [routerLinkActiveOptions]="{exact:true}">All Events</a>
```

routerLinkActive="active" 添加在含有 routerlink 的标签上，当路由与标签指定的路由匹配，则添加 active class。

默认匹配路由字符串策略是 start with。

添加 ` [routerLinkActiveOptions]="{exact:true}"` 是完全符合。

### feature module 或 lazy loadable module 

与 appModule 的区别：

1. 不导入 BrowserModule
2. 导入的 RouterModule 调用 forChild 方法

## 表单

分为两种：

1. Template-based
2. Model-based 也称 reactive forms

表单如果都通过事件绑定的方式太复杂，所以引入 ngModel。

[(ngModel)] 双向绑定使用双括号 banana box

使用时根据需要决定如何绑定。

### Template-based Form

需要引入 FormsModule 注册到 module 中。

```html
<form #loginForm="ngForm" (ngSubmit)="login(loginForm.value)" autocomplete="off">
	<input (ngModel)="userName" name="userName" id="userName" type="text" class="form-control" placeholder="User Name..." />
</form>
```

通过模板变量声明 Form，`#loginForm="ngForm"` 指定该 Form 为 Angular 表单。

使用 ngSubmit 提交，参数是模板变量的值 `loginForm.value`。

用 name 声明 Form 中的成员属性。

#### 表单验证

```html
<em *ngIf="newEventForm.controls.date?.invalid && (newEventForm.controls.date?.touched)">Required</em>
```

基于模板的表单验证特别长，因为 formControl 不是 public 类型，它是 Form 的属性，所以在表单验证时特别繁琐。

### Model-based Form

还需要引入 ReactiveFormsModule 注册到 module 中。

步骤如下：

1. 创建每个 FormControl
2. 创建一个 FormGroup
3. 为 FormGroup 配置 FormControl
4. 给 HTML 中的 form 绑定 FormGroup
5. 给 HTML 中的 field 绑定 FormControlName

formControlName 绑定时不需要带任何括号，只是指定属性名

#### 表单验证

表单验证是 FormControl 的第二个参数：

```typescript
let firstName = new FormControl(this.auth.currentUser.firstName, Validators.required);
```

如果有多个验证使用数组。

### 自定义表单验证

#### validator 是什么

validator 就是一个函数：

- 返回 null 时 valid
- 返回 error 对象时 invalid

error 对象 { key: value }

#### 自定义表单验证方法

```typescript
export function restrictedWords(...words) {
  return (control: FormControl): { [key: string]: any } => {
    if (!words) {
      return null;
    }
    let invalidWords = words.map(w => control.value.includes(w) ? w : null)
      .filter(w => w !== null);
    return invalidWords && invalidWords.length > 0
      ? { 'restrictedWords': invalidWords.join(', ') }
      : null;
  }
}
```

这里为了使用 FormControl 使用了闭包。

注意 key 值 `restrictedWords` 与方法名一致。

