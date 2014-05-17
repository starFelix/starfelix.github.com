---
layout: post
title: "正确编写Designated Initializer的几个原则"
date: 2014-04-13 22:24
comments: true
categories: 
---

Designated Initializer(指定初始化器)在Objective－C里面是很重要的概念，但是在日常开发中我们往往会忽视它的重要性，以至于我们写出的代码具有潜藏的Bug，且不易发现。保证良好的编写Designated Initializer的风格，可以让我们节约很多时间。
前段时间[@吴发伟Ted](http://weibo.com/wufawei)分享了一篇Twitter团队的一篇博客，里面讲述了Designated Initializer正确的模板以及需要注意的问题。但是里面关于`initWithCoder`描述不是很清晰，且随后[@an00na](http://weibo.com/an00na)给出了[不同的看法](http://wangling.me/2014/03/objective-c-initializer-design-rules.html)。我会在接下来的文章讲述验证他们给出的编写Designated Initializer的原则，并对`initWithCoder`的分歧做一个分析，了解其背后的机制。


#准备工作
为了能够跟踪代码的实际调用顺序，在下面的实例分析中，我将会使用`Xtrace`。这是一个在调试上非常强大的一个库，实现原理是通过Hook的方式跟踪消息。详细可以看[Github上的说明](https://github.com/johnno1962/Xtrace)。
需要注意的是，Xtrace里面设定了不跟踪`initWithCoder`，由于我们后面分析的需要我们需要把`Xtrace.h`里面的一段代码做一些小改动：   
{% codeblock lang:objc %}
#define XTRACE_EXCLUSIONS "^(initWithCoder|_UIAppearance_|"\
    "_(initializeFor|performUpdatesForPossibleChangesOf)Idiom:|"\
    "timeIntervalSinceReferenceDate)|(WithObjects(AndKeys)?|Format):$"
{% endcodeblock %}   

这段代码标示了不跟踪的@selector，我们需要将`initWithCoder`删除，才能跟踪这个方法。   
接下来，我们需要在`AppDelegate.m`里面加入一段代码：  
 
{% codeblock lang:objc %}
- (id)init{
    if (self=[super init]){
        [Xtrace includeMethods:@"^init"];//1.
        [Xtrace traceClass:[ViewController class]];
        [Xtrace traceClass:[TestInitView class]];
        [Xtrace traceClass:[SubTestInitView class]];
        [Xtrace traceClass:[NSObject class]];//2.
    }
    return self;
}
{% endcodeblock %}

1. 由于每个类对象调用的方法很多，为了不被干扰，声明我们只跟踪以`init`开头的方法。  
2. 这几行声明了我们将会跟踪的类。也就是说，一旦这些类调用了我们跟踪的方法，就会有信息输出。

#分析代码
我会先给出我认为应该遵循的原则，并对每个原则做实际分析。  

* **每个类的正确初始化过程应当是按照从子类到父类的顺序，依次调用每个类的Designated Initializer。并且用父类的Designated Initializer初始化一个子类对象，也需要遵从这个过程。**

* **如果子类指定了新的初始化器，那么在这个初始化器内部必须调用父类的Designated Initializer。并且需要重写父类的Designated Initializer，将其指向子类新的初始化器**

`TestInitView`是一个继承于UIView，它重新指定的初始化器为`initWithFrame:andName:`。现在，假设这个类的初始化器如下：    
{% codeblock lang:objc %}    
//Designated Initializer
- (instancetype)initWithFrame:(CGRect)frame andName:(NSString *)name{
	//incorrect
    if (self = [super init]){
        self.name=name;
    }
    return self;
}    
{% endcodeblock %}
可以看到在里面并没有调用UIView的Designated Initializer`initWithFrame:`。那么会有什么后果呢？   
我们用这个Designated Initializer生成一个TestInitView对象：    
{% codeblock lang:objc %}  
TestInitView *testView = [[TestInitView alloc] initWithFrame:CGRectZero andName:@""];
{% endcodeblock %}
运行程序，我们会看到Xtrce的跟踪记录如下：    

```
| [<TestInitView 0x8c6c840> initWithFrame:{} andName:<__NSCFConstantString 0x6d278>]
|  [<TestInitView 0x8c6c840>/UIView init]
|   [<TestInitView 0x8c6c840>/UIView initWithFrame:{}]
|    [<TestInitView 0x8c6c840>/NSObject init]
|    -> <TestInitView 0x8c6c840> (init)
|  -> <TestInitView 0x8c6c840> (initWithFrame:)
| -> <TestInitView 0x8c6c840> (init)
| -> <TestInitView 0x8c6c840> (initWithFrame:andName:)
```  

咦，似乎没有问题啊。整个继承链的初始化器都被调用了。等等，如果我们用UIView的Designated Initializer生成一个TestInitView对象会怎样呢？
{% codeblock lang:objc %}  
TestInitView *testView = [[TestInitView alloc] initWithFrame:CGRectZero];
{% endcodeblock %}
运行代码后，我们得到的调用过程如下：  

```
| [<TestInitView 0xa16cd40>/UIView initWithFrame:{}]
|  [<TestInitView 0xa16cd40>/NSObject init]
|  -> <TestInitView 0xa16cd40> (init)
| -> <TestInitView 0xa16cd40> (initWithFrame:)
```   

这时会发现TestInitView的初始化器`initWithFrame:andName:`没有被调用。  
我们再修改下代码：  
{% codeblock lang:objc %}    
//Designated Initializer
- (instancetype)initWithFrame:(CGRect)frame andName:(NSString *)name{
	//incorrect
    if (self = [super initWithFrame:frame]){
        self.name=name;
    }
    return self;
}    
{% endcodeblock %}  
我们依然使用UIView的Designated Initializer，然后运行程序得到下面的结果：

```
| [<TestInitView 0xa81d3f0>/UIView initWithFrame:{}]
|  [<TestInitView 0xa81d3f0>/NSObject init]
|  -> <TestInitView 0xa81d3f0> (init)
| -> <TestInitView 0xa81d3f0> (initWithFrame:)
```   

TestInitView的初始化器依然没有被调用。原因就是没有我们没有重写父类UIView的Designated Initializer。修改后我们的最终代码如下：   
{% codeblock lang:objc %}    
//Designated Initializer
- (instancetype)initWithFrame:(CGRect)frame andName:(NSString *)name{
	//incorrect
    if (self = [super initWithFrame:frame]){
        self.name=name;
    }
    return self;
}    
//super override
- (id)initWithFrame:(CGRect)frame
{
    return [self initWithFrame:frame andName:@""];
}
{% endcodeblock %}  
继续测试我们的代码，得到的结果如下：   

```
| [<TestInitView 0xa549920> initWithFrame:{}]
|  [<TestInitView 0xa549920> initWithFrame:{} andName:<__NSCFConstantString 0x6ce2c>]
|   [<TestInitView 0xa549920>/UIView initWithFrame:{}]
|    [<TestInitView 0xa549920>/NSObject init]
|    -> <TestInitView 0xa549920> (init)
|  -> <TestInitView 0xa549920> (initWithFrame:)
| -> <TestInitView 0xa549920> (initWithFrame:andName:)
| -> <TestInitView 0xa549920> (initWithFrame:)
```    

OK,运行正确！

* **你可以不自定义Designated Initializer，也可以重写父类的Designated Initializer，但需要调用直接父类的Designated Initializer。**

也就是说，上面的代码，我们可以不写Designated Initializer，也可以这么修改： 
   
{% codeblock lang:objc %}    
//Designated Initializer
- (instancetype)initWithFrame:(CGRect)frame{
	//incorrect
    if (self = [super initWithFrame:frame]){
        self.name=name;
    }
    return self;
}    
{% endcodeblock %}  

但是下面这样写就是错误的，因为调用的不是直接父类的Designated Initializer   

{% codeblock lang:objc %}    
//Designated Initializer
- (instancetype)initWithFrame:(CGRect)frame{
	//incorrect
    if (self = [super init]){
        self.name=name;
    }
    return self;
}    
{% endcodeblock %}  

* **如果有多个Secondary initializers(次要初始化器)，它们之间可以任意调用，但最后必须指向Designated Initializer。在Secondary initializers内不能直接调用父类的初始化器。**
这句话可能看着有点糊涂，还是让们继续看看代码：   

{% codeblock lang:objc %}    
#import "TestInitView.h"

@implementation TestInitView

//super override
- (id)initWithFrame:(CGRect)frame
{
    return [self initWithFrame:frame andName:@""];
}

//Designated Initializer
- (instancetype)initWithFrame:(CGRect)frame andName:(NSString *)name{
    if (self = [super initWithFrame:frame]){
        self.name=name;
    }
    return self;
}

//Instance secondary initializer
- (instancetype)initWithName:(NSString *)name{
    return [self initWithFrame:CGRectZero andName:name];
}

//Instance secondary initializer
- (instancetype)initWithName2:(NSString *)name{
    return [self initWithName:name];
}

//Class secondary initializer
+ (instancetype)testInitViewWithName:(NSString *)name{
    TestInitView *testView=[[TestInitView alloc] initWithFrame:CGRectZero andName:name];
    if (testView){
        //do something here
    }
    return testView;
}

@end  
{% endcodeblock %}   

可以看到方法`initWithName2`调用的顺序是`initWithName2`->`initWithName`->`initWithFrame:andName:`，最后指向了Designated Initializer。同时需要注意的是，**Secondary initializers不仅可以是实例方法，也可以是静态方法**，如代码实例中的`testInitViewWithName:`。

#initWithCoder:

文章开头我就提到两篇引用文章关于initWithCoder的分歧。
首先，如果直接父类也实现了NSCoding协议，那么子类的`initWithCoder`也必须调用父类的`initWithCoder`。  
这点大家的观点都是相同的，[苹果官方文档](https://developer.apple.com/library/mac/documentation/cocoa/Conceptual/Archiving/Articles/codingobjects.html#//apple_ref/doc/uid/20000948-BCIHBJDE)也是这么说的。    
主要的分歧在于当直接父类不实现NSCoding时，子类的`initWithCoder`应该调用哪个初始化器。

Twitter团队的那片博客给出的看法是：   
>There’s a problem with the example provided in the documentation for initWithCoder:, specifically the call to [super (designated initializer)]. If you’re a direct subclass of NSObject, calling [super (designated initializer)] won’t call your [self (designated initializer)]. **If you’re not a direct subclass of NSObject, and one of your ancestors implements a new designated initializer, calling [super (designated initializer)] WILL call your [self (designated initializer)].** This means that apple’s suggestion to call super in initWithCoder encourages non-deterministic initialization behavior, and is not consistent with the solid foundations laid by the designated initializer pattern. **Therefore, my recommendation is that you should treat initWithCoder: as a secondary initializer, and call [self (designated initializer)], not [super (designated initializer)], if your superclass does not conform to NSCoding.**    

直接理解有点困难，我们用代码来说话。我们新建一个类`SubTestInitView`,它继承于`TestInitView`，且它实现NSCoding协议。  
 
{% codeblock lang:objc %} 
#import "SubTestInitView.h"

@implementation SubTestInitView

- (instancetype)initWithFrame:(CGRect)frame andName:(NSString *)name{
    if (self = [super initWithFrame:frame andName:name]){
        self.subName=name;
    }
    return self;
}

- (id)initWithCoder:(NSCoder *)aDecoder{
    if (self=[super initWithFrame:CGRectZero andName:@""]){
    
    }
    return self;
}

- (void)encodeWithCoder:(NSCoder *)aCoder{
    
}

@end
{% endcodeblock %} 

可以看到，我们在`initWithCoder:`里面调用的是TestInitView的Designated Initializer。让我们看看调用顺序(P.S. 在xib里面拖入一个View，然后指定其类型为SubTestInitView，这样就会调用其`initWithCoder:`)：   

```
| [<SubTestInitView 0xa00a8c0> initWithCoder:<UINibDecoder 0x990fa00>]
|  [<SubTestInitView 0xa00a8c0>/TestInitView initWithFrame:{} andName:<__NSCFConstantString 0x6df0c>]
|   [<SubTestInitView 0xa00a8c0>/UIView initWithFrame:{}]
|    [<SubTestInitView 0xa00a8c0>/NSObject init]
|    -> <SubTestInitView 0xa00a8c0> (init)
|  -> <SubTestInitView 0xa00a8c0> (initWithFrame:)
| -> <SubTestInitView 0xa00a8c0> (initWithFrame:andName:)
| -> <SubTestInitView 0xa00a8c0> (initWithCoder:)
```  
根据引用的内容，其认为会调用顺序中有[self (designated initializer)]，也就是SubTestInitView 的`- (instancetype)initWithFrame:(CGRect)frame andName:(NSString *)name`，但是实际结果并不是这样的。子类的Designated Initializer并没有被调用。而最后其建议将`initWithCoder:`视为Secondary initializers，然后在里面调用[self (designated initializer)]。这样做的目的是为了保证[self (designated initializer)]会被调用，以保证初始化是正确的。   
对此，[@an00na](http://weibo.com/an00na)则提出不一致的看法。   
[@an00na](http://weibo.com/an00na)引入了数据源的概念，他认为`initWithCoder:`是用另一个数据源NSDecoder初始化的。一个类可以由多个数据源初始化，也就是说可以有多个Designated Initializer。因此，`initWithCoder`是Designated Initializer，而根据我们前面提到的原则，我们应该调用父类的Designated Initializer。   
这个时候，我产生了疑惑。按照这个说法，此时应该如代码所示，调用父类TestInitView的`initWithFrame:andName:`明显有问题，因为并非是同个数据源。可是父类并没有实现NSCoding，我们可以调用他的`initWithCoder`(实际上是UIView的)吗？   
我注意到UIButton的继承顺序是UIButton->UIControl->UIView...，UIControl同样没有实现NSCoding，那么对于这种情况，apple又是如何做的呢？我跟踪了UIButton的生成过程：    

```
| [<UIButton 0x9c7e400> initWithFrame:{}]
|  [<UIButton 0x9c7e400>/UIControl initWithFrame:{}]
|   [<UIButton 0x9c7e400>/UIView initWithFrame:{}]
|    [<UIButton 0x9c7e400>/NSObject init]
|    -> <UIButton 0x9c7e400> (init)
|  -> <UIButton 0x9c7e400> (initWithFrame:)
| -> <UIButton 0x9c7e400> (initWithFrame:)
| -> <UIButton 0x9c7e400> (initWithFrame:)
2014-04-16 01:36:12.948 BlogDemo[2128:60b] -------------
| [<UIButton 0x8eba7f0> initWithCoder:<UINibDecoder 0x9373600>]
|  [<UIButton 0x8eba7f0>/UIControl initWithCoder:<UINibDecoder 0x9373600>]
|   [<UIButton 0x8eba7f0>/UIView initWithCoder:<UINibDecoder 0x9373600>]
|    [<UIButton 0x8eba7f0>/NSObject init]
|    -> <UIButton 0x8eba7f0> (init)
|  -> <UIButton 0x8eba7f0> (initWithCoder:)
| -> <UIButton 0x8eba7f0> (initWithCoder:)
| -> <UIButton 0x8eba7f0> (initWithCoder:)
```

间隔线上面是以`[UIButton buttonWithType:UIButtonTypeCustom]`方式初始化的UIButton，下面的则是通过xib初始化的UIButton。我们可以看到，下面的初始化顺序并没由调用到`initWithFrame:`,这也证明了[@an00na](http://weibo.com/an00na)关于数据源的看法。而同时可以发现中间调用了UIControl的`initWithCoder:`!   
于是，在这里，我认为如果你的自定义类的其中一个父类实现了NSCoding协议，而且你有扩展属性，虽然你本身没有声明会实现NSCoding协议，你也需要实现NSCoding的两个方法：`initWithCoder:`和`encodeWithCoder:`,并对你扩展的属性做coding操作。    
给TestInitView补上这两个方法，SubTestInitView的`initWithCoder`也调用父类的`initWithCoder`后，让我们再看看实际的调用顺序：   

```
| [<SubTestInitView 0xa664e70> initWithCoder:<UINibDecoder 0xd880600>]
|  [<SubTestInitView 0xa664e70>/TestInitView initWithCoder:<UINibDecoder 0xd880600>]
|   [<SubTestInitView 0xa664e70>/UIView initWithCoder:<UINibDecoder 0xd880600>]
|    [<SubTestInitView 0xa664e70>/NSObject init]
|    -> <SubTestInitView 0xa664e70> (init)
| -> <SubTestInitView 0xa664e70> (initWithCoder:)
| -> <SubTestInitView 0xa664e70> (initWithCoder:)
| -> <SubTestInitView 0xa664e70> (initWithCoder:)
```  

#多数据源

经过上面的探讨，我们了解了多数据源的设计模型。但是这带来了另外一个问题：如何使得通过不同数据源初始化动作是一致的？   
比如说，我们往往需要再对象初始化紧接着做一些动作。由于多数据源的存在，并且不同的数据源提供的数据内容不一样，我们应当如何处理？   
很自然的，我们可能会想到讲部分动作重构成一个方法，无论从哪个数据源初始化，都调用这个方法。    
如果是自己编写的不同数据源的Designated Initializer，具体调用这个方法的时机需要自己把握。我在这里着重讲一下UIView或UIViewController的子类在处理多数据源需要注意的问题。   
按前面所述，UIView和UIViewController的子类需要实现NSCoding的两个方法。但是我们需要注意的是，同样是调用initWithCoder，却仍然可能是不同的数据源：UINibDecoder或者NSKeyedUnarchiver。    
这两者分别代表什么情况呢？UINibDecoder是指xib初始化，而NSKeyedUnarchiver是指通过归档文件(如plist/txt等物理文件)初始化。它们最大的区别是UINibDecoder仅包含了xib里面能够设置的数据，而NSKeyedUnarchiver包括了整个对象的数据，也就是说如果是从NSKeyedUnarchiver初始化的，是不再需要额外的初始化动作了。而如果是从UINibDecoder初始化的，则在对象初始化后，会调用其`awakeFromNib`方法，我们需要把额外的初始化动作写在这里。   
根据这个，我再次改写了TestInitView如下：    

{% codeblock lang:objc %}
#import "TestInitView.h"

@implementation TestInitView

//super override
- (id)initWithFrame:(CGRect)frame
{
    return [self initWithFrame:frame andName:@""];
}

//Designated Initializer
- (instancetype)initWithFrame:(CGRect)frame andName:(NSString *)name{
    if (self = [super initWithFrame:frame]){
        [self someInit];
        self.name=name;
        //correct From Name
    }
    return self;
}

- (id)initWithCoder:(NSCoder *)aDecoder{
    if (self=[super initWithCoder:aDecoder]) {
        self.name=[aDecoder decodeObjectForKey:@"name"];
        //correct From NSKeyedUnarchiver
    }
    return self;
}

- (void)encodeWithCoder:(NSCoder *)aCoder{
    [super encodeWithCoder:aCoder];
    [aCoder encodeObject:self.name forKey:@"name"];
}

- (void)awakeFromNib{
    [super awakeFromNib];
    [self someInit];
    //correct From UINibDecoder
}

- (void)someInit{
    self.name=@"";
    //anyelse....
}

@end
{% endcodeblock %} 
这样我们就可以保证无论是从什么数据源初始化，得出的结果都是正确的了。事实上，如果在工程中我们没有对UIView或UIViewController的子类对象有归档的需求的话，NSCoding的两个方法可以省去不写。   

#小技巧    
有时候我们可能会忘记这些需要注意的地方，幸运的是，Xcode为我们提供了一个很棒的编译警告。
在合适的文件前面加入下面一段宏：   

{% codeblock lang:objc %}
#ifndef NS_DESIGNATED_INITIALIZER
#if __has_attribute(objc_designated_initializer)
#define NS_DESIGNATED_INITIALIZER __attribute((objc_designated_initializer))
#else
#define NS_DESIGNATED_INITIALIZER
#endif
#endif
{% endcodeblock %} 

然后在你需要标示为Designated Initializer的方法后面加上 NS_DESIGNATED_INITIALIZER。如下：    
{% codeblock lang:objc %}
//Designated Initializer
- (instancetype)initWithFrame:(CGRect)frame andName:(NSString *)name NS_DESIGNATED_INITIALIZER;
//Instance secondary initializer
- (instancetype)initWithName:(NSString *)name __attribute((objc_designated_initializer));
//Class secondary initializer
+ (instancetype)testInitViewWithName:(NSString *)name;
{% endcodeblock %}   
这样，当你没有按照规则的时候，会得到下面的警告：     
![图一：没有调用默认初始化器警告](http://www.starfelix.com/images/design-warning.png) 

#综述
经过一番长篇大论:)，我们可以总结正确编写Designated Initializer的原则如下：   

1. 每个类的正确初始化过程应当是按照从子类到父类的顺序，依次调用每个类的Designated Initializer。并且用父类的Designated Initializer初始化一个子类对象，也需要遵从这个过程。
2. 如果子类指定了新的初始化器，那么在这个初始化器内部必须调用父类的Designated Initializer。并且需要重写父类的Designated Initializer，将其指向子类新的初始化器。
3. 你可以不自定义Designated Initializer，也可以重写父类的Designated Initializer，但需要调用直接父类的Designated Initializer。
4. 如果有多个Secondary initializers(次要初始化器)，它们之间可以任意调用，但最后必须指向Designated Initializer。在Secondary initializers内不能直接调用父类的初始化器。
5. 如果有多个不同数据源的Designated Initializer，那么不同数据源下的Designated Initializer应该调用相应的[super (designated initializer)]。如果父类没有实现相应的方法，则需要根据实际情况来决定是给父类补充一个新的方法还是调用父类其他数据源的Designated Initializer。比如UIView的`initWithCoder`调用的是NSObject的`init`。
6. 需要注意不同数据源下添加额外初始化动作的时机。

P.S.尽管我做了很多实例分析研究，但经验所限可能仍有不足或不对之处，欢迎指正。本文内大量内容来源Twitter团队博客的那篇文章以及[@an00na](http://weibo.com/an00na)的文章，我仅仅是用中文做了讲述，在此表示十分感谢！