---
layout: post
title: 'Mac通过homebrew安装supervisor，并添加process'
date: 2020-05-30
author: pyihe
tags: 工具
---

## 一. 安装supervisor

### 执行安装命令
```
brew search supervisor
brew install supervisor
```

### 安装后的配置
安装成功后，会有提示是否需要让supervisor随系统启动，需要的话执行命令：
```
brew services start supervisor
```
安装成功后，supervisor的安装目录为：/usr/local/Cellar/supervisor， supervisor配置文件supervisord.ini在/usr/local/etc下

**注：安装成功后执行supervisorctl时需要带参数, 否则会报错: "http://localhost:9001 refused connection"**
```
supervisorctl -c /usr/local/etc/supervisord.ini
```
解决办法是在配置文件中取消一行代码注释，如下图:<br>
![](/assets/img/2020-05-30/取消注释.jpg)
详细参考: [解决办法出处](https://askubuntu.com/questions/911994/supervisorctl-3-3-1-http-localhost9001-refused-connection)

配置完毕后即可通过supervisorctl管理process，执行supervisorctl可查看process列表以及运行状态:<br>
![](/assets/img/2020-05-30/查看process.jpg)

## 二. 添加process(如etcd)
1. 在/usr/local/etc目录下新建目录supervisor.d，用于存放所有process的配置文件，进入supervisor.d并新建etcd.ini:<br>
![](/assets/img/2020-05-30/etcd配置.jpg)
<br>关于etcd.ini中的每个配置项，在supervisor.ini中有说明，建议直接拷贝过来进行修改；
2. 配置文件编辑成功后，在supervisorctl中执行update，加载etcd.ini， 如果配置文件正确，etcd会直接running<br>
![](/assets/img/2020-05-30/list.jpg)