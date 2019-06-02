---
title: mysql配置远程登录
date: 2019-05-04 11:04:44
tags:
- mysql
- Linux
categories:
- Linux
---

# mysql配置远程登录

#### 1、给用户远程登录权限：
```shell
grant all privileges  on *.* to root@'%' identified by "password";

FLUSH PRIVILEGES
```

root可以替换为其他用户名， %表示任意ip， 也可以换成指定的ip， password替换成密码

使用以下命令可以查看用户的登录权限
```shell
select host,user,password from user;
```

#### 2、 lnmp环境配置iptables

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

