---
layout: post
title: Data replication
description:  《Designing data-intensive application》是2018年入手的第一本技术书，粗略的看了Replication的内容。整体感觉质量很高，更偏向与理论上的归纳。作者经验丰富，常常会横向对比不同数据库的设计方法和理念，从时间成本上来看是很值得细细品味的一本书。
categories: books tech database distributed
author: lambdae
---

*  **引言**

    ![sample post](https://img1.doubanio.com/lpic/s29419939.jpg)
    
    《Designing data-intensive application》是2018年入手的第一本技术书，粗略的看Replication的内容。整体感觉质量很高，更偏向理论上的归纳。作者经验丰富，常常会横向对比不同数据库的设计方法和理念。从时间成本上来看是很值得细细品味的一本书，很多reference也有展开的必要。


*  **正文**

    第五章的Replication，主要介绍了以下几种常见的数据同步的模式：

    1. **single-leader replication**
    2. **multi-leader replication**
    3. **leaderless replication**

