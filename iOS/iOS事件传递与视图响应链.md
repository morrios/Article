# iOS 中事件的响应链和传递链



iOS 事件的主要由：**响应连** 和 **传递链** 构成。一般事件先通过传递链，传递下去。响应链，如果上层不能响应，那么一层一层通过响应链找到能响应的`UIResponse`。

* 响应连：由最基础的`view`向系统传递，`first view` -> `super view` -> ... -> `view controller` -> `window` -> `Application` -> `AppDelegate`
* 传递链：有系统向最上层`view`传递，`Application` -> `window` -> `root view` -> ... -> `first view`

iOS 中只有继承了`UIResponse`的对象才能够接受处理事件。`UIResponse`是响应对象的基类，定义了处理上述各种事件的接口。常见的子类有：`UIView`，`UIViewController`，`UIApplication`和`UIApplicationDelegate`.

我们借用一下官方文档的图，可以看出事件响应链基本上和视图的层级一致。一个`UIResponse` 的 `nextResponse` 指向他的下一个响应者，你可以重写`nextResponse`方法改变下一个响应者。比如一个`view`如果是`UIController`的根视图，它的`nextResponse`会指向`UIController`，否则就会指向它的父视图。

![](https://user-gold-cdn.xitu.io/2019/6/12/16b4b107620f66d7)

### 响应者链

当事件发生了，必须知道有谁来响应。在iOS中，由响应者链来对事件进行响应。

响应者链是由一个不同对象组成的层次结构，其中的每个对象将依次获得响应事件的机会。当发生事件时，事件首现将被发送到第一响应者，第一响应者基本是事件发生的事图，也就是用户触摸屏幕的地方。事件将沿着响应者链一直向下传递，直到被接受并作出处理。

一般来说，第一响应者是个视图对象或者其子类对象，当其被触摸后事件被交由它处理，如果它不处理，事件就会被传递给它的视图控制器对象 ViewController（如果存在），然后是它的父视图（superview）对象（如果存在），以此类推，直到顶层视图。接下来会沿着顶层视图（top view）到窗口（UIWindow 对象）再到程序（UIApplication 对象）。如果整个过程都没有响应这个事件，该事件就被丢弃。

基本上，在响应者链只要有对象处理事件，事件酒停止传递。

```markdown
First Response -> Window -> Application -> nil
```

通常情况下，我们在**First Response** 这里就会响应请求，然后下面的事件分发机制

### 事件分发

**First Response（第一响应者）**，指的是当前接受触摸的响应者对象，是响应者的开端。响应者链和事件分发的使命都是找出第一响应者。

iOS 检测到手指触摸操作（Touch）时，会将其打包成一个 `UIEvent` 对象，并放入当前活动`Application`的事件队列中去。`UIApplication`会依次从队列里取出事件，传递给 `window`对象，`window`会使用`hitTest:withEvent:` 方法寻找出操作初始点所在视图。

事件传递的两个核心方法

```objective-c
- (nullable UIView *)hitTest:(CGPoint)point withEvent:(nullable UIEvent *)event;   
- (BOOL)pointInside:(CGPoint)point withEvent:(nullable UIEvent *)event;
```

其中UIView不接受事件处理的情况有

```markdown
1. hidden ＝ YES,隐藏的视图.
2. userInteractionEnabled = NO,禁止用户操作的视图.
3. alpha <0.01, 透明视图
```

`hitTest:withEvent:`方法的处理流程如下：

* 首先调用当前视图的`pointInside:withEvent:` 方法判断触摸点是否在当前视图内
* 若返回NO，则`hitTest:withEvent:` 返回 nil。
* 若返回YES，则向当前视图的所有子视图发送`hitTest:withEvent:` 消息，所有子视图的遍历顺序是从最顶视图一直到最低层视图，即从 `subviews` 数组的末尾向前遍历，直到有子视图返回非空对象，或者全部子视图遍历完毕。
* 若第一次有子视图返回非空对象，则`hitTest:withEvent:`返回此对象，处理结束。
* 若所有子视图都返回空，则`hitTest:withEvent:`返回自身。

示例性代码如下：

```objective-c
- (UIView *)hitTest:(CGPoint)point withEvent:(UIEvent *)event{
    UIView *touchView = self;
    if ([self pointInside:point withEvent:event] &&
       (!self.hidden) &&
       self.userInteractionEnabled &&
       (self.alpha >= 0.01f)) {
        for (UIView *subView in self.subviews) {
            [subview convertPoint:point fromView:self];
            UIView *subTouchView = [subView hitTest:subPoint withEvent:event];
            if (subTouchView) {
                touchView = subTouchView;
                break;
            }
        }
    } else {
        touchView = nil;
    }
    return touchView;
}
```

### 说明

1. 如果最终`hitTest` 没有找到第一响应者，或第一响应者没有处理该事件，则该事件会沿着响应者链向上回溯。若最终 `UIWindow`  `UIApplication`都不能处理该事件，则会被丢弃。
2. `hitTest:withEvent:`方法会忽略隐藏的视图，禁止用户操作的视图，以及alpha小于0.01的视图。
3. 如果一个子视图的区域超过父视图的 bound 区域(父视图的 clipsToBounds 属性为 NO，这样超过父视图 bound 区域的子视图内容也会显示)，那么正常情况下对子视图在父视图之外区域的触摸操作不会被识别, 因为父视图的 `pointInside:withEvent:` 方法会返回 NO, 这样就不会继续向下遍历子视图了。当然，也可以重写 `pointInside:withEvent:` 方法来处理这种情况。
4. 我们可以重写 `hitTest:withEvent:` 来达到某些特定的目的。