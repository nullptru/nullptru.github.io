---
title: React源码Scheduler（三）React的调度算法实现
date: 2019-07-21 17:57:19
tags: React
categories:
	- 技术文档
---
本文源码基于 `React 16.8.6 (March 27, 2019)`，仅记录一些个人阅读源码的分享与体会。

[欢迎大家交流和探讨](https://geasscn.com)

+ [React 源码 Scheduler（一）浏览器的调度](https://geasscn.com/2019/07/06/React%20%E6%BA%90%E7%A0%81Scheduler%EF%BC%88%E4%B8%80%EF%BC%89%E6%B5%8F%E8%A7%88%E5%99%A8%E7%9A%84%E8%B0%83%E5%BA%A6/)
+ [React 源码 Scheduler (二) React 的调度流程](https://geasscn.com/2019/07/13/React%20%E6%BA%90%E7%A0%81Scheduler%EF%BC%88%E4%BA%8C%EF%BC%89React%E7%9A%84%E8%B0%83%E5%BA%A6%E6%B5%81%E7%A8%8B/)
+ React 源码 Scheduler（三）React的调度算法实现

### 前言
在上两节中，笔者介绍了在浏览器中存在的 `requestAnimationFrame` 和 `requestIdleCallback` 两种调度方法及在 React 中一个任务的调度流程。同时，读者也了解了 React 团队采用了 `requestIdleCallback` 的形式实现调度，但由于[该 API 的兼容性和实际渲染频率](https://github.com/facebook/react/issues/13206?source=post_page---------------------------#issuecomment-418923831)的因素，React 团队最终自己实现了一个内部的该函数。

在本节中，我们就将详细介绍在浏览器中，React 内部是如何实现自己的调度算法。

### 概览
本文中所涉及的源码位于 `packages/scheduler/src/forks/SchedulerHostConfig.default.js`。

为了更好的对文件整体有个好的认知，我们依旧从类图入手。

![](https://geasscn.com/images/85fa385992ce7bb5c586ddeb8f1569ff_hd.png)

从成员变量的 `rAFID` 与 `rAFTimeoutID` 来看，React 使用了 `requestAnimationFrame` 与 `setTimeout` 两种方案模拟 `requestIdleCallback`。除去几个 bool 变量外，我们需要关注 4 个时间标识`timeoutTime`,`frameDeadline`,`previourFrameTime`,`activeFrameTime` 和 1 个 [MessageChannel](https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel)。通过任务到期时间及当前帧与上一帧的时间信息，计算分片时间。之后通过 MessageChannel 的 microTask 执行异步调度，这就是 React 调度实现的一个大体思路。

> MessageChannel 形成一个通信管道，允许数据从一端透传到另一端，应用于 websocket 数据传递。

### 源码解析
#### requestHostCallback
调度的入口函数为[requestHostCallback](https://geasscn.com/2019/07/13/React%20%E6%BA%90%E7%A0%81Scheduler%EF%BC%88%E4%BA%8C%EF%BC%89React%E7%9A%84%E8%B0%83%E5%BA%A6%E6%B5%81%E7%A8%8B/#scheduleHostCallbackIfNeeded)，也就是笔者在上一节中遗漏的几个函数之一。函数接收外部传入的需调度的任务和超时时间来决定任务是否立即执行或者开启调度任务。

```javascript
  requestHostCallback = function(callback, absoluteTimeout) {
    scheduledHostCallback = callback;
    timeoutTime = absoluteTimeout;
    // 已经超时就直接执行无需调度
    if (isFlushingHostCallback || absoluteTimeout < 0) {
      port.postMessage(undefined);
    } else if (!isAnimationFrameScheduled) { // 未超时且没调度开启一个调度任务
      isAnimationFrameScheduled = true;
      requestAnimationFrameWithTimeout(animationTick);
    }
  };
```
通过源码可以看到开启调度任务的方法实际为 MessageChannel 通信，之所以不采用直接调用方法。笔者猜想一方面是因为调度函数可能存在异步逻辑等阻碍线程执行，另一方面在 js 事件循环队列里 microTask 的任务优先级高，便于加快执行。

我们暂且跳过 `postMessage` 的内容，先看看需要调度时执行的逻辑。

#### requestAnimationFrameWithTimeout

```javascript
const requestAnimationFrameWithTimeout = function(callback) {
  // 同时调度 setTimeout 和 requestAnimationFrame
  rAFID = localRequestAnimationFrame(function(timestamp) {
    // 取消 timeout
    localClearTimeout(rAFTimeoutID);
    callback(timestamp);
  });
  rAFTimeoutID = localSetTimeout(function() {
    // 取消 requestAnimationFrame
    localCancelAnimationFrame(rAFID);
    callback(getCurrentTime());
  }, 100);
};
```
这里我们看到，在调度函数中同时使用了 `setTimeout` 和 `requestAnimationFrame`。一般而言，第一眼看到都会产生：因为兼容性问题，用 `setTimout` 作为兜底方案的想法。暂且不说 React 实际先通过能力检查校验过方法存在，仅 `setTimeout` 100ms 的参数就告诉了我们降级假设是错误的。那么，是否存在 `requestAnimationFrame` 无法生效的场景呢？ `requestAnimationFrame` 是根据刷新率每一帧进行调用，当页面位于后台不可见时，实际上函数是不会被调用的。因此，为了保证页面在后台仍能成功执行任务，采用了低频率的 `setTimeout` 方案作为共存。

这里还有一点，对于当前时间的选择，采用的方案是以 `Performance.now()` 优先，`Date.now()` 兜底的策略。对此，[stackoverflow](https://stackoverflow.com/questions/30795525/performance-now-vs-date-now) 上有关解释表示因为 `Performance.now()` 具有更高的精确度。至于是否还有其它方面的考量，欢迎阐述你的想法。

#### animationTick
`animationTick` 顾名思义如时钟滴答般记录动画的时长，也是 React 调度里对于各个帧时长的计算之处。总的来说，笔者觉得这是一个挺有趣的函数。

```javascript
const animationTick = function(rafTime) {
  if (scheduledHostCallback !== null) {
    requestAnimationFrameWithTimeout(animationTick);
  } else { /*...*/ return; }
  // frameDeadline： 上一帧的 rafTime + activeFrameTime
  let nextFrameTime = rafTime - frameDeadline + activeFrameTime;
  if (
    nextFrameTime < activeFrameTime &&
    previousFrameTime < activeFrameTime &&
    !fpsLocked
  ) {
    if (nextFrameTime < 8) {
      // 防御性代码，不支持超过 120hz 的频率
      nextFrameTime = 8;
    }
    // 启发式动态调整 activeFrameTime
    activeFrameTime =
      nextFrameTime < previousFrameTime ? previousFrameTime : nextFrameTime;
  } else {
    previousFrameTime = nextFrameTime;
  }
  frameDeadline = rafTime + activeFrameTime;
  if (!isMessageEventScheduled) {
    isMessageEventScheduled = true;
    port.postMessage(undefined);
  }
};
```

笔者之前写这类递归函数，都是在函数尾写的，而 React 在函数开头的执行顿时眼前一亮。对此，代码注释的官方解释是这样的。

> 将回调放在帧的首部确保了它会在最邻近的帧内被调用  
> 如果将回调放在函数尾部，将会冒浏览器跳过一帧直到下下帧才触发回调的风险

不得不说，这个细节值得我们学习。

接着往下，我们看到了 React 内部对于每帧执行 js 任务的耗时的计算公式。`下一帧的时间(nextFrameTime) = 当前时间(rafTime) - 上一帧的时间(frameDeadline) + 活跃帧的时间(activeFrameTime)`。而`activeFrameTime` 有个初始值为 33，也就是说 1s 约渲染 30 帧。而 React 官方支持的最高帧数是 120。因此必然需要一个启发式机制来根据屏幕刷新率更改该值，也就是接下来的代码段。

React 团队认为，当连续两个帧的执行时间，都小于我们预设的 `activeFrameTime`，那么我们认为我们处于一个高刷新率的机器上运行，因此动态调整为前两帧中较大一帧的时间。

之后便是记录当前帧的终止时间，一个很简单的公式：`当前时间(rafTime) + 活跃帧的时间(activeFrameTime)`。

最后，在没有任务正处理情况下通过 `postMessage` 进行任务的调度处理。

#### onmessage
笔者说过 React 通过 MessageChannel 进行异步调度，通过一个端口 `postMessage` 发送消息，另一个端口接收处理消息。

```javascript
channel.port1.onmessage = function(event) {
	isMessageEventScheduled = false;
	
	const prevScheduledCallback = scheduledHostCallback;
	const prevTimeoutTime = timeoutTime;
	scheduledHostCallback = null;
	timeoutTime = -1;
	
	const currentTime = getCurrentTime();
	
	let didTimeout = false;
	// 如果没时间了
	if (frameDeadline - currentTime <= 0) {
	  if (prevTimeoutTime !== -1 && prevTimeoutTime <= currentTime) { // 判断是否任务超时
	    didTimeout = true;
	  } else { // 没时间且没任务未超时，重新调度
	    if (!isAnimationFrameScheduled) {
	      isAnimationFrameScheduled = true;
	      requestAnimationFrameWithTimeout(animationTick);
	    }
	    // 恢复现场
	    scheduledHostCallback = prevScheduledCallback;
	    timeoutTime = prevTimeoutTime;
	    return;
	  }
	}
	// 如果有时间或者超时了，执行任务
	if (prevScheduledCallback !== null) {
	  isFlushingHostCallback = true;
	  try {
	    prevScheduledCallback(didTimeout);
	  } finally {
	    isFlushingHostCallback = false;
	  }
	}
};
```

在看过上一节的读者其实会发现，这段和 `Scheduler` 里的 `flushWork` 其实有异曲同工之处。首先获取 `requestHostCallback` 里获得的回调方法，在有空闲时间或者任务超时的时候执行任务，在没时间未超时的情况下重新进行调度，等待下一帧的机会。

### 总结

同样的，让笔者以一个流程图进行总结。

![](https://geasscn.com/images/eea41e3e1f09ceb96eb00d0924a44aa0_hd.png)

至此，关于 React 调度相关的源码阅读也就到此一段落了。其实从源码的阅读上笔者发现，大部分情况下软件开发框架设计其实用不到很高深的数学功底（当然数学好也是很重要的）。更多的关注点在于对细节的把控，如递归回调的位置，启发式帧时间的修改等。还有就是对功能模块的抽象，这才是一个项目可发展的根基。