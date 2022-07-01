---
title: Git实用命令
date: 2022-07-01 16:00:00
categories: 
  - Git
tags:
  - 计算机技术
---


# Git实用命令

有一部分Git命令平时我们很少用到，但是实用性比较强，在此做记录。

## Git bisect

通过二分法帮助排查出问题的commit.

eg.:

假如某一天突然发现当前版本出了BUG，但是不知道是哪次commit引入的bug。上次正常的发布版本v1距离当前HEAD有100次提交。这个时候需要排查出有bug的那次提交。应该怎么做？

### normally

通过git log/show 依次检查出问题的提交，找到出问题的代码。

通过IDE自带的可视化工具，依次检查出问题的提交。

### bisect

为方便理解，明确下文中会用到的几个关键词：

- HEAD: 指向某次Commit的指针,默认情况下指向当前分支
- head: 在本文中我们定义小写的head指向我们发现BUG时,HEAD的指向


操作如下：

```git 

git bisect start HEAD HEAD~100 // 这里也可以使用commit hash代替

```

因为v1的版本的功能是正常的，所以引发bug的提交一定在head和V1之间。所以这条命令的意思是`将head和head~100设为我们要二分搜索的两个端点` 

> head是bad V1是good
> 也可以使用 git bisect start  & git bisect bad & git bisect good HEAD~100来代替上边的命令

git会在`端点`之间的100次提交中，选择中间的一次提交。

执行这行命令时，当前HEAD指向的应该是距离head的51的位置的提交，即`从v1之后的51次commit`

这个时候我们需要判断在当前commit状态下项目是否出现了BUG。

假设在head~51的位置，我们发现项目并没有出现BUG，则：

```bash

git bisect good

```

上边的命令代表当前的版本是一个“好”的版本，即没有bug。

随后,git会将HEAD指向head~26的位置，

如果测试中发现，当前HEAD版本是有BUG的，则输入：

```
git bisect bad

```

随后,git会将HEAD指向head~37(大概，瞎写的)的位置。

然后循环上边两个处理逻辑，直到最终得到出BUG的commit。



## 总结

git bisect命令会让git通过二分查找，帮助我们快速的找到出问题的bad commit。

一开始我们需要指定bad的版本(head)，和good的版本(v1)，从而创建一个需要测试的区间。

git会自动选择这个区间的中间提交并切换HEAD到该提交上并且会输出日志：

```
Bisecting: 12 revisions left to test after this (roughly 4 steps)
```

日志代表： 在此之后还有12个修订版本需要测试(大约4个步骤) 

日志内容根据bisect的基点和进行过程中动态会变化，上例仅参考。

 