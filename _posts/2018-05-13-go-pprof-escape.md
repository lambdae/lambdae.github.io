---
layout: post
title: 使用pprof分析变量逃逸过程
description:  Golang 变量逃逸
categories: golang pprof tech
author: lambdae
---


*  **问题背景**

    在优化ac自动机时发现在匹配过程中有大量时间消耗在GC里面，通过pprof发现match过程有很多的临时变量逃逸到heap里，增加了很多的GC压力，简要记录下问题定位的过程。



* **问题定位**

    首先需要在测试程序添加生成pprof数据的代码段。

    ```go
    f, err := os.Create("benchmark.prof")
    if err != nil {
    	log.Fatal(err)
    }
    defer f.Close()
    pprof.StartCPUProfile(f)
    defer pprof.StopCPUProfile()

    go func() {
    	http.ListenAndServe(":8787", http.DefaultServeMux)
    }()

    ...
    // 等待一段时间做问题分析
    fmt.Println("\nCTL+C exit http pprof")
    time.Sleep(15 * time.Minute)
    ```

    查看各函数调用申请的内存对象大小。

    ```go
    go tool pprof -alloc_space -svg http://localhost:8787/debug/pprof/heap > ~/Desktop/go_heap.svg
    ```
    ![image](https://raw.githubusercontent.com/lambdae/lambdae.github.io/master/images/go_heap.png)

    我们发现matchOf申请了大量的内存，于是怀疑matchOf可能存在变量逃逸，使用-gcflags -m重新生成测试程序发现确实存在MatchToken临时变量逃逸到heap。

    ```go
    go build -gcflags -m   | grep escape

    ../acmatcher.go:165: &MatchToken literal escapes to heap
    ../acmatcher.go:165: &MatchToken literal escapes to heap
    ```
    ​

    ​



