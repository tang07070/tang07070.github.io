title: Quartz-2D 学习（五）：渐变（Gradient）
date: 2016-01-07 23:30:10
tags: [Quartz, Core Graphics]
categories: Quartz-2D
---
这里的渐变是指颜色渐变，Quartz 2D提供了两种渐变方式，轴向渐变和径向渐变

### 轴向渐变（Axial Gradient）
轴向渐变也叫线性渐变，是指对于从渐变起点到渐变终点的连线，所有垂直于该连线的线，具有不变的颜色值，可以参考下面的图示：

![](/image/quartz-2d/gradient-axial-1.png)

其中lx和ly便是垂直于轴线的线，其颜色是不变的，Cora Animation中的`CAGradientLayer`实现的就是轴向渐变，或者说是封装了Quartz 2D的轴向渐变方法，提供了面向对象的实现。

为了创建一个轴向渐变效果，需要指定起点、终点以及各个关键点的位置和颜色，比如要实现下图的效果：

![](/image/quartz-2d/gradient-axial-2.png)

其代码如下：

```
void drawAxialGradient(CGContextRef ctx, CGRect rect) {
    CGGradientRef gradient;
    CGColorSpaceRef colorSpace;
    
    CGFloat location[] = {0.0, 1.0};
    CGFloat components[] = {1.0, 0.2, 0.2, 1.0,
                            0.2, 0.5, 0.9, 1.0};
    
    colorSpace = CGColorSpaceCreateWithName(kCGColorSpaceGenericRGB);
    gradient = CGGradientCreateWithColorComponents(colorSpace, components, location, 2);
    CGColorSpaceRelease(colorSpace);
    
    CGContextBeginPath(ctx);
    CGPoint lines[] = {{rect.size.width/2, 0}, {rect.size.width, rect.size.height}, {0, rect.size.height}};
    CGContextAddLines(ctx, lines, 3);
    CGContextClosePath(ctx);
    CGContextClip(ctx);
    
    CGPoint startPt = {rect.size.width/2, 0};
    CGPoint endPt = {rect.size.width/2, rect.size.height};
    CGContextDrawLinearGradient(ctx, gradient, startPt, endPt, 0);
    CGGradientRelease(gradient);
}
```

这里指定的起点是三角形的顶点，终点是三角形的底边中间点，因为起点和终点的连线是垂直的，根据轴向渐变的特性，故三角形每条水平线的颜色都是不变的；指定的关键点有两个，0.0和1.0，因为是按照百分比指定的，所以也就是起点和终点的位置，颜色分别是RGBA的`1.0, 0.2, 0.2, 1.0`和`0.2, 0.5, 0.9, 1.0`

对于绘制过程，首先使用`CGGradientCreateWithColorComponents`方法根据关键点信息创建`CGGradientRef`对象，然后裁剪出三角形Path，最后使用`CGContextDrawLinearGradient`方法绘制轴向渐变

### 径向渐变（Radial Gradient）
径向渐变也是两个端点连线的轴向渐变，不过两个端点不再只是一个像素点，而是一个圆，例如下图渐变效果：

![](/image/quartz-2d/gradient-radial-1.png)

其代码如下：

```
void drawRadialGradient(CGContextRef ctx, CGRect rect) {
    CGGradientRef gradient;
    CGColorSpaceRef colorSpace;
    
    CGFloat location[] = {0.0, 1.0};
    CGFloat components[] = {1.0, 0.2, 0.2, 1.0,
        0.2, 0.5, 0.9, 1.0};
    
    colorSpace = CGColorSpaceCreateWithName(kCGColorSpaceGenericRGB);
    gradient = CGGradientCreateWithColorComponents(colorSpace, components, location, 2);
    CGColorSpaceRelease(colorSpace);
    
    CGPoint startPt = {30, 30};
    CGPoint endPt = {rect.size.width/2, rect.size.height/2};
    CGContextDrawRadialGradient(ctx, gradient, startPt, 20, endPt, 50, 0);
    CGGradientRelease(gradient);
}
```

和轴向渐变相比，`CGGradientRef`对象的创建是完全相同的，只是使用`CGContextDrawRadialGradient`方法并指定端点半径

径向渐变比较常见的使用场景可能就是同心圆渐变，如下图效果：

![](/image/quartz-2d/gradient-radial-2.png)

实现起来也很简单，起点和终点坐标相同，并且起点半径设置为0即可