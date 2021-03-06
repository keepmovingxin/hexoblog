---
title: Lua数据结构 — TString（二）
date: 2016-06-06 21:13:36
copyright: false
tags: [Lua, Lua数据结构]
categories: 游戏开发
toc: false
qrcode: true
---

存储lua里面的字符串的**TString数据结构**:（lobject.h 196-207）

![](/images/luaTstring/lua-tstring-01.png)


<!--more-->

其它结构中也会有**L_Umaxalign dummy**这个东西，来看看L_Umaxaliagn:

![](/images/luaTstring/lua-tstring-02.png)

从字面意思上就是保证**内存能与最大长度的类型进行对齐**，事实上也是做这件事，这里感觉lua想给各种不同设备做一种嵌入式脚本，这里要保证与最大的长度对齐能**保证CPU运行高效不会罢工**。

tsv才是TString的主要数据结构：

1. CommonHeader：这个是和GCObject能对应起来的GCheader
2. reserved：保留位
3. hash：每个字符串在创建的时候都会用有冲突的哈希算法获取哈希值以提高性能
4. len：字符串长度

哈希是lua里一个很重要的优化手段，具体的哈希算法相关知识在文章最后会补充说明一下，字符串的hast表放在L->l_G->strt中，这个成员的类型是stringtable，我们再来看看**stringtable数据结构**：（lstate.h 38-42）

![](/images/luaTstring/lua-tstring-03.png)

stringtable结构很简单：

1. hash：一个GCObject的表，在这里其实是个TString*数组
2. nuse：已经有的TString个数
3. size：hash表的大小(可动态扩充)

接下来看看stringtable是怎么动态调整大小的:
1、动态扩充stringtable：(lstring.c 60-70)

![](/images/luaTstring/lua-tstring-04.png)

每次newlstr的之后，都会判断nuse是否已经大于table的size，如果是的话就会重新resize这个stringtable的大小为原来的2倍。

2、动态浓缩stringtable：(lgc.c 433-436)

![](/images/luaTstring/lua-tstring-05.png)

在gc的时候，会判断nuse是否比size/4还小，如果是的话就重新resize这个stringtable的大小为原来的1/2倍。

3、resize算法：(lstring.c 22-47)

![](/images/luaTstring/lua-tstring-06.png)

resize时，需要根据每个节点的哈希值重新计算新位置，然后放到newhash里。

**字符串在哪里？** 看完TString和stringtable，大家都没有发现究竟字符串放在哪里，从内存上看其实字符串直接放在了**TString**后面，这样还能省掉一个成员：(lstring.c 56)

![](/images/luaTstring/lua-tstring-07.png)

性能问题：

在这里说一点lua的性能问题，虽然不在这个主题的讨论范围。由上面可以知道lua的字符串是带hash值的，所以我们拿着一个字符串去做比较、查询、传递等操作都是**非常高效**的。

但是我们也可以看到**每次创建一个新的字符串**都会做很多操作，所以这里**不建议频繁做字符串创建、连接、销毁等操作**，最好能**缓存**一下。

补充：字符串的哈希算法：

常用的字符串哈希函数比较如下：

![](/images/luaTstring/lua-tstring-08.png)

其中数据1为100000个字母和数字组成的随机串哈希冲突个数。数据2为100000个有意义的英文句子哈希冲突个数。数据3为数据1的哈希值与1000003(大素数)求模后存储到线性表中冲突的个数。数据4为数据1的哈希值与10000019(更大素数)求模后存储到线性表中冲突的个数。

经过比较，得出以上平均得分。平均数为平方平均数。可以发现，BKDRHash无论是在实际效果还是编码实现中，效果都是最突出的。APHash也是较为优秀的算法。DJBHash,JSHash,RSHash与SDBMHash各有千秋。PJWHash与ELFHash效果最差，但得分相似，其算法本质是相似的。

在Lua中使用到的是JSHash算法：

![](/images/luaTstring/lua-tstring-09.png)

具体JS Hash算法的冲突性解决和性能上面，我也不懂，具体要找paper看看，但是从数据比较上看，JSHash是属于较好的算法，可也有比JSHash算法更好的字符串哈希算法。

**Lua数据结构**系列转自阿里云博客，作者是**罗日健**。
原文链接：[http://blog.aliyun.com/768](http://blog.aliyun.com/768)

