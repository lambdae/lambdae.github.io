---
layout: post
title: 怎样实现一个多线程无锁的单例模式
description: 实现一个基本threadlocal的单例模式 
categories: multithread tech 
author: lambdae
---

* **概述**

    本文介绍一种基于threadlocal实现的单例模式，多线程调用同一个静态类方法操作各自私有内存对象。代码逻辑是统一的，但操作的内存对象是独立的，规避了资源共享的线程互斥。
    

* **实现**

    为便于验证所有代码都写在一个cc文件，c++代码有点搓，望海涵指正。

    ```c
    #include <map>
    #include <assert.h>
    #include <pthread.h>
    #include <thread>
    #include <mutex>
    #include <vector>
    #include <unistd.h>

    using std::map;
    #define TEST_ALLOC_LEN 100

    class singleton
    {
        public:
            typedef map<int, int> KvTag;
            // disable copy constructor
            singleton(const singleton &) = delete;
            singleton &operator=(const singleton &) = delete;

            static KvTag &tags();
            static void insert_tag(int k, int v);
            static int query_tag(int k);
            static void tls_addr();

        private:
            // thread local object
            struct tls
            {
                KvTag tags_;
                void* heap_ptr_;

                tls() {
                    heap_ptr_ = malloc(sizeof(int) * TEST_ALLOC_LEN);
                }
                ~tls() {
                    printf("call ~tls()\n");
                    free(heap_ptr_);
                }
            };

            static __thread tls *context_;
            // use for register threadlocal dctor per thread
            static pthread_once_t once_;
            singleton();
            static tls* context();
            static void reg_();
            static void init();
            static void dctor(void *obj);
    };

    __thread singleton::tls *singleton::context_ = NULL;
    pthread_once_t singleton::once_ = PTHREAD_ONCE_INIT;
    static pthread_key_t tls_key_;

    void singleton::init()
    {
        context_ = new singleton::tls();
        if (context_ != NULL) {
            pthread_once(&once_, reg_);
            pthread_setspecific(tls_key_, context_);
        }
    }

    void singleton::reg_()
    {
        pthread_key_create(&tls_key_, dctor);
    }

    void singleton::dctor(void *obj)
    {
        printf("call dctor()\n");
        singleton::tls *context = (singleton::tls*)obj;
        if (context != NULL) {
            delete context;
        }
    }

    singleton::tls *singleton::context()
    {
        if (context_ == NULL) {
            init();
        }
        assert(context_ != NULL);
        return context_;
    }

    singleton::KvTag& singleton::tags()
    {
        return context()->tags_;
    }

    void singleton::insert_tag(int k, int v)
    {
        context()->tags_.insert({k, v});
    }

    int singleton::query_tag(int k)
    {
        KvTag::iterator iter;
        iter = context()->tags_.find(k);
        if (iter != context()->tags_.end()) {
            return iter->second;
        }
        return -1;
    }

    void singleton::tls_addr()
    {
        printf("map addr:%p\nheap obj:%p\n", &context()->tags_, context()->heap_ptr_);
    }


    int main()
    {
        std::vector<std::thread> pool;
        const int nworker = 8;
        for (int i = 0; i < nworker; i++)
        {
            int k = i + 1;
            std::thread worker([k]() {
                    singleton::insert_tag(k, k + 1000);
                    int val = singleton::query_tag(k);
                    printf("%d : %d\n", k, val);
                    singleton::tls_addr();
            });
            pool.push_back(std::move(worker));
            // sleep(1);
        }

        for (int i = 0; i < nworker; i++) {
            printf("wait: %d\n", i);
            pool.at(i).join();
        }

        return 0;
    }
    ```


    编译运行以上代码，终端输出:

    ```c
    wait: 0
    5 : 1005
    3 : 1003
    6 : 1006
    7 : 1007
    8 : 1008
    2 : 1002
    1 : 1001
    4 : 1004
    map addr:0x7fca18d00390
    heap obj:0x7fca18d003b0
    map addr:0x7fca18e001d0
    heap obj:0x7fca18e001f0
    map addr:0x7fca18d00580
    heap obj:0x7fca18d005a0
    map addr:0x7fca18e003c0
    heap obj:0x7fca18e003e0
    map addr:0x7fca18d00770
    heap obj:0x7fca18d00790
    map addr:0x7fca18d00010
    heap obj:0x7fca18d00030
    map addr:0x7fca18e00010
    heap obj:0x7fca18e00030
    map addr:0x7fca18d001d0
    heap obj:0x7fca18d001f0
    call dctor()
    call dctor()
    call dctor()
    call dctor()
    call dctor()
    call dctor()
    call dctor()
    call dctor()
    call ~tls()
    call ~tls()
    call ~tls()
    call ~tls()
    call ~tls()
    call ~tls()
    call ~tls()
    call ~tls()
    wait: 1
    wait: 2
    wait: 3
    wait: 4
    wait: 5
    wait: 6
    wait: 7
    ```

    以上可得知，**pthread_key_create(&tls_key_, dctor)**注册了threadlocal的析构函数，当线程结束后，运行时自动调用**dctor**函数对各线程的threadlocal对象进行清理，释放threadlocal指向堆的内存空间。因为posix threadlocal仅支持简单类型的使用，想要在线程局部里使用复杂对象就必须使用指针指向堆申请的复杂对象，而pthread_key_create提供了清理threadlocal复杂对象的机制。
 

