title: Quartz-2D 学习（一）：总览
date: 2015-12-17 21:54:01
tags: [Quartz, Core Graphics]
categories: Quartz-2D
---

Quartz 2D 是Apple提供给开发者的2D绘图引擎，也是Core Graphics Framework 的一部分，Quartz 2D API是以C函数（以CG开头）的方式给出的，通过Quartz 2D API可以轻松绘制出复杂的2D矢量图形。UIKit应该是iOS开发者很熟悉的UI Framework，其内部对Core Graphics 做了封装，让开发者可以更快速、便捷的实现UI视图，但是凡事有利必有弊，上层封装除了带来便利的同时，也抹杀了很多自由度，不过UIKit还是留了一条后路，开发者可以通过覆写UIView的`drawRect:`方法来实现自定义视图，当然这只是Quartz 2D 用途的一部分，为了能对Quartz 2D有个整体的认识，首先看一下Quartz 2D 都包括哪些方面。

### 画布（Page）
同一般的绘图引擎一样，Quartz 2D 的绘图操作都是在一块「画布」上，Apple对其的定义是Page，我们将内容绘制到Page上的操作是不可逆的，没有方法可以抹掉之前的绘制内容，但是元素绘制是有先后顺序的，也就是Z轴的关系，后绘制的内容会覆盖先绘制的内容，用这种方法，可以覆盖我们想抹掉的内容，让它看起来被"抹掉"了。

### 上下文（Graphics Context）
Graphics Context 是Quartz 2D 中很重要的一个概念，通过Context建立了从绘制到具体设备的桥梁，让我们可以对不同的设备使用相同的绘图操作，实现了设备无关性，这里的设备指代：PDF，Bitmap，Window等。Quartz 2D 使用`CGContextRef`来封装Graphics Context，基本上每个绘制函数都是以`CGContextRef`作为第一个参数的。

我们可以使用的Graphics Context有以下几种：

+ Bitmap Context：绘制到Bitmap上
+ Window Context：绘制到window上
+ PDF Context：创建PDF文件
+ Layer Context：绘制到Layer（CGLayerRef）上，主要为了离屏渲染和重复性绘制

下图会有个比较形象的感受：
![](/image/quartz-2d/overview-context-diagram.png)

### 数据类型（Data Types）
除了Graphics Context 之外，Quartz 2D 提供了很多的数据类型可供操作，其都是以CG开头，如下：

+ Path相关：CGPathRef
+ Bitmap相关：CGImageRef
+ Layer相关：CGLayerRef
+ Pattern相关：CGPatternRef
+ Gradient相关：CGShadingRef，CGGradientRef，CGFunctionRef
+ Color相关：CGColorRef，CGColorSpaceRef
+ Text相关：CGFontRef

### 状态（Graphics State）
我们可以认为graphics state是当前绘制上下文的"配置文件"，很多属性设置都保存在其中，比如设置的line width，fill color等，当我们调用draw函数时，Quartz 2D 会根据graphics state中的配置完成具体的绘制操作。

Quartz 2D 维护了一个graphics state栈，当我们开始一个新的绘制操作时，可以使用`CGContextSaveGState`来把当前的graphics state压入栈中，待完成绘制后，使用`CGContextRestoreGState`从栈中还原当前的graphics state，从而不影响其他后续的绘制操作。

graphics state中包含以下内容：

+ 坐标系转换（CTM）
+ 裁剪区域
+ Line属性，如width，join，cap等
+ 反锯齿设置
+ fill和stroke color
+ alpha
+ color space
+ font
+ blend mode

### 坐标系（Coordinate Systems）
Quartz 2D 的坐标系和UIKit的坐标系是上下镜像的，UIKit坐标系的原点在左上角，而Quartz 2D 坐标系的原点在左下角，如下图：

![](/image/quartz-2d/overview-coordinate.png)

Quartz 2D 提供了一系列CTM函数可供操作坐标系，如平移、旋转、缩放，UIKit在创建Graphics Context后，会将坐标系旋转到和UIKit坐标系一致，所以在`drawRect:`中的Context坐标系实际已经被旋转了，类似的还有`UIGraphicsBeginImageContextWithOptions`返回的Context。

### 内存管理（Memory Management）
Quartz 2D 使用的是Core Foundation内存管理模型，也就是引用计数机制，但是不在ARC控制之下，所以需要自己retain和release，方式上有两种选择，使用特定对象的retain和release方法，如`CGColorSpaceRetain`、`CGColorSpaceRelease`方法，也可以使用Core Foundation的`CFRetain`、`CFRelease`方法。

以下两点需要谨记：

+ create或copy对象一定要在使用后release，一般情况下是获取函数中带有"Create"和"Copy"关键字
+ 非create或copy对象任何情况下都不要调用release


