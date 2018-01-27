---
layout: post
title: Nginx如何通过CAS操作解决惊群效应
description:  简述nginx如何通过CAS实现用户态的try_lock操作解决潜在的惊群效应
categories: tech os
author: lambdae
---

- **引言**

 
    惊群效应：当多个进程同事监听一个fd时，当fd相关的事件触发会唤醒多个进程去竞争该fd，但能获取到该事件的只有一个进程。如果我们使用mutex进行临界区保护的话，未抢到fd的进程就会睡眠。

    实际上，现在的linux内核已经处理accept()可能引起的惊群效应，但nginx为支持多平台，自己处理了可能触发的惊群效应。在nginx中，当事件触发时，多个进程会调用**ngx_trylock_accept_mutex()**尝试获取该事件。虽然nginx命名叫xxx_mutex(), 但实际上是由**CAS**操作实现的spin_trylock.即每个进程都尝试获取该事件。未能获取的就继续处理自己的事务。实际上尝试获取锁的代价仅仅是一个**CAS**汇编指令。


- **调用栈**

    **ngx_trylock_accept_mutex()**


    ```c
  ngx_int_t
  ngx_trylock_accept_mutex(ngx_cycle_t *cycle)
  {
      if (ngx_shmtx_trylock(&ngx_accept_mutex)) {

          ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                         "accept mutex locked");

          if (ngx_accept_mutex_held && ngx_accept_events == 0) {
              return NGX_OK;
          }

          if (ngx_enable_accept_events(cycle) == NGX_ERROR) {
              ngx_shmtx_unlock(&ngx_accept_mutex);
              return NGX_ERROR;
          }

          ngx_accept_events = 0;
          ngx_accept_mutex_held = 1;

          return NGX_OK;
      }

      ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                     "accept mutex lock failed: %ui", ngx_accept_mutex_held);

      if (ngx_accept_mutex_held) {
          if (ngx_disable_accept_events(cycle, 0) == NGX_ERROR) {
              return NGX_ERROR;
          }

          ngx_accept_mutex_held = 0;
      }

      return NGX_OK;
  }
  ```
   **ngx_shmtx_trylock()**
   ```c
  ngx_uint_t
  ngx_shmtx_trylock(ngx_shmtx_t *mtx)
  {
      return (*mtx->lock == 0 && ngx_atomic_cmp_set(mtx->lock, 0, ngx_pid));
  }
  ```
  **ngx_atomic_cmp_set()**
  ```c
  static ngx_inline ngx_atomic_uint_t
  ngx_atomic_cmp_set(ngx_atomic_t *lock, ngx_atomic_uint_t old,
      ngx_atomic_uint_t set)
  {
      u_char  res;

      __asm__ volatile (

           NGX_SMP_LOCK
      "    cmpxchgq  %3, %1;   "
      "    sete      %0;       "

      : "=a" (res) : "m" (*lock), "a" (old), "r" (set) : "cc", "memory");

      return res;
  }
    ```
    最终就是CAS操作了。模型简化为，epoll唤醒多进程同时做CAS操作，成功的就处理该事件，失败的就继续处理自己的事务，惊群效应就这样完全在用户态解决了。

---

- **总结**
    
    其实在上述场景中，nginx是实现了spinlock的一部分，有兴趣的可以去深入了解下spinlock的实现细节，其中会涉及到中断处理和PAUSE指令的优化等。其实个人探究spinlock的出发点是，在我们的产品中有切换大流量分类DFA的需求，backgroud线程（生产者）会异步的构造这个DFA, 构造完成后就会赋值给临界区的指针变量。为了保证DFA规则的实时性和多线程的安全性。消费者每次在回调时做CAS操作尝试获取最新的DFA指针。我们知道指针的读取和赋值的时间是足够短的（几条汇编指令）。在不影响流量吞吐的情况下，我们也是采用了和nginx一样的做法。

    现在的mutex实现大都在用户态尝试几遍spinning，失败后才会去sleep，如果不是很确定spinning的时间足够短的话，就老老实实的用mutex吧。这里有关于怎样提高spinning的资料，详细参考：https://software.intel.com/en-us/articles/benefitting-power-and-performance-sleep-loops