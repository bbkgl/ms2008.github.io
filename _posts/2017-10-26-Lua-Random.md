---
layout:     post
title:      Lua 中的随机数
subtitle:   随机数生成对我们太重要了，我们不能让它随机生成
date:       2017-10-26
author:     ms2008
header-img: img/post-bg-universe.jpg
catalog:    true
tags:
    - Lua
    - Random
    - LuaJIT
    - PRNG
---

Lua 随机数算法用的是 libc 中的 `rand`, 也就是 LCG。然而这个算法的随机性一般。尤其是在一些平台上，当随机种子变化非常小的时候，产生的随机数变化也非常小。这样再经过 Lua 的精度取舍之后，产生的随机序列仍然很相似（<u>伪随机的结果变成可预知性</u>）。

lua-l 上也讨论过这个问题 [msg00564](http://lua-users.org/lists/lua-l/2007-03/msg00564.html)，lua 的作者之一 @lhf 给出的解决方案是先弹出前面几个看起来「不怎么随机」的随机数。另外，作者也写过一个基于 MT 算法的 C lib: [lrandom](http://webserver2.tecgraf.puc-rio.br/~lhf/ftp/lua/#lrandom), 有兴趣的同学可以去看下。

然而在 [lua-wiki](http://lua-users.org/wiki/MathLibraryTutorial) 上有一种更为巧妙的实现（*这个用例同样是有缺陷的，这里只是为了引出上面的问题。以后我会单独讨论这个问题*）：

```lua
local seed = 123456
for i=1,2 do
    math.randomseed(seed + (i-1)/10)
    local num = {}
    for j=1,10 do
        table.insert(num, math.random(100))
    end
    print(table.concat(num, ","))
end

math.randomseed(tostring(123456.1):reverse())
local num3 = {}
for i=1,10 do
    table.insert(num3, math.random(100))
end
print(table.concat(num3, ","))
```

运行结果:

```
61,66,19,75,44,61,68,1,33,4
61,66,19,75,44,61,68,1,33,4
85,40,79,80,92,20,34,77,28,56
```

就是把变化较小的 seed 倒过来（低位变高位），再取高位 6 位。这样，即使 seed 变化很小，但是因为低位变了高位， 种子数值变化将会很大，就可以使伪随机序列生成的更好一些。

这里我也来介绍一个方法，效果也还不错（利用匿名 table 的地址来生成变化较大的 seed）：

```lua
math.randomseed(os.time()+assert(tonumber(tostring({}):sub(7))))
```

**值得庆幸的是 LuaJIT 已经将随机算法替换为 Tausworthe，也就是 LFSR (亦或 TURN)，循环长度达到 2<sup>223</sup>，并且能产生出质量更高的随机数。**

> **Enhanced PRNG for math.random()**
><br/><br/>
> LuaJIT uses a Tausworthe PRNG with period 2<sup>223</sup> to implement math.random() and math.randomseed(). The quality of the PRNG results is much superior compared to the standard Lua implementation which uses the platform-specific ANSI rand().

在 LuaJIT 再次运行上面的用例：

```lua
local seed = 123456
for i=1,2 do
    math.randomseed(seed + (i-1)/10)
    local num = {}
    for j=1,10 do
        table.insert(num, math.random(100))
    end
    print(table.concat(num, ","))
end
```

运行结果:

```
96,80,47,13,41,27,81,31,29,13
93,63,35,31,16,70,79,76,26,72
```

可以看到生成的随机数质量已经非常不错了。我们继续来缩小 seed 的差距，看下 LFSR 的表现如何：

```lua
local seed = 1.5089477744541
for i=1,2 do
    seed = seed + 0.0000000000001
    math.randomseed(seed)
    local num = {}
    for j=1,10 do
        table.insert(num, math.random(100))
    end
    print(table.concat(num, ","))
end
```

运行结果:

```
13,8,8,25,15,36,23,84,79,44
21,8,72,38,74,60,73,6,79,8
```

可以看到当 seed 变化非常小的时候 LFSR 同样会表现不俗。综合来说，LFSR 已经是一个很好的随机算法，可足够快速地产生健壮随机数。<u>但是其并不安全（包括 MT 算法，因为 MT 也是基于 LFSR 的），在安全因素比重很大的地方比如 csrf 或密码重置的 token 等等，应该尽量避免使用</u>。