按钮的应用，这是比较基础的只是了，废话不多说，直接上代码：
```
class _MyHomePageState extends State<MyHomePage> {
  int _counter = 0;
  int distance = 2;
  void _incrementCounter() {
    setState(() {
      distance +=2;
      _counter += distance;
    });
  }
  void _descCounter() {
    setState(() {
      distance -=2;
      _counter -= distance;
    });
  }
  void _resetCount(){
    setState(() {
      _counter = 0;
      distance = 0;
    });
  }
  @override
  Widget build(BuildContext context) {

    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.spaceEvenly,
          children: <Widget>[
            myText(title: '老弟来了几次？',backgroundColor: Colors.red,fontSize: 30,),
            myText(title: '老弟来了'+'$_counter'+'次',backgroundColor: Colors.green,fontSize: 30,),
            myText(title: 'Step:'+'$distance',backgroundColor: Colors.red,fontSize: 30,),
            RaisedButton(onPressed: _incrementCounter,child: new Text("push me + "+"$_counter"),),
            RaisedButton(onPressed: _descCounter,
              child: new  Text('push me -'+'$distance')),
            RaisedButton(onPressed: _resetCount,
                child: new  Text('reset me =0')),
          ],
        ),
      ),
    );
  }
}
```
代码定义了三个函数`_incrementCounter ` `_descCounter ` `_resetCount `，三个按钮，主Container用的layout是Column，样式是`MainAxisAlignment.spaceEvenly`，间隔大小一样。
效果图：
![效果图](https://upload-images.jianshu.io/upload_images/783986-a53daed625338013.gif?imageMogr2/auto-orient/strip)
Flutter和RN基本一致，采用`setState`函数来出发渲染操作。把需要改变的值写在setState中就可以实现当值改变的时候，UI也跟着改变。

我们来看一下Layout如何使用的？下边三个按钮如何改成横向的？
其实这个page是竖向的，所有元素则都是竖向的，但是我们在一个元素中重新定义一个Container，设置`child`的`Row`，则单个元素的子元素变成横向的，上边的代码变成:
```
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.spaceEvenly,
          children: <Widget>[
            myText(title: '老弟来了几次？',backgroundColor: Colors.red,fontSize: 30,),
            myText(title: '老弟来了'+'$_counter'+'次',backgroundColor: Colors.green,fontSize: 30,),
            myText(title: 'Step:'+'$distance',backgroundColor: Colors.red,fontSize: 30,),
            new Container(
              child: Row(
                mainAxisAlignment: MainAxisAlignment.spaceEvenly,
                children: <Widget>[
                  RaisedButton(onPressed: _incrementCounter,child: new Text("push me + "+"$_counter"),),
                  RaisedButton(onPressed: _descCounter,
                      child: new  Text('push me -'+'$distance')),
                  RaisedButton(onPressed: _resetCount,
                      child: new  Text('reset me =0')),
                ],
              ),
            )
          ],
        ),
      ),
```
改变之后样式是：
![改变之后样式](https://upload-images.jianshu.io/upload_images/783986-6c78d6c21aa1c3e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
其实flutter的UI架构是一个Container决定一种layout，每一次新的布局，则需要新生成一个Container。现在我们来分析一个复杂的UI:

![淘宝](https://upload-images.jianshu.io/upload_images/783986-002a819d15f9c8c4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
伪代码实现如下:
```
contain column{
contain row
 subContain row
contain column 
}
```
最外层一个Container，banner一个Container，icon一个Container，imageList一个Container。每个Container单独布局。
基本架构如下：
![UI控件架构图](https://upload-images.jianshu.io/upload_images/783986-06ef8157d7fae0b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在看一下成品图：
![成品图](https://upload-images.jianshu.io/upload_images/783986-7d930221aa566be1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
class _MyHomePageState extends State<MyHomePage> {
  int _counter = 0;
  int distance = 2;
  void _incrementCounter() {
    setState(() {
      distance +=2;
      _counter += distance;
    });
  }
  void _descCounter() {
    setState(() {
      distance -=2;
      _counter -= distance;
    });
  }
  void _resetCount(){
    setState(() {
      _counter = 0;
      distance = 0;
    });
  }
  @override
  Widget build(BuildContext context) {

    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.spaceEvenly,
          children: <Widget>[
            myText(title: '老弟来了几次？',backgroundColor: Colors.red,fontSize: 30,),
            myText(title: '老弟来了'+'$_counter'+'次',backgroundColor: Colors.green,fontSize: 30,),
            myText(title: 'Step:'+'$distance',backgroundColor: Colors.red,fontSize: 30,),
            new Container(
              child: Row(
                mainAxisAlignment: MainAxisAlignment.spaceEvenly,
                children: <Widget>[
                  RaisedButton.icon(
                      icon: new Image.asset('source/icon_3.png',width: 20,),
                      onPressed: _incrementCounter,
                      label: new Text("push + "+"$_counter"),
                  color: Colors.white,),
                  RaisedButton.icon(
                      icon:new Image.asset('source/icon_1.png',width: 20,),
                      onPressed: _descCounter,
                      label: new  Text('push -'+'$distance'),
                      color: Colors.white
                  ),
                  RaisedButton.icon(
                    icon:new Image.asset('source/icon_2.png',width: 20,) ,
                      onPressed: _resetCount,
                      label: new  Text('reset =0'),
                      color: Colors.white,
                  ),

                ],
              ),
            ),
            setTwoIconLine(),
          ],
        ),
      ),
    );
  }
}
class setTwoIconLine extends StatelessWidget{
  @override
  Widget build(BuildContext context) {
    // TODO: implement build
    return new Container(
      child: Column(
        children: <Widget>[
          setLineIcon(),
          setLineIcon(),
        ],
      ),
    );
  }
}
class setLineIcon extends StatelessWidget{
final List showTitles;
setLineIcon({this.showTitles});

  @override
  Widget build(BuildContext context) {
    return  new Container(
      padding: const EdgeInsets.only(top: 2,bottom: 3),
      child: Row(
        mainAxisAlignment: MainAxisAlignment.spaceEvenly,

        children: <Widget>[
          _View(bgColor: Colors.green,title: "小白来了",iconname: 'source/icon_1.png',),
          _View(bgColor: Colors.red,title: "小鹿来了",iconname: 'source/icon_2.png',),
          _View(bgColor: Colors.green,title: "小红来了",iconname: 'source/icon_3.png',),
          _View(bgColor: Colors.red,title: "小黑来了",iconname: 'source/icon_1.png',),
        ],
      ),
    );
  }
}
class _View extends StatelessWidget{
  final Color bgColor;
  final String title;
  final String iconname;
  _View({this.bgColor,this.title,this.iconname});
  @override
  Widget build(BuildContext context) {
    return new Container(
      color: Colors.white,
      padding: const EdgeInsets.all(0),
        width: 80,
        height: 80,
      child: Column(
        children: <Widget>[
          imageIcon(iconname:this.iconname),
          new Container(
            padding: const EdgeInsets.only(top: 10),
            child: Text(this.title,style: TextStyle(fontSize: 12),),
          )
        ],
      ),
    );
  }
}
class imageIcon extends StatelessWidget{
  final String iconname;
  imageIcon({this.iconname});
  @override
  Widget build(BuildContext context) {
    // TODO: implement build
    return new Container(
        padding: const EdgeInsets.only(top: 10),
        child: Image.asset(this.iconname,width: 40,),
    );
  }
}
```

今天先到这里，后边在写一个具体功能的Demo，喜欢的可以Start哦。
[代码在这里flutter_02](https://github.com/ifgyong/demo)



参考文章：
- [flutter.dev](https://flutter.dev/)
- [docs.flutter.io](https://docs.flutter.io/flutter/material/RaisedButton-class.html)
