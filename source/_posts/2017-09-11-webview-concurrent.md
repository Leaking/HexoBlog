---
title: WebView资源并发加载的另一种思路
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
上面两个重载方法，第二个方法是Android 5.0才支持的方法，相比第一个方法，我们能从第二个方法的新参数`WebResourceRequest`中获取除了url之外的请求参数，比如mineType，我们也可以发现，如果我们不重载这两个方法，最终这两个方法都是返回null，其中第二个方法是调用了第一个方法。如果返回null，则由`WebView`处理资源的加载，如果返回非null，那我们则可以从网络或者本地加载资源，然后封装到`WebResourceResponse`中返回即可，最终WebView就会使用我们返回的资源。所以我们可以借助这个方法托管Webview资源的下载。

接下来我们再看看这两个方法的方法说明.

> Notify the host application of a resource request and allow the application to return the data. If the return value is null, the WebView will continue to load the resource as usual. Otherwise, the return response and data will be used. NOTE: This method is called on a thread other than the UI thread so clients should exercise caution when accessing private data or the view system.

我们可以发现这个方法运行在非UI线程，其实还有一个重要的问题上面没有强调的，就是这个方法每次被调用都是同个非UI线程（`Chrome_FileThread`）执行，所以多个资源并发时，这个方法是阻塞的，而一般同个HTML难以避免会有多个资源存在。

单线程请求页面中的每个资源，这样的页面加载速度不能接受的，接下来我们看看有什么可以突破的地方。我们从`WebResourceResponse`入手，先看它的构造函数

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

```xml
09-14 10:39:52.482 28644-28705/com.tencent.webviewsample I/MainActivity: Reading data from Thread-4[11409] url is https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1505545242&di=8a65cd2cf3995201b8028f853f592c35&imgtype=jpg&er=1&src=http%3A%2F%2Fh.hiphotos.baidu.com%2Fzhidao%2Fpic%2Fitem%2F060828381f30e9240ff2cd434c086e061d95f76a.jpg
09-14 10:39:52.482 28644-29235/com.tencent.webviewsample I/MainActivity: Reading data from Thread-7[11421] url is https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1505545266&di=2eff701f8f336f6427bdd8a50321b016&imgtype=jpg&er=1&src=http%3A%2F%2Fscimg.jb51.net%2Fallimg%2F160131%2F14-1601311A539C3.jpg
09-14 10:39:52.482 28644-28721/com.tencent.webviewsample I/MainActivity: Reading data from Thread-6[11419] url is https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1505544739&di=aab230dbfbc901260fc148e6e87ab059&imgtype=jpg&er=1&src=http%3A%2F%2Fimg3.redocn.com%2F20100521%2FRedocn_2010052023544076.jpg

```

最终惊奇发现，读取图片数据流时是并发进行的，那么我们就可以突破`shouldInterceptRequest`单线程执行的问题了，只需要代理WebResourceResponse中指向资源的InputStream成员，当系统读取数据流的第一个字时去执行资源下载。


我们使用以下内容，对比单线程和多线程两种图片加载方式的速度

```html
<!DOCTYPE html>
<html>  
<head>    
<script>    
window.addEventListener('load', function(ev) {      
	App.onLoad('onload');   //Java层注册JavasciptInterface APP，使'load'回调判断图片加载完毕
}); 
</script>  
</head> 
  <body>    
  <img  onload='console.log(1)' src='https://rescdn.qqmail.com/zh_CN/htmledition/images/webp/logo/qqmail/qqmail_logo_default_35h206ff1.png' />    
  <img  onload='console.log(3)' src='https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1505544739&di=aab230dbfbc901260fc148e6e87ab059&imgtype=jpg&er=1&src=http%3A%2F%2Fimg3.redocn.com%2F20100521%2FRedocn_2010052023544076.jpg' />    
  <img  onload='console.log(4)' src='https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1505545111&di=2d4fc96817efd328d2865bf4ac37559e&imgtype=jpg&er=1&src=http%3A%2F%2Fimg1.pconline.com.cn%2Fpiclib%2F200812%2F22%2Fbatch%2F1%2F19891%2F1229912689801kpqtgqzpxq.jpg'  />    
  <img  onload='console.log(5)' src='https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1505545242&di=8a65cd2cf3995201b8028f853f592c35&imgtype=jpg&er=1&src=http%3A%2F%2Fh.hiphotos.baidu.com%2Fzhidao%2Fpic%2Fitem%2F060828381f30e9240ff2cd434c086e061d95f76a.jpg'  />    
  <img  onload='console.log(6)' src='https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1505545266&di=2eff701f8f336f6427bdd8a50321b016&imgtype=jpg&er=1&src=http%3A%2F%2Fscimg.jb51.net%2Fallimg%2F160131%2F14-1601311A539C3.jpg'  />    
  <img  onload='console.log(7)' src='https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1505545753&di=c72dc06135b35c589bf9fcfbce5ff7e1&imgtype=jpg&er=1&src=http%3A%2F%2Fimg2.niutuku.com%2Fdesk%2Fanime%2F4446%2F4446-8866.jpg'  />    
  <img  onload='console.log(8)' src='https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1505545753&di=c72dc06135b35c589bf9fcfbce5ff7e1&imgtype=jpg&er=1&src=http%3A%2F%2Fimg2.niutuku.com%2Fdesk%2Fanime%2F4446%2F4446-8866.jpg'  />    
  <img  onload='console.log(9)' src='https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1505545784&di=efb541bc5fe7807ea9097ebb5c21e756&imgtype=jpg&er=1&src=http%3A%2F%2Fscimg.jb51.net%2Fallimg%2F140404%2F10-140404220101L9.jpg'  /> 
  </body>
</html>
```

最终多次试验对比后，数据如下

试验次数（单位:ms）   |1  |2  |3  |4  |5  |6  |7  |...|平均|
---|---|---|---|---|---|---|---|---|---|
单线程|407|416|364|477|377|317|433|...|398.7|
多线程|298|275|256|255|258|307|306|...|279.2|

多线程的方式比单线程的方式速度上提升了30%。


总结一下，本主介绍了一个简单但是效果显著的方法，解决托管WebView资源下载时的并发问题，该方法已稳定运行在QQ邮箱Android客户端几个版本了。





