---
title: git自动补全
date: 2016-07-11 15:31:26
categories:
tags: [git]
---

为 git 添加自动补全可以为平时的开发工作提升很多效率。而其方法也非常容易：

1. 下载 git-completion 脚本
```sh
wget https://raw.githubusercontent.com/git/git/master/contrib/completion/git-completion.bash
```
2. 配置自动加载以上脚本
若期望整个系统生效，则在 /etc/profile 中（若为单个用户，则在 ~/.bashrc 中）添加:
```sh
source {git-completion.bash}
```