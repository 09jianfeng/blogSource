title: 用usb远程连接越狱设备进行调试
date: 2015-04-5 17:29:51
categories: [iOS,逆向]
tags: [iOS,逆向]
---

### 移动端要做的操作：http://iphonedevwiki.net/index.php/SSH_Over_USB
itnl_rev8 文件下的文件拷贝一下，

放到 /Library/itunnel 如果没有这个目录的话就创建这个目录

修改/Library/LaunchDaemons/com.usbmux.itunnel.plist 这个plist文件，没有则创建之
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Label</key>
	<string>com.usbmux.itunnel</string>
	<key>ProgramArguments</key>
	<array>
		<string>/path/to/itnl</string>
		<string>--iport</string>
		<string>22</string>
		<string>--lport</string>
		<string>2222</string>
	</array>
	<key>RunAtLoad</key>
	<true/>
	<key>KeepAlive</key>
	<true/>
</dict>
</plist>
```

---
### pc端要做的操作：  http://iphonedevwiki.net/index.php/SSH_Over_USB

进入目录 `/Users/JFChen/Documents/ iosReserve/usbdebug`执行

```
./tcprelay.py -t 22:2222
```

然后用ssh连接

```
ssh root@localhost -p 2222
```

默认密码还是：
alpine

如果遇到这样的错误：

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```
去.ssh/known_hosts文件找到localhost的那条rsa，删掉即可