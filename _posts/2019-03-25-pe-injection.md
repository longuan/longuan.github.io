---
title: "pe注入的三种方法"
---


一直觉得windows相关的东西比较复杂，Windows内核、Windows漏洞利用、软件逆向等等，都比Linux平台多了好多难以理解的东西。再加上如今Windows安全形势的式微，一直没敢碰它。

Windows api是真的好用。


## 进程注入

往进程注入的东西大致可以分为三类：

1. 直接注入shellcode
2. 注入dll
3. 注入exe

按难度来说的话，dll注入最容易成功，exe注入和shellcode注入不相上下。其实原理很简单，工作总量是差不多的，你借助操作系统越多，你自己的工作量就越少，难度就越小。从后面的解释可以很容易看出工作量的差异。

所以没事的话优先选择注入dll，除非留作业要求注入exe。

## dll注入

dll是pe文件的一种，也是进程注入的主力，而且研究的比较多，实现方法也比较多。

我这里就简单了解了LoadLibrary的方法。

大致就是四个步骤，用windowsAPI表示：

1. OpenProcess （打开目标进程）
2. VirtualAllocEx （在目标进程中分配内存）
3. WriteProcessMemory （在目标进程中写内存）
4. CreateRemoteThread （调用目标进程中的代码）

这四个步骤是一种范式，不仅适用于dll注入，应该适用于所有的进程注入。


详细点说：

1. 打开目标进程
2. 在目标进程中分配长度不小于dll名称的内存
3. 将dll名称写入分配的内存
4. 使用CreateRemoteThread函数，创建一个新线程，这个线程将以dll名称为参数调用LoadLibrary函数
    - dllmain函数将得到执行，同时收到DLL_PROCESS_ATTACH信号

可以注意到，我们做的东西非常少，更多是交给了LoadLibrary函数。


## exe自注入

对于dll，系统提供的有LoadLibrary函数，但是要将exe注入到目标进程，就不是简单调用一个api就能完成的了。

exe自注入是一种折中的实现，先让exe执行起来，然后再把自己在自己进程中的映像拷贝到目标进程中，然后靠CreateRemoteThread去启动它的另一个函数。

与dll注入相比，拷贝的东西不过是从dll名称字符串变为pe在内存中的映像。但是在拷贝之前需要修正reloc段，因为你在自己进程中的基地址是A，到目标进程中基地址就不一定是A了。所以需要解析reloc段，将里面的值都加上这个差值。

这种方法，我们借助操作系统来装载exe文件，然后对其映像做些改动就可以移植了。

## 注入exe

这个的目标是注入一个不相关的、没有运行起来的exe文件，并在目标进程中执行。

根据网上的资料，至少需要做如下事情：

1. 把exe文件展开到内存中
2. 根据在目标进程中申请到的基址来修正reloc段
3. 填写IAT

也就是在exe自注入的基础上，多做了1和3两步。其实要完成的东西就是一个简易版的exe加载器。

我完成的那个例子只能把指定的exe注入到非特权的32位进程，离通用性还很远。

其实可以留个坑--通用的exe注入器。

## 参考

- [以上三个例子](https://github.com/solei1/PEInjection)
- https://blog.csdn.net/xfgryujk/article/details/81360292
- https://github.com/virtable/mmLoader

