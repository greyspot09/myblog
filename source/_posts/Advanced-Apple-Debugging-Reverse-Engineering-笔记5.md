---
title: Advanced-Apple-Debugging-Reverse-Engineering-第11章笔记
date: 2017-10-10 10:45:09
tags: [lldb,debug]
---
#Chapter 11: Assembly & Memory

## Setting up the Intel-Flavored Assembly Experience™
将下面的命令添加到.lldbinit中

```
settings set target.x86-disassembly-flavor intel 
settings set target.skip-prologue false
```
第一行告诉lldb使用intel风格的形式来显示x86的汇编代码。
第二行告诉lldb跳过function prologue(the beginning section of a function that prepares the stack and registers)

## Creating the cpx command

```
command alias -H "Print value in ObjC context in hexadecimal" -h "Print in hex" -- cpx expression -f x -l objc --
```
## Bits, bytes, and other terminology

- Nybble: 4 bits, a single value in hexadecimal
- Half word: 16 bits, or 2 bytes
- Word: 32 bits, or 4 bytes
- Double word or Giant word: 64 bits or 8 bytes.

## The RIP register
RIP(instruction pointer register)即指令寄存器。用于寻址代码段存储区内的下一条指令。我们可以通过修改RIP寄存器的值，以达到改变函数调用的目的。

``` Swift
class AppDelegate: NSObject, NSApplicationDelegate {
  
    func applicationWillBecomeActive( _ notification: Notification) {
        print("\(#function)")
        self.aBadMethod()
    }
    
    func aBadMethod() {
        print("\(#function)")
    }
    
    func aGoodMethod() {
        print("\(#function)")
    }
}
```
在`aBadMethod`上添加断点，在Xcode中设置，`Debug\Debug Workflow\Always Show Disassembly`断点时显示汇编代码。运行之后，断点在aBadMethod。

![](http://onkcruzxc.bkt.clouddn.com/2017-10-10-15076058137648.jpg)

我们先看看rip的值

```
(lldb) cpx $rip
(unsigned long) $0 = 0x0000000100008a10
```
0x0000000100008a10刚好是aBadMethod的函数入口。 然后我们需改rip的值，让程序执行aGoodMethod。我们先使用下面的命令找到aGoodMethod的函数地址。

```
(lldb) image lookup -vrn ^Registers.*aGoodMethod
```
结果如下图， -v 用于打印更多详细的信息。
![](http://onkcruzxc.bkt.clouddn.com/2017-10-10-15076059611674.jpg)
上图中，黄色部分，` range = [0x100008c20-0x100008ded)`就是aGoodMethod在内存中的物理地址。然后我们膝盖rip的值,并且点击运行，已经执行了`aGoodMethod`。
```
(lldb) regist write rip 0x100008c20
aGoodMethod()
applicationWillBecomeActive
```


##Registers and breaking up the bits

![](http://onkcruzxc.bkt.clouddn.com/2017-10-10-15076035645808.jpg)

```
(lldb) register write rdx 0x0123456789ABCDEF
(lldb) p/x $rdx
(unsigned long) $0 = 0x0123456789abcdef
(lldb) p/x $edx
(unsigned int) $1 = 0x89abcdef
(lldb) p/x $dx
(unsigned short) $3 = 0xcdef
(lldb) p/x $dl
(unsigned char) $4 = 0xef
(lldb) p/x $dh
(unsigned char) $5 = 0xcd
```


##Registers R8 to R15

```
(lldb) register write $r9 0x0123456789abcdef
(lldb) p/x $r9
(unsigned long) $6 = 0x0123456789abcdef
(lldb) p/x $r9d
(unsigned int) $7 = 0x89abcdef
(lldb) p/x $r9w
(unsigned short) $8 = 0xcdef
(lldb) p/x $r9l
(unsigned char) $9 = 0xef
```

## Breaking down the memory

还是上面的例子，我们断点在`aBadMethod`上。

```
(lldb) cpx $rip
(unsigned long) $1 = 0x0000000100008a10
(lldb) memory read -fi -c1 0x0000000100008a10
->  0x100008a10: 55  pushq  %rbp
(lldb) expression -f i -l objc -- 0x55
(int) $2 = 55  pushq  %rbp
```
从上面可以看出，16进制的0x55就是“pushq  %rbp”操作码。

```
(lldb) p/i 0x55
(Int) $R0 = 55  pushq  %rbp
```
也可以使用`p/i`来查看。

我们打印`0x0000000100008a10`的后边的汇编看看。

```
(lldb) memory read -fi -c10 0x0000000100008a10
->  0x100008a10: 55                    pushq  %rbp
    0x100008a11: 48 89 e5              movq   %rsp, %rbp
    0x100008a14: 48 81 ec c0 00 00 00  subq   $0xc0, %rsp
    0x100008a1b: 4c 89 6d f8           movq   %r13, -0x8(%rbp)
    0x100008a1f: b8 01 00 00 00        movl   $0x1, %eax
    0x100008a24: 89 c7                 movl   %eax, %edi
    0x100008a26: e8 43 06 00 00        callq  0x10000906e               ; symbol stub for: generic specialization <preserving fragile attribute, Any> of Swift._allocateUninitializedArray<A>(Builtin.Word) -> (Swift.Array<A>, Builtin.RawPointer)
    0x100008a2b: 48 89 c7              movq   %rax, %rdi
    0x100008a2e: 48 89 45 a8           movq   %rax, -0x58(%rbp)
    0x100008a32: 48 89 55 a0           movq   %rdx, -0x60(%rbp)
(lldb) p/i 0x4889e5
(Int) $R1 = e5 89  inl    $0x89, %eax
```

在使用`p/i 0x4889e5`打印地址`0x100008a11: 48 89 e5`上的内容是的时候，结果居然不是`movq   %rsp, %rbp`而是`inl    $0x89, %eax`。

这是因为x64的架构是小端字节序（little-endian）。

打印`0x100008a11: 48 89 e5`的指令应该使用`p/i 0xe58948`。

```
(lldb) p/i 0xe58948
(Int) $R3 = 48 89 e5  movq   %rsp, %rbp
```
这次结果正确了。

观察下面几个命令的执行结果。

```
(lldb) memory read -s1 -c20 -fx 0x0000000100008a10
0x100008a10: 0x55 0x48 0x89 0xe5 0x48 0x81 0xec 0xc0
0x100008a18: 0x00 0x00 0x00 0x4c 0x89 0x6d 0xf8 0xb8
0x100008a20: 0x01 0x00 0x00 0x00
```
```
(lldb) memory read -s2 -c20 -fx 0x0000000100008a10
0x100008a10: 0x4855 0xe589 0x8148 0xc0ec 0x0000 0x4c00 0x6d89 0xb8f8
0x100008a20: 0x0001 0x0000 0xc789 0x43e8 0x0006 0x4800 0xc789 0x8948
0x100008a30: 0xa845 0x8948 0xa055 0x8de8
```
```
(lldb) memory read -s4 -c20 -fx 0x0000000100008a10
0x100008a10: 0xe5894855 0xc0ec8148 0x4c000000 0xb8f86d89
0x100008a20: 0x00000001 0x43e8c789 0x48000006 0x8948c789
0x100008a30: 0x8948a845 0x8de8a055 0x48000006 0x48a87d8b
0x100008a40: 0xe8984589 0x0000067a 0xc1058b48 0x48000025
0x100008a50: 0x48a0558b 0xb9184289 0x00000003 0x05e8cf89
```



