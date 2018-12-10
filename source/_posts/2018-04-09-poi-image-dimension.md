---
title: Apache POI获取Excel内嵌图片大小的一处bug
date: 2018-04-09 20:38:40
tags: POI
---

近来用Apache POI解析Excel内容，遇到一个解析图片的坑，这篇文章将解释如何正确获取Excel中图片的正确大小和位置。我想，这篇文章很适合和我一样正在用Apache POI解析Excel内容的人阅读，替你避开一些坑，以及让你知道Apache POI中一处解析图片大小的bug。


我们知道，Apache POI中对Excel的图片的大小和位置存储在ClientAnchor对象中，而该对象可以通过Picture获取

```java
ClientAnchor anchor = picture.getClientAnchor();
```

而图片的大小和位置，最终是通过ClientAnchor的以下几个变量进行描述

```java
row1, col1, row2, col2, dx1, dy1, dx2, dy2
```

请看下图，图中标出了以上八个变量的含义


![](/images/poi_image_size_01.png)

row1, col1代表图片左上角所在的格子，而dx1, dy1表示图片在这个左上角格子的偏移量；
row2, col2代表图片右下角所在的格子，而dx2, dy2表示图片在这个右下角格子的偏移量。

从图中不难发现，要获取图片位置，只需把在row1, col1之前的列宽，行高分别累加起来，然后再分别加上dx1, dy1的偏移量，即可得到图片在sheet中的位置。


而要获取图片大小，从图中对几个变量的注释，也不难理解如何获取图片大小，我们可以使用POI的工具方法获取图片大小[org.apache.poi.ss.util.ImageUtils#getDimensionFromAnchor](https://github.com/apache/poi/blob/2395c89b4521bc98f0b337d309679c8386bd8c45/src/java/org/apache/poi/ss/util/ImageUtils.java#L225)

由于计算图片高度与计算宽度原理一致，所以我们只介绍一下计算宽度的思路。上面图中，图片横跨了4列，我们默认这四列宽度分别为width1, width2, width3, width4。那么

```java
图片宽度 = (width1 - dx1) + width2 + width3 + dx2
```

到此为止貌似一切都很顺利，但是后来上线发现一个问题，当一张图片只占据一个方格时，[org.apache.poi.ss.util.ImageUtils#getDimensionFromAnchor](https://github.com/apache/poi/blob/2395c89b4521bc98f0b337d309679c8386bd8c45/src/java/org/apache/poi/ss/util/ImageUtils.java#L225) 这个方法计算图片大小是有bug的，

我们先来看看，当图片只占据一个方格时的样子

![](/images/poi_image_size_02.png)

很明显，此时

```java
图片宽度 = dx2 - dx1
```

综上，我们可以得出计算图片大小的结论，我们依然只列出结算宽度的过程

```java
如果col1 < col2， 默认这其中横跨的列，它们的列宽分别是width1, width2, width3, ... , widthn，那么

图片宽度 = (width1 - dx1) + width2 + width3 + ... + width(n-1) + dx2

如果col1 = col2，那么

图片宽度 = dx2 - dx1

```

最后附上实现代码[Gist](https://gist.github.com/Leaking/8e0ac94aeb80800c2376f39010eac41d)

