---
title: 描述文件获取UDID
date: 2016-12-06 10:49:03
tags: [iOS]
---


## 一、创建网页快捷图标
### 步骤
首先给mac系统装一个工具。“iPhone 配置实用工具”。网上下载安装包：iPhoneConfigUtility.dmg。装好后在“Launchpad”里的“其它”里。运行它，并点击左上角的新建按钮创建一个配置描述文件。如下图设置：

设置好描述：
![](../img/udid1.png)

这里写图片描述

设置好桌面显示和跳转url
![](../img/udid2.png)

导出没有签名的证书：

![](../img/udid3.png)

导出的文件命名为：2.mobileconfig

这个文件放到网上，然后用手机safari打开，就可以给手机的桌面创建一个网页的快捷方式了。

由于是未签名的，影响用户使用，以下讲讲如何给他签名。

### 证书签名

由于要签名，所以需要去申请一个ssl的证书。证书都是要钱的。

不过刚好找到了这个https://www.startssl.com/网站，可以免费申请一个证书使用一年。

进去填个邮箱，然后收到验证码，激活一下。就可以登录进去了。

登录进去后，就是申请域名证书了。填好域名，再填一个域名的邮箱，收到验证码再激活一下。然后再进入这里。 
![](../img/udid4.png)

这里需要CSR串,因为软件是win版本的，那先用windows下载软件装好，再设置如下： 
![](../img/udid5.png)

就可以生成一个1.key文件和一串CSR。

把CSR串贴回到网站，再点击submit。很快就可以配置好一个域名的证书了。它会提示下载证书，也可以一会回到证书列表里自己下载。下载回来的证书文件为一个压缩包，压缩包里有四个包分别为：
```
ApacheServer.zip 
IISServer.zip 
NginxServer.zip 
OtherServer.zip
```
看名字也知道是什么。

这里我把ApacheServer.zip解压来用。里面有：
```
1_root_bundle.crt 
2_aaa.com.crt
```
然后先用命令把1_root_bundle.crt转成pem格式的先。
```
openssl x509 -in 1_root_bundle.crt -out 1_root_bundle.pem -outform PEM
```
结合上面得到的，总共有用的东西如下：
```
1.key 
1_root_bundle.pem 
2_aaa.com.crt 
2.mobileconfig
```
然后利用上面的证书对2.mobileconfig进入签名：
```
openssl smime -sign -in 2.mobileconfig -out 2signed.mobileconfig -signer 2_aaa.com.crt -inkey 1.key -certfile 1_root_bundle.pem -outform der -nodetach
```
最后生成的2signed.mobileconfig就是我们想要的证书了。

把这个证书放到网上，然后用手机的safari浏览器打开，提示安装的时候就会看到这个证书是绿色已签名的了。

## 二、利用safari取出手机的UDID值。
虽然从ios7开始，手机的udid就取不到了。但是苹果公司允许开发者通过IOS设备和Web服务器之间的某个操作，来获得IOS设备的UDID(包括其他的一些参数)。
也有人把这种工具做成应用放到appstore，并审核通过。

