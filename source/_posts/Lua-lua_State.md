---
title: Lua数据结构 — lua_State（六）
date: 2016-06-10 21:03:16
copyright: false
tags: [Lua, Lua数据结构]
categories: 游戏开发
toc: false
qrcode: true
---

前面各种Lua的数据类型基本都说得差不多了，剩下最后一个数据类型：**lua_State**，我们可以认为是**”脚本上下文”**，主要是包括当前脚本环境的运行状态信息，还会有gc相关的信息。

<!--more-->

Lua这门语言考虑了多线程的情况，在脚本空间中能够开多个线程相关脚本上下文，而大家会共用一个全局脚本状态数据，如下：

![](/images/luaState/lua-state-01.png)

全局数据global_state的数据结构如下：

![](/images/luaState/lua-state-02.png)

global_state主要是用于GC的数据链表，下面简要说明几个:

1. stringtable strt：这个是在TString那章说到的全局字符串哈希表
2. TValue lregistry：对应LUAREGISTRYINDEX的全局table.
3. TString *tmname[TM_N]：元方法的名称字符串。
4. Table *mt[NUM_TAGS]：基本类型的元表，这是Lua5.0的特性。

mt成员在作者介绍文章中说到:

![](/images/luaState/lua-state-03.png)

在上面代码中，我们看到a支持一个tostring的方法，a是数值类型，我们可以为数值类型添加任意的方法。Lua文章中说到一个用途，就是对于unicode和gbk的字符串的len方法能自己实现。

其它成员就不一一介绍了，下面来介绍与线程相关的脚本上下文lua_State：

![](/images/luaState/lua-state-04.png)

我们看到，luaState也带有CommonHeader头，在第一章中也提到了GCObject中有luaState th这个成员，由此可见lua_State也会是被回收的对象之一。

考虑回一个线程中的脚本上下文，我们再来逐个分析每个成员：

- lu_byte status：线程脚本的状态，线程可选状态如下：

![](/images/luaState/lua-state-05.png)

- StkId top：指向当前线程栈的栈顶指针，typedef TValue *StkId
- StkId base：指向当前函数运行的相对基位置，具体可参考第四章的闭包

- globalState *lG：指向全局状态的指针
- CallInfo *ci：当前线程运行的函数调用信息
- const Instruction *savedpc：函数调用前，记录上一个函数的pc位置
- StkId stack_last：栈的实际最后一个位置（栈的长度是动态增长的）
- StkId stack：栈底
- CallInfo *end_ci：指向函数调用栈的栈顶
- CallInfo *base_ci：指向函数调用栈的栈底
- int stacksize：栈的大小
- int size_ci：函数调用栈的大小
- unsigned short nCcalls：当前C函数的调用的深度
- unsigned short baseCcalls：用于记录每个线程状态的C函数调用深度的辅助成员
- lu_byte hookmask：支持哪些hook能力，有下列可选的

![](/images/luaState/lua-state-06.png)

- lu_byte allowhook：是否允许hook
- int basehookcount：用户设置的执行指令数(LUA_MASKCOUNT下有效)
- int hookcount：运行时，跑了多少条指令（LUA_MASKCOUNT下有效）
- lua_Hook：用户注册的hook回调函数
- TValue l_gt：当前线程的全局的环境表
- TValue env：当前运行的环境表
- GCObject *openupval、gclist：用于gc，详细将会在GC一章细说
- struct lua_longjmp *errorJmp：发生错误的长跳转位置，用于记录当函数发生错误时跳转出去的位置。

![](/images/luaState/lua-state-07.png)

本系列总结：

整个系列文章回答了我们对Lua中最基本的一个问题：“一个Lua变量究竟是什么？”。由此我们深入并引申出各种知识，在脚本中我们觉得弱类型变量用起来很痛快，而其实它的内部实现其实是如此的复杂。

对于实现一门脚本语言，必须实现的是解释器、虚拟机、上下文数据3大部分：

![](/images/luaState/lua-state-08.png)

上下文数据这一层是脚本最基础，最底层的东西，它决定了这门脚本究竟能做什么。抛开解释器和虚拟机，我们依然可以单纯地通过C接口，在C++这一层就能操作脚本的上下文数据。

有空再研究一下Lua的GC，解释器等等。

**Lua数据结构**系列转自阿里云博客，作者是**罗日健**。
原文链接：[http://blog.aliyun.com/795](http://blog.aliyun.com/795)

