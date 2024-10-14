---
title: Git 内部工作原理
tags:
- linux
date: 2021-05-10
draft: true
---

首先大家都知道 git 工作依靠项目根目录下 .git/ 目录中的文件：

概念 working tree, index, repo

# Git Object
> [很好的一个博客，部分资料来源](https://www.yiibai.com/git)
来源: [这才是真正的Git——Git内部原理揭秘！](https://zhuanlan.zhihu.com/p/96631135)
[git可视化网站](http://onlywei.github.io/explain-git-with-d3)

<iframe src="/images/git.mp4" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true" width="560" height="315" style="border:none;overflow:hidden;display:block;margin:0 auto;"></iframe>

1. 修改/新增文件: 只影响工作区
2. `git add`: 将该文件全量内容复制一份压缩成blob对象，暂存区更新索引指向这个blob对象
3. `git commit`: 根据索引指向的所有blob生成tree对象，以及指向tree对象的commit对象；移动分支指针指向这个commit对象

git object逻辑对应关系:
```
.git
|- HEAD
   |- refs/heads/<branch>
      |- objects/<commit>
         |- objects/<tree>
            |- objects/<blob>
            |- ...
```

## 和分支 tag的关系
1. `cat .git/HEAD`: 查看HEAD指向的分支，例:
```
ref: refs/heads/master
```
2. `cat refs/heads/master`: 查看该分支指向的commit
```
c78737c99264ac38b0b3a978049f327a866b355f
```

## blob
> 储存文件的内容
**注意**: 根据文件内容的hash值创建，若新增一个内容一样（即使文件名不一样）的文件，`git add`后不会创建新blob，`git commit`后会在tree中新增一条记录（包含新文件名等信息）指向同一个blob

例: b7b1aea3e468d4b3f9044232d16f739c5d99c053
```
# hello-github
The first step to GitHub!
try 1
```

## commit
> 储存上一个commit、对应的快照、作者、提交信息

例: c78737c99264ac38b0b3a978049f327a866b355f
```
tree bfd97ce5b0880b7e3d4e05c63fe5e9ddede93798
parent 563a88bbe4b98666e317da6e42838152ac461ccd
author zjm <1344584929@qq.com> 1620444537 +0800
committer zjm <1344584929@qq.com> 1620444537 +0800

1st commit
```
## tree
> 储存目录结构，每个目录项包括文件权限、文件名、索引（类似文件系统的目录文件）

例: bfd97ce5b0880b7e3d4e05c63fe5e9ddede9379
```
040000 tree 6b82bffd686b1b7f13bc5afb141627f9b58b63cf    .ipynb_checkpoints
100644 blob b7b1aea3e468d4b3f9044232d16f739c5d99c053    README.md
```
    
## tag TODO


# 理解 Git 命令

![](/images/git_stash.jpg)

