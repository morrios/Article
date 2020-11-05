[toc]

### VC的生命周期 （简单）

initWithNibName:bundle: 初始化UIViewController，执行关键数据初始化操作，非StoryBoard创建UIViewController都会调用这个方法.

initWithCoder: 如果通过StoryBoard进行视图管理，程序不会直接初始化一个UIViewController，StoryBoard会自动初始化或在segue被触发时自动初始化，因此方法initWithNibName:bundle不会被调用，但是initWithCoder会被调用。

loadView :当访问UIViewController的View属性时，View如果此时为nil,那么ViewController会自动调用loadView方法来初始化一个UIView并赋值给UIViewController的View；如果没有重载lodaView方法，则UIViewController会从nib或StoryBoard中查找默认的loadView，默认的loadView会返回一个空白的UIView对象.

viewDidLoad:当loadView将view载入内存中，可以对加载网络数据，视图的UI初始化.

viewWillAppear:系统在载入所有的数据后，在视图展示之前还可以进行进一步的调整(比如设置状态栏方向).

viewWillLayoutSubviews:view 即将布局其Subviews.

viewDidLayoutSubviews:view已经布局其Subviews.

viewDidAppear:视图完全展示在界面之前最后的调整.

viewWillDisappear:在视图切换是，当前视图在即将被移除、或被覆盖是，此时还没有调用removeFromSuperview。

viewDidDisappear:view已经消失或被覆盖，此时已经调用removeFromSuperView;

dealloc:视图被销毁.

didReceiveMemoryWarning:在内存足够的情况下，app的视图通常会一直保存在内存中，但是如果内存不够，一些没有正在显示的viewController就会收到内存不够的警告，然后就会释放自己拥有的视图，以达到释放内存的目的。但是系统只会释放内存，并不会释放对象的所有权，所以通常我们需要在这里将不需要显示在内存中保留的对象释放它的所有权，将其指针置nil。

### 有A、B 两个页面，从A push 到 B ，生命周期调用

```objective-c
SecondViewController - viewDidLoad
ViewController - viewWillDisappear
SecondViewController - viewWillAppear
ViewController - viewDidDisappear
SecondViewController - viewDidAppear
```

### 实现 hitTest 方法

```objc
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event{
    if (self.userInteractionEnabled == NO || self.alpha < 0.1 || self.hidden == true) {
        return nil;
    }
    
    if ([self pointInside:point withEvent:event]) {
        for (NSInteger i = self.subviews.count - 1; i >= 0; i --) {
            UIView *sub = [self.subviews objectAtIndex:i];
            CGPoint subPoint = [sub convertPoint:point fromView:self];
            UIView *res = [sub hitTest:subPoint withEvent:event];
            if (res) {
                return res;
            }
        }
        return self;
    }
    return nil;
}
```

### block 里有super，会造成循环引用吗？

```objective-c
self.block = ^{
      [super class];
   };
```

答案：会的

首先，self -> block -> self，循环引用

正常的方法调用会转化为`id objc_msgSend(id self, SEL _cmd, ...);`

`[super class]`,转化为`objc_msgSendSuper(void /* struct objc_super *super, SEL op, ... */ )`

```objectivec
注意：这里第一个参数 是一个objc_super的结构体 这个结构体的结构为：
struct objc_super {
    __unsafe_unretained _Nonnull id receiver;
    __unsafe_unretained _Nonnull Class super_class;
};
```

这里的`receiver`就是self

### 有一个单例，A、B 两个页面，单例有一个block，在B 页面赋值，返回A页面，发生什么情况，会循环引用吗？

在B页面实现：

```objc
 [TestManager share].Block = ^{
        NSLog(@"%@", [self class]);
    };
```

在A页面调用

```objc
if ([TestManager share].Block) {
        [TestManager share].Block();
    }
```

会循环引用

### 判断平衡二叉树

