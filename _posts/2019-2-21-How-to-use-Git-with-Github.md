---
layout: post
title: How to use git with github.com
date:   2019-02-21 
author: James Kim
categories: Git-Github
---

## Remote branch를 local branch로 가져오기 ##

```shell
$ git checkout -b <local branch name> origin/<remote branch name>
```

## local에서 branch 변경하기 ##

```shell
$ git checkout <new branch name>
```

## local brance 삭제하기 ##

```shell
$ git checkout <a different branch>
$ git branch -D <a branch name to delete>
```