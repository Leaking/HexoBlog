---
title: 微盘文件系统的同步
date: 2019-11-26 23:12:06
tags: [FileSystem]
---

我们平时在互联网上使用的文件存储服务常见的有两种，第一种是百度网盘，第二种是Dropbox，Google Drive这一流派。他们最大的区别在于是不是被墙了，不过，我们今天不是讨论GFW，也不是讲科学上网。而是探讨如何实现类似Dropbox的文件系统同步方案。程序员群体，大部分人应该都使用或了解过Dropbox，如果没有也无妨，我们会先简单介绍一下Dropbox的基本功能，以及一些充钱让人变强之后才具备的功能。   

这篇文章和大家分享一下微盘文件同步系统的设计与实现，分享一些遇到的问题以及解决思路，还有一些文件系统相关的特性，还会涵盖一些文件系统安全相关的问题。我们围绕以下几点展开介绍：1、介绍整体框架，2、介绍关键关键概念，3、分模块介绍

首先明确我们的目的，我们的目的是同步本地文件树与服务器的文件树。此处我们列出几个关键概念，整个框架都围绕这几个点展开

![](/images/wedrive_4part.png)

微盘基础功能介绍


# 数据格式

以下Remote-FileInfo是从服务器拉取的文件数据格式，Local-FileInfo是在本地数据库存储的文件数据格式，包含了Remote-FileInfo以及一些本地信息，比如路径信息，文件mtime，inode。

+ Remote-FileInfo: 
  + FileId：文件唯一标志ID
  + Filename: 文件名
  + ParentId：路径ID
  + SpaceId: 分区ID
  + Size：文件大小
  + Status: 是否删除
  + Type：类型（文件夹、文件、微文档）
  + Md5：文件MD5
  + Seq：文件在整个树中的sequence，每个文件修改内容/重命名/移动/删除，seq都会增加，而且整棵树是共同维护一个递增的seq，用于提供给客户端增量拉取数据。后面会详细介绍。
+ Local-FileInfo：包含Remote-FileInfo的字段，以及一些本地使用的字段
  + Remote-FileInfo
  + Relative Path： A/B/C，用于和本地文件系统映射
  + Relative ID： id1/id2/id3，用于一次性查询出文件夹中都有什么什么文件
  + Mtime：文件在本地的mtime
  + Inode：文件在本地文件系统中的唯一ID，用于实现后续的重命名和移动事件的识别


文件基本操作有以下五类
+ 新建：新增一条文件索引
+ 删除：修改Remote-FileInfo.Status
+ 重命名: 修改Remote-FileInfo.Filename
+ 移动: 修改Remote-FileInfo.ParentId
+ 修改内容: 修改Remote-FileInfo.Md5以及Size
  
以上文件基本操作同步到服务器时，服务器会相应的修改Remote-FileInfo.seq的值，将它递增，但不是简单的加1，而是整个空间的所有的文件索引维护一个共同的递增seq。

客户端增量拉取文件树，也是通过这个seq来实现的。我们可以看一下下面一个简单的例子

三个文件夹a/b/c，最里面是一个文件d.txt，d.txt的绝对路径是a/b/c/d.txt

Status：1=存在, 2=删除
Type 1=文件, 2=目录
文件夹类型没有md5和size的值

服务器上存储了下面这份数据。

| 属性 | 意义 |
| ---------- | :-----------: |
| fileId | 文件唯一标志 |
| filename | 文件名 | 
| parentId | 父目录Id |
| type | 文件类型：1 = 文件 2 = 文件夹 |
| status | 是否删除： 1 = 存在 2 = 已被删除 |
| md5 | md5 |
| size | size |
| seq | 文件发生任何修改，重命名/移动/删除/修改内容/新增文件，都在当前空间所有文件的最大seq基础上加1 |
| reletivePath | 相对路径，用于索引数据和文件系统数据的映射 |
| reletiveId | 相对路径ID，辅助状态位计算，快速获取文件夹内所有文件, reletiveId LIKE %idx%|
| mtime | 文件下载到本地后，记录文件的mtime到数据库，用于校验本地文件是否发生修改 |
| inode | 文件在文件系统当前分区的唯一iD，用于监听模块 |







| FileId | Filename | ParentId | Status | Type| Md5 | Size |Seq   |
| ---------- | :-----------: |:-----------: |:-----------: |:-----------: |:-----------: |:-----------: |
| id1 |a | id0  |1| 2 |            |    |1 |
| id2 |b | id1  |1| 2 |            |    |2 |
| id3 |c | id2  |1| 2 |            |    |3 |
| id4 |d.txt | id3  |1| 1 | xxxxmd51   |100 |4 |

甲用户和乙用户都第一次安装应用，他们都是拿seq=0去请求数据，服务器返回seq>0的所有文件索引，所以两位用户都拉取到上面这份数据，得到新的最大seq=4，下次拉取数据就拿seq=4去请求了。

甲用户做了以下几件事

+ 重命名：a -> a1
  + a.seq = 5
  + a.name = a1
+ 修改内容：
  + d.seq = 6
  + d.md5 = xxxxmd52
  + d.size = 200
+ 移动：把c文件夹从b移动到a目录下
  + c.seq = 7
  + c.parentId = id1 
+ 删除：删除b文件夹
  + b.seq = 8
  + b.status = 2

甲用户做了以上四步操作，修改相关字段，同步给服务器，这个时候服务器的数据是这样的。

| FileId | Filename | ParentId | Status | Type| Md5 | Size |Seq   |
| ---------- | :-----------: |:-----------: |:-----------: |:-----------: |:-----------: |:-----------: |
| id1 |a1 | id0  |1| 2 |            |    |5 |
| id2 |b | id1  |2| 2 |            |    |8 |
| id3 |c | id1  |1| 2 |            |    |7 |
| id4 |d.txt | id3  |1| 1 | xxxxmd52   |200 |6 |

这个时候乙用户拿本地最大seq=4去请求，后台返回所有seq>4的数据，乙用户就可以拉到最新数据了。

我们把所有文件索引同步到本地之后，可以理解为是把服务器上的一棵文件树拉取并保存到了我们的本地数据库，我们下一步要做的事情，就是把这棵树映射到文件系统上。为了完成这个目标，我们需要来构建这颗树的路径信息。

接下来做一项很重要的工作


![](/images/wedrive_pic_01.png)


# 路径构建与解析

我们从服务器拉取的文件索引信息，只能得知这个文件在哪个目录下，但是如果想要知道某个文件索引映射到文件系统后的完整路径，需要层层递归查询到文件树的根部。如果这个路径最终只是用于保存下载文件，或者创建本地文件夹，那我们可以在执行下载/创建时才创建，因为这些行为频率并不高。但是在其他需求场景下，这种处理方式的效率和性能，都是是不合理的。我们举两个例子：

1、盘符监听
我们的盘符监听模块，会从文件系统获取到一个事件以及该事件对应的文件在本地的绝对路径，那这个路径关联我们应用里具体哪一条文件索引呢？文件触发事件的频率是很高的，我们不可能每次都去做复杂的数据库查询来获取文件索引，所以我们需要快速地通过一条绝对路径查询

2、文件夹状态
在我们的客户端和盘符内，文件夹都有一个状态图标，
![](/images/wedrive_folderstate.png)

云朵表示内部还有未下载文件，绿色表示文件夹内部全部文件已经下载到本地。为了实时更新文件夹的状态图标，我们需要一个方式，快速获取文件夹下有什么文件。

很明显，我们的文件路径信息是查多改少的数据，那么我们就有必要预先构建好这些文件的路径信息。所以我们在数据库里会冗余存储两列。

```
relativePath: filenameA/filenameB/filenameC
relativeId: iDA/iDB/iDC
```

relativePath可以用来将文件系统与文件索引相互映射，而relativeId则可以快速计算文件夹状态，比如，如果要计算filenameA文件夹下有什么什么文件，假设filenameA的fileID=iDA，我们只需要一条简单地sql，任何relativeId包含iDA的文件，都在filenameA目录下。

以上就是我们构建路径的目的，那么如何高效地构建路径呢？


每个文件在本地的绝对路径由以下四个部分组成

> 微盘根目录 + 企业名称 + 空间名称 + 文件相对路径

前面三部分都很简单，可以单独维护，最麻烦的是最后一部分：文件相对路径。我们尝试过两种方式构建

## 为每个节点递归寻找parent

遍历每个节点，然后根据当前节点的ParentId寻找它的Parent，这样递归寻找每个祖先节点，直到递归到树的根部（SpaceId == ParentId）停下来。这种算法时间复杂度比较糟糕，特别是当树退化成一个链表时（每一级都只有一个文件），这个时候时间复杂度是Ο(n^2)的 

## 从树的根部开始构建，向下传递

我们知道，假设某个目录的相对路径是X，则目录下地文件A地相对路径一定时X/A，所以，我们其实可以从文件树地根部开始为每个文件构建相对路径，按将相对路径传递给下一级，这种方式下，计算整棵树的复杂度就是Ο(n)

