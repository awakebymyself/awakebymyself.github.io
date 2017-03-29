---
layout: post
title:  "一些不常用的git命令备忘（经常忘记..）"
categories: Git
tags:  Git
author: Lzg
---

* content
{:toc}

关于gradle　publistToMavenLocal时候包的版本不对的问题
`git push mine master tags` | `git pull origin master`

**　在当前分支下面拿另外一个分支的commit
`git cherry-pick commit id` 可以是多个分支

** 修改commit log中某一个commit
`git rebase -i HEAD ~~~`这个代表修改最近三次的commit, 在打开的文件里面
找到你想要的commit, 将`pick` 修改为e（Edit） 或者其他等等
