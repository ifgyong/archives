layout其实就是布局，分为`Row`和`Column`,一个是横向的，一个是竖向的，那么我们就拿一个页面进行具体分析：
![page](https://upload-images.jianshu.io/upload_images/783986-3b712adce3375278.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这是iOS系统的HeadthAPP，的data页面，他是有两部分组成，一个是头部的四个image和底部的listView，可以这样子实现伪代码：
```
 @override
  Widget build(BuildContext context) {

    return Scaffold(
      appBar: AppBar(
        title: Text(widget.title),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.start,
          children: <Widget>[
new A(),
new B()(
          ],
        ),
      ),
    );
  }
```
顶部的A和底部的B,A和B再单独封装成组件。首先整个页面上使用Column排列，第二行ListView就是第二个元素，然后这个ListView内部可以Row或者Column排列两种方式，
A可以这样子实现:
```
Widget _buildDecoratedImage(int imageIndex) => Expanded(
      child: Container(
        decoration: BoxDecoration(
          border: Border.all(width: 10, color: Colors.black38),
          borderRadius: const BorderRadius.all(const Radius.circular(8)),
        ),
        margin: const EdgeInsets.all(4),
        child: Image.asset('images/pic$imageIndex.jpg'),
      ),
    );
//每次排列2个image
Widget _buildImageRow(int imageIndex) => Row(
  children: [
//左边image
    _buildDecoratedImage(imageIndex),
//右边的image
    _buildDecoratedImage(imageIndex + 1),
  ],
);
//布局4个图片
Widget A() => Container(
  decoration: BoxDecoration(
    color: Colors.black26,
  ),
  child: Column(
    children: [
      _buildImageRow(1),
      _buildImageRow(3),
    ],
  ),
);

```
下边的listView可以这样子实现:
```
Widget _buildList() => ListView(
      children: [
        _tile('CineArts at the Empire', '85 W Portal Ave', Icons.theaters),
        _tile('The Castro Theater', '429 Castro St', Icons.theaters),
        _tile('Alamo Drafthouse Cinema', '2550 Mission St', Icons.theaters),
        _tile('Roxie Theater', '3117 16th St', Icons.theaters),
        _tile('United Artists Stonestown Twin', '501 Buckingham Way',
            Icons.theaters),
        _tile('AMC Metreon 16', '135 4th St #3000', Icons.theaters),
        Divider(),
        _tile('Kescaped_code#39;s Kitchen', '757 Monterey Blvd', Icons.restaurant),
        _tile('Emmyescaped_code#39;s Restaurant', '1923 Ocean Ave', Icons.restaurant),
        _tile(
            'Chaiya Thai Restaurant', '272 Claremont Blvd', Icons.restaurant),
        _tile('La Ciccia', '291 30th St', Icons.restaurant),
      ],
    );

ListTile _tile(String title, String subtitle, IconData icon) => ListTile(
      title: Text(title,
          style: TextStyle(
            fontWeight: FontWeight.w500,
            fontSize: 20,
          )),
      subtitle: Text(subtitle),
      leading: Icon(
        icon,
        color: Colors.blue[500],
      ),
    );
```

