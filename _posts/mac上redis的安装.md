---
title: mac上redis的安装
date: 2016-06-05 22:34:10
tags: [php,数据库]
---

## 安装

1、[下载redis](http://redis.io/download)

2、修改文件夹名字，并且编译

```
mv redis-2.8.7 redis
cd redis/
sudo make
sudo make test
sudo make isntall
```

3、 配置数据库文件要保存哪里

```
你会在redis 目录里找到一个 redis.conf 的配置文件,打开编辑此配置文件,找到 dir  .  这一行配置.

此配置是将内存中的数据写入一个文件,这个数据库文件要保存到什么地方 就是这个配置项起到的作用.

我在mac根目录下创建了 opt 的文件夹(注意此文件夹必须有可读写权限)

所以这一行的配置是 dir  /opt/redis/

修改后保存配置文件,同时将配置文件移动到 /etc 目录下.
```

4、上面第三步 make install 成功后 (或者 sudo make test),你就应该在这个目录下看到redis

```
/usr/local/bin/redis-server
```

5、启动redis服务

```
/usr/local/bin/redis-server /etc/redis.conf
```

6、 服务器端已经顺利启动,下一步我们可以启动客户端来尝试连接一下redis服务端,同时写入数据测试一把

```
: /usr/local/bin master ⚡ $ ./redis-cli 
127.0.0.1:6379> get zhangzhi
(nil)
127.0.0.1:6379> set jianfeng nihao
OK
127.0.0.1:6379> get jianfeng
"nihao"
127.0.0.1:6379> append jiafeng !
(integer) 1
127.0.0.1:6379> 
```


## php 安装 phpredis扩展

1、 [下载 phpredis/phpredis](https://github.com/phpredis/phpredis)

2、 `cd phpredis-develop`

3、运行下面的命令

```
phpize
./configure [--enable-redis-igbinary]
make && make install
```
**`error`** 如果运行 phpize出现错误

```
Configuring for:
PHP Api Version:         20121113
Zend Module Api No:      20121212
Zend Extension Api No:   220121212
Cannot find autoconf. Please check your autoconf installation and the
$PHP_AUTOCONF environment variable. Then, rerun this script.
```

则需要修复一下

```
# cd /usr/src
# wget http://ftp.gnu.org/gnu/m4/m4-1.4.16.tar.gz
# sudo tar -zvxf m4-1.4.16.tar.gz
# cd m4-1.4.16/
# sudo ./configure && make && make install
# cd ../
# sudo wget http://ftp.gnu.org/gnu/autoconf/autoconf-2.62.tar.gz
# sudo tar -zvxf autoconf-2.62.tar.gz
# cd autoconf-2.62/
# sudo ./configure && make && make install
```

4、make install的 安装错误

```
Installing shared extensions:     /usr/lib/php/extensions/no-debug-non-zts-20121212/
cp: /usr/lib/php/extensions/no-debug-non-zts-20121212/#INST@46020#: Operation not permitted
make: *** [install-modules] Error 1
FAIL: 2
```

`修复方法：` 

[禁用 10.11 的sip](https://coolestguidesontheplanet.com/what-is-sip-in-osx-10-11-el-capitan/)

OSX 10.11 El Capitan ships with a new security process referred to as SIP or Security Integrity Protection also known as rootless.下面这几个目录是被保护的

```
/System
/sbin
/usr  (but not /usr/local).
```

**`禁用SIP in OSX`**

* 重启电脑，按住 'command' + 'r' 进入恢复模式。
* 运行终端模式，并且输入命令 `csrutil disable` 禁用 SIP
* `csrutil  enable`启用SIP
* `csrutil status`查看状态


### 配置php.ini文件
如果 /etc/ 目录下没有php.ini文件，则 复制一份

```
sudo cp php.ini.default php.ini
```
然后 `vim /etc/php.ini` 在文件的最后添加 `extension=redis.so`

运行 `php -m` 查看模块 有redis的话，就是配置成功了。
