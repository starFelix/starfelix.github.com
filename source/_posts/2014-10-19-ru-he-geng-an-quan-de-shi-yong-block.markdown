---
layout: post
title: "如何更安全的使用Block"
date: 2014-10-19 17:16
comments: true
categories: 
---
前段时间，在公司的项目中发现了内存泄露。查到最后发现是由于没有正确使用Block导致的内存引用循环。简单的解决方案就是补上关键字`__weak`。那么问题来了，为何[UIView animateWithDuration:animations:completion:]或者GCD中可以不需要声明__weak呢？    
![导语](http://www.starfelix.com/images/crazy.png)    
#如何选择回调方式    
在讲这个问题之前，我们先看看Objcio上是怎么判断使用何种回调方式：
![选择](http://img.objccn.io/issue-7/communication-patterns-flow-chart.png)    
从这张图可以看到选择block的回掉方式的前提条件是sender能保证将callback被设为nil。定这个条件的目的是为了避免内存泄露。为何这样可以避免内存泄露呢？我们先看看通常我们是如何解决由使用block引入的内存引用循环问题。

#引用循环   
下面是一个内存引用循环的例子    
{% codeblock lang:objc %}
typedef void(^my_completion_block_t)(NSArray* result);

@interface UsersViewController : UIViewController
@property (nonatomic, copy) my_completion_block_t completion;
@property (nonatomic) NSArray* users;
@end
{% endcodeblock %}    
这里可以看到，`UsersViewController`持有一个block的强引用。如果一个不小心，我们会写出下面这样的代码：    
{% codeblock lang:objc %}
self.completion = ^(NSArray* users){
    self.users = users;
}
[self fetchUsers];
{% endcodeblock %}    
self持有了对completion的强引用，而completion的代码块内引用了self会引起对self的强引用，这便是一个经典的引用循环的案例。    
在MRC下，`__block id x;`可以让block不强引用x。在ARC下，`__block id x;`默认的行为会强引用x。为了在ARC下解决这个问题，你可以使用`__unsafe_unretained __block id x;`。但是，这样做非常危险，因为当你的block里访问x时，x可能已经被释放了，变成野指针。你有两个更好的解决办法，一是换成使用`__weak`，二是在block内设置`__block`值为nil来打破引用循环。  
如下：  
1.使用__weak来表示对self的弱引用（代码里面不直接使用weakSelf的原因是为了在block里面保持对self的强引用，避免代码运行到一半self被释放）    
{% codeblock lang:objc %}
UsersViewController* __weak weakSelf = self;
self.completion = ^(NSArray* users) {
    UsersViewController* strongSelf = weakSelf;
    if (strongSelf) {
        strongSelf.users = users;
    }
    else {
        // the view controller does not exist anymore
    }
}  
[usersViewController fetchUsers];
{% endcodeblock %}    
2.使用__block关键字修饰self，然后在block里设置为nil  
{% codeblock lang:objc %}
UsersViewController* __block blockSelf = self;
self.completion = ^(NSArray* users) {
    blockSelf.users = users;
    blockSelf = nil;
}   
[usersViewController fetchUsers];
{% endcodeblock %}    

#更友好的方式
这个时候我们的疑问来了，既然都可以解决内存循环引用的问题，为何还会有前面这个条件呢？
原因就是这种方式严重依赖调用者的使用方式，在日常使用中带来严重的代码质量隐患。而在苹果的sdk中几个使用block的方法中调用者都不需要特别的处理。
下面是一个测试代码：  
{% codeblock lang:objc %}
- (void)viewDidLoad
{
    [super viewDidLoad];
    NSString *temp=[[NSString alloc] initWithFormat:@"%@",@"我是temp"];
    NSLog(@"%p",temp);
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 30 * NSEC_PER_SEC), dispatch_get_main_queue(), ^{
        NSLog(@"%@",temp);
    });
    [temp release];
}
{% endcodeblock %}    
上面代码的作用是进入一个controller后然后延时30s打印一个log，为了方便分析内存，我们使用的是mrc。使用Instructment分析内存如下：   
![内存1](http://www.starfelix.com/images/memory1.png)
从图片我们可以看到，temp是在30s后被释放。从Responsible Caller我们可以得出结论，block被持有，然后在30s触发执行后被释放。block中被capture的变量也被释放。如此也避免了内存问题。
把代码修改一下：  
{% codeblock lang:objc %}
- (void)viewDidLoad
{
    [super viewDidLoad];
    NSString *temp=[[NSString alloc] initWithFormat:@"%@",@"我是temp"];
    NSLog(@"%p",temp);
    __block NSString *temp2=temp;
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 30 * NSEC_PER_SEC), dispatch_get_main_queue(), ^{
        NSLog(@"%@",temp2);
    });
    [temp release];
}
{% endcodeblock %}    
同样使用Instructment分析，结果等到NSLog代码执行的时候代码crash了，内存记录如下：  
![内存2](http://www.starfelix.com/images/memory2.png)   
这说明，如果我们滥用__block，可能会出现在block运行的时候，访问对象已经被释放，造成访问野指针错误。
系统SDK向我们展示的是一种更友好使用block的方式，让调用者不需要担心内存引用循环也不会有访问野指针的风险。

#结论  
1.block的持有方式必须保证是copy，这样block中capture的对象才会被retain，保证block在执行的时候capture对象不会被提前释放。  
2.为了避免引用循环，被copy的block必须保证在某个时刻被释放，以破坏引用循环。使得所有对象能被正常的释放。