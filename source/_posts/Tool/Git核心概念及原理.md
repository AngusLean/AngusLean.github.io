---
title: Git核心概念及原理
date: 2019/04/20
updated: 2019/04/20
tags:
   - Git
   - 协作
categories:
   - Tool
---



Git是现在最为常用的版本管理工具，无论是在各大开源项目，或者是公司内部项目，现在几乎都会采用Git作为版本管理工具。但是，很多人对Git的使用仅限于在IDE或者图形化工具里Pull/Push，而对Git的真正强大一无所知。本文将对**Git的核心概念及其实现**做个简短的介绍，力求把Git最重要的点写出来。其中大部分内容都是我在公司的一次内部Git培训中所讲的。

<!--more-->


## Git的设计原则
### 诞生背景
对于Git诞生的背景，网上有很多相关内容，这里就不复制。
Git是linux内核的最初开发者linus一手设计的，在最初设计这个工具的时候它所面对的目标就是linux内核这种多人异地协作开发、特性多、代码库较大的项目。所以，它为这个工具指定了如下几个要求:
- 快速
- 足够简单
- 对非线性开发的强力支持（数千个分支并行开发）
- 完全分布式
- 能够处理像linux内核一样大的项目（速度、数据大小）
### 优势
在上述前提下，Git很快就诞生了。并且很快在开源世界流行起来，进一步的，几乎完全取代了传统的SVN工具。那么，相较于SVN，它到底有哪些“致命”的优势呢？
从我对SVN有限的使用经验来看，如下几点最为重要:
- 非集中式存储
非集中式存储，也就是说不依赖于特定的服务器来完成功能。传统的SVN需要依赖特定的服务器来实现版本管理功能，对于断网、服务器崩溃这些情况完全无能为力。而Git则是完全的分布式，不依赖于特定服务器来进行版本管理(我们日常使用时，依然需要访问我们的Git服务器，但这仅仅是为了不同的人的协作，而对于版本管理则完全式本地进行的，后面会有原理性的说明 )。彻底的去中心化极大的增强了Git的可用性，毕竟没人能够接受自己的仓库代码丢失。
- 分支的强力支持
如上述的设计目标，Git能够同时支持上千个分支开发，并且还能够以极快的速度和很小的空间占用来运行。对于任何一个协作者较多的项目，这都是一个不可或缺的特性。想象一下你同时在几个分支工作，结果每次切换分支都要几十秒甚至更多，那就完全没有效率可言。
## 文件提交与更新
### 基本使用方式
相较于传统的svn的提交，在命令行里，我们使用Git提交代码时好像变得更为麻烦:
```
git add *
git commit -m "提交信息"
git push
```
提交一个文件需要3条命令，这好像是一件非常复杂的事情。当然，现代IDE对这些功能做了很大程度的封装。在IDEA里，默认是这样的:

