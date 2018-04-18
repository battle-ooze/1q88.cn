---
title: 使用 fabricjs 实现模块连接线效果
date: 2018-04-16 20:19:42
tags: [fabric, canvas, javascript]
categories: Code
---

> 最近在一个项目里需要实现一个连接图的交互，使用了 fabric 这个 canvas 库，它本身封装了画布和一些基础形状，这里主要分享一下其中连接线的实现。

<!-- More -->

先看最终效果：

{% jsfiddle ffk9hkjw result light 100% 400 %}

## 模块节点

> 一个模块节点包括展示模块信息的矩形方框和表示可连接点的圆形组成，所以可以用一个 `fabric.Group` 对象来把这两种形状封装起来。

fabric 提供了 `fabric.util.createClass` 方法，可以用来继承它提供的基础类，在其基础上新增或者修改父类的属性和方法，这里从 `fabric.Group` 继承一个 `fabric.LinkNode` 

```Js
fabric.LinkNode = fabric.util.createClass(fabric.Group, {
  type: 'link-node'
})
```

每个  `LinkNode` 里包括节点名称、连接点两边部分。用 `fabric.Rect` 和  `fabric.Text` 组合来显示模块名称：

```Js
function createRect (text) {
  return new fabric.Group([
    new fabric.Rect({
      width: 150,
      height: 50,
      rx: 4,
      fill: '#fff',
      shadow: { color: '#c2cbd4', blur: 4, offsetY: 1, offsetX: 0 }
    }),
    new fabric.Text(text, {fontSize: 14}),
  ])
}
```

用 `fabric.Circle` 表示连接点，在创建 `LinkNode` 时，需要传入连接点的信息，这里简化为：

```Js
let points = {
  pre: 1, // 前置节点数量
  next: 2 // 后置节点数量
}
```

连接点应当均匀分布在 `LinkNode` 的上下两边，所以需要根据矩形的宽度计算每个点的间隔从而确定点的位置：

```Js
let interval = rect.width / (points.next + 1) // 后置节点间隔
```

由矩形和点就组成了一个 `LinkNode`，改写构造函数，在初始化的时候就把形状添加到 `group` 上：

```Js
LinkNode.prototype.initialize = function (option) {
  this.rect = this.createRect({text: option.name})
  this._points = this.createLinkPoint(option.points, this.rect)
  this.callSuper('initialize', [this.rect, ...this._points], option)
}
```

效果如下：

