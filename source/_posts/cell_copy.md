---
title: Cell自带的长按复制
date: 2018-03-09 14:48:46
---

系统提供了 UITableViewCell 长按复制的代理方法，遵守实现即可。
![效果图1](http://upload-images.jianshu.io/upload_images/3111822-81107944ab0650db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

需要遵守 `UITableViewDelegate` 协议，并设置代理对象.
##### 1.设置哪一行显示。
```
- (BOOL)tableView:(UITableView *)tableView shouldShowMenuForRowAtIndexPath:(NSIndexPath *)indexPath{
      return YES;// 设置哪里显示。
}
```
如果指定 cell 才有该功能 通过`indexPath`判断下。返回 **YES** 表示可以显示。**NO**表示不显示。

#####2. 指定哪一行显示哪些操作
```
// 设置只有 section = 0 的cell 才能显示并且只有 copy 一个操作
// 根据  action 和 indexPath 来决定哪行哪些操作可以显示
- (BOOL)tableView:(UITableView *)tableView canPerformAction:(SEL)action forRowAtIndexPath:(NSIndexPath *)indexPath withSender:(id)sender {
    NSLog(@"-------%@",NSStringFromSelector(action));
    if(action == @selector(copy:) && indexPath.section == 0){
        return YES;
    }
    return NO;
}
```
该代理方法会多次调用来确定哪些操作可以显示, 我通过打印  action 发现了 `cut:、copy:、paste:、select:、selectAll: ....`一些操作, 但是当我全部返回YES时发现显示的只有`cut:、copy:、paste:` 这三个，并且当我设置这三个其中一个为NO时, `select:`或者其他非该三个 action 时 显示的就只有两个了，通过这样的测试我发现 可以显示的操作只有 `cut:、copy:、paste:`。如果是我的操作不对造成的这样的效果，欢迎在评论区提出，谢谢.
![全部返回YES的效果](http://upload-images.jianshu.io/upload_images/3111822-b5ad6a8d92e4e1b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#####3、执行操作，设置操作对应的内容
```
// 长按选择操作后调用
- (void)tableView:(UITableView *)tableView performAction:(SEL)action forRowAtIndexPath:(NSIndexPath *)indexPath withSender:(id)sender{
    // 根据 indexPath 和 action 来判定是不是对应的操作。
    if (action == @selector(copy:)) {
        [UIPasteboard generalPasteboard].string = phoneStr;
    }
    // 实现操作执行后应该实现的功能。
}
```

