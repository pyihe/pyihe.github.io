---
layout: post
title: 'supervisor启动进程时报错: unknown error making dispatchers for etcd: EACCES'
date: 2020-06-01
author: pyihe
tags: [问题记录, supervisor]
---

### 问题描述

通过supervisor启动etcd，配置文件配好后，start etcd时报错: unknown error making dispatchers for 'etcd': EACCES,截图如下：<br>
![](/assets/img/2020-05-31/error.jpg)

### 解决方法

网上搜了一下没有具体关于这个错误的解决办法，但是在搜索过程中，注意到了EACCES大概是目录权限的类的错误，于是查看配置文件，自己的配置文件目
录中涉及目录的参数有：command、directory、stdout_logfile、stderr_logfile，检查目录权限和分组时，发现日志所属用户为root，但是自己
supervisor配置文件的配置的user为非root用户，通过：sudo chown -R user:group log_dir， 更改所属用户组后，再次update，问题解决！<br>
![](/assets/img/2020-05-31/succ.jpg)

纪录一下，希望能帮到遇到同样问题的朋友！