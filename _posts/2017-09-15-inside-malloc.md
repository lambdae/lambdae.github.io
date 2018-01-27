---
layout: post
title: 简要分析Jemalloc与常见内存分配器
description:  横向对比各自类slab内存分配器，以及对内存管理的个人总结
categories: os slab tech
author: lambdae
---

*  **引言**

    工作中造了个的虐心小轮子--内存分配器，本来是想直接移植Jemalloc，但综合业务场景还真不太合适。主要原因是我们的API server的client上限是已知的，而且活跃的client长期维持在一个稳定的数量，api的key和value都小于4k。总的来说，server的内存需求是上限是固定，且主要是小内存分配。我们更希望做一个能预先分配好上限内存，超过上限时能向系统申请，长期闲置时能回收到系统。当然稍微修改下Jemalloc也能满足这些需求，但Jemalloc对大内存的管理和Tcache机制比较复杂，作者为了足够的高效用了各种trick的宏定义和数据结构。总的来说，我们的需求比较简单，引入复杂机制的代码后期定位问题的时候需要完整的理解Jemalloc，加大了工程维护的时间成本。因此，我们决定借鉴Jemalloc的核心思想针对业务做个可预分配的，可自定义的，支持多规模小size的内存分配器。结果也是很乐观的，在模拟正常业务的场景下对比了我们的实现和Jemalloc、TCmalloc、glibc的性能，提升了3-4倍时间效率（没有开启基于thread local实现的线程无锁内存分配机制）。这已经满足了基本需求，对于IO来说内存分配的时间损失还是太小了。


    Jemalloc很优秀维护快10年，作者尝试能想到的想不到的优化，各版本数据结构改动变化挺大的。以下算是浅显的对比各开源组件的slab实现和Jemalloc的异同，主要是对malloc()和free()的内在探究。有时间再总结下我们的slab实现，有很多地方和Jemalloc还是不同的。主要的我们的场景足够简单，不需要做成通用的模块。也希望能听到不同的声音，激励提高。毕竟这些资料都是一点一点查阅积累的，纰漏和错误难免，见谅！


* **简述**


![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片1.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片2.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片3.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片4.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片5.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片6.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片7.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片8.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片9.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片10.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片11.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片12.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片13.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片14.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片15.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片16.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片17.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片18.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片19.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片20.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片21.PNG)
![image](https://raw.githubusercontent.com/iohub/iohub.github.io/master/res/InsideJemalloc/幻灯片22.PNG)

