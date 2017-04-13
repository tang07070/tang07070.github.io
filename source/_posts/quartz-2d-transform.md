title: Quartz-2D 学习（三）：变换（Transform）
date: 2015-12-22 23:20:59
tags: [Quartz, Core Graphics]
categories: Quartz-2D
---
Quartz 2D绘制模型有两个坐标系空间，用户空间和设备空间：

+ 用户空间：代码中操作的是坐标，使用浮点数表示
+ 设备空间：设备真正的分辨率，使用像素表示

我们在实际使用中只需要关注用户空间，不需要关注设备空间，Quartz 2D会在绘制和显示时将用户空间映射到设备空间，如果想要修改默认用户空间，可以通过修改当前变换矩阵（current transformation matrix, or CTM）来实现。

### CTM
通过修改CTM可以影响绘制的内容，我们可以做的修改包括平移（translate），旋转（rotate）和缩放（scale），可以修改其中之一或者进行组合修改，因为CTM也属于Graphics State，所以可以使用`CGContextSaveGState`保存当前CTM，使用`CGContextRestorState`恢复之前的CTM。

下面分别对三种修改方式举个例子，原图使用之前[Path文章](http://tang07070.com/2015/12/20/quartz-2d-path/)的示例图:

![](/image/quartz-2d/path-fill-even-odd-example.png)

#### 平移
平移包括横向和纵向，也就是对现坐标系进行x，y坐标偏移，默认坐标系下x轴向右为正增长，y轴向上为正增长，使用以下代码在原图绘制三角形之前进行CTM平移操作：

```
CGContextTranslateCTM(ctx, 30, 30);
```

结果如图：

![](/image/quartz-2d/transform-translate.png)

因为例子中使用的是UIView创建的Graphics Context，其CTM已经被修改过，所以对于y轴，向下为正增长

#### 旋转
通过对CTM进行旋转操作，可以使坐标系进行设定角度的旋转，其旋转锚点为坐标系原点，使用一下代码进行旋转负15度操作：

```
CGContextRotateCTM(ctx, radians(-15));
```

其中`radians`函数将角度转为浮点数，定义如下：

```
static inline double radians (double degrees) {return degrees * M_PI/180;}
```

结果如图：

![](/image/quartz-2d/transform-rotate.png)

同样是因为使用的是UIView修改过的CTM，所以其锚点为左上角

#### 缩放
对CTM进行缩放操作后，任何绘制都会被乘上设定的缩放因子，并且x轴和y轴可以分别设定不同的缩放因子，使用如下代码是x轴、y轴都缩小0.5倍：

```
CGContextScaleCTM(ctx, .5, .5);
```

结果如图：

![](/image/quartz-2d/transform-scale.png)

因为整个坐标系都被缩放了，所以不仅三角的大小改变了，其原点也会相应的改变

#### 组合变换
当对CTM进行不只一种变换时，具体变换的先后顺序会影响到最终的图形，下面贴下官方示例，首先使用如下代码：

```
CGContextTranslateCTM (myContext, w/4, 0);
CGContextScaleCTM (myContext, .25,  .5);
CGContextRotateCTM (myContext, radians ( 22.));
```

结果如下：

![](https://developer.apple.com/library/ios/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/Art/tsr_rooster.gif)

换一下代码顺序，使用如下代码：

```
CGContextRotateCTM (myContext, radians ( 22.));
CGContextScaleCTM (myContext, .25,  .5);
CGContextTranslateCTM (myContext, w/4, 0);
```

结果如下：

![](https://developer.apple.com/library/ios/documentation/GraphicsImaging/Conceptual/drawingwithquartz2d/Art/rst_rooster.gif)

### 仿射变换
上面说的CTM变换会直接修改当前Context的坐标映射矩阵，如果只是想对矩阵进行变换，可以使用Quartz提供的仿射变换（CGAffineTransform），UIView的`transform`属性便是一个`CGAffineTransform`对象，仿射变换同样提供三种变换方式：平移、旋转、缩放，也可以将两个仿射变换进行联合。

通过`CGAffineTransformMakeTranslation`方法可以创建一个`CGAffineTransform`对象，而不带Make的版本`CGAffineTransformTranslation`则是在一个已有的`CGAffineTransform`对象上进行平移操作，旋转和缩放同理。

如果想要联合两个仿射变换，可以使用`CGAffineTransformConcat`方法，其会返回联合后的新`CGAffineTransform`对象。如果想将仿射变换应用到当前CTM中，可以使用`CGContextConcatCTM`方法。