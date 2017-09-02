---
title: mac os开启php+Apache+MySQL环境搭建
date: 2017-05-22 10:26:33
tags: [后端]
---

# mac os 10.12 配置apache php mysql

## 启动apache
用命令启动/停止/重启

```
sudo apachectl start/stop/restart
```

查看 apache版本

```
httpd -v
```

## apache目录

Apache对应的根目录以及网址是

```
/Library/WebServer/Documents/
http://localhost
```

Apache对应的配置文件在目录 `/etc/apache2/httpd.conf` 这里。包括开启php等等。

### 配置用户目录

在 `~` 目录下创建Sites目录，用户级别的网站都放到这个目录下

* 用户对应的目录是（注意这个需要后面的一些配置才能生效）

```
~/Sites
http://localhost/~username/
```

* 创建用户配置conf文件

去目录 `/etc/apache2/users/` 下创建 username.conf文件

```
<Directory "/Users/JFChen/Sites/">
    Options Indexes MultiViews
    AllowOverride All
    Order allow,deny
    Allow from all
</Directory>
```

文件保存后，赋予权限 `sudo chmod 755`

* 修改 Apache 配置文件 （升级了mac os系统后如果这个配置失效了，记得重新更改一次）

打开httpd.conf文件
```
sudo nano /etc/apache2/httpd.conf
```

把下面几行前面的`#`去掉开启。

```
LoadModule authz_core_module libexec/apache2/mod_authz_core.so
LoadModule authz_host_module libexec/apache2/mod_authz_host.so
LoadModule userdir_module libexec/apache2/mod_userdir.so
Include /private/etc/apache2/extra/httpd-userdir.conf
```

下一步：

```
sudo nano /etc/apache2/extra/httpd-userdir.conf
```

开启：

```
Include /private/etc/apache2/users/*.conf
```

再下一步：

```
nano /etc/apache2/httpd.conf
```

查找并修改
```
<Directory />
	AllowOverride none
	Require All Denied
```

为

```
<Directory />
    AllowOverride none
    Require all granted
</Directory>
```


然后重启Apache，在浏览器输入 localhost/~username 就可以访问到用户目录了。

## 启动php

```
sudo nano /etc/apache2/httpd.conf
```

```
LoadModule php5_module libexec/apache2/libphp5.so
```

## 启动mysql

1、去mysql官网下载dmg,安装。

2、在~/.bash_profile 设置环境变量 ，设置后记得执行 `source ~/.bash_profile`生效

```
export PATH="/usr/local/mysql/bin:$PATH"
```

3、如果在一开始运行mysql就出现 `access denied for user`
需要更改密码。

```
1、先stop 掉mysql
2、然后 sudo mysqld_safe --skip-grant-tables
3、然后打开另外一个终端

$ mysql
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 13
Server version: 5.7.10 MySQL Community Server (GPL)

mysql> use mysql;
Database changed

mysql> UPDATE mysql.user SET authentication_string=PASSWORD('MyNewPass') WHERE User='root'; 
ERROR 1054 (42S22): Unknown column 'password' in 'field list'

mysql> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)

mysql> \q
Bye

```


4、退出安全模式。 执行命令 `mysql -u root -p` 然后输入密码。就可以正常使用了


## 附录
### nano 命令行

```
Ctrl+w 搜索，然后直接编辑
Ctrl+x 退出（如果有修改文件，会弹出Yes or No 保存文件，输入Y，然后会弹出文件名，不需要更改文件名的话，按下enter就行）
```