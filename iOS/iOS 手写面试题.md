# iOS

[toc]

### 1. iOS类（class）和结构体（struct）有什么区别

类是引用类型，结构体是值类型。（一个类可以被多次引用，结构体是值传递）

类可以继承，结构体没有继承特性

class 需要自己构建constructor

内存中，引用类型，存储在堆（heap）上，值类型，存储在栈（stack）上。相比于栈，堆上的操作更加复杂，现在在；苹果swift 中，更推荐使用结构体。

struct 结构较小，比较适合复制操作，相对比类实例被多次引用更加安全。

### 2. iOS 什么事是KVO 和 KVC，他们的使用场景是什么

#### KVC

`KVC` (Key-value-coding)，健值编码，iOS 中开发者可以通过 key 直接访问对象的属性，或者给对象的属性赋值，而不需要明确的存取方法，这样就可以在运行时动的访问和修改对象的属性。

`KVC`的定义都是对 `NSObject` 的扩展来实现的，OC 中有个显示的 `NSKeyValueCoding` 类别名，所以对于继承 `NSObject`的都能使用 `KVC` （一些 swift 类和结构体是不支持`KVC`的，因为没有继承`NSObject`）

```objective-c
- (nullable id)valueForKey:(NSString *)key;                          //直接通过Key来取值

- (void)setValue:(nullable id)value forKey:(NSString *)key;          //通过Key来设值

- (nullable id)valueForKeyPath:(NSString *)keyPath;                  //通过KeyPath来取值

- (void)setValue:(nullable id)value forKeyPath:(NSString *)keyPath;  //通过KeyPath来设值

```

##### KVC 设值

`KVC` 要设值，就要对象对应的 key，但是寻找 key 的是有个顺序：

```markdown
当一个对象调用setValue:forKey: 方法时,方法内部会做以下操作:
1.判断有没有指定key的set方法,如果有set方法,就会调用set方法,给该属性赋值
2.如果没有set方法,判断有没有跟key值相同且带有下划线的成员属性(_key).如果有,直接给该成员属性进行赋值
3.如果没有成员属性_key,判断有没有跟key相同名称的属性.如果有,直接给该属性进行赋值
4.如果都没有,就会调用 valueforUndefinedKey 和setValue:forUndefinedKey:方法
```

如果上述方法或者成员变量都不存在，系统会执行该对象的**setValue：forUndefinedKey：** 方法，默认是抛出异常。

如果想让某一类禁用 `KVC`，那么重写 `accessInstanceVariablesDirectly`,翻译NO，这样如果没有找到setName,就会直接调用 **setValue：forUndefinedKey：**,抛出异常

```objective-c
+ (BOOL)accessInstanceVariablesDirectly {
    return NO;
}
```

##### KVC使用keyPath

一个类的成员变量可能是自定义的类 或者 其他的复杂数据类型，用`setValu: forKey:` 比较繁琐，这时候苹果提供了一个解决方案，健路径，就是按照路径寻找key。

```objective-c
//Test生成对象
Test *test = [[Test alloc] init];
//Test1生成对象
Test1 *test1 = [[Test1 alloc] init];
//通过KVC设值test的"test1"
[test setValue:test1 forKey:@"test1"];
//通过KVC设值test的"test1的name"
[test setValue:@"xiaoming" forKeyPath:@"test1.name"];
```

##### KVC处理数值和结构体类型属性

不是每一个方法都返回对象，但是valueForKey：总是返回一个id对象，如果原本的变量类型是值类型或者结构体，返回值会封装成NSNumber或者NSValue对象。 这两个类会处理从数字，布尔值到指针和结构体任何类型。然后开以者需要手动转换成原来的类型。 尽管valueForKey：会自动将值类型封装成对象，但是setValue：forKey：却不行。你必须手动将值类型转换成NSNumber或者NSValue类型，才能传递过去。 因为传递进去和取出来的都是id类型，所以需要开发者自己担保类型的正确性，运行时Objective-C在发送消息的会检查类型，如果错误会直接抛出异常。

##### KVC 使用

* 动态的取值和设置
* 用`KVC`来访问和修改私用变量
* `Model`和字典转换
* 修改一些控件的内部属性 （例如：`UITextField` 中的 `placeHolderText`）
* 操作集合

#### KVO

> KVO 是通过isa_swizzing 实现的，基本流程是编译器自动为被观察对象创造一个派生类，并将被观察对象的isa  指向这个派生类。如果用户注册目标对象的某一个属性的观察，那么派生类会重写这个方法，并添加通知变化代码。OC 在发送消息的时候，会通过 isa 指针找到当前对象所属的类对象。而类对象保存着当前对象的实例方法，因此在向此对象发送消息的时候，实际上是发送到啦派生类对象的方法。由于编辑器对派生类的方法进行了override ，并添加了通知代码，因此会向注册的对象发送通知。注意派生类只重写注册了观察者的属性方法

