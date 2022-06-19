---
title: "JS Promise States and Fates"
date: 2021-02-10T20:57:40+08:00
categories: ["Frontend"]
---

# States and Fates

This document helps clarify the different adjectives surrounding promises, by dividing them up into two categories: **states** and **fates**.

> 本文档阐明围绕 Promise 的各种形容词，把它们分为两部分：**状态**和**命运**。

## States

Promises have three possible mutually exclusive states: fulfilled, rejected, and pending.

> Promise 有三种可能的互斥独立状态：已兑现、已拒绝、待定。

- A promise is *fulfilled* if `promise.then(f)` will call `f` "as soon as possible."

> - 如果 `promise.then(f)` 会马上调用 `f`，那么 promise 是已兑现状态。

- A promise is *rejected* if `promise.then(undefined, r)` will call `r` "as soon as possible."

> - 如果 `promise.then(undefined, r)` 会马上调用 `r`，那么 promise 是已拒绝状态。

- A promise is *pending* if it is neither fulfilled nor rejected.

> - 如果它既不是已兑现也不是已拒绝，那么 promise 是待定状态。

We say that a promise is *settled* if it is not pending, i.e. if it is either fulfilled or rejected. Being settled is not a state, just a linguistic convenience.

> 如果 promise 不是待定状态，我们说它*已敲定*，无论它是已兑现还是已拒绝。已敲定不是一个状态，知识一个语言上的便利。

## Fates

Promises have two possible mutually exclusive fates: resolved, and unresolved.

> Promise 有两种可能的互斥独立命运：已决议，未决议。

- A promise is *resolved* if trying to resolve or reject it has no effect, i.e. the promise has been "locked in" to either follow another promise, or has been fulfilled or rejected.

> - 如果 promise 已决议，尝试决议或拒绝它是没有任何效果，promise 被接下来其他 promise 锁定，或者已被兑现或拒绝。

- A promise is *unresolved* if it is not resolved, i.e. if trying to resolve or reject it will have an impact on the promise.

> - 如果 promise 未决议，如果尝试兑现或拒绝会对 promise 产生影响。
