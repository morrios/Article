# 关于iOS的离屏渲染的分析

### 1. 离屏渲染的简单介绍

* app 在帧缓存区之外开辟一块临时的缓存区，用来进行额外的渲染和合并
* 离屏渲染可能会造成性能上的损耗
* 最大可存储屏幕像素点的2.5倍
* 由系统触发

### 2. 检测离屏渲染

- 可以通过在模拟器上，Debug-> Color Off-Screen Rendered

- 其中出现黄色背景的，则为触发了离屏渲染

  <img src="https://tva1.sinaimg.cn/large/007S8ZIlly1gik8hodracj30kg0j0q66.jpg" style="zoom:33%;" />

### 3. 离屏渲染的触发方式

1. layer 的cornerRadius+maskToBounds。叠加图层才会触发，比如UIImageView同时设置背景颜色+图片
2. shouldRasterize 光栅化
3. 半透明视图混合时
4. 毛玻璃效果
5. 绘制文字的layer

### 4 .离屏渲染的分析

```

常规渲染：App -> FrameBuffer -> Display
离屏渲染：App -> OffScreen Buffer(进行计算合并等操作) -> FrameBuffer -> Display

```

常规渲染：

* 当系统绘制好需要展示的图形后，直接提交到帧缓存区，然后显示到屏幕上
* 每次帧缓存区绘制完成，显示到屏幕上，原来帧缓存区里的内容会立即抛弃掉，进行下一次的渲染

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1gik9vgy9cmj30zk0em0ua.jpg" style="zoom:50%;" />

离屏渲染

* 当需要显示的图片一次绘制完成不了的时候，就需要在每一部分绘制好后，先提交到OffScreen Buffer 去
* 当最后一个内容全部绘制完成后，在合并计算，最后把合成的内容提交到 frameBuffer 中，最后显示到屏幕

（离屏渲染比常规渲染多了一步 ，提交OffScreen Buffer，合成计算）

<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1gik9vxpgwjj30qw0bbjub.jpg" style="zoom:67%;" />

### 5. 为什么需要离屏渲染

比如，如下代码

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gikefehg8bj30e204h0ti.jpg)

背景色 backgroundColor 需要渲染，content (image) 也需要裁剪渲染，按照常规渲染步骤，按照画家算法，先远后近原则：

* 背景色 backgroundColor 渲染后的位图，先进入**帧缓存区 -> 屏幕**，帧缓存区清空
* content (image) 位图进入 **帧缓存区 -> 屏幕**，帧缓存区清空
* 裁剪内容？没有东西可以裁剪了，帧缓存区已经被清空了

所以，这时候需要**额外开辟一块缓冲区**，等待 **合成/裁剪完成后 -> 帧缓冲区 -> 屏幕显示**。那么额外处理复杂数据的地方就是**离屏渲染缓冲区(Offscreen Buffer)**，所以在帧缓存区之前要多一个离屏渲染缓冲区。

下面就是离屏渲染流程图

* 先渲染Mask纹理，保存到离屏缓存区
* 计算layer纹理部分，保存到离屏缓存区
* 在开辟的离屏渲染缓存区里，将Mask和layer内容混合计算渲染，最后在提交到FrameBuffer -> 展示到屏幕上

![](https://tva1.sinaimg.cn/large/007S8ZIlly1gikf0kcxaqj30qk0jwtes.jpg)

### 6. 离屏渲染的触发方式

##### 1. cornerRadius + masksToBounds

```objective-c
 img1.layer.cornerRadius = 50;
 img1.layer.masksToBounds = YES;
```

上面代码就能实现一个圆角，也是作为很多触发离屏渲染的示例代码，但是这样一定会触发离屏渲染吗？

```objective-c
    UIImageView *img1 = [[UIImageView alloc]init];
    img1.frame = CGRectMake(100, 320, 100, 100);
//    img1.backgroundColor = [UIColor blueColor];
    [self.view addSubview:img1];
    img1.layer.cornerRadius = 50;
    img1.layer.masksToBounds = YES;
    img1.image = [UIImage imageNamed:@"22.jpg"];
```

按照上面代码打开注释，就会发现触发了离屏渲染，关闭，则没有触发。

我们已经可以看到 `cornerRadius+masksToBound` 并不是一定会引发离屏渲染。关键是要看有几层渲染数据：backgroundcolor、content(image、text等)。

`cornerRadius+masksToBound` 只有在设置了content且背景不是透明时，才会出现离屏渲染。

如果一定要使用`cornerRadius+masksToBound`的方式裁切图片，不要设置**backgroundColor**。

Tip：cornerRadius并不是对content(image)有效，根据苹果官方解释：cornerRadius 的文档中明确说明对 cornerRadius 的设置只对 CALayer 的** backgroundColor 和 borderWidth、borderColor**起作用。

##### 2. 毛玻璃效果

例如实现一个毛玻璃效果

- 渲染内容
- 捕获内容
- 水平模糊
- 垂直模糊
- 合成过程，然后提交
- 提交 FrameBuffer -> Display

渲染的位图并不能直接给帧缓存区等待显示，而要经过模糊处理之后才能将最后的渲染数据 -> 帧缓冲区-> 显示。

##### 3. `shadow` 阴影

`shadow` 是什么？

`shadow`是一个矩形，是一个背景色，所以要在layer 的下方，`shadow`是根据layer而来，要根据layer 确定自身的位置大小

如果没有离屏渲染，根据画家算法，要先将`shadow`放入帧缓存区，先显示。但是layer没有，不可能先渲染`shadow`。只能利用离屏渲染缓冲区，等待`shadow`、layer等渲染并合并完成后，再送入帧缓存区等待显示。

##### 4. shouldRasterize 光栅化

根据官方文档，如果开启光栅化，会将layer最后的渲染，包括阴影、裁切等的最终效果变成位图放入离屏缓冲区，等待复用。但是，离屏缓冲区的大小不能超过屏幕的2.5倍，否则被释放；其次，layer如果不是静态的，比如imageview的image需要改变，label的text会发生改变等会发生频繁改变的，开启shouldRasterize离屏渲染会影响效率；还有，离屏缓冲区是有时间限制的，超过100ms如果没有被使用，也会被释放。

所以，我们要善用shouldRasterize：

- 如果layer不是静态的，不建议开启
- 如果layer不能不能被复用，也不建议开启，cell被复用，动画中的layer等
- 超过100ms没有被复用，也不建议开启
- 超出离屏渲染控件 屏幕像素的2.5倍，也不建议开启


##### 5. group opacity 组透明度

当有很多sublayer 时候，我们对靠后的大sublayer 设置透明度，会首先在离屏渲染区等待整个layer 全部sublayer 完成后，在根据组透明度，计算新的颜色，进行颜色整合，最后提交帧缓冲区。并不是每渲染一层sublayer 立刻显示。如果opacity为1则不需要调整透明度，正常画家算法显示。

##### 6. 使用了masks

masklayer 作为遮罩，显示在其所在的大layer以及大layer的所有子sublayer之上。masklayer可能也会带有透明度、形状。那么，我们必须在离屏渲染缓冲区内完成Image和Mask的裁切合并处理，才能将最终的Masked Image -> 帧缓冲区显示。

