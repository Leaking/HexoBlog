---
title: 如何为Android应用提供全局的HttpDNS服务
date: 2017-11-18 17:58:40
toc: true
tags: NetWork
---


由于一些ISP的LocalDNS的问题，用户经常会获得一个不是最佳的DNS解析结果，导致网络访问缓慢，其中原因无非三点，第一：ISP的LocalDNS缓存；第二：ISP为了节约成本，转发DNS请求到其他ISP；第三：ISP递归解析DNS时，可能由于NAT解析错误，导致出口IP不对。这些问题也促进了各大互联网公司推出自己的DNS服务，也就是HttpDNS，传统的DNS协议是通过UDP实现，而HttpDNS是通过Http协议访问自己搭建的DNS服务器。关于HttpDNS设计的初衷，推荐阅读这篇文章[【鹅厂网事】全局精确流量调度新思路-HttpDNS服务详解](https://mp.weixin.qq.com/s?__biz=MzA3ODgyNzcwMw==&mid=201837080&idx=1&sn=b2a152b84df1c7dbd294ea66037cf262&scene=2&from=timeline&isappinstalled=0#rd)。

而对于Android应用，我们要如何接入HttpDNS服务呢？首先，你需要找一个可以用的HttpDNS服务器，比如[腾讯云的HttpDNS服务器](https://cloud.tencent.com/document/product/379)或者[阿里云的HttpDNS服务器](https://help.aliyun.com/product/30100.html?spm=5176.7741956.3.1.YDqOmh)，这些服务都是让客户端提交一个域名，然后返回若干个IP解析结果给客户端，得到IP之后，如果客户端简单粗暴地将本地的网络请求的域名替代成IP，会面临很多问题：

+ 1、Https如何进行域名验证
+ 2、如何处理SNI的问题，一个服务器使用多个域名和证书，服务器不知道应该哪个证书。
+ 3、WebView中的资源请求要如何托管
+ 4、第三方组件中的网络请求，我们要如何为它们做HttpDNS
+ ...


以上四点，腾讯云和阿里云的接入文档对前三点都给出了相应的解决方案，其实，不仅仅第四点的问题无法解决，腾讯云和阿里云对其他几点的解决方案也都不算完美，因为它们都有一个共同问题，不能全局处理，需要逐个使用网络请求的地方去相应地解决这些问题，而且这种接入HttpDNS的方式对代码的侵入性太强，缺乏可插拔的便捷性。

有没有其他侵入性更低的方式呢？接下来让我们来探索几种通过Hook的方式来为Android应用提供全局的HttpDNS服务。

 
## Native hook

可以借助dlopen的方式hook系统NDK中网络连接connect方法，在hook实现中处理域名解析（可参考[Android hacking: hooking system functions used by Dalvik](http://shadowwhowalks.blogspot.com/2013/01/android-hacking-hooking-system.html)），我们也确实在很长一段时间里都是使用这种方式处理HttpDNS，但是，从Android 7.0发布后，系统将阻止应用动态链接非公开NDK库，这种库可能会导致您的应用崩溃，可参考[Android 7.0 行为变更](https://developer.android.com/about/versions/nougat/android-7.0-changes.html#ndk)。总结一下，从Android 7.0开始，如果Target API小于等于23，则应用第一次启动时会弹出一个Toast提示(部分国产ROM上没严格遵守这个规定，目前在MIUI上有看到Toast而已)，如果Target API大于等于24，则直接Crash。

虽然目前我们的Target API只是23，但是迟早是要面临上面的问题的，所以我们开始探索使用新的方式进行Hook，native层行不通，那么只能在Java层寻找新的出路。


## Java hook

QQ邮箱Android端的网络请求主要分两种，一种走Http流量，比如自己的cgi请求都是Http流量，另一种直接走Socket，这主要是请求外域邮箱(163,126等等)，而我们的HttpDNS服务，只提供解析腾讯的域名，不提供外部域名，所以，我们其实可以只为Http流量部分HttpDNS解析。

而让我们分析一下，目前Java层的Http请求是怎么发出的，可以分为两种方式，

+ 1、直接使用HttpURLConnection，或者基于HttpURLConnection封装的Android-async-http，Volley。Okhttp等第三方库。
+ 2、使用OkHttp，OkHttp按照Http1.x, Http2.0, SPDY的语义，用刀耕火种的方式，从Socket一步步实现Http（可能你会想，Android 4.4开始，HttpURLConnection的实现不是使用了OkHttp吗？不急，按下不表，后面解释）

那我们可以针对以上两种方案，提供HttpDNS服务，首先从OkHttp开始吧，

### OkHttp

OkHttp开放了如下代码所示的DNS接口，我们可以为每个`OkHttpClient`设置自定义的DNS服务，否则将使用一个默认的DNS服务，那么我们可以为每个`OkHttpClient`设置我们的HttpDNS服务，但是这种方式不能一劳永逸，每增加一个`OkHttpClient`我们都需要手动做相应修改，而且，第三方依赖库中的`OkHttpClient`我们更是无能为力。其实，我们可以通过反射，替换掉`Dns.SYSTEM`这个默认的DNS实现，这样就可以一劳永逸了。


```java

/**
 * A domain name service that resolves IP addresses for host names. Most applications will use the
 * {@linkplain #SYSTEM system DNS service}, which is the default. Some applications may provide
 * their own implementation to use a different DNS server, to prefer IPv6 addresses, to prefer IPv4
 * addresses, or to force a specific known IP address.
 *
 * <p>Implementations of this interface must be safe for concurrent use.
 */
public interface Dns {
  /**
   * A DNS that uses {@link InetAddress#getAllByName} to ask the underlying operating system to
   * lookup IP addresses. Most custom {@link Dns} implementations should delegate to this instance.
   */
  Dns SYSTEM = new Dns() {
    @Override public List<InetAddress> lookup(String hostname) throws UnknownHostException {
      if (hostname == null) throw new UnknownHostException("hostname == null");
      return Arrays.asList(InetAddress.getAllByName(hostname));
    }
  };

  /**
   * Returns the IP addresses of {@code hostname}, in the order they will be attempted by OkHttp. If
   * a connection to an address fails, OkHttp will retry the connection with the next address until
   * either a connection is made, the set of IP addresses is exhausted, or a limit is exceeded.
   */
  List<InetAddress> lookup(String hostname) throws UnknownHostException;
}

```

### HttpURLConnection

这里说的HttpURLConnection，包含了所有基于HttpURLConnection封装的第三方网络库，如Android-async-http，Volley等等。那么，我们要如何统一的处理所有HttpURLConnection的DNS呢？

我们要从前面提到的问题开始切入，Android 4.4开始，HttpURLConnection的实现使用了OkHttp的实现，那么这两套东西是怎么结合在一起的呢？在这里，先提一个我最开始存在的一个疑问，在很久以前，还没对OkHttp的代码进行阅读，我无知地以为OkHttp也是和其他三俗的网络库一样，基于HttpURLConnection进行封装，拓展一下缓存机制，并发管理等等，那Android系统的HttpURLConnection还基于OkHttp实现？岂不是陷入“鸡生蛋，蛋生鸡，先有鸡还是先有蛋?”的问题。这个疑问现在看起来很幼稚，最终答案是，OkHttp的实现不是基于HttpURLConnection，而是自己从Socket开始，重新实现的。

回到刚才的问题，HttpURLConnection是通过什么方式，将内核实现切换到OkHttp实现，我们一般都这样构建一个HttpURLConnection

```java

HttpURLConnection urlConnection = (HttpURLConnection) url.openConnection();

```

那么，到URL这个类中寻找，HttpURLConnection是如何被构建出来的，

```java

/**
 * The URLStreamHandler for this URL.
 */
transient URLStreamHandler handler;

public URLConnection openConnection() throws java.io.IOException {
    return handler.openConnection(this);
}

```
继续寻找这个URLStreamHandler的实现

```java


static URLStreamHandlerFactory factory;

public static void setURLStreamHandlerFactory(URLStreamHandlerFactory fac) {
    synchronized (streamHandlerLock) {
        if (factory != null) {
            throw new Error("factory already defined");
        }
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkSetFactory();
        }
        handlers.clear();
        factory = fac;
    }
}

 /**
 * Returns the Stream Handler.
 * @param protocol the protocol to use
 */
static URLStreamHandler getURLStreamHandler(String protocol) {

    URLStreamHandler handler = (URLStreamHandler)handlers.get(protocol);
    if (handler == null) {

        boolean checkedWithFactory = false;

        // Use the factory (if any)
        if (factory != null) {
            handler = factory.createURLStreamHandler(protocol);
            checkedWithFactory = true;
        }

        //...

        // Fallback to built-in stream handler.
        // Makes okhttp the default http/https handler
        if (handler == null) {
            try {
                if (protocol.equals("file")) {
                    handler = (URLStreamHandler)Class.
                        forName("sun.net.www.protocol.file.Handler").newInstance();
                } else if (protocol.equals("ftp")) {
                    handler = (URLStreamHandler)Class.
                        forName("sun.net.www.protocol.ftp.Handler").newInstance();
                } else if (protocol.equals("jar")) {
                    handler = (URLStreamHandler)Class.
                        forName("sun.net.www.protocol.jar.Handler").newInstance();
                } else if (protocol.equals("http")) {
                    handler = (URLStreamHandler)Class.
                        forName("com.android.okhttp.HttpHandler").newInstance();
                } else if (protocol.equals("https")) {
                    handler = (URLStreamHandler)Class.
                        forName("com.android.okhttp.HttpsHandler").newInstance();
                }
            } catch (Exception e) {
                throw new AssertionError(e);
            }
        }

        //...
    }

    return handler;

}


```

到这里，我们找到了OkHttp的影子，Android这里反射获取的`com.android.okhttp.HttpHandler`和`com.android.okhttp.HttpsHandler`，它们都是`URLStreamHandler`的实现，

![URLStreamHandler](/images/urlstreamhandler.png)

可以到[AOSP external模块](https://android.googlesource.com/platform/external/okhttp/+/master/android/main/java/com/squareup/okhttp)中找到它们，URLStreamHandler的职责主要是构建URLConnection。上面`getURLStreamHandler`的代码，我们可以另外注意到一点，这里有一个`URLStreamHandler`的工厂实现，也就是`URLStreamHandlerFactory factory`，这个工厂默认为空，如果我们为它赋予一个实现，则可以让系统通过这个工厂，获取我们自定义的`URLStreamHandler`，这就是我们统一处理所有HttpURLConnection的关键所在，我们只需为系统提供一个自定义的`URLStreamHandlerFactory`，在其中返回一个自定义的`URLStreamHandler`，而这个`URLStreamHandler`可以返回我们提供了HttpDNS服务的URLConnection。

到此为止，我们大致知道如何统一处理所有`HttpURLConnection`，接下来需要揣摩的问题有两个:

+ 1、如何实现一个自定义的`URLStreamHandlerFactory`
+ 2、Android系统会使用了哪个版本的OkHttp呢？

关于如何实现自定义的`URLStreamHandlerFactory`，这里就不具体描述了，可以参考[AOSP的实现](https://android.googlesource.com/platform/external/okhttp/+/master/okhttp-urlconnection/src/main/java/com/squareup/okhttp/OkUrlFactory.java?autodive=0%2F%2F%2F%2F%2F%2F%2F)，或者最新[OkHttp的实现](https://github.com/square/okhttp/blob/master/okhttp-urlconnection/src/main/java/okhttp3/OkUrlFactory.java)。在这些自定义工厂中，我们都可以为其设置一个自定义的`OkhttpClient`，所以，我们也可以和前面一样，为`OkhttpClient`设置自定义的DNS服务，到此为止，我们就实现全局地为HttpURLConenction提供HttpDNS服务。


另一个问题，Android系统会使用哪个版本的OkHttp呢？以下是截止目前AOSP master分支上最新的OkHttp版本

![AOSP使用了哪个版本的OkHttp](/images/android_external_okhttp_version.png)

Android Framework竟然只使用了OkHttp2.6的代码，不知道是出于什么考虑，Android使用的OkHttp版本迟迟没有更新，可以看一下OkHttp的[CHANGELOG.md](https://github.com/square/okhttp/blob/master/CHANGELOG.md)，从2.6版本到如今最新的稳定版3.8.1，已经添加了诸多提高稳定性的bugfix、feature。所以，如果我们为应用提供一个自定义的`URLStreamHandlerFactory`，还有一个好处，就是可以使HttpURLConnection获得最新的Okhttp优化。

除此之外，还可以做很多事情，比如利用基于责任链机制的[Interceptors](https://github.com/square/okhttp/wiki/Interceptors)来做Http流量的抓包工具，或者Http流量监控工具，可以参考[chuck](https://github.com/jgilfelt/chuck).

到目前为止，我们已经可以处理所有的Http流量，为其添加HttpDNS服务，虽然已经满足我们的业务，但是还不够，作为一个通用的组件，还是需要为TCP流量也提供HttpDNS服务，也就是，如何处理所有的Socket的DNS，而如果一旦为Socket提供了统一的HttpDNS服务，也就不用再去处理Http流量的DNS，接下来开始介绍我们是如何处理的。

### 如何全局处理所有Socket的DNS

关于这个问题，我们考虑过两种思路，第一种，使用SocketImplFactory，构建自定义的SocketImpl，这种方式会相对第二种方式复杂一点，这一种方式还没真正执行，不过，这种方式有另外一个强大的地方，就是可以实现全局的流量监控，接下来可能会围绕它来做流量监控。接下来介绍另一种方式。

我们从Android应用默认的DNS解析过程入手，发现默认的DNS解析，都是调用以下`getAllByName`接口

```java

public class InetAddress implements java.io.Serializable {

	//,,,

    static final InetAddressImpl impl = new Inet6AddressImpl();

    public static InetAddress[] getAllByName(String host) throws UnknownHostException {
        return impl.lookupAllHostAddr(host, NETID_UNSET).clone();
    }	

	//,,,

}

```

而进入代码，我们可以发现，Inet6AddressImpl就是一个标准的接口类，我们完全可以动态代理它，以添加我们的HttpDNS实现，再将新的Inet6AddressImpl反射设置给上面的`InetAddressImpl impl`



## 遇到的问题

1、在使用













































