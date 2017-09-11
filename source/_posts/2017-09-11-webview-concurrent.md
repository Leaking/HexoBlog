---
title: WebView资源并发加载
date: 2017-09-11 10:28:26
toc: false
tags: Android
---

我们经常通过`WebViewClient`的`shouldInterceptRequest`方法拦截WebView请求，自己托管页面资源的加载，我们先来看一下这个方法能为我们做什么：

```java
    @Deprecated
    public WebResourceResponse shouldInterceptRequest(WebView view,
            String url) {
        return null;
    }
    public WebResourceResponse shouldInterceptRequest(WebView view,
            WebResourceRequest request) {
        return shouldInterceptRequest(view, request.getUrl().toString());
    }
```
上面两个重载方法，第二个是Android 5.0才支持的方法，相比第一个方法，我们能从新的参数`WebResourceRequest`中获取除了url之外的请求参数，比如mineType，我们也可以发现，如果我们不重载这两个方法，最终这两个方法都是返回null，其中第二个方法是调用了第一个方法。如果返回null，则由`WebView`处理资源的加载，如果返回非null，那我们则可以从网络或者本地加载资源，然后封装到`WebResourceResponse`中返回即可，最终WebView就会使用我们返回的资源。所以我们可以借助这个方法托管Webview资源的下载。

接下来我们再看看这两个方法的方法说明.

> Notify the host application of a resource request and allow the application to return the data. If the return value is null, the WebView will continue to load the resource as usual. Otherwise, the return response and data will be used. NOTE: This method is called on a thread other than the UI thread so clients should exercise caution when accessing private data or the view system.

我们可以发现这个方法运行在非UI线程，其实还有一个问题上面的说明没有强调的，就是这个方法每次被调用都是同个非UI线程，所以多个资源并发时，这个方法是阻塞的。


日志截图，，，，


单线程请求页面中的每个资源，这样的页面加载速度不能接受的，那是否有突破的地方呢？我们从`WebResourceResponse`这个对象入手，先看它的构造函数

```java
public WebResourceResponse(String mimeType, String encoding, InputStream data) {
        mMimeType = mimeType;
        mEncoding = encoding;
        setData(data);
}
public WebResourceResponse(String mimeType, String encoding, int statusCode,
            String reasonPhrase, Map<String, String> responseHeaders, InputStream data) {
        this(mimeType, encoding, data);
        setStatusCodeAndReasonPhrase(statusCode, reasonPhrase);
        setResponseHeaders(responseHeaders);
}
```

我们发现，最终我们传递给WebView的资源，都是存储在WebResourceResponse中的一个InputStream字节流，我们代理这个字节流成员变量，看看最终WebView读取资源时是在哪个线程？










我们先来看看







