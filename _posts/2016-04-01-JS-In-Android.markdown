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
4. java注入js函数
5. webview的基本设置

## 示例
### 1. 调用的基本格式
webView中java调用js的基本格式为**webView.loadUrl(“javascript:methodName(parameterValues)”)**
js中回调java方法的格式为：**window.jsInterfaceName.methodName(parameterValues)**

### 2. Java调用js无返回值
**注意**：如果js代码中function有返回值，在java代码中没有处理返回值（应该使用evaluateJavascript处理），则首次调用js不会出问题，但是之后调用就会出现异常：
```
I/chromium: [INFO:CONSOLE(1)] "Uncaught ReferenceError: testFunc1 is not defined", source:  (1)
```
2.1. 先看js代码：
```javascript
<html>
    <script language="javascript">
        function testFunc1(var1)
        {
            alert("Hello")
            return_var = "原创文章：" + var1;
            <!--调用Android代码中的 testFunc1 的函数-->
            window.demo.testFunc1(return_var);
            <!--有无返回值直接影响java的调用成功与否，处理返回值使用evaluateJavascript接收返回值，否则不会成功(第一次成功，之后再次调用就会失败)-->
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
2.2. java代码：
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
2.3. 从java中调用js代码：
```java
String string = "http://www.sollyu.com";
//webView调用js的基本格式为webView.loadUrl(“javascript:methodName(parameterValues)”)
m_WebView.loadUrl("javascript:testFunc1(\"" + string + "\")");

String string = "这里有两个参数:";
int nInt = 191067617;
m_WebView.loadUrl("javascript:testFuncAdd(\"" + string + "\"," + String.valueOf(nInt) + ")"); // 通用这里有2个参数
```

### 3. java调用js有返回值
如果js的function有返回值，则需要使用evaluateJavascript进行处理，对返回值进行处理。
3.1. 先看js代码：
```javascript
<html>
    <script language="javascript">
        <!--有return返回值必须处理，如果不处理在java中就会出现异常
        I/chromium: [INFO:CONSOLE(1)] "Uncaught ReferenceError: testFunc1 is not defined", source:  (1)-->
        function testFunc1(var1)
        {
            alert("Hello")
            return_var = "原创文章：" + var1;
            <!--调用Android代码中的 testFunc1 的函数-->
            window.demo.testFunc1(return_var);
            <!--处理返回值使用evaluateJavascript接收返回值-->
            return return_var
        }
    </script>
</html>
```
3.2. java代码：
```java
/**
     * 上面限定了结果返回结果为String，对于简单的类型会尝试转换成字符串返回，对于复杂的数据类型，建议以字符串形式的json返回。
     evaluateJavascript方法必须在UI线程（主线程）调用，因此onReceiveValue也执行在主线程。
     * @param webView
     */
    @TargetApi(Build.VERSION_CODES.KITKAT)
    private void testEvaluateJavascript(WebView webView, String str) {
        webView.evaluateJavascript("testFunc1(\""+str+"\")", new ValueCallback<String>() {

            @Override
            public void onReceiveValue(String value) {
                Log.i("Log", "onReceiveValue value=" + value);
            }
        });
    }
```
3.3. java调用js代码：
```java
testEvaluateJavascript(m_WebView, string);
```
返回值顺序：
(1)先弹出alert弹出框hello，
(2)调用JsInteration2的testFunc1方法
(3)最后返回值并打印出onReceiveValue value="原创文章：http://www.sollyu.com"

### 4. java直接注入js函数
**使用场景**：修改远程html网页加载时的效果，如要更改图片加载方式，拦截部分超链接点击事件等。
4.1 更改图片加载方式
点击网页的图片时使用android原始图片显示器查看，或本地保存等。
4.1.1 网页源代码：
```html
<html>
<!--实现点击图片在activity中显示-->
<body>
<img src="http://pic.sc.chinaz.com/files/pic/pic9/201508/apic14052.jpg" width="100" height="100">
<br />
<img src="http://pic.sc.chinaz.com/files/pic/pic9/201508/apic14052.jpg" width="200" height="200">
<br />
<!--截取超链接，从外部浏览器打开-->
<a href="http://www.w3school.com.cn/">Visit W3School</a>
<p>通过改变 img 标签的 "height" 和 "width" 属性的值，您可以放大或缩小图像。</p>

</body>
</html>
```
4.1.2 java代码：
```java
public class JsInteration {
        private Context context;

        public JsInteration(Context context) {
            this.context = context;
        }

        // 这两个函数可以在JavaScript中调用.Uncaught TypeError: Object [object Object] has no method,如果只在4.2版本以上的机器出问题，那么就是系统处于安全限制的问题了
        @JavascriptInterface
        public void openImage(String url) {
            try{
                System.out.println("===image:"+url);
                Intent intent = new Intent(Intent.ACTION_VIEW);
                intent.setData(Uri.parse(url));
                startActivity(intent);
            }catch (Exception e){
                e.printStackTrace();
            }
        }

