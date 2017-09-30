---
title: Advanced-Apple-Debugging-Reverse-Engineering-第6-7章笔记
date: 2017-09-30 12:20:34
tags: [lldb,debug]
---

#第六章 Thread, Frame & Stepping Around
在demo上添加一个`Signals.MasterViewController.viewWillAppear(Swift.Bool) -> ()` 的符号断点，运行程序之后，断点在viewWillAppear上。

## Frames
```
(lldb) thread backtrace
* thread #1, queue = 'com.apple.main-thread', stop reason = breakpoint 6.1
  * frame #0: 0x000000010c96d6f0 Signals`MasterViewController.viewWillAppear(animated=false, self=0x0000000000000000) at MasterViewController.swift:61
    frame #1: 0x000000010c96dd53 Signals`@objc MasterViewController.viewWillAppear(_:) at MasterViewController.swift:0
    frame #2: 0x000000010e1389b0 UIKit`-[UIViewController _setViewAppearState:isAnimating:] + 444
    frame #3: 0x000000010e139011 UIKit`__52-[UIViewController _setViewAppearState:isAnimating:]_block_invoke + 265
    frame #4: 0x0000000110d0b33a CoreFoundation`-[__NSSingleObjectArrayI enumerateObjectsWithOptions:usingBlock:] + 58
