---
title: "RCTF的RNote系列"
---

上周末是RCTF2018，忙里偷闲看看了题。别的也不会，只能去做pwn。看了十几分钟，果然不出我所料，还是一如既往的难。最后五道PWN只做出来一道，还花了一整天的时间，早上10点调试到到晚上9点。没错，做出来的那道就是百人斩的babyheap。。

第二天看了一下具有百人斩潜质的RNote3，结果找不到漏洞点，然后加上没时间，就没再看。搜了一下发现，RCTF2017还有两道RNote的题目，就把这四道RNote题都复现了一下。在这里记录一下学到的知识和经验。


## RNote1

### 发现漏洞点

add函数内，读入title时存在off-by-one漏洞。但是这个off-by-one不是修改下一个chunk的size域，而是直接修改程序存储的chunk的地址。这样可以构造use after free，既能信息泄露，还能利用fastbin attack完成get shell。

```c
for ( i = 0; (signed int)i <= size; ++i )   // 应该是 <，不是<=
{
    if ( read(0, &buf, 1uLL) < 0 )
        exit(1);
    *(_BYTE *)(addr + (signed int)i) = buf;
    if ( *(_BYTE *)((signed int)i + addr) == 10 )
    {
        *(_BYTE *)((signed int)i + addr) = 0;
        return i;
    }
}
```

### 漏洞利用

fastbin attack改写__malloc_hook的值为one_gadget。


## babyheap

### 发现漏洞点

add函数内，读入note的content的函数中存在off-by-one漏洞，不过这次是NULL byte，而且NULL byte覆盖的是下一chunk的size域。可以通过构造，overlap一个chunk。


```c
for ( i = 0; i < size; ++i )
{
    buf = 0;
    if ( read(0, &buf, 1uLL) < 0 )
        sub_B90((__int64)"read() error");
    *(_BYTE *)(addr + i) = buf;
    if ( buf == 10 )
        break;
}
*(_BYTE *)(i + addr) = 0;
```
跟RNoe1很像，不过调试起来很麻烦。

### 漏洞利用

- 首先想到unlink漏洞，不过chunk的指针都在mmap出来的内存里面，没办法泄露。
- shrink the chunk

