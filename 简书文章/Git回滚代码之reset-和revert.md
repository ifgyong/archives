上篇文章讲过了[Git合并分支](https://www.jianshu.com/p/3012607c0450)，这里不多做讲述，在开发中，有可能遇到的情况是可能有个bug在上个节点已经提交过了，这个版本有复现，现在需要再次测试上个节点的代码，怎么办？

那么我们今天研究一下`git reset` 和`git revert`的区别和联系。
先看下`git reset [--soft | --mixed [-N] | --hard | --merge | --keep] [-q] [<commit>]`
`git reset [<mode>][<commit>]`
目的是重置当前分支的`head`指向到`commit`，并根据`model`更新索引(将其重置为`<commit>`)，如果省略`model`,则默认`model`是`--mixed`。`model`必须是下边的列表：
##### --soft
不用修改索引文件和工作树(和所有模式一样，将`head`指向`commit`)，这和`git status`一样，将保存所有要提交的更改。

##### --mixed
重置索引，但不重置工作树（即保留更改的文件，但不标记为提交），并报告未更新的内容。这是默认操作。(需要重新添加`git add`)

如果指定了-n，则删除的路径将标记为有意添加

##### --hard
重置索引树和工作树。自<commit>之后，对工作树中跟踪文件的任何更改都将被丢弃。
##### --merge
重置索引并更新工作树中<commit>和head之间不同的文件，但保留索引和工作树之间不同的文件（即具有未添加的更改）。如果在<commit>和索引之间不同的文件有未分页的更改，重置将中止。

换句话说，--merge执行类似于git read tree-u-m<commit>的操作，但会转发未合并的索引项。
##### --keep
重置索引项并更新工作树中<commit>和head之间不同的文件。如果<commit>和head之间不同的文件发生了本地更改，重置将中止。


If you want to undo a commit other than the latest on a branch,is your friend.
如果你不想取消当前分支的最近的commit，请看一下[git revert ](https://git-scm.com/docs/git-revert) 。这下边会讲。

我们造一个demo如下：
![demo](https://upload-images.jianshu.io/upload_images/783986-8aa14f320406515a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

现在我们的HEAD指向节点`d248cd`，但是我们想看节点的`file_2`的代码，
我们执行命令
```
git reset --soft head^1

Yong:test_git_revert Yong$ git log --graph
* commit b53ccde9dce1691b555dd5e42c8eb346f736ef81 (HEAD -> master)
| Author: fanguangyong <fanguangyong@xqafu.com>
| Date:   Fri May 10 14:29:07 2019 +0800
|
|     add file_2
|
* commit f5f2498c24c2771f906c2a1ac02b540f092fdcbb
| Author: fanguangyong <fanguangyong@xqafu.com>
| Date:   Fri May 10 14:28:55 2019 +0800
|
|     add file_1
|
* commit e002e347a24ad3079b283156887a7e7d67f881cc
  Author: fanguangyong <fanguangyong@xqafu.com>
  Date:   Fri May 10 14:28:05 2019 +0800

      Initial Commit

```
然后`add file_3`不见了,
``` Yong$ git status
On branch master
Changes to be committed:
  (use "git reset HEAD <file>..." to unstage)

	modified:   test_git_revert/ViewController.m
```
可以看出来 代码回退过来了，刚才提交的记录也没了，只是将head指针前移动了一个，但是修改的`test_git_revert/ViewController.m`文件是更改了，想要执行代码，还是所有的代码，紧紧是记录回退了，代码却是所有的代码(代码未动),只是工作区的 跟踪状态不一样。
![记录](https://upload-images.jianshu.io/upload_images/783986-b98e185994a5b293.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如何测试完了在次更新到最新的节点呢？
```
Yong:test_git_revert Yong$ git commit -am 'add file_3_repeat'
[master 200cb3f] add file_3_repeat
 1 file changed, 1 deletion(-)
Yong:test_git_revert Yong$ git log --graph
* commit 200cb3fddf04c03cb1661bf7dc9e7f84525c28b3 (HEAD -> master)
| Author: fanguangyong <fanguangyong@xqafu.com>
| Date:   Fri May 10 17:01:15 2019 +0800
|
|     add file_3_repeat
|
* commit b53ccde9dce1691b555dd5e42c8eb346f736ef81
| Author: fanguangyong <fanguangyong@xqafu.com>
| Date:   Fri May 10 14:29:07 2019 +0800
|
|     add file_2
|
* commit f5f2498c24c2771f906c2a1ac02b540f092fdcbb
| Author: fanguangyong <fanguangyong@xqafu.com>
| Date:   Fri May 10 14:28:55 2019 +0800
|
|     add file_1
|
* commit e002e347a24ad3079b283156887a7e7d67f881cc
  Author: fanguangyong <fanguangyong@xqafu.com>
  Date:   Fri May 10 14:28:05 2019 +0800

      Initial Commit
```
刚才`add file_3`已经消失了，因为又提交了一次，`message`变成了`add file_3_repeat`，新增了一条提交记录。

那么我们修改一下文件,当前`status`是：
![status](https://upload-images.jianshu.io/upload_images/783986-7d4875e316ed19bf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
然后执行回滚
```
git reset --soft head^1
```
结果是：
![--soft](https://upload-images.jianshu.io/upload_images/783986-b8baf68a1a2bac6a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
则没有提交缓存去的文件没有丢失，回滚的代码的状态是追踪，但是没添加到缓存区。
测试完成当前版本的时候，则再次`git add`
![git add](https://upload-images.jianshu.io/upload_images/783986-e8621c7d3ff5ae80.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### --mixed
```
Yong:test_git_revert Yong$ git reset --mixed head^1
Unstaged changes after reset:
M	test_git_revert/ViewController.m

Yong:test_git_revert Yong$ git log --graph
* commit b53ccde9dce1691b555dd5e42c8eb346f736ef81 (HEAD -> master)
| Author: fanguangyong <fanguangyong@xqafu.com>
| Date:   Fri May 10 14:29:07 2019 +0800
|
|     add file_2
|
* commit f5f2498c24c2771f906c2a1ac02b540f092fdcbb
| Author: fanguangyong <fanguangyong@xqafu.com>
| Date:   Fri May 10 14:28:55 2019 +0800
|
|     add file_1
|
```
代码混滚了，直接输出了最后的一次更改。提交的记录也丢失。
修改好代码 执行 `git add `。
![git add](https://upload-images.jianshu.io/upload_images/783986-8cf35d956daa72d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
##### --hard

![hard](https://upload-images.jianshu.io/upload_images/783986-718d6473284edb64.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
执行之后查看本地git缓存区的数据有变化吗？
![数据](https://upload-images.jianshu.io/upload_images/783986-f5a88a0c935c9047.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
--hard丢失节点3的数据。
![如何恢复--hard丢失的节点数据](https://upload-images.jianshu.io/upload_images/783986-00109ed870652c5e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

再测试一下本地有未追踪的文件的时候：
![未追踪file](https://upload-images.jianshu.io/upload_images/783986-f004fc11b5e0e8cb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
然后回滚代码：
![回滚代码](https://upload-images.jianshu.io/upload_images/783986-43a91e38bc67a228.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
文件丢失了怎么找回呢？那么我们测试一下`git reflog`看看能不能找回?
![能不能找回](https://upload-images.jianshu.io/upload_images/783986-31e17f4b874eee32.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
现在我们的head指向`2b8edf8`，那么我们让`head`指向到`e55c3e4`
```
$ git reset --hard head@{1}
HEAD is now at e55c3e4 add file3
```
![未提交的丢失](https://upload-images.jianshu.io/upload_images/783986-002bade7b479a450.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
我们执行查找lost的命令：
```
git fsck --lost-found


Checking object directories: 100% (256/256), done.
dangling commit e55c3e450d0a743b6a64db248065b3fdece153d1
dangling blob 6c3bfcdf65a27027ca3168bf2981487fa4ad5dfa
```
`e55c3e450d0a743b6a64db248065b3fdece153d1`是我们最后提交的代码节点，
`6c3bfcdf65a27027ca3168bf2981487fa4ad5dfa`是我们最后`add`之后形成的`commit`
然后执行`git show 6c3bfcdf65a27027ca3168bf2981487fa4ad5dfa`

![git show](https://upload-images.jianshu.io/upload_images/783986-65ecda9aa55ee473.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如果该条记录数 是`dangling commit id`类型直接使用`git merge commit_id`即可实现数据恢复，如果是`dangling blob id`类型则需要使用`git show 6c3bfcd> somefileName将其转储到文件`或者使用使用脚本[下载本数据写入到相关的文件](https://falstaff.agner.ch/2014/11/10/fix-git-repository/)
最后修改的文件成功找回。


那么不会回退节点怎么处理呢？
`git`给我们提供了`git revert`，会形成一个新的节点。

名称
`git-revert `- 恢复一些现有的提交

描述
给定一个或多个现有提交，还原相关修补程序引入的更改，并记录一些记录它们的新提交。这需要您的工作树是干净的（没有HEAD提交的修改）。

注意：git revert用于记录一些新的提交以反转某些早期提交的效果（通常只有一个错误的提交）。如果你想丢弃工作目录中所有未提交的更改，你应该看到git-reset [1]，特别是--hard选项。如果你想在另一个提交中提取特定文件，你应该看到git-checkout [1]，特别是git checkout <commit> -- <filename>语法。请注意这些替代方案，因为它们都会丢弃工作目录中未提交的更改。

`--continue`
使用.git / sequencer中的信息继续正在进行的操作 。可以在解决失败的挑选或恢复中的冲突后继续使用。

`-- quit`
忘记当前正在进行的操作。在樱桃挑选或恢复失败后，可用于清除顺序器状态。

`--abort`
取消操作并返回到预序列状态。

例子
`git revert HEAD~3`
还原HEAD中第四个最后一次提交所指定的更改，并使用还原的更改创建一个新提交。

`git revert -n master~5..master~2`

将提交所做的更改从master（包含）中的第五次提交恢复为master（包含）中的第三次提交，但不要使用还原的更改创建任何提交。恢复仅修改工作树和索引。

恢复一个提交记录并生成一个新的记录：
![新的记录](https://upload-images.jianshu.io/upload_images/783986-3c16bcdb7b658caa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![恢复并生成一个新的记录](https://upload-images.jianshu.io/upload_images/783986-b0a5855b29d605b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

生成了这个，又不想要了这个节点怎么办？
执行`git reset --hard head^1`即可
![hard](https://upload-images.jianshu.io/upload_images/783986-07ea2d0bde529591.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### Q & A：
##### 1.想恢复到某个版本并且测试代码怎么办？
使用`git reset --hard [commit_id]`回滚，测试完之后，使用`git reflog`查看记录，并再次使用`git reset --hard [commit_id]`回滚到指定代码版本。
##### 2.想恢复到某个版本并且测试代码,又测试出来一些bug该怎么办？
使用`git reset --hard [commit_id]`回滚之后修改代码，修改完成之后可以进行 `git push -f`进行强制推送到远程，前提是同学没有进行push，否则会覆盖掉其他同学的代码导致其他业务线出问题。

###### `--hard`恢复容易丢文件，切不容易查找，慎用。回滚代码并生成新的提交记录可使用`git revert [commit_id]`，用完之后再使用`--hard`丢掉即可。使用`reset`的时候最好有为缓存的文件及时`add `和`commit`，这也是个好习惯。

参考资料：
[官方资料](https://git-scm.com/docs/git-apply)
[falstaff](https://falstaff.agner.ch/2014/11/10/fix-git-repository/)






















