---
layout:     post
title:      "读煎蛋客户端源码总结"
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

学习首先的从读源码开始，煎蛋源码来自[JanDan源码](https://github.com/ZhaoKaiQiang/JianDan_OkHttpWithVolley.git)，它使用okhttp作为Volley底层请求框架的煎蛋高仿客户端。

# 知识点
1. 学习煎蛋源码框架，并说出其框架**优点和缺点**，并结合说明我们框架需要**改进**的地方。
2. 学习一个框架中如何选择合适的网络框架，每种网络框架的特点。需要学习网络框架包括[OkHttp](https://github.com/square/okhttp),[Asynchronous Http Client for Android](https://github.com/loopj/android-async-http) [Volley](https://android.googlesource.com/platform/frameworks/volley)
3. 网络请求的原理，可访问[网络请求原理](http://www.jianshu.com/p/3141d4e46240)
4. 使用了哪些优秀的开源库,[内存泄露检测库](https://github.com/liaohuqiu/leakcanary-demo)
5. 一种下拉刷新，上拉自动加载更多的方案使用：**SwipeRefreshLayout+com.socks.jiandan.view.AutoLoadRecyclerView**
6. 图片缓存方案：[Android-Universal-Image-Loader](https://github.com/nostra13/Android-Universal-Image-Loader), [picasso](https://github.com/square/picasso)及[文档](http://square.github.io/picasso/), [facebook的fresco](https://github.com/facebook/fresco)及[中文文档](http://www.fresco-cn.org/docs/index.html#_), [阿里的Cube ImageLoader](https://github.com/etao-open-source/cube-sdk)及[文档](http://cube-sdk.liaohuqiu.net/)

# 阅读收获
## 框架的优点
Volley：Google 提供的网络通信库，使得网络请求更简单、更快速
Asynchronous Http Client for Android：
Android 异步 Http 请求
项目地址：https://github.com/loopj/android-async-http
文档介绍：http://loopj.com/android-async-http/
特点：
(1) 在匿名回调中处理请求结果
(2) 在 UI 线程外进行 http 请求
(3) 文件断点上传
(4) 智能重试
(5) 默认 gzip 压缩
(6) 支持解析成 Json 格式
(7) 可将 Cookies 持久化到 SharedPreferences
OkHttp:
square 开源的 http 工具类
项目地址：https://github.com/square/okhttp
文档介绍：http://square.github.io/okhttp/
特点：(1) 支持 SPDY( http://zh.wikipedia.org/wiki/SPDY )协议。SPDY 协议是 Google 开发的基于传输控制协议的应用层协议，通过压缩，多路复用(一个 TCP 链接传送网页和图片等资源)和优先级来缩短加载时间。
(2) 如果 SPDY 不可用，利用连接池减少请求延迟
(3) Gzip 压缩
(4) Response 缓存减少不必要的请求
Google推出了官方的针对Android平台上的网络通信库volley，能使网络通信更快，更简单，更健壮，Volley在提供了高性能网络通讯功能的同时，对网络图片加载也提供了良好的支持，完全可以满足简单REST客户端的需求, 我们没有理由不跟上时代的潮流。另外，但volley的扩展性很强，可以根据需要定制你自己的网络请求。所以，最后推荐还是使用volley进行开发，当然其他几个库也是非常具有学习以及参考意义的，可以将他们的精髓之处汲取到volley框架的拓展开发之中，做出自己理想的http通讯框架

## 框架的缺点

# 读后任务
1. 改进框架
2. 改进网络框架
3. 融入优秀的开源库
4. 代码结构优化

参考网址[Trinea Android开源汇总](https://github.com/Trinea/android-open-project#%E4%B8%89%E7%BD%91%E7%BB%9C%E8%AF%B7%E6%B1%82)