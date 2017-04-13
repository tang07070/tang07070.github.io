title: Quartz-2D 学习（二）：路径（Path）
date: 2015-12-20 15:30:05
tags: [Quartz, Core Graphics]
categories: Quartz-2D
---
Quartz 2D 中的path包含一系列的形状和subpath，通过path可以绘制出各式各样的图形，对其进行描边（stroke）、填充（fill）或是裁剪（clip），也可以组合很多subpath来生成复杂的图形，总之如果想绘制自定义图形，那一定需要用到path。

下图是一些path示例：

![](/image/quartz-2d/path-example.png)

### 基本元素
再复杂的图形，进行拆解后其基本组成元素也不过几种，点、直线、曲线、弧线等，在Quartz 2D中也是如此。

#### 点（Point）
Point就是一个坐标，使用`CGPoint`表示，在Quart 2D 中维护了一个current point，其作用是指定一些线性（line、arc等）操作的起始点。想要移动current point，可以使用`CGContextMoveToPoint`方法。

#### 线（Line）
这里的Line指的是从起点要终点的一条直线，这里的起点就是上面说的current point，所以只需要指定终点既可，添加line的方法有两种：

+ `CGContextAddLineToPoint`：添加一条到指定终点的line
+ `CGContextAddLines`：其参数是一个`CGPoint`数组，第一个点作为起点，其他点作为终点依次添加line

#### 弧线（Arc）
Arc是指圆的一个片段，Quartz 提供了两种方式确定Arc：

+ 圆心和半径确定一个圆，Arc需要在圆的基础上指定起始角度和结束角度，使用`CGContextAddArc`方法，其角度如下：

![](/image/quartz-2d/path-arc1.png)

+ 三点连线可以确定一个角，Arc需要在根据半径在这个角中确定具体位置，使用`CGContextAddArcToPoint`方法，如下图，其中P1是current point，红色标识的是确定的arc：

![](/image/quartz-2d/path-arc2.png)

#### 曲线（Curve）
Curve指的是贝赛尔曲线，Quartz中支持二次和三次贝塞尔曲线：

+ 二次贝塞尔曲线：`CGContextAddCurveToPoint`

![](/image/quartz-2d/path-curve-2.png)

+ 三次贝塞尔曲线：`CGContextAddQuadCurveToPoint`

![](/image/quartz-2d/path-curve-3.png)

#### 矩形（Rectangle）
这里就是一般的矩形，添加方式和point一样有两种

+ `CGContextAddRect`添加一个矩形
+ `CGContextAddRects`添加一组矩形

#### 椭圆（Ellipse）
在这里的椭圆不需要指定焦点、中心点等，只要给定一个矩形，Quartz会以矩形中心点为圆心，使用贝赛尔曲线绘制出一个椭圆，该方法为`CGContextAddEllipseInRect`。

### 使用Path
了解了Path的几种基本元素之后，下面我们看看如何创建、绘制以及裁剪Path。

#### 创建Path
使用`CGContextBeiginPath`方法可以开启一个新的Path过程，特别注意的是这时current point还是空的，调用任何绘制方法都是无效的，所以在开始新的path后，需要立即使用`CGContextMoveToPoint`方法指定current point，随后便可以开始任意的添加元素了。

当绘制结束后可以使用`CGContextClosePath`方法关闭当前path，这是会自动添加一条从current point到path起点的line，形成闭合的回路，随后会开启新的subpath，这时如果不手动指定current point的话，subpath会使用上一个subpath的终点作为其current point。

当调用绘制函数后，Quartz会根据当前Path完成相应的绘制，并且在绘制完成后使当前Path**无效**，如果需要新增Path就需要再次调用`CGContextBeiginPath`方法开启新的Path。

可能会有这样的场景：需要把Path当成一个变量保存起来已备后续使用，这时可以使用`CGPathRef`或`CGMutablePathRef`对象，步骤如下：

1. 使用`CGPathCreateMutable`创建`CGMutablePathRef`对象
2. 使用`CGPathMoveToPoint`添加current point
3. 使用`CGPathXXX`系列方法添加元素，如`CGPathAddArcToPoint`
4. 使用`CGPathCloseSubpath`关闭path（可选）
5. 使用`CGContextAddPath`将`CGMutablePathRef`对象添加到context中
6. 使用`CGContextDrawPath`绘制path

