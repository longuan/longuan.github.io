---
title: "golang中sync.Map的设计与实现"
---



sync.Map是golang标准库中提供的并发安全的map。在某些场景下，使用sync.Map比map+Mutex/RWMutex有更优越的表现。为了弄清具体哪些场景应该使用sync.Map，以及它的优缺点，下面进行详细分析。

sync.Map的源码实现只有一个文件src/sync/map.go，在GitHub仓库中[^1]可以看到此文件的所有commit历史。从commit历史来看，sync.Map的源码变动不大，注释倒是修改了好几次。我看的源码是golang 1.17，属于是比较新的版本了。  

sync.Map是在2017年作为golang 1.9的一个feature发布的。它的作者bcmills，当年也发表了一个talk[^2][^3]，对sync.Map做了介绍。

## 为什么需要sync.Map

这一节的内容主要来自sync.Map作者的talk，毕竟没人比他更明白sync.Map设计的初衷。

从talk中可以知道，sync.Map之所以诞生，主要是因为标准库有需要。将sync.Map置于标准库中，其他标准库就可以使用，比如reflect包、json的encode、gob的Register等等现在都在使用sync.Map。标准库的需求点就是sync.Map的努力方向。

那么标准库对于map的使用遇到了哪些问题呢？sync.RWMutex is slow

![](/assets/images/cahce-contention-in-rwmutex.png)
上图中通过一连串的分析，最终发现是由于cache contention才导致整个操作变慢。具体一点就是，由于RWMutex的RLock方法每次都会对成员变量readerCount做`atomic.AddInt32`，如果在多个分别运行在不同CPU上的线程中，同时对同一个RWMutex加读锁，那么每次做完`atomic.AddInt32`都会invalidate其他几个CPU 的local cache（比如L1、L2 cache）中的readerCount变量所在的cache line，并把自己的readerCount标识为up-to-date。其他几个CPU会重新拉取up-to-date的数据到自己的local cache中。这些并发的RLock调用最终因为cache contention变成了串行。

MESI协议中，一个CPU如果想更新一个共享的cache line，需要向其他CPU的local cache发送invalidate message，然后在自己的local cache中更改此cache line的状态为exclusive，再更新此cache line。一个CPU想要读取某个cache line，但是local cache中此cache line状态是invalid，那么会从其他CPU的local cache中拷贝此cache line的最新值，并把此cache line的状态设置为share。invalidate cache line和cache line的拷贝相对来说都是比较耗时的。

所以现在，reflect包中的sync.RWMutex+map组合都替换成了sync.Map。

这里提出一个问题：sync.Map是如何解决**cache contention**的？解决的代价是什么？

## 使用方法

sync.Map提供了自己的一套API，与map的大概类比如下表所示。可以看出sync.Map的方法命名更倾向于atomic那一类，这也是作者的初衷——`sync.Map ~= a map of atomic pointers`。

||map|sync.Map|
|--|--|--|
|获取某个value|`map[key]`|`m.Load(key)`|
|添加key-value|`map[key] = value`|`m.Store(key, value) / m.LoadOrStore(key, value)`|
|删除一个 key|`delete(map, key)`|`m.Delete(key) / m.LoadAndDelete(key)`|
|遍历|`for…range`|`m.Range(func())`|

同时sync.Map缺少一些方法，比如len、swap（最新版本已经有了），据作者表示，可以加但是没必要，可以在github上看到详细讨论[^4]。

## 设计与实现

后面所述，纯属个人理解，如有不对，敬请指正。

### 设计要点

**首要问题是解决cache contention**，提高读取效率。
- 既然RWMutex和mutex效率都很低，那么就加一个read-only的缓存，作为fast path。read-only可以消除cache contention。
- 同时用一个原子变量指向read-only缓存，就可以保证read-only在替换的时候是并发安全的。减少了锁的使用。
- slow path用一个read-write map来兜底。由于read-write map需要能够读写，所以使用mutex保护
- 为了避免一直走slow path，在合适的时机将read-write map转换为read-only

有了基本思路，可以进行如下简单设计：
`Load(key)`：
1. read-only里找到了key，直接返回对应的value
2. read-only里没有此key，去read-write里找

`Store(key, value)`:
1. 将数据直接写入到read-write map中
2. 如果不是一个新key，就替换掉read-only map

`Delete(key)`:
1. 从read-write map中删除key
2. 替换掉read-only map

上面这个愚蠢的设计，只适合极端读多写少的场景，其他场景下性能完全不如sync.RWMutex+map。但是别急，后面加一点点细节，就成了现在的sync.Map。

**要点1**——引入entry：

牢记名言——“计算机科学领域的任何问题都可以通过增加一个间接的中间层来解决”。
```go
type entry struct {
    p unsafe.Pointer // *interface{}
}
```

