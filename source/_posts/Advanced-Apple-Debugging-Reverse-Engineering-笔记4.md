---
title: Advanced-Apple-Debugging-Reverse-Engineering-第10章笔记
date: 2017-10-09 10:53:35
tags: [lldb,debug]
---




#Chapter 10: Assembly Register Calling Convention

## x86_64 vs ARM64


mac目前使用的大部分是x86_64，即64位系统，Apple在2010年之后已经停止了制造32位的Mac 。
`uname -m` 这个命令可以查看mac的电脑硬件架构。

iOS设备都是32-bit ARM 处理器。 在iPhone5及iPhone5之前的设备都是32位的。32位不支持安装iOS10。

## x86_64寄存器调用约束

在x64处理器上，一共有*16*个寄存器。分别是*RAX, RBX, RCX, RDX, RDI, RSI, RSP, RBP 和R8到R15*。

在函数调用的时候，参数通常这样存储

- 第一个参数： RDI
- 第二个参数： RSI
- 第三个参数： RDX
- 第四个参数： RCI
- 第五个参数： R8
- 第六个参数： R9

如果有个参数多余6个，其他的参数存储在stack(栈)上。

比如下面的代码

``` Objective-C
NSString *name = @"Zoltan"; 
NSLog(@"Hello world, I am %@. I'm %d, and I live in %@.", name, 30, @"my father's basement");
```

在调用NSLog的时候，参数会这样传递
```
RDI = @"Hello world, I am %@. I'm %d, and I live in %@."; 
RSI = @"Zoltan"; 
RDX = 30;
RCX = @"my father's basement"; 
NSLog(RDI, RSI, RDX, RCX);
```

##Objective-C和寄存器

``` Objective-C
[UIApplication sharedApplication];
```
编译器会将上面的代码编译成
```
id UIApplicationClass = [UIApplication class]; 
objc_msgSend(UIApplicationClass, "sharedApplication");
```

