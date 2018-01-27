---
layout: post
title: 完美哈希在Jemalloc的应用
description:  简述malloc(size_t size)过程中，通过静态的完美哈希解决size到slab槽位的映射.
categories: tech os
author: lambdae
---


*  **引言**

    Jemalloc在malloc(size_t sz)时会通过一个静态的完美哈希在O(1)的时间复杂度查找到大小 >= sz的slab槽位(jemalloc定义为bin)，该hash表使用宏在编译时生成。通过该哈希表，jemalloc的小内存分配流程可以简述为：

    ```C
    malloc(size_t sz) -> hash(sz) -> bin的槽位index -> 
    根据index获得bin在chunk的元数据指针 -> 获取指针指向对象的bitmap(freelist) -> 返回void*指针给用户
	```

    注：jemalloc大内存块对象根据地址空间由黑红树管理，不在小内存slab的线性布局内。jemalloc可用的小内存对象使用bitmap记录，和linux slab、TCmalloc的freelist链表不同。freelist构造起来会更简单，jemalloc为什么用bitmap有时间再探究，挖个坑。

    下面对相关代码写了简单的测试，很直观的就能看懂其构造思想。



* **简析**

    源码测试
    ```c
	#include <stdio.h>
	#include <stdint.h>

	// SIZE_CLASSES slab布局宏定义
	// SIZE_CLASS(index, increase, size)  
	//   其中index就是slab的槽位，increase是size累加增长数（如16=8+8, 64=48+16）
	//   size就是该slab最大支持分配的单元大小
	//   SIZE_CLASS会根据不同的需求给出不同的定义
	#define SIZE_CLASSES                        \
    	SIZE_CLASS(0,   8,  8)                  \
    	SIZE_CLASS(1,   8,  16)                 \
    	SIZE_CLASS(2,   16, 32)                 \
    	SIZE_CLASS(3,   16, 48)                 \
    	SIZE_CLASS(4,   16, 64)                 \
    	SIZE_CLASS(5,   16, 80)                 \
    	SIZE_CLASS(6,   16, 96)                 \
    	SIZE_CLASS(7,   16, 112)                    \
    	SIZE_CLASS(8,   16, 128)                    \
   		SIZE_CLASS(9,   32, 160)                    \
    	SIZE_CLASS(10,  32, 192)                    \
    	SIZE_CLASS(11,  32, 224)                    \
    	SIZE_CLASS(12,  32, 256)                    \
    	SIZE_CLASS(13,  64, 320)                    \
    	SIZE_CLASS(14,  64, 384)                    \
    	SIZE_CLASS(15,  64, 448)                    \
    	SIZE_CLASS(16,  64, 512)                    \
    	SIZE_CLASS(17,  128,    640)                    \
    	SIZE_CLASS(18,  128,    768)                    \
    	SIZE_CLASS(19,  128,    896)                    \
    	SIZE_CLASS(20,  128,    1024)                   \
    	SIZE_CLASS(21,  256,    1280)                   \
    	SIZE_CLASS(22,  256,    1536)                   \
    	SIZE_CLASS(23,  256,    1792)                   \
    	SIZE_CLASS(24,  256,    2048)                   \
    	SIZE_CLASS(25,  512,    2560)                   \
    	SIZE_CLASS(26,  512,    3072)                   \
    	SIZE_CLASS(27,  512,    3584)                   

    // perfect hash表定义
	const static uint8_t    size2bin_tbl[] = {

	#define S2B_8(i)    i,
	#define S2B_16(i)   S2B_8(i) S2B_8(i)
	#define S2B_32(i)   S2B_16(i) S2B_16(i)
	#define S2B_64(i)   S2B_32(i) S2B_32(i)
	#define S2B_128(i)  S2B_64(i) S2B_64(i)
	#define S2B_256(i)  S2B_128(i) S2B_128(i)

	#define S2B_512(i)  S2B_256(i) S2B_256(i)
	#define S2B_1024(i) S2B_512(i) S2B_512(i)
	#define S2B_2048(i) S2B_1024(i) S2B_1024(i)
	#define S2B_4096(i) S2B_2048(i) S2B_2048(i)
	#define S2B_8192(i) S2B_4096(i) S2B_4096(i)

    // SIZE_CLASS定义：调用S2B_x(i) 按照x填充slab槽位
	#define SIZE_CLASS(bin, delta, size)    S2B_##delta(bin)
			// 按slab布局生成hash table
        	SIZE_CLASSES

	#undef S2B_8
	#undef S2B_16
	#undef S2B_32
	#undef S2B_64
	#undef S2B_128
	#undef S2B_256
	#undef S2B_512
	#undef S2B_1024
	#undef S2B_2048
	#undef S2B_4096
	#undef S2B_8192
	#undef SIZE_CLASS

	};

	// 最小slab槽位的size移位
	#define SHIFT_TINY_SIZE 3

    // hash函数size映射的slab槽位
	#define SMALL_SIZE2BIN(size)  size2bin_tbl[(size -1) >> SHIFT_TINY_SIZE]

    // 槽位到size的映射
	const static uint32_t    bin2size_tbl[] = {
    
    // SIZE_CLASS定义为：填充一次size
	#define SIZE_CLASS(bin, delta, size)   (uint32_t)size,
			// 同上
        	SIZE_CLASSES

	#undef SIZE_CLASS
	};

	// 输出size2bin_tbl哈希表
	int main(int argc, char const *argv[])
	{
		int n = sizeof(size2bin_tbl) / sizeof(char);
		int last = -1, curr;
		for (int i = 0; i < n; i++) {
			curr = size2bin_tbl[i];
			if (curr != last) {
				printf("\n");
			}
			printf("%d ", curr);
			last = curr;
		}

		return 0;
	}
```

    终端输出size2bin_tbl哈希表内容：

    ```c 
0 
1 
2 2 
3 3 
4 4 
5 5 
6 6
7 7 
8 8 
9 9 9 9 
10 10 10 10 
11 11 11 11
12 12 12 12 
13 13 13 13 13 13 13 13 
14 14 14 14 14 14 14 14 
15 15 15 15 15 15 15 15 
16 16 16 16 16 16 16 16 
17 17 17 17 17 17 17 17 17 17 17 17 17 17 17 17 
18 18 18 18 18 18 18 18 18 18 18 18 18 18 18 18
19 19 19 19 19 19 19 19 19 19 19 19 19 19 19 19 
20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 20 
21 21 21 21 21 21 21 21 21 21 21 21 21 21 21 21 21 21 21 21 21 21 21 21 21 21 21 21 21 21 21 21 
22 22 22 22 22 22 22 22 22 22 22 22 22 22 22 22 22 22 22 22 22 22 22 22 22 22 22 22 22 22 22 22 
23 23 23 23 23 23 23 23 23 23 23 23 23 23 23 23 23 23 23 23 23 23 23 23 23 23 23 23 23 23 23 23
24 24 24 24 24 24 24 24 24 24 24 24 24 24 24 24 24 24 24 24 24 24 24 24 24 24 24 24 24 24 24 24 
25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 25 
26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 26 
27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27 27

    ```
    hash查找函数
    ```c
     // hash函数size映射的slab槽位
	#define SMALL_SIZE2BIN(size)  size2bin_tbl[(size-1) >> SHIFT_TINY_SIZE]
	
	example:
	    // 8 = 2^3 (SHIFT_TINY_SIZE = 3)
	    hash(19): 19-1 = 18, 18/8 = 2
	    slab2: 32 > 19

	    hash(27)： 27-1 = 26, 16/8 = 2
	    slab2: 32 > 27
    ```


*  **总结**

    综合上面的分析，我们知道，该hash的思想就是去掉size的低位(最小slab大小，SHIFT_TINY_SIZE)，取高位的掩码mask=size-1做为key查找槽位。而mask所有可能落在的区域都填充了槽位的索引。
  ​
