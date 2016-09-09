---
title: openssl使用简介
date: 2016-08-29 11:08:50
tags: [加密,计算机基础]
---

参考：
<https://www.openssl.org/docs/manmaster/>

## SSL
Secure Sockets Layer,现在应该叫"TLS",但由于习惯问题,我们还是叫"SSL"比较多.http协议默认情况下是不加密内容的,这样就很可能在内容传播的时候被别人监听到,对于安全性要求较高的场合,必须要加密,https就是带加密的http协议,而https的加密是基于SSL的,它执行的是一个比较下层的加密,也就是说,在加密前,你的服务器程序在干嘛,加密后也一样在干嘛,不用动,这个加密对用户和开发者来说都是透明的。

简单地说,OpenSSL是SSL的一个实现,SSL只是一种规范.理论上来说,SSL这种规范是安全的,目前的技术水平很难破解,但SSL的实现就可能有些漏洞,如著名的"心脏出血".OpenSSL还提供了一大堆强大的工具软件。

## openssl 用例

openssl是一个强大的加密、解密工具，由开源组织维护。利用openssl工具，我们可以实现一些常见的摘要算法。

示例：

```
# base64
echo -n "phpgao" | openssl base64
# 解密 -d == decrypt
echo "cGhwZ2Fv" | openssl base64 -d
# 对一个文件计算摘要，下同
openssl base64 -in nginx.conf
# md5
echo -n "phpgao" | openssl md5
# sha128
echo -n "phpgao" | openssl sha1
```

```
# 对称加密
# 使用rc4算法加密php字符串，使用密钥phpgao，输出使用base64编码
echo -n "php" | openssl rc4 -k phpgao -base64

# 使用rc4算法解密字符串，使用密钥phpgao，输入使用base64编码
echo U2FsdGVkX18f3qEoEhVf+hsNOg== | openssl rc4 -d -k phpgao -base64
```



```
# rsa非对称加密 生成私钥

# genrsa 指使用rsa算法生成密钥文件
# -des3 指的是给私钥加密的算法(可选)
openssl genrsa -des3 -out key_rsa 4096
openssl genrsa -out key_rsa 4096
# 移除已经存在的密码(先做一下备份)
cp key_rsa old.key
openssl rsa -in old.key -out key_rsa

# 根据刚才创建的私钥创建公钥
openssl rsa -in key_rsa -pubout -out key_rsa.pub

# 使用公钥加密数据
# -in 输入文件
echo -n "phpgao" | openssl rsautl -encrypt -pubin -inkey key_rsa.pub -out a.txt

# 使用私钥解密数据
openssl rsautl -decrypt -inkey key_rsa -in a.txt

# 查看密钥信息
openssl rsa -in key_rsa -text -noout
openssl rsa -pubin -in key_rsa.pub -text -noout

# 检查私钥
openssl rsa -check -in key_rsa
# 检查一个CSR文件(后面有讲)
openssl req -text -noout -verify -in domain.csr
# 检查证书信息
openssl x509 -text -noout -in domain.crt

```


```
# 编码转换
# PEM转为DER
openssl x509 -in cert.crt -outform der -out cert.der

# DER转为PEM
openssl x509 -in cert.crt -inform der -outform pem -out cert.pem

```

## CA证书

### 原理

证书与密钥不同，CA证书指的是权威机构给我们颁发的证书，我们的电脑或者电子设备中内置了根证书，我们的电脑信任根证书。

```
   +-----------------+                    +--------------------+
   |                 |                    |                    |
   |                 |                    |                    |
   |                 |                    |                    |
   |   browser/PC    |  ① GET / HTTP/1.0 |                    |
   |                 |-------------------->  sites over https  |
   |                 ① WAIT! Where is CA ?|                   |
   |                 +-------------------->                    |
   |                 |  ④ Anyway,give me |                    |
   +-----+----^------+   the code!        +--------------------+
         |    |
         |    |
         |    |③ He is good!
② Is's it|    |
a bad site?   |
         |    |    +----------------------+
         |    |    |                      |
         |    +----+                      |
         |         |                      |
         |         |         CA           |
         +--------->                      |
                   |                      |
                   |                      |
                   +----------------------+

```

1、 网站生成自己的公钥

2、CA机构用自己的私钥加密网站的公钥以及相关信息。

3、客户端信任CA，并拥有CA的公钥，客户端就可以使用公钥解密加密后的证书，并从证书中得到网站的公钥。

4、如果能用CA的公钥解密出数据，说明网站的证书是经过CA认证的，客户端就可以放心访问网站了，如果系统发现证书不是权威CA机构颁发的，会警告用户。

5、客户端使用网站的公钥加解密数据，然后进行信息交换。


### 证书格式
 
* PEM (Privacy Enhanced Mail) 格式

通常常用于数字证书认证机构（Certificate Authorities，CA），扩展名为.pem, .crt, .cer, and .key。内容为Base64编码的ASCII码文件，有类似"-----BEGIN CERTIFICATE-----" 和 "-----END CERTIFICATE-----"的头尾标记。服务器认证证书，中级认证证书和私钥都可以储存为PEM格式（认证证书其实就是公钥）。Apache和nginx等类似的服务器使用PEM格式证书。


* DER (Distinguished Encoding Rules) 格式

与PEM不同之处在于其使用二进制而不是Base64编码的ASCII。扩展名为.der，但也经常使用.cer用作扩展名，所有类型的认证证书和私钥都可以存储为DER格式。Java是其典型使用平台。

### CSR (Certificate Signing Request)
它是网站向CA机构申请数字身份证书时使用的请求文件，最常见的格式是PKCS。在生成请求文件前，网站需要准备一对非对称密钥。私钥信息自己保存，请求中会附上公钥信息以及国家，城市，域名，Email等信息，csr中还会附上签名信息。当网站准备好CSR文件后就可以提交给CA机构，等待他们给我们签名，签好名后我们会收到crt文件，即证书。


生成CSR文件，开始签名,然后去 CA机构请求 CRT文件

```
openssl ca -h

# 先生成一个CA根证书
openssl req -x509 -nodes -days 36500 -newkey rsa:2048 -keyout key_ca_rsa -out ca.crt

# 使用CA的证书签名 密钥，CSR文件，生成 domain.crt文件
openssl ca -policy policy_anything -days 365 -cert ca.crt -keyfile key_rsa -in domain.csr -out domain.crt

# 可能会出错，因为缺少一些必要文件
mkdir -p demoCA/newCerts
touch demoCA/index.txt 
echo -n "01" > demoCA/serial
```

或者自己直接生成CRT文件。

```
# 生成证书和私钥
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout key_rsa -out key.crt
# 从已存在的私钥生成生成证书
openssl req -x509 -new -days 365 -key domain.key -out domain.crt


# 检查证书信息
openssl x509 -text -noout -in domain.crt
```