## 如何增量更新相对路径


有了路径构建模块，就需要一个对应地路径解析模块，二者关系请看下路

![](/images/wedrive_pathbuilder.png)


# 增量更新文件树

当客户端从服务器拉取到文件系统的更新时，我们可以通过对比服务器返回的数据和数据库中的数据，识别出各个更新文件，以及它们的更新类型。一般有以下五种

1、新增
2、删除 
3、重命名
4、移动
5、内容更新

当文件系统中只有一个文件发生变化时，我们处理这总更新时比较简单的，但是，当有多个文件发生更新，甚至一个文件发生多种更新事件，发生更新的文件具有上下级关系时，对文件系统做更新就显得比较复杂。

举一个比较直观的例子

![](/images/wedrive10.png)

除了以上复杂的变形，可能还有其他因素带来干扰，比如，

1、Windows上，某些软件打开了一个文件，则这个文件无法被重命名、移动、删除，以及这个文件路径上的所有父文件夹都无法做重命名、移动、删除
2、一个文件可能在做上传下载，如果将本地文件重命名、移动了，上传下载任务失败了怎么办

我们先从文件树的变形开始讲起，直接给结论，我们最新的算法大概是这样。





# 如果把这个任务丢给后台做呢

# 更新本地文件



# 分级拉取的难点与实现

如果分级拉取，会有什么问题

1、图标状态怎么计算？后天不可能帮你计算，因为是每个用户自己的
2、除了顾及客户端，和数据库，还得顾及盘符文件的去向

# 同步形态

向下

server ------> db -------> fs

向上（盘符）

fs(watcher) ------> server -------> db

向上（客户端）

client -------> server -------> db -------> fs 

## node的文件监听是怎么实现的呢？

说到文件监听，可能大家最开始想到的linux中的select/poll/epoll这一套东西，不过epoll这一套是用于IO多路复用，比如监听一个socket文件是否已经有内容可以读了。而我们这里要监听的，是文件内容更新，重命名，新增文件，删除文件等等。而现代的文件系统都从底层支持了文件变化的监听，如下图

|       System     | File Monitor     |
| ---------- | :-----------: |
| Windows     |  ReadDirectoryChangesW     |
| Darwin     | FSEvents or kqueue     |
| BSDs     |  kqueue     |
| Linux     |  inotify     |
| Solaris     |  event ports     |

nodejs底层的libuv就对上面这些接口进行了封装，这里也是libuv中碎片化最严重的一个模块。我们的应用运行于Mac OS X和Windows系统上，所以我们着重来讲讲它们使用的FSEvents，kqueue，ReadDirectoryChangesW

我们的系统是运行在Electron5.0.8之上，其中node和libuv依赖和版本关系是这样（electron内嵌了一套node）

`electron 5.0.8 -> node 12.00 -> libuv 1.27.0`

|       系统     | 实现     | 
| ---------- | :-----------: |
| Windows     |  ReadDirectoryChangesW |
| MacOS 10.7以下 | kqueue |
| MacOS 10.7及以上   | FSEvents |

## Mac OS X - kqueue
kqueue是BSD上开发的对应Linux上epoll的一套接口，不过它涵盖的功能比epoll更宽广。还另外还支持文件内容监听，跨进程通信等等，如下图。

![](/images/wedrive_kqueue.png)

但是拿kqueue来做一整个文件树的监听的话，每个文件夹都需要打开一个文件句柄(file descriptor)，而每个进程能打开的文件句柄数量是有限的，所以kqueue作为任意大小的文件树的监听，并不合适。

## Mac OS X - FSEvents
Mac OS X后来推出了新的文件监听方案：FSEvents，这套东西比较有趣，我们稍微花点篇幅介绍一下它的原理。你是否偶然注意到，Mac上有这么一个fseventsd的系统进程。

![](/images/wedrive_fsevent01.png)

用户空间发生对文件的修改时，该修改记录到文件系统后，文件系统将文件的修改事件记录到/dev/fsevent文件，这是一个符号设备文件，然后系统进程fsevent读取/dev/fsevent文件中的事件，系统进程fsevent再和其他通过FSEvent API注册了文件监听的应用程序通信，将它们感兴趣的事件传递过去。

![](/images/wedrive_fsevents.png)

Mac上文件系统中的每个分卷的根目录都有这样一个隐藏目录: .fseventsd
![](/images/wedrive_fsevent02.png)

