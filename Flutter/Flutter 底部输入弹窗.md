## Flutter 底部输入弹窗

### 要实现一个底部弹窗，带输入框的，如下图



<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1gicf8rhmeyj30u01sz47b.jpg" style="zoom:25%" />

### BottomSheet

首先，我们知道这个可以利用 BottomSheet 实现这个功能。BottomSheet 是从底部弹出的菜单，展示 BottomSheet 有两种方式，分别是 `showBottomSheet` 和 `showModalBottomSheet 。两种方式只有在展示类型上的差别，方法调用无差，`showBottomSheet` 会充满整个屏幕，而 `showModalBottomSheet` 展示的高度不会超过半个屏幕的高度。所以，我们在这里选择使用 `showModalBottomSheet。

### showModalBottomSheet

在这里注意一点， showModalBottomSheet 是有默认高度的，我看通过查看源代码可以知道

```dart
 maxHeight: isScrollControlled
        ? constraints.maxHeight
        : constraints.maxHeight * 9.0 / 16.0,
```

他的高度为 constraints.maxHeight * 9.0 / 16.0。但是如果我们要的高度没有这么高，可以通过包一层 Container 设置高度即可。

```dart
showModalBottomSheet(
      context: context,
      builder: (ctx) {
        return Container(
            height: 100,
            );
      },
   );
```



<img src="https://tva1.sinaimg.cn/large/007S8ZIlly1gicfo66w2wj30u01szau8.jpg" style="zoom:14%" />

### 随键盘弹出上移

接下来我们在 Container 中加入一个 TextField ，发现键盘并没有随着键盘上移。这里我们使用 Stack + Positioned 针对底部的距离布局

~~~dart
Stack(
        children: <Widget>[
          Positioned(
            left: 12,
            bottom: (MediaQuery.of(context).viewInsets.bottom < 0)
                ? 0
                : MediaQuery.of(context).viewInsets.bottom,
            right: 0,
            top: top,
            child: Container(
              height: 100,
            ),
          )
        ],
      )
~~~

主要是根据 MediaQuery.of(context).viewInsets.bottom 设置底部距离。

接下来我们把 TextField 和主要代码加进去

~~~dart
void _showDi() {
    final top = 12.0;
    final txBottom = 40.0;
    final txHeight = 100.0;
    showModalBottomSheet(
      context: context,
      isScrollControlled: true,
      builder: (ctx) {
        return StatefulBuilder(builder: (ctx2, state) {
          return Container(
            height: MediaQuery.of(ctx2).viewInsets.bottom +
                txHeight +
                top +
                txBottom,
            color: Colors.white,
            child: Stack(
              children: <Widget>[
                Positioned(
                    left: 12,
                    bottom: (MediaQuery.of(ctx2).viewInsets.bottom < 0)
                        ? 0
                        : MediaQuery.of(ctx2).viewInsets.bottom,
                    right: 0,
                    top: top,
                    child: Container(
                      child: Column(
                        crossAxisAlignment: CrossAxisAlignment.start,
                        children: <Widget>[
                          Row(
                            crossAxisAlignment: CrossAxisAlignment.end,
                            children: <Widget>[
                              Expanded(
                                  child: Container(
                                padding: EdgeInsets.all(8),
                                height: txHeight,
                                color: ColorUtil.c_F7F7F7,
                                child: TextField(
                                  // scrollPadding: EdgeInsets.zero,
                                  autofocus: true,
                                  maxLines: 4,
                                  style: TextStyle(
                                      fontSize: 15, color: ColorUtil.c_353535),
                                  decoration: InputDecoration(
                                      contentPadding: EdgeInsets.zero,
                                      isDense: true,
                                      border: InputBorder.none),
                                ),
                              )),
                              Container(
                                padding: EdgeInsets.only(left: 12, right: 12),
                                child: Text("发送",
                                    style: TextStyle(
                                        fontSize: 15,
                                        color: ColorUtil.c_FF803E)),
                              ),
                            ],
                          ),
                          Container(
                            height: txBottom,
                            child: Image.asset(
                              "assets/images/evaImage_pcker.png",
                              height: 22,
                              width: 22,
                            ),
                          ),
                        ],
                      ),
                    ))
              ],
            ),
          );
        });
      },
    );
  }
~~~



