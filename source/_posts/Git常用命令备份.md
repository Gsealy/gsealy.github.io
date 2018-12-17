---
title: Git常用命令备份
date: 2018-12-10 11:40:57
tags:
- Git
---

**备份一下日常用命令**

> **引用**
>
> [git代码统计-segmentfault](https://segmentfault.com/a/1190000008542123)

所有在`Windows`下用[cmder](http://cmder.net/)都可以执行

1. 查看提交次数：

```bash
git shortlog --numbered --summary
55  Gsealy
 3  JEDI
```
2. 删除本地提交

```bash
git reset --soft HEAD~1
```

3. 查看git上的个人代码量（修改`username`）

```bash
git log --author="username" --pretty=tformat: --numstat | awk '{ add += $1; subs += $2; loc += $1 - $2 } END { printf "added lines: %s, removed lines: %s, total lines: %s\n", add, subs, loc }' -
```

4. 统计每个人增删行数

```bash
git log --format='%aN' | sort -u | while read name; do echo -en "$name\t"; git log --author="$name" --pretty=tformat: --numstat | awk '{ add += $1; subs += $2; loc += $1 - $2 } END { printf "added lines: %s, removed lines: %s, total lines: %s\n", add, subs, loc }' -; done
```

5. 添加或修改的代码行数

```bash
git log --stat|perl -ne 'END { print $c } $c += $1 if /(\d+) insertions/'
```

