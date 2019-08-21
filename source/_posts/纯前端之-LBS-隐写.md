---
title: 纯前端之LBS图片隐写
time: 2018-04-01 23:50:20
tags:
  - LBS
  - 前端
categories:
	- 技术文档
---

### 前言

最近看到一个在图片里隐藏信息的东西，觉得挺有趣，就稍微去了解了下。查各种资料得知是一种叫隐写的技术，大多是借助后端完成。
然后为了证明前端是无所不能的，对的。就用纯前端的方式写了个简单的 LBS 图片隐写工具。

好了，废话不多说，先丢上一些基本信息：

github: [图片隐写](https://github.com/nullptru/frontend_learning/tree/master/js/LBS)

### 正文

隐写术是一门关于信息隐藏的技巧与科学，所谓信息隐藏指的是不让除预期的接收者之外的任何人知晓信息的传递事件或者信息的内容。
一般用来传递一些隐秘信息或者起到数字签名的一些目的。本文中采用的 LBS 算法是隐写里的基本算法，鲁棒性较低，但作为一个玩具
玩玩也是不错的。

#### LBS 算法

LBS 全称为 least significant bit(最低有效位)。我们都知道，一个图片由一个个像素点构成，一个像素点色彩一般由 RGB3 种色彩通
道组成(当然也可以加入 A 通道)，值域分布在 0-255 之间，假使我们将其中的 R 通道数值+1，即便变态色彩识别能力的人也几乎无法
区分。而 LBS 算法的原理则是将需要隐写的二进制数据写入这些色彩通道中。以二进制的形式表现通道值，将最低位置为需要隐写的数
据的二进制值。理论上而言，一张 256*256 像素值的图片，采用 RGBA 四通道，最多能隐写的数据大小为 32k 数据(2^8 * 2^8 \* 4 /
8 / 2 ^ 10)。当然对应的，使用的通道数越多，图片被识别率就越高。

所谓置数据为最低有效位，实际上二进制最后一位为 1，则十进制为奇数，为 0，则十进制为偶数，因此可以翻译为。当写入数据为 1，
像素值为奇数，写入为 0，像素值为偶数。

```javascript
// t 为二进制数据
// r 为像素p的r通道数值
// rr 为加密后像素p的r通道值

// 加密
rr = r - (r % 2) + t

// 解密
r = rr % 2
```

是不是超简单！当然，可以在此基础上做很多优化，有兴趣的小伙伴就自己去尝试下吧。

### 隐写数据处理

前端 js 在二进制流的处理上，的确没有后端语言方便，但既然要玩，总还是能玩的对吧。简单的拿文本来举例，字符串函数中，我们可
以通过`charCodeAt`函数获取对应字符的`code`值，然后通过`toString(2)`便可以将其转变为二进制流字符串，对于英文等 ASCII 码值
，正好是一个字节，而汉字，颜文字等多字节的情况，虽然我们可以正常将其转为二进制流，但还原的时候，哪些是单字节，哪些多字节
，顿时一脸懵逼，臣妾做不到啊......

既然直接处理不行，那我们就换个思路，将所有数据都转为单字节流是不是就好了。而浏览器端有个很好用数据编码函数，那就
是`encodeURIComponent`，可以数据编码为`%`+UTF-8 字节流的形式，进而我们可以将其转换为单字节流的二进制流，解密的时候，通
过`decodeURIComponent`还原就好～

> **注**  
> encodeURICompoent 可用来编码任意字符串  
> encodeURI 是设计用来编码 URI 的，因此对于 https:\\中的':\\'这类字符并不进行编码。

```javascript
// 进行位数补全
const padNumber = (num, fill) => {
  var len = ('' + num).length;
  return (Array(
      fill > len ? fill - len + 1 || 0 : 0
  ).join(0) + num);
}

// 编码为utf8字节流
const encodeUtf8 = (str) => {
  const encodeStr = encodeURIComponent(str);
  const bytes = [];
  for (let i = 0; i < encodeStr.length; i++) {
    const c = encodeStr.charAt(i);
    if (c === '%') {
        const hex = encodeStr.charAt(i + 1) + encodeStr.charAt(i + 2);
        const hexVal = parseInt(hex, 16);
        bytes.push(hexVal);
        i += 2;
    } else bytes.push(c.charCodeAt(0));
  }
  return bytes;
};

// 解码
const decodeUtf8 = (bytes) => {
  let encoded = "";
  for (let i = 0; i < bytes.length; i++) {
      encoded += '%' + padNumber(bytes[i].toString(16), 2);
  }
  return decodeURIComponent(encoded);
};
```

### 图片数据处理

在 H5 的 Canvas API 出现后，前端对于图片的处理就变得十分方便了。话不多说，直接上代码。

```javascript
const image = new Image();
image.src = 'xxx.png';
image.onload = () => { // 要在图片onload函数内进行逻辑操作，确保图片加载
    // 创建临时canvas
    const canvasEle = document.createElement('canvas');
    const ctx = canvasEle.getContext('2d');
    canvasEle.width = image.width;
    canvasEle.height = image.height;
    ctx.drawImage(image, 0, 0);
    // 通过getImageData函数获取图片信息
    // 其中imageData.data为一个width * height * 4的数组
    // 分别包含了rgba通道的信息
    const imageData = ctx.getImageData(0, 0, image.width, image.height);
    // 进行LBS加密
    const encodePixels = encodeLBS(imageData.data, text2bin(data));
    // 将数据通过putImageData写回canvas
    const newImage = new ImageData(new Uint8ClampedArray(encodePixels), image.width, image.height);
    ctx.putImageData(newImage, 0, 0);
}
```

由此就可以完成一个基础的 LBS 隐写算法～

### 后记

对于计算量较大的情况下，或许可以通过将逻辑计算放入 webworker 中进行优化，也可以在 LBS 算法中进行优化加入纠错机制什么的，
大家可以试试～

参考文献：

1. [文字隐写——树洞](https://aoaoao.me/1308.html)（PS：貌似是个女装大佬
2. [LBS Steganography](https://github.com/RobinDavid/LSB-Steganography)
