---
title: WebKit与JS的简单交互
date:  2017-09-13 16:12:12
---



###JS调OC

######OC中的实现
1.创建一个WKWebViewConfiguration
  ```
   WKWebViewConfiguration 是用来配置WKWebView的
   WKWebViewConfiguration *configuration = [[WKWebViewConfiguration alloc] init];
   configuration.userContentController = [WKUserContentController new];
    // 注册一个处理事件 参数1:谁来响应这个操作事件(类似于代理，遵守 WKScriptMessageHandler 协议)  参数2: 操作唯一标识符
    //[configuration.userContentController addScriptMessageHandler:self name:@"操作的唯一标识"]; 
   [configuration.userContentController addScriptMessageHandler:self name:@"testAction"];   
  ```
2.创建一个WKWebView 与 WKWebViewConfiguration关联起来
```
self.webView = [[WKWebView alloc] initWithFrame:self.view.bounds configuration:configuration];  
self.webView.backgroundColor = [UIColor whiteColor];
[self.webView loadRequest:[NSURLRequest requestWithURL:@"........."]]; 
[self.view addSubview:self.webView];
```
3. 实现 WKScriptMessageHandler 协议中的 userContentController: didReceiveScriptMessage 方法来监听上述配置中的操作事件
```
#pragma mark - WKScriptMessageHandler
- (void)userContentController:(WKUserContentController *)userContentController didReceiveScriptMessage:(WKScriptMessage *)message{
    NSLog(@"%@",message.name); // 注册操作事件时的唯一标示符
    NSLog(@"%@",message.body); // JS返回回来的数据
    // 判定name来决定是哪个操作事件
    if([message.name isEqualToString:@"testAction"]){
      NSLog(@"testAction")
    }
}
```
4.使用结束后 确定该标识不需要用 需要删除 要不然会强引用
```
[configuration.userContentController removeScriptMessageHandlerForName:@"testAction"];
```

###### Swift

```
let configuration = WKWebViewConfiguration()
configuration.userContentController = WKUserContentController()
configuration.userContentController.add(self, name: "testAction")
        
self.webView = WKWebView.init(frame: self.view.bounds, configuration: configuration)
self.view.addSubview(self.webView)
let url = URL(string: "----")!
self.webView.load(URLRequest(url: url))
```
```
func userContentController(_ userContentController: WKUserContentController, didReceive message: WKScriptMessage) {
    print(message.name)
    print(message.body)        
 }
```
```
使用结束同样需要删除
configuration.userContentController.removeScriptMessageHandler(forName: "testAction")
```
同样也是要遵守协议 WKScriptMessageHandler





######JS中的实现
```
 在哪里需要调用OC 那么就在哪个函数中调用
 例:
  <script>
    function testAction() {
        <!--window.webkit.messageHandlers.方法名(与注册的一致(message.name) ).postMessage(JS传下来的数据(message.body))-->
      <!--data 可以是数组 字典 json字符串.....-->
        window.webkit.messageHandlers.testAction.postMessage(data)
    }
  </script>
```
###OC调JS
######OC中的实现
```
WKWebView中直接有方法可以调用
// 字符串内容为:JS中的方法名(参数) 参数可以是数组 字典 json字符串
NSString *jsStr = [NSString stringWithFormat:@"showResult('10086')"];
[self.webView evaluateJavaScript:jsStr completionHandler:^(id _Nullable result, NSError * _Nullable error) {
    NSLog(@"%@----%@",result, error);
}];
```
###### Swift
```
let arr = ["1","2","3"]
let js = "showResult(\(arr))"
self.webView.evaluateJavaScript(js, completionHandler: { (result, error) in
    print(result ?? "nil",error ?? "nil")
})
```
completionHandler 会调中的 result 是js方法的返回值 (result是个 ***id / Any*** 类型，也就是说js中可以返回任意值)


######JS中的实现
```
  <script>
    function showResult(result) {
      alert(result)
      return "hello,word"
    }
  </script>
```

###其他
##### 关于 JS中的alert弹框在WKWebView中无法弹出
原生的alert弹框在WK上面无法弹出
解决方法 :
`1. 不使用原生的alert, 自定义一个弹框`

`2. 遵守WKUIDelegate  实现WKWebView的UIDelegate`
```
// 显示一个按钮。点击后调用completionHandler回调
- (void)webView:(WKWebView *)webView runJavaScriptAlertPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(void))completionHandler{
    UIAlertController *alertController = [UIAlertController alertControllerWithTitle:@"提示" message:message?:@"" preferredStyle:UIAlertControllerStyleAlert];
    [alertController addAction:([UIAlertAction actionWithTitle:@"确认" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
        completionHandler();
    }])];
    [self presentViewController:alertController animated:YES completion:nil];
    
}

// 显示两个按钮，通过completionHandler回调判断用户点击的确定还是取消按钮
- (void)webView:(WKWebView *)webView runJavaScriptConfirmPanelWithMessage:(NSString *)message initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(BOOL))completionHandler{
    UIAlertController *alertController = [UIAlertController alertControllerWithTitle:@"提示" message:message?:@"" preferredStyle:UIAlertControllerStyleAlert];
    [alertController addAction:([UIAlertAction actionWithTitle:@"取消" style:UIAlertActionStyleCancel handler:^(UIAlertAction * _Nonnull action) {
        completionHandler(NO);
    }])];
    [alertController addAction:([UIAlertAction actionWithTitle:@"确认" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
        completionHandler(YES);
    }])];
    [self presentViewController:alertController animated:YES completion:nil];
}

// 显示一个带有输入框和一个确定按钮的，通过completionHandler回调用户输入的内容
- (void)webView:(WKWebView *)webView runJavaScriptTextInputPanelWithPrompt:(NSString *)prompt defaultText:(NSString *)defaultText initiatedByFrame:(WKFrameInfo *)frame completionHandler:(void (^)(NSString * _Nullable))completionHandler{
    UIAlertController *alertController = [UIAlertController alertControllerWithTitle:prompt message:@"" preferredStyle:UIAlertControllerStyleAlert];
    [alertController addTextFieldWithConfigurationHandler:^(UITextField * _Nonnull textField) {
        textField.text = defaultText;
    }];
    [alertController addAction:([UIAlertAction actionWithTitle:@"完成" style:UIAlertActionStyleDefault handler:^(UIAlertAction * _Nonnull action) {
        completionHandler(alertController.textFields[0].text?:@"");
    }])];
    
    [self presentViewController:alertController animated:YES completion:nil];
}
```
相当于是系统在检测到alert弹框时，会调用这些代理方法，然后通过代理方法把对应参数告诉我们，我们自己实现alert方法来弹框

##### 关于 JS中的select下拉框在WKWebView中的显示样式
js 中 select 下拉框在不同的浏览器显示的样式就不一样，在WKWebView中会已UIPickerView的形式展示，如果无法满足需求，需要自定义下拉框
