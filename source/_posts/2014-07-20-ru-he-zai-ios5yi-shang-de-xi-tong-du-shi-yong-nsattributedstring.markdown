---
layout: post
title: "如何在ios5以上的系统都使用NSAttributedString"
date: 2014-07-20 21:48
comments: true
categories: 
---

前几日遇到一个产品需求，需要将一段文本按下面的格式显示。    
![图一：示例格式](http://www.starfelix.com/images/attributedString_right.png)     
我马上想到的是用NSAttributedString，但是项目又得支持iOS5，怎么办呢？为了解决这个问题我做了以下几种尝试。
#方法一：CATextLayer
CATextLayer是iOS提供的，可以直接使用NSAttributedString而不需要自己处理渲染行为的类库。我尝试的代码如下：    
{% codeblock lang:objc %}  
    NSString *text=@"1.Hello Everyone!This is an article which introduce how to use NSAttributedString in iOS5.\n2.这段文字需要保持每行的缩进。为了实现这种效果，我们需要使用NSAttributedString.\n3.剩下的都是废话，凑字数用的。";
 
    CATextLayer *textLayer=[CATextLayer layer];
    textLayer.frame=UIEdgeInsetsInsetRect(self.view.bounds, UIEdgeInsetsMake(50, 20, 50, 20));
    textLayer.contentsScale=[UIScreen mainScreen].scale;                                //1.
    textLayer.wrapped=YES;                                                              //2.
    NSMutableAttributedString *attStr=[[NSMutableAttributedString alloc] initWithString:text];
    NSRange range=[text rangeOfString:@"字数"];
    [attStr addAttribute:(id)kCTForegroundColorAttributeName value:(__bridge id)([UIColor redColor].CGColor) range:range];      //3.
    [attStr addAttribute:(id)kCTFontAttributeName value:(__bridge id)(CTFontCreateWithName((CFStringRef)@"Helvetica", 14, NULL)) range:NSMakeRange(0, [text length])]; 
    CGFloat lineBreakMode=kCTLineBreakByWordWrapping;
    CGFloat firstLineIndent=0;
    CGFloat headIndent=[@"1." sizeWithFont:[UIFont systemFontOfSize:14]].width;
    CTParagraphStyleSetting paragraphStyles[3] = {                                          //4.
        {.spec = kCTParagraphStyleSpecifierLineBreakMode, .valueSize = sizeof(CTLineBreakMode), .value = (const void *)&lineBreakMode},
        {.spec = kCTParagraphStyleSpecifierFirstLineHeadIndent, .valueSize = sizeof(CGFloat), .value = (const void *)&firstLineIndent},
        {.spec = kCTParagraphStyleSpecifierHeadIndent, .valueSize = sizeof(CGFloat), .value = (const void *)&headIndent}
    };
    CTParagraphStyleRef paragraphStyle = CTParagraphStyleCreate(paragraphStyles, 3);       //5.
    [attStr addAttribute:(NSString *)kCTParagraphStyleAttributeName value:(__bridge id)paragraphStyle range:NSMakeRange(0, [text length])];
    CFRelease(paragraphStyle);
    textLayer.string=attStr;
    [self.view.layer addSublayer:textLayer];
{% endcodeblock %}

部分代码的解释：   
1. 设置 CATextLayer 的显示精细度。retina屏为2，非retina屏为1.    
2. 设置 CATextLayer 自动换行。    
3. 设置NSAttributedString在指定的范围(range)中字体使用红色。kCTForegroundColorAttributeName表示前景色的key，在这里可以理解为设置字体颜色的Key。    
4. CTParagraphStyleSetting 段落的格式。代码中设置了3个格式，分别表示为换行模式、首行缩进、每行缩进。    
5. 根据CTParagraphStyleSetting生成CTParagraphStyleRef。    

记得把attStr赋给textLayer.string，应用我们的格式化字符串。运行代码后显示效果如下：    
![图一：示例格式](http://www.starfelix.com/images/attributedString_wrong.png)     
很遗憾，显示不完全正确。虽然红色的部分显示正确了，但是缩进没有正确显示。我修改了一些参数，又上网找了一些资料。最后发现CATextLayer不支持段落的格式。。。

>注：如果遇到使用CATextLayer显示的中文字体不正确，那可能是因为系统对中文字体默认使用了日文字体。你可以在中文显示范围内指定“STHeitiSC-Light”这样就可以显示正常了。

#方法二——第三方库TTTAttributedLabel
最省事的办法现在行不通了，本着懒人的本性，考虑一下第三方库。[TTTAttributedLabel](https://github.com/mattt/TTTAttributedLabel)是大神Mattt写的一个用于显示富文本的库。具体使用详见Github的项目主页，我就不再赘述。我写的代码如下：
{% codeblock lang:objc %}  
    TTTAttributedLabel *label=[[TTTAttributedLabel alloc] initWithFrame:UIEdgeInsetsInsetRect(self.view.bounds, UIEdgeInsetsMake(50, 20, 50, 20))];
    label.numberOfLines=0;
    label.font=[UIFont systemFontOfSize:14];
    label.backgroundColor=[UIColor whiteColor];
    [label setText:text afterInheritingLabelAttributesAndConfiguringWithBlock:^(NSMutableAttributedString *string){
        CGFloat lineBreakMode=kCTLineBreakByWordWrapping;
        CGFloat firstLineIndent=0;
        CGFloat headIndent=[@"1." sizeWithFont:[UIFont systemFontOfSize:14]].width;
        CTParagraphStyleSetting paragraphStyles[3] = {
            {.spec = kCTParagraphStyleSpecifierLineBreakMode, .valueSize = sizeof(CTLineBreakMode), .value = (const void *)&lineBreakMode},
            {.spec = kCTParagraphStyleSpecifierFirstLineHeadIndent, .valueSize = sizeof(CGFloat), .value = (const void *)&firstLineIndent},
            {.spec = kCTParagraphStyleSpecifierHeadIndent, .valueSize = sizeof(CGFloat), .value = (const void *)&headIndent}
        };
        CTParagraphStyleRef paragraphStyle = CTParagraphStyleCreate(paragraphStyles, 12);
        [string addAttribute:(NSString *)kCTParagraphStyleAttributeName value:(__bridge id)paragraphStyle range:NSMakeRange(0, [text length])];
        CFRelease(paragraphStyle);
        NSRange range=[text rangeOfString:@"字数"];
        [string addAttribute:(id)kCTForegroundColorAttributeName value:(__bridge id)([UIColor redColor].CGColor) range:range];
        return string;
    }];
    [self.view addSubview:label];
{% endcodeblock %}
运行后显示结果如同方法一，依然无效。T_T   
#方法三——继承CATextLayer重写渲染方法
前面两个方法都行不通，最后我只能上网再找啊找，终于被我找到新的方法。虽然这个方法略繁琐，但是总比不能用的强。
首先我们需要新建一个继承于CATextLayer的类，然后重写他的渲染函数`drawInContext:`。（由于只是demo代码，所以我直接把字符串初始化的代码写到渲染函数里面了）    
{% codeblock lang:objc %}
- (void)drawInContext:(CGContextRef)ctx{
    CGContextSetFillColorWithColor(ctx, [[UIColor darkTextColor] CGColor]);
    UIGraphicsPushContext(ctx);
    
    NSString *text=@"1.Hello Everyone!This is an article which introduce how to use NSAttributedString in iOS5.\n2.这段文字需要保持每行的缩进。为了实现这种效果，我们需要使用NSAttributedString.\n3.剩下的都是废话，凑字数用的。";
    NSMutableAttributedString *attStr=[[NSMutableAttributedString alloc] initWithString:text];
    NSRange range=[text rangeOfString:@"字数"];
    [attStr addAttribute:(__bridge id)kCTForegroundColorAttributeName value:(id)([UIColor redColor].CGColor) range:range];      //3.
    [attStr addAttribute:(id)kCTFontAttributeName value:(__bridge id)(CTFontCreateWithName((CFStringRef)@"Helvetica", 14, NULL)) range:NSMakeRange(0, [text length])];      //3.
    CGFloat lineBreakMode=kCTLineBreakByWordWrapping;
    CGFloat firstLineIndent=0;
    CGFloat headIndent=[@"1." sizeWithFont:[UIFont systemFontOfSize:14]].width;
    CTParagraphStyleSetting paragraphStyles[3] = {                                          //4.
        {.spec = kCTParagraphStyleSpecifierLineBreakMode, .valueSize = sizeof(CTLineBreakMode), .value = (const void *)&lineBreakMode},
        {.spec = kCTParagraphStyleSpecifierFirstLineHeadIndent, .valueSize = sizeof(CGFloat), .value = (const void *)&firstLineIndent},
        {.spec = kCTParagraphStyleSpecifierHeadIndent, .valueSize = sizeof(CGFloat), .value = (const void *)&headIndent}
    };
    CTParagraphStyleRef paragraphStyle = CTParagraphStyleCreate(paragraphStyles, 3);       //5.
    [attStr addAttribute:(NSString *)kCTParagraphStyleAttributeName value:(__bridge id)paragraphStyle range:NSMakeRange(0, [text length])];
    [attStr drawInRect:self.bounds];
    
    UIGraphicsPopContext();
    [super drawInContext:ctx];
}  
{% endcodeblock %}
参照方法一初始化，然后运行代码。
What？运行中报错？
```
-[__NSCFType lineBreakMode]: unrecognized selector sent to instance 0x8f1bc50
```
这把我给苦恼的。继续上网找原因。原来是ios5以后不再使用C对象，而改使用OC对象来设置格式。我根据网上给出的信息，修改代码如下：    
{% codeblock lang:objc %}
- (void)drawInContext:(CGContextRef)ctx{
    CGContextSetFillColorWithColor(ctx, [[UIColor darkTextColor] CGColor]);
    UIGraphicsPushContext(ctx);
    
    NSString *text=@"1.Hello Everyone!This is an article which introduce how to use NSAttributedString in iOS5.\n2.这段文字需要保持每行的缩进。为了实现这种效果，我们需要使用NSAttributedString.\n3.剩下的都是废话，凑字数用的。";
    NSMutableAttributedString *attStr=[[NSMutableAttributedString alloc] initWithString:text];
    NSRange range=[text rangeOfString:@"字数"];
    if ([NSParagraphStyle class]) {
        [attStr addAttribute:NSForegroundColorAttributeName value:[UIColor redColor] range:range];
        [attStr addAttribute:NSFontAttributeName value:[UIFont systemFontOfSize:14] range:NSMakeRange(0, [text length])];
        NSMutableParagraphStyle *paragraphStyle=[[NSMutableParagraphStyle defaultParagraphStyle] mutableCopy];
        paragraphStyle.firstLineHeadIndent=0;
        paragraphStyle.headIndent=[@"1." sizeWithFont:[UIFont systemFontOfSize:14]].width;
        [attStr addAttribute:NSParagraphStyleAttributeName value:paragraphStyle range:NSMakeRange(0, [text length])];
    }else{
        [attStr addAttribute:(__bridge id)kCTForegroundColorAttributeName value:(id)([UIColor redColor].CGColor) range:range];      //3.
        [attStr addAttribute:(id)kCTFontAttributeName value:(__bridge id)(CTFontCreateWithName((CFStringRef)@"Helvetica", 14, NULL)) range:NSMakeRange(0, [text length])];      //3.
        CGFloat lineBreakMode=kCTLineBreakByWordWrapping;
        CGFloat firstLineIndent=0;
        CGFloat headIndent=[@"1." sizeWithFont:[UIFont systemFontOfSize:14]].width;
        CTParagraphStyleSetting paragraphStyles[3] = {                                          //4.
            {.spec = kCTParagraphStyleSpecifierLineBreakMode, .valueSize = sizeof(CTLineBreakMode), .value = (const void *)&lineBreakMode},
            {.spec = kCTParagraphStyleSpecifierFirstLineHeadIndent, .valueSize = sizeof(CGFloat), .value = (const void *)&firstLineIndent},
            {.spec = kCTParagraphStyleSpecifierHeadIndent, .valueSize = sizeof(CGFloat), .value = (const void *)&headIndent}
        };
        CTParagraphStyleRef paragraphStyle = CTParagraphStyleCreate(paragraphStyles, 3);       //5.
        [attStr addAttribute:(NSString *)kCTParagraphStyleAttributeName value:(__bridge id)paragraphStyle range:NSMakeRange(0, [text length])];
    }
    [attStr drawInRect:self.bounds];
    
    UIGraphicsPopContext();
    [super drawInContext:ctx];
}    
{% endcodeblock %}
运行代码，运行没报错，运行结果如下：    
![图一：示例格式](http://www.starfelix.com/images/attributedString_right.png)     
这回终于是正确的了！    