同样在绘制之后添加的Path会失效，但是创建的`CGMutablePathRef`对象可以继续使用。

#### 绘制Path
当创建Path并添加完元素后就可以进行绘制了，绘制Path有两种方式，描边（stroke）和填充（fill），可以选择其中之一或者都选择，如下图所示，其中红色部分是描边，蓝色部分是填充。

![](/image/quartz-2d/path-draw-1.png)

##### 描边（Stroke）
Quartz提供了很多方法可以进行stroke操作，如`CGContextStrokePath` `CGContextDrawPath`等，下面主要介绍一下stroke相关属性。

+ 线宽（Width）：设置线宽的方法为`CGContextSetLineWidth`，需要注意的是线宽会从中心点向两边增长，举个例子：如果中心点为0，当设置线宽为5时，那么线所占的宽度为[-2, 2]。
+ 连接点（Join）：设置连接点的方法为`CGContextSetLineJoin`，Quartz提供了三种方式
  + Miter（default）：![](/image/quartz-2d/path-join-miter.png)
  + Round： ![](/image/quartz-2d/path-join-round.png)
  + Square： ![](/image/quartz-2d/path-join-square.png)
+ 端点（Cap）：设置端点的方法为`CGContextSetLineCap`，Quartz也提供了三种方式
  + Butt（default）：![](/image/quartz-2d/path-cap-butt.png)
  + Round：![](/image/quartz-2d/path-cap-round.png)
  + Square：![](/image/quartz-2d/path-cap-square.png)
+ 虚线模式（Dash Pattern）：设置虚线模式的方法为`CGContextSetLineDash`，其中length数组是指[线-空]的长度，比如"--- --  "，这种虚线的length数组为{3，1，2，2}
+ 颜色（Color）：设置stroke颜色的方法为`CGContextSetStrokeColor`和`CGContextSetStrokeColorWithColor`
+ 模式（Pattern）：设置stroke模式的方法为`CGContextSetStrokePattern`，Pattern相关的内容会在后面写
  
##### 填充（Fill）
Quartz提供的Fill Path方法为`CGContextFillPath`，`CGContextEOFillPath`等，那带"EO"的和不带"EO"的版本有什么区别呢？当有多个SubPath有重叠部分的时候，Quartz有两种填充规则（fill rule），在此之前需要知道的是Path是有方向的，跟元素的添加顺序有关，默认为顺时针方向

+ nonzero winding number rule：内外同向时填充外部和内部，内外反向时只填充外部
+ even-odd rule：不管内外同向或内外反向，都只填充外部

示意图如下：

![](/image/quartz-2d/path-fill-rule.png)

举个例子，使用如下代码创建Path：

```
    CGContextBeginPath(ctx);
    CGContextAddRect(ctx, rect);
    CGContextClosePath(ctx);
    CGPoint lines[] = {{rect.size.width/2, 40}, {rect.size.width - 40, rect.size.height - 40}, {40, rect.size.height - 40}};
    CGContextAddLines(ctx, lines, 3);
    CGContextClosePath(ctx);
```
这里的Rect是默认方向的，也就是顺时针方向，三角形按照添加顺序，也是顺时针方向，下面我们使用两种规则分别绘制：

+ 使用`CGContextDrawPath(ctx, kCGPathFillStroke)`绘制：
![](/image/quartz-2d/path-fill-winding-number-example.png)
+ 使用`CGContextDrawPath(ctx, kCGPathEOFillStroke)`绘制：
![](/image/quartz-2d/path-fill-even-odd-example.png)

##### 混合模式（Blend Mode）
混合模式是指Quartz如何处理前景和背景叠加的问题，对此有如下公式：

> result = (alpha * foreground) + (1 - alpha) * background

更多的混合模式举例可参见[http://openlayers.org/en/v3.9.0/examples/blend-modes.html](http://openlayers.org/en/v3.9.0/examples/blend-modes.html)

#### 裁剪Path
创建Path并添加元素之后不一定非要绘制，也可以将画布裁剪为Path的形状，比较常见的用途是作为图片的mask。裁剪后的画布状态是包含在当前context state中的，也就是可以Save和Restore的，另外裁剪Path和绘制Path有很多相同之处：

+ 裁剪规则和绘制规则相同，也是nonzero winding number rule和even-odd rule，分别对应方法`CGContextClip`和`CGContextEOClip`
+ 裁剪后context中的Path也会**失效**
