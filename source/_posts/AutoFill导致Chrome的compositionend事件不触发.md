---
title: AutoFill导致Chrome的compositionend事件不触发
date: 2019-09-25 21:17:52
tags: 前端
categories:
	- 技术文档
---

### 问题

Chrome 75.0.3770.100 版本，在 [IME(input method editor) 模式 ]([https://zh.wikipedia.org/wiki/%E8%BE%93%E5%85%A5%E6%B3%95](https://zh.wikipedia.org/wiki/输入法))下，输入内容触发 autofill 面板展示，选择任意一项，然后继续输入完毕。此时 compositionEnd 事件不触发。

在 Vue 组件库 ElementUI 的 2.12.0 版本前，在 IME 模式下通过检测 compositionend 事件设置输入完毕，从而触发正式 input 事件。因为此问题将导致 input 事件无法正确触发bug。

重现链接：[传送门](https://codepen.io/vellengs/pen/OJLWggN)

```javascript
// element-ui 2.12.0
<template>
  <div>
    {{username}}
    <el-input
      name="username"
      v-model="username"
      autocomplete="on"
      placeholder="请输入内容"></el-input>
  </div>
</template>

<script>
export default {
  data() {
    return {
      username: ''
    }
  }
}
</script>
```

### IME 事件

对于输入框事件，除了基础的 `input`, `change` 事件外，有着专门针对 IME 的 `composition` 事件

- compositionstart：当一个组合文本输入系统开始文本输入，在 keydown 事件之后触发
- compositionupdate：当一个组合文本输入系统接收一个新字符时触发
- compositionend：当文本输入完成或者因为元素焦点转移等各种因素被取消时触发

IME系统中，[w3c 的规范](https://w3c.github.io/uievents/#events-composition-types)下，composition 过程中的事件触发顺序如下：

- compositionstart
- beforeinput
- compositionupdate
- input
- ...
- compositionend

### -webkit-autofill

-webkit-autofill 为 webkit 内核的浏览器所具有的伪类，通过该伪类我们可以对自填充的状态进行定制展现。

当输入框 autocomplete 属性为 on 时，浏览器将根据历史输入提供对应的自填充面板。同时，对应 input 框将被自动添加上 -webkit-autofill 伪类。

如下是一个[伪类的应用实例](https://codepen.io/team/css-tricks/pen/oxyJxR)：

```css
/* Change Autocomplete styles in Chrome*/
input:-webkit-autofill,
input:-webkit-autofill:hover, 
input:-webkit-autofill:focus,
textarea:-webkit-autofill,
textarea:-webkit-autofill:hover,
textarea:-webkit-autofill:focus,
select:-webkit-autofill,
select:-webkit-autofill:hover,
select:-webkit-autofill:focus {
  border: 1px solid green;
  -webkit-text-fill-color: green;
  -webkit-box-shadow: 0 0 0px 1000px #000 inset;
  transition: background-color 5000s ease-in-out 0s;
}
```



### 解决方案

从 w3c 规范中，我们可以确认是 Chrome 本身的 bug 导致 compositionend 事件不触发。因此在 IME 系统下触发 autofill 时候，我们无法通过原生事件确定输入的结束。

这里我们采用了 [Klarna Ui](https://github.com/klarna/ui/blob/v4.10.0/Field/styles.scss#L228-L241) 库中的一个 workaround 处理该问题。

该方法思路是，既然我们无法通过 compositionend 事件监测，不妨找该 case 的其他触发因素。可以确定，该 bug 的触发，一定伴随着自动填充的功能。鉴于此，我们考虑从 autofill 上做文章。

从上述几节中我们了解到目前并没有 autocomplete 的事件监听器，但在 webkit 内核浏览器中我们能通过 -webkit-autofill 伪类的状态识别，因此我们不妨在 css 中埋下对应状态进行事件监测。

```css
input:-webkit-autofill {
  animation-name: onAutoFillStart;
  transition: background-color 0.3s ease-in-out 0s;
}

input:not(:-webkit-autofill) {
  animation-name: onAutoFillCancel;
}
@keyframes onAutoFillStart {
  from {/**/}
  to {/**/}
}
@keyframes onAutoFillCancel {
  from {/**/}
  to {/**/}
}
```

通过 `onAutoFillStart` 和 `onAutoFillCancel` 两个动画，我们向 JS 暴露了 autofill 面板出现和消失的时机。在 JS 中通过监听 `animationstart` 事件来判断 autofill 状态。

```javascript
// add animation listener
this.getInput().addEventListener('animationstart', (e) => {
  switch (e.animationName) {
    case 'onAutoFillStart':
      this.isComposing = false;
      break;
    case 'onAutoFillCancel':
      /* do something */
      break;
  }
})
```

之后配合 composition 事件来判断输入的完成。

```javascript
handleCompositionStart() {
  this.isComposing = true;
}
handleCompositionUpdate(event) {
  const text = event.target.value;
  this.isComposing = true;
}
handleCompositionEnd(event) {
  if (this.isComposing) {
    this.isComposing = false;
    this.handleInput(event);
  }
}
handleInput(event) {
  if (this.isComposing) return;
  /* do something */
}
```

### 参考文献

1. [-webkit-autofill css tricks](https://css-tricks.com/snippets/css/change-autocomplete-styles-webkit-browsers/)
2. [compositionend event MDN](https://developer.mozilla.org/en-US/docs/Web/API/Element/compositionend_event)
3. [W3C Composition Event Types](https://w3c.github.io/uievents/#events-composition-types) 
4. [Vue 相关 bug](https://github.com/vuejs/vue/issues/7058)
5. [klarna ui  autofill workaround](https://github.com/klarna/ui/blob/v4.10.0/Field/styles.scss#L228-L241)