(lldb) frame info
frame #0: 0x000000010c96d6f0 Signals`MasterViewController.viewWillAppear(animated=false, self=0x0000000000000000) at MasterViewController.swift:61
(lldb) frame select 1
frame #1: 0x000000010c96dd53 Signals`@objc MasterViewController.viewWillAppear(_:) at MasterViewController.swift:0
   1   	/**
   2   	 * MasterViewController.swift
   3   	 *
   4   	 * Copyright (c) 2017 Razeware LLC
   5   	 *
   6   	 * Permission is hereby granted, free of charge, to any person obtaining a copy
   7   	 * of this software and associated documentation files (the "Software"), to deal
```

## Stepping
### Stepping over
```
(lldb) run   #重启进程
(lldb) next
```
### Stepping in
```
(lldb) step
(lldb) settings show target.process.thread.step-in-avoid-nodebug
(lldb) step -a0
```

### Steping out
```
(lldb) finish
```
## Examing Data in the stack

断点在viewWillAppear(_:)，执行下面的命令
```
(lldb) frame variable
(Bool) animated = false
(Signals.MasterViewController) self = <uninitialized>
(lldb) n
(lldb) frame variable
(Bool) animated = false
(Signals.MasterViewController) self = 0x00007f875b757030 {
  UIKit.UITableViewController = {
    baseUIViewController@0 = <extracting data from value failed>

    _tableViewStyle = 0
    _keyboardSupport = nil
    _staticDataSource = nil
    _filteredDataSource = 0x000060000045e030
    _filteredDataType = 0
  }
  detailViewController = nil
  a = 2
}
```

使用'-F'

```
(lldb) frame variable -F self
self = 0x00007f875b757030
self =
self.isa = 0x00007f875b757030
self._overrideTransitioningDelegate = nil
self._view = 0x00007f875c859000
self._tabBarItem = nil
self._navigationItem = 0x00007f875b710810
self._toolbarItems = nil
self._title = nil
self._nibName = 0x000060000045e210 "7bK-jq-Zjz-view-r7i-6Z-zg0"
self._nibBundle = 0x0000600000281fe0 "/Users/liucien/Library/Developer/CoreSimulator/Devices/6B02AF95-6124-4D0A-86D5-C9E3000F6890/data/Containers/Bundle/Application/FE2339E6-0B72-43E4-B307-C35A8964B910/Signals.app"
self._parentViewController = 0x00007f875b501810
self._childModalViewController = nil
.................

```

#第七章 image

`image`是`target modules`的别名。image命令可以查询加载的模块信息。

## modules
```
(lldb) image list
[ 0] 13A9466A-2576-3ABB-AD9D-D6BC16439B8F 0x00000001013aa000 /usr/lib/ dyld 
[ 1] 493D07DF-3F9F-30E0-96EF-4A398E59EC4A 0x000000010118e000 / Applications/Xcode.app/Contents/Developer/Platforms/ iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk/usr/lib/ dyld_sim 
[ 2] 4969A6DB-CE85-3051-9FB2-7D7B2424F235 0x000000010115c000 /Users/ derekselander/Library/Developer/Xcode/DerivedData/Signalsbqrjxlceauwfuihjesxmgfodimef/Build/Products/Debug-iphonesimulator/ Signals.app/Signals

```

```
(lldb) image list Foundation
[ 0] 4212F72C-2A19-323A-84A3-91FEABA7F900 0x0000000101435000 / Applications/Xcode.app/Contents/Developer/Platforms/ iPhoneSimulator.platform/Developer/SDKs/iPhoneSimulator.sdk//System/ Library/Frameworks/Foundation.framework/Foundation
```
输入出的内容包括:
    1.第一列的UUID（4212F72C-2A19-323A-84A3-91FEABA7F900）是Foundation模块的唯一标识别。
    2.UUID后面的地址（0x0000000101435000是Foundation module在进程空间中的地址。
    3.Foundation module的路径。

```
(lldb) image dump symtab UIKit -s address
Symtab, file = /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/System/Library/Frameworks/UIKit.framework/UIKit, num_symbols = 119427 (sorted by address):
               Debug symbol
               |Synthetic symbol
               ||Externally Visible
               |||
Index   UserID DSX Type            File Address/Value Load Address       Size               Flags      Name
------- ------ --- --------------- ------------------ ------------------ ------------------ ---------- ----------------------------------
[    0]      0     Code            0x0000000000001cd0 0x000000010f5fccd0 0x0000000000000070 0x000e0000 -[UIGestureKeyboardIntroduction initWithLayoutStar:completion:]
[    1]      1     Code            0x0000000000001d40 0x000000010f5fcd40 0x000000000000085f 0x000e0000 -[UIGestureKeyboardIntroduction showGestureKeyboardIntroduction]
[    2]      2     Code            0x000000000000259f 0x000000010f5fd59f 0x00000000000002df 0x000e0000 __64-[UIGestureKeyboardIntroduction showGestureKeyboardIntroduction]_block_invoke
```

一些已经见过的用法
```
(lldb) image lookup -rn UIViewController
(lldb) image lookup -rn '\[UIViewController\ '
(lldb) image lookup -rn '\[UIViewController\(\w+\)\ '   #改进版，可以包含categories里的符号
```

## Hunting for code
![](http://onkcruzxc.bkt.clouddn.com/1506761520.png )

如上图，在这里添加断点，并且运行demo。

```
(lldb) frame info
frame #0: 0x0000000100cb20a0 Commons`__34+[UnixSignalHandler sharedHandler]_block_invoke((null)=0x0000000100cb7210) + 16 at UnixSignalHandler.m:68
```
这里有个*_block_invoke*。
我们可以找到所有的_block_invoke。
```
(lldb) image lookup -rn _block_invoke
```
这个打印了这个程序加载的所有的Objective-C blocks。

```
(lld) image lookup -rn _block_invoke Commons
6 matches found in /Users/liucien/Library/Developer/Xcode/DerivedData/Signals-gvytwdetzthkqkhjwmylmezagfmo/Build/Products/Debug-iphonesimulator/Signals.app/Frameworks/Commons.framework/Commons:
        Address: Commons[0x0000000000002370] (Commons.__TEXT.__text + 1216)
        Summary: Commons`__32-[UnixSignalHandler initPrivate]_block_invoke at UnixSignalHandler.m:78        Address: Commons[0x00000000000025e0] (Commons.__TEXT.__text + 1840)
        Summary: Commons`__32-[UnixSignalHandler initPrivate]_block_invoke.27 at UnixSignalHandler.m:105        Address: Commons[0x0000000000001f30] (Commons.__TEXT.__text + 128)
        Summary: Commons`__34+[UnixSignalHandler sharedHandler]_block_invoke at UnixSignalHandler.m:67        Address: Commons[0x00000000000027a0] (Commons.__TEXT.__text + 2288)
        Summary: Commons`__38-[UnixSignalHandler appendSignal:sig:]_block_invoke at UnixSignalHandler.m:119        Address: Commons[0x00000000000027e0] (Commons.__TEXT.__text + 2352)
        Summary: Commons`__38-[UnixSignalHandler appendSignal:sig:]_block_invoke_2 at UnixSignalHandler.m:123        Address: Commons[0x0000000000002b60] (Commons.__TEXT.__text + 3248)
        Summary: Commons`__38-[UnixSignalHandler appendSignal:sig:]_block_invoke_3 at UnixSignalHandler.m:135
```

这个找到了所有Commons模块的Objective-C blocks.

```
(lldb) rb appendSignal.*_block_invoke -s Commons
Breakpoint 2: 3 locations.
```

给Commons模块中的-[UnixSignalHandler appendSignal:sig:]函数内所有的block都加上断点。

-[UnixSignalHandler appendSignal:sig:]函数代码。
```
- (void)appendSignal:(siginfo_t *)siginfo sig:(int)sig {
  
  static int signalID = 0;
  static dispatch_queue_t queue = nil;
  static dispatch_once_t onceToken;
  // 断点1，__38-[UnixSignalHandler appendSignal:sig:]_block_invoke
  dispatch_once(&onceToken, ^{
    queue = dispatch_queue_create("SignalQueue", DISPATCH_QUEUE_SERIAL);
  });
  
  //断点2 __38-[UnixSignalHandler appendSignal:sig:]_block_invoke_2
  dispatch_async(queue, ^{
    
    NSLog(@"Appending new signal: %@", signalIntToName(sig));
    signalID++;
    UnixSignal *unixSignal = nil;
    if (siginfo == NULL) {
      unixSignal = [[UnixSignal alloc] initWithSignalValue:nil signum:SIGSTOP breakpointID:signalID];
    } else {
      unixSignal = [[UnixSignal alloc] initWithSignalValue:[NSValue valuewithSiginfo:*siginfo] signum:sig breakpointID:signalID];
    }
    
    [(NSMutableArray *)self.signals addObject:unixSignal];
    dispatch_async(dispatch_get_main_queue(), ^{
      [[NSNotificationCenter defaultCenter] postNotificationName:kSignalHandlerCountUpdatedNotification object:nil];
    });
  });
}
```

如下图，调试上面这个函数。
![](http://onkcruzxc.bkt.clouddn.com/1506762644.png )

使用下面的命令导出Commons模块的符号，可以找到上图中__block_literal_5的信息。

```
(lldb) image dump symfile Commons
```
![](http://onkcruzxc.bkt.clouddn.com/1506762914.png )


```
(lldb) frame variable
(__block_literal_5 *)  = 0x00000001060c7170
(int) sig = 17
(siginfo_t *) siginfo = 0x0000000000000000
(UnixSignalHandler *) self = 0x000060000047ca80
(UnixSignal *) unixSignal = 0x0000000106004a3e
(lldb) po ((__block_literal_5 *)0x00000001060c7170)
__NSMallocBlock__

(lldb) p/x ((__block_literal_5 *)0x00000001060c7170)->__FuncPtr
(void (*)()) $2 = 0x0000604000113440
(lldb) image lookup -a 0x0000604000113440 //不知道为什么，这个没有任何输出。跟原书的例子不一样。
(lldb) po ((__block_literal_5 *)0x0000604000113440)->sig //
warning: `self' is not accessible (subsituting 0)
<nil>
```


### Snooping aroud

```
(lldb) po 0x0000000105630170
warning: `self' is not accessible (subsituting 0)
__NSMallocBlock__
```
我们来看看__NSMallocBlock__
```
(lldb) image lookup -rn __NSMallocBlock__
```
什么都没有输出，说明他没有实现他父类的任何方法。

```
po [__NSMallocBlock__ superclass]
warning: `self' is not accessible (subsituting 0)
__NSMallocBlock
(lldb) image lookup -rn __NSMallocBlock
5 matches found in /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/System/Library/Frameworks/CoreFoundation.framework/CoreFoundation:
        Address: CoreFoundation[0x0000000000055d30] (CoreFoundation.__TEXT.__text + 346528)
        Summary: CoreFoundation`-[__NSMallocBlock retain]        Address: CoreFoundation[0x0000000000056cd0] (CoreFoundation.__TEXT.__text + 350528)
        Summary: CoreFoundation`-[__NSMallocBlock release]        Address: CoreFoundation[0x0000000000198c30] (CoreFoundation.__TEXT.__text + 1669280)
        Summary: CoreFoundation`-[__NSMallocBlock retainCount]        Address: CoreFoundation[0x0000000000198c40] (CoreFoundation.__TEXT.__text + 1669296)
        Summary: CoreFoundation`-[__NSMallocBlock _tryRetain]        Address: CoreFoundation[0x0000000000198c50] (CoreFoundation.__TEXT.__text + 1669312)
        Summary: CoreFoundation`-[__NSMallocBlock _isDeallocating]
(lldb) 
```
__NSMallocBlock里面都是一些内存管理的方法。我们来看看它的父类。

```
(lldb) po [__NSMallocBlock superclass]
warning: `self' is not accessible (subsituting 0)
NSBlock

(lldb) image lookup -rn 'NSBlock\ '
6 matches found in /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/System/Library/Frameworks/CoreFoundation.framework/CoreFoundation:
        Address: CoreFoundation[0x0000000000041990] (CoreFoundation.__TEXT.__text + 263680)
        Summary: CoreFoundation`-[NSBlock copy]        Address: CoreFoundation[0x000000000007aad0] (CoreFoundation.__TEXT.__text + 497472)
        Summary: CoreFoundation`-[NSBlock copyWithZone:]        Address: CoreFoundation[0x0000000000198b30] (CoreFoundation.__TEXT.__text + 1669024)
        Summary: CoreFoundation`+[NSBlock allocWithZone:]        Address: CoreFoundation[0x0000000000198b50] (CoreFoundation.__TEXT.__text + 1669056)
        Summary: CoreFoundation`+[NSBlock alloc]        Address: CoreFoundation[0x0000000000198b70] (CoreFoundation.__TEXT.__text + 1669088)
        Summary: CoreFoundation`-[NSBlock invoke]        Address: CoreFoundation[0x0000000000198b80] (CoreFoundation.__TEXT.__text + 1669104)
        Summary: CoreFoundation`-[NSBlock performAfterDelay:]
```

我们将0x0000000105630170转化成NSBlock，直接调用他的invoke方法。

```
(lldb) po id $block = (id)0x0000000105630170
warning: `self' is not accessible (subsituting 0)
(lldb) po [$block retain]
warning: `self' is not accessible (subsituting 0)
__NSMallocBlock__

(lldb) po [$block invoke]
error: warning: couldn't get cmd pointer (substituting NULL): no variable named '_cmd' found in this frame
error: Execution was interrupted, reason: Attempted to dereference an invalid ObjC Object or send it an unrecognized selector.
The process has been returned to the state before expression evaluation.
```
` po [$block invoke]` 这个执行出错了，跟原书有点出处。‘warning: `self' is not accessible (subsituting 0)’打印这个是因为，我们到达block断点时，还没有证实进入。所以需要在lldb中执行下`nex`命令。

### 调试私有方法

```
(lldb) image lookup -rn (?i)\ _\w+description\]
```
![](http://onkcruzxc.bkt.clouddn.com/1506765457.png )

在这个结果中，我们注意到有个IvarDescription属于UIKit。我们来看看这个IvarDescription

```
(lldb) image lookup -rn NSObject\(IvarDescription\)
7 matches found in /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/System/Library/Frameworks/UIKit.framework/UIKit:
        Address: UIKit[0x0000000000070aa8] (UIKit.__TEXT.__text + 454104)
        Summary: UIKit`-[NSObject(IvarDescription) __ivarDescriptionForClass:]        Address: UIKit[0x0000000000070c61] (UIKit.__TEXT.__text + 454545)
        Summary: UIKit`-[NSObject(IvarDescription) _ivarDescription]        Address: UIKit[0x0000000000070d55] (UIKit.__TEXT.__text + 454789)
        Summary: UIKit`-[NSObject(IvarDescription) __propertyDescriptionForClass:]        Address: UIKit[0x0000000000071220] (UIKit.__TEXT.__text + 456016)
        Summary: UIKit`-[NSObject(IvarDescription) _propertyDescription]        Address: UIKit[0x0000000000071302] (UIKit.__TEXT.__text + 456242)
        Summary: UIKit`-[NSObject(IvarDescription) __methodDescriptionForClass:]        Address: UIKit[0x000000000007182b] (UIKit.__TEXT.__text + 457563)
        Summary: UIKit`-[NSObject(IvarDescription) _methodDescription]        Address: UIKit[0x000000000007190d] (UIKit.__TEXT.__text + 457789)
        Summary: UIKit`-[NSObject(IvarDescription) _shortMethodDescription]
(lldb) 
```

```
(lldb) po [[UIApplication sharedApplication] _ivarDescription]
<UIApplication: 0x7fca3af005d0>:
in UIApplication:
	_delegate (<UIApplicationDelegate>*): <Signals.AppDelegate: 0x60000003cca0>
	_event (UIEvent*): <UIEvent: 0x60000024fed0>
	_motionEvent (UIEvent*): <UIMotionEvent: 0x600000188880>
	_remoteControlEvent (UIEvent*): <UIRemoteControlEvent: 0x60000007bf00>
	_remoteControlEventObservers (long): 0
	_topLevelNibObjects (NSArray*): nil
	_networkResourcesCurrentlyLoadingCount (long): 0
    ...................

(lldb) image lookup -rn '\[UIStatusBar\ set'
20 matches found in /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/Library/CoreSimulator/Profiles/Runtimes/iOS.simruntime/Contents/Resources/RuntimeRoot/System/Library/Frameworks/UIKit.framework/UIKit:
        Address: UIKit[0x000000000063ad41] (UIKit.__TEXT.__text + 6525041)
        Summary: UIKit`+[UIStatusBar setTintOverrideEnabled:withColor:]        Address: UIKit[0x000000000063af54] (UIKit.__TEXT.__text + 6525572)
        Summary: UIKit`-[UIStatusBar setSuppressesGlow:]        Address: UIKit[0x000000000063afaf] (UIKit.__TEXT.__text + 6525663)
        Summary: UIKit`-[UIStatusBar setBackgroundAlpha:]        Address: UIKit[0x000000000063b0a4] (UIKit.__TEXT.__text + 6525908)
        Summary: UIKit`-[UIStatusBar setShowsOnlyCenterItems:]        Address: UIKit[0x000000000063e2b4] (UIKit.__TEXT.__text + 6538724)
        Summary: UIKit`-[UIStatusBar setTintColor:]        Address: UIKit[0x000000000063e314] (UIKit.__TEXT.__text + 6538820)
        Summary: UIKit`-[UIStatusBar setTintColor:withDuration:]        Address: UIKit[0x000000000063e4b7] (UIKit.__TEXT.__text + 6539239)
        Summary: UIKit`-[UIStatusBar setOrientation:]        Address: UIKit[0x000000000063e5dc] (UIKit.__TEXT.__text + 6539532)
        Summary: UIKit`-[UIStatusBar setHidden:animationParameters:]        Address: UIKit[0x000000000063e713] (UIKit.__TEXT.__text + 6539843)
        Summary: UIKit`-[UIStatusBar setSuppressesHiddenSideEffects:]        Address: UIKit[0x000000000063fa94] (UIKit.__TEXT.__text + 6544836)
        Summary: UIKit`-[UIStatusBar setEnabledCenterItems:duration:]        Address: UIKit[0x000000000064017b] (UIKit.__TEXT.__text + 6546603)
        Summary: UIKit`-[UIStatusBar setRegistered:]        Address: UIKit[0x000000000064024b] (UIKit.__TEXT.__text + 6546811)
        Summary: UIKit`-[UIStatusBar setPersistentAnimationsEnabled:]        Address: UIKit[0x00000000006402d5] (UIKit.__TEXT.__text + 6546949)
        Summary: UIKit`-[UIStatusBar setSimulatesLegacyAppearance:]        Address: UIKit[0x0000000000640346] (UIKit.__TEXT.__text + 6547062)
        Summary: UIKit`-[UIStatusBar setForegroundColor:animationParameters:]        Address: UIKit[0x0000000000640470] (UIKit.__TEXT.__text + 6547360)
        Summary: UIKit`-[UIStatusBar setForegroundAlpha:animationParameters:]        Address: UIKit[0x0000000000640543] (UIKit.__TEXT.__text + 6547571)
        Summary: UIKit`-[UIStatusBar setLegibilityStyle:animationParameters:]        Address: UIKit[0x00000000006407a7] (UIKit.__TEXT.__text + 6548183)
        Summary: UIKit`-[UIStatusBar setStyleRequest:animationParameters:]        Address: UIKit[0x0000000000640a43] (UIKit.__TEXT.__text + 6548851)
        Summary: UIKit`-[UIStatusBar setAction:forRegionWithIdentifier:]        Address: UIKit[0x0000000000641d98] (UIKit.__TEXT.__text + 6553800)
        Summary: UIKit`-[UIStatusBar setStatusBarWindow:]        Address: UIKit[0x0000000000641dbc] (UIKit.__TEXT.__text + 6553836)
        Summary: UIKit`-[UIStatusBar setTimeHidden:]
(lldb) po (BOOL)[[UIStatusBar class] isSubclassOfClass:[UIView class]]
YES

(lldb) po [[UIApplication sharedApplication] statusBar]
<UIStatusBar: 0x7fca3c828a00; frame = (0 0; 375 20); opaque = NO; autoresize = W+BM; layer = <CALayer: 0x604000231720>>

(lldb) po [0x7fca3c828a00 setBackgroundColor:[UIColor purpleColor]]
0x00006040000b9c01
```

如下图，状态栏的颜色变成了紫色。
![](http://onkcruzxc.bkt.clouddn.com/1506766030.png )

## 其他
后面自己尝试看下Swift的Closure/didSet/willSet property helpers的结构。

```
struct TestStruct {
  var a = "abc" {
    didSet {
      print("didSet a:" + a)
    }
    willSet {
      print("willSet a:" + a)
    }
  }
  
  func closureTest(closure: () -> Void) {
    closure()
  }
}


class MasterViewController: UITableViewController {
  override func viewDidLoad() {
    super.viewDidLoad()

    var test = TestStruct()
    test.a = "222"
    test.closureTest {
      print("hahaha")
    }

    [1,2].map {
      return $0 + 1
    }
    
}
    
```
上面的viewDidLoad里面有两个closure，我们调试看看。
```
image lookup -rn 'closure #'  Signals
15 matches found in /Users/liucien/Library/Developer/Xcode/DerivedData/Signals-culiiujhqnysbxfesxnutfbtktfz/Build/Products/Debug-iphonesimulator/Signals.app/Signals:
        Summary: Signals`closure #1 () -> () in Signals.MasterViewController.viewDidLoad() -> () at MasterViewController.swift:69        Address: Signals[0x0000000100002d40] (Signals.__TEXT.__text + 5824)
        Summary: Signals`closure #2 (Swift.Int) -> Swift.Int in Signals.MasterViewController.viewDidLoad() -> () at MasterViewController.swift:73        Address: Signals[0x0000000100006c30] (Signals.__TEXT.__text + 21936)
        .........................
```

生成了包含了类似`closure #1 () -> () in Signals.MasterViewController.viewDidLoad() -> () `的符号。


```
(lldb) image dump symtab Signals -s address
..........................
[   17]     49 D X Code            0x0000000100001810 0x0000000104f4a810 0x0000000000000140 0x000f0000 Signals.TestStruct.a.didset : Swift.String
[   18]     53 D X Code            0x0000000100001950 0x0000000104f4a950 0x0000000000000140 0x000f0000 Signals.TestStruct.a.willset : Swift.String
...........................
```

Swift 里的Struce的willSet/didSet会分别生成`模块名字.对象类型名.属性名.{didset|willset} ：属性类型`的符号。