##实践
下面将用一个运行起来如下图的demo坐一些尝试。
![](http://onkcruzxc.bkt.clouddn.com/2017-10-09-15075334120413.jpg)


在demo中添加符号断点`-[NSViewController viewDidLoad]`。运行，停在断点之后，运行下面的命令
```
(lldb) po $rdi
<Registers.ViewController: 0x6040000e2280>

(lldb) po $rsi
140733848704322

(lldb) po (char *)$rsi
"viewDidLoad"

(lldb) po (SEL)$rsi
"viewDidLoad"
```

下面我们尝试将边框改成红色，添加如下的断点
```
(lldb) breakpoint set -o -S "-[NSWindow mouseDown:]" 
(lldb) continue
```

*-o* 参数指的是，这是一个one-shot breakpoint.只会命中一次的断点。

运行之后，我们点击App的列表外面，断点之后，执行下面的命令

```
(lldb) po $rdi
<NSWindow: 0x6040001e0300>

(lldb) po $rsi
140733848463351

(lldb) po (char*)$rsi
"mouseDown:"

(lldb) po (char*)$rdx
NSEvent: type=LMouseDown loc=(195.594,36.8984) time=21345.7 flags=0 win=0x6040001e0300 winNum=16145 ctxt=0x0 evNum=8877 click=1 buttonNumber=0 pressure=1 deviceID:0x0 subtype=0

(lldb) po [$rdi setBackgroundColor: [NSColor redColor]]
<object returned empty description>

(lldb) continue
```
然后界面变成了
![](http://onkcruzxc.bkt.clouddn.com/2017-10-09-15075338563887.jpg)

##Swift和寄存器
Swift有两需要注意
1. 寄存器不能在Swift调试上下文中使用。
2. Swift没有动态派发。

在Swift函数调用的时候，不需要objc_msgSend，除非，在忙代码中函数方法使用了*dynamic*关键字。

示例代码

``` Swift
    func executeLotsOfArguments(one: Int, two: Int, three: Int,
                                four: Int, five: Int, six: Int,
                                seven: Int, eight: Int, nine: Int,
                                ten: Int) {
        print("arguments are: \(one), \(two), \(three), \(four), \(five), \(six), \(seven), \(eight), \(nine), \(ten)")
    }
```
调用`executeLotsOfArguments`，
```
    self.executeLotsOfArguments(one: 1, two: 2, three: 3, four: 4,
                                five: 5, six: 6, seven: 7,
                                eight: 8, nine: 9, ten: 10)
```

在`executeLotsOfArguments`函数里出打上断点,运行之后，我们来观察一下寄存器

```
(lldb) register read -f d
General Purpose Registers:
       rax = 10
       rbx = 7
       rcx = 4
       rdx = 3
       rdi = 1
       rsi = 2
       rbp = 140732920750976
       rsp = 140732920750304
        r8 = 5
        r9 = 6
       r10 = 9
       r11 = 8
       r12 = 4312870496
       r13 = 105827995099904
       r14 = 88
       r15 = 4312870496
       rip = 4294983836  Registers`Registers.ViewController.executeLotsOfArguments(one: Swift.Int, two: Swift.Int, three: Swift.Int, four: Swift.Int, five: Swift.Int, six: Swift.Int, seven: Swift.Int, eight: Swift.Int, nine: Swift.Int, ten: Swift.Int) -> () + 76 at ViewController.swift:51
    rflags = 514
        cs = 43
        fs = 0
        gs = 0
```
可以看到，RDI, RSI, RDX, RCX, R8 and R9分别保存了前6个参数。

You may also notice other parameters are stored in some of the other registers. While this is true, it’s simply a leftover from the code that sets up the stack for the remaining parameters. Remember, *parameters after the sixth one go on the stack*.

## RAX返回值寄存器

我们修改下`executeLotsOfArguments`的实现，添加一个String类型的返回值。

``` Swift
    func executeLotsOfArguments(one: Int, two: Int, three: Int,
                                four: Int, five: Int, six: Int,
                                seven: Int, eight: Int, nine: Int,
                                ten: Int) -> String {
        print("arguments are: \(one), \(two), \(three), \(four), \(five), \(six), \(seven), \(eight), \(nine), \(ten)")
        return "Mom, what happened to the cat?"
    }
```

按照如下的代码调用
```
    let _ = self.executeLotsOfArguments(one: 1, two: 2, three: 3, four: 4,
                                five: 5, six: 6, seven: 7,
                                eight: 8, nine: 9, ten: 10)
```
在`executeLotsOfArguments`这个里面添加一个断点，我们是这捕获 executeLotsOfArguments 的返回值试试。

```
(lldb) finish
arguments are: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10
(lldb) register read rax
     rax = 0x0000000100009450  "Mom, what happened to the cat?"
```


##修改寄存器的值

准备工作，我们使用命令行打开一个iPhone模拟器,在命令行执行下面的命令

```
xcrun simctl list
```

我们找到一个iPhone7的模拟器

```
iPhone 7 (62629F3B-E6C7-400E-AC67-2ACBB4240FC4) (Shutdown)
```
打开模拟器

```
open /Applications/Xcode.app/Contents/Developer/Applications/Simulator.app --args -CurrentDeviceUDID 62629F3B-E6C7-400E-AC67-2ACBB4240FC4
```

模拟器启动之后，我们使用lldb调试SpringBoard（iOS的首页进程）

``` shell
lldb -n SpringBoard
```

lldb启动之后，我们在 `-[UILabel setText:]`上设置一个断点，并且修改text参数。

```
(lldb) p/x @"Yay! Debugging"
(__NSCFString *) $0 = 0x0000600000c36dc0 @"Yay! Debugging"
(lldb) b -[UILabel setText:]
Breakpoint 1: where = UIKit`-[UILabel setText:], address = 0x000000010433ca3c
(lldb) breakpoint command add
Enter your debugger command(s).  Type 'DONE' to end.
> po $rdx =0x0000600000c36dc0
> continue
> DONE
```

效果如下
![](http://onkcruzxc.bkt.clouddn.com/2017-10-09-15075379856824.gif)




重新进入通知界面，新创建的UILabel的文字已经成了"Yay! Debugging".

## 总结

- 在Objective-C的函数调用中，`RDI`寄存器保存的是调用者`NSObject`的引用，`RSI`保存的是`Selector`,`RDX`保存的是第一个参数
- 在Swift中，`RDI`保存的是第一个参数，`RSI`保存的是第二个参数，
- `RAX`寄存器去保存的是函数的返回值，Objective-C和Swift一样。
- 在Objective-C的调试上下文中，使用寄存器的时候一定要带上`$`符号。








