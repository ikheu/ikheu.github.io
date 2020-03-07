---
layout: post
title:  一个基于 canvas 的画板
categories: 前端
tags: Python
---
* content
{:toc}


虽然我自己的定位是做后端的，但技术主管强调要搞全栈，因此工作中也要写一些前端页面。一个产品经理找我写个画板，他更不会管我的技术定位了。目标是做个画板，具有以下功能：

- 可以更改画笔颜色、粗细，能擦除绘图
- 撤销绘图
- 设置画板背景色
- 将画板保存成图片

demo见：[drawing_board](https://github.com/ikheu/frontend/blob/master/drawing_board/index.html)
。最后的效果长这样：

![](/assets/img/demo.png){:width="300px"}

所有的实现以及遇到的问题，都可以在网上找到。这里做一个总结。

## 画笔粗细、颜色

粗细通过 `ctx.lineWidth` 设置；颜色要设置 `ctx.strokeStyle` 和 `ctx.fillStyle` 这两个参数才行。

## 橡皮功能

`ctx.globalCompositeOperation` 的默认值是 `"source-over"`，这时可以正常绘制图形。将其改成 `"destination-out"`，这时画笔可将绘制的线条擦除。橡皮其他部分与画笔的逻辑一样，因此代码可以复用。更改 `lineWidth` 便可调节橡皮大小。有的实现中直接将画笔的颜色改成画板的背景色，以此达到“擦除”的效果。但画板默认是透明的，这种错误实现会导致在输出的图片有问题。

## 撤回功能

想要撤回就要记住历史状态。将每次画下的图形数据粗暴地存储在一个数组中，撤销时将数组最后一个元素设置为 canvas 的图形数据即可。注意这里每次记录的都是完整的数据，而不是像 git 一样记录每次的修改，因此不能无限制地记录绘画历史。设置了数组的最大值 `MaxSaveLimit`，保存的次数超出这个值后，最早的数据将找不到。上文中提到，擦除与普通绘制无本质区别，因此“擦除”的历史也自然是可以撤回的。

```javascript
// 保存
function saveImageData(data) {
    (lastImageData.length == MaxSaveLimit) && (lastImageData.shift());
    lastImageData.push(data);
}

function cancel() {
    if (lastImageData.length < 1)
        return false;
    ctx.putImageData(lastImageData[lastImageData.length - 1], 0, 0);
    lastImageData.pop();
}
```

## 设置背景色

直接设置 canvas 节点的背景色不起作用，canvas 的背景实际上仍然是透明色。将 `ctx.globalCompositeOperation` 设置为 `destination-over`，通过填充实现背景色功能：

```javascript
ctx.globalCompositeOperation = "destination-over";
ctx.fillStyle = backgroundColor;
ctx.fillRect(0, 0, canvas.width, canvas.height);
```

## 保存功能

使用 `canvas.toDataURL` 获得用于图片展示的 URL，然后创建 a 标签，实现图片下载功能：

```javascript
function downLoad() {
    let url = canvas.toDataURL("image/png");
    let oA = document.createElement("a");
    oA.download = 'canvas';
    oA.href = url;
    document.body.appendChild(oA);
    oA.click();
    oA.remove();
}
```

## 其他问题

### canvas 尺寸

和背景色一样，直接更改节点的样式无法设置 canvas 的尺寸，要通过 `canvas.width`、`canvas.height` 来设置 canvas 的长和高。

### 清晰度

可能出现绘图模糊的现象，可根据实际设备的 `devicePixelRatio` 值对 `ctx` 进行缩放： 

```javascript
let ratio = window.devicePixelRatio;
let width = canvas.width, height = canvas.height;
canvas.style.width = width + "px";
canvas.style.height = height + "px";
canvas.height = height * window.devicePixelRatio;
canvas.width = width * window.devicePixelRatio;
ctx.scale(window.devicePixelRatio, window.devicePixelRatio);
```

