---
layout: post
title: 0ctf的两道题
---



## malloc_hook & one_gadget rce

题目：n1ctf-vote

第一次知道malloc_hook这种方法，瞬间觉得自己菜得不行。

1. unsorted bin 泄露libc地址
2. uaf 修改 chunk的fd指针
3. 利用fastbin atack 申请到malloc_hook附近
4. 写入onegadget，触发

fastbin attack有一个限制就是malloc fastchunk之前会检查这个堆块的size，刚好可以利用错位在malloc_hook前面构造出来一个0x00007f。这个题可以利用malloc_hook完成get shell，而0ctf-babyheap2018就不可以，就是因为这个check通不过。

参考：https://github.com/david942j/one_gadget

## stack pivot

题目：0ctf-babystack 2018

一开始就想到是`return2dlresolve`，但是溢出的空间只有24字节，明显不够用，于是就卡在这了。

今天搜了一下大佬的writeup，发现解决办法是用`stack povit`。简单来说这个方法就是把栈劫持到一个能控制的空间，再接着ROP。

在这个题目里，只能溢出24个字节的内容，那么先构造一个这样的ROP链
```
'A'*40   // 40是溢出的偏移
p32(0x804a600)  // 把ebp劫持到0x804a600
p32(0x8048446)  // 这个位置的代码如下，主要目的就是利用leave 劫持esp完成栈迁移；ret劫持eip完成ROP
                //  lea     eax, [ebp-0x28]
                //  push    eax             ; buf
                //  push    0               ; fd
                //  call    _read
                //  add     esp, 10h
                //  nop
                //  leave
                //  retn
80
padding
```

这段ROP完成工作如下：

- 先把ebp劫持到0x804a600
- 再调用read，将读入的内容存在0x804a5d8(ebp-0x28)
- leave相当于`mov sp bp; pop bp`，此时sp = bp + 4
- ret，执行流到达sp

```
为什么是0x804a600

gdb-peda$ vmmap
Start      End        Perm  Name
0x08048000 0x08049000 r-xp  /home/beatjean/Desktop/0ctf/babystack/babystack
0x08049000 0x0804a000 r--p  /home/beatjean/Desktop/0ctf/babystack/babystack
0x0804a000 0x0804b000 rw-p  /home/beatjean/Desktop/0ctf/babystack/babystack

并且这个程序的逻辑空间最多到0x804A030，也就是说这大段空间是没用到的
```


在被劫持的栈中布置成如下(以下布局是执行完leave;ret)：
```
....      <- 下面完成ret2dlresolve攻击
200
addr_bss
0
pppr       
read@plt     <- 0x804a604 (esp)
0x804a600    <- 0x804a600 (ebp)
...
padding      <- 0x804a5d8
```

参考：

- https://kileak.github.io/ctf/2018/0ctf-qual-babystack/
- http://tacxingxing.com/2017/05/10/stack-pivot/

## 0ctf-babyheap 2018

看这个程序的第一遍看不出来有什么能利用的点，又看了一遍找到`one-byte overflow`。然后经过大佬提醒可以利用unsorted bin泄露libc的地址，

### <del>malloc_hook</del>

这时第一想法就是用`fastbin attack `去写malloc_hook拿shell。调了一下行不通，因为程序限制能申请的最大堆块大小是0x58，当你申请malloc_hook当堆块的时候，会把0x7f给切开，分为0x10和0x6f，把0x6f返回回来，这时size检查就不通过了。

### alloc to main_arena

既然不能直接malloc到malloc_hook，那么只能曲线救国。

- 先用fastbin attack在main_arena上布置好一个数值当作size，0x41
- 再用fastbin attack申请到main_arena当作堆块
- update刚才的堆块，改写main_arena.top = malloc_hook-0x10
- 再申请堆块，也就是从top chunk切割，此时能将malloc_hook当作堆块返回
- 改写malloc_hook为 onegadget
- 触发onegadget

题目给的libc是2.24的，我虚拟机的是2.23，然后我调试的时候死活用不了这个2.24，不管是用pwntools的env设置LD_PRELOAD，还是替换libc.so.6这个软链接，还是直接替换libc-2.23.so。。。那就这样吧


### House of Orange

这个大佬用的这种方法，https://github.com/zounathtan/ctf/tree/master/writeups/2018-0ctf/babyheap。。。学习中