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
上面两个重载方法，第二个是Android 5.0才支持的方法，相比第一个方法，我们能从新的参数`WebResourceRequest`中获取除了url之外的请求参数，比如mineType，我们也可以发现，如果我们不重载这两个方法，最终这两个方法都是返回null，其中第二个方法是调用了第一个方法。如果返回null，则由WebView处理资源的加载，如果返回非null，那我们则可以从网络或者本地加载资源，然后封装到WebResourceResponse中返回即可，最终WebView就会使用我们返回的资源。所以我们可以借助这个方法托管Webview资源的下载。

接下来我们再看看这两个方法的方法说明.

> Notify the host application of a resource request and allow the application to return the data. If the return value is null, the WebView will continue to load the resource as usual. Otherwise, the return response and data will be used. NOTE: This method is called on a thread other than the UI thread so clients should exercise caution when accessing private data or the view system.

我们可以发现这个方法运行在非UI线程，其实还有一个重要的问题上面没有强调的，就是这个方法每次被调用都是同个非UI线程（`Chrome_FileThread`）执行，所以多个资源并发时，这个方法是阻塞的。

单线程请求页面中的每个资源，这样的页面加载速度不能接受的，我们从`WebResourceResponse`入手，先看它的构造函数

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
我们通过静态代理的方式，代理WebResourceResponse中指向资源的InputStream成员，看看最终WebView处理资源数据时是在哪个线程。


```java
    @Override
    public WebResourceResponse shouldInterceptRequest(WebView view, final String url) {
        Log.i(TAG, String.format("shouldInterceptRequest in thread: %s[%s]  url: %s", Thread.currentThread().getName(), Thread.currentThread().getId() + "", url));
        return new WebResourceResponse("", "utf-8", new InputStream() {
            private InputStream inputStream = null;

            @Override
            public int read() throws IOException {
                Log.i(TAG, String.format("Reading data in thread %s[%s] url  %s ", Thread.currentThread().getName(), Thread.currentThread().getId(), url));
                if (inputStream == null) {
                    try {
                        Request request = new Request.Builder()
                                .url(url)
                                .build();
                        Response response = okHttpClient.newCall(request).execute();
                        inputStream = response.body().byteStream();
                    } catch (Exception e) {
                        Log.e(TAG, "Fail Reading WebResourceResponse  url: " + url);
                    }
                }
                if (inputStream == null) {
                    return -1;
                } else {
                    return inputStream.read();
                }
            }

            @Override
            public void close() throws IOException {
                super.close();
                if (inputStream != null) {
                    inputStream.close();
                }
            }
        });
    }
```



09-14 10:39:52.482 28644-28705/com.tencent.webviewsample I/MainActivity: Reading data from Thread-4[11409] url is https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1505545242&di=8a65cd2cf3995201b8028f853f592c35&imgtype=jpg&er=1&src=http%3A%2F%2Fh.hiphotos.baidu.com%2Fzhidao%2Fpic%2Fitem%2F060828381f30e9240ff2cd434c086e061d95f76a.jpg
09-14 10:39:52.482 28644-29235/com.tencent.webviewsample I/MainActivity: Reading data from Thread-7[11421] url is https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1505545266&di=2eff701f8f336f6427bdd8a50321b016&imgtype=jpg&er=1&src=http%3A%2F%2Fscimg.jb51.net%2Fallimg%2F160131%2F14-1601311A539C3.jpg
09-14 10:39:52.482 28644-28721/com.tencent.webviewsample I/MainActivity: Reading data from Thread-6[11419] url is https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1505544739&di=aab230dbfbc901260fc148e6e87ab059&imgtype=jpg&er=1&src=http%3A%2F%2Fimg3.redocn.com%2F20100521%2FRedocn_2010052023544076.jpg







