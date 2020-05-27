# Git相关问题

## Git rebase && Git merge

#### **什么是 rebase**?

**git rebase** 你其实可以把它理解成是“重新设置基线”，将你的当前分支重新设置开始点。这个时候才能知道你当前分支于你需要比较的分支之间的差异。

基线是上一个变更的结束，下一个变更的开始。

 原理很简单：rebase需要基于一个分支来设置你当前的分支的基线，这基线就是当前分支的开始时间轴向后移动到最新的跟踪分支的最后面，这样你的当前分支就是最新的跟踪分支。这里的操作是基于文件事务处理的，所以你不用怕中间失败会影响文件的一致性。在中间的过程中你可以随时取消rebase 事务。

官方解释:  [https://git-scm.com/book/zh/v2/Git-](https://links.jianshu.com/go?to=https%3A%2F%2Fgit-scm.com%2Fbook%2Fzh%2Fv2%2FGit-)分支-变基

#### **git rebase 和 git merge 有啥区别？**

**rebase**会把你当前分支的 commit 放到公共分支的最后面,所以叫变基。就好像你从公共分支又重新拉出来这个分支一样。
 举例:如果你从 master 拉了个feature分支出来,然后你提交了几个 commit,这个时候刚好有人把他开发的东西合并到 master 了,这个时候 master 就比你拉分支的时候多了几个 commit,如果这个时候你 rebase master 的话，就会把你当前的几个 commit，放到那个人 commit 的后面。

![img](https:////upload-images.jianshu.io/upload_images/1547393-a7e4e04dd5ee4c09.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/332/format/webp)



**merge** 会把公共分支和你当前的commit 合并在一起，形成一个新的 commit 提交

![img](https:////upload-images.jianshu.io/upload_images/1547393-5f57703ff8b889d3.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/584/format/webp)





##### **注意:**

- 不要在公共分支使用rebase
- 本地和远端对应同一条分支,优先使用rebase,而不是merge

###### 抛出问题:

**为什么不要再公共分支使用rebase?**
 因为往后放的这些 commit 都是新的,这样其他从这个公共分支拉出去的人，都需要再 rebase,相当于你 rebase 东西进来，就都是新的 commit 了

- 1-2-3 是现在的分支状态
- 这个时候从原来的master ,checkout出来一个prod分支
- 然后master提交了4.5，prod提交了6.7
- 这个时候master分支状态就是1-2-3-4-5，prod状态变成1-2-3-6-7
- 如果在prod上用rebase master ,prod分支状态就成了1-2-3-4-5-6-7  （把当前分支的 commit 放到公共分支的后面
- 如果是merge
   1-2-3-6-7-8
   ........ |*4-5*|
- 会出来一个8，这个8的提交就是把4-5合进来的提交 （会把公共分支和你当前的commit 合并在一起，形成一个新的 commit 提交）

**merge和rebase实际上只是用的场景不一样**
 **更通俗的解释一波.**
 比如rebase,你自己开发分支一直在做,然后某一天，你想把主线的修改合到你的分支上,做一次集成,这种情况就用rebase比较好.把你的提交都放在主线修改的头上
 如果用merge，脑袋上顶着一笔merge的8,你如果想回退你分支上的某个提交就很麻烦,还有一个重要的问题,rebase的话,本来我的分支是从3拉出来的,rebase完了之后,就不知道我当时是从哪儿拉出来的我的开发分支
 同样的,如果你在主分支上用rebase, rebase其他分支的修改,是不是要是别人想看主分支上有什么历史,他看到的就不是完整的历史课,这个历史已经被你篡改了

**常用指令**

- git rebase -I dev 可以将dev分支合并到当前分支
   这里的”-i“是指交互模式。就是说你可以干预rebase这个事务的过程，包括设置commit message，暂停commit等等。
- git rebase –abort 放弃一次合并
- **合并多次commit操作:**
   1 git rebase -i dev
   2 修改最后几次commit记录中的pick 为squash
   3 保存退出,弹出修改文件,修改commit记录再次保存退出(删除多余的change-id 只保留一个)
   4 git add .
   5 git rebase --continue



## Git Flow

### 1. 简介

Git Flow定义了一个项目发布的分支模型，为管理具有预定发布周期的大型项目提供了一个健壮的框架。

### 2、流程解析

1. **master**分支存放所有正式发布的版本，可以作为项目历史版本记录分支，不直接提交代码。仅用于保持一个对应线上运行代码的 code base。
2. **develop**分支这个分支是我们是我们的主开发分支，包含所有要发布到下一个Release的代码，这个主要合并与其他分支，比如Feature分支
3. **feature**分支为新功能分支，feature分支都是基于develop创建的，开发完成后会合并到develop分支上。同时存在多个
4. **release**分支基于最新develop分支创建，当新功能足够发布一个新版本(或者接近新版本发布的截止日期)，从develop分支创建一个release分支作为新版本的起点，**用于测试，所有的测试bug**在这个分支改。测试完成后合并到master并打上版本号，同时也合并到develop，更新最新开发分支。(一旦打了release分支之后不要从develop分支上合并新的改动到release分支)，同一时间只有1个，生命周期很短，只是为了发布。
5. **hotfix**分支基于master分支创建，对线上版本的bug进行修复，完成后直接合并到master分支和develop分支，如果当前还有新功能release分支，也同步到release分支上。同一时间只有1个，生命周期较短



![1366859-eda8da6a7d2385ad](E:\研究生学习\Work\技术笔记\JVM 垃圾回收.assets\1366859-eda8da6a7d2385ad.webp)

### 3. 创建流程

第一步为master分支配套一个develop分支。简单来做可以本地创建一个空的develop分支，push到服务器上：

```undefined
git branch –b develop
git push -u origin develop
```

以后这个分支将会包含了项目的全部历史，而master分支将只包含了部分历史。其它开发者这时应该克隆中央仓库，建好develop分支的跟踪分支：



```php
git clone ssh://user@host/path/to/repo.git
git checkout -b develop origin/develop
```

现在每个开发都有了这些历史分支的本地拷贝。从develop分支拉一个特性分支进行开发



```undefined
git checkout -b some-feature develop
git push(如果这个功能需要多个人协作，建议push)
```

用老套路添加提交到各自功能分支上：编辑、暂存、提交：



```csharp
git status
git add
git commit
git push(如果这个功能需要多个人协作)
```

添加了提交后，如果团队使用Pull Requests，这时候可以发起一个用于合并到develop分支。否则就直接合并到本地的develop分支后push到中央仓库



```cpp
git pull origin develop
git checkout develop
git merge some-feature
git push
git branch -d some-feature
git push origin --delete some-feature (如果这个功能需要多个人协作) 
```

第一条命令在合并功能前确保develop分支是最新的。注意，功能决不应该直接合并到master分支。
 然后用一个新的分支来做发布准备。这一步也确定了发布的版本号：



```css
git checkout -b release-1.0.0 develop
```

这个分支是清理发布、执行所有测试、更新文档和其它为下个发布做准备操作的地方，像是一个专门用于改善发布的功能分支。只要创建这个分支并push到中央仓库，这个发布就是功能冻结的。任何不在develop分支中的新功能都推到下个发布循环中。
 一旦准备好了对外发布，合并修改到master分支和develop分支上，删除发布分支。



```css
git checkout master
git merge release-1.0.0
git push
git checkout develop
git merge release-1.0.0
git push
git branch -d release-1.0.0
git push origin --delete release-1.0.0
```

发布分支是作为功能开发（develop分支）和对外发布（master分支）间的缓冲。只要有合并到master分支，就应该打好Tag以方便跟踪。



```css
git tag -a 1.0.0 -m "Initial public release" master
git push --tags
```

对外发布后，发现了当前版本的一个Bug，从master分支上拉出了一个Hotfix分支，提交修改以解决问题，然后直接合并回master分支



```bash
git checkout -b issue-#001 master
Fix the bug…..
git checkout master
git merge issue-#001
git push
```

就像发布分支，维护分支中新加这些重要修改需要包含到develop分支中，然后才删除这个Hotfix分支



```bash
git checkout develop
git merge issue-#001
git push
git branch -d issue-#001
```



### 4. 常用命令

***git flow init***：初始化一个现有的 git 库,将会设置一些初始的参数，如分支前缀名等，建议用默认值。

***git flow feature start [featureBranchName]***:  创建一个基于'develop'的feature分支，并切换到这个分支之下。

***git flow feature finish [featureBranchName]***:  完成开发新特性,  合并 MYFEATURE 分支到 'develop',  删除这个新特性分支,  切换回 'develop' 分支。

***git flow feature publish [featureBranchName]***：发布新特性分支到远程服务器，也可以使用git的push命令

***git flow feature pull origin [featureBranchName]***：取得其它用户发布的新特性分支，并签出远程的变更。也可以使用git的pull命令

***git flow feature track [featureBranchName]***：跟踪在origin上的feature分支。

***git flow release start [releaseBranchName]***：开始准备release版本，从 'develop' 分支开始创建一个 release 分支。

***git flow release publish[releaseBranchName]***：创建 release 分支之后立即发布允许其它用户向这个 release 分支提交内容。

***git flow release track[releaseBranchName]***：签出 release 版本的远程变更。

***git flow release finish [releaseBranchName]***：归并 release 分支到 'master' 分支，用 release 分支名打 Tag，归并 release 分支到 'develop'，移除 release 分支。

***git flow hotfix start [hotfixBranchName]***：开始 git flow 紧急 修复，从master上建立hotfix分支。

***git flow hotfix finish [hotfixBranchName]***：结束 git flow 紧急修复，代码归并回 develop 和 master 分支。相应地，master 分支打上修正版本的 TAG

## 参考

* https://www.jianshu.com/p/4079284dd970
* https://www.jianshu.com/p/34b95c5eedb6