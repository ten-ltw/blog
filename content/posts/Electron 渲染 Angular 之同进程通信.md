---
title: "Electron 渲染 Angular 之同进程通信"
date: 2022-06-22T22:58:42+08:00
draft: true
---

测试同事发现了一个问题，之前完成的导出文件一直是空的，发现在使用 `showSaveFilePicker` 时发生了异常:

{{< admonition failure >}}
NotAllowedError: The request is not allowed by the user agent or the platform in the current context.
{{< /admonition >}}

查到 [electron github issue](https://github.com/electron/electron/issues/28422) 中，这个问题一直存在，FileSystem 的 API 在 electron 11 中就添加了进来，但是可能是出于安全原因，到 electron 19 都没有放开这个权限能够被 electron 渲染的静态单页面来实现。

所以只能通过 angular 的渲染进程来通知 electron 来保存文件。