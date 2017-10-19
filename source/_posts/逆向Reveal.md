---
title: 逆向Reveal
date: 2017-10-18 22:20:38
tags: [逆向]
---


最近在读《Advanced-Apple-Debugging-Reverse-Engineering》，想找个MacOS上的app练下手。以前买了Reveal，改成订阅制之后没有重新订阅。偶尔需要用一下，所以就选了Reveal。

### ptrace反动态调试
首先在终端执行`lldb -f /Applications/Reveal.app/Contents/MacOS/Reveal`,并且使用`run`命令开始调试。

![](http://onkcruzxc.bkt.clouddn.com/15083369269912.jpg)
如上图，在lldb中执行`run`命令之后，Reveal进程马上退出了，并且返回45。这是因为Reveal使用了`ptrace`来达到反动态调试的目的。
    接下来我们使用Hopper来看看Reveal的反编译代码，打开Hopper，选择'File'，'Read Executable to Disassemble'选项，选择`/Applications/Reveal.app/Contents/MacOS/Reveal`。等加载完成之后，我们在Hopper中搜索ptrace

![](http://onkcruzxc.bkt.clouddn.com/15083374751605.jpg)

从上图中的`0000000100378b20         jmp        qword [_ptrace_ptr]                         ; _ptrace, CODE XREF=EntryPoint+37`可以看出，ptrace又一个函数调用 ，点击`EntryPoint+37`跳到调用的地方。



``` asm
000000010029319a         push       rbp
000000010029319b         mov        rbp, rsp
000000010029319e         push       r14
00000001002931a0         push       rbx
00000001002931a1         mov        r14, rsi
00000001002931a4         mov        ebx, edi
00000001002931a6         inc        qword [0x1004d82c8]
00000001002931ad         inc        qword [0x1004d82d0]
00000001002931b4         mov        edi, 0x1f                                    ; argument "request" for method imp___stubs__ptrace
00000001002931b9         xor        esi, esi                                    ; argument "pid" for method imp___stubs__ptrace
00000001002931bb         xor        edx, edx                                    ; argument "addr" for method imp___stubs__ptrace
00000001002931bd         xor        ecx, ecx                                    ; argument "data" for method imp___stubs__ptrace
00000001002931bf         call       imp___stubs__ptrace
00000001002931c4         mov        edi, ebx                                    ; argument "argc" for method imp___stubs__NSApplicationMain
00000001002931c6         mov        rsi, r14                                    ; argument "argv" for method imp___stubs__NSApplicationMain
00000001002931c9         pop        rbx
00000001002931ca         pop        r14
00000001002931cc         pop        rbp
00000001002931cd         jmp        imp___stubs__NSApplicationMain
                        ; endp
```

我们可以根据`mov edi, 0xe`这一行看出，调用`ptrace`的时候，传入的第一个参数是`0x1f`。

根据ptrace.h的内容

```

#define	PT_TRACE_ME	0	/* child declares it's being traced */
#define	PT_READ_I	1	/* read word in child's I space */
#define	PT_READ_D	2	/* read word in child's D space */
#define	PT_READ_U	3	/* read word in child's user structure */
#define	PT_WRITE_I	4	/* write word in child's I space */
#define	PT_WRITE_D	5	/* write word in child's D space */
#define	PT_WRITE_U	6	/* write word in child's user structure */
#define	PT_CONTINUE	7	/* continue the child */
#define	PT_KILL		8	/* kill the child process */
#define	PT_STEP		9	/* single step the child */
#define	PT_ATTACH	ePtAttachDeprecated	/* trace some running process */
#define	PT_DETACH	11	/* stop tracing a process */
#define	PT_SIGEXC	12	/* signals as exceptions for current_proc */
#define PT_THUPDATE	13	/* signal for thread# */
#define PT_DENY_ATTACH	14	/* attach to running process with signal exception */

#define	PT_FORCEQUOTA	30	/* Enforce quota for root */
#define	PT_DENY_ATTACH	31

#define	PT_FIRSTMACH	32	/* for machine-specific requests */

__BEGIN_DECLS


int	ptrace(int _request, pid_t _pid, caddr_t _addr, int _data);
```

`#define	PT_DENY_ATTACH	31` 31刚好是16进制的0x1f。这是一种常见的iOS/macOS反调试方式。我们将这个PT_DENY_ATTACH改成PT_DENY_ATTACH即可。在Hopper中选中`mov edi, 0x1f`这行，按Option+A快捷键，输入 `mov edi，0xe`，回车即可。如下图
![](http://onkcruzxc.bkt.clouddn.com/15083387215674.jpg)

然后选中 File 菜单中的 'Produce New executable...'选项，覆盖/Applications/Reveal.app/Contents/MacOS/Reveal这个文件。这样就可以了

### Damaged

然后我们使用lldb的run命令重新调试，Reveal会弹出一个对话框。如下图：
![](http://onkcruzxc.bkt.clouddn.com/15083390547698.jpg)
这应该是因为因为我们修改了Reveal可执行文件导致的。我们试试去掉这个对话框，在Hopper的Strings选项卡中搜索‘Damage’关键字，我们搜索到下面的的一行
```
db         "Application Damaged Alert", 0              ; DATA XREF=sub_100156540+193
```
这个应该是那个对话框，点击XREF的地方，我们跟踪下这个函数，跳到了函数`sub_100156540`，


```
    sub_100156540:
0000000100156540         push       rbp                                         ; CODE XREF=sub_100156690+155
0000000100156541         mov        rbp, rsp
0000000100156544         push       r15
0000000100156546         push       r14
0000000100156548         push       r13
......................
00000001001565fa         inc        qword [0x1004d4488]
0000000100156601         lea        rdi, qword [0x100395b50]                    ; "Application Damaged Alert", argument #1 for method sub_1001a1510
0000000100156608         mov        esi, 0x19                                   ; argument #2 for method sub_1001a1510
............
```

我们在确定下会不会走这个函数，在lldb中添加一个断点`b -a 0x0000000100156540`，**记住地址前面一定要加0x,表示是16位进制的地址。**，重新run之后，确实走到了这个函数,如下图
![](http://onkcruzxc.bkt.clouddn.com/15083396726347.jpg)

我们再根据sub_100156540的 'CODE XREF'看看他的调用者，追了几个`CODE XREF`之后，我们看到了一个函数
![](http://onkcruzxc.bkt.clouddn.com/15083398659619.jpg)
这个函数没有返回值，我们让这个函数直接返回看看，在函数入口按Option+A,输入‘ret’之后，我们重新生成Reveal，在重新调试。这次Damaged的对话框没有出来，但是有弹出了另一个对话框如下图
![](http://onkcruzxc.bkt.clouddn.com/15083400433919.jpg)

### Activation Screen
根据提示语在Hopper的Strings选项卡中搜索一下，好想也没有搜索到有用的内容。我们看看Reveal还有引入其他的库，果然在Frameworks中找个一个DevMateKit.framework。我们把他的内容用Hopper看看。把下图中的文件导入到Hopper中，
![](http://onkcruzxc.bkt.clouddn.com/15083403696454.jpg)

在DevMateKit的hopper窗口中搜索Welcome，搜索到如下的结果

![](http://onkcruzxc.bkt.clouddn.com/15083405954930.jpg)
我们看看这个DMWelcomeStepController会不会走，
在lldb中添加断点，在重新run一次看看，


```
(lldb) b +[DMWelcomeStepController defaultNibName]
Breakpoint 1: where = DevMateKit`+[DMWelcomeStepController defaultNibName], address = 0x0000000000014884
(lldb) run
Process 65679 launched: '/Applications/Reveal.app/Contents/MacOS/Reveal' (x86_64)
...........
Process 65679 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x00000001006e3884 DevMateKit`+[DMWelcomeStepController defaultNibName]
DevMateKit`+[DMWelcomeStepController defaultNibName]:
->  0x1006e3884 <+0>: push   rbp
    0x1006e3885 <+1>: mov    rbp, rsp
    0x1006e3888 <+4>: push   rbx
    0x1006e3889 <+5>: push   rax
Target 0: (Reveal) stopped.
```

断点在了+[DMWelcomeStepController defaultNibName],我们继续调试这个函数


``` shell
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 1.1
    frame #0: 0x00000001006e3884 DevMateKit`+[DMWelcomeStepController defaultNibName]
DevMateKit`+[DMWelcomeStepController defaultNibName]:
->  0x1006e3884 <+0>: push   rbp
    0x1006e3885 <+1>: mov    rbp, rsp
    0x1006e3888 <+4>: push   rbx
    0x1006e3889 <+5>: push   rax
Target 0: (Reveal) stopped.
(lldb) finish
Process 65679 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = step out
    frame #0: 0x00000001006f0342 DevMateKit`+[DMStepController defaultViewController] + 47
DevMateKit`+[DMStepController defaultViewController]:
->  0x1006f0342 <+47>: mov    rdi, rax
    0x1006f0345 <+50>: call   0x100743596               ; symbol stub for: objc_retainAutoreleasedReturnValue
    0x1006f034a <+55>: mov    r15, rax
    0x1006f034d <+58>: mov    rdi, qword ptr [rip + 0xb623c] ; (void *)0x00007fffb17d7328: NSBundle
Target 0: (Reveal) stopped.
(lldb) finish
Process 65679 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = step out
    frame #0: 0x00000001006f93eb DevMateKit`-[DMActivationController stepControllerForStep:] + 457
DevMateKit`-[DMActivationController stepControllerForStep:]:
->  0x1006f93eb <+457>: mov    rdi, rax
    0x1006f93ee <+460>: call   0x100743596               ; symbol stub for: objc_retainAutoreleasedReturnValue
    0x1006f93f3 <+465>: mov    r12, rax
    0x1006f93f6 <+468>: xor    r15d, r15d
Target 0: (Reveal) stopped.
(lldb) finish
Process 65679 stopped
* thread #1, queue = 'com.apple.main-thread', stop reason = step out
    frame #0: 0x00000001006f96b2 DevMateKit`-[DMActivationController updateStepControllerForCurrentStep] + 123
DevMateKit`-[DMActivationController updateStepControllerForCurrentStep]:
->  0x1006f96b2 <+123>: mov    rdi, rax
    0x1006f96b5 <+126>: call   0x100743596               ; symbol stub for: objc_retainAutoreleasedReturnValue
    0x1006f96ba <+131>: mov    r12, rax
    0x1006f96bd <+134>: test   r12, r12
Target 0: (Reveal) stopped.
```

我们看看堆栈

```
(lldb) thread backtrace
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 2.2
  * frame #0: 0x00000001006e3884 DevMateKit`+[DMWelcomeStepController defaultNibName]
    frame #1: 0x00000001006f0342 DevMateKit`+[DMStepController defaultViewController] + 47
    frame #2: 0x00000001006f93eb DevMateKit`-[DMActivationController stepControllerForStep:] + 457
    frame #3: 0x00000001006f96b2 DevMateKit`-[DMActivationController updateStepControllerForCurrentStep] + 123
    frame #4: 0x00000001006f673c DevMateKit`-[DMActivationController runActivationWindowInMode:initialActivationInfo:withCompletionHandler:] + 701
    frame #5: 0x0000000100023f05 Reveal`___lldb_unnamed_symbol1299$$Reveal + 1877
    frame #6: 0x0000000100026ffc Reveal`___lldb_unnamed_symbol1333$$Reveal + 12
    frame #7: 0x0000000100884670 RxSwift`RxSwift.(AnonymousObservable in _95EBF5692819D58425EC2DD0512D115A).run<A where A == A1.E, A1: RxSwift.ObserverType>(A1, cancel: RxSwift.Cancelable) -> (sink: RxSwift.Disposable, subscription: RxSwift.Disposable) + 288
```

我们把 `[DMActivationController stepControllerForStep:] `  这个函数弄成空函数试试效果，
![](http://onkcruzxc.bkt.clouddn.com/15083419194531.jpg)
结果，重启Reveal之后，Reveal直接crash了。我们在试试`[DMActivationController updateStepControllerForCurrentStep]`函数弄成空的
![](http://onkcruzxc.bkt.clouddn.com/15083420371862.jpg)

运行reveal，发现正常启动起来了。也可以查看iOS模拟器的app了。