`KVO`的实现基于 Runtime 的强大动态能力。当一个 Person 类，添加了观察后，系统会生成一个 `NSKVONotifying_Person` 类，并将对象的 isa 指针指向新的类。

##### 重写setter

在setter 中，会添加一下两个方法的调用

```objective-c
- (void)willChangeValueForKey:(NSString *)key;
- (void)didChangeValueForKey:(NSString *)key;
```

### 3、KVO 是否能监听数组的count

正常是不能的，因为count 是`readonly`的，猜测只是实现了get 方法

```objc
@property (readonly) NSUInteger count;
```

但是`KVO`的底层是通过重写了熟悉的`set`方法，所以，监听不到

```objective-c
 - (void)_NSSetIntValueAndNotify{
    //将要修改age的值
    [self willChangeValueForKey:@"age"];
    //调用父类的setAge方法去修改age的值
    [super setAge:age];
    //完成修改age的值，并且执行observeValueForKeyPath方法
    [self didChangeValueForKey:@"age"];
}
```

但是，我们可以通过3种方法监听到：

1. `hook` 数组的`insert`、`remove`方法，手动调用`willChangeValueForKey`、`didChangeValueForKey`
2. [针对可变数组`NSMutableArray`](http://blog.xtxiete.com/2018/12/11/KVOMutableArray%E7%9A%84%E5%88%86%E6%9E%90%E5%92%8C%E7%90%86%E8%A7%A3/)

```objc
[[self.observerModel mutableArrayValueForKeyPath:@"array"] addObject:@"ff"];
```

```objc
-(void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context{
     NSLog(@"触发了");
    if([keyPath isEqualToString:@"array"]){
    }
}
```

**调用`mutableArrayValueForKey:`到底做了什么？**

`mutableArrayValueForKey:` 是`NSKeyValueCoding`协议里的一个方法，平时使用`KVC`用的最多的是`valueForKey:`这个方法，`mutableArrayValueForKey:`有点类似`valueForKey:`，都是通过`key`搜索返回一个对象，只是`mutableArrayValueForKey:`返回的是可变数组对象。另外一点就是调用`mutableArrayValueForKey:`后`KVC`的搜索顺序。

调用`mutableArrayValueForKey`后，`KVC`先会搜索类中是否有`insertObject:inAtIndex:` , `removeObjectFromAtIndex:` 或者 `insertAdIndexes` , `removeAtIndexes` 格式的方法，如果至少找到一个`insert`方法和一个`remove`方法，那么返回一个可以响应`NSMutableArray`所有方法的代理集合，那么给这个代理集合发送`NSMutableArray`的方法，以`insertObject:inAtIndex:` , `removeObjectFromAtIndex:` 或者 `insertAdIndexes` , `removeAtIndexes`组合的形式调用。

3. 重写一个数组类

   根据`mutableArrayValueForKey:`方法`KVC`的搜索顺序后，并且结合`KVO`的原理，我们可以新写一个`KVOMutableArrayObserver`类

   ```objective-c
   @interface ArrayTest : NSObject
   
   @property (nonatomic, strong) NSMutableArray* arrar;
   
   - (instancetype)init;
   - (instancetype)initWithMutableArray:(NSMutableArray*)array;
   
   - (void)removeObjectFromArrarAtIndex:(NSUInteger)index;
   - (void)removeArrarAtIndexes:(NSIndexSet *)indexes;
   - (void)insertObject:(id)obj inArrarAtIndex:(NSUInteger)index;
   - (void)insertArrar:(NSArray *)array atIndexes:(NSIndexSet *)indexes;
   @end
   ```

   ```objective-c
   @implementation ArrayTest
   - (instancetype)init
   {
       return [self initWithMutableArray:[NSMutableArray new]];
   }
   
   - (instancetype)initWithMutableArray:(NSMutableArray*)array
   {
       if((self = [super init]))
       {
           _arrar = array;
       }
       return self;
   }
   
   - (void)removeObjectFromArrarAtIndex:(NSUInteger)index;
   {
       [self.arrar removeObjectAtIndex:index];
   }
   
   - (void)removeArrarAtIndexes:(NSIndexSet *)indexes
   {
       [self.arrar removeObjectsAtIndexes:indexes];
   }
   
   - (void)insertObject:(id)obj inArrarAtIndex:(NSUInteger)index
   {
       NSLog(@"insertObject：%@ inArrarAtIndex:%ld",obj,index);
       [self.arrar insertObject:obj atIndex:index];
   }
   
   - (void)insertArrar:(NSArray *)array atIndexes:(NSIndexSet *)indexes;
   {
       [self.arrar insertObjects:array atIndexes:indexes];
   }
   
   - (void)willChangeValueForKey:(NSString *)key{
       [super willChangeValueForKey:key];
   
       NSLog(@"willChangeValueForKey : %@",key);
   }
   
   - (void)didChangeValueForKey:(NSString *)key{
   
       NSLog(@"didChangeValueForKey : %@ --begin",key);
   
       [super didChangeValueForKey:key];
   
       NSLog(@"didChangeValueForKey : %@ --end",key);
   
   }
   
   - (void)willChange:(NSKeyValueChange)changeKind valuesAtIndexes:(NSIndexSet *)indexes forKey:(NSString *)key{
       [super willChange:changeKind valuesAtIndexes:indexes forKey:key];
   
        NSLog(@"willChangeValueForKey : %@",key);
   }
   
   - (void)didChange:(NSKeyValueChange)changeKind valuesAtIndexes:(NSIndexSet *)indexes forKey:(NSString *)key{
       NSLog(@"didChangeValueForKey : %@ --begin",key);
       [super didChange:changeKind valuesAtIndexes:indexes forKey:key];
       NSLog(@"didChangeValueForKey : %@ --end",key);
   }
   ```

   然后找个控制器实例化，给数组添加`KVO`监听

   ```objective-c
   self.array = [NSMutableArray array];
   
   self.arrayTest = [[ArrayTest alloc] initWithMutableArray:self.array];
   
   [self.arrayTest addObserver:self forKeyPath:@"arrar" options:NSKeyValueObservingOptionNew|NSKeyValueObservingOptionOld context:nil]
   NSLog(@"添加KVO监听之后：-%@", object_getClass(self.arrayTest));
   NSLog(@"添加监听之后：- %p", [self.arrayTest methodForSelector:@selector(insertObject:inArrarAtIndex:)]);
   // 断点 使用LLDB打印一下地址的IMP
   
   [self.arrayTest insertObject:@"111" inArrarAtIndex:0]
   
   - (void)observeValueForKeyPath:(NSString *)keyPath ofObject:(id)object change:(NSDictionary<NSKeyValueChangeKey,id> *)change context:(void *)context{
   
       NSLog(@"观察的属性值改变了：%@",change);
   }
   ```

   

### 4、KVO 防止crash 方案

### 5、UIViewController的生命周期详解

1. `+(void)initialize` 第一次初始化的时候调用，再次创建不会调用 `initialize`方法。
2. `init`和`initCode` 初始化调用，代码初始化调用`init`，从 `nib`或者`xib`、`storyboard` 初始化会调用  `initCode`。
3. `loadView` 是开始加载`view`的方法，除非手动调用，否者生命周期只调用一次
4. `viewDidLoad` 类成员对象和变量的初始化我们都会放在这个方法中。在创建类后无论视图展现还是消失，这个方法也只会在布局是调用一次。
5. `viewWillAppear:(BOOL)animate` 视图将出现在屏幕之前，马上这个视图就会被展现在屏幕上了
6. `viewWillLayoutSubviews` 方法是在将要布局子视图的时候调用
7. `viewDidLayoutSubviews` 方法是在子视图布局完成后调用。
8. `viewDidAppear:(BOOL)animate` 方法是视图已经出现。
9. `viewWillDisappear:(BOOL)animate` 方法是视图即将消失。
10. `viewDidDisappear:(BOOL)animate` 视图已经消失
11. `dealloc` 被释放时调用

### 5. isKindOfClass 和 isMemberOfClass的区别

两者都是 `NSObject` 比较 `Class`的方法

`isKindOfClass` 来确定一个对象是否是一个类的成员，或者是派生自该类的成员

 isMemberOfClass` 只能确定一个对象是否是当前类的成员

例如：一个派生自`NSObject` 的类，`isMemberOfClass`不能检测当前类是基于`NSObject`类这一事实，但是`isKindOfClass`可以

```objective-c
[[NSMutableData data] isKindOfClass:[NSData class]]; // YES
[[NSMutableData data] isMemberOfClass:[NSData class]]; // NO
```

### 6. iOS frame和Bounds 以及frame和bounds区别

1. frame 不管对于位置还是大小，改变的都是自己的本身
2. frame 的位置是以父视图的坐标系为参照，从而确定当前试图在父视图中的位置
3. frame 大小改变的时候，当前试图的左上角的位置不回发生改变，只是大小改变了。
4. frame 是自己相对于父视图的定位，bounds 是自身的坐标系，默认以左上角为原点，所有子视图以这个坐标系的原点为基准点。
5. bounds 改变位置时，改变的是子视图的位置，自身没有影响；其实就是改变了本身的坐标系原点，默认是坐标系原点是左上角。
6. bounds 的大小改变是，当前试图的中心点不会发生改变，当前试图的大小发生改变，看起来效果就像缩放一样。





