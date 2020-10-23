## Flutter 文本垂直居中

~~~dart
Container(
  color: ColorUtil.aaRandomColor(),
  height: txBottom,
  child: Column(
    mainAxisAlignment: MainAxisAlignment.center,
    children: <Widget>[Text("文本")]
  ))
~~~

利用 Column 子控件居中