这个文件夹里的文件是记录当前分区所有文件事件的日志，这些日志类似于NTFS 的USD日志，.fseventsd里的日志大有用途，比如Mac的Time Machine在做备份时，就可以从.fseventsd日志中读取上次备份事件点之后发生变化的文件，针对这些文件做备份，而如果由于其他原因，.fseventsd日志丢失了，Time Machine则需要做一次全盘扫描，依靠每个文件的modification time来找出上次备份之后发生变化的所有文件。.fseventsd日志格式不难理解，在看雪论坛挖到一篇关于
.fseventsd日志解析的文章，挺有意思的，其中引申到.fseventsd日志的取证用途以及这种文件日志带来的安全和隐私的问题。

[苹果FSEvent深层文件系统调用记录方法论](https://bbs.pediy.com/thread-221578.htm)

<!-- inotify on Linux, FSEvents on Darwin, kqueue on BSDs, ReadDirectoryChangesW on Windows, event ports on Solaris, unsupported on Cygwin -->

目前我们使用的版本依赖时这样
`electron 5.0.8 -> node 12.00 -> libuv 1.27.0`

libuv 1.27.0中对Mac的文件监听的实现，在macos 10.7以下使用Kqueue，macos 10.7及以上使用fsevent。


## Windows - ReadDirectoryChangesW

ReadDirectoryChangesW是，Windows提供的文件监听接口，系统版本兼容性比较好，从XP就开始支持了。

# 如何使用node的文件监听

```javascript
fs.watch('dir', { recursive: true }, (event, filename) => {
  
})
```
以上接口的使用很简单，我们来讲讲它的不足，其中的event只有两种值

+ rename: 表示新建文件或者删除文件
+ change: 文件内容修改

很明显欠缺了重命名和移动事件，而我们需要的事件有以下几种，下面的文件代指文件和文件夹

+ 文件新建
+ 文件删除
+ 文件内容修改（不包含文件夹，虽然修改文件夹的扩展属性也会触发change事件）
+ 文件重命名
+ 文件移动

重命名和移动一个文件，我们都会收到两个事件，首先是一个删除事件，然后是一个新建事件，而且两个事件相邻很近一起发生，我们可以对比这两个事件对应的文件inode（可以理解为文件系统中文件的唯一ID），如果是一样的，则代表这两个事件是重命名或者移动，重命名和移动取决于是不是通个目录下的两个文件。

除了事件类型不齐全之外，我们还面临以下几个问题

+ 文件的修改可能来自用户在finder/explorer中，也可能是来自我们的客户端内，这些都会触发事件，客户端内的操作触发的事件我们是需要过滤的，但是这两种操作的触发的事件是一模一样的，我们并不能识别出来，于是需要在客户端的文件操作的相关业务点，预埋一个白名单，而且白名单里的数据是一次性的，命中一次就移除

+ 很多编辑器会带来一些临时文件，以及一些文件系统相关的临时文件，这些文件是可以不同步，这个需要维护一个白名单，目前这方面的白名单，大概有下面这些

```javascript
  // finder/explore临时文件
  /^.DS_Store$/,
  /^thumbs.db$/,
  /^desktop.ini$/,
  // office临时文件
  /^~\$/,
  /^.~/,
  /^~.+\.tmp$/,
  // vim临时文件
  /.swp$/,
  /.swo$/,
  /.swn$/,
  // windows下，.字符结尾文件是特殊文件，空格结尾也是，这些文件无法手动删除
  /\.$/,
  / $/,
  // .WeDrive是项目私有隐藏文件
  /^.WeDrive$/
```
综上，我们整体的监听模块，可以用下图概括

![](/images/wedrive_fileevent.png)




## 写作笔记、素材
不能一味按照dropbox功能实现来写
如果介绍处微盘和dropbox的不同

usn是NTFS用于记录文件变更的日志，everything就是基于它做搜索的
https://docs.microsoft.com/en-us/windows/win32/fileio/change-journals

kqueue会占用大量fd,(待验证)
https://news.ycombinator.com/item?id=9063477


fsevent !!
OS X 10.5和10.6仅能捕获与文件夹相关的事件， 从OS X 10.7开始引入文件事件。 类似于Windows系统的NTFS日志（该日志记录一直文件系统活动，并将数据存储在UsnJrnl:$J中），FSEvents也一直记录文件系统变化，并将数据存储在FSEvent日志文件中。
这篇很赞
https://bbs.pediy.com/thread-221578.htm
https://eclecticlight.co/2017/09/12/watching-macos-file-systems-fsevents-and-volume-journals/


https://docs.microsoft.com/en-us/windows/win32/fileio/change-journals



windows各个文件系统的特性支持
https://docs.microsoft.com/en-us/windows/win32/fileio/filesystem-functionality-comparison