---
layout:     post
title:      "JS与OC交互，JS中调用OC方法"
date:       2018-5-9 9:15:00
author:     "Karthus"
header-img: "img/post_header_bg.jpg"
tags:
    - JavaScript
    - Obejctive-C
    - Brige
---

  最近用到JS和OC原生方法调用的问题，查了许多资料都语焉不详，自己记录一下吧，如果有误欢迎联系我指出。

####JS中调用OC方法有三种方式：
1.通过获取JSContext的方式直接调用OC方法
2.通过继承自JSExport的方式调用delegate中的方法
3.截取URL的方式（此种方式资料很多，就不写了）

先上OC代码

    -(void)webViewDidFinishLoad:(UIWebView *)webView
    {
        self.jsContext =  [self.webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"];
    
        // 方法一
        __weak typeof(self) weakSelf = self;
        self.jsContext[@"getMessage"] = ^(){
            return [weakSelf blockCallMessage];
        };
    
        // 方法二
        self.jsContext[@"JavaScriptInterface"] = self;
    }

    - (NSString *)blockCallMessage
    {
        return @"call via block";
    }

    #pragma mark - JSCallDelegate
      // 提供给JS调用的方法
    - (NSString *)tipMessage
    {
        return @"call via delegate";
    }


    
HTML代码
            
    <html>
    <head>
    </head>
    <body>
        <script>
            function buttonClick1()
            {
                // 方法一
                 var token = getMessage();

                alert(token)
            }
            function buttonClick2()
            {
                // 方法二
                var token = JavaScriptInterface.tipMessage();
            
                alert(token)
            }
            </script>
        <button id="abc" onclick="buttonClick1()">function 1</button>
        <button id="abcd" onclick="buttonClick2()">function 2</button>
    </body>
    </html>

方法一中的**jsContext[@"getMessage"]**需要和JS中调用的方法名一致，既JS中需要直接调用getMessage()，而jsContext[@"getMessage"]则赋值为一个block，这个block中调用的方法就是JS中调用getMessage()就是执行这个block；

方法二中的**jsContext[@"JavaScriptInterface"] = self**其中的**JavaScriptInterface**可以随便命名，保持JS中和该命名保持一致即可，JS中通过**JavaScriptInterface**来调用继承自JSExport的delegate中的方法，即**JavaScriptInterface.tipMessage()**

[demo下载](https://github.com/Karthus1110/JSCallOCDemo)