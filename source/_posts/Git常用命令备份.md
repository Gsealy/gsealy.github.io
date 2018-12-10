---
title: Git常用命令备份
date: 2018-12-10 11:40:57
tags:
- Git
---

**备份一下日常用命令**

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