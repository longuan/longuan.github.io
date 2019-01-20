---
title: "阅读afl源码"
---


从源码的角度分析一下AFL，学习一下具体的实现细节和fuzz逻辑

主要是afl-fuzz.c文件


## 变量

### out_file

类型是字符串指针，注释是`file to fuzz, if any`

1. 首先用getopt看，是否提供了“-f”选项，有的话赋给out_file。

2. 调用detect_file_args函数。如果有@@标志（就是说目标程序接受文件输入），并且未使用-f选项，就给out_file默认值——“<outdir>/.cur_input”；

3. 如果此时out_file还是一个NULL（就是说目标程序接受标准输入），就调用setup_stdio_file函数。setup_stdio_file函数使用默认的“<outdir>/.cur_input”作为fuzzed data的输入文件，用out_fd这个文件描述符指向它。也就说，通过out_file是否为空来判断目标程序的输入类型。

4. 之后在init_forkserver函数中，这样之后fork出来的子进程都会这样接收输入：
```
if (out_file) {
  dup2(dev_null_fd, 0); // 不使用标准输入，使用out_file作为文件输入
} else {
  dup2(out_fd, 0);      // 使用“<outdir>/.cur_input”作为标准输入
  close(out_fd);
}
```

5. 之后在write_to_testcase函数中，如果out_file存在，就把数据写入到out_file中，否则写入到out_fd中。


### virgin_bits

开始被setup_shm函数初始化，所有bit都为1。fuzz始终也就这一次初始化，不像trace_bits，每次run_target都要初始化。

后来基本就跟has_new_bits函数绑定了。

virgin_bits的每个byte代表一个边，不过不是边的命中计数，而是边的命中计数状态。详细见has_new_bits函数解释。


### top_rated

top_rated数组的每一项都代表一个边，内容是queue entry。意思就是能访问到这个边的最优（执行时间乘以输入size最小）case。

top_rated靠update_bitmap_score函数维护，在cull_queue函数中被用来挑选favored种子。


## 函数

### perform_dry_run

函数原型：`void perform_dry_run(char** argv)`

函数注释：`Perform dry run of all test cases to confirm that the app is working as expected. This is done only for the initial inputs, and only once.`

函数分析：

1. 循环队列（这个队列是read_testcases产生的）
    1. 把这个queue_entry的输入文件(fname, len)读入内存
    2. 调用calibrate_case
    3. 得到calibrate_case的返回值，也就是run_target的返回值，也就是fork server的waitpid返回值。对返回值做分析


### calibrate_case

函数原型：`u8 calibrate_case(char** argv, struct queue_entry* q, u8* use_mem, u32 handicap, u8 from_queue)`

函数注释: 
```
   Calibrate a new test case. This is done when processing the input directory
   to warn about flaky or otherwise problematic test cases early on; and when
   new paths are discovered to detect variable behavior and so on.
```

函数分析：

1. 判断fork server有没有启动，没有的话执行init_forkserver
2. 如果之前queue_entry运行过，就把运行结果拷贝给first_trace。
3. 然后就是一个for循环——write_to_testcase,run_target,hash32,has_new_bits（var_bytes干啥用的？）
4. 然后调用update_bitmap_score看当前queue entry是否是favored
5. 最后有mark_as_variable（干啥用的？）

函数名字是校准case，内容差不多是调整var_byte和top_rated

这个函数基本只被perform_dry_run和save_if_interesting函数调用，前者是初始化，后者是添加新case到队列中。


### init_forkserver

函数原型：`void init_forkserver(char** argv)`

函数注释：
```
Spin up fork server (instrumented mode only). The idea is explained here:

   http://lcamtuf.blogspot.com/2014/10/fuzzing-binaries-without-execve.html

   In essence, the instrumentation allows us to skip execve(), and just keep
   cloning a stopped child. So, we just execute once, and then send commands
   through a pipe. The other part of this logic is in afl-as.h.
```

函数分析：

这个函数只是用来启动fork server，而fork server的维护就交给了main_payload。

使用fork+setsid+execv派生出来一个独立的进程，运行目标程序作为fork server。

https://lcamtuf.blogspot.com/2014/10/fuzzing-binaries-without-execve.html


### run_target

函数原型：`u8 run_target(char** argv, u32 timeout)`

函数注释：`Execute target application, monitoring for timeouts. Return status information. The called program will update trace_bits[].`

函数分析：

1. 把trace_bits全部置零，由于是共享内存，所以是把计数清空，重新执行目标程序
2. 命令parent fuzzer进程fork出来子进程，也就是child fuzzer进程
    1. fork出来的child fuzzer进程关闭两个FORKSRV_FD文件描述符，之后进入afl_store（在trace_bits计数后恢复执行，猜测应该是恢复到main函数）
    2. 执行fork的parent fuzzer进程，使用write将子进程pid传送给afl-fuzz，然后waitpid(子进程pid)，然后把子进程退出的status发送给afl-fuzz，然后继续等待afl-fuzz的fork命令
3. afl-fuzz接收到子进程的pid，并且读取parent fuzzer的waitpid结果（也就是子进程退出的status）
4. 以count_class_lookup[256]把trace_bits(也就是分支的命中情况)分到各个桶中，但是这里分了32和64两种情况并不影响
5. 然后使用WIFSIGNALED和WTERMSIG确定退出的状态，需要注意的是子进程超时并且退出状态为SIGKILL或者使用asan并且返回状态是MSAN_ERROR才返回FAULT_CRASH


### has_new_bits

函数原型：`inline u8 has_new_bits(u8* virgin_map)`

函数注释：
```
   Check if the current execution path brings anything new to the table.
   Update virgin bits to reflect the finds. Returns 1 if the only change is
   the hit-count for a particular tuple; 2 if there are new tuples seen. 
   Updates the map, so subsequent calls will always return 0.

   This function is called after every exec() on a fairly large buffer, so
   it needs to be fast. We do this in 32-bit and 64-bit flavors.
```

ret为0，说明此次执行没什么结果
ret为1，说明此次执行某个边的命中数出现了跃迁
ret为2，说明此次执行发现了新的边

函数分析：

之前在run_target中，已经把trace_bits合规化，就是说共享内存每个byte的值只有九种可能。

使用`*current & *virgin`作为判断条件，是因为current[0]只有九种可能0b10000000、0b01000000...，做与运算就相当于取virgin的某个位。而virgin的某个bit为1就说明未出现过，为0就说明出现过。所以`*current & *virgin`不为0就说明出现了新的情况，然后判断是命中数的跃迁还是发现了新的边。


### update_bitmap_score

函数原型：`void update_bitmap_score(struct queue_entry* q)`

函数注释：
```
   When we bump into a new path, we call this to see if the path appears
   more "favorable" than any of the existing ones. The purpose of the
   "favorables" is to have a minimal set of paths that trigger all the bits
   seen in the bitmap so far, and focus on fuzzing them at the expense of
   the rest.

   The first step of the process is to maintain a list of top_rated[] entries
   for every byte in the bitmap. We win that slot if there is no previous
   contender, or if the contender has a more favorable speed x size factor.
```

函数分析：

同步遍历trace_bits和top_rated，对于trace_bits中的每个byte，看当前的queue entry是否是favored。是的话更新top_rated，置score_changed为1，更新trace_mini。

这个函数不会单独调用，而是被calibrate_case或者trim_case调用。

> trace_mini是浓缩版的trace_bits。trace_mini的一个bit代表trace_bits的一个byte，当bit为1代表对应的byte有数值，当bit为0代表对应的byte为0。


### cull_queue

函数原型：`void cull_queue(void)`

函数注释：
```
   The second part of the mechanism discussed above is a routine that
   goes over top_rated[] entries, and then sequentially grabs winners for
   previously-unseen bytes (temp_v) and marks them as favored, at least
   until the next run. The favored entries are given more air time during
   all fuzzing steps. 
```

函数分析：

1. 如果top_rated数组没有更新（体现在update_bitmap_score函数中），那么直接返回
2. 把队列里所有entry的favored都置0
3. 进入一个for循环。temp_v的原理和trace_mini一样。temp_v开始全是1，1代表这条边还没有queue entry走过，一遍遍的循环会慢慢减少1的数量。最终会标记一个较小的favored集合——这个集合能覆盖目前发现的所有边（这个集合不是最优的）。
4. mark_as_redundant


### fuzz_one 

函数原型：`u8 fuzz_one(char** argv)`

函数注释：
```
   Take the current entry from the queue, fuzz it for a while. This
   function is a tad too long... returns 0 if fuzzed successfully, 1 if
   skipped or bailed out. 
```

函数分析：

1. 以99%的概率跳过fuzz过的或者non-favored的case，以75%的概率跳过non-favored的未fuzz的不是第一轮的case，以95%的概率跳过fuzz过的或者第一轮的non-favored case
2. CALIBRATION (only if failed earlier on)
3. 然后调用trim_case在不影响trace_bits的情况下剪裁输入文件，并把剪裁结果写回磁盘
4. 然后调用calculate_score对当前queue entry计算分数。执行时间越短，访问的边越多，depth越大，handicap越大，分数越高。分数会用在havoc阶段
5. bit flip变异
6. arithmetic变异
7. interest变异
8. dictionary变异
9. havoc变异
10. splicing变异


### common_fuzz_stuff

函数原型：`u8 common_fuzz_stuff(char** argv, u8* out_buf, u32 len)`

函数注释：
```
   Write a modified test case, run program, process results. Handle
   error conditions, returning 1 if it's time to bail out. This is
   a helper function for fuzz_one.
```

函数分析：

这个函数是在对输入文件变异之后调用，所做的工作正如函数注释所说。分别对应三个函数write_to_testcase、run_target、save_if_interesting。


### save_if_interesting

函数原型：`u8 save_if_interesting(char** argv, void* mem, u32 len, u8 fault)`

函数注释：
```
This handles FAULT_ERROR for us:

   Check if the result of an execve() during routine fuzzing is interesting,
   save or queue the input test case for further analysis if so. Returns 1 if
   entry is saved, 0 otherwise.
```

函数分析：

如果fault是crash_mode，仅当状态改变（计数跃迁和发现新边）才再次将文件加入队列，然后使用函数calibrate_case对这个queue entry校准。

如果fault是FAULT_TMOUT，如果是FAULT_CRASH（crash_mode不就是FAULT_CRASH吗？？）。


## 参考

1. https://blog.csdn.net/Chen_zju/article/details/80791268
2. https://blog.csdn.net/qq_32464719/article/details/80592902
3. https://bbs.pediy.com/thread-249912.htm
4. 等等