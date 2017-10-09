---
title: iOS逆向笔记
date: 2017-10-09 10:54:02
tags:
---

#iOS 逆向开发的一些记录


####  通过usb接口ssh登陆到ios设备
不知道为什么10.1越狱后，无法通过原ip地址ssh登陆。
```
brew install usbmuxd
iproxy 4567 22
```


### theos 安装
参考[https://github.com/theos/theos/wiki/Installation](https://github.com/theos/theos/wiki/Installation)

### theos编译，打包错误

```
dpkg-deb: error: obsolete compression type 'lzma'; use xz instead

Type dpkg-deb --help for help about manipulating *.deb files;
Type dpkg --help for help about installing and deinstalling packages.
make: *** [internal-package] Error 2
```

参考https://github.com/theos/theos/issues/211
解决办法：

```

I also tried replacing dm.pl from the link and adding lzma support - but still get the "use xz instead" error:
(change in line 148)

sub compressed_fd {
	my $sref = shift;
	return IO::Compress::Gzip->new($sref, -Level => 9) if $::compression eq "gzip";
	return IO::Compress::Bzip2->new($sref) if $::compression eq "bzip2";
	return "|lzma -9c" if $::compression eq "lzma";
	open my $fh, ">", $sref;
	return $fh;
}

sub compressed_filename {
	my $fn = shift;
	my $suffix = "";
	$suffix = ".gz" if $::compression eq "gzip";
	$suffix = ".bz2" if $::compression eq "bzip2";
	$suffix = ".lzma" if $::compression eq "lzma";
	return $fn.$suffix;
}

Edit:
I've managed to switch to gzip (not tested yet - packaged but not yet installed on device)
In theos_path/makefiles/package/deb.mk

#_THEOS_PLATFORM_DPKG_DEB_COMPRESSION ?= lzma
_THEOS_PLATFORM_DPKG_DEB_COMPRESSION ?= gzip
```


打包好安装的时候，可以使用命令行直接安装，需要配置Makefile文件
```
THEOS_DEVICE_IP=127.0.0.1
THEOS_DEVICE_PORT=4567

export THEOS_DEVICE_IP=127.0.0.1;export THEOS_DEVICE_PORT=4567
```




### theos log读取
不知道为什么iOS10上没有用生成/var/log/syslog
使用命令
```
 socat - UNIX-CONNECT:/var/run/lockdown/syslog.sock
```
也没有对应的log输出，只有一些系统的log

方法一 使用idevicesyslog ，目前我使用这中
```
	brew reinstall --HEAD libimobiledevice  //ios10上好行要用head的
	idevicesyslog | grep "iOSREE"
```
方法二 使用系统的console.app
方法三 使用Apple Configurator 2
在 Mac App Store 中，下载 Apple Configurator 2，双击对应的设备，在左侧边栏中选择 Console，在问题重现后，点击 save 保存所有输出。


### 砸壳
砸壳，最好在TargetApp所在Documents目录下进行，否则可能因为沙盒的原因而失败
第一步，用 Cycript找出 TargetApp的Documents目录路径。StoreApp 的Documents目录位于/
```
[[NSFileManager defaultManager] URLsForDirectory:NSDocumentDirectory inDomains:NSUserDomainMask][0]
```
第二步，砸壳 ，会在当前目录下生成一个TargetApp.decrypted的文件。
```
DYLD_INSERT_LIBRARIES=~/dumpdecrypted.dylib /path/to/executable
```



###### 错误
```
crypted.dylib /var/containers/Bundle/Application/AA9AAEEC-4B71-4DB2-93B7-F744D538909B/Unread.app/Unread
dyld: could not load inserted library 'dumpdecrypted.dylib' because no suitable image found.  Did find:
	dumpdecrypted.dylib: required code signature missing for 'dumpdecrypted.dylib'


Abort trap: 6
```

##### 解决
需要给dumpdecrypted.dylib 签名
在mac上执行

```
export CODESIGN_ALLOCATE="/Applications/Xcode.app/Contents/Developer/Toolchains/ XcodeDefault.xctoolchain/usr/bin/codesign_allocate" //这个文件不一定是这个路径
codesign --force --sign "iPhone Developer: feng shen" dumpdecrypted.dylib
```

使用签名的dumpdecrypted.dylib重新砸壳


#### 使用lldb，debugserver
在iOS上执行

```
debugserver -x backboard *:1234 /Applications/MobileSMS.app/ MobileSMS
```
或

```
debugserver *:1234 -a "WeChat"
```
```
 debugserver *:1234 main2_64
```


在Mac 上执行

```
lldb
process connect connect://192.168.0.23:1234
```

打印alsr

```
image list -o -f
```





#### 使用Reveal查看App的UI

在Cydia中安装Reveal Loader
重启targetApp，在Mac的Reveal上可以看到这个这个app。

如果安装之后，在reveal看不到targetAPP。检查iOS设备上/Library/目录下是否有一个名为RHRevealLoader的目录
	
	1. 若没有则创建该目录：mkdir /Library/RHRevealLoader
 	2. 在Mac上启动Reveal并选择Help → Show Reveal Library in Finder，
 	3. 这将会打开Finder窗口，并显示一个名为iOS-Libraries的文件夹
 	4. 将该目录下的libReveal.dylib通过scp或者iFunBox上传到刚才的手机目录

 #### 寄存器
 
 po $x0             //打印self
 x2 等是参数
 lr 调用该函数的入口
 
 ```
(lldb) po $x0
<DTUIEntry: 0x174095130>

(lldb) po $x2
NSConcreteNotification 0x17084fdb0 {name = DTSignOutNotify; userInfo = {
    DTSignOutAccordKey = 0;
    DTSignOutReasonKey = "\U4f60\U5f53\U524d\U767b\U5f55\U7684\U975e\U5b98\U65b9\U5ba2\U6237\U7aef\Uff0c\U5df2\U7981\U6b62\U8be5\U5ba2\U6237\U7aef\U7684\U767b\U5f55\U4f7f\U7528\U3002\U8bf7\U4e0b\U8f7d\U5e76\U4f7f\U7528\U5b98\U65b9\U5ba2\U6237\U7aef\U3002";
}}

(lldb) p/x $lr
(unsigned long) $14 = 0x000000018db115ec

 ```
 


