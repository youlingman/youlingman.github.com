---
title: HttpURLConnection和webview的协同使用
date: 2017-03-27 14:34:40
tags: [android]
---

### webview中使用post加载

在项目中遇到一个需求，需要在android的webview中对特定请求（其实就是第一次打开webview的时候）进行可自定义header的post请求，而原生webview提供的post API是这样的：

	public void postUrl(String url, byte[] postData);

不可以自定义header，反观get API可以自定义header，真让人头大。

	void loadUrl (String url, 
                Map<String, String> additionalHttpHeaders)

<!--more-->

网上找了一轮，有以下几种实现思路：

* 自定义WebViewClient，里面可以统一拦截请求并实现对应处理逻辑，这里的问题是需要额外规则来适配特定的请求；
* 重写webview，为了这个小需求重写webview不太现实；
* 手写http post请求，然后使用webview的loadData方法加载返回内容；

由于开发框架封了一个公共的webview，对webview或者webviewclient的改动可能会影响到其它业务组件，最后决定对第一次请求，使用HttpURLConnection手动post请求，然后打开webview并把返回的内容（HTML文本）load进去，对原有逻辑影响最小。

webview可以认为是基于webkit的浏览器客户端，里面封装好了对请求的相关处理，单独实现请求还需要注意相关的特殊处理，下面是踩过的两个坑。

### HttpURLConnection和webview协同-重定向处理

HttpURLConnection有一个方法setInstanceFollowRedirects可以设置是否自动follow重定向。

	public void setInstanceFollowRedirects(boolean followRedirects)

但是如果原URL是http而目标URL是https的话这个方法是不能自动跳转的，搜了下[这里](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=4620571)提到java由于安全原因不允许不同协议间的自动重定向跳转，需要应用层面去实现各自的重定向逻辑。

实现逻辑很简单，就是判一下返回码，如果为重定向的返回码，则把header里Location域的url取出来load进webview里。

### HttpURLConnection和webview协同-cookie处理

根据官方文档，android应用的webview实例的cookie都是由[CookieManager](https://developer.android.com/reference/android/webkit/CookieManager.html)来管理的，因此HttpURLConnection请求返回的cookie需要同步到CookieManager里，然后这个CookieManager里面坑不少，主要有以下几个：

* 如果http response里返了多条cookie，需要调用多次setCookie方法来完成设置；
* setCookie方法设置的cookie要带上path信息，否则webview里的ajax请求不能同步到对应cookie（这个setCookie方法为啥不提供一个接受HttpCookie类型参数的签名呢，只接受String类型的参数用起来太不方便了）；
* 在低于21的SDK版本里，CookieManager的removeSessionCookie方法和CookieSyncManager的sync方法是**非阻塞异步操作**，而且具体逻辑封起来了，没办法进行有效的线程同步，这意味着啥呢，就是说可能cookie设置完了然后removeSessionCookie方法才被调用，然后刚设置好的cookie就被清掉了，或者前一个请求的cookie还没设置完呢后续页面已经打开了，然后前一个请求的cookie就没办法同步到后续页面里。解决方案十分暴力，就是当系统SDK版本低于21时，在removeSessionCookie方法和sync方法后面sleep上300到500ms，亲测有效。