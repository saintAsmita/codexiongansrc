---
title: Git的操作展示
date: 2019-05-31 18:06:37
tags: Git
---

本文主要是介绍在实际开发过程中，对于Git的一些操作，用来给大家分享的同时，也用以记录。对于稍微复杂点的操作，我会配上图文例子
<!--more-->

## Git 结构
如果你有时间，可以直接看这篇别人家的高端文章，清晰易懂[比较清晰的讲git结构和版本回退的](https://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-%E9%87%8D%E7%BD%AE%E6%8F%AD%E5%AF%86#r_git_reset)
强烈推荐看一看，尤其是结合那些图：head，索引和工作区。
## 版本回退之 reset

### soft,默认参数与hard
1. **soft**
试想一个场景，你add了好多东西，而且commit了，但是push之前后悔了，不应该在master上提交，还是想撤销了这些commit，然后暂存一下，再搞其他的。那么怎么撤销提交呢。
就可以用soft参数了。soft仅仅是将HEAD回到你指定的地方，仅此而已，索引（暂存区）和工作区（你编辑文件的地方）都不变。
```
git reset --soft 目的版本号
```
执行此命令仅仅是撤销了commit，并没有撤销add，因为索引没变化。所以这时候你执行`git status`，看到的是add之后的状态，待提交。

2. **默认参数**
add并且提交了`commit1`与`commit2`的修改，此时二者被放到了本地仓库，且本地的head已经指向了这个提交。那么这时候只想提交一个，而不是提交两个，怎么办呢？
如下图：
![](https://res.cloudinary.com/saintasmita/image/upload/v1559368641/codexiongan/1_qc18gf.png)
做法：
`git log`找到你要回退的目的地，然后执行
```
git reset 你的目的地的版本号
```
这个命令就完成了两步：
1、将本地仓库HEAD指针回退一下；
2、索引（暂存区）的指向也回退一下，工作区的东西不动。
如此，就回到了仅仅修改没有add，没有提交的状态了。

`git status`查看如下图了，发现两个文件又回到了未add和commit的状态：
![](https://res.cloudinary.com/saintasmita/image/upload/v1559369356/codexiongan/2_giv9h1.png)

3. **hard**
如果你已经提交了，但是又想完全抛弃这些提交，撤销add，连工作区的那些更改也都撤销，这时候你需要
```
git reset --hard 目的地版本号
```
如果你不小心执行了这个，发现没必要回退，又想找回来已经提交的，那没问题。
```
git reset --hard 想找回的提交ID
```
即可，毕竟这些命令指示移动各种指针，又没有删除节点。
当然了，`--hard`**会覆盖掉你没提交的那些文件的，而且再也找不回来的。**
