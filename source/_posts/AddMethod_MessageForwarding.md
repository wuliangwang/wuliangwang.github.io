---
title: iOS - 动态添加方法和消息转发
date:2017-12-15
---

#####假设
假设，声明一个 `TestObect` 的类，.h中声明了 `test `方法,但是.m中未实现，但是调用了这个方法会报一个经典的错误
![没有找到方法实现](http://upload-images.jianshu.io/upload_images/3111822-d2d8794b21d502e5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

ok，前提已经说完了，我们就从找这个错误原因讲起。
##### 解决
首先，该方法在调用时，系统会查看这个对象能否接收这个消息（查看这个类有没有这个方法，或者有没有实现这个方法。），如果不能并且只在不能的情况下，就会调用下面这几个方法，给你“补救”的机会，你可以先理解为几套防止程序crash的备选方案，我们就是利用这几个方案进行消息转发，注意一点，前一套方案实现后一套方法就不会执行。如果这几套方案你都没有做处理，那么程序就会报错crash。

```
方案一：
+ (BOOL)resolveInstanceMethod:(SEL)sel；
方案二：
- (id)forwardingTargetForSelector:(SEL)aSelector;
方案三：
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector;
- (void)forwardInvocation:(NSInvocation *)anInvocation;
```
抽象解释一下消息转发: 比赛足球时，脚下有球的那名球员，如果他的位置不利于射门或者他的球即将被对方球员抢断，这时最好是把球传出去，这里的球就相当于消息。

###### 1.resolveInstanceMethod
当你调用方法时，系统会调用 `resolveInstanceMethod ` 这个方法,参数SEL就是你调用的方法，那么我们可以重写这个方法，从而实现动态添加一个方法，来使程序不会崩溃.
```
// 需要导入 #import <objc/runtime.h>
+ (BOOL)resolveInstanceMethod:(SEL)sel{
    if (sel == @selector(test)) {        
        IMP imp = class_getMethodImplementation(self, @selector(run));
        class_addMethod([self class], sel, imp, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}
```

![动态添加方法](http://upload-images.jianshu.io/upload_images/3111822-20e65edd94ef9259.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


#####科普一下
1. IMP获取
如果添加方法实现是用OC风格写的 那么用 `class_getMethodImplementation` 来获取IMP
如果是用C风格写的函数 
```
void run(id self, SEL _cmd) {
    NSLog(@" %@ %s", sel_getName(_cmd));
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == @selector(test)) {
        class_addMethod([self class], sel, (IMP)run, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}
```

2. class_addMethod 参数
- 参数一: Class：我们需要一个class，比如我的[self class]。 
- 参数二: SEL：方法
- 参数三: IMP ：IMP就是Implementation的缩写，它是指向一个方法实现的指针，每一个方法都有一个对应的IMP。这里需要的是IMP，所以你不能直接写方法，参照上述 1 来获取
- 参数四: const char *types：
”v@:”意思就是这已是一个void类型的方法，没有参数传入。 
 “i@:”就是说这是一个int类型的方法，没有参数传入。 
”i@:@”就是说这是一个int类型的方法，有一个对象参数传入。 
因为每一个方法会默认隐藏两个参数，self、_cmd，self代表方法调用者，_cmd代表这个方法的SEL，该参数就是用来描述这个方法的返回值、参数的，`v` 代表返回值为void，`@` 表示self，`:` 表示_cmd。
 [详见官网](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html)

- 注意: 用这个方法添加的方法是无法直接调用的，必须用performSelector：调用。
因为performSelector是运行时系统负责去找方法的，在编译时候不做任何校验；如果直接调用编译是会自动校验。 然后class_addMethod添加方法是在运行时添加的，你在编译的时候还没有这个本类方法，所以当然不行啦。


###### 2.forwardingTargetForSelector
forwardingTargetForSelector 是NSObject的函数，来决定谁执行方法
```
TestObect.m
+ (BOOL)resolveInstanceMethod:(SEL)sel{
    return [super resolveInstanceMethod:sel];
}

- (id)forwardingTargetForSelector:(SEL)aSelector{
    if (aSelector == @selector(test)) {
        return [ForwardTestObect new];
    }
    return [super forwardingTargetForSelector:aSelector];
}

ForwardTestObect.m
- (void)test{
    NSLog(@"%@  --- test ---",self);
}
```
![打印截图](http://upload-images.jianshu.io/upload_images/3111822-1f48606b8325ce2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


###### 3.methodSignatureForSelector和forwardInvocation

methodSignatureForSelector用来生成方法签名，这个签名就是给forwardInvocation中的参数NSInvocation调用的。
开头我们要找的错误unrecognized selector sent to instance原因，原来就是因为methodSignatureForSelector这个方法中，由于没有找到run对应的实现方法，所以返回了一个空的方法签名，最终导致程序报错崩溃。
```
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector{
    if (aSelector == @selector(test)) {
        return [NSMethodSignature signatureWithObjCTypes:"v@:"];
    }
    return [super methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)anInvocation{
    SEL sel = [anInvocation selector];
    //创建接受对象
    ForwardTestObect *forward = [ForwardTestObect new];
    //判断是否实现了这个方法
    if ([forward respondsToSelector:sel]) {
        // 唤醒这个方法
        [anInvocation invokeWithTarget:forward];
    }else{
        [super forwardInvocation:anInvocation];
    }
}
```
![打印截图](http://upload-images.jianshu.io/upload_images/3111822-1b0c05b9a7a37da4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中: [NSMethodSignature signatureWithObjCTypes:"v@:"]; 方法的参数参考上述 class_addMethod 方法的参数四
forwardInvocation: 方法就是一个不能识别消息的分发中心，将这些不能识别的消息转发给不同的接收对象，或者转发给同一个对象，再或者将消息翻译成另外的消息，亦或者简单的“吃掉”某些消息，因此没有响应也不会报错。这一切都取决于方法的具体实现。
注意:forwardInvocation:方法只有在消息接收对象中无法正常响应消息时才会被调用。所以，如果我们向往一个对象将一个消息转发给其他对象时，要确保这个对象不能有该消息的所对应的方法。否则，forwardInvocation:将不可能被调用。

##### 最后
1、调用resolveInstanceMethod给个机会让类添加这个实现这个函数
2、调用forwardingTargetForSelector让别的对象去执行这个函数
3、调用methodSignatureForSelector（函数符号制造器）和forwardInvocation（函数执行器）灵活的将目标函数以其他形式执行。
如果都不中，调用doesNotRecognizeSelector抛出异常。

方案1是动态添加方法,方案2和方案3 就是OC的消息转发机制








