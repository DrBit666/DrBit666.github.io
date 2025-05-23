---

title: "【Git】：merge与rebase"
date: 2025-03-15 13:53:00 +0800
tags: Git
categories: 技术

---

### 1.来源
```txt
最近公司在服务上线过程中，客户端同事误以将未经测试的代码合并到主分支上，造成新包出现了问题。
最后只能让客户端组长将误提的代码挨个revert，暂时解决了问题。
不过这样也造成了一个新问题：后续误提的代码再合并到master时，可能会有文件缺失。

问题产生的原因很简单，在我心里也引出了一个新课题：很多人对于git的使用很多仅限于pull,push,merge等操作

我们需要对Git有更深入的了解，至少心中需要有一张提交链表图(或者能看懂提交链表图)，才能避免在Git使用过程中出错
```

本文将从Git基础理论出发，展开讲讲Git的提交链表，以及分支合并中merge与rebase的使用

---

### 2.Git基础理论

* Git作为分布式版本协作系统，为我们提供了文件处理协作、文件版本控制、文件快照等功能。那么，Git到底是怎么保存数据的？

#### 2.1.Git是如何保存数据的

在Git中，我们常提到的分区有四个：

* 工作区：working directory,也就是根目录下包含.git文件夹的目录，表示当前目录由Git管理
* 暂存区：staged directory,使用git add . 即可将目录下所有文件添加到暂存区，暂存区的文件会被git追踪(tracked)
* 本地仓库：local repository，使用git commit时，会将当前暂存区中的文件提交到本地仓库，形成一次git提交
* 远程仓库：remote repository，可以与多个本地仓库关联，用于git协作

```txt
[工作区] ---- git add ----> [暂存区] ---- git commit ----> [本地仓库] ---- git push ----> [远程仓库]
   |                         |                          |                          |
编辑文件                  准备提交快照             记录历史快照             同步到远程
```

我们知道，Git可以帮我们维护不同版本的代码数据。实际上，Git维护的是每个版本的`文件快照`。
也就是说，在每次提交时，Git会将`暂存区`中的所有文件以快照形式提交到`.git`文件夹下，形成一次提交。

在一次提交中，往往存储了以下内容：
* commitId：提交的唯一标识符
* file-tree：本次提交的文件快照树
* author：本次提交的作者信息
* timestamp
* 以及一个指向本次提交的上一个提交的指针，形成了一棵提交树(首次提交git init为根提交，没有向前的指针)

### 3.Git分支

一般情况下，由根提交往后一直提交生成的一个链表被称之为主分支，一般为master分支。

尽管我们将master称之为'分支'，实际上他就是一个指向主链表最新提交的指针

![master只是指向主链表最新提交的指针](assets/pic/2025-03-15/master只是指向主链表最新提交的指针.png)

* 如何检出一个新分支？

```zsh
# develop分支
git branch dev
```

当使用git branch时，实际上是创建了一个新指针，它指向了当前分支所指向的提交，此时你的head指针还是指向master

![检出develop](assets/pic/2025-03-15/检出develop.png)

* 切换到新分支
```zsh
git checkout dev

```

此时，你的本地head指向了dev分支，checkout的过程也就是切换head的过程

![checkoutdev](../../assets/pic/2025-03-15/checkoutdev.png)

#### 3.1.Git 分支合并

##### 3.1.1.merge合并

当你在master分支和dev分支都分别有后续合并后，此时如果你需要提交合并请求，将dev分支的代码合并到master

此时分支状态如下：

![merge1](assets/pic/2025-03-15/merge1.png)


```zsh
# 本地切换到master分支并拉最新代码，使master指向主链表最新的提交
git checkout master
git pull
```

![merge2](assets/pic/2025-03-15/merge2.png)


```zsh
# 执行合并
git merge dev
```

merge合并操作，会将两只分支的最新快快照(33sbf和8kzis)合并，并与两个分支的最近共同祖先(3zzwc)合并
![merge3](assets/pic/2025-03-15/merge3.png)

合并完成后,会将dev分支链表上的提交按时间顺序与主分支链表合并，`然后生成一个新的merge commit`，并将master指向这个新的merge commit

![merge4](assets/pic/2025-03-15/merge4.png)


#### 3.1.2.rebase分支变基
除了merge，有时我们也会使用rebase来变基分支

我们还是假设合并前分支情况如下

![merge2](assets/pic/2025-03-15/merge2.png)


此时我们将变基dev分支到master分支

```zsh
git checkout dev
git rebase main

git checkout main
git merge --no-ff dev  # --no-ff 禁用快速提交，如果main分支也有变动的话，会生成一个新的合并提交
```

合并完成后，得到的git分支情况如下

![merge5](assets/pic/2025-03-15/merge5.png)


简单来说，rebase就是将当前分支的基改为目标分支的最新提交(所以叫变基)
比如，master分支有A,B,C的提交
此时feature在C提交时checkout出来(`此时feature是基于C检出的`)，并且feature后面进行了两次提交D,E
此时，master分支也进行了两次提交F,G
此时执行

```zsh
git checkout feature
git rebase master
```

feature分支将从基于C改为基于G，并重新提交D,E
由于D,E本来是基于C提交的，因此重新提交时，可能有冲突需要你来解决
重新提交后,feature分支链表变为A,B,C,F,G,D',E'

所以，此时执行

```zsh
git checkout master
git merge feature
```

master的提交记录也会变成A,B,C,D,F,G,D',E'。看起来就是将feature分支的提交按顺序放到了master后面，实际上D',E'是解决冲突后的D和E

#### 3.1.3.merge vs rebase

merge vs rebase

* merge
  * 优点：
    * 保留完整提交历史，由于merge commit的存在，方便查看不同分支的合并情况
    * 适用于多人协作，避免改写历史，且合并后不会影响远程仓库中的提交
  * 缺点：
    * 提交历史变得更复杂，尤其是有多个分支并行开发，会产生很多merge commit
* rebase
  * 优点：
    * 提交历史是线性的，更清晰
    * 避免不必要的merge commit,提交记录更精简清晰
  * 缺点：
    * 改写了提交历史，更可能导致冲突，特别是多人协作时不建议对已推送的分支知行rebase
    * 可能破坏他人基于旧分支的开发进度

因此，在合并选择上，如果是多人协作开发，更推荐使用merge，对于个人开发且希望对历史有线性整理的话，可以选择使用rebase


