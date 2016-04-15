---
layout:     post
title:      "学习Android中使用JS的混合模式初解"
subtitle:   "read JanDan Android source result?"
date:       2016-03-18 17:00:00
author:     "BlakeQu"
header-img: "img/home-bg-o.jpg"
tags:
    - 煎蛋
    - 源码总结
    - Android
---
> "学而不思则罔，思而不学则殆"

## 掌握要点
1. 调用的基本格式
2. 有返回值和无返回值的区别
3. 在使用java调用js的注意事项

## 示例
1. 调用的基本格式
webView中java调用js的基本格式为**webView.loadUrl(“javascript:methodName(parameterValues)”)**
js中回调java方法的格式为：**window.jsInterfaceName.methodName(parameterValues)**

2. Java调用js无返回值
先看js代码：
```javascript
<html>
    <script language="javascript">
        function testFunc1(var1)
        {
            alert("Hello")
            return_var = "原创文章：" + var1;
            <!--调用Android代码中的 testFunc1 的函数-->
            window.demo.testFunc1(return_var);
            <!--有无返回值直接影响java的调用成功与否，处理返回值使用evaluateJavascript接收返回值，否则不会成功-->
            <!--return return_var;-->
        }
        function testFuncAdd(var1,var2)
        {
            return_var = var1 + var2;
            <!--这里的参数就为返回到java的值-->
            window.demo.testFuncAdd(return_var);
        }
    </script>
</html>
```
java代码：
```java
public class JsInteration2 {
        // 这两个函数可以在JavaScript中调用.Uncaught TypeError: Object [object Object] has no method,如果只在4.2版本以上的机器出问题，那么就是系统处于安全限制的问题了
        @JavascriptInterface
        public void testFunc1(String string) {
            Message msg = new Message();
            msg.what = 0;
            msg.obj = string;
            mHandler.sendMessage(msg);
        }

        @JavascriptInterface
        public void testFuncAdd(String val1) {
            Message msg = new Message();
            msg.what = 1;
            msg.obj = val1;
            mHandler.sendMessage(msg);
        }
    }
```

4. java调用js有返回值
先看js代码：
```javascript
<html>
    <script language="javascript">
        <!--有return返回值必须处理，如果不处理在java中就会出现异常
        I/chromium: [INFO:CONSOLE(1)] "Uncaught ReferenceError: toastMessage is not defined", source:  (1)-->
        function testFunc1(var1)
        {
            alert("Hello")
            return_var = "原创文章：" + var1;
            <!--调用Android代码中的 testFunc1 的函数-->
            window.demo.testFunc1(return_var);
            <!--有无返回值直接影响java的调用成功与否，处理返回值使用evaluateJavascript接收返回值-->
            return return_var
        }
    </script>
</html>
```

## 注意事项
### 1. 混淆
在debug版本中运行正常，但是发布版本就没有效果，是因为没有混淆，需要加入以下配置
```java
-keepattributes*Annotation* 
-keepattributes *JavascriptInterface*
-keepclasscom.example.javajsinteractiondemo$JsInteration{*;}
```
### 2. Alert无法弹出
需要设置WebChromeClient，如下设置
```java
myWebView.setWebChromeClient(new WebChromeClient() {});
```
### 3. Uncaught ReferenceError: functionName is not defined
**网页的js代码没有加载完成**，就调用了js方法。解决方法是在网页加载完成之后调用js方法
```java
myWebView.setWebViewClient(new WebViewClient() {

  @Override
  public void onPageFinished(WebView view, String url) {
      super.onPageFinished(view, url);
      //在这里执行你想调用的js函数
  }
  
});
```
### 4. Uncaught TypeError: Object [object Object] has no method
**安全限制问题**
如果只在4.2版本以上的机器出问题，那么就是系统处于安全限制的问题了。Android文档这样说的:
```
警告：如果你的程序目标平台是17或者是更高，你必须要在暴露给网页可调用的方法（这个方法必须是公开的）加上@JavascriptInterface注释。如果你不这样做的话，在4.2以以后的平台上，网页无法访问到你的方法。
```
**解决办法**：将targetSdkVersion设置成17或更高，引入@JavascriptInterface注释
例如：
```java
public class JsInteration {

        @JavascriptInterface
        public void toastMessage(String message) {
            Toast.makeText(getApplicationContext(), message, Toast.LENGTH_LONG).show();
        }

        @JavascriptInterface
        public void onSumResult(int result) {
            Log.i("Log", "onSumResult result=" + result);
        }
    }
```
### 5.主线程问题
All WebView methods must be called on the same thread
bug:
```
E/StrictMode( 1546): java.lang.Throwable: A WebView method was called on thread 'JavaBridge'. All WebView methods must be called on the same thread. (Expected Looper Looper (main, tid 1) {528712d4} called on Looper (JavaBridge, tid 121) {52b6678c}, FYI main Looper is Looper (main, tid 1) {528712d4})
E/StrictMode( 1546):   at android.webkit.WebView.checkThread(WebView.java:2063)
E/StrictMode( 1546):   at android.webkit.WebView.loadUrl(WebView.java:794)
E/StrictMode( 1546):   at com.xxx.xxxx.xxxx.xxxx.xxxxxxx$JavaScriptInterface.onCanGoBackResult(xxxx.java:96)
E/StrictMode( 1546):   at com.android.org.chromium.base.SystemMessageHandler.nativeDoRunLoopOnce(Native Method)
E/StrictMode( 1546):   at com.android.org.chromium.base.SystemMessageHandler.handleMessage(SystemMessageHandler.java:27)
E/StrictMode( 1546):   at android.os.Handler.dispatchMessage(Handler.java:102)
E/StrictMode( 1546):   at android.os.Looper.loop(Looper.java:136)
E/StrictMode( 1546):   at android.os.HandlerThread.run(HandlerThread.java:61)
```
在js调用后的Java回调线程并不是主线程。如打印日志可验证
```
ThreadInfo=Thread[WebViewCoreThread,5,main]
```
解决上述的异常，将webview操作放在主线程中即可。
```java
webView.post(new Runnable() {
    @Override
    public void run() {
        webView.loadUrl(YOUR_URL).
    }
});
```


## 参考
1. [技术小黑屋](http://droidyue.com/blog/2014/09/20/interaction-between-java-and-javascript-in-android/index.html)
2. 