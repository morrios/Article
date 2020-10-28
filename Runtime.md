# Runtime 

Runtime 虽说是C语言的超集，但是和 C语言不同， C语言函数的调用，在编译期就决定好了，编译后顺序执行，但是 OC 是动态语言，在编译器不知道调用哪个函数。所以 Runtime 就是解决如何在运行时期找到调用方法这样的问题。

### Runtime 消息传递

```objective-c
[obj foo] -> obj_msgSend(obj, foo)
```

* 首先，通过 `obj` 的 `isa` 指针找到它的 `Class`
* 在 `Class`中的 `method_list`知道要执行的方法`foo`
* 如果没找到要找的方法 `foo`，就往`superclass`中找
* 一旦找到`foo`，执行 `IMP`

简而言之：

```objective-c
instance -> class -> method -> SEL -> IMP -> 实现函数
```

但是执行的过程中，我们会发现，可能这个`Class`中的方法很多，`method_list`很大，每次寻找这个方法，都要遍历一次`method_list`。但是很多方法是不经常调用，只有一部分是频繁调用的，这时候次次遍历`method_list`就会很不合理。

如果我们把经常执行的方法缓存起来，就会大大的提升查询效率。`objc_cache`就是为此出现的。找到 `foo`后，就已`foo`的`method_name`作为key，`method_imp`作为`value`存起来。当再次收到`foo`消息的时候，直接从`cache`里找到，避免去遍历`method_list`。

![](https://tva1.sinaimg.cn/large/0081Kckwly1gk3wqwp18aj30fa0fzt9b.jpg)