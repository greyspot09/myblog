---
title: Advanced Apple Debugging & Reverse Engineering 笔记1-4章节笔记
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

## 第四章 Stopping in Code

### Unix Signals介绍
Unix Signals是进程的一种通知机制。进程的停止/恢复主要以来两哥Signal，*SIGSTOP*和*SIGCONT*.

- SIGSTOP  保存进程的状态，暂停进程运行。
- SIGCONT  恢复进程运行。

Xcode中的暂停按钮（如图1），就是使用了Signals机制来实现。
![(图1)](http://onkcruzxc.bkt.clouddn.com/1506658435.png )

### Xcode breakpoints

- Symbolic Breakpoint... 
- Swift Error Breakpoint
- Exception Breakpoint. 

### LLDB breakpoint syntax

*image lookup*命令导出符号地址，参数*-n*查找符号/函数的名字,*-r*使用正则查找。
```
(lldb) image lookup -n "-[UIViewController viewDidLoad]"
(lldb) image lookup -rn test
```

#### Objective-C 属性

下面的代码
``` Objective-C
@interface TestClass : NSObject 
@property (nonatomic, strong) NSString *name; 
@end
```

会自动生成下面两个方法：
```
-[TestClass name]
-[TestClass setName:]
```

我们可以在lldb中使用`image lookup -n "-[TestClass name]"`的命令打印出相关信息。
会打印出类似

    1 match found in /Users/derekselander/Library/Developer/Xcode/ DerivedData/Signals-bqrjxlceauwfuihjesxmgfodimef/Build/Products/Debugiphonesimulator/Signals.app/Signals:
    Address: Signals[0x0000000100001470] (Signals.__TEXT.__text + 0) 
    Summary: Signals`-[TestClass name] at TestClass.h:28

#### Swift 属性

swift的属性会生成类似下面的代码:
```
ModuleName.Classname.PropertyName.(getter|setter)
```

比如:
```
class SwiftTestClass: NSObject { 
    var name: String!
}
```

在lldb中执行

```
image lookup -rn Signals.SwiftTestClass.name.setter
```

打印出

```
2 matches found in /Users/liucien/Library/Developer/Xcode/DerivedData/Signals-dkwmrjaubwzkugblocsacqvjqdfl/Build/Products/Debug-iphonesimulator/Signals.app/Signals:
        Address: Signals[0x000000010000cb00] (Signals.__TEXT.__text + 44608)
        Summary: Signals`@objc Signals.SwiftTestClass.name.setter : Swift.ImplicitlyUnwrappedOptional<Swift.String> at SwiftTestClass.swift        Address: Signals[0x000000010000cbc0] (Signals.__TEXT.__text + 44800)
        Summary: Signals`Signals.SwiftTestClass.name.setter : Swift.ImplicitlyUnwrappedOptional<Swift.String> at SwiftTestClass.swift:28

```

找到了两个符号。一个是为Objective-C生成的，带‘@objc’。猜测在Swift4中可能会只有一个符号。Swift4中，NSObject的子类，不会再自动生成Objective-C的方法。


#### 创建断点

##### 普通断点：
```
(lldb) b -[UIViewController viewDidLoad]
```

##### 使用正则：
```
(lldb) b Signals.SwiftTestClass.name.getter : Swift.ImplicitlyUnwrappedOptional<Swift.String>
```

可以简化成

```
(lldb) rb SwiftTestClass.name.setter
```

甚至简化成

```
(lldb) rb name\.setter
```

如果要在每个UIViewController的Instance method上打断点，可以使用

```
(lldb) rb '\-\[UIViewController\ '
```

但是上面这个命令会略过 Objective-C categories，这类函数的签名类似`(-|+)[ClassName(categoryName) method]` 这种形式

改进
```
(lldb) rb '\-\[UIViewController(\(\w+\))?\ '
```

##### 限制范围
使用*-f*，可以给目标文件中所有的的getters/setters, blocks/closures, extensions/ categories,functions/methods加上断点。
```
(lldb) rb . -f DetailViewController.swift
```

使用*-s*，给目标library重的所有符号添加断点。
```
(lldb) rb . -s Commons
(lldb) rb . -s UIKit
```

使用*-o*选项，可以限制每个断点只命中一次。当命中一个断点之后，这个断点会自动被删除。
```
(lldb) rb . -s UIKit -o
```


#### 修改和删除断点
执行下面的命令
```
(lldb) b main
Breakpoint 1: 44 locations.
(lldb) breakpoint list 1
1: name = 'main', locations = 44, resolved = 44, hit count = 0
  1.1: where = Signals`main at AppDelegate.swift, address = 0x0000000108070540, resolved, hit count = 0 
  1.2: where = Foundation`-[NSThread main], address = 0x00000001083b89e3, resolved, hit count = 0 
  1.3: where = Foundation`-[NSBlockOperation main], address = 0x00000001083c57d6, resolved, hit count = 0 
  1.4: where = Foundation`-[NSFilesystemItemRemoveOperation main], address = 0x00000001083fee99, resolved, hit count = 0 
  1.5: where = Foundation`-[NSFilesystemItemMoveOperation main], address = 0x00000001083ff9ee, resolved, hit count = 0 
  .............
```

使用下面的命令可以查看简要信息
```
(lldb) breakpoint list 1 -b
1: name = 'main', locations = 44, resolved = 44, hit count = 0
```

其他查看断点的命令：
```
(lldb) breakpoint list 1 3 
(lldb) breakpoint list 1-3
```

删除：
```
(lldb) breakpoint delete 1
(lldb) breakpoint delete 1.1
```