[UDID Sender](https://itunes.apple.com/cn/app/udid-sender/id306603975?mt=8)

### 步骤
- 1、在你的Web服务器上创建一个udid.mobileconfig的XML格式的描述文件；

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
    <dict>
        <key>PayloadContent</key>
        <dict>
            <key>URL</key>
            <string>http://192.168.0.215/web/uu.php</string>
            <key>DeviceAttributes</key>
            <array>
                <string>UDID</string>
                <string>IMEI</string>
                <string>ICCID</string>
                <string>VERSION</string>
                <string>PRODUCT</string>
            </array>
        </dict>
        <key>PayloadOrganization</key>
        <string>AppFree, Inc.</string>
        <key>PayloadDisplayName</key>
        <string>AppFree</string>
        <key>PayloadVersion</key>
        <integer>1</integer>
        <key>PayloadUUID</key>
        <string>9CF421B3-9853-4454-BC8A-982CBD3C907C</string>
        <key>PayloadIdentifier</key>
        <string>com.gpon.profile-service</string>
        <key>PayloadDescription</key>
        <string>This temporary profile will be used to find and display your current device's UDID.</string>
        <key>PayloadType</key>
        <string>Profile Service</string>
    </dict>
</plist>

```

注意：mobileconfig下载时设置文件内容类型Content Type为：application/x-apple-aspen-config(遇到问题的都是因为这个)

或者像这里用一个简单页面做好下载mobileconfig文件。就无问题。

```
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
 <meta content="width=device-width, initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0,user-scalable=no" name="viewport" id="viewport" />
<title>获取您的UDID</title>
<body>
<div id="content">

UUDI:<input style="" name="" value="<?php echo $_GET['udid']?>" /> 

<a class="buttons" href="udid.mobileconfig">1.点击获取您的UDID</a>

</div>
</body>
</html>
```
- 2、引导用户安装证书如下：

![](../img/udid6.png)

- 3、服务器接收数据的URL文件；

```
uu.PHP

<?php
$data = file_get_contents('php://input');
//file_put_contents("data.txt", $data); 

header('HTTP/1.1 301 Moved Permanently');  //这里一定要301跳转,否则设备安装会提示"无效的描述文件"
header("Location: http://192.168.0.215/web/index.php?udid=".rawurlencode($data));
?>

```

再附一份解析这个data取出UDID值的php

```
<?
$data = file_get_contents('php://input');
 
$plistBegin   = '<?xml version="1.0"';
$plistEnd   = '</plist>';
 
$pos1 = strpos($data, $plistBegin);
$pos2 = strpos($data, $plistEnd);
 
$data2 = substr ($data,$pos1,$pos2-$pos1);
 
$xml = xml_parser_create();
 
xml_parse_into_struct($xml, $data2, $vs);
xml_parser_free($xml);
 
$UDID = "";
$CHALLENGE = "";
$DEVICE_NAME = "";
$DEVICE_PRODUCT = "";
$DEVICE_VERSION = "";
$iterator = 0;
 
$arrayCleaned = array();
 
foreach($vs as $v){
 
    if($v['level'] == 3 && $v['type'] == 'complete'){
    $arrayCleaned[]= $v;
    }
$iterator++;
}
 
$data = "";
 
$iterator = 0;
 
foreach($arrayCleaned as $elem){
                $data .= "\n==".$elem['tag']." -> ".$elem['value']."<br/>";
                switch ($elem['value']) {
                    case "CHALLENGE":
                        $CHALLENGE = $arrayCleaned[$iterator+1]['value'];
                        break;
                    case "DEVICE_NAME":
                        $DEVICE_NAME = $arrayCleaned[$iterator+1]['value'];
                        break;
                    case "PRODUCT":
                        $DEVICE_PRODUCT = $arrayCleaned[$iterator+1]['value'];
                        break;
                    case "UDID":
                        $UDID = $arrayCleaned[$iterator+1]['value'];
                        break;
                    case "VERSION":
                        $DEVICE_VERSION = $arrayCleaned[$iterator+1]['value'];
                        break;                       
                    }
                    $iterator++;
}
 
$params = "UDID=".$UDID."&CHALLENGE=".$CHALLENGE."&DEVICE_NAME=".$DEVICE_NAME."&DEVICE_PRODUCT=".$DEVICE_PRODUCT."&DEVICE_VERSION=".$DEVICE_VERSION;

// enrollment is a directory
header('Location: http://192.168.0.215/web?'.rawurlencode($params));
?>
```

值得注意的是重定向一定要使用301重定向,有些重定向默认是302重定向,这样就会导致安装失败,设备安装会提示"无效的描述文件


- 4、当用户设备完成数据的手机后，返回提示给客户端用户；下面是苹果传来的数据的例子

```
    <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
    <plist version="1.0">
      <dict>
        <key>IMEI</key>
        <string>12 123456 123456 7</string>
        <key>PRODUCT</key>
        <string>iPhone8,1</string>
        <key>UDID</key>
        <string>b59769e6c28b73b1195009d4b21cXXXXXXXXXXXX</string>
        <key>VERSION</key>
        <string>15B206</string>
      </dict>
    </plist>
```

- 5，签名也像上面那样对它进行签名就行

```
openssl smime -sign -in udid.mobileconfig -out udidsigned.mobileconfig -signer 2_aaa.com.crt -inkey 1.key -certfile 1_root_bundle.pem -outform der -nodetach
```