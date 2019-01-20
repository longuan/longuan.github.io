---
title: "libev源码阅读与剖析"
---



- [简介](#简介)
  - [与Reactor模式的关系](#与Reactor模式的关系)
  - [watcher的“继承”与“封装”](#watcher的继承与封装)
  - [ev_p, ev_a这四个宏](#evpevp和evaeva这四个宏)
- [watcher简介](#watcher简介)
  - [支持的事件](#支持的事件)
  - [支持的event](#支持的event)
  - [watcher的状态、对watcher的通用操作](#watcher的状态对watcher的通用操作)
- [ev_loop](evloop)
  - [创建](#创建)
  - [运行](#运行)
    - [ev_run函数](#evrun函数)
  - [backend](#backend)
  - [epollpoll](#epollpoll)
- [io watcher分析](#io-watcher分析)
  - [anfds](#anfds)
  - [ev_io_start 函数](#eviostart-函数)
- [其他](#其他)
  - [array_needsize 宏](#arrayneedsize-宏)
  - [pendings 数组](#pendings-数组)
- [参考](#参考)



## 简介



​        libev 是一个基于Reactor模式的高性能事件循环网络库。它和libevent很像，按照作者的介绍，可以作为libevent的替代者，能够提供更高的性能，而且代码比较精简。主页是http://software.schmorp.de/pkg/libev.html。

​        在阅读代码之前，先熟悉一些libev中用到的“编程技巧”。



### 与Reactor模式的关系



​        通常来说，Reactor由5个角色组成：

- Handle：即操作系统中的句柄，是操作系统对资源的一种抽象，可以是打开的文件、一个连接(Socket)、Timer等。
- Synchronous Event Demultiplexer：同步事件多路复用器，实际上通常是系统调用。比如linux中的select、poll、epoll等。它会同时监控多个句柄，等待事件的到来。
- Initiation Dispatcher：事件分发器，它提供了注册、删除与调用event handler的方法。当Synchronous Event Demultiplexer检测到handle上有事件发生时，便会通知initiation dispatcher进行事件转发。
- Event Handler：事件处理抽象类，提供一个通用接口handle_event。
- Concrete Event Handler：具体的事件处理器，继承自Event Handler，会实现具体的事件处理逻辑。



​        在libev中，`struct ev_loop`是框架的事件循环核心，负责事件循环、监控事件、管理事件、分发事件等等，相当于Reactor中的Synchronous Event Demultiplexer和Initiation Dispatcher。libev用watcher来管理处理各种事件，每类事件都有对应的watcher实现，比如IO事件的watcher是`struct ev_io`，信号事件的watcher是`struct ev_signal`。watcher相当于Event Handler和Concrete Event Handler。



### watcher的“继承”与“封装”



​        libev不仅可以监控IO事件，也可以监控信号、定时器等等事件。为了化繁为简，libev将所有事件的公用部分抽象成一个结构体`struct ev_watcher` 。

```c
/* shared by all watchers */
#define EV_WATCHER(type)			\
  int active; /* private */			\
  int pending; /* private */			\
  EV_DECL_PRIORITY /* private */		\
  EV_COMMON /* rw */				\
  EV_CB_DECLARE (type) /* private */

/* base class, nothing to see here unless you subclass */
typedef struct ev_watcher
{
  EV_WATCHER (ev_watcher)
} ev_watcher;
```

​        `struct ev_watcher`衍生出其他所有的watcher，就像其他wacher的父类。虽然c语言中没有面对对象的机制，但是libev简单实现了一个结构体的“继承”机制。具体以`struct ev_io`为例，

```c
#define EV_WATCHER_LIST(type)			\
  EV_WATCHER (type)				\
  struct ev_watcher_list *next; /* private */

/* base class, nothing to see here unless you subclass */
typedef struct ev_watcher_list
{
  EV_WATCHER_LIST (ev_watcher_list)
} ev_watcher_list;

/* invoked when fd is either EV_READable or EV_WRITEable */
/* revent EV_READ, EV_WRITE */
typedef struct ev_io
{
  EV_WATCHER_LIST (ev_io)

  int fd;     /* ro */
  int events; /* ro */
} ev_io;
```

​        展开之后，`struct ev_watcher` 在空间布局上位于开头部分，因此可以使用`struct ev_watcher` 指针指向`ev_io`的实例，达到c++中的继承效果。fd和events是`ev_io`独有的变量，分别表示关注的文件描述符和事件。

```c
typedef struct ev_io
{
  int active;
  int pending;
  int priority;   // 可以没有
  void *data;     // 给watcher附加自定义数据，可以没有
  void (*cb)(EV_P_ struct type *w, int revents);
  struct ev_watcher_list *next;
  int fd;   
  int events; 
} ev_io;
```

​        libev提供了很多的set方法，这也是注释中`private`的意思，告诉大家这里是私有变量，起到“封装”的作用，当然这只是“君子约定”。

```c
#define ev_cb_(ev)                           (ev)->cb /* rw */
#define ev_cb(ev)                            (memmove (&ev_cb_ (ev), &((ev_watcher *)(ev))->cb, sizeof (ev_cb_ (ev))), (ev)->cb)

#if EV_MINPRI == EV_MAXPRI
# define ev_priority(ev)                     ((ev), EV_MINPRI)
# define ev_set_priority(ev,pri)             ((ev), (pri))
#else
# define ev_priority(ev)                     (+(((ev_watcher *)(void *)(ev))->priority))
# define ev_set_priority(ev,pri)             (   (ev_watcher *)(void *)(ev))->priority = (pri)
#endif
#ifndef ev_set_cb
/* memmove is used here to avoid strict aliasing violations, and hopefully is optimized out by any reasonable compiler */
# define ev_set_cb(ev,cb_)                   (ev_cb_ (ev) = (cb_), memmove (&((ev_watcher *)(ev))->cb, &ev_cb_ (ev), sizeof (ev_cb_ (ev))))
#endif
```

​        对于具体的watcher，比如`ev_io` 、`ev_signal`会有对应的`ev_io_set`、`ev_signal_set`方法来设置其中的特有变量。





### `EV_P`、`EV_P_`和`EV_A`，`EV_A_`这四个宏

​        libev支持同时存在多个ev_loop。因此在声明函数时需要指明在哪个loop中操作，`struct ev_loop *loop`常常作为第一个参数，在调用函数时也需要传实参。像下面这样，

```c
void ev_io_start (struct ev_loop *loop, ev_io *w);   // 声明
ev_io_start (loop, &w->io);          // 调用
```

​        但是如果只存在一个ev_loop，可以把这个ev_loop当作一个随处可访问的“全局变量”。在声明和调用函数的时候就不用加loop参数了。libev为了统一这两种情况，就使用了`EV_P`、`EV_P_`和`EV_A`，`EV_A_`这四个宏。

| 宏    | 含义                         |
| ----- | ---------------------------- |
| EV_P  | ev_loop parameter            |
| EV_P_ | ev_loop parameter 多一个逗号 |
| EV_A  | ev_loop argument             |
| EV_A_ | ev_loop argument 多一个逗号  |

```c
/* support multiple event loops? */
#if EV_MULTIPLICITY
struct ev_loop;
# define EV_P  struct ev_loop *loop   
# define EV_P_ EV_P,                        
# define EV_A  loop                        
# define EV_A_ EV_A,                        
#else
# define EV_P void
# define EV_P_
# define EV_A
# define EV_A_
#endif

void ev_io_start (EV_P_ ev_io *w);   // 声明
ev_io_start (EV_A_ &w->io);          // 调用
```

​        libev也分别对这两种情况下的`struct ev_loop`做了各自的定义（其实没必要吧）

```c
#if EV_MULTIPLICITY

  struct ev_loop
  {
    ev_tstamp ev_rt_now;
    #define ev_rt_now ((loop)->ev_rt_now)
    #define VAR(name,decl) decl;
      #include "ev_vars.h"                // 这个头文件里定义了ev_loop所有的成员变量
    #undef VAR                     
  };
  #include "ev_wrap.h"                  // 这个头文件里封装了对成员变量的访问

  static struct ev_loop default_loop_struct;
  EV_API_DECL struct ev_loop *ev_default_loop_ptr = 0; /* needs to be initialised to make it a definition despite extern */

#else

  EV_API_DECL ev_tstamp ev_rt_now = EV_TS_CONST (0.); /* needs to be initialised to make it a definition despite extern */
  #define VAR(name,decl) static decl;
    #include "ev_vars.h"                //将ev_loop的所有成员变量都当作全局变量
  #undef VAR

  static int ev_default_loop_ptr;

#endif
```

​        如果支持多个 event loop，那么 `ev_default_loop_ptr` 就是一个静态的 `struct ev_loop` 类型的结构体，其中包含了各种成员，比如 `ev_tstamp ev_rt_now;` `int pendingpri;` 等等。如果不支持多个 event loop，则上述的 `struct ev_loop` 结构就不存在，其成员都是以静态变量的形式进行定义，而 `ev_default_loop_ptr` 也只是一个 int 变量，用来表明 loop 是否已经初始化成功。

​        可以看到在EV_MULTIPLICITY没有启用的情况下，实际不存在ev_loop这个结构体。为了方便阅读源码，下面默认EV_MULTIPLICITY开启。

## watcher简介

​        watcher是不透明的，需要用户来分配内存，注册感兴趣的事件。主要用来解答三个问题：

1. 对哪些事件感兴趣
2. 对哪里句柄感兴趣
3. 当感兴趣的句柄上发生了感兴趣的事件，如何处理事件



​        ev_loop会在运行时监控所有watcher指定的句柄和事件，当某个句柄上发生某事件，会调用对应watcher的回调函数处理事件。



### 支持的事件



​         libev支持十几种事件，每种事件都有一个对应的watcher，分别是：

- 基础事件
  - ev_io                    // IO事件监控
  - ev_timer              // 相对定时器
  - ev_periodic           // 绝对定时器
  - ev_signal             // 信号处理
  - ev_child              // 子进程状态监控
  - ev_stat               // 文件属性监控
- 对循环的hook
  - ev_idle               // event loop空闲时触发的事件
  - ev_prepare            // 在event loop循环之前调用的
  - ev_check              // 在event loop循环之后调用的
  - ev_cleanup            // 在event loop退出时触发
- 扩展事件
  - ev_embed              // 嵌入另一个后台循环
  - ev_fork               // fork事件
  - ev_async              // 线程间异步事件



​         前面说过，所有的watcher共用一个“父类”——`struct ev_watcher `。



### 支持的event

```c
/* eventmask, revents, events... */
enum {
  EV_UNDEF    = (int)0xFFFFFFFF, /* guaranteed to be invalid */
  EV_NONE     =            0x00, /* no events */
  EV_READ     =            0x01, /* ev_io detected read will not block */
  EV_WRITE    =            0x02, /* ev_io detected write will not block */
  EV__IOFDSET =            0x80, /* internal use only */
  EV_IO       =         EV_READ, /* alias for type-detection */
  EV_TIMER    =      0x00000100, /* timer timed out */
#if EV_COMPAT3
  EV_TIMEOUT  =        EV_TIMER, /* pre 4.0 API compatibility */
#endif
  EV_PERIODIC =      0x00000200, /* periodic timer timed out */
  EV_SIGNAL   =      0x00000400, /* signal was received */
  EV_CHILD    =      0x00000800, /* child/pid had status change */
  EV_STAT     =      0x00001000, /* stat data changed */
  EV_IDLE     =      0x00002000, /* event loop is idling */
  EV_PREPARE  =      0x00004000, /* event loop about to poll */
  EV_CHECK    =      0x00008000, /* event loop finished poll */
  EV_EMBED    =      0x00010000, /* embedded event loop needs sweep */
  EV_FORK     =      0x00020000, /* event loop resumed in child */
  EV_CLEANUP  =      0x00040000, /* event loop resumed in child */
  EV_ASYNC    =      0x00080000, /* async intra-loop signal */
  EV_CUSTOM   =      0x01000000, /* for use by user code */
  EV_ERROR    = (int)0x80000000  /* sent when an error occurs */
};
```



### watcher的状态、对watcher的通用操作



​         watcher有四种状态：

- initialized，调用`ev_TYPE_init`对 watcher 进行初始化之后。
- started/running/active, 调用`ev_TYPE_start`之后的状态，并且开始等待事件。在这个状态下，除了特别提及的少数情况之外，watcher的内容不能被改变。
- pending,  当 watcher 是 active 并且一个让 watcher 感兴趣的事件到来，那么 watcher 进入 pending，事件等待被处理。除了特别提及的少数情况之外，watcher的内容不能被改变。
- stopped, 调用`ev_TYPE_stop`之后。



​         ev_TYPE_init、ev_TYPE_start、ev_TYPE_stop，ev_TYPE_set是对watcher的操作，TYPE是libev支持的事件类型具体名称，比如io、signal等等。



​        所有的ev_TYPE_init都是宏定义，展开之后是先调用ev_init，再调用具体的ev_TYPE_set 。也就是说先初始化“父类”，在初始化具体的“子类”。

​        ` ev_TYPE_start(EV_P_ ev_TYPE *w)`函数作用是将具体的ev_TYPE这个watcher加入到loop中。

​        `ev_TYPE_stop(EV_P_ ev_TYPE *w)`函数作用就是停止监控这个w这个watcher，从loop中删除。



​        由于ev_loop对于不同的watcher有着不同的组织方式，所以ev_TYPE_start函数在将具体的watcher绑定到loop中时，有着不同的操作逻辑。具体的` ev_TYPE_start`和`ev_TYPE_stop`函数分析会放到ev_loop分析之后。

## ev_loop

​        ev_loop是libev的核心，用来执行事件循环，组织各种各样的watcher，分发事件等等。ev_loop的成员变量都被放在“ev_vars.h”文件中，通过include展开。



### 创建

​        在EV_MULTIPLICITY宏启用的情况下，ev_loop有两种创建方式：

1. `struct ev_loop *ev_default_loop (unsigned int flags)`函数，创建默认的ev_loop，如果已经创建，则直接返回默认的ev_loop。这个函数不是线程安全的，不能在多线程中同时使用。会监控SIGCHLD信号。
2. `struct ev_loop *ev_loop_new (unsigned int flags)`函数，这个函数是线程安全的，但是创建出来的ev_loop不能处理信号。

​        多线程中，libev的一种常见使用方式是为每个线程动态创建一个循环，并在 “主” 线程中使用默认的循环。

​        flags 参数可被用于指定特殊的行为或要使用的特定后端，且通常被指定为 `0`（或 `EVFLAG_AUTO`）。

```c
/* flag bits for ev_default_loop and ev_loop_new */
enum {
  /* the default */
  EVFLAG_AUTO       = 0x00000000U, /* not quite a mask */
  /* flag bits */
  EVFLAG_NOENV      = 0x01000000U, /* do NOT consult environment */
  EVFLAG_FORKCHECK  = 0x02000000U, /* check for a fork in each iteration */
  /* debugging/feature disable */
  EVFLAG_NOINOTIFY  = 0x00100000U, /* do not attempt to use inotify */
#if EV_COMPAT3
  EVFLAG_NOSIGFD    = 0, /* compatibility to pre-3.9 */
#endif
  EVFLAG_SIGNALFD   = 0x00200000U, /* attempt to use signalfd */
  EVFLAG_NOSIGMASK  = 0x00400000U, /* avoid modifying the signal mask */
  EVFLAG_NOTIMERFD  = 0x00800000U  /* avoid creating a timerfd */
};
/* method bits to be ored together */
enum {
  EVBACKEND_SELECT   = 0x00000001U, /* available just about anywhere */
  EVBACKEND_POLL     = 0x00000002U, /* !win, !aix, broken on osx */
  EVBACKEND_EPOLL    = 0x00000004U, /* linux */
  EVBACKEND_KQUEUE   = 0x00000008U, /* bsd, broken on osx */
  EVBACKEND_DEVPOLL  = 0x00000010U, /* solaris 8 */ /* NYI */
  EVBACKEND_PORT     = 0x00000020U, /* solaris 10 */
  EVBACKEND_LINUXAIO = 0x00000040U, /* linux AIO, 4.19+ */
  EVBACKEND_IOURING  = 0x00000080U, /* linux io_uring, 5.1+ */
  EVBACKEND_ALL      = 0x000000FFU, /* all known backends */
  EVBACKEND_MASK     = 0x0000FFFFU  /* all future backends */
};
```



### 运行

```c
void ev_run (EV_P_ int flags); 
void ev_break (EV_P_ int how); 
```

这里比较关注flags和how这两个参数。flags有下面这几个：

- 0.通常这是我们想要的，每次轮询在poll都会等待一段时间然后处理pending事件。
- EVRUN_NOWAIT.运行一次，在poll时候不会等待。这样效果相当于只是处理pending事件。
- EVRUN_ONCE.运行一次，但是在poll时候会等待，然后处理pending事件。

而how有下面这几个：

- EVBREAK_ONE.只是退出一次ev_run这个调用。通常来说使用这个就可以了。
- EVBREAK_ALL.退出所有的ev_run调用。这种情况存在于ev_run在pengding处理时候会递归调用。



#### ev_run函数

```c
int ev_run (EV_P_ int flags)
{
...

  EV_INVOKE_PENDING; /* invoke all pending watchers */ /* in case we recurse, ensure ordering stays nice and clean ,确保之前发生的事件都已经被处理*/

  do
    {
#if EV_VERIFY >= 2  // 当EV_VERIFY >= 2时，用于校验当前的结构体是否正常
      ev_verify (EV_A);
#endif

#ifndef _WIN32
      if (ecb_expect_false (curpid)) /* penalise the forking check even more */
        if (ecb_expect_false (getpid () != curpid))
          {  // 这个进程是新fork出来的，会执行ev_fork事件的回调
            curpid = getpid ();
            postfork = 1;
          }
#endif

#if EV_FORK_ENABLE
      /* we might have forked, so queue fork handlers */
      if (ecb_expect_false (postfork))
        if (forkcnt)
          {
            queue_events (EV_A_ (W *)forks, forkcnt, EV_FORK);
            EV_INVOKE_PENDING;
          }
#endif

#if EV_PREPARE_ENABLE
      /* queue prepare watchers (and execute them) */
      if (ecb_expect_false (preparecnt))
        {
          queue_events (EV_A_ (W *)prepares, preparecnt, EV_PREPARE);
          EV_INVOKE_PENDING;
        }
#endif

      if (ecb_expect_false (loop_done))
        break;

      /* we might have forked, so reify kernel state if necessary */
      if (ecb_expect_false (postfork))
        loop_fork (EV_A);

      /* 根据fdchanges数组调整有改变的文件描述符和事件 */
      /* update fd-related kernel structures using backend_modify()*/
      fd_reify (EV_A);

      /* 计算poll应该阻塞的时间,这个时间与定时器超时时间有关 */
      /* calculate blocking time */
      {
......
#if EV_FEATURE_API
        ++loop_count;
#endif
        assert ((loop_done = EVBREAK_RECURSE, 1)); /* assert for side effect */
          /*调用IO复用函数，例如epoll_poll()*/
        backend_poll (EV_A_ waittime);
        assert ((loop_done = EVBREAK_CANCEL, 1)); /* assert for side effect */

        pipe_write_wanted = 0; /* just an optimisation, no fence needed */

        ECB_MEMORY_FENCE_ACQUIRE;
        if (pipe_write_skipped)
          {
            assert (("libev: pipe_w not active, but pipe not written", ev_is_active (&pipe_w)));
            ev_feed_event (EV_A_ &pipe_w, EV_CUSTOM);
          }

        /* update ev_rt_now, do magic */
        time_update (EV_A_ waittime + sleeptime);
      }
      
      /* 如果栈顶元素的超时时间已经超过了当前时间，则将栈顶元素的监控器添加到
       * loop->pendings中，并调整堆结构，接着判断栈顶元素是否仍超时，一致重复，
       * 直到栈顶元素不再超时。
       */
      /* queue pending timers and reschedule them */
      timers_reify (EV_A); /* relative timers called last */
#if EV_PERIODIC_ENABLE
      periodics_reify (EV_A); /* absolute timers called first */
#endif

#if EV_IDLE_ENABLE
      /* queue idle watchers unless other events are pending */
      idle_reify (EV_A);
#endif

#if EV_CHECK_ENABLE
      /* queue check watchers, to be executed first */
      if (ecb_expect_false (checkcnt))
        queue_events (EV_A_ (W *)checks, checkcnt, EV_CHECK);
#endif

      EV_INVOKE_PENDING;
    } while (ecb_expect_true (
    activecnt
    && !loop_done
    && !(flags & (EVRUN_ONCE | EVRUN_NOWAIT))
  ));

  if (loop_done == EVBREAK_ONE)
    loop_done = EVBREAK_CANCEL;
...

  return activecnt;
}
```

总体流程：

- 触发那些已经pending的watchers，包括fork watcher和prepare watcher.
- 判断是否loop_done
- 用fd_reify调整有改变的fd和事件
- 计算出poll阻塞的等待时间，并且进行必要的sleep
- backend_poll开始轮询,并且整理好pending事件
- 使用timers_reify和periodics_reify调整相对定时器和绝对定时器的堆结构
- 触发那些已经pending的watchers，包括check watcher和io watcher



### backend

ev_loop中有几个与backend相关的成员变量，

```c
VARx(int, backend)
VARx(int, backend_fd)
VARx(ev_tstamp, backend_mintime) /* assumed typical timer resolution */
VAR (backend_modify, void (*backend_modify)(EV_P_ int fd, int oev, int nev))
VAR (backend_poll  , void (*backend_poll)(EV_P_ ev_tstamp timeout))
```

backend指明此ev_loop使用的是哪种系统调用。可取值如下

```c
enum {
  EVBACKEND_SELECT   = 0x00000001U, /* available just about anywhere */
  EVBACKEND_POLL     = 0x00000002U, /* !win, !aix, broken on osx */
  EVBACKEND_EPOLL    = 0x00000004U, /* linux */
  EVBACKEND_KQUEUE   = 0x00000008U, /* bsd, broken on osx */
  EVBACKEND_DEVPOLL  = 0x00000010U, /* solaris 8 */ /* NYI */
  EVBACKEND_PORT     = 0x00000020U, /* solaris 10 */
  EVBACKEND_LINUXAIO = 0x00000040U, /* linux AIO, 4.19+ */
  EVBACKEND_IOURING  = 0x00000080U, /* linux io_uring, 5.1+ */
  EVBACKEND_ALL      = 0x000000FFU, /* all known backends */
  EVBACKEND_MASK     = 0x0000FFFFU  /* all future backends */
};

#if EV_USE_EPOLL
      if (!backend && (flags & EVBACKEND_EPOLL   )) backend = epoll_init     (EV_A_ flags);
#endif
```

​        创建ev_loop的两个函数中都会调用`loop_init`函数，loop_init函数会具体选择一种可用的系统调用来使用，同时进行相应的初始化，比如epoll_init、select_init。

​        在具体的初始化中，会设置backend_poll、backend_modify、backend_mintime三个成员变量的值。backend_poll和backend_modify是函数指针，`void (*backend_poll)(EV_P_ ev_tstamp timeout)`，初始化后会指向具体的函数。比如epoll_init函数，

```c
int epoll_init (EV_P_ int flags)
{
  if ((backend_fd = epoll_epoll_create ()) < 0)
    return 0;

  backend_mintime = EV_TS_CONST (1e-3); /* epoll does sometimes return early, this is just to avoid the worst */
  backend_modify  = epoll_modify;
  backend_poll    = epoll_poll;

  epoll_eventmax = 64; /* initial number of events receivable per poll */
  epoll_events = (struct epoll_event *)ev_malloc (sizeof (struct epoll_event) * epoll_eventmax);

  return EVBACKEND_EPOLL;
}
```



### epoll_poll

当backend是EVBACKEND_EPOLL时，backend_poll函数指针指向epoll_poll函数。

```c
static void epoll_poll (EV_P_ ev_tstamp timeout)
{
......
  EV_RELEASE_CB;
  eventcnt = epoll_wait (backend_fd, epoll_events, epoll_eventmax, EV_TS_TO_MSEC (timeout));
  EV_ACQUIRE_CB;
......
  for (i = 0; i < eventcnt; ++i)
    {
      struct epoll_event *ev = epoll_events + i;

      int fd = (uint32_t)ev->data.u64; /* mask out the lower 32 bits */
      int want = anfds [fd].events;
      int got  = (ev->events & (EPOLLOUT | EPOLLERR | EPOLLHUP) ? EV_WRITE : 0)
               | (ev->events & (EPOLLIN  | EPOLLERR | EPOLLHUP) ? EV_READ  : 0);

......
      fd_event (EV_A_ fd, got);
    }

  /* if the receive array was full, increase its size */
  if (ecb_expect_false (eventcnt == epoll_eventmax))
    {
      ev_free (epoll_events);
      epoll_eventmax = array_nextsize (sizeof (struct epoll_event), epoll_eventmax, epoll_eventmax + 1);
      epoll_events = (struct epoll_event *)ev_malloc (sizeof (struct epoll_event) * epoll_eventmax);
    }
......
}
```

epoll_poll主要就是：

- 调用epoll_wait等待事件到来
- fd_event解析事件的类型，然后按照优先级放到pendings里面



## IO watcher分析

文件IO是网络编程的核心部分，这里对IO watcher进行简单分析。

```c
#include <ev.h>
#include <stdio.h>

// every watcher type has its own typedef'd struct with the name ev_TYPE
ev_io stdin_watcher;
ev_timer timeout_watcher;

// all watcher callbacks have a similar signature
// this callback is called when data is readable on stdin
static void stdin_cb (EV_P_ ev_io *w, int revents)
{
	puts ("stdin ready");
	// for one-shot events, one must manually stop the watcher
	// with its corresponding stop function.
	ev_io_stop (EV_A_ w);

	// this causes all nested ev_run's to stop iterating
	ev_break (EV_A_ EVBREAK_ALL);
}

int main (void)
{
	// use the default event loop unless you have special needs
	// struct ev_loop *loop = EV_DEFAULT; /* OR ev_default_loop(0) */
	EV_P = EV_DEFAULT;

	// initialise an io watcher, then start it
	// this one will watch for stdin to become readable
	ev_io_init(&stdin_watcher, stdin_cb, /*STDIN_FILENO*/ 0, EV_READ);
	ev_io_start(EV_A_ &stdin_watcher);

	ev_loop(EV_A_ 0); /* now wait for events to arrive */

	return 0;
}
```



### anfds

​        ev_loop使用anfds管理ev_io。anfds是一个元素为ANFD的数组，ANFD是一个链表，所以anfds是一个元素为链表的数组。

```c
VARx(ANFD *, anfds)
VARx(int, anfdmax)      // anfds数组所能容纳的最大元素个数

typedef ev_watcher_list *WL;

/* file descriptor info structure */
typedef struct
{
  WL head;
  unsigned char events; /* the events watched for */
  unsigned char reify;  /* flag set when this ANFD needs reification (EV_ANFD_REIFY, EV__IOFDSET) */
  unsigned char emask;  /* some backends store the actual kernel mask in here */
  unsigned char eflags; /* flags field for use by backends */
#if EV_USE_EPOLL
  unsigned int egen;    /* generation counter to counter epoll bugs */
#endif
#if EV_SELECT_IS_WINSOCKET || EV_USE_IOCP
  SOCKET handle;
#endif
#if EV_USE_IOCP
  OVERLAPPED or, ow;
#endif
} ANFD;
```

​        既然ANFD是文件描述符信息结构体，怎么没在结构体里看到fd变量？这是因为fd已经作为anfds数组的下标，anfds[fd] = ANFD。这样可以快速找到与指定fd相关的所有watcher。综上，anfds结构如图：

![](/assets/images/libev-anfds-arch.png)



### ev_io_start 函数

`void ev_io_start(EV_P_ ev_io *w)` 函数是把w加入到loop中，源码如下：

```c
void ev_io_start (EV_P_ ev_io *w) EV_NOEXCEPT
{
  int fd = w->fd;

  if (ecb_expect_false (ev_is_active (w)))
    return;

  assert (("libev: ev_io_start called with negative fd", fd >= 0));
  assert (("libev: ev_io_start called with illegal event mask", !(w->events & ~(EV__IOFDSET | EV_READ | EV_WRITE))));
.......

  ev_start (EV_A_ (W)w, 1);
  array_needsize (ANFD, anfds, anfdmax, fd + 1, array_needsize_zerofill);
  wlist_add (&anfds[fd].head, (WL)w);

  /* common bug, apparently */
  assert (("libev: ev_io_start called with corrupted watcher", ((WL)w)->next != (WL)w));

  fd_change (EV_A_ fd, w->events & EV__IOFDSET | EV_ANFD_REIFY);
  w->events &= ~EV__IOFDSET;
}
```

ev_io_start函数内容主要是以下几点：

1. `ev_start (EV_A_ (W)w, 1);`

```c
void ev_start (EV_P_ W w, int active)
{
  pri_adjust (EV_A_ w);  // 调整w的优先级
  w->active = active;    // 设置w的状态的active
  ev_ref (EV_A);         // 递增loop的activecnt
}
```

2. array_needsize对anfds数组扩容，wlist_add把w加入到anfds[fd]的链表中。
3. fd_change函数递增fdchangecnt，将fd加入到fdchanges数组中。fdchanges数组用来指出哪些fd发生了变化，新增了watcher，删除了watcher等等。在下次事件循环之前，由fd_reify函数进行调整。



## 其他



### array_needsize 宏

​         在Libev中，如果数组需要扩容，会使用array_needsize宏进行处理。比如，元素类型为ANFD的数组anfds，当前大小为anfdmax，要扩大为fd+1，新扩容的空间使用array_needsize_zerofill进行初始化：

`array_needsize (ANFD, anfds, anfdmax, fd + 1, array_needsize_zerofill);`

​        array_needsize宏的主要工作在array_realloc函数中，

```c
static void *array_realloc (int elem, void *base, int *cur, int cnt)
{
  *cur = array_nextsize (elem, *cur, cnt);   // 计算最终的元素个数
  return ev_realloc (base, elem * *cur);     // 负责realloc
}
```

array_nextsize负责计算数组的最终元素个数，但它并没有直接返回fd+1，而是先把原来的数组大小翻倍，再将数组所占内存大小向上调整成MALLOC_ROUND的倍数，然后计算出实际的扩容后的数组元素个数。

```c
/* find a suitable new size for the given array, */
/* hopefully by rounding to a nice-to-malloc size */
inline_size int array_nextsize (int elem, int cur, int cnt)
{
  int ncur = cur + 1;

  do
    ncur <<= 1;
  while (cnt > ncur);

  /* if size is large, round to MALLOC_ROUND - 4 * longs to accommodate malloc overhead */
  if (elem * ncur > MALLOC_ROUND - sizeof (void *) * 4)
    {
      ncur *= elem;
      ncur = (ncur + elem + (MALLOC_ROUND - 1) + sizeof (void *) * 4) & ~(MALLOC_ROUND - 1);
      ncur = ncur - sizeof (void *) * 4;
      ncur /= elem;
    }

  return ncur;
}
```



### pendings 数组

​        pendings是一个二维数组，按照优先级组织事件已经发生的watcher。ANPENDING结构记录了处于PENDING状态的监视器以及触发的事件。

```c
VAR (pendings, ANPENDING *pendings [NUMPRI])
VAR (pendingmax, int pendingmax [NUMPRI])   //记录pendings数组第二维的容量
VAR (pendingcnt, int pendingcnt [NUMPRI])   // 记录pendings数组第二维的长度
VARx(int, pendingpri) /* highest priority currently pending */
    
/* stores the pending event set for a given watcher */
typedef struct
{
  W w;
  int events; /* the pending event set for the given watcher */
} ANPENDING;
```

**ev_feed_event**负责把已经被触发的w加入到pendings数组。

```c
void ev_feed_event (EV_P_ void *w, int revents) EV_NOEXCEPT
{
  W w_ = (W)w;
  int pri = ABSPRI (w_);

  if (ecb_expect_false (w_->pending))
      // w_->pending不为0，该watcher已经处于PENDING状态
    pendings [pri][w_->pending - 1].events |= revents;
  else
    {
      w_->pending = ++pendingcnt [pri];  // pendingcnt[pri]记录了pendings[pri]目前的长度
      array_needsize (ANPENDING, pendings [pri], pendingmax [pri], w_->pending, array_needsize_noinit);
      pendings [pri][w_->pending - 1].w      = w_;
      pendings [pri][w_->pending - 1].events = revents;
    }
  pendingpri = NUMPRI - 1;
}
```

w_->pending的值：

- 该值为0表示该watcher第一次被激活，
- 不为0表示的是该watcher已经处于PENDING状态，而其具体的值，代表该watcher在pendings [pri]中的位置，也就是当前(loop->pendingcnt) [pri]的值。



**EV_INVOKE_PENDING**宏用来遍历pendings数组，调用所有已经触发事件的watcher的callback。

```c
# define EV_INVOKE_PENDING ev_invoke_pending (EV_A)

void
ev_invoke_pending (EV_P)
{
  pendingpri = NUMPRI;

  do
    {
      --pendingpri;

      /* pendingpri possibly gets modified in the inner loop */
      while (pendingcnt [pendingpri])
        {
          ANPENDING *p = pendings [pendingpri] + --pendingcnt [pendingpri];

          p->w->pending = 0;
          EV_CB_INVOKE (p->w, p->events);
          EV_FREQUENT_CHECK;
        }
    }
  while (pendingpri);
}
```







## 参考

- libev 源码详解：https://jin-yang.github.io/post/linux-libev-source-code-details-introduce.html
- libev 详解：https://metacpan.org/pod/distribution/EV/libev/ev.pod
- libev 使用示例：https://jin-yang.github.io/post/linux-libev.html



