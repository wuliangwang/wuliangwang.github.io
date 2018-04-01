---
title: iOS - 线程 / 进程 的通信
date: 2018-03-01
---





#### 1. 线程中的通信

* 线程中通信的体现
```
1 .一个线程传递数据给另一个线程
2 .在一个线程中执行完特定任务后，转到另一个线程继续执行任务
```
在iOS多线程开发中，有NSThread、GCD、NSOpeartion几种方式，
对应的线程间通信
#####1.1 NSThread
`NSThread可以先将自己的当前线程对象注册到某个全局的对象中去，这样相互之间就可以获取对方的线程对象，然后就可以使用下面的方法进行线程间的通信了，由于主线程比较特殊，所以框架直接提供了在主线程执行的方法`

回到主线程执行
```
/*
 *  aSelector： 执行方法
 *  arg: 携带参数
 */
- (void)performSelectorOnMainThread:(SEL)aSelector withObject:(nullable id)arg waitUntilDone:(BOOL)wait;
```
去指定线程执行任务
```
/*
 *   aSelector： 执行方法
 *   thr: 指定线程
 *   arg: 携带参数
 */
- (void)performSelector:(SEL)aSelector onThread:(NSThread *)thr withObject:(id)arg waitUntilDone:(BOOL)wait;
```
关于 **waitUntilDone**
```
waitUntilDone的含义:
如果传入的是YES: 那么会等到主线程中的方法执行完毕, 才会继续执行下面其他行的代码
如果传入的是NO: 那么不用等到主线程中的方法执行完毕, 就可以继续执行下面其他行的代码
```

#####1.2 GCD
* 什么是GCD
  全称是Grand Central Dispatch，可译为“牛逼的中枢调度器”
  纯C语言，提供了非常多强大的函数

* GCD的使用
  GCD中有2个核心概念
  （1）任务：执行什么操作
  （2）队列：用来存放任务

  GCD的使用就2个步骤
  （1）定制任务
  （2）确定想做的事情

  将任务添加到队列中，GCD会自动将队列中的任务取出，放到对应的线程中执行
  注意：任务的取出遵循队列的FIFO原则：先进先出，后进后出
```
  /*
  * queue：队列
  * block：任务 
  */
（1）用同步的方式执行任务 dispatch_sync(dispatch_queue_t queue, dispatch_block_t block);
（2）用异步的方式执行任务 dispatch_async(dispatch_queue_t queue, dispatch_block_t block);
```
* 线程通信
```
// queue 表示某个子线程
dispatch_async(subQueue, ^{
        // 在子线程中
  dispatch_async(dispatch_get_main_queue(), ^{
      #回到主线程
    });
});
```
关于GCD 远远不止这些功能, 这里这是讲了最基本的简单通信. 

#####1.3 NSOperation
* 简介
  NSOperation是基于GCD的，那么使用起来也和GCD差不多，其中，NSOperation相当于GCD中的任务，而NSOperationQueue则相当于GCD中的队列。

* NSOperation实现多线程的使用步骤分为三步：
  创建任务：先将需要执行的操作封装到一个NSOperation对象中。
  创建队列：创建NSOperationQueue对象。
  将任务加入到队列中：然后将NSOperation对象添加到NSOperationQueue中。

* 获取队列
```
// 主队列
// 凡是添加到主队列中的任务（NSOperation），都会放到主线程中执行
NSOperationQueue *queue = [NSOperationQueue mainQueue];
```
```
// 其他队列（非主队列）
// 添加到这种队列中的任务（NSOperation），就会自动放到子线程中执行
// 同时包含了：串行、并发功能
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
```

* 添加任务
  `- (void)addOperation:(NSOperation *)op`
  需要先创建任务，再将创建好的任务加入到创建好的队列中去
```
// 1.创建队列
NSOperationQueue *queue = [[NSOperationQueue alloc] init];

// 2. 创建操作  
// 创建NSInvocationOperation    
NSInvocationOperation *op1 = [[NSInvocationOperation alloc] initWithTarget:self selector:@selector(test) object:nil];    
// 创建NSBlockOperation    
NSBlockOperation *op2 = [NSBlockOperation blockOperationWithBlock:^{
  NSLog(@"op2 ---- %@", [NSThread currentThread]);
}];

// 3. 添加操作到队列中：addOperation:   
[queue addOperation:op1]; // [op1 start]    
[queue addOperation:op2]; // [op2 start]

- (void)run{
  NSLog(@"op1 ---- %@", [NSThread currentThread]);
}
```
`- (void)addOperationWithBlock:(void (^)(void))block`
无需先创建任务，在block中添加任务，直接将任务block加入到队列中。
```
// 1. 创建队列
NSOperationQueue *queue = [[NSOperationQueue alloc] init];

// 2. 添加操作到队列中：addOperationWithBlock:
[queue addOperationWithBlock:^{
  NSLog(@"%@", [NSThread currentThread]);
}];
```
* 线程通信
```
NSOperationQueue *queue = [[NSOperationQueue alloc] init];
[queue addOperationWithBlock:^{
  NSLog(@"1 ---- %@", [NSThread currentThread]);
  # 回到主线程
  [[NSOperationQueue mainQueue] addOperationWithBlock:^{
        NSLog(@"2 ---- %@", [NSThread currentThread]);
  }];
}];
```
NSOperation 和GCD一样 不止以上这些功能，这里只是讲述了基本的使用和简单的线程通信.

