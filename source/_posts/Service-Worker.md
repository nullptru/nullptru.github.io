---
title: Service Worker
date: 2018-06-14 23:31:24
tags: 
  - 前端
  - JavaScript
categories:
	- 技术文档
---

要了解Service Worker相关知识，需要对于Web Worker有一定基础的了解。

## 什么是Service Worker
Service Worker(简称为SW)是基于Web Worker的事件驱动的，他们执行的机制都是新开一个线程去处理一些额外的，以前不能直接处理的任务。目前SW主要功能包括了浏览器端的请求拦截代理，推送通知和后台同步等一系列功能...目前使用最多的应该是利用SW实现的本地代理缓存，从而实现良好的离线体验，这也是PWA(Progressive Web Application)技术的基础。(具体会单独开一篇讲

今天就让我们聊一聊Web的离线缓存和SW所带来的解决方案。

## Web缓存的前世今生
在SW技术诞生之前，要在前端进行数据缓存，方案无非以下几种。

### 基于浏览器Cookie机制
`Cookie` 应该是前端缓存数据最原始的方案了。但 `cookie` 的设计本质就只是为了网络请求头中附带部分验证信息等，根本不存在做前端缓存的机制。

### 基于浏览器localStorage机制
[localStorage](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/localStorage) 相对于 `cookie` 扩大了容量，本地持久化的能力及方便易用的API也被广泛接受。但在应用缓存上，对静态文件的无力和容量的限制仍然是应用缓存的一个瓶颈。

### 基于浏览器IndexDB
`IndexDB` 作为一个前端 `NoSql` 数据库。我们可以在离线情况下从中获取对应的数据信息，在数据持久化存储方面，可谓是十分实用了。然而与之前几个存储方案相同的是，无法对静态文件进行存储。

### 基于浏览器Header的缓存
通过浏览器头 `Last-Modified`, `Etag` 和 `expires` 等元信息，对于 `online` 的情况下，在这些值相同的情况下可以直接从缓存中读取数据，帮助我们减少大量网络请求。然而在 `offline` 的情况下我们就又可以去见那只可爱的小恐龙了～(Chrome的offline游戏

### APP Cache
正主来了，H5的 [APP Cache API](https://developer.mozilla.org/en-US/docs/Web/HTML/Using_the_application_cache) 的诞生似乎为web离线存储带来了一线曙光。通过 `Manifest` 文件内对静态文件的引用声明，在第一次访问网页的时候会对文件内内容进行本地缓存，之后便可通过缓存读取相应文件，即使在离线环境下也能进行访问，然后配合IndexDB做数据级的存储，想想就觉得很棒是不是。然而，之后暴露的[局限性](http://alistapart.com/article/application-cache-is-a-douchebag)真是让人对其又爱又恨。

铺垫了那么多，让我们看看目前真正的正主又是怎么做的吧。

## SW的生命周期

![生命周期](https://image-static.segmentfault.com/147/939/1479397622-567b4b079ee5d_articlex)

从上图中我们可以看到一个Service Worker的生命周期由以下几个部分组成： 
1. install：初始阶段，做一些静态资源的存储 
2. activated：新旧 `sw` 更新时候的生命周期，处理一些历史缓存 
3. fetch/message：动态代理网络请求 
4. terminated： `sw` 终止

现在就让我们具体来一步步分析

### 注册sw脚本
首先在主线程中加载 `sw` 的脚本文件，在这里我们做一个浏览器能力检测以避免浏览器不支持 `sw` 。如果可用，则在页面加载后通过 `register` 方法注册位于 `/service-worker.js` 的服务工作线程。这里有一点需要注意的就是，本例中 `sw` 文件位于网站根目录下，也就是说 `sw` 将接收所有来自该网站的请求，如果注册在 `/example/sw.js` 下，则只能接收来自 `/example` 域下的请求，例如 `/example/article` 而无法接收 `/article` 请求。而 `register` 第二个参数可以自定义 `sw` 可访问的域，默认值为 `sw` 文件所在域。

```
// install sw
if ('serviceWorker' in navigator) {
  navigator.serviceWorker
    .register('/service-worker.js')
    .then(() => {
      console.log('service worker registration successful'); 
    })
    .catch((err) => {
      console.warn('service worker registration failed', err.message);
    });
}
```

### 安装SW
在主线程中注册完 `sw` 后，便会进入 `sw` 的第一个 `install` 生命周期。一般性而言，我们会在这个步骤中通过 `CacheStorageAPI` 对网站的静态资源进行缓存操作。

```
// service-worker.js
const CACHE_NAME = 'sw-cache-v1';
const filesToCache = [
  '/',
  '/styles/main.css',
  '/script/main.js'
];

self.addEventListener('install', function(event) {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(cache => cache.addAll(filesToCache);)
  );
});
```

通过event.waitUntil保证Promise事件的完成，内部代码就很简单了，根据 `CACHE_NAME` 打开对应缓存，之后通过 `addAll` 将需缓存的文件添加进去就ok了。

在这个过程中，有一点需要格外注意。那就是在 `addAll` 的过程中，一旦一个文件下载失败，整个 `sw` 的 `install` 生命周期便会终止，虽然得以与浏览器机制，即使失败在下次运行时仍会进行 `sw` 安装，但我们仍需确保 `filesToCache` 中的依赖项不应太长，对应资源均可访问。

在这个过程中，可以通过建立非依赖性存储对这一过程优化，具体可以参考Jake的[这篇文章](https://developers.google.com/web/fundamentals/instant-and-offline/offline-cookbook/#on-install-not)。

### 更新SW
当用户访问网站，浏览器检测到 `sw` 文件发生字节差异时，便会将其视为新 `sw` 。从而执行 `install` 和 `active` 生命周期。在 `active` 生命周期中，一般进行新旧缓存的更替操作。至于为什么不在 `install` 事件中完成的原因在于，如果在安装步骤中清除了任何旧缓存，则继续控制所有当前页面的任何旧 `sw` 将突然无法从缓存中提供文件。

```
// service-worker.js
self.addEventListener('activate', function(event) {

  const cacheWhitelist = ['cache-v1'];

  event.waitUntil(
    caches.keys().then(function(cacheNames) {
      return Promise.all(
        cacheNames.map(function(cacheName) {
          if (cacheWhitelist.indexOf(cacheName) === -1) {
            return caches.delete(cacheName);
          }
        })
      );
    })
  );
});
```
在该例中，我们建立了一个 `cache` 白名单，每次更新 `sw` 的时候，都将删除所有不在白名单中的同域 `cache` 缓存。

### Fetch的动态缓存
在完成 `sw` 的安装后， `sw` 便可以通过 `fetch` 事件监听注册域内所有网站的请求事件。由此我们便可以进行一些动态的 `fetch` 缓存和对请求的编程性控制。

```
// service-worker.js
self.addEventListener('fetch', function(event) {
  event.respondWith(
    caches.match(event.request)
      .then(function(response) {
        // Cache hit - return response
        if (response) {
          return response;
        }
        // IMPORTANT: Clone the request. A request is a stream and
        // can only be consumed once. Since we are consuming this
        // once by cache and once by the browser for fetch, we need
        // to clone the response.
        const fetchRequest = event.request.clone();

        return fetch(fetchRequest).then(
          (response) => {
            // Check if we received a valid response
            if(!response || response.status !== 200 || response.type !== 'basic') {
              return response;
            }

            // IMPORTANT: Clone the response. A response is a stream
            // and because we want the browser to consume the response
            // as well as the cache consuming the response, we need
            // to clone it so we have two streams.
            var responseToCache = response.clone();

            caches.open(CACHE_NAME)
              .then(function(cache) {
                cache.put(event.request, responseToCache);
              });

            return response;
          }
        );
      }
    )
  );
});
```
在这个示例中，我们实现了一种 `cacheFirst` 的策略。当监听到一个请求，先在 `cache` 中查看是否已被缓存，如果已缓存则直接返回，否则真正发送请求并缓存请求结果供下次使用。除此外，还有 `networkFirst`，`cacheOnly` 等一系列缓存策略，在实际使用过程中应当根据具体情况选择不同方案，对于这些缓存的具体实现和理解，同样可以阅读Jake的[offline-cookbook这篇文章](https://developers.google.com/web/fundamentals/instant-and-offline/offline-cookbook/)。

## 局限性
到目前为止，我们针对 `sw` 的生命周期进行了相应缓存方案的介绍说明。通过 `install` 事件中对静态资源的缓存和对请求的监听拦截，我们可以将近乎所有get请求事件缓存起来以实现离线可使用应用的开发。这对于资讯类服务而言是十分有帮助的。

然而，一门技术当然存在相应的缺点不足和适用性，目前根据实践和一些文章说明。对 `sw` 的使用列出以下几点说明： 
1. 必须采用可信任的 `https` 协议， `sw` 所在网站和存储的内容都要经过 `https` 协议访问。 
2. 缓存策略的选择设计，不当的缓存策略会导致即使页面更新依旧无法获取最新资源。 
3. 浏览器支持性，虽然新版本主流浏览器基本支持，但考虑到不支持 `sw` 功能的浏览器所占比例， `sw` 应作为渐进式功能增强方案而非必需品。 
4. 因为考虑推送功能，所以 `sw` 的生命周期不仅仅在打开页面的时候才存在，则需尽量避免文件全局变量的使用，防止被外界干扰(具体案例见 [Service Worker初体验](http://www.alloyteam.com/2016/01/9274/)) 
5. 调试困难，目前并没有良好的调试工具进行测试

## 参考文献
1. [Service Worker——Google Developer](https://developers.google.com/web/fundamentals/primers/service-workers/?hl=zh-cn)
2. [Service Worker初体验](http://www.alloyteam.com/2016/01/9274/)
3. [offline-cookbook](https://developers.google.com/web/fundamentals/instant-and-offline/offline-cookbook/)
4. [Service Worker 简介](https://lavas.baidu.com/pwa/offline-and-cache-loading/service-worker/service-worker-introduction)
