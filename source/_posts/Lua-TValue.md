---
title: Lua数据结构 — TValue（一）
date: 2016-06-05 22:06:28
copyright: false
tags: [Lua, Lua数据结构]
categories: 游戏开发
toc: false
qrcode: true
---

数据结构的设计，在一定程度上奠定了整个系统的设计，所以决定写一个对Lua主要数据结构的分析文章，本来打算写一篇就好了，但是每个数据类型其实都有点复杂，一篇的话篇幅太长，所以就拆开几篇来写了。

<!--more-->

为什么是从TValue说起，**TValue是实现Lua弱数据类型的主要数据结构**，不但在脚本中的值使用了TValue，连Lua的实现中，很多数据结构也依赖于TValue，TValue一定程度上贯穿了整个Lua。先说一下Lua里面的**数据类型**:（lua.h :69）

![](/images/luaTValue/lua-tvalue-01.png)

从上面的定义中可以看到，**Lua的值类型有9种**，其中LUA_TNONE是用于判断这个变量是否等于为空使用的，这个是Lua内部使用的，后面再详细说明。现在来看Lua里面的**TValue数据结构**:（lobject.h 71-75）

![](/images/luaTValue/lua-tvalue-02.png)

在Lua里面，一个变量使用TValue这个类型来存储的，int tt就是上面宏的类型值（4个字节），而Value则是一个union（8个字节）。在这个union中，其实分工也十分明确:

![](/images/luaTValue/lua-tvalue-03.png)

在Value中，void\* p、lua_Number n、int b都是不用回收的值类型，而GCObject\* gc则都是需要回收的对象，下面是**GCObject数据结构**:（lstate.h 133-145）

![](/images/luaTValue/lua-tvalue-04.png)

GCObject也是一个union，存储了一个GCheader，这个GCHeader主要用于**GC回收机制**使用，GC回收机制超出了这次讨论话题，暂时先忽略。真正存储值的结构是TString、Udata、Closure等等，每个存储数据的结构都会有GCheader，接下来几篇文章将会开始逐个数据类型进行解释。

![](/images/luaTValue/lua-tvalue-05.png)

**Lua数据结构**系列转自阿里云博客，作者是**罗日健**。
原文链接：[http://blog.aliyun.com/761](http://blog.aliyun.com/761)