####2.进程的通信
进程是容纳运行一个程序所需要所有信息的容器。在iOS中每个APP里就一个进程，所以进程间的通信实际上是APP之间的通信.

#####2.1 URL Scheme
* 什么是URL Scheme
  由于苹果的app都是在沙盒中，相互是不能访问数据的。但是苹果还是给出了一个可以在app之间跳转的方法：URL Scheme。简单的说，URL Scheme就是一个可以让app相互之间可以跳转的协议。每个app的URL Scheme都是不一样的，如果存在一样的URL Scheme，那么系统就会响应先安装那个app的URL Scheme，因为后安装的app的URL Scheme被覆盖掉了，是不能被调用的。

* URL Scheme怎么使用
  App1通过openURL的方法跳转到App2，并且在URL中带上想要的参数，有点类似http的get请求那样进行参数传递。

源 APP1 的配置
源App1需要在info.plist中配置LSApplicationQueriesSchemes，指定目标App2的scheme
![源 APP1 的配置](http://upload-images.jianshu.io/upload_images/3111822-a373512510b93081.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

目标 APP2 的配置
然后在目标App2的info.plist中配置好URL types，表示该app接受何种URL scheme的唤起。
![目标 APP2 的配置](http://upload-images.jianshu.io/upload_images/3111822-dcce91b08e42c437.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


也可以在 项目-> info -> URL types 中直接添加
![配置方法二](http://upload-images.jianshu.io/upload_images/3111822-29f215842f653328.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

源 APP1 的代码调用
```
UIApplication *application = [UIApplication sharedApplication];
NSURL *URL = [NSURL URLWithString:@"testApp2://hello,word"];
[application openURL:URL options:@{} completionHandler:nil];
```
目标 APP2 的代码接收
```
AppDelegate.m 中
- (BOOL)application:(UIApplication *)app openURL:(NSURL *)url options:(NSDictionary<UIApplicationOpenURLOptionsKey,id> *)options{
    NSLog(@"%@",url);
    return YES;
}
```
![打印结果](http://upload-images.jianshu.io/upload_images/3111822-96a29b608cfdec57.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#####2.2 UIPasteboard
UIPasteboard是剪切板功能，因为iOS的原生控件UITextView，UITextField 、UIWebView，我们在使用时如果长按，就会出现复制、剪切、选中、全选、粘贴等功能，这个就是利用了系统剪切板功能来实现的。而每一个App都可以去访问系统剪切板，所以就能够通过系统剪贴板进行App间的数据传输了。
UIPasteboard典型的使用场景就是淘宝的淘口令。
* 使用
  源APP赋值
  `[[UIPasteboard generalPasteboard] setString:@"hello,word"];`
  目标APP取值
  `NSString *value = [UIPasteboard generalPasteboard].string;`

#####2.3 Keychain
>Keychain是iOS中一个安全存储容器，本质是一个sqlite数据库，位置在/private/var/Keychains/keychain-2.db。它是独立于每个App的沙盒之外的，所以即使App被删除之后，Keychain里面的信息依然存在。基于安全和独立于app沙盒的两个特性，Keychain主要用于给app保存登录和身份凭证等敏感信息，这样只要用户登录过，即使用户删除了app重新安装也不需要重新登录。
> Keychain用于App间通信的一个典型场景也和app的登录相关，就是统一账户登录平台。使用同一个账号平台的多个app，只要其中一个app用户进行了登录，其他app就可以实现自动登录不需要用户多次输入账号和密码。一般开放平台都会提供登录SDK，在这个SDK内部就可以把登录相关的信息都写到keychain中，这样如果多个app都集成了这个SDK，那么就可以实现统一账户登录了。

 [具体使用](https://www.jianshu.com/p/6bc772bdeece)
目前我自己还不太熟悉这个框架的使用,后续了解清楚会补充该框架的本人理解及使用方法.

#####2.4 分享
通过将源APP数据分享给目标APP 实现通信
原生分享 
* UIActivityViewController(轻量级信息分享),
```
// Items: 元素1是标题，元素2是URL
UIActivityViewController *avc = [[UIActivityViewController alloc]initWithActivityItems:@[@"title",url] applicationActivities:nil];
    
[self presentViewController:avc animated:YES completion:nil];
    
//分享结果回调方法
UIActivityViewControllerCompletionWithItemsHandler myblock = ^(NSString * __nullable activityType, BOOL completed, NSArray * __nullable returnedItems, NSError * __nullable activityError){
    };
avc.completionWithItemsHandler = myblock;
```
* UIDocumentInteractionController (主要是共享文档，预览)
```
NSString *urlStr = [[NSBundle mainBundle] pathForResource:@"1.png" ofType:nil];
NSURL *url = [NSURL fileURLWithPath:urlStr];
    
UIDocumentInteractionController *documentVc = [UIDocumentInteractionController interactionControllerWithURL:url];
documentVc.delegate = self;

[documentVc presentPreviewAnimated:YES];
```
```
#pragma mark - UIDocumentInteractionController 代理方法
// 预览的视图控制器
- (UIViewController *)documentInteractionControllerViewControllerForPreview:(UIDocumentInteractionController *)controller{
    return self;
}
// 父View
- (UIView *)documentInteractionControllerViewForPreview:(UIDocumentInteractionController *)controller{
    return self.view;
}
// 预览界面的尺寸
- (CGRect)documentInteractionControllerRectForPreview:(UIDocumentInteractionController *)controller{
    return self.view.bounds;
}
```

* 第三方SDK分享










