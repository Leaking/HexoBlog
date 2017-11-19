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

 
## native 







