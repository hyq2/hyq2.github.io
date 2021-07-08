---
layout: post
title:  "GOT/PLT 初识"
date:   2021-02-23 19:31:29 +0900
categories: jekyll update
---

**静态库（.a文件）**

它被合并在每个进程的代码段中

**动态库（.so文件）**

在装入时或运行时动态地被加载并链接

**第一次加载运行时进行动态链接**

动态链接器（ld-linux.so)

标准c库（libc.so)

**在已经开始运行后进行**



## 共享库内模块外数据的引用情况(模块间调用,跳转)

### **ONE**

在.data节开始处设置一个指针数组**（全局偏移表，GOT）**，指针指向一个全局变量

**GOT与引用数据的指令之间的相对距离固定**

**函数之间的相对位置也是固定的**————**只要能知道其中一个函数的真正地址，我们就可以计算出任意库函数的地址**

### （PWN）但是要知道**libc版本**

- **ONE**

```python
libc = LibcSearcher('__libc_start_main', libc_start_main_addr) #使用工具LibcSearcher 来计算 offset，libcbase
libcbase = libc_start_main_addr - libc.dump('__libc_start_main') #函数的真正地址 - 在libc中的偏移 = libcbase
```



- **TWO**

```python
from pwn import *

lib = ELF(libc_32.so.6 ) #相对于 py 文件的位置;给出了libc库

#libc文件信息
lib_write_addr = lib.symbols["write"]
lib_system_addr = lib.symbols["system"]
lib_bin_sh_addr = next(lib.search(b'/bin/sh')) #b :表明是bit流，需要用next打包一下!(?)

libcbase_addr = write_addr - lib_write_addr
system_addr = base_addr + lib_system_addr
bin_sh_addr = base_addr + lib_bin_sh_addr

#or
print(hex(elf.symbols['system']-elf.symbols['write'])) #计算在libc中函数之间的offset
```



- **THREE**

```
readelf -s libc_32.so.6|grep 函数名
找 write@@GLIBC_2.0 和 system@@GLIBC_2.0
用 system 的第二列减掉 write 的第二列得到相对位置 -0x99a80
前提已经知道了libc版本噢！并且下载下来到目录里面。
```



GOT表里存的时所引用的地址，填这个地址 在汇编器为GOT每一项生成一个重定位项(.rel.data节)

加载时，动态链接器对GOT表项进行重定位，填入所引用的地址



先call (就是下一条指令) --引用符号

然后把引用处的位置 pop 到 %ebx



### TWO

延时绑定（重定位）

GOT是.data节一部分（指针数组）

GOT[0]  ：dynamic节首址 ——该节中包含动态链接器的基本信息，如符号表位置，重定位表位置等；

GOT[1]  ： 动态链接器的标识信息

GOT[2]  :  动态链接器延时绑定代码的入口地址（进行重定位的代码）

GOT[3] :  被调用共享库函数的GOT项



PLT是.text节一部分（结构数组），每项16B（指令）

PLT[0]  ：pushl xxxxxxx(GOT[1]) 

jmp *GOT[2] (延时绑定代码)

**延时绑定代码**根据GOT[1]和ID确定函数地址从而填入GOT[3]，并转到函数执行----第二次调用时call后到PLT[1]的jmp *GOT[3]时，就直接去调用该函数了

即第一次调用需要重定位,即重定位以后GOT[3]表项里面就是函数的真正地址，此后就可直接调用

PLT[1]  :  对应一个共享库函数



PLT[1]

1.jmp *GOT[3] (GOT表里存的地址)

跳到了PLT[1]的第二条指令

2.push $0x0(ID)  ID = 0标识函数

3.jmp XXXXXXXX(PLT[0])



**即函数调用** 

**call xxxxxxx  (PLT[1])  ---->  去取GOT表里面的函数地址  ----> 执行函数**





![img](https://img-blog.csdn.net/20170123155535419?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXFfMTg2NjEyNTc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)