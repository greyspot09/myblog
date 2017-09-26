---
title: Advanced Apple Debugging & Reverse Engineering 笔记1-3章节笔记
date: 2017-09-26 23:17:30
tags: lldb,逆向
---


## 第一章


### Rootless
MacOS的默认设置会阻止调试系统app。这个特性被称为*System Integrity Protection*，也叫*Rootless*.
比如：

```
    lldb -n Finder
    error: attach failed: cannot attach to process due to System Integrity Protection
```


调试MacOS的系统App，需要先使用下面的方法禁用*Rootless*。

```
    1.重启MacOS电脑
    2.进入黑屏之后，按CMD+R，进入Recovery Mode。
    3.找到Utilities，打开Terminal
    4.在Terminal中输入
        `csrutil disable; reboot`
    5.电脑重启之后将会禁用Rootless。
```



#### tty的例子

- 1.打开Terminal 在tab1输入`tty`。会打印出类似`/dev/ttys027`的内容。
- 2.在terminal中重新打开一个tab2，输入`echo "hello debugger" 1>/dev/ttys027`。会在tab1中输出"hello debugger"

#### 调试Xcode
在终端输入

```
lldb
(lldb) file /Applications/Xcode.app/Contents/MacOS/Xcode
(lldb) process launch -e /dev/ttys027 --  #输出到/dev/ttys027中
```

Objective-C中的NSLog 和Swift中的 print 函数，都是输出到stderr。


#### 查找点击事件的类
```
(lldb) breakpoint set -n "-[NSView hitTest:]"
Breakpoint 1: where = AppKit`-[NSView hitTest:], address = 0x000000010338277b
```






## 第二章 Help & Apropos
### help命令
```
(lldb) help
(lldb) help breakpoint
(lldb) help breakpoint name
```
###The "apropos" command
```
(lldb) apropos swift
(lldb) apropos "reference count"
```

## 第三章 Attaching with LLDB

### lldb附加到一个已经存在进程,可以使用下面两种方法：

```
    1. lldb -n Xcode
    2. lldb -p `pgrep -x Xcode`
```

### lldb附加到一个未来的进程
方法1:
```
lldb -n Finder -w  #Finder下次启动的时候，lldb会附加到Finder的进程上
```

方法2:
```
lldb -f /System/Library/CoreServices/Finder.app/Contents/MacOS/Finder
(lldb) process launch
```

### 启动参数



```
lldb -f /bin/ls
输出
(lldb) target create "/bin/ls" 
Current executable set to '/bin/ls' (x86_64).
```

lldb创建一个target

```
(lldb) process launch
输出
Process 7681 launched: '/bin/ls' (x86_64) 
... # Omitted directory listing output
Process 7681 exited with status = 0 (0x00000000)
```

上面的命令会启动target目标程序。

```
(lldb) process launch -w /Applications
等价于

    $ cd /Applications 
    $ ls
```

```
(lldb) process launch -- /Applications
等价于
    ls /Applications
```

注意`process launch -w`和`process launch --`的区别，一个相当于*当前工作区目录地址*



```
(lldb) process launch -- ~/Desktop
Process 8103 launched: '/bin/ls' (x86_64) 
ls: ~/Desktop: No such file or directory
Process 8103 exited with status = 1 (0x00000001)

(lldb) process launch -X true -- ~/Desktop 
没有报错
```

注意`-X`的用处,可以扩展shell的参数。

```
(lldb) run ~/Desktop  //'run' is an abbreviation for 'process launch -X true --'
```

```
(lldb) process launch -o /tmp/ls_output.txt -- /Applications
Process 15194 launched: '/bin/ls' (x86_64) 
Process 15194 exited with status = 0 (0x00000000)
```

ls 输出的结果在ls_output.txt中。
使用`-o`参数重定向stdin的输出到文本中


```
(lldb) target delete
(lldb) target create /usr/bin/wc
(lldb) process launch -i /tmp/wc_input.txt
(lldb) run
(lldb) process launch -n
```
使用`-i`参数重定向stdin的输出到文本中


