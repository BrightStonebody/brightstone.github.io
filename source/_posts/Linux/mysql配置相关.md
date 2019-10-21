---
title: mysql配置相关
date: 2019-05-04 11:04:44
tags:
- mysql
- Linux
categories:
- Linux
---

## 1. 尽量使用MariaDB而不是mysql

之前有配置过mycat，刚开始使用的mysql，使用yum安装的，默认版本为8.0
但是安装之后mycat始终无法登陆，输入密码后总是显示密码错误。后来google了一下，发现是因为**mysql8.0的加密方式发生了编码，导致mycat无法正确识别密码**
于是尝试切换到5.7版本，但是使用yum安装无论如何都无法切换到我想要的版本。重装为MariaDB后没有这个问题了

而且，mysql的命令行终端很明显有bug，没有MariaDB好用

## 2. MariaDB更新密码
mariadb的安装可以查看官网，上面有yum安装的教程。
安装之后默认是没有密码的。需要更新密码。
另外，**发现mariadb在本机是可以不需要密码直接登陆的，不知道原因。但是远程登陆是一定要密码的**

中间出现过很奇怪的现象：一同不知道啥的操作之后，输入root密码总是显示密码错误。跳过密码直接回车可以登陆，但是use mysql;命令之后出现access deny ''@'localhost'之类的报错。没有找到原因和解决办法，最后是直接重置了vps

#### 1. mysql_secure_installation 命令
安装之后调用mysql_secure_installation 命令，进入到mariadb的初始化，可以设置密码

#### 2. 登陆mysql更改密码

```shell
# 2.1 更新 mysql 库中 user 表的字段：
use mysql;  
UPDATE user SET password=password('newpassword') WHERE user='root';  
flush privileges;  
exit;
 
# 2.2 或者，使用 set 指令设置root密码：在mariadb 10.4之后的版本只能使用该方法
SET password for 'root'@'localhost'=password('newpassword');  

```


## 3. mysql配置远程登录

#### 1. 给用户远程登录权限：
```sql
use mysql
update user set host='%' where user ='root';
FLUSH PRIVILEGES;
grant all privileges  on *.* to root@'%' identified by "password";
FLUSH PRIVILEGES;
```

root可以替换为其他用户名， %表示任意ip， 也可以换成指定的ip， password替换成密码

使用以下命令可以查看用户的登录权限

```shell
select host,user,password,plugin from user;
```

**注意**
root密码的加密方式最好为mysql_native_password，这样子使用中间件登陆才不会出问题。
可以使用如下命令更改加密方式：

```sql
update user set plugin='mysql_native_password' where user='root';
```

#### 2. lnmp环境配置iptables

lnmp一键安装环境默认是禁用iptables远程登录的，

* 查看iptables规则：
```shell
iptables -L -n --line-numbers
```
输入样例如下：
![图片](/images/mysql_iptables.png)
可以看到3306端口的target为drop

* 删除对应的drop规则

```shell
iptables -D INPUT 5
```


iptables的使用参考：
[https://www.vpser.net/security/linux-iptables.html]()

#### 3.非集成centos环境开放3306端口

##### centos-7以上

firewalld 防火墙（centos-7）运行命令,并重启：

```shell
firewall-cmd --zone=public --add-port=3306/tcp --permanent
firewall-cmd --reload
```

##### centos-7以前
iptables 防火墙（centos6.5及其以前）运行命令

```shell
vim /etc/sysconfig/iptables
```

在文件内添加下面命令行，然后重启

```shell
-A INPUT -p tcp -m state --state NEW -m tcp --dport 3306 -j ACCEPT
```

```shell
service iptables restart
```