        @JavascriptInterface
        public void openUrl(String url) {
            try{
                System.out.println("===url:"+url);
                Intent intent = new Intent(Intent.ACTION_VIEW);
                intent.setData(Uri.parse(url));
                startActivity(intent);
            }catch (Exception e){
                e.printStackTrace();
            }
        }
    }
```
4.1.3 java调用
```java
mWebView.getSettings().setJavaScriptEnabled(true);
        //Alert无法弹出,应该是没有设置WebChromeClient
        mWebView.setWebChromeClient(new WebChromeClient() {
        });
        //Uncaught ReferenceError: functionName is not defined,问题出现原因，网页的js代码没有加载完成，就调用了js方法
        mWebView.setWebViewClient(new WebViewClient() {
            @Override
            public void onPageFinished(WebView view, String url) {
                super.onPageFinished(view, url);
                //在这里执行你想调用的js函数，如果不在载入结束时执行，就会找不到js方法网页的js代码没有加载完成，就调用了js方法
                System.out.println("load finish" + url);
                testOpenImage();
                testGetUrl();
            }
        });
        mWebView.addJavascriptInterface(new JsInteration(this), "demo");
        //例子1,也就是说可以用“file///android_asset”访问assets下文件；可以用“file:///android_res”来访问res下文件。
        mWebView.loadUrl("file:///android_asset/image.html");
```
4.1.4 注入方法
```java
/**
     * 采用直接注入js函数，给图片点击增加事件
     */
    private void testOpenImage(){
        // 这段js函数的功能就是，遍历所有的img几点，并添加onclick函数，函数的功能是在图片点击的时候调用本地java接口并传递url过去
        mWebView.loadUrl(
                "javascript:(function(){" +
                        "var objs = document.getElementsByTagName(\"img\"); " +
                        "for(var i=0;i<objs.length;i++)  " +
                        "{" +
                        "    var var1 = objs[i].src;"+//必须定义在内部function之外
                        "    objs[i].onclick=function()  " +
                        "    {  " +
                        "        console.log('this:'+this.src);"+
                        "        console.log('var:'+var1);"+
//                        "        window.demo.openImage(objs[i].src);  " + //注意：在function()内部，使用objs[i].src是无效的，必须是this.src或者使用变量var v = objs[i].src
                        "        window.demo.openImage(this.src);  " + //正确的，使用this
//                        "        window.demo.openImage(var1);  " + //正确的,使用var
                        "    }  " +
                        "}" +
                        "})()");
    }
```
**注意事项**：
在function内部嵌套的function，如onclick点击嵌套函数要调用外部变量，只能使用this或局部变量
this代表了objs[i]即调用onclick的变量，局部变量必须先定义var1,然后在内部使用。

4.2 重置超链接点击事件
java代码：
```java
/**
     * 采用直接注入js函数,拦截a标签的超链接，跳转到外部浏览器
     */
    private void testGetUrl(){
        //如果标签有id的版本
        /*mWebView.loadUrl(
                "javascript:(function(){"
                        + " var url = document.getElementById(\"visit\");"
                        + " var href = url.href;"
                        + " url.href='';" //设置为空格或#点击后都会回到顶部
                        + " url.onclick=function()"
                        + " {"
                        + "   window.demo.openUrl(href);"
                        + " }"
                        + "})()");*/
        mWebView.loadUrl(
                "javascript:(function(){"
                + " var objs = document.getElementsByTagName(\"a\");"
                + " for(var i=0;i<objs.length;i++)  "
                + " {"
                + "    if(objs[i].href){"
                + "     var href = objs[i].href;"
                + "     console.log(href+'  '+href.indexOf('#')+' '+objs[i].innerHTML);"//indexof查找#的位置,objs[i].innerHTML是标签的内容
                + "     objs[i].href='javascript:;';" //设置javascript:;以禁止跳转，#可跳转到顶部，http://www.cnblogs.com/lipanpan/p/4095524.html
                + "     objs[i].onclick=function()  "
                + "     {  "
                + "        window.demo.openUrl(href);  "
                + "     }  "
                + "  }"
                + " }"
                + "})()");
    }
```
**注意事项**：
1.indexOf,innerHTML的用法
2.禁止跳转的方法以及a标签用法-[参考链接](http://www.cnblogs.com/lipanpan/p/4095524.html)

### 5. WebView的基本配置
一般WebView不做多余配置，没有交互要求，只是很简单的载入网页的话，基本下面的配置就够用了
```java
WebSettings mWebSettings = mWebView.getSettings();
        mWebSettings.setSupportZoom(true);
        mWebSettings.setLoadWithOverviewMode(true);
        mWebSettings.setUseWideViewPort(true);
        mWebSettings.setDefaultTextEncodingName("GBK");
        mWebSettings.setLoadsImagesAutomatically(true);
```
上面可以看到，对于基本的使用，配置是很简单的，没什么太多需要自己注意的，接下来就是一些手机APP上载入网页的时候用得比较多的情况了，下面会一一列出。
其他H5高级用法可参见-[超链接](http://frank-zhu.github.io/android/2015/08/19/android-html5-web-view/)

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
2. [安卓WebView相关设置](http://frank-zhu.github.io/android/2015/08/19/android-html5-web-view/)