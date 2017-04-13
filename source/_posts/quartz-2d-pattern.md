title: Quartz-2D 学习（四）：模式（Pattern）
date: 2015-12-27 21:17:20
tags: [Quartz, Core Graphics]
categories: Quartz-2D
---
模式是指绘制到Graphics Context上的重复绘制操作，可以把模式当做一种特殊的"颜色"，就像使用颜色一样使用模式，如Stroke和Fill。既然是重复性操作，那么肯定会有最小的模式单元，通过对其有规则的排列来组成整个模式序列，下面就来看下模式单元。

### 模式单元（Pattern Cell）
模式单元是一个矩形，如下图：

![](/image/quartz-2d/pattern-cell-single.png)

其中只有一个蓝色的三角形，黑色的边框如果不明确绘制的话是不会被自动添加的，Quartz为我们提供了一个回调函数来绘制模式单元，其定义如下：

```
typedef void (*CGPatternDrawPatternCallback)(void * __nullable info,
                                             CGContextRef __nullable context);
```
第一个参数`info`是用来进行信息传递的，第二个参数`context`便是当前绘制的上下文，其中的绘制操作和平时绘制无异（模板模式时有点区别，后面会说到），比如对上图的绘制代码如下：

```
void patternCellCallback(void *info, CGContextRef ctx) {
    CGSize sz = {H_PSIZE, V_PSIZE};
    CGPoint lines[] = {{sz.width/2, sz.height/2}, {0, sz.height}, {sz.width, sz.height}};
    CGContextBeginPath(ctx);
    CGContextAddLines(ctx, lines, 3);
    CGContextSetFillColorWithColor(ctx, [UIColor blueColor].CGColor);
    CGContextClosePath(ctx);
    CGContextFillPath(ctx);
}
```

除了对模式单元的绘制外，Quartz还允许我们进行自定义的便是模式单元之前的横纵间距，下图是多个模式单元的组合：

![](/image/quartz-2d/pattern-cell-multi.png)

当然，w和h都可以是0的，但是一旦指定了大小就不能再改变了。

### 两种模式
Quartz为我们提供了两种类型的模式可供使用：着色模式（Colored Pattern）和模板模式（Stencil Pattern）。他们有各自的适用场景，但并不是只能二者取其一，任何一种都是可以满足所有需求的，只是一个会比另一个方便一些。

模式使用`CGPatternRef`封装，其创建方法为:

```
CG_EXTERN CGPatternRef __nullable CGPatternCreate(void * __nullable info,
    CGRect bounds, CGAffineTransform matrix, CGFloat xStep, CGFloat yStep,
    CGPatternTiling tiling, bool isColored,
    const CGPatternCallbacks * __nullable callbacks)
    CG_AVAILABLE_STARTING(__MAC_10_0, __IPHONE_2_0);
```

其参数如下：

+ info：传递给模式单元回掉函数的自定义数据
+ bounds：模式单元大小
+ matrix：模式单元使用的仿射变换矩阵，如果不需要变化可以使用`CGAffineTransformIdentity`
+ xStep：x轴步长，包括单元长+间隔
+ yStep：y轴步长，同上
+ tiling：平铺模式，有如下几种，其失真程度递增：
	+ kCGPatternTilingNoDistortion：
	+ kCGPatternTilingConstantSpacingMinimalDistortion：
	+ kCGPatternTilingConstantSpacing： 
+ isClored：是否是着色模式
+ callbacks：模式单元绘制回调

#### 着色模式
前面的例子便是着色模式，其重点是在模式单元的回调函数中绘制带有颜色的图形，也就是把颜色相关的逻辑放到回调函数中，在外部使用时便可不关注模式单元的颜色了，下面是创建并使用着色模式的代码：

```
void drawColoredPattern(CGContextRef ctx, CGRect rect) {
    CGPatternRef pattern;
    CGColorSpaceRef patternSpace;
    CGFloat alpha = 1;
    static const CGPatternCallbacks callbacks = {0, &coloredPatternCb, NULL};
    
    CGContextSaveGState(ctx);
    patternSpace = CGColorSpaceCreatePattern(NULL);
    CGContextSetFillColorSpace(ctx, patternSpace);
//    CGContextSetStrokeColorSpace(ctx, patternSpace);
    CGColorSpaceRelease(patternSpace);
    
    pattern = CGPatternCreate(NULL,
                              CGRectMake(0, 0, H_PSIZE, V_PSIZE),
                              CGAffineTransformIdentity,
                              H_PSIZE+3, V_PSIZE+3,
                              kCGPatternTilingNoDistortion,
                              true,
                              &callbacks);
    
    CGContextSetFillPattern(ctx, pattern, &alpha);
//    CGContextSetStrokePattern(ctx, pattern, &alpha);
    CGPatternRelease(pattern);
    CGContextFillRect(ctx, rect);
//    CGContextStrokeRect(ctx, rect);
    CGContextRestoreGState(ctx);
}
```

以上是使用模式来进行Fill操作，如若使用模式来进行Stroke操作，只需把Fill相关方法替换为Stroke相关方法（上述代码中注释部分）。

#### 模板模式
所谓模板模式，就是提供只提供Path信息而无关颜色，当然只是在模板单元回调中忽略颜色而已，还是使用上面的三角形示例，模板模式的模式单元回调代码为：

```
void stencilPatternCb(void *info, CGContextRef ctx) {
    CGSize sz = {H_PSIZE, V_PSIZE};
    CGPoint lines[] = {{sz.width/2, sz.height/2}, {0, sz.height}, {sz.width, sz.height}};
    CGContextBeginPath(ctx);
    CGContextAddLines(ctx, lines, 3);
    CGContextClosePath(ctx);
    CGContextFillPath(ctx);
}
```

可以注意到与着色模式的模式单元回调相比，仅仅少了设置Fill颜色的操作，但是颜色总归是要被设置的，实际上颜色信息已经包含在当前`ctx`中，创建并使用模式的代码如下：

```
void drawStencilPattern(CGContextRef ctx, CGRect rect) {
    CGPatternRef pattern;
    CGColorSpaceRef baseSpace;
    CGColorSpaceRef patternSpace;
    CGFloat color[] = {0, 0, 1, 1};
    static const CGPatternCallbacks callbacks = {0, &stencilPatternCb, NULL};
    
    CGContextSaveGState(ctx);
    baseSpace = CGColorSpaceCreateDeviceRGB();
    patternSpace = CGColorSpaceCreatePattern(baseSpace);
    CGContextSetFillColorSpace(ctx, patternSpace);
    CGColorSpaceRelease(baseSpace);
    CGColorSpaceRelease(patternSpace);
    
    pattern = CGPatternCreate(NULL,
                              CGRectMake(0, 0, H_PSIZE, V_PSIZE),
                              CGAffineTransformIdentity,
                              H_PSIZE+3, V_PSIZE+3,
                              kCGPatternTilingNoDistortion,
                              false,
                              &callbacks);
    
    CGContextSetFillPattern(ctx, pattern, color);
    CGPatternRelease(pattern);
    CGContextFillRect(ctx, rect);
    CGContextRestoreGState(ctx);
}
```
可以看到相比着色模式，模板模式的颜色信息是在创建模式时指定的，如此可以方便不同颜色的模式复用，当然将颜色信息使用`info`结构来传递也是可以实现的，这也是为什么之前说两种模式都可以实现需求的原因。