![off-by-one-null-byte](http://of38fq57s.bkt.clouddn.com/off-by-one-null-Byte.PNG)

- [simple heap trick](https://lyoungjoo.github.io/2018/05/21/RCTF-2018-Write-Up/)

这种方法很好，很妙，很大佬，学习学习。修改下一chunk的prev_size值和inuse位，欺骗unlink，造成overlap chunk。既能泄露信息，又能get shell。


我使用的是shrink the chunk，步骤太多，调了一天，结果写出来的exp极其复杂和丑陋。还是自己脑子不够使。。


## RNote2

### 发现漏洞点

这个题自己构造了一个双向链表，来把所有note串在一起。这个特点让我想起了前段时间的HITBCTF的once那一题。

这个题的漏洞是扩展堆块使用的strncat判断字符串长度出现错误，从而造成堆溢出。

举个例子解释：

- 一块空闲内存存放的内容是AAAAAAAAAAAA（长度为12）
- 用户输入BBBB，且没有null作为结尾
- 想要使用strncat，在用户的输入后面拼接上“CCCC”
- 如果拼接后的字符串将要存放在前面提到的空闲内存，那么strncat的结果是“BBBBAAAAAAAACCCC”（长度为16），造成溢出

strncat在识别用户输入出现错误，BBBB代替了未初始化化内存的前四个字符，将“BBBBAAAAAAAA”看做用户输入。

### 漏洞利用

信息泄露很容易使用unsorted bin实现

然后使用溢出来overlap chunk，fastbin attack， malloc_hook，one_gadget一把梭。


## RNote3

### 发现漏洞点

这个题套路更深。漏洞点是栈中变量未初始化，可以未预期free，造成use after free。

仔细观察，view函数与delete栈帧布局相同,而且view函数在使用指针变量前初始化，delete函数中指针变量使用前未初始化。

还有一个帮助发现漏洞的异常点是，遍历note_ptr_list未找到note的情况未作处理：
```c
unsigned __int64 delete()
{
  signed int i; // [rsp+4h] [rbp-1Ch]
  note *ptr; // [rsp+8h] [rbp-18h]
  char s1; // [rsp+10h] [rbp-10h]
  unsigned __int64 v4; // [rsp+18h] [rbp-8h]

  v4 = __readfsqword(0x28u);
                                        //对ptr缺少初始化
  printf("please input note title: ");
  read_with_null((__int64)&s1, 8u);
  for ( i = 0; i <= 31; ++i )
  {
    if ( note__ptr_list[i] && !strncmp(&s1, (const char *)note__ptr_list[i], 8uLL) )
    {
      ptr = (note *)note__ptr_list[i];
      break;                  
    }
  }
                //代码异常点，未找到相关的note没有直接return，而是接着运行。
  if ( ptr )
  {
    free((void *)ptr->content_ptr);
    free(ptr);
    note__ptr_list[i] = 0LL;   // 此时i是31
  }
  else
  {
    puts("not a valid title");
  }
  return __readfsqword(0x28u) ^ v4;
}
```


### 漏洞利用

这样的话，先view一个note，然后delete一个不存在的note，那么libc就会把这个note和content空间给释放，但是程序依然记录作使用中。use after free，泄露信息，加上get shell。



## RNote4


### 发现漏洞点

漏洞点非常明显，very pure的堆溢出存在于edit函数。

但是程序没有任何输出，无法泄露信息。

### 漏洞利用

能通过这个堆溢出写任何能写的地址，更不用说堆中的各种攻击。但是要怎么控制EIP呢？在没有任何libc和heap的信息下。

还是大佬们会玩： [overwrite strtab pointer](https://lyoungjoo.github.io/2018/05/21/RCTF-2018-Write-Up/)，解法和exp都来自于这个链接。

### 再复习一下_dl_runtime_resolve函数:

当初次调用库函数，会使用_dl_runtime_resolve函数来找库函数的真实地址，并将其填充至`.got.plt`表相应的位置。

这个函数的细节简单分为：

1. 从`.rel.plt`找到相应的Elf64_Rela条目

```c
typedef struct
{
  Elf64_Addr    r_offset;       /* .got.plt中对应的地址 */
  Elf64_Xword   r_info;         /* Relocation type and symbol index */
  Elf64_Sxword  r_addend;       /* Addend */
} Elf64_Rela;
```
对应ida中
```
LOAD:00000000004004D0                 Elf64_Rela <601FF8h, 800000006h, 0> ; R_X86_64_GLOB_DAT __gmon_start__
LOAD:00000000004004E8                 Elf64_Rela <602080h, 0C00000005h, 0> ; R_X86_64_COPY stdin
LOAD:0000000000400500 ; ELF JMPREL Relocation Table
LOAD:0000000000400500                 Elf64_Rela <602018h, 100000007h, 0> ; R_X86_64_JUMP_SLOT free
LOAD:0000000000400518                 Elf64_Rela <602020h, 200000007h, 0> ; R_X86_64_JUMP_SLOT __stack_chk_fail
LOAD:0000000000400530                 Elf64_Rela <602028h, 300000007h, 0> ; R_X86_64_JUMP_SLOT memset
LOAD:0000000000400548                 Elf64_Rela <602030h, 400000007h, 0> ; R_X86_64_JUMP_SLOT alarm
LOAD:0000000000400560                 Elf64_Rela <602038h, 500000007h, 0> ; R_X86_64_JUMP_SLOT read
LOAD:0000000000400578                 Elf64_Rela <602040h, 600000007h, 0> ; R_X86_64_JUMP_SLOT __libc_start_main
```

2. 以free函数为例，symbol_index = 0x100000007 >> 32 = 1，symbol_index是free函数这个Elf32_Sym结构体在.dynsym段的索引。
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
对应的数据是
```
5F 00 00 00 12 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
```

3. 然后得到st_name = 0x5f, 也就是在.dynstr段的偏移

```
  0x004003f8 006c6962 632e736f 2e360065 78697400 .libc.so.6.exit.
  0x00400408 5f5f7374 61636b5f 63686b5f 6661696c __stack_chk_fail
  0x00400418 00737464 696e0063 616c6c6f 63006d65 .stdin.calloc.me
  0x00400428 6d736574 00726561 6400616c 61726d00 mset.read.alarm.
  0x00400438 61746f69 00736574 76627566 005f5f6c atoi.setvbuf.__l
  0x00400448 6962635f 73746172 745f6d61 696e0066 ibc_start_main.f
  0x00400458 72656500 5f5f676d 6f6e5f73 74617274 ree.__gmon_start
  0x00400468 5f5f0047 4c494243 5f322e34 00474c49 __.GLIBC_2.4.GLI
  0x00400478 42435f32 2e322e35 00                BC_2.2.5.
```
偏移0x5f就是字符串“free”

4. 然后就根据函数名称得到函数真实地址，填充到.got.plt，也就是Elf64_Rela.r_offset

这个题我们只需攻击第三步，把dynstr段中的“free”替换为“system”，这样就劫持了free函数触发shell。



## experience


就read函数本身而言，其与漏洞密切相关。要不就是漏洞点所在，要不可以泄露信息，或者协助其他漏洞点。

- RNote1的读入用户输入函数，造成off-by-one
- babyheap的读入用户输入函数，造成off-by-one NULL byte
- RNote2的读入用户输入函数，没有用null结尾，协助strncat函数造成溢出
- 读入的字符串没有以null结尾，十有八九能泄露信息或者存在漏洞



[Github链接](https://github.com/solei1/CTF/tree/master/RCTF-RNote)