使用此entry结构体来代替真正的value，有如下几个好处：
- 减少内存占用，不需要把真正的value复制两份分别存储在read-only和read-write中
- 可以将Delete操作简化为Store nil操作，延迟删除，不必每次Delete都走slow path
- 可以通过read-only直接更新value，这样的话就是无锁操作，会更快，减少走slow path
	- 如果keys是stable，那么所有的操作都可以在read-only中完成

引入entry之后，可以进行如下设计：
- read-only里面有这个key，执行load或者store之后返回（delete相当于store nil）
- read-only里面没有这个key，如果read-write存在，就去read-write里面做load/store/delete
- read-only里面没有这个key，如果read-write不存在
	- Load、Delete操作的话可以直接返回此key不存在
	- Store操作的话就需要新建read-write，然后写入这个key

**要点2**——替换read-only：

添加新key，会直接写入到read-write map，后续对这个新key的Load操作都会走slow path。为了解决这个问题，sync.Map会记录load和delete的穿透次数，如果达到一个阈值，会用read-write更新替换掉read-only。

阈值的选取？
- 当前代码中是使用穿透的次数与read-write map的长度作比较。如果大于，说明有很多操作都走了slow path，就会替换read-only。

如何生成sync.Map最新的状态并替换掉read-only？
- 增量+merge的方法。read-write map只记录read-only中没有的key，后面要替换read-only的时候，将read-write和read-only中的数据做merge生成最新的数据，然后替换read-only，并将read-write map清空。
- 全量的方法。read-write时刻记录着全部的数据，要替换read-only只需将read-write复制出来一份，然后用其替换掉ead-only。
- Copy-On-Write的方法。在有新key到来，需要新建read-write的时候，从read-only做COW。要替换read-only只需将指针指向read-write，然后将read-write指针置空。这也是目前sync.Map采用的方式。

为什么不记录Store的穿透次数？
其实最开始的实现中，只有Load发生穿透才会被记录下来，后来在这个commit[^6]中，把Delete的穿透操作也记录下来了。至于为什么不记录Store的穿透次数，我估计是因为sync.Map的目标场景并不是这种写多的场景。

**要点3**——区分nil和expunged entry：

前面提到，通过引入entry，使得delete操作能够转换为store nil的操作，延迟删除，效率更高。那么延迟到什么时候删除呢？

答案已经很明显了——做COW的时候。那么又产生了一个问题——在从read-only拷贝出来一份作为read-write的时候是否拷贝nil entry（也就是被删除的那些key-value）：
- 如果不拷贝。一个已经删除的key会只存在于read-only中，不存在于read-write中。如果后续有对此key的store操作，会导致此key只出现在read-only中，而不出现在read-write中，此时用read-write替换read-only则会丢失数据。
- 如果拷贝。这就相当于没有做删除，read-write会变得无限大。

引入expunged就是为了解决这个问题——标识一个key是被真正删除了：
- nil表示这个key是被延迟删除了，可以直接store。
- expunged表示这个key是被真正删除了，在read-write map中不存在。要想再次store这个key，必须要在read-write map中添加这个key。


**要点4**——引入amended：

amended有两种理解方法：
- 一种是，amended为true，表示read-write map中有read-only没有的key；为false，表示read-only与read-write map一致。
- 另一种是，amended为true，表示read-write map存在；为false，表示read-write map不存在。

我觉得第二种更好理解，因为read-write map是从read-only中COW出来的，所以read-write存在的话一定比read-only多了一些新key。

引入amended的好处是：
- 如果load一个不存在的key，在read-only中肯定找不到，正常是会访问read-write map，也就是进入slow path。有了amended，就可以避免read-write map中没有新key，会无脑进入slow path的问题。
- 可以快速判断是否要进入slow path。因为amended也是read only的，跟read-only map共生共死，所以访问amended是不用加锁的。 


### 实现

根据设计所述，就有了最后的实现，关键数据结构如下所示：
```go
type Map struct {
    mu      Mutex
    read    atomic.Value   // readOnly
    dirty   map[interface{}]*entry
    misses  int
}

type readOnly struct {
    m       map[interface{}]*entry
    amended bool // true if the dirty map contains some key not in m.
}

type entry struct {
    p unsafe.Pointer // *interface{}
}
```

详细的实现就不解释了，基本都是按照上面的设计要点做的。下面讲几个我比较感兴趣的实现细节：

**Double-checked Locking**：

Double-checked Locking是一种”优化小技巧“，可以在不需要获取锁的情况下不进入临界区，减少锁争抢带来的开销。在单例模式中非常常见：
```java
public class DclSingleton {
    private static volatile DclSingleton instance;
    public static DclSingleton getInstance() {
        if (instance == null) {
            synchronized (DclSingleton .class) {
                if (instance == null) {
                    instance = new DclSingleton();
                }
            }
        }
        return instance;
    }
}
```
即使有很多线程通过了第一道检查，也只有一个线程能进入到临界区。关键是要将写入和第二次检查都放置在临界区内。

sync.Map中大量使用了这种模式，以Load为例：
```go
func (m *Map) Load(key interface{}) (value interface{}, ok bool) {
	检查read-only
	mu.Lock()
	再次检查read-only
	读写dirty
	替换read-only
	mu.Unlock()
}
```


**Compare-and-swap loop**：

CAS loop是一个非常实用的无锁编程技巧[^5]。常见的写法就是：
```go
var i int32
for {
	p := atomic.LoadInt32(&i)
	if atomic.CompareAndSwapInt32(&i, p, p+1) {
		break
	}
}
```

众所周知，两个原子操作进行简单组合之后，整个过程就不是原子的。但是CompareAndSwapInt32操作是特殊的，它天然能避免写写冲突（虽然无法避免ABA的情况，但在sync.Map场景下可以接受），再加上atomic的所有操作都能避免读写冲突，因此LoadInt32和CompareAndSwapInt32组合起来是线程安全的。如果没有其他线程并发地对i写入，那么这个循环只需要跑一次就能退出。如果有其他线程并发地对i写入，并且是在LoadInt32和CompareAndSwapInt32之间，那么CompareAndSwapInt32会失败，循环重试，直至CompareAndSwapInt32成功。总体来说，这个`+1`的操作在外界看来是原子的。

sync.Map源码中，对一个entry的修改分为两类：
- 有锁保护的，直接用的是对应原子方法。比如`entry.storeLocked`，`entry.unexpungeLocked`
- 无锁保护的，用的都是上面介绍的CAS loop方法。比如`entry.delete`, `entry.tryStore`等等

这里可以对比看一下：
```go
func (e *entry) storeLocked(i *interface{}) {
	atomic.StorePointer(&e.p, unsafe.Pointer(i))
}

func (e *entry) tryStore(i *interface{}) bool {
	for {
		p := atomic.LoadPointer(&e.p)
		if p == expunged {
			return false
		}
		if atomic.CompareAndSwapPointer(&e.p, p, unsafe.Pointer(i)) {
			return true
		}
	}
}
```

storeLocked 方法调用的前提是entry必须是unexpunged，因为真正删除的entry不能直接store，前面设计要点中讲过原因。要确保entry必须要unexpunged，就必须要读一次entry，然后才能调用storeLocked方法。为了确保这个读写过程是线程安全的，这里采用了加锁。就是由storeLocked的调用者保证`判断entry是unexpunged（读），然后调用entry的storeLocked（写）`这个过程是始终加锁的。

tryStore方法中会自己判断entry是否是expunged，如果不是会进行CAS loop。就是因为要判断entry是否是expunged，这里才会写成CAS loop，否则的话可以直接用`atomic.StorePointer`。通过CAS loop，可以保证`总是在entry是unexpunged（读）的情况下做store（写）`这一过程是原子的。

**Range**：

这里主要想说的是：sync.Map的Range方法是不会阻塞读写的，所以range过程中数据是会变的。下面具体说说怎么变。

Range方法在开头会使用read-write map替换掉read-only，然后会迭代读取这个read-only。read-only是不会变，它是一个key-entry的集合。但是这里需要注意的是entry是对value指针的封装，所以Range方法是能觉察到对现有key的store。但是无法看到新添加的key，而且如果后续read-only又被替换了，那么Range也是无感的，会一直读取那个read-only。

也就是说Range方法从始至终，看到的key集合是不变的，value集合是可能会变的。


## 适用场景

就像map无法应对并发场景的问题，sync.Map也不可能做到完美。但是针对某些场景做到了更优，那么结合sync.Map的设计和实现来说，它的适用场景都有哪些呢？

- **stable keys**：  key集合是稳定的。比如统计每个省份有多少人，省份名称作为key，人数作为value，key总共有34个。
	- 这个很好理解，因为stable keys情况下，read-only map最终会趋于固定，不会再变，每次读写都可以走fast path。
- disjoint stores： 每个CPU core（或者线程）专门负责各自keys的store操作，每个CPU core的keys不相交。最理想的是write-once, read-many。
	- 多个CPU core对同一个key同时写入，也会引起cache contention，就像RWMutex.RLock那样。
	- disjoint stores相当于每个CPU core会独占一些keys的store操作，这些store操作只需要更新CPU的local cache就可以，不会与其他cache发生content。
- concurrent loops：多core并发，且有大量写入。否则使用sync.RWMutex+map也是个不错  的选择。
	- concurrent表示有多个core同时访问，loops表示有大量写入。只有在此情况下才会有cache contention，否则的话用RWMutex+map可能也是个不错的选择。

上面三个是sync.Map作者认为的最适用sync.Map的场景。

比如标准库中的一个use case：Lazy construction，也是sync.Map的original motivating example。
-  reflect type、json.Encoder等等，具有stable keys特征（类型有限），虽然会有一个warm-up的过程
标准库中的另一个use case：registries。


这里可以回忆一下：sync.Map作者为什么认为具有上面三个特征的场景是最适合sync.Map？sync.Map分别做了哪些优化？

另外是作者的建议：if cache contention is not the problem, sync.Map is probably not the solution.


## benchmark

map_bench_test.go中提供了大量的benchmark测试。主要用来对比三种map的性能：
- DeepCopyMap 。跟设计要点开始时那个愚蠢的设计差不多
- RWMutexMap 。sync.RWMutex+map的实现
- sync.Map

下面依次列出来在我电脑上跑这些benchmark的对比结果：
```
								 DeepCopyMap     RWMutexMap     sync.Map
BenchmarkLoadMostlyHits          5.879 ns/op     34.85 ns/op    6.267 ns/op
BenchmarkLoadMostlyMisses        4.510 ns/op     33.64 ns/op    4.681 ns/op
BenchmarkLoadOrStoreBalanced     ~               323.8 ns/op    332.9 ns/op
BenchmarkLoadOrStoreUnique       ~               513.2 ns/op    551.1 ns/op
BenchmarkLoadOrStoreCollision    2.434 ns/op     53.98 ns/op    2.779 ns/op
BenchmarkLoadAndDeleteBalanced   ~               55.01 ns/op    4.890 ns/op
BenchmarkLoadAndDeleteUnique     ~               58.43 ns/op    5.025 ns/op
BenchmarkLoadAndDeleteCollision  101.3 ns/op     39.08 ns/op    1.475 ns/op
BenchmarkRange                   1627 ns/op      40243 ns/op    1649 ns/op
BenchmarkDeleteCollision         89.25 ns/op     37.41 ns/op    1.598 ns/op
```


可以看出来，除了在BenchmarkLoadOrStoreUnique和BenchmarkLoadOrStoreBalanced两个测试中，sync.Map稍微落后于RWMutexMap，其他测试中都优于RWMutexMap。

- BenchmarkLoadOrStoreUnique 测试内容为：多个线程都在一直store新的key。前面提过，在sync.Map中添加新key会走slow path，会退化为map+sync.Mutex。
- BenchmarkLoadOrStoreBalanced 测试内容为：多个线程并行运行，每个线程先做多次load、再做多次store，或者先做多次store、再做多次load，load和store次数相同。load的key都是已经预先写入的，store的key都是新的。也再次证明了sync.Map在添加新key的场景下表现不佳，不过相比BenchmarkLoadOrStoreBalanced，sync.Map与RWMutexMap的差距缩小。


## 总结

sync.Map通过使用read-only、entry中间层和原子操作等等优化，减少了很多场景下的cache contention和lock contention。在读负载较重的场景下，表现非常优异；在需要频繁添加新key的场景下，表现不如RWMutexMap。

这里强调一点，说sync.Map只适合读多写少的场景是错误的，应该说sync.Map不适合频繁写入新key的场景。

sync.Map的设计及优化真的无可挑剔、惊为天人。虽然初读时比较晦涩，但是弄清整个思路之后，就会发现有一种浑然天成的美感。更让人吃惊的是，带上注释、空行，代码竟然还不到四百行。

下面也说一下sync.Map的**缺点**：
1. 占用内存比map略有上升
2. 多了一层指针嵌套，对运行时的时间和空间来说都有些增加
3. 影响到用户代码的逃逸分析
4. 需要在运行时额外使用type-assertion，相比之下map可以在编译期做
5. 换了一套API，并且不全（比如没有len方法）


![](/assets/images/so-much-pointers-in-syncmap.png)


[^1]: https://github.com/golang/go/commits/master/src/sync/map.go
[^2]: https://www.youtube.com/watch?v=C1EtfDnsdDs
[^3]: https://github.com/gophercon/2017-talks/tree/master/lightningtalks/BryanCMills-AnOverviewOfSyncMap
[^4]: https://github.com/golang/go/issues/20680
[^5]: https://preshing.com/20150402/you-can-do-any-kind-of-atomic-read-modify-write-operation/
[^6]: https://github.com/golang/go/commit/2e8dbae85ce88d02f651e53338984288057f14cb