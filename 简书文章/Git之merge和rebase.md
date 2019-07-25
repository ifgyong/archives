作为几年的开发者，会用merge但是不很清楚和rebase的区别,有点尴尬，今天查了一下资料，大概了解了一下，现在详细讲解一下这两者用法以及区别。


![初始化Git文件](https://upload-images.jianshu.io/upload_images/783986-3b1fea6d7edce3f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### master merge
分支2 和master
现在在master分支上执行
```
Git merge 2
Yong:git_02 Yong$ git merge 2
Auto-merging 1.txt
CONFLICT (content): Merge conflict in 1.txt
Automatic merge failed; fix conflicts and then commit the result.
Yong:git_02 Yong$
```
报错了，因为我们修改的是同一个文件的相同位置，所以会冲突。
我们修改好这个冲突文件，冲突文件是：
![冲突文件](https://upload-images.jianshu.io/upload_images/783986-d6bdbc1e733eaf29.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
下图是打开文件冲突内容：
![冲突内容](https://upload-images.jianshu.io/upload_images/783986-a409091ec5e3dc2e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上边是`HEAD`分支的内容，`=====`下边是2分支的内容，项目中根据实际情况作出取舍。
修改成：
![修改](https://upload-images.jianshu.io/upload_images/783986-87023b51bb9a5803.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
修改完之后进行提交：
```
Yong:git_02 Yong$ git add .
Yong:git_02 Yong$ git commit -a -m 'fix conflict'
[master b02486a] fix conflict
Yong:git_02 Yong$
```
提交后分支记录为：
![image.png](https://upload-images.jianshu.io/upload_images/783986-ffd66196f6404b32.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
然后我们切换成`checkout 2`查看内容，
![分支2内容](https://upload-images.jianshu.io/upload_images/783986-c9621ed33a6a9bba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
对，没看错，分支2还是原来的内容。那小伙伴该有什么疑问呢？刚才我们合并产生冲突了，而且冲突已经修复了，修复之后的内容是`5`，那么为什么`branch 2`内容没变呢？因为我们是在`master`上`merge`的，分支2合并到了master，master上有master和分支2的内容，但是分支2的代码没变。我们在看一下 `master`的内容：
![master](https://upload-images.jianshu.io/upload_images/783986-a6f2110b69e067c2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 所以我们多人合作开发的时候，产生冲突了，然后又用merge合并了分支，把冲突改好了最后也提交了，下一个任务来的时候，还是基于自己的老分支开发，等任务结束的时候，合并分支发现？怎么冲突还在？解决这个问题的 话，每次新的任务都从稳定的`dev`new 一个分支出来，在新的分支开发任务，这样子，每次切出来的分支都是一个新的分支，而且是有同事提交的代码的。不会之前的冲突，下次提交的时候也冲突第二次。
或者旧的任务完成了，分支2 要`git pull origin master`，保持自己分支是最新的内容。


那么我们现在将分支2的内容改成
```
1234
5
```
然后在`master`分支上
```
git merge 2
```
冲突了：
![冲突](https://upload-images.jianshu.io/upload_images/783986-397f6815d465378e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
打开冲突看一下，原来第一次合并冲突在分支2中的内容并没有修改，然后新增了一行5，导致旧版本`1234`和`5`都冲突了。
![冲突](https://upload-images.jianshu.io/upload_images/783986-71d7bb754ad35521.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


### master rebase
![image.png](https://upload-images.jianshu.io/upload_images/783986-552a443b9309a08c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![冲突节点](https://upload-images.jianshu.io/upload_images/783986-262a8d5d0e5f2b52.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
修改成：
![image.png](https://upload-images.jianshu.io/upload_images/783986-794d61ce9dabe83d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
然后执行：
```
git rebase --continue
```
第二次冲突：
![第二次冲突](https://upload-images.jianshu.io/upload_images/783986-d8edcb9fd7a4c3a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

冲突修改完成了，结果如图：
![结果](https://upload-images.jianshu.io/upload_images/783986-027f067797b5b20a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

解决完冲突：
![解决完冲突](https://upload-images.jianshu.io/upload_images/783986-ac86cb697fbcb72c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后我们在分支`branch2`上修改文件：
旧版：
![初始化Git文件](https://upload-images.jianshu.io/upload_images/783986-3b1fea6d7edce3f4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

旧版和新版对比：
所有节点在一条线上，说明合并到了一个分支上，重合的节点内容是一致的。`branch2`上的内容还是旧版内容，开发新的任务需要从新pull或者 从`dev`新切除分支来解决内容不同步问题。
然后从`branch2`作出修改：




然后在其中`branch2`分支上 修改，提交，生成一个新的节点，从master上出来的。
![branch2修改](https://upload-images.jianshu.io/upload_images/783986-af8eb48f08550e24.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
然后在`rebase`的基础上进行`merge`操作，差异很明显。

![rebase and merge](https://upload-images.jianshu.io/upload_images/783986-96a93cd760db1730.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



```
$ git log --graph --pretty=oneline --abbrev-commit
*   d6ce4b2 (HEAD -> master) merge from 2
|\
| * 9e7c360 (2) 1.txt add line
* | 206cfd1 add6
* | e67db5a 1_m
|/
* 91052c8 4
* a8331c2 3
* 680986b 2
* e6a896f 1.txt
```

> 其实rebae就是按照时间线的前后顺序进行合并，a分支和b分支，`a rebase b`，b的最后一个节点分别和a节点进行对比和合并。a和b合并成一条线。
`a merge b`只是a分支和b分支最后一个提交节点进行对比合并，中间的节点不进行对比和合并。所以合并结果是2条线。

参考资料：
- [git-scm.com](https://git-scm.com/book/zh/v2/Git-%E5%88%86%E6%94%AF-%E5%8F%98%E5%9F%BA)




















