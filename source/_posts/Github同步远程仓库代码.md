---
title: Github同步远程仓库代码
tags:
  - Github
  - Git
abbrlink: a07a809d
date: 2019-01-22 10:56:18
---

> 最近提了自己第一个PR，总体也是磕磕绊绊。先记录下fork代码的更新吧。

**参考：**[同步一个 fork](https://gaohaoyang.github.io/2015/04/12/Syncing-a-fork/)

### Fork

第一步当然是先`fork`人家的项目了。

以Github上面的`hutool`为例,在人家的项目页面点右上角的`Fork`即可：

![fork](https://gsealy-1257917518.cos.ap-beijing.myqcloud.com/gsealy.github.io/github/fork.png)

然后克隆`fork`到自己仓库的项目到本地就可以了：

```bash
> git clone https://github.com/Gsealy/hutool.git
```

### 远程仓库有新的提交

当远程仓库有新的提交变动的时候，需要同步更新一下，要不然容易产生冲突。

**Configuring a remote for a fork**

- 给 fork 配置一个 remote
- 主要使用 `git remote -v` 查看远程状态

```bash
$ git remote -v
origin  https://github.com/Gsealy/hutool.git (fetch)
origin  https://github.com/Gsealy/hutool.git (push)
```

- 添加一个将被同步给 fork 远程的上游仓库

```bash
git remote add upstream https://github.com/looly/hutool.git
```

- 再次查看状态确认是否配置成功

```bash
$ git remote -v
origin  https://github.com/Gsealy/hutool.git (fetch)
origin  https://github.com/Gsealy/hutool.git (push)
upstream        https://github.com/looly/hutool.git (fetch)
upstream        https://github.com/looly/hutool.git (push)
```

**Syncing a fork**

- 从上游仓库 fetch 分支和提交点，传送到本地，并会被存储在一个本地分支 `upstream/v4-master `，(这块因为做过同步了，所以在网上找的，大概改了下)

```bash
$ git fetch upstream
# remote: Counting objects: 75, done.
# remote: Compressing objects: 100% (53/53), done.
# remote: Total 62 (delta 27), reused 44 (delta 9)
# Unpacking objects: 100% (62/62), done.
# From https://github.com/looly/hutool
#  * [new branch]      v4-master     -> upstream/v4-master
#  * [new branch]      v4-dev     -> upstream/v4-dev
```

1. 改变本地分支为`v4-master`

```bash
$ git checkout v4-master
Switched to branch 'v4-master'
Your branch is up to date with 'origin/v4-master'.
```

2. 合并`v4-master`代码

```bash
$ git merge upstream/v4-master
Updating 6a5e2e3d..84be6d34
Fast-forward
....
42 files changed, 396 insertions(+), 136 deletions(-)
```

3. 同步到自己的仓库

```bash
$ git push origin v4-master
Enumerating objects: 436, done.
Counting objects: 100% (311/311), done.
Delta compression using up to 8 threads.
Compressing objects: 100% (56/56), done.
Writing objects: 100% (169/169), 16.81 KiB | 1.29 MiB/s, done.
Total 169 (delta 67), reused 164 (delta 64)
remote: Resolving deltas: 100% (67/67), completed with 51 local objects.
To https://github.com/Gsealy/hutool.git
   6a5e2e3d..84be6d34  v4-master -> v4-master
```

**注：**其他分支按照1-3步从新做一遍就好了🔚

------

