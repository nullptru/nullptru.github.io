---
title: React 源码 Scheduler（一）浏览器的调度
date: 2019-07-06 22:01:08
tags:
  - React
---

本文源码基于 `React 16.8.6 (March 27, 2019)`，仅记录一些个人阅读源码的分享与体会。

[欢迎大家交流和探讨](https://geasscn.com)

### 背景

`Schedule` 即任务的调度，我们知道 JavaScript 是单线程运行的。因此，浏览器无法同时相应 JS 任务与用户的 UI 操作，如此在执行 UI 操作的时候，便会带给用户一定卡顿感，也就是我们所谓的「丢帧」。

对此情况，React 采用的是时间分片的策略，将任务细化为不同优先级，利用浏览器的空闲时间进行任务的执行以保证 UI 操作的流畅。浏览器的调度 API 主要分为两种，分别是高优先级的 `requestAnimationFrame` 与低优先级的 `requestIdleCallback`。

### RequestAnimationFrame
[`requestAnimationFrame`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestAnimationFrame) 在每一帧的开始阶段执行，一般用来进行复杂动画的绘制。该函数接受一个接收 [`DOMHighResTimeStamp `](https://developer.mozilla.org/zh-CN/docs/Web/API/DOMHighResTimeStamp) 参数的 `callback` 函数作为参数，返回一个 `requestId` 供 [`cancelAnimationFrame`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/cancelAnimationFrame) 以取消。

由于该函数每帧开始必执行，因此我们可以基于此，在每帧开始时执行一定任务，实现一个简单的时间分片调度。

```javascript
// create 1000 tasks 
const tasks = Array.from({ length: 1000 }, () => () => { console.log('task run'); })

const doTasks = (fromIndex = 0) => {
	const start = Date.now();
	let i = fromIndex;
	let end;
	
	do {
		tasks[i++](); // do task
		end = Date.now();
	} while(i < tasks.length && end - start < 20); // Do tasks in 20ms
	
	console.log('tasks remain: ', 1000 - i);
	// if remaining tasks exsis when timeout. Run at next frame
	if (i < tasks.length) {
		requestAnimationFrame(doTasks.bind(null, i));
	}
}

// start tasks scheduler
requestAnimationFrame(doTasks.bind(null, 0))

/** 
output:
	168 task run
	tasks remain:  832
	178 task run
	asks remain:  654
	162 task run
	tasks remain:  492
	119 task run
	tasks remain:  373
	158 task run
	tasks remain:  215
	87 task run
	tasks remain:  128
	125 task run
	tasks remain:  3
	3 task run
	tasks remain:  0
*/
```
我们可以看到，通过 `requestAnimationFrame` 的调度，我们实现了一个简单的时间分片功能，在每帧留出 20ms 进行 js 的任务执行。但这时候就引入一个问题：20ms 是如何确定的？如果一个时间点任务实际需要耗时小于 20ms，那多出的时间岂不是浪费了？为了解决这个问题，就引出了我们的第二个调度 API: `requestIdleCallback`。

### RequestIdleCallback
与每帧执行的 `requestAnimationFrame` 相对，[`requestIdleCallback`](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/requestIdleCallback) 是一个低优先级调度，当且仅当浏览器空闲时才会执行任务的调度。这就解决了之前例子里如何确定任务应该执行时间这一问题。`requestIdleCallback` 接收两个参数。第一个参数为接受一个 [`IdleDeadline`](https://developer.mozilla.org/zh-CN/docs/Web/API/IdleDeadline)参数的 `callback` 函数，第二个参数为可选的 `options`，包含一个 `timeout` 配置项，指定该回调的超时时间，以保证任务不至于饿死。由此，我们便可基于此对上述代码进行修改。

```
const tasks = Array.from({ length: 1000 }, () => () => { console.log('task run'); })
const doTasks = (fromIndex = 0, idleDeadline) => {
	let i = fromIndex;
	let end;
	
	console.log('time remains: ', idleDeadline.timeRemaining());
	do {
		tasks[i++](); // do task
	} while(i < tasks.length && idleDeadline.timeRemaining() > 0); // Do tasks in 20ms
	
	console.log('tasks remain: ', 1000 - i);
	// if remaining tasks exsis when timeout. Run at next frame
	if (i < tasks.length) {
		requestIdleCallback(doTasks.bind(null, i));
	}
}

// start tasks scheduler
requestIdleCallback(doTasks.bind(null, 0))

/**
output:
	time remains:  49.970000000000006
	360 task run
	tasks remain:  640
	time remains:  49.77
	395 task run
	tasks remain:  245
	time remains:  29.255000000000003
	215 task run
	tasks remain:  30
	time remains:  49.96000000000001
	30 task run
	tasks remain:  0
*/
```
第二个版本的代码，我们通过 `idleDeadline.timeRemaining()` 获取当前剩余时间进行任务的调度。在复杂情况下，会出现浏览器空闲时间过少导致任务堆积问题，这时候第二个参数的 `timeout` 配置就派上用场了。有兴趣的小伙伴可以自己试试。

在 React 中的任务调度，也采用了 `requestIdleCallback` 实现调度，但由于[该 API 的兼容性问题](https://www.caniuse.com/#search=requestIdleCallback)（Safari 这个新生代的 IE），FB 内部自己基于 `requestAnimationFrame` 实现了一个 `requestIdleCallback` 的 polyfill。我们将在下一节中进行介绍（如果还有的话）。
