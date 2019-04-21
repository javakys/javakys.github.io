---
layout: post
title: How to clear git history on local git repository
date:   2019-04-21 
author: James Kim
categories: Git-Github
---

Remote Git Repository와의 Link가 깨진 Local Git Repository를 정리해서 Remote Repository와 연결을 생성하기 위한 과정.

### Local Git history를 제거한다 ###
```shell
$ rm -rf .git
```

### Local Git을 새롭게 만든다 ###
```shell
$ git init
$ git add .
$ git commit -m 'initial commit'
```

### Local Git을 Remote Git(on Github)에 Push한다. ###
```shell
$ git remote add origin <github 주소>
$ git push origin master
```