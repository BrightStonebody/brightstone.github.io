---
title: apt常用命令
date: 2019-05-02 19:49:35
tags: 
- Linux
categroies:
- Linux
---

# apt常用命令
参考：
[apt命令安装指定版本](https://www.centos.bz/2017/07/ubuntu-apt-cache-version-list-install-specify-version/)
## 更新
> sudo apt-get update     //更新源
sudo apt-get upgrade    //更新已安装的包
sudo apt-get dist-upgrade       //升级系统
sudo apt-get dselect-upgrade    //使用 dselect 升级

## 查询
> apt-cache search package      
> //搜索包　　
apt-cache show package          
//获取包的相关信息，如说明、大小、版本等　
apt-cache depends package       
//了解使用依赖
apt-cache rdepends package      
//是查看该包被哪些包依赖

```shell
apt-cache madison <package name>  //列出所有来源的版本...madison是一个apt-cache子命令，可以通过man apt-cache查询更多用法。

apt-cache policy <<package name>>  /*将列出所有来源的版本。信息会比上面详细一点*/

apt-show-versions -a <<package name>> //列举出所有版本，且能查看是否已经安装。还可以通过apt-show-versions -u <>来查询是否有升级版本。
```

## 安装
> sudo apt-get install package      
> //安装包　　
sudo apt-get install package - - reinstall      
//重新安装包　　
sudo apt-get -f install         
//修复安装   ("-f = ——fix-missing")

## 删除
sudo apt-get remove package 
删除包　　
sudo apt-get remove package - - purge 
删除包，包括删除配置文件等
sudo apt-get clean && sudo apt-get autoclean 
清理无用的包
sudo apt-get check 
检查是否有损坏的依赖

## 其他
> sudo apt-get build-dep package 
> 安装相关的编译环境
apt-get source package 
下载该包的源代码