![idea中git提交](https://gitee.com/angus_lean/markDownPic/raw/master/2019/04/idea-git-commit.png)
通常，右下方的选项我们会选择`Commit and Push`。这里其实对于很多文件，IDEA默认帮我们做了上面命令里的`add`，而这里的`Commit and Push`则对应着上面的两条命令。
那么，Git为什么要设计这样的流程?
上面我们说到，GIt的设计目标有一个很重要的点是**“完全分布式"**，而Git为了实现这一点所采用的实现方式就是每个人都拥有完整的仓库-也就是说，你克隆了一个项目，那你本地就拥有这个项目完整的历史。
这个模型是这样的:
![git本地存储模型](https://gitee.com/angus_lean/markDownPic/raw/master/2019/04/git-local-model.png)
上图反应到我们的项目结构就是
>
```
.git目录
xxx项目文件
xxx项目文件
xxx项目目录
```
其中，`.git`目录就是我们的**版本库**, 而所有的项目文件则就是**工作区**。`git add, git commit`这两条命令都是为我们的**版本库**服务，他们的区别是这样的:
- `git add`表示把文件**暂时保存**。对于任意更改了的文件需要做这个操作
- `git commit`则表示**提交**这个文件到版本库中

简单的说，我们通过`add`命令来标识哪些文件需要被Git保存一下，而通过`commit`命令来标识**提交**哪些文件到版本库中。我们对文件的更改完成过后，就需要把这个文件此时的状态"保存"到版本库中，那么，就是这个`commit`命令。一个正常的工作流是这样的：
1. 修改代码，写完过后。 `git add xxx`暂存这个文件
2. 如果这时候写完了，则`git commit `提交当前所有暂存的文件。如果还有其他需要更改的，则继续更改
3. 对于需要git管理的文件，**任何**时刻我们想保存当前的文件状态时，都可以`git add`来保存。而`commit`则是对所有暂存的文件统一做一次提交

上述的流程给了我们在代码编写时文件版本管理很大的灵活空间：对文件的多个状态保存多次、合并多个文件为一次提交、撤销当前对文件的更改到上一次更改/上一个版本等。上图中的`stage`就对应者我们的**暂存区**
那么，对于一次"提交",git内部到底发生了什么，又是如何保存我们的文件内容？
### 实现原理
#### 文件
对于版本管理工具如何保存我们的文件，有两个最显而易见的方法：
- 保存多个版本之间的差异，也就是保存差异
- 保存整个文件，也就是保存快照

对于保存文件差异来说，每次提交文件都是保存的差异，在对文件进行回退或者查看某个指定版本内容的时候需要遍历所有的差异文件，传统的SVN就是采用的这种方式。它有如下的特点:
1. 不同版本的文件的空间占用较小
2. 版本跳转需要遍历多个版本，对于较大的项目，存在效率上的问题

![svn本地存储模型](https://gitee.com/angus_lean/markDownPic/raw/master/2019/04/vcs-git-file.png)
而对于每次更改都保存整个文件，则有如下的特点:
1. 不同版本的文件占用的空间相较于差异保存会大一些
2. 版本跳转会非常快，因为直接是文件更改

Git的设计就是为了处理像linux内核这样的庞大项目，”空间换时间“是一个很显而易见的问题（远远不止这一个原因），Git选择了保存快照的方式，也就是**Git保存快照而不是差异**。
这也就是说，**每次文件`commit`， 都会在版本库中保存一份这个状态的文件**,对于每一次提交，GIt都会为所有更改的文件创建一个关联的SHA-1值，并且把这个SHA-1值作为文件名称。文件里再通过另外的格式表示文件原始名称。
![git本地存储模型](https://gitee.com/angus_lean/markDownPic/raw/master/2019/04/git-file.png)
上面说到的仅仅是如何保存某个指定的文件，那么对于目录、目录变化、目录与文件的关系又是如何保存的呢？
#### 目录
Git把目录也表示为一个文件，当然这个文件的名字依然是一个SHA-1值，然后在这个文件里保存对应目录下文件的SHA-1值。也就是这样的形状:
![git本地目录模型](https://gitee.com/angus_lean/markDownPic/raw/master/2019/04/git-tree-model.png)
一个简单的例子是:
>
```
目录名:  src
创建时间: 2019.04.18 16:40:04
创建人:  张三
子文件列表:
    文件1 SHA-1
    文件2 SHA-1
子目录列表:
    文件夹1 SHA-1
    文件夹2 SHA-1
```
这样，我们就把目录与文件串联起来了。 那么，我们对一个文件更改并提交了，则会造成:
- 生成一个新的文件，并且有对应的SHA-1
- 父级目录本身也生成一个新的文件，这个新文件中文件的SHA-1值都不改变，仅仅只有更改了文件的SHA-1值改变。
- 不断向上遍历，直到新的根目录
通过这样的网状结构，Git就完整的表达了某个目录的很多个版本，且版本之间要进行切换也仅仅只需要访问对应的文件即可。效率上很高。
那么，对于`commit`本身，又是如何保存的呢？
#### `commit`提交
从上面文件提交到根目录的更改，我们可以看到一个事实：对于每一次commit, 最终始终会生成一个新的目录的表达文件。而提交本身只需要保存这个表达文件的
SHA-1值以及提交时间、提交人、提交信息、上一次提交就可以了。所以对于`commit`本身，也表达为一个文件，这个文件里保存一个根目录的SHA-1即可。也就是这样的：
![git commit模型](https://gitee.com/angus_lean/markDownPic/raw/master/2019/04/git-commit-model.png)
我们通过Git的内建命令具体查看一下一次提交的内容:
![git commit模型](https://gitee.com/angus_lean/markDownPic/raw/master/2019/04/git-commit-model-demo.png)
上图命令里的`cat-file`是Git提供的通过文件的SHA-1查看文件内容的命令。 b4442这个SHA-1就是一次提交，我们可以看到它里面包含了这么几个信息:
- 这次提交对应的目录的SHA-1（bcbc这个）
- 上一次提交的SHA-1
- 提交作者
- 提交的注释信息

图里下部分则展示了bcbc这个SHA-1对应的内容。可以看到它里面有这么几种类型:
- `blog`,也就是上面我们说的文件
- `tree`，也就是目录

这样，我们清晰的看到了Git `commit`的整体实现方式。这些SHA-1文件其实就在目录`.git/objects`里，只是Git把SHA-1值的前面2位提取出来了以优化目录结构，可以根据具体
的SHA-1值在`.git/objects`目录里找到对应的文件。

两次提交的模型也就是：
![git commit模型](https://gitee.com/angus_lean/markDownPic/raw/master/2019/04/git-commit-model-demo1.png)

#### `tag`
进一步的，对于 `tag`，我们可以以完全相同的方式来实现。也就不多赘述

#### 分支
分支与`commit`所不同的是，`commit`代表的是状态值，而分支是线性的。但是，通过某个具体的`commit`我们可以找到对应的所有链条。所以，分支的实现也和`commit`如出一辙：
![git commit模型](https://gitee.com/angus_lean/markDownPic/raw/master/2019/04/git-branch-model.png)
一个具体的提交的示意图：
```
git branch master
```
![git commit模型](https://gitee.com/angus_lean/markDownPic/raw/master/2019/04/git-branch-demo1-1.png)

```
git checkout -b iss53
```
![git commit模型](https://gitee.com/angus_lean/markDownPic/raw/master/2019/04/git-branch-demo1-2.png)
```
$ echo 'Hello World' >> index.html
$ git commit -a -m 'added a new footer [issue 53]'
```
![git commit模型](https://gitee.com/angus_lean/markDownPic/raw/master/2019/04/git-branch-demo1-3.png)

提交总会产生新的commit,而分支则总是指向了 某个`commit`。 分支、Tag，都是保存在目录`.git/refs`里。

而对于标识**当前项目的状态**, Git新增了一个叫做`HEAD`的指针，这个指针里面的内容直接指向的是分支。当我们切换分支的时候，HEAD也就跟着变化，
这样就知道我们处于哪个分支了，进一步的，可以找到当前的所有文件。
```
$ cat HEAD
ref: refs/heads/source
$ cat refs/heads/source  //source是一个分支
06090e8a98e07b1a5fa25c6d3880e30868e23738
```

## 协作
在我看来，协作分为2部分：
- 不同人之间的代码的同步
- 不同人对同一个文件更改的合并

下面分别介绍这两部分
#### 代码同步
在上面我们较为仔细的介绍了Git本地代码库的实现方式，可以发现一个很明显的特点：本地的代码库已经包含了项目的整个版本包，所以我们在同步的时候也比较方便，使用一个中央仓库来保存
一份项目，然后本地和远程同步时直接比较SHA-1即可。有不同的则是冲突，进入冲突处理流程。
所以，在Git里，Github/Gitlab之类的工具仅仅是提供了”同步“的功能以及其他的管理需要功能点。但是对于版本管理这一个点来说，本地代码库是已经满足要求的。
#### 分支合并
分支合并是新人在使用Git最容易犯错的一个地方，除开粗心大意删除他人代码的原因，理解了合并的原理后再出问题的机率不大。看例子：
![git分支合并模型](https://gitee.com/angus_lean/markDownPic/raw/master/2019/04/git-branch-merge-1.png)
master分支当前指向提交C4，iss53分支当前指向提交C5.我们需要把iss53分支合并到master分支。其中，这两个分支的最近公共祖先是提交C2

![git分支合并模型](https://gitee.com/angus_lean/markDownPic/raw/master/2019/04/git-branch-merge-2.png)
合并的逻辑是 (C5-C2)+C4,**差异性比较**＊.也就是说当前两个分支的状态与最近公共祖先的差异的累加。
 一个非常容易引起问题的是一个分支上删除了某个目录或文件，另一个分支对改目录或文件没有更改，合并时会自动删除它而不会冲突。
合并时会创建一个新的提交，如果在merge时只提交了部分代码，会导致其他代码丢失！
对于单文件的合并逻辑是:
    1. 判断文件文件是否双方都删除了或者一方删除
　合并结果是删除文件(包括暂存区和工作目录)
    2. 一方添加，一方没有添加文件
　另外一个分支没有添加，则做更新暂存区操作；另外一个分支添加文件则git做添加文件、更新暂存区操作
    3. 双方都添加了文件
　首先判断权限是否一致，否则报错。然后做添加文件、更新暂存区操作
    4.双方都更改了文件
　对于上图所示的合并，iss53合并到master，就是把C5-C2的更改合并到C4上。必须注意的“差异变更”

## 常用命令
### 代码回退
![git代码回退](https://gitee.com/angus_lean/markDownPic/raw/master/2019/04/git-code-revert.png)
具体的操作参考上图即可。如果彻底的理解了本文所述的逻辑，在具体情况下要进行的代码回退会有一个清晰的思路。
### 分支相关
Q1: 多个feature分支，多个merge导致git log看起很乱，想整理为一条直线？
```
git checkout feature-xxx
git rebase master
```
如下是`merge`与`rebase`的区别
merge:
![git代码回退](https://gitee.com/angus_lean/markDownPic/raw/master/2019/04/git-merge-2.png)

rebase:
![git代码回退](https://gitee.com/angus_lean/markDownPic/raw/master/2019/04/git-rebase.png)


Q2: 在错误的分支做了提交，想要“迁移”到其他分支？
```
git branch feature
git reset --hard origin/master
git checkout feature
```

Q3: 在合并后的分支上做了更改，想把这部分更改”复制“到其他分支上?
```
git checkout feature-xxx
git cherry-pick 1dfd9d 3zz33d xxxxx
```