![WX20180416-170212](https://1q88-1255837662.cos.ap-shanghai.myqcloud.com/WX20180416-170212.png)





## 画布

> 下面要实现的交互是当鼠标按住一个连接点并拖动的时候，会从连接点拖出一条连接线。因为连接点对象属于 `LinkNode` 这个 `Group`，默认的拖动效果是拖动整个 `LinkNode`，我们需要针对拖动这一动作做特殊处理，即当拖动的是连接点时，效果为从点击的连接点拖出一根连接线，而如果拖动的是模块的其他地方，则是移动 `LinkNode`。

### 创建新的画布类

从 `fabric.Canvas` 继承一个新的类 `LinkCanvas`，用于处理连接交互。

```Js
const LinkCanvas = fabric.util.createClass(fabric.Canvas, {
  type: 'link-canvas'
 })
```

### 获取鼠标点击的对象

在画布中触发的事件会告诉我们事件触发的 `target` ，但是对于 `Group` 对象来说，默认的 `target` 只有`Group` 对象而不会具体到是 `Group` 中的哪个子对象。这个可以通过在 `Group` 上设置 `subTargetCheck` 参数解决：

![WX20180416-170738](https://1q88-1255837662.cos.ap-shanghai.myqcloud.com/WX20180416-170738.png)

```Js
LinkNode.prototype.subTargetCheck = true // 设置当事件触发时，check 事件是在哪些子对象上触发
```

这样在事件参数中的 `subTargets` 属性中可以获取到实际点击的对象：

![WX20180416-170936](https://1q88-1255837662.cos.ap-shanghai.myqcloud.com/WX20180416-170936.png)

用一个方法整合表示获取当前事件对象：

```Javascript
function getCurrentTarget (e) {
  return (e.subTargets && e.subTargets[0]) || e.target
}
```

## 连接线

连接线可以视作一个 `fabric.Path`，只需要连接线的开始点和结束点就可以画出这条线，所以可以把线的路径和样式封装起来：

```js
const LinkLine = fabric.util.createClass(fabric.Path, {
  type: 'link-line',
  fill: 'transparent',
  stroke: '#ccc',
  strokeWidth: 1,
  // 不允许拖动
  selectable: false,
  lockMovementX: true,
  lockMovementY: true,
    
  // 对象转换的原点设置到左上角，方便计算
  originX: 'left',
  originY: 'top',
  initialize ([x1, y1, x2, y2], options) {
    this.callSuper('initialize', [['M', 0, 0]], options)
    this.setPath([x1, y1, x2, y2])
  },
  setPath ([x1, y1, x2, y2]) {
    // 根据 起始点计算出路径(this.path)
  }
})
```

### 路径

在 `fabric.Path` 中，使用类似 [SVG Path](https://developer.mozilla.org/zh-CN/docs/Web/SVG/Tutorial/Paths) 的规则来表示形状，来看下我们需求的形状：

![WX20180416-171235](https://1q88-1255837662.cos.ap-shanghai.myqcloud.com/WX20180416-171235.png)

折线可以分成5部分，其中1、5是垂直线（V），3是水平线（H），2、4是弧形（A），设起始点坐标为`x1, y1` ，结束点坐标为 `x2, y2` ，弯曲处半径为 `4px`，我们来看怎么写出这条折线的 `path`：

```Javascript
// 1和5非常靠近的时候，弧线的半径会小于4px，所以这里取最小值
let r = Math.min(4, Math.abs(x2 - x1), Math.abs(y2 - y1)) 

// rx / ry 表示圆弧结束点在对应的轴上的方向
let rx = x1 > x2 ? -1 : 1
let ry = y1 > y2 ? -1 : 1
```

#### 1. 起点

```js
// 移动到起点
['M', x1, y1]
```

#### 2. 垂直线

```js
// V 表示垂直线，参数只有一个，表示 y 轴移动到的位置
['V', y1 + ((y2 - y1) / 2 - ry * r)] 
```

#### 3. 圆弧

A 的参数比较多：

```
A rx ry x-axis-rotation large-arc-flag sweep-flag x y
```

其中，`rx` 和 `ry` 分别是x轴半径和y轴半径，就是圆弧的半径`r`；
`x-axis-rotation` 是 x 轴的旋转角度，这里是一个正圆，怎么旋转都是一样的，设置为0；
`large-arc-flag` 是角度大小，`sweep-flag` 表示弧线的方向，可以根据下图理解一下：

![SVGArcs_Flags](https://1q88-1255837662.cos.ap-shanghai.myqcloud.com/SVGArcs_Flags.png)

两点之间，相同半径的弧线会有以上四种可能，这两个参数就是用于控制圆弧的路径。
`large-arc-flag` 的参数中，0表示小于180度（小弧），1表示大于180度（大弧）；
`sweep-flag` 的参数中，0表示逆时针方向，1表示顺时针方向；
所以对于 3 这个圆弧来说，弧度总是小于180度，`large-arc-flag` 设为0；
而弧线方向取决于起始点和结束点的相对位置，结束点在起始点的右下和左上时为逆时针，右上和左下为顺时针，所以：

```javascript
let sweepFlag = +((y1 - y2) / (x1 - x2) < 0)
```

最后两个参数`x`，`y` 就是结束点的位置，根据以上可以写出这一部分的路径为：

```Javascript
['A', r, r, 0, 0, sweepFlag, x1 + rx * r, (y2 - y1) / 2 + y1]
```

#### 4. 水平线

```js
// H 表示水平线，参数是 x 轴移动到的位置
['H', x2 - rx * r]
```

5、6 两部分与2、3类似，只是方向不同，所可以得出整条线的 Path：

```javascript
let r = Math.min(4, Math.abs(x2 - x1), Math.abs(y2 - y1))
let rx = x1 > x2 ? -1 : 1
let ry = y1 > y2 ? -1 : 1
let sweepFlag = +((y1 - y2) / (x1 - x2) < 0)
this.path = [
  ['M', x1, y1],
  ['V', y1 + ((y2 - y1) / 2 - ry * r)],
  ['A', r, r, 0, 0, sweepFlag, x1 + rx * r, (y2 - y1) / 2 + y1],
  ['H', x2 - rx * r],
  ['A', r, r, 0, 0, +!sweepFlag, x2, (y2 - y1) / 2 + ry * r + y1],
  ['V', y2]
]
```

### 更新对象属性

fabric 的 `Path` 对象在默认情况下，如果实时修改对象中的 `path` 属性，对象的属性并不会根据新的 `path` 更新，比如线的长度变化时，会使整个对象的 `width` `height` `left` `top` 这些属性都发生变化，但是 fabric 不会在 `path` 变化时触发这些基本属性的更新。而我们需要在连接时，根据鼠标位置不停的修改连接线的结束点并实时展示，所以需要在 `setPath` 之后，手动更新这几个属性。在文档里并没有提到如何修改在修改新的 `path` 属性后实时生效，从 fabric 的源码里可以看到，`Path` 对象在初始化的时候会执行一个 `this._setPositionDimensions(options)` 方法，作用是根据 `path` 计算对象的边界和位置，并重新设置。所以我们可以在 `setPath` 后调用这个方法：

```javascript
function setPath () {
  // .....
  this.pathOffset = null // 在_setPositionDimensions里会重新设置这个参数，这里先置空
  this._setPositionDimensions({})
  this.setCoords() // 根据当前的宽高角度重新设置边界点的位置
}
```

### 事件监听

首先监听画布的 `mouse:down` 事件，如果事件目标是连接点，则移除所有当前选中的对象，并且阻止 `LinkNode` 的移动，同时往画布中添加连接线：

```js
function onMouseDown (e) {
  let target = e.subTargets && e.subTargets[0]
  if (!target || target.type !== 'link-point') return
  let group = point.group
  let x = point.group.left + point.left
  let y = point.group.top + point.top
  let line = this.createLinkLine(x, y)

  // 移除所有选中和锁定 group
  this.discardActiveObject()
  group.lockMovementX = true
  group.lockMovementY = true

  this.add(line)
  line.sendToBack()
  this._currentLinkLine = line
  this._currentStartPoint = target
}

canvas.on('mouse:down', onMouseDown)
```

然后监听画布的 `mouse:move` 事件，当画布上有正在连接的连接线存在时，实时更新线的 `path`

```javascript
function _handleMouseMove (e) {
  // 如果有连接线存在，改变连接线的结束点位置
  if (this._currentLinkLine) {
    let x1 = this._currentLinkLine.path[0][1]
    let y1 = this._currentLinkLine.path[0][2]
    this._currentLinkLine.setPath([
      x1, y1, 
      e.e.offsetX - this.viewportTransform[4], e.e.offsetY this.viewportTransform[5]
    ])
    this.requestRenderAll()
  }
}
```

最后监听 `mouse:up` 事件，如果鼠标在另一个可连接的点上松开，则将两点连接（设置线的终点），否则删除连接线：

```javascript
function  _handleMouseUp (e) {
  let target = this.getCurrentTarget(e)
  if (this._currentLinkLine) {
    this.endLink(target)
  }
}
```

### perPixelTargetFind

上面事件里，我们都需要获取当前鼠标指向的对象，对于形状为矩形的对象来说比较简单，而判断鼠标是否指向的是连接线就比较麻烦了。默认情况下，fabric 的事件中获取到的 `target` 依据的是形状所占的矩形区域：

![WX20180416-190103](https://1q88-1255837662.cos.ap-shanghai.myqcloud.com/WX20180416-190103.png)

如果两根线相距比较近的话，通过矩形区域判定就会有问题，比如下图，鼠标在红点的时候，实际上离红线更近，但是如果按照矩形范围判定，灰线的层级在更上面，那么 `target` 就会是灰线。

![WX20180416-190127](https://1q88-1255837662.cos.ap-shanghai.myqcloud.com/WX20180416-190127.png)

而我们期望的判定范围应该是下图浅蓝色部分：

![WX20180416-190149](https://1q88-1255837662.cos.ap-shanghai.myqcloud.com/WX20180416-190149.png)

对于这个问题，fabric 提供了一个 `perPixelTargetFind` 属性，对于 `perPixelTargetFind` 设置为 `true` 的对象，则是基于每个像素在画布上找对象，而不是根据边界框。在画布上提供了 `targetFindTolerance` 属性，用于设置容忍的范围，也就是上图浅蓝色部分区域的大小。
所以对于 `LinkLine` 可以设置：

```js
const LinkLine = fabric.util.createClass(fabric.Path, {
  type: 'link-line',
  // ...
  perPixelTargetFind: true
})
```

在 `fabric.LinkCanvas` 上：

```Javascript
const LinkCanvas = fabric.util.createClass(fabric.Canvas, {
  type: 'link-canvas',
  // ...
  targetFindTolerance: 10
})
```

这样我们就可以精准的选中连接线。

### 删除线

能精准地在事件中获取对应目标后，可以给连接线增加删除功能。
`LinkLine` 中增加方法，显示和删除 `×` 符号：

```js
function showClose () {
  if (!this._close) {
    this._close = new fabric.Text('×', {
      fontSize: 20,
      fill: 'red'
    })
  }
  this.canvas.add(this._close)
  let dims = this._parseDimensions()
  this._close.left = dims.left + dims.width / 2
  this._close.top = dims.top + dims.height / 2
  this.canvas.requestRenderAll()
}

function removeClose () {
  if (this._close) {
    this.canvas.remove(this._close)
  }
}
```

分别绑定到 `mouseover` 和 `mouseout` 事件中，这样在鼠标 hover 到对应连接线的时候就会显示删除提示：

![Apr-16-2018 19-16-58](https://1q88-1255837662.cos.ap-shanghai.myqcloud.com/Apr-16-2018 19-16-58.gif)

之后再监听 `mouseup` 事件，`remove` 线和删除符号即可。

## 总结

这样我们基本实现了需求，总结下来有几点：

1. 采用类的继承，在 fabric 原有的 `Group `，`Path`，`Canvas` 等 `Class` 上进行扩展；
2. 设置 `subTargetCheck` 属性，获取触发在 `Group` 上的事件实际对应的的子对象；
3. 使用 **SVG Path** 画出线条形状；
4. 设置 `perPixelTargetFind` 和 `targetFindTolerance` 属性精准选择对象；
5. 通过事件监听触发对象状态变化和重绘。
