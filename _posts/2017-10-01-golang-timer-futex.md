---
layout: post
title: 浅析Go语言和beanstalk的timer设计
description: 简要分析timer的实现方式
categories: os tech
author: lambdae
---

* **Golang的timer**

    golang的timer是用**最小堆**(数据结构)+**futex**(fast userspace mutex)实现的。对比前文提到的方式，golang的timer只有在timer协程需要睡眠和唤醒的时候才会触发系统调用，插入和删除操作都在用户态完成的。相比之下，用户态和内核态的切换就少了许多。从数据结构上来看，**epoll**的fd是由**黑红树**管理的，插入、删除、查找时间复杂度都是log(n)。而做为定时器，我们更关心的是离当前时间最短的expire时间点，并不需要所有expire时间点都有序。再者，实现红黑树比实现最小堆复杂多了，每次为了理解红黑树的旋转和重着色等操作，都要看一遍算法导论，没多久又模糊了，哎这个复杂的机制。

    最小堆的插入和删除伴随着堆化的操作，时间复杂度也是log(n)。获取堆顶节点时间复杂度为O(1)，在定时器gorutine中会频繁的获取堆顶节点的expire时间计算gorutine睡眠时间，最小堆是很适合这样的场景的。

    以下依次分析timer的调用栈：

     ```go
    func (tb *timersBucket) addtimerLocked(t *timer) {
	   // when must never be negative; otherwise timerproc will overflow
	    // during its delta calculation and never expire other runtime timers.
	    if t.when < 0 {
		     t.when = 1<<63 - 1
	    }
	    t.i = len(tb.t)
	    tb.t = append(tb.t, t)
	    // 执行堆化操作
	    siftupTimer(tb.t, t.i)
	    // 当前事件最先触发deadline
	    if t.i == 0 {
		    // siftup moved to top: new earliest deadline.
		    if tb.sleeping {
			     tb.sleeping = false
			     // 唤醒定时器futex睡眠
			     notewakeup(&tb.waitnote)
		    }
		    if tb.rescheduling {
			     tb.rescheduling = false
			     goready(tb.gp, 0)
		    }
	    }
	    // 启动定时器loop线程
	    if !tb.created {
		     tb.created = true
		     go timerproc(tb)
	    }
  }
     ```

    ```go
    // timer gorutine 周期性的管理二叉堆
    func timerproc() {
    	timers.gp = getg()
    	for {
    		lock(&timers.lock)
    		timers.sleeping = false
    		now := nanotime()
    		delta := int64(-1)
    		for {
    			if len(timers.t) == 0 {
    				delta = -1
    				break
    			}
    			t := timers.t[0]
    			delta = t.when - now
              // 需要休眠等待到expire时间点
    			if delta > 0 {
    				break
    			}
    			if t.period > 0 {
    				// leave in heap but adjust next time to fire
    				t.when += t.period * (1 + -delta/t.period)
    				siftdownTimer(0)
    			} else {
    				// remove from heap
    				last := len(timers.t) - 1
    				if last > 0 {
    					timers.t[0] = timers.t[last]
    					timers.t[0].i = 0
    				}
    				timers.t[last] = nil
    				timers.t = timers.t[:last]
    				if last > 0 {
    					siftdownTimer(0)
    				}
    				t.i = -1 // mark as removed
    			}
    			f := t.f
    			arg := t.arg
    			seq := t.seq
    			unlock(&timers.lock)
    			if raceenabled {
    				raceacquire(unsafe.Pointer(t))
    			}
    			f(arg, seq)
    			lock(&timers.lock)
    		}
    		if delta < 0 || faketime > 0 {
    			// No timers left - put goroutine to sleep.
    			timers.rescheduling = true
    			goparkunlock(&timers.lock, "timer goroutine (idle)", traceEvGoBlock, 1)
    			continue
    		}
    		// At least one timer pending. Sleep until then.
    		timers.sleeping = true
    		noteclear(&timers.waitnote)
    		unlock(&timers.lock)
          // 调用futex休眠等待到delta超时时间
    		notetsleepg(&timers.waitnote, delta)
    	}
	}
	```

    notetsleep最终会调用os_linux.go中的futexsleep

    ```go

    func futexsleep(addr *uint32, val uint32, ns int64) {
    	var ts timespec

    	// Some Linux kernels have a bug where futex of
    	// FUTEX_WAIT returns an internal error code
    	// as an errno. Libpthread ignores the return value
    	// here, and so can we: as it says a few lines up,
    	// spurious wakeups are allowed.
    	if ns < 0 {
    		futex(unsafe.Pointer(addr), _FUTEX_WAIT, val, nil, nil, 0)
    		return
    	}

    	// It's difficult to live within the no-split stack limits here.
    	// On ARM and 386, a 64-bit divide invokes a general software routine
    	// that needs more stack than we can afford. So we use timediv instead.
    	// But on real 64-bit systems, where words are larger but the stack limit
    	// is not, even timediv is too heavy, and we really need to use just an
    	// ordinary machine instruction.
    	if sys.PtrSize == 8 {
    		ts.set_sec(ns / 1000000000)
    		ts.set_nsec(int32(ns % 1000000000))
    	} else {
    		ts.tv_nsec = 0
    		ts.set_sec(int64(timediv(ns, 1000000000, (*int32)(unsafe.Pointer(&ts.tv_nsec)))))
    	}
    	futex(unsafe.Pointer(addr), _FUTEX_WAIT, val, unsafe.Pointer(&ts), nil, 0)
    }
    ```

    到这里我们看到，调用futex可以让gorutine做纳秒级的睡眠，时间精度非常高！

* **futex是什么？**

    还是维基百科定义清晰，拿来主义：Futex 由一块能够被多个进程共享的内存空间（一个对齐后的整型变量）组成；这个整型变量的值能够通过汇编语言调用CPU提供的原子操作指令来增加或减少，并且一个进程可以等待直到那个值变成正数。Futex 的操作几乎全部在应用程序空间完成；只有当操作结果不一致从而需要仲裁时，才需要进入操作系统内核空间执行。这种机制允许使用 futex 的锁定原语有非常高的执行效率：由于绝大多数的操作并不需要在多个进程之间进行仲裁，所以绝大多数操作都可以在应用程序空间执行，而不需要使用（相对高代价的）内核系统调用。

    这里有比较详细的分析futex是怎样实现mutex的资料：https://lwn.net/Articles/360699/ 围绕futex的机制有很多内容可以展开，精力所限这里没必要再深入了。有兴趣可以仔细阅览下，收获很多的。


    以下在centos 7上模拟测试了golang timer睡眠的过程。每睡眠1秒后唤醒打印消息再睡眠...

    ```c
    #include <stdio.h>
    #include <stdint.h>

    #include <sys/syscall.h>
    #include <linux/futex.h>
    #include <time.h>


    static int futex(int *uaddr, int futex_op, int val,
                 const struct timespec *timeout, int *uaddr2, int val3) 
    {
               return syscall(SYS_futex, uaddr, futex_op, val,
                              timeout, uaddr, val3);
    }


    void futexsleep(int *addr, uint32_t val, uint64_t ns) {
    	struct timespec ts;
    	ts.tv_sec = ns / 1000000000;
    	ts.tv_nsec = ns % 1000000000;

    	futex(addr, FUTEX_WAIT, val,  &ts, NULL, 0);
    }

    int main(int argc, char const *argv[])
    {

    	int cnt = 20;
    	int addr = 0;
    	int val = 0;
    	uint64_t ns = 1000000000 * 1 + 1000000 * 123;
    	while (cnt--) {
    		printf("sleeping!\n");
    		futexsleep(&addr, val, ns);
    	}
    	
    	return 0;
    }
    ```

    最后依次休眠后输出了20个sleeping，至此golang的timer实现细节就过了一遍，GO使用了一个专用的内核线程(即GPM调度模型的M）去做futex睡眠，让内核去调度，当超时事件触发后把gorutine加入到runnable队列里，让Go调度器去执行timeout的callback。
    
    

* **Golang定时器伪码流程**


     ```go
    // 定时器loop线程
    func  timerproc() {
        loop {
            delta = getMinTimerHeapDelta()
            // 线程睡眠时间片 10e6 纳秒 = 1 微秒
            ns = 10e6 
            for n in [1..delta/ns] {
                // 当其他gorutine添加deadline更近的事件时调用notewakeup() 
                // 唤醒futex的线程 处理deadline更近的事件
                if wake_by_other || delta <= 0 {
                    break
                }
                if delta < ns {
                    ns = delta
                }
                // 调用futex 睡眠 1 微秒
                notesleepng(ns)
                delta -= ns
            }
        }
    }
   
    ```

