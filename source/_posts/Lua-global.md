---
title: Lua 全局变量的那些事儿
date: 2016-05-27 22:30:12
copyright: false
tags: Lua
categories: 游戏开发
toc: true
qrcode: true
---

最近项目查了一个问题，最后发现和`_G[moduleName]`这个置为`nil`有关系，找了点资料看看里面的坑还是蛮深的，所以记录一下。

### 全局环境表 `_G`
Lua把所有的全局变量都放在一个称为全局环境的表_G中，这个表只是个普通的表。注意`_G._G == _G`。
由于`_G`是一个普通的表，所以提供了以动态名称访问全局变量的形式，这又是Lua的一种对元编程的支持。

如`_G[varname] = value`，更一般的问题是允许使用动态字段名，如_G["read.io"]默认是不会取出read模块的io字段的，但是使用下面这样实现：

<!--more-->

```
function getfield(f)  
    local v = _G
    for w in string.gmatch(f, "[%w_]+") do
        v = v[w]
    end
    return v
end

function setfield(t, v)  
    local t = _G
    for w, d in string.gmatch(f, "([%w_]+)(%.?)") do
        if d == "." then
            t[w] = t[w] or {}
            t = t[w]
        else
            t[w] = v
        end
    end
end
```

### 全局变量声明
全局变量不需要声明，虽然这对一些小程序来说很方便，但程序很大时，一个简单的拼写错误可能引起bug并且很难发现。然而，如果我们喜欢，我们可以改变这种行为。因为Lua所有的全局变量都保存在一个普通的表中，我们可以使用metatables来改变访问全局变量的行为。
第一个方法如下：

```
setmetatable(_G, {
    __newindex = function (_, n)
       error("attempt to write to undeclared variable "..n, 2)
    end,
   
    __index = function (_, n)
       error("attempt to read undeclared variable "..n, 2)
    end,
})
```

这样一来，任何企图访问一个不存在的全局变量的操作都会引起错误：

```
> a = 1
stdin:1: attempt to write to undeclared variable a
```

但是我们如何声明一个新的变量呢？使用rawset，可以绕过metamethod：

```
function declare (name, initval)
    rawset(_G, name, initval or false)
end
```

or 带有 false 是为了保证新的全局变量不会为 nil。注意：你应该在安装访问控制以前（before installing the access control）定义这个函数，否则将得到错误信息：毕竟你是在企图创建一个新的全局声明。只要刚才那个函数在正确的地方，你就可以控制你的全局变量了：

```
> a = 1
stdin:1: attempt to write to undeclared variable a
> declare "a"
> a = 1       -- OK
```

但是现在，为了测试一个变量是否存在，我们不能简单的比较他是否为nil。如果他是nil访问将抛出错误。所以，我们使用rawget绕过metamethod：

```
if rawget(_G, var) == nil then
    -- 'var' is undeclared
    ...
end
```

改变控制允许全局变量可以为nil也不难，所有我们需要的是创建一个辅助表用来保存所有已经声明的变量的名字。不管什么时候metamethod被调用的时候，他会检查这张辅助表看变量是否已经存在。代码如下：

```
local declaredNames = {}
function declare (name, initval)
    rawset(_G, name, initval)
    declaredNames[name] = true
end
setmetatable(_G, {
    __newindex = function (t, n, v)
    if not declaredNames[n] then
       error("attempt to write to undeclared var. "..n, 2)
    else
       rawset(t, n, v)   -- do the actual set
    end
end,
    __index = function (_, n)
    if not declaredNames[n] then
       error("attempt to read undeclared var. "..n, 2)
    else
       return nil
    end
end,
})
```

两种实现方式，代价都很小可以忽略不计的。第一种解决方法：`metamethods`在平常操作中不会被调用。第二种解决方法：他们可能被调用，不过当且仅当访问一个值为nil的变量时。

### 非全局变量 `_ENV`
Lua中其实没有真正的全局变量，一个chunk会编译类似下面的代码：
```
local _ENV = the global environment  
function  (...)  
    _ENV.var1 = _ENV.var2 + 3
end 
```
也就是说Lua5.2中，会这样处理全局变量： 
1. 在一个名为`_ENV`的`upvalue`作用域中编译`chunk` 
2. 将所有的自由名称转换成`_ENV.var`
3. load或loadfile函数用全局环境初始化`_ENV`

也可以通过显式使用`_EVN`来访问被遮蔽的全局变量：

```
a = 13  
local a = 12  
print(a)      --> 12  
print(_ENV.a) --> 13  
```

`_ENV`最大的作用就是修改代码片的环境。如使用_ENV可以限制代码对全局变量的访问：

```
local print, sin = print, math.sin  
_ENV = nil  
print(1) --> 1  
print(math.cos(13)) -- error  
```

想要修改环境的同时还能访问全局变量，通常说方法如下：

```
a = 1  
local newgt = ()  
setmetatable(newgt, {__index = _G})  
_ENV = newgt  
print(a)    --> 1  
a = 10  
print(a)    --> 10  
print(_G.a) --> 1  
_G.a = 20  
print(_G.a) --> 20  
```

我们还可以为函数定义私有执行环境：

```
function factory (_ENV)  
   return function ()
      return a
   end
end  
print(factory{a = 6}()) -> 6  
print(factory{a = 7}()) -> 7  
```

load函数有一个可选参数，可以由用户指定_ENV，这样就可以限制外部的运行环境。如果一个chunk加载后要以不同的环境多次运行，这时就不能通过参数在加载时指定环境了，这时可以每次都使用debug.setupvalue函数指定运行环境，另外，为了避免使用debug库，还可以像下面这样：

```
f = loadwithprefix("local _ENV = ...", io.lines(filename, "*L")) -- 因为Lua会把chunk编译成可参数函数，所以前面的这一句相当于把chunk的第一个参数赋值给_ENV  
...
env = {}  
f(env)  -- 使用env作为环境调用chunk  
```

### 全局变量与环境
lua中真正存储全局变量的地方不是在_G里面，而是在`setfenv（i,table）`的table中，所有当前的全局变量都在这里面找，只不过在程序开始时lua会默认先设置一个变量
`_G`=这个里面的table而已。所以在新设置环境后，如果还想找到之前的全局变量，通常需要附加上为新的table设置元表`{_index=_G}`

下面的几个例子：
```
a=1
print(a)
print(_G.a)
```
正常情况，输出1,1
```
a=1
setfenv(1,{})
print(a)
print(_G.a)
```
这时会出错说找不到`print`，因为当前的全局变量表示空的，啥也找不到的
```
a=1
setfenv(1,{_G=_G})
_G.print(_G.a)
print(a)
```
这时`_G.print(_G.a)`可以正常吗，因为可以在新的table中找到一个叫`_G`的表，这个`_G`有之前的奈尔全局变量，但是下面的`print(a)`则找不到`print`，因为当前的`table{_G=_G}`没有一个叫`print`的东西

```
local mt={__index=_G}
local t={}
setmetatable(t,mt)
setfenv(1,t)
print(a)
print(_G.a)
```
这是正确输出，因为新的全局表采用之前的表做找不到时的索引，原先的表里面存在`print` 、`_G`、 `a`这些东西
`setfenv`的第一个参数可以是当前的堆栈层次，如1代表当前代码块，2表调用当前的上一层，也可以是具体的那个函数名，表示在那个函数里。
每个新创建的函数都将继承创建它的那个函数的全局环境

#### require
`require`的意义就是导入一堆可用的名称，这些名称（非local的）都包含在一个table中，这个table再被包含在当前的全局表（“通常的那个`_G`”）中，这样访问一个模块中的变量就可以使用`_G.table.**`了，（刚开始学习lua时还以为模块里的名称在导入后直接就是在`_G`中的）
`a=require("")`的a取决于这个导入的文件的返回值，没有返回值时`true`，所以在标准的情况下模块的结尾应该`return`这个模块的名字，这样a就是这个模块的table了（当然不这样做也ok，只是a就不是这个模块名了）

### 理解 `_ENV` 和 `_G`
5.1之前, 全局变量存储在`_G`这个table中, 这样的操作:
`a = 1 `
相当于：
```
_G['a'] = 1
```

但在5.2之后， 引入了`_ENV`叫做环境，与`_G`全局变量表产生了一些混淆，需要从原理上做一个理解。
在5.2中， 
操作`a = 1`
相当于

```
_ENV['a'] = 1
```

这是一个最基础的认知改变，其次要格外注意`_ENV`不是全局变量，而是一个`upvalue`(非局部变量)。

其次，`_ENV['_G']`指向了`_ENV`自身，这一目的是为了兼容5.1之前的版本，因为之前你也许会用到：

`_G['a'] = 2` ， 在5.2中， 这相当于`_ENV['_G']['a']`，为了避免5.1之前的老代码在5.2中运行错误，所以5.2设置了`_ENV['_G']=_ENV`来兼容这个问题。然而你不要忘记`_ENV['_G']=_ENV`，所以一切都顺理成章了。

在5.1中，我们可以为一段代码块（或者函数）设置环境，使用函数`setfuncs`，这样会导致那一段代码/函数访问全局变量的时候使用了`setfuncs`指定的table，而不是全局的`_G`。

在5.2中，`setfuncs`遭到了废弃，因为引入了`_ENV`。 通过在函数定义前覆盖`_ENV`变量即可为函数定义设置一个全新的环境，比如：

```
a = 3
function get_echo()
local _ENV={print=print, a = 2}
return function echo()
print(a)
end
end

get_echo()()
```

会打印2，而不是3，因为`echo`函数的环境被修改为`{print=print, a=2}`，而`print(a)`相当于访问`_ENV['a']`（先忘掉那为了兼容而存在的`_G`）。
这就是`_ENV`的基本用法了。
另外，不得不提到lua的C支持中关于全局变量与环境的细节，只能简单描述，你必须自己试试才能记得清楚。

`lua_setglobal/lua_getglobal`都是操作`lua_State`注册表中`LUA_RIDX_GLOBALS`伪索引指向的全局变量表，与lua中访问`_ENV['a']`或者`a`是不同的。

`lua_load`加载lua代码后会返回一个函数，默认会给这个函数设置一个`upvalue`就叫`_ENV`，起值是`LUA_RIDX_GLOBALS`的全局变量表，你可以`lua_setupvalue`设置这个函数的`upvalue`，即下标1的`upvalue`，因为这个位置是这个函数的`_ENV`表存放位置（你可以通过`lua_setupvalue`的返回值印证这一点）

这里巧妙的是，`lua_State`会在创建时保证`LUA_RIDX_GLOBALS`的全局变量表中包含一个指向自己的`_G`元素，这样就保证了在不调用`lua_setupvalue`的情况下该返回函数的`_ENV['_G']`是指向自己的，即`LUA_RIDX_GLOBALS`这个全局表。（其实你的lua解释器就是简单的`lua_load`后`pcall`的，对于一个刚启动`lua_State`来说是没有`_ENV`的，是lua解释器load你的代码时自动给带上的`_ENV`，其值是`lua_state`的`LUA_RIDX_GLOBALS`全局表。）

一些有意思的东西是需要你自己摸索的，lua语言自身就很简练，并且所有东西都不是什么神秘的事情，可以通过读源码或者试验摸索得到。

最后，提一下，`lua_state`启动后在注册表里`LUA_RIDX_GLOBALS`下标存放的全局表一定有一个元素是指向自己的，即`_G`.

### 参考
https://www.lua.org/manual/5.1/manual.html
http://www.shouce.ren/api/lua/5/_88.htm
http://blog.csdn.net/leonwei/article/details/7739930
http://blog.aforget.net/shen-ru-luahe-czhi-si/



