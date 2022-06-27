---
title: "在 angular 的 library 中添加静态 XML 作为静态 HTML 渲染"
date: 2022-06-27T13:36:22+08:00
categories: ["Frontend"]
tags: ["Angular"]
hiddenFromHomePage: true
---

在当前项目中 `End User License Agreement` 是一个静态 XML 文件，在打包在了根目录下直接通过 Angular JS 渲染。
本次开发是升级这个应用为 Angular，并且将其以 Library 的形式引入公司的一个前端的壳内，成为公司整个项目的一个可订阅的功能。
总结需求就是在 Angular 前端工程中获取一个子 Libaray 的静态文件。

## 单页面获取 XML 的访问权限

第一步要通过 http 就能够直接访问到 xml。[http://localhost:4200/eula.xml](http://localhost:4200/eula.xml)

### 尝试直接在工程中添加子工程 assets 的目录

项目背景是一个独立的 Windows 程序，本应在这次 release 时进入 docker，成为公司整个软件中的一个订阅，而因为其他项目的延迟保持本项目独立。
也就是说本次 release 我们的前端项目还是一个完整的应用直接进行发布，所以在整个项目的 `angular.json` 文件中直接为主工程添加子 Library 的静态文件目录，这样未来在删除主项目的时候不会使静态文件丢失。
修改后编译失败，错误如下：

> An unhandled exception occurred: The projects/..../src/assets asset path must start with the project source root.

也就是说 angular 不允许将 root 目录以外的路径设为资源文件进行编译。那么这个偷懒的方法肯定不行了。

### 调查目前渲染出来的图片是如何访问的

接下来调查下 angular serve 和 build 后这些 assets 到底在什么地方。
首先，`ng serve` 起来的 webpack server 中，可以通过根目录直接访问图片文件。devtool 中也可看到，在 `localhost：4200` 节点下确实有所有的添加在 assets 下面的图片。
以子 Library 的 assets 目录为关键字，在所有代码中搜索发现，子 Library 目录下，有个配置文件 `ng-package.json` 中面有子 Library 的 assets 的配置。
那么将 XML 文件放到子 Library 中，build 成功，serve 成功。build output 文件中有静态 XML 文件，但是画面无法访问到这个静态文件。
这时想起来我们的子 Library 并不是真个页面的入口，真正渲染的 index 是外面的这个壳，也就是主工程，再次查看 build 结果中主工程的目录结构。
> build 根目录下除了主目录的 assets 目录外其他的图片都直接放在了根目录下。
也就是说子 Library 的 assets 下文件都会自动打包到主工程的编译结果的根目录下。但是我添加的 XML 文件就没有被打包进来。

### 另外一种方式让 webpack 打包

一直没有调查出来 angular 是如何配置才会将子目录的 assets 文件夹内的静态 resources 直接添加到根目录下，猜测这里可能会有个 filter 只添加图片 extension。这里结合前两种方式，直接将子 Library 的文件打包到主工程的根目录下，在 `angular.json` 的主项目节点的中配置 assets：
```JSON
{
    "glob":"**/*.xml",
    "input": "projects/..your library../src/assets",
    "output": "/"
}
```
这样，XML 文件会被原封不动的打包到主项目的根目录下，暂时先这样解决，回头如果有安全问题再继续调查。

## 在 innterhtml 指令渲染 XML

此时已经能够通过 http 获取 xml 文件的数据，就通过 Angular 提供的 `HttpClient` service 来使用 http get 方法来获取 xml 文件内容。

### 获取 xml 数据
将 `HttpClient` 注入为 `http` 后使用如下代码：

```typescript
    public elua: Observable<string>;
    // ...
    ngInit(): void {
        this.eula = this.http.get('eula.xml');
    }
```

报错，这里说 `get` 的返回值是 Object，不能赋值给 string。
瞅了一眼 get 的描述文件很有意思，这里放下源码：
```typescript  
   /**
   * Constructs a `GET` request that interprets the body as a JSON object and
   * returns the response body as a JSON object.
   *
   * @param url     The endpoint URL.
   * @param options The HTTP options to send with the request.
   *
   *
   * @return An `Observable` of the response body as a JSON object.
   */
  get(url: string, options?: {
    headers?: HttpHeaders|{[header: string]: string | string[]},
    context?: HttpContext,
    observe?: 'body',
    params?: HttpParams|
          {[param: string]: string | number | boolean | ReadonlyArray<string|number|boolean>},
    reportProgress?: boolean,
    responseType?: 'json',
    withCredentials?: boolean,
  }): Observable<Object>;
```
这里印象中 http 的返回值应该是个 any 的 response，因为 http service 并不知道用户请求的是什么，他可能是任何东西。
上下翻阅可以发现相关的 get 方法的重载，返回值通过形参 `options` 的 `responseType` 来决定。
[github httpclient get](https://github.com/angular/angular/blob/3a60063a54d850c50ce962a8a39ce01cfee71398/packages/common/http/src/client.ts#L1179) 的实际实现果然还是直接用 any 返回，但是这样的描述文件就能达到字符串配置来控制返回值，这在类型安全中是非常实用的，如果不配置正确的 options，就不会得到正确的返回值，而这个错误不用在 runtime 中去解决，而在编译时就能够发现。
尝试自己写一个这样的 typescript 例子：
```typescript
interface ITester {
    doAction(options: { action: 'delete' }): DeleteResult;
    doAction(options: { action: 'update' }): UpdateResult;
    doAction(options: { action: 'insert' }): InsertResult;
}
class Tester implements ITester {
    doAction(options: IOptions): any {
        return {};
    }
}
interface IOptions {
    action: 'delete' | 'update' | 'insert';
}
class DeleteResult { }
class UpdateResult { }
class InsertResult { }

let tester:ITester = new Tester();

let result:InsertResult = tester.doAction({ action: 'delete' });
```
并没有成功，再继续调查。