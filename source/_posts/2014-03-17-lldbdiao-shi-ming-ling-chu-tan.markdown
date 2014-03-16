---
layout: post
title: "LLDB调试命令初探"
date: 2014-03-17 00:50
comments: true
categories: 
---
<img src="https://developer.apple.com/library/ios/documentation/IDEs/Conceptual/gdb_to_lldb_transition_guide/art/lldb_in_xc5_command_window_2x.png" alt="../art/lldb_in_xc5_command_window_2x.png" width="562" height="183.5">  
如果你在平时的开发中从未使用过调试器，那你恐怕不知道一个调试器的作用有多大。你可能只满足于通过`printf`或者`NSLog`输出信息用于调试。但你只要试着尝试在调试中开始使用调试器*LLDB*，你会马上感受到调试器给你带来的便利。  
*LLDB*是*LLVM*下的调试器。Xcode从4.0开始编译器开始改用*LLVM*，相应的调试器也从*gdb*改为*LLDB*。而从 Xcode5.0开始所有工程也被自动设置为使用*LLDB*。下面本文从初学者的角度讲解在日常的开发中如何使用*LLDB*以及*LLDB*常用的命令。  
##初识*LLDB*
你可能从未使用过*LLDB*，那让我们先来热热身。
在调试器中最常用到的命令是`p`（用于输出基本类型）或者`po`（用于输出 Objective-C 对象）。如下，你可以通过输入po 和 view 来输出 view 的信息:  

	po [self view]
随后调试器会输出这个 object 的 description。在这个例子中可能是这样的信息：  

	(UIView *) $1 = 0x0824c800 <UITableView: 0x824c800; frame = (0 20; 768 1004); clipsToBounds = YES; autoresize = W+H; gestureRecognizers = <NSArray: 0x74c3010>; layer = <CALayer: 0x74c2710>; contentOffset: {0, 0}>
	
什么？在什么地方可以输入这个命令？OK，首先，我们需要先设置一个断点。如下图所示，我在`viewDidLoad:`中设置了一个了一个断点：
      
![图二：断点图](http://www.cimgf.com/wp-content/uploads/2012/12/Screenshot-121212-1214-AM.png)

接下来运行程序，然后程序会停留在断点处，从下图你可以看到在什么地方输入*LLDB*命令：

![图三：输入命令位置](http://www.cimgf.com/wp-content/uploads/2012/12/Screenshot-121212-1219-AM.png)

你可能需要的是 view 下 subview 的数量。由于 subview 的数量是一个 int 类型的值，所以我们使用命令`p`：  

	p (int)[[[self view] subviews] count]
最后你看到的输出会是：

	(int) $2 = 2
是不是很简单？  
细心的朋友可能会发现输出的信息中带有`$1`、`$2`的字样。实际上，我们每次查询的结果会保存在一些持续变量中($[0-9]+)，这样你可以在后面的查询中直接使用这些值。比如现在我接下来要重新取回`$1`的值：

	(lldb) po $1
	(UIView *) $1 = 0x0824c800 <UITableView: 0x824c800; frame = (0 20; 768 1004); clipsToBounds = YES; autoresize = W+H; gestureRecognizers = <NSArray: 0x74c3010>; layer = <CALayer: 0x74c2710>; contentOffset: {0, 0}>
可以看到，我们依然可以取到之前[self view]的值。
    
*LLDB*命令还可以用在断点上，详细的使用可以参见[这个文章](http://my.oschina.net/notting/blog/115294)
##常用命令
下面补充说明其它一些常用的命令：  

* expr  

可以在调试时动态执行指定表达式，并将结果打印出来。常用于在调试过程中修改变量的值。  
![图四：expr截图](http://www.starfelix.com/images/lldb-p1.png)  
如图设置断点，然后运行程序。程序中断后输入下面的命令：

	expr a=2
你会看到如下的输出：

	(int) $0 = 2
继续运行程序，程序输出的信息是：

	实际值：2
很明显可以看出，变量a的值被改变。
除此之外，还可以使用这个命令新声明一个变量对象，如：

	expr int $b=2
	p $b
下面的命令用于输出新声明对象的值。（注意，对象名前要加$）  

* call

call即是调用的意思。其实上述的po和p也有调用的功能。因此一般只在不需要显示输出，或是方法无返回值时使用call。
和上面的命令一样，我们依然在`viewDidLoad:`里面设置断点，然后在程序中断的时候输入下面的命令：

	call [self.view setBackgroundColor:[UIColor redColor]]
继续运行程序，看看view的背景颜色是不是变成红色的了！在调试的时候灵活运用call命令可以起到事半功倍的作用。

* bt

打印调用堆栈，加all可打印所有thread的堆栈。不详细举例说明，感兴趣的朋友可以自己试试。

* image

image 命令可用于寻址，有多个组合命令。比较实用的用法是用于寻找栈地址对应的代码位置。
下面我写了一段代码

	NSArray *arr=[[NSArray alloc] initWithObjects:@"1",@"2", nil];
	NSLog(@"%@",arr[2]);
这段代码有明显的错误，程序运行这段代码后会抛出下面的异常：

```  
*** Terminating app due to uncaught exception 'NSRangeException', reason: '*** -[__NSArrayI objectAtIndex:]: index 2 beyond bounds [0 .. 1]'
*** First throw call stack:
(
	0   CoreFoundation                      0x0000000101951495 __exceptionPreprocess + 165
	1   libobjc.A.dylib                     0x00000001016b099e objc_exception_throw + 43
	2   CoreFoundation                      0x0000000101909e3f -[__NSArrayI objectAtIndex:] + 175
	3   ControlStyleDemo                    0x0000000100004af8 -[RootViewController viewDidLoad] + 312
	4   UIKit                               0x000000010035359e -[UIViewController loadViewIfRequired] + 562
	5   UIKit                               0x0000000100353777 -[UIViewController view] + 29
	6   UIKit                               0x000000010029396b -[UIWindow addRootViewControllerViewIfPossible] + 58
	7   UIKit                               0x0000000100293c70 -[UIWindow _setHidden:forced:] + 282
	8   UIKit                               0x000000010029cffa -[UIWindow makeKeyAndVisible] + 51
	9   ControlStyleDemo                    0x00000001000045e0 -[AppDelegate application:didFinishLaunchingWithOptions:] + 672
	10  UIKit                               0x00000001002583d9 -[UIApplication _handleDelegateCallbacksWithOptions:isSuspended:restoreState:] + 264
	11  UIKit                               0x0000000100258be1 -[UIApplication _callInitializationDelegatesForURL:payload:suspended:] + 1605
	12  UIKit                               0x000000010025ca0c -[UIApplication _runWithURL:payload:launchOrientation:statusBarStyle:statusBarHidden:] + 660
	13  UIKit                               0x000000010026dd4c -[UIApplication handleEvent:withNewEvent:] + 3189
	14  UIKit                               0x000000010026e216 -[UIApplication sendEvent:] + 79
	15  UIKit                               0x000000010025e086 _UIApplicationHandleEvent + 578
	16  GraphicsServices                    0x0000000103aca71a _PurpleEventCallback + 762
	17  GraphicsServices                    0x0000000103aca1e1 PurpleEventCallback + 35
	18  CoreFoundation                      0x00000001018d3679 __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__ + 41
	19  CoreFoundation                      0x00000001018d344e __CFRunLoopDoSource1 + 478
	20  CoreFoundation                      0x00000001018fc903 __CFRunLoopRun + 1939
	21  CoreFoundation                      0x00000001018fbd83 CFRunLoopRunSpecific + 467
	22  UIKit                               0x000000010025c2e1 -[UIApplication _run] + 609
	23  UIKit                               0x000000010025de33 UIApplicationMain + 1010
	24  ControlStyleDemo                    0x0000000100006b73 main + 115
	25  libdyld.dylib                       0x0000000101fe95fd start + 1
	26  ???                                 0x0000000000000001 0x0 + 1
)
libc++abi.dylib: terminating with uncaught exception of type NSException
```
现在，我们怀疑出错的地址是0x0000000100004af8（可以根据执行文件名判断，或者最小的栈地址）。为了进一步精确定位，我们可以输入以下的命令：

	image lookup --address 0x0000000100004af8
命令执行后返回：

	Address: ControlStyleDemo[0x0000000100004af8] (ControlStyleDemo.__TEXT.__text + 13288)
    Summary: ControlStyleDemo`-[RootViewController viewDidLoad] + 312 at RootViewController.m:53
我们可以看到，出错的位置是`RootViewController.m`的第53行。  
-   -   -
更多的命令可以参见[这个网址](http://lldb.llvm.org/lldb-gdb.html)。  
另外，facebook开源了他们扩展的[LLDB命令库](https://github.com/facebook/chisel)，有兴趣的朋友也可以安装看看。
##简称和别名
很多时候，*LLDB*完整的命令是很长的。比如前面所说的`image lookup --address`这个组合命令。为了方便日常的使用，提高效率，*LLDB*命令也提供通过简称的方式调用命令。还是这个命令，我们用简称就可以写为`im loo -a`，是不是简单多了。  
如果你是从gdb时代就开始使用调试器的，你会发现，有些命令如`p`、`call`等命令和*gdb*下是一致的。其实这些命令是*LLDB*一些命令的别名，比如`p`是`frame variable`的别名，`p view`实际上是`frame variable view`。除了系统自建的*LLDB*别名，你也可以自定义别名。比如下面这个命令

	command alias ioa image lookup --address %1
是将我前面所介绍过的一个命令`image lookup --address`添加了一个`ioa`的别名。然后执行下面的命令：

	(lldb) ioa 0x0000000100004af8
      Address: ControlStyleDemo[0x0000000100004af8] (ControlStyleDemo.__TEXT.__text + 13288)
      Summary: ControlStyleDemo`-[RootViewController viewDidLoad] + 312 at RootViewController.m:53
可以看到，我们得到了我们想要的结果，而命令却大大缩短。    
这里我就不再详细展开，有兴趣的朋友可以查看[这个网址](http://lldb.llvm.org/tutorial.html)。
##常见问题
上面我们简单的学习了如何使用*LLDB*命令。但有时我们在使用这些*LLDB*命令的时候，依然可能会遇到一些问题。比如下面这个命令。  

	(lldb) p NSLog(@"%@",[self.view  viewWithTag:1001])
	error: 'NSLog' has unknown return type; cast the call to its declared return type
	error: 1 errors parsing expression
如果在使用*LLDB*命令中发现有 unknown type 的类似错误（多见于id类型，比如NSArray中某个值），那我们就必须显式声明类型。比如上面这个命令，我们得这么修改。

	p (void)NSLog(@"%@",[self.view  viewWithTag:1001])
这样就能得到正确的结果了。
##总结
通过上面一些简单的讲解，相信朋友们已经知道如何使用*LLDB*命令来提高自己的效率了。Enjoy it！