> 最近一直在整理之前的笔记，由于之前的笔记仅是自己看的，也没太注意质量与内容，自己知道怎么回事就行了。等到想着发出来，分享给大家看看，才发现要做的工作不少，也算是自己给自己挖坑了......但正所谓温故而知新，整理过后其实也算又有收获，也希望看到文章的朋友能有所获就更好了。
>
> 笔记的内容，知识点有浅有深（大多都算浅的，菜是原罪）。有的是软件的使用，有的是框架的原理，也有的可能就一两句话，刚好解决了我的疑惑，随手就复制粘贴过来了......废话不说了，直接进入今天的正题——Git。
>
> Git，应该是现在较多公司所采用的版本控制系统，GitHub也是被戏称为全球最大同性交友网站，国内的码云也发展的十分迅速，我所在的公司也是利用码云在做代码托管。这篇文章不算是Git的入门教程，仅算是我自用的常用命令速查手册，有需要看详细教程的，可以参考廖雪峰老师的教程，我当时就是照着他的教程入门，这篇笔记也是当时整理自廖老师的博客内容，原文链接：[廖雪峰的官方网站-Git教程](https://www.liaoxuefeng.com/wiki/896043488029600)。
>

# 1.概念补充

## 1.1 集中式和分布式

Git是目前世界上最先进的分布式版本控制系统。

- 集中式版本控制系统，版本库是集中存放在中央服务器的，而干活的时候，用的都是自己的电脑，所以要先从中央服务器取得最新的版本，然后开始干活，干完活了，再把自己的活推送给中央服务器。

- 分布式版本控制系统根本没有“中央服务器”，每个人的电脑上都是一个完整的版本库

Git与SVN区别：本地是否有完整的版本库历史！

假设SVN服务器没了，那你丢掉了所有历史信息，因为你的本地只有当前版本以及部分历史信息。

假设GitHub服务器没了，你不会丢掉任何git历史信息，因为你的本地有完整的版本库信息。你可以把本地的git库重新上传到另外的git服务商。

## 1.2 工作区和暂存区

#### 工作区（Working Directory）

就是你在电脑里能看到的目录

#### 版本库（Repository）

工作区有一个隐藏目录`.git`，这个不算工作区，而是Git的版本库。

Git的版本库里存了很多东西，其中最重要的就是称为stage（或者叫index）的暂存区（**就像购物车一样**），还有Git为我们自动创建的第一个分支`master`，以及指向`master`的一个指针叫`HEAD`。

![](https://s1.ax1x.com/2020/08/23/d0qBZR.jpg)



git add的反向命令git checkout，撤销工作区修改，即把暂存区最新版本转移到工作区，

git commit的反向命令git reset HEAD，就是把仓库最新版本转移到暂存区。

# 2.操作

## 2.1 创建版本库

```
git init
```

## 2.2 版本控制

### 查看修改

```
git diff xxx（文件名）
```

git diff 比较的是工作区文件与暂存区文件的区别（上次git add 的内容） ，如果暂存区没有文件，则比较工作区中的文件与上次提交到版本库中的文件

git diff --cached 比较的是暂存区的文件与仓库分支里（上次git commit 后的内容）的区别 

git diff比较的是工作目录中当前文件和暂存区域快照之间的差异， 也就是修改之后还没有暂存起来的变化内容。若要查看已暂存的将要添加到下次提交里的内容，可以用 git diff --cached 命令。

请注意，git diff 本身只显示尚未暂存的改动，而不是自上次提交以来所做的所有改动。 所以有时候你一下子暂存了所有更新过的文件后，运行 git diff 后却什么也没有，就是这个原因。

提交后，用`git diff HEAD -- readme.txt`命令可以查看工作区和版本库里面最新版本的区别：

### 提交

先统一提交暂存区

```
git add xxx（文件名）/git add .(提交当前目录下所有)
```

再提交到本地仓库

```
git commit -m "add distributed"
```

其中，-m即添加注释

### 查看提交历史

多次提交后，可以通过`git log`来查看日志

```
$ git log
commit c886584771f3e0a4856a10d98c4616ef2e88b0c2 (HEAD -> master)
Author: wyf <xxx@163.com>
Date:   Sun Dec 29 17:35:33 2019 +0800

    add second

commit 2336d2f0b0b6a616a6e88285a936283f08e5cc13
Author: wyf <xxx@163.com>
Date:   Sun Dec 29 17:34:30 2019 +0800

    add first
```

可以添加`--pretty=oneline`参数，将信息归并为一行

```
$ git log --pretty=oneline
c886584771f3e0a4856a10d98c4616ef2e88b0c2 (HEAD -> master) add second
2336d2f0b0b6a616a6e88285a936283f08e5cc13 add first
```

其中，最前面的字符串为`commit id`（版本号）

### 版本回退

首先，Git必须知道当前版本是哪个版本，在Git中，用`HEAD`表示当前版本，也就是最新的提交`1094adb...`（注意我的提交ID和你的肯定不一样），上一个版本就是`HEAD^`，上上一个版本就是`HEAD^^`，当然往上100个版本写100个`^`比较容易数不过来，所以写成`HEAD~100`。

```
git reset --hard HEAD^
```

回退至上一个版本

回退至指定版本号的仓库内容

```
git reset --hard commit-id
```

其中，版本号`commit-id`没必要写全，前几位就可以了，只要能确保唯一即可。

### 查看命令历史

`git reflog`用来记录你的每一次命令：

```
$ git reflog
c886584 (HEAD -> master) HEAD@{0}: commit: add second
2336d2f HEAD@{1}: commit: add first
```

### 撤销修改

```
git checkout -- fileName
```

命令`git checkout -- fileName`意思就是，把文件在**工作区的修改全部撤销**，这里有两种情况：

一种是文件自修改后还没有被放到暂存区，现在，撤销修改就回到和版本库一模一样的状态；

一种是文件已经添加到暂存区后，又作了修改，现在，撤销修改就回到添加到暂存区后的状态。

总之，就是让这个文件回到最近一次`git commit`或`git add`时的状态。

如果已经提交至暂存区，可以使用`git reset HEAD <file>`可以把**暂存区的修改撤销掉**（unstage），重新放回工作区

### 删除文件

新建文件并commit，直接在工作区删除

1.一是确实要从版本库中删除该文件，那就用命令`git rm`删掉，并且`git commit`：

```
$ git rm test.txt
rm 'test.txt'
$ git commit -m "remove test.txt"
```

2.另一种情况是删错了，因为版本库里还有呢，所以可以很轻松地把误删的文件恢复到最新版本：

```
$ git checkout -- test.txt
```

## 2.3 远程仓库

### 添加远程库

已经在本地创建了一个Git仓库后，又想在GitHub创建一个Git仓库，并且让这两个仓库进行远程同步

假设，在GitHub上已经创建空仓库`learngit`，在本地运行命令

```
$ git remote add origin git@github.com:xxx(github账户名)/learngit.git(仓库名)
```

添加后，远程库的名字就是`origin`，这是Git默认的叫法，也可以改成别的，但是`origin`这个名字一看就知道是远程库。

下一步，就可以把本地库的所有内容推送到远程库上：

```
$ git push -u origin master
```

把本地库的内容推送到远程，用`git push`命令，实际上是把当前分支`master`推送到远程。

由于远程库是空的，我们第一次推送`master`分支时，加上了`-u`参数，Git不但会把本地的`master`分支内容推送的远程新的`master`分支，还会把本地的`master`分支和远程的`master`分支关联起来，在以后的推送或者拉取时就可以简化命令。

极端情况下，也可以强制push

```
$ git push -u origin master -f
```

之后，只要本地作了提交，就可以通过命令：

```
$ git push origin master
```

把本地`master`分支的最新修改推送至GitHub，现在，你就拥有了真正的分布式版本库！

**总结：**

要关联一个远程库，使用命令`git remote add origin git@server-name:path/repo-name.git`；

关联后，使用命令`git push -u origin master`第一次推送master分支的所有内容；

此后，每次本地提交后，只要有必要，就可以使用命令`git push origin master`推送最新修改；

### 从远程库克隆

已经有了远程库

```
$ git clone 远程仓库地址
```

## 2.4 分支管理

一开始的时候，`master`分支是一条线，Git用`master`指向最新的提交，再用`HEAD`指向`master`，就能确定当前分支，以及当前分支的提交点：

![git-br-initial](https://s1.ax1x.com/2020/08/23/dBVDwq.png)

每次提交，`master`分支都会向前移动一步，这样，随着你不断提交，`master`分支的线也越来越长。

当我们创建新的分支，例如`dev`时，Git新建了一个指针叫`dev`，指向`master`相同的提交，再把`HEAD`指向`dev`，就表示当前分支在`dev`上：

![git-br-create](https://s1.ax1x.com/2020/08/23/dBV4mR.png)

你看，Git创建一个分支很快，因为除了增加一个`dev`指针，改改`HEAD`的指向，工作区的文件都没有任何变化！

不过，从现在开始，对工作区的修改和提交就是针对`dev`分支了，比如新提交一次后，`dev`指针往前移动一步，而`master`指针不变：

![git-br-dev-fd](https://s1.ax1x.com/2020/08/23/dBVO6H.png)

假如我们在`dev`上的工作完成了，就可以把`dev`合并到`master`上。Git怎么合并呢？最简单的方法，就是直接把`master`指向`dev`的当前提交，就完成了合并：

![git-br-ff-merge](https://s1.ax1x.com/2020/08/23/dBZ9tf.png)



所以Git合并分支也很快！就改改指针，工作区内容也不变！

合并完分支后，甚至可以删除`dev`分支。删除`dev`分支就是把`dev`指针给删掉，删掉后，我们就剩下了一条`master`分支：

![git-br-rm](https://s1.ax1x.com/2020/08/23/dBZe7q.png)

### 创建和合并分支

查看分支：`git branch`

创建分支：`git branch <name>`

切换分支：`git checkout <name>`或者`git switch <name>`

创建+切换分支：`git checkout -b <name>`或者`git switch -c <name>`

合并某分支到当前分支：`git merge <name>`

删除分支：`git branch -d <name>`

**注意：**git 切换分支时会把未add或未commit的内容带过去。 也就是说，对于所有分支而言， 工作区和暂存区是公共的。

### 解决冲突

当Git无法自动合并分支时，就必须首先解决冲突。解决冲突后，再提交，合并完成。

解决冲突就是把Git合并失败的文件手动编辑为我们希望的内容，再提交。

用`git log --graph`命令可以看到分支合并图。

发生合并冲突后，Git用`<<<<<<<`，`=======`，`>>>>>>>`标记出不同分支的内容

```
Git is a distributed version control system.
Git is free software distributed under the GPL.
<<<<<<< HEAD
Creating a new branch is quick & simple.
=======
Creating a new branch is quick AND simple.
>>>>>>> feature1
```

### 分支管理

通常，合并分支时，如果可能，Git会用`Fast forward`模式，但这种模式下，删除分支后，会丢掉分支信息。

如果要强制禁用`Fast forward`模式，Git就会在merge时生成一个新的commit，这样，从分支历史上就可以看出分支信息。

可以在合并时使用`--no-ff`参数，表示禁用`Fast forward`：

```
git merge --no-ff -m "merge with no-ff" dev
```

合并后的历史有分支，能看出来曾经做过合并，而`fast forward`合并就看不出来曾经做过合并。

### 暂时储存未完成的分支

分支`dev` 最初也是从`master` 分支上衍生出来的。而此时你要再从该分支上切换到其主分支。那么你需要先把该`dev`分支上的改动提交后才能切换，但是该`dev`分支上还没有完成全部的修改，你不想提交。那么此时你就要选择 `stash` 它们

```
$ git stash
```

然后就可以切换到需要开发的分支处理了，开发完成提交即可。

```
$ git commit -m "fix bug 101"
[issue-101 4c805e2] fix bug 101
 1 file changed, 1 insertion(+), 1 deletion(-)
```

如果后续要再切回来继续开发，用`git stash list`命令看看：

```
$ git stash list
stash@{0}: WIP on dev: f52c633 add merge
```

工作现场还在，Git把stash内容存在某个地方了，但是需要恢复一下，有两个办法：

一是用`git stash apply`恢复，但是恢复后，stash内容并不删除，你需要用`git stash drop`来删除；

另一种方式是用`git stash pop`，恢复的同时把stash内容也删了

另外，在master分支上修复了bug后，我们要想一想，dev分支是早期从master分支分出来的，所以，这个bug其实在当前dev分支上也存在。

在master分支上修复的bug，想要合并到当前dev分支，可以用`git cherry-pick <commit>`命令，把bug提交的**修改“复制”到当前分支**，避免重复劳动。

```
git cherry-pick 4c805e2
```

`4c805e2`就是之前修复bug提交后的xommit-id。

**补充理解：**

**总的来说，就是，在分支下进行的工作，如果不commit的话，回到master，就会显示出你在分支下你添加的工作。这个时候，你在master下修改完bug提交后，正在分支进行的工作也会提交了。为了避免这个情况，你就在分支下，git stash将工作隐藏，这个时候，切换到master时候，修改了bug，提交。分支的内容不会被提交上去。**

### 删除一个没有被合并的分支

```
$ git branch -d feature-vulcan
error: The branch 'feature-vulcan' is not fully merged.
If you are sure you want to delete it, run 'git branch -D feature-vulcan'.
```

销毁失败。Git友情提醒，`feature-vulcan`分支还没有被合并，如果删除，将丢失掉修改，如果要强行删除，需要使用大写的`-D`参数。。

现在我们强行删除：

```
$ git branch -D feature-vulcan
Deleted branch feature-vulcan (was 287773e).
```

### 推送分支

推送分支，就是把该分支上的所有本地提交推送到远程库。推送时，要指定本地分支，这样，Git就会把该分支推送到远程库对应的远程分支上：

```
$ git push origin master
```

如果要推送其他分支，比如`dev`，就改成：

```
$ git push origin dev
```

### 抓取分支

当你从远程仓库克隆时，实际上Git自动把本地的`master`分支和远程的`master`分支对应起来了，并且，远程仓库的默认名称是`origin`。要查看远程库的信息，用`git remote`：

```
$ git remote
```

或者，用`git remote -v`显示更详细的信息：

本地克隆远程仓库后，默认情况下，只能看到本地的`master`分支。不信可以用`git branch`命令看看：

```
$ git branch
* master
```

现在，要在`dev`分支上开发，就必须创建远程`origin`的`dev`分支到本地，于是用这个命令创建本地`dev`分支：

```
$ git checkout -b dev origin/dev
```

### 多人推送至远程仓库冲突

假如其他人的最新提交和你试图推送的提交有冲突

先用`git pull`把最新的提交从`origin/dev`抓下来

可能`git pull`也失败了，原因是没有指定本地`dev`分支与远程`origin/dev`分支的链接，根据提示，设置`dev`和`origin/dev`的链接：

```
git branch --set-upstream-to=origin/dev dev
```

然后，在本地合并，解决冲突，再推送。

```
git branch --set-upstream-to=origin/<branch> dev
```

## 2.5 标签管理

Git的标签虽然是版本库的快照，但其实它就是指向某个commit的指针（跟分支很像对不对？但是分支可以移动，标签不能移动），所以，创建和删除标签都是瞬间完成的。

### 创建标签

- 命令`git tag <tagname>`用于新建一个标签，默认为`HEAD`，也可以指定一个commit id，`git tag <tagname> commit-id`；
- 命令`git tag -a <tagname> -m "blablabla..."`可以指定标签信息；
- 命令`git tag`可以查看所有标签。

### 操作标签

- 命令`git push origin <tagname>`可以推送一个本地标签；
- 命令`git push origin --tags`可以推送全部未推送过的本地标签；
- 命令`git tag -d <tagname>`可以删除一个本地标签；
- 命令`git push origin :refs/tags/<tagname>`可以删除一个远程标签。











