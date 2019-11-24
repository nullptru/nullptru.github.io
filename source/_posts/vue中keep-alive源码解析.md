---
title: vue中keep-alive源码解析
date: 2019-11-25 01:16:37
tags:
  - 源码解析
categories:
	- 技术文档
---
本文所论述内容为 vue 2.6.8 版本。

## keep-alive

[keep-alive](https://cn.vuejs.org/v2/guide/components-dynamic-async.html) 是 vue 提供的一个组件级缓存的方案，作为一个不渲染真实 dom 节点的组件，keep-alive 将缓存其第一个子元素。从而保证页面跳转返回时保留页面原有的状态。

## 源码实现

因为 keep-alive 内部涉及诸如 actived 等生命周期，我们暂不展开细讲，仅简单介绍该组件所做的一些事。该部分源码位于 `[vue/src/core/components/keep-alive.js](https://github.com/vuejs/vue/blob/dev/src/core/components/keep-alive.js)` 。

先看下源码大致结构，本质上也就是一个 vue 组件。

```javascript
export default {
  name: 'keep-alive',
  abstract: true,

  props: {
    include: patternTypes,
    exclude: patternTypes,
    max: [String, Number]
  },

  created () {
    this.cache = Object.create(null)
    this.keys = []
  },

  destroyed () {
    for (const key in this.cache) {
      pruneCacheEntry(this.cache, key, this.keys)
    }
  },

  mounted () {
    this.$watch('include', val => {
      pruneCache(this, name => matches(val, name))
    })
    this.$watch('exclude', val => {
      pruneCache(this, name => !matches(val, name))
    })
  },

  render () {
    const slot = this.$slots.default
    const vnode: VNode = getFirstComponentChild(slot)
    // ...
    return vnode || (slot && slot[0])
  }
}
```

略去细节，我们可以看到组件定义 `abstract` 为 true，代表该组件并未真实 dom 节点。组件接收 3 个参数，include 和 exclude 作为管理 cache key 值的依据，并在 mounted 的生命周期中设置 watch 来及时刷新缓存。同时 max 作为最大缓存数进行了缓存的限制。在 created 生命周期中，创建组件缓存对象，注意到这里并不是用 data 设置的原因是缓存对象并不是响应式的，这也告诉我们一点：在开发中，仅仅需要做响应式处理的数据才存放在 data 对象中。在 destroyed 函数中遍历缓存进行了缓存清理。此外就是一个 render 函数，接下来我们将详细说明下 keep-alive 的 render 渲染。

### render

```javascript
render () {
    const slot = this.$slots.default
    const vnode: VNode = getFirstComponentChild(slot)
    const componentOptions: ?VNodeComponentOptions = vnode && vnode.componentOptions
    if (componentOptions) {
      // check pattern
      const name: ?string = getComponentName(componentOptions)
      const { include, exclude } = this
      if (
        // not included
        (include && (!name || !matches(include, name))) ||
        // excluded
        (exclude && name && matches(exclude, name))
      ) {
        return vnode
      }
      // ...
    }
    return vnode || (slot && slot[0])
  }
}
```

render 函数中，先获取组件的第一个子组件。根据子组件的 name 属性结合 include 和 exclude 参数进行判断是否使用缓存的组件，如果命中，则不使用缓存。

```javascript
render () {
    // ...
    if (componentOptions) {
      // ...
      const { cache, keys } = this
      const key: ?string = vnode.key == null
        ? componentOptions.Ctor.cid + (componentOptions.tag ? `::${componentOptions.tag}` : '')
        : vnode.key
      if (cache[key]) {
        vnode.componentInstance = cache[key].componentInstance
        // make current key freshest
        remove(keys, key)
        keys.push(key)
      } else {
        cache[key] = vnode
        keys.push(key)
        // prune oldest entry
        if (this.max && keys.length > parseInt(this.max)) {
          pruneCacheEntry(cache, keys[0], keys, this._vnode)
        }
      }

      vnode.data.keepAlive = true
    }
    return vnode || (slot && slot[0])
  }
}
```

紧接着就到了组件缓存的部分，直接使用组件 key 或者 cid+tag 的组合 (之所以 cid 是为了防止出现相同tag 组件) 作为缓存 key，采用 LRU 策略进行组件的缓存。如果已缓存，则使用缓存组件的 componentInstance 实例进行渲染，同时设置 data 的 keepAlive 为 true 作为组件是缓存状态的标识。

到此，keep-alive 的核心部分就讲解完了，抛开源码别的部分的处理，keep-alive 组件本身只是一个基础 LRU 缓存策略的高阶组件。最后让我们过一下几个辅助函数。

### pruneCacheEntry

```javascript
function pruneCacheEntry (cache: VNodeCache, key: string, keys: Array<string>, current?: VNode) {
  const cached = cache[key]
  if (cached && (!current || cached.tag !== current.tag)) {
    cached.componentInstance.$destroy()
  }
  cache[key] = null
  remove(keys, key)
}
```

该函数为缓存清理的入口，在组件缓存且当前组件的 tag 不等于缓存组件的 tag 的时候进行组件的 destroy 生命周期调用同时清除组件缓存。

**这里有一个坑，在设置缓存的时候，会存在缓存的多个组件 tag 标签值相同的情况。例如缓存多个 key 值不同的同一个组件，在这种情况下，每个组件实例本质上对应不同的 componentInstance，在这个场景下并不会调用 destroy 方法清除 vue 组件实例的绑定关系等事件监听而造成内存的占用。**

### matches
```javascript
function matches (pattern: string | RegExp | Array<string>, name: string): boolean {
  if (Array.isArray(pattern)) {
    return pattern.indexOf(name) > -1
  } else if (typeof pattern === 'string') {
    return pattern.split(',').indexOf(name) > -1
  } else if (isRegExp(pattern)) {
    return pattern.test(name)
  }
  /* istanbul ignore next */
  return false
}
```
matches 简单的处理了数组类型，字符串类型和正则类型的匹配，从而判断组件是否在 include 和 exclude 的情况下该缓存。

## 总结

至此，我们初略浏览了 keep-alive 的源码，了解了其本质原理为 LRU 组件缓存。keep-alive 虽好，但是推荐慎用。笔者认为该组件适用场景为数量固定，需要跳转回退保存状态的小组件。而对于组件需要保存状态但却切换频繁，组件实例频繁刷新的情况，建议采用其它方案，或许会导致一系列的诸如内存泄漏等问题。但也或许是笔者使用姿势不正确的原因，在下一篇中我们将通过一个案例，说明 keep-alive 在实践中产生的问题。如果有好的解决方案，也希望能告知一二，欢迎大家随时交流讨论。