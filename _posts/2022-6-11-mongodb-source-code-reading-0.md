---
title: "Mongodb内核代码阅读0"
---

- StringData
- Status、StatusWith
    - Status
    - StatusWith
- invariant、assert、exception
    - invariant
    - assert
    - exception
- Decorable
    - Decorable、Decoration、D、T之间的关系
    - 如何使用Decorable这套机制
    - 添加一个装饰品的内部实现
    - 取出一个装饰品的内部实现
- TransportLayer、ServiceExecutor、ServiceEntryPoint
    - 三者的使用
    - TransportLayer
        - 接收连接
    - ServiceEntryPoint
    - ServiceExecutor
        - ServiceExecutorAdaptive
        - ServiceExecutorSynchronous



Mongodb是目前最流行的文档数据库，代码开源。出于工作需要以及个人兴趣，打算看一下它内核的源码。代码在github上就有，我看的是4.2版本的，https://github.com/mongodb/mongo/tree/v4.2 。从github上看，这个项目至少已经有了15年历史，能够健康发展到今天，源码想必一定经过了不少次迭代，有不少历史沉淀和精华。简单统计了一下，c/c++代码约80W行，并不是一个小数目，应该也是我第一次看这么大型的项目源码。

自己也没什么源码阅读经验，加上这么大的代码量以及这个源码至少有15年的历史，开始时我是懵逼的，就像看天书一样，26个英文字母都认识，但是连一起就看不懂了。特别是它的源码大量使用到了C的宏和C++的模板等等，优点还是很明显的，可以让代码看起来更简洁，更有表达力，执行效率更高，但是对于初学者很不友好。其实这不是这个项目的问题，是我的问题，因为源码本来就是给开发人员看的，每个项目都有自己的“暗号”和“字典”，通过这些“暗号”和“字典”，开发人员能够大幅提高开发效率。所以，我决定慢点来，从项目的“暗号”和“字典”开始了解源码。

这篇文章是阅读mongodb内核源码的第0章，算是我的一个起步，总结了mongodb内核代码中常见的一些名词和机制。

## StringData

> A StringData object wraps a 'const std::string&' or a 'const char*' without copying its contents. The most common usage is as a function argument that takes any of the two forms of strings above. Fundamentally, this class tries go around the fact that string literals in C++ are char[N]'s.

StringData只有两个成员变量，都是私有的，一个是data，一个是size，分别表示字符串的头指针和长度。这个类和C++17中引入的string_view基本等价，多了一些便捷的工具函数，比如startsWith、endsWith等等。

可能是十几年前C++17标准没有出来，开发人员自己写来作此用途的。

StringData里有一个嵌套结构体，TrustedInitTag。它没有任何成员和方法，用途只是用作占位。其作用就是来标识此StringData的构造是否是安全的，如果不安全则进行必要的检查，保证得到的StringData 实例是可用的。

```cpp
class StringData{
private: 
  constexpr StringData(const char* c, size_t len, TrustedInitTag) : _data(c), _size(len) {}
public:
  constexpr StringData() = default;   
  StringData(const char* str) : StringData(str, str ? std::strlen(str) : 0) {} 
  StringData(const std::string& s) : StringData(s.data(), s.length(), TrustedInitTag()) {}  
  StringData(const char* c, size_t len) : StringData(c, len, TrustedInitTag()) {
    invariant(_data || (_size == 0));
  }   // 确保_data为nullptr时_size也为0
}
```

另外需要提的是一个友元函数，对双引号进行了重载，其作用是将一个string literal在编译期转化为StringData。这个特性是C++11引入的，https://en.cppreference.com/w/cpp/language/user_literal。

```cpp
/**
 * Constructs a StringData from a user defined literal.  This allows
 * for constexpr creation of StringData's that are known at compile time.
 */
constexpr friend StringData operator"" _sd(const char* c, std::size_t len);

constexpr StringData operator"" _sd(const char* c, std::size_t len) {
    return StringData(c, len, StringData::TrustedInitTag{});
}

constexpr StringData kMajorityReadConcernStr = "majority"_sd;
```


## Status、StatusWith

Status和StatusWith在mongodb源码中经常见到，地位类似于golang中的error。

### Status

Status常被用作函数的返回值，以此判断函数的执行结果。

```cpp
class Status{
    struct ErrorInfo {
        AtomicWord<unsigned> refs;     // reference counter
        const ErrorCodes::Error code;  // error code
        const std::string reason;      // description of error cause
        const std::shared_ptr<const ErrorExtraInfo> extra;

        static ErrorInfo* create(ErrorCodes::Error code,
                                 StringData reason,
                                 std::shared_ptr<const ErrorExtraInfo> extra);

        ErrorInfo(ErrorCodes::Error code, StringData reason, std::shared_ptr<const ErrorExtraInfo>);
    };

    ErrorInfo* _error;
}
```

Status只有一个成员变量，是一个ErrorInfo指针，当此指针为nullptr或者`_error->code`为0时，表示此status是ok的，没有错误产生。其他情况的Status是不OK的，表示有错误产生，具体错误信息在`_error`中。

`ErrorCodes::Error`是一个枚举类型，列出了所有可能的error code。reason是一个常量字符串。ErrorExtraInfo是附加信息，是一个抽象类可以先不管。Status之间是可以共享同一个ErrorInfo的，也就是`_error`指向同一个地址，refs记录了对此ErrorInfo实例的引用数。

### StatusWith

StatusWith是Status的加强版，相比Status，多携带了一个返回值。C++只允许函数有一个返回值，在很多时候很不方便，比如同时需要返回value和error。StatusWith就相当于golang中的这种写法：

```go
value, err := foo(bar);
// StatusWith<ValueType> s = foo(bar)
```

一般使用时，对于一个StatusWith实例s，先判断`s._status`是否是ok的，如果不ok就取出`s._status`进入错误处理流程，如果ok就取出`s._t`进入正常结果处理流程。

```cpp
template <typename T>
class StatusWith {
private:
    Status _status;
    boost::optional<T> _t;
};
```

## invariant、assert、exception

### invariant

invariant是一个宏，在代码中随处可见，用来确保运行时的一些“不变式”是成立的。

```cpp
#define invariant(...) BOOST_PP_OVERLOAD(MONGO_invariant_, __VA_ARGS__)(__VA_ARGS__)

#define BOOST_PP_OVERLOAD(prefix, ...) BOOST_PP_CAT(prefix, BOOST_PP_VARIADIC_SIZE(__VA_ARGS__))
```

可以看到invariant宏只是使用`BOOST_PP_OVERLOAD(MONGO_invariant_, __VA_ARGS__)`来“动态”生成一个函数名，然后调用它。下面看`BOOST_PP_CAT`和`BOOST_PP_VARIADIC_SIZE`两个宏

`BOOST_CAT`只是把两个token拼接起来：
```cpp
#define BOOST_PP_CAT(a, b) BOOST_PP_CAT_I(a, b)
#define BOOST_PP_CAT_I(a, b) a ## b
```

`BOOST_PP_VARIADIC_SIZE`用来获取可变参数列表的长度，它的实现比较巧妙，但是限制了可变参数列表最大长度为64。
```cpp
#define BOOST_PP_VARIADIC_SIZE(...) BOOST_PP_VARIADIC_SIZE_I(__VA_ARGS__, 64, 63, 62, 61, 60, 59, 58, 57, 56, 55, 54, 53, 52, 51, 50, 49, 48, 47, 46, 45, 44, 43, 42, 41, 40, 39, 38, 37, 36, 35, 34, 33, 32, 31, 30, 29, 28, 27, 26, 25, 24, 23, 22, 21, 20, 19, 18, 17, 16, 15, 14, 13, 12, 11, 10, 9, 8, 7, 6, 5, 4, 3, 2, 1,)
// 第65个参数是size
#define BOOST_PP_VARIADIC_SIZE_I(e0, e1, e2, e3, e4, e5, e6, e7, e8, e9, e10, e11, e12, e13, e14, e15, e16, e17, e18, e19, e20, e21, e22, e23, e24, e25, e26, e27, e28, e29, e30, e31, e32, e33, e34, e35, e36, e37, e38, e39, e40, e41, e42, e43, e44, e45, e46, e47, e48, e49, e50, e51, e52, e53, e54, e55, e56, e57, e58, e59, e60, e61, e62, e63, size, ...) size
```

所以`invariant(...)`会根据可变参数列表的长度来决定调用不同的函数，目前mongodb中只定义了`MONGO_invariant_1` 和`MONGO_invariant_2`。

`MONGO_invariant_1`接收一个参数，如果此参数为false、或者为0、或者为nullptr，就会触发abort。
```cpp
NOINLINE_DECL void invariantFailed(const char* expr, const char* file, unsigned line) noexcept {
    severe() << "Invariant failure " << expr << ' ' << file << ' ' << std::dec << line << std::endl;
    breakpoint();
    severe() << "\n\n***aborting after invariant() failure\n\n" << std::endl;
    std::abort();
}
```

`MONGO_invariant_2`接受两个参数，只是比`MONGO_invariant_1`多了一个msg，其他一样。

### assert

源码中有好几种assert，可以分为三类：
- fassert（fatal assert），fassertFailed直接终止程序运行，类似invariant
- uassert（user assert），test failed之后抛出异常
- massert（message assert），test failed也是抛出异常
- dassert（debug assert），如果当前是debug build，则调用invariant

### exception

mongodb中跟exception有关的类没几个，处理异常的地方也比较少，因为主要依靠前面提到的error code进行错误处理。

![](/assets/images/mongodb-exceptions.png)

其中最重要的一个就是DBException这个抽象类，它的代码如下所示，有且仅有一个成员`_status`，类型就是上面说过的Status。

```cpp
/** Most mongo exceptions inherit from this; this is commonly caught in most threads */
class DBException : public std::exception {
public:
    /**
     * Returns true if this DBException's code is a member of the given category.
     */
    template <ErrorCategory category>
    bool isA() const {
        return ErrorCodes::isA<category>(code());
    }
private:
    Status _status;
};
```

利用模板，能够在运行时捕获status为指定error code的异常。

## Decorable

Decorable是源码中一个比较有意思也比较复杂的机制。它提供了一个能力：可扩展地管理类之间一对多的从属关系。比如`ServiceContext`，一个`ServiceContext`实例就代表了一个mongod database service或者mongos routing service，它会有很多组件，比如logical clock、AuthorizationManager、replication coordinator等等。

这些组件都是独立的类，那么如何管理组件与ServiceContext的关系呢？一个比较简单的做法是使用composition（组合），也就是说logical clock、AuthorizationManager、replication coordinator这些组件作为`ServiceContext`的成员变量存在，要使用这些组件直接访问对应的成员变量就可以了。缺点就是每增删一个模块都要改动`ServiceContext`的代码，非常不方便，比如增加一个成员变量。

那么Decorable是如何做的呢？首先说明，这个Decorable跟装饰器模式没有关系，只是字面意思，*使其能够被装饰*。

还是以`ServiceContext`为例，它的定义如下，是一种非典型的[CRTP](https://en.wikipedia.org/wiki/Curiously_recurring_template_pattern)应用：

```cpp
class ServiceContext final : public Decorable<ServiceContext> {
}
```

![](/assets/images/mongodb-decorable-class.png)

从上面的类图中可以看出，Decorable类只有一个成员变量，且是私有的，只有一个公开方法，且是static的，还有一个内嵌类Decoration。

```cpp
template <typename D>
class Decorable {
private:
    DecorationContainer<D> _decorations;
public:
    template <typename T>
    static Decoration<T> declareDecoration() {}
    
    template <typename T>
    class Decoration {}
}
```

### Decorable、Decoration、D、T之间的关系

以下面代码中的Service和AuthorizationManager为例，结合上面的代码，可以看出：D是ServiceContext类，T是`std::unique_ptr<AuthorizationManager>`。

```cpp
class ServiceContext final : public Decorable<ServiceContext> {
}

const auto getAuthorizationManager = ServiceContext::declareDecoration<std::unique_ptr<AuthorizationManager>>();
```

也就是说，通俗一点讲，继承自`Decorable<D>`类的D有了“被装饰”的特性，它可以被加上任意类型T的“装饰品”。Decorable类提供了这种能力——使一个类变得“可被装饰”。

Decoration表示一个装饰品，通过指定T的类型，将其实例化为指定的装饰品。后面的讲述中将交替使用“装饰品”和“组件”这两个词。

### 如何使用Decorable这套机制

还是以ServiceContext和AuthorizationManager为例，使用起来也就两步：

1. 使用ServiceContext的declareDecoration静态方法来添加一个装饰品
	- 要在ServiceContext实例化之前完成所有的declareDecoration。mongodb代码中一般调用declareDecoration并将返回结果赋给一个全局变量
2. 实例化一个ServiceContext，所有的装饰品会自动被构造
3. 使用这个全局变量，通过被重载的括号运算符来取出绑定在一个ServiceContext实例上的AuthorizationManager实例。

```cpp
const auto getAuthorizationManager = ServiceContext::declareDecoration<std::unique_ptr<AuthorizationManager>>(); // 1

AuthorizationManager* AuthorizationManager::get(ServiceContext* service) {
    return getAuthorizationManager(service).get();
} // 3
```

### 添加一个装饰品的内部实现

Decorable的declareDecoration静态方法是用来添加一个装饰品的唯一外部接口。declareDecoration之后，new出来的ServiceContext实例都会有各自的AuthorizationManager。

那么这是如何实现的呢？简单来说，每个ServiceContext实例都有一个byte类型的数组（继承自Decorable类），这个数组是用来存放各个组件的（通过placement new）。反之各个组件都在这个数组中有各自的位置（也就是数组下标），通过这个位置能够取出对应的组件。同时为了能够“灵活地”增删组件（相比于硬编码一个Decorable都有哪些组件来说），引入了Registry这个第三方，专门用来注册组件，同时做预分配。

下面进行剖析，首先看Decorable的declareDecoration方法，只是获取一个静态的registry实例，然后调用它的declareDecoration方法：
```cpp
    template <typename T>
    static Decoration<T> declareDecoration() {
        return Decoration<T>(getRegistry()->template declareDecoration<T>());
    }

    static DecorationRegistry<D>* getRegistry() {
        static DecorationRegistry<D>* theRegistry = new DecorationRegistry<D>();
        return theRegistry;
    }
```

注意，对每个模板类D，编译器都会生成一个特例getRegistry函数，这里的theRegistry局部静态变量是一个“模板静态变量”，且被`DecorationRegistry<D>`的所有实例共享。比如ServiceContext的所有实例共享同一个theRegistry。

下面看DecorationRegistry的declareDecoration方法：
```cpp
template <typename DecoratedType>
class DecorationRegistry {
public:
    void construct(DecorationContainer<DecoratedType>* const container) const {
        using std::cbegin;
        auto iter = cbegin(_decorationInfo);
        using std::cend;
        for (; iter != cend(_decorationInfo); ++iter) {
            iter->constructor(container->getDecoration(iter->descriptor));
        }
    }
private:
    // 使用placement new来控制装饰品的位置
    template <typename T>
    static void constructAt(void* location) {
        new (location) T();
    }

    template <typename T>
    static void destroyAt(void* location) {
        static_cast<T*>(location)->~T();
    }
    // 记录一个组件的构造函数、析构函数和被分配的位置
    struct DecorationInfo {
        DecorationInfo() {}
        DecorationInfo(
            DecorationConstructorFn inConstructor,
            DecorationDestructorFn inDestructor)
            : descriptor(std::move(inDescriptor)),
              constructor(std::move(inConstructor)),
              destructor(std::move(inDestructor)) {}

        size_t _index;
        DecorationConstructorFn constructor;
        DecorationDestructorFn destructor;
    };

    typename DecorationContainer<DecoratedType>::DecorationDescriptor declareDecoration(
        const size_t sizeBytes,
        const size_t alignBytes,
        const DecorationConstructorFn constructor,
        const DecorationDestructorFn destructor) {
        const size_t misalignment = _totalSizeBytes % alignBytes;
        if (misalignment) {
            _totalSizeBytes += alignBytes - misalignment;
        }
        typename DecorationContainer<DecoratedType>::DecorationDescriptor result(_totalSizeBytes);   // 给一个组件分配位置
        _decorationInfo.push_back(DecorationInfo(result, constructor, destructor));
        _totalSizeBytes += sizeBytes;  // 递增需要的空间大小
        return result;
    }

    std::vector<DecorationInfo> _decorationInfo;   // 所有的组件信息
    size_t _totalSizeBytes{sizeof(void*)};     // 实例化所有组件需要的空间
};
```

所有的组件都通过registry进行注册，并给各个组件分配了一个位置。这个过程是在main函数被调用之前完成的，也就是所有的Decorable类实例化之前。Decorable类实例化的时候会去读取对应的registry中的内容，并按照注册时预分配的位置初始化各个组件。这个过程如下：
- 实例化Decorable时，会构造`_decorations`
- DecorationContainer的构造函数会调用`_registry`的construct方法
- construct方法会依次使用placement new在`_decorations`的`_decorationData`中构造已经注册的各个组件

```cpp
template <typename DecoratedType>
class DecorationContainer {
public:
    explicit DecorationContainer(Decorable<DecoratedType>* const decorated,
                                 const DecorationRegistry<DecoratedType>* const registry)
        : _registry(registry),
          _decorationData(new unsigned char[registry->getDecorationBufferSizeBytes()]) {
        _registry->construct(this);
    }

    ~DecorationContainer() {
        _registry->destroy(this);
    }

    /**
     * Gets the decorated value for the given descriptor.
     *
     * The descriptor must be one returned from this DecorationContainer's associated _registry.
     */
    void* getDecoration(DecorationDescriptor descriptor) {
        return _decorationData.get() + descriptor._index;
    }

private:
    const DecorationRegistry<DecoratedType>* const _registry;
    const std::unique_ptr<unsigned char[]> _decorationData;
};
```

### 取出一个装饰品的内部实现

明白了如何添加一个装饰品，那么取出一个装饰品就很清楚了，就是通过`operator()`方法

`operator()`方法根据索引获取一个`void*`指针并返回。

```cpp
template <typename T>     // T是“装饰品”，例如ServiceContext的认证管理组件
class Decoration {       
public:
	// 重载了双括号，用来从实例D中取出实例T，比如从一个ServiceContext实例中取出它的AuthorizationManager
	T& operator()(D& d) const {
		return static_cast<Decorable&>(d)._decorations.getDecoration(this->_raw);
	}

	// 获取实例T所属的D实例
	D* owner(T* const t) const {
		return static_cast<D*>(getOwnerImpl(t));
	}

private:
	// 这里利用了对象的内存布局手动强制获取Decorable实例。Decorable没有虚函数，且只有一个成员变量
	const Decorable* getOwnerImpl(const T* const t) const {
		return *reinterpret_cast<const Decorable* const*>(
			reinterpret_cast<const unsigned char* const>(t) - _raw._raw._index);
	}

	friend class Decorable;

	explicit Decoration(
		typename DecorationContainer<D>::template DecorationDescriptorWithType<T> raw) : _raw(std::move(raw)) {}

	typename DecorationContainer<D>::template DecorationDescriptorWithType<T> _raw;         // 此装饰品的位置/描述符
};

template <typename DecoratedType>
class DecorationContainer {
public:
    void* getDecoration(DecorationDescriptor descriptor) {
        return _decorationData.get() + descriptor._index;
    }
private:
    const DecorationRegistry<DecoratedType>* const _registry;
    const std::unique_ptr<unsigned char[]> _decorationData;
};
```


## TransportLayer、ServiceExecutor、ServiceEntryPoint

作为一个服务端软件，最先考虑到的就是如何和客户端进行通信，比如建连接、读写数据。在mongodb中，这个过程还是很复杂的，主要是为了用有限的资源提供尽量高的性能。TransportLayer就是负责这方面的一个类，它同时与ServiceExecutor、ServiceEntryPoint协同工作，完成整个mongo实例的会话管理、消息收发。

粗略地讲：
- TransportLayer 主要负责监听端口，建立与客户端的连接
    - 目前仅支持基于asio的TransportLayerASIO
- ServiceExecutor 异步调度执行消息的收发和处理
    - ServiceExecutorSynchronous：模拟了a thread per connection。默认
    - ServiceExecutorAdaptive：使用线程池的方法处理所有的connection
- ServiceEntryPoint 管理客户端会话，处理客户端请求
    - ServiceEntryPointImpl 
        - ServiceEntryPointMongod
        - ServiceEntryPointMongos


### 三者的使用

在mongod启动时会设置并启动ServiceExecutor、ServiceEntryPoint和TransportLayer，如下面代码所示。

```cpp
ExitCode _initAndListen(int listenPort) {
    ...
    // 设置ServiceEntryPoint
    serviceContext->setServiceEntryPoint(stdx::make_unique<ServiceEntryPointMongod>(serviceContext));

    if (!storageGlobalParams.repair) {
        auto tl = transport::TransportLayerManager::createWithConfig(&serverGlobalParams, serviceContext);
        // TransportLayer初始化
        auto res = tl->setup();
        if (!res.isOK()) {
            error() << "Failed to set up listener: " << res;
            return EXIT_NET_ERROR;
        }
        // 设置TransportLayer
        serviceContext->setTransportLayer(std::move(tl));
    }
    ...
    
    // 依次启动三者
    auto start = serviceContext->getServiceExecutor()->start();
    start = serviceContext->getServiceEntryPoint()->start();
    start = serviceContext->getTransportLayer()->start();
}

std::unique_ptr<TransportLayer> TransportLayerManager::createWithConfig(
    const ServerGlobalParams* config, ServiceContext* ctx) {
    auto sep = ctx->getServiceEntryPoint();
    auto transportLayerASIO = stdx::make_unique<transport::TransportLayerASIO>(opts, sep);
    ...

    // 设置ServiceExecutor
    if (config->serviceExecutor == "adaptive") {
        auto reactor = transportLayerASIO->getReactor(TransportLayer::kIngress);
        ctx->setServiceExecutor(stdx::make_unique<ServiceExecutorAdaptive>(ctx, std::move(reactor)));
    } else if (config->serviceExecutor == "synchronous") {
        ctx->setServiceExecutor(stdx::make_unique<ServiceExecutorSynchronous>(ctx));
    }
    ...
}
```

三者之间的关系比较密切：
- TransportLayer会接受新连接，然后将连接交给serviceExecutor
- serviceExecutor负责读写任务和请求处理的调度，会调用ServiceEntryPoint的接口来处理请求
- ServiceEntryPoint处理客户端发来的请求

下面进行具体分析

### TransportLayer

TransportLayer是一个抽象类，TransportLayerASIO是它的唯一实现，代码如下。

`ASIOReactor`其实只是一个asio中的`io_context`，如果不了解asio，可以把`ASIOReactor`看作一个epoll。TransportLayerASIO里面有三个Reactor，分别监听三类socket连接。其中，`_ingressReactor`只在ServiceExecutor为adaptive模式下才使用到。

```cpp
class TransportLayerASIO final : public TransportLayer {
public:
    Status setup() final;
    Status start() final;
    
private:
    // _ingressReactor 监听所有已经accepted的sockets连接
    std::shared_ptr<ASIOReactor> _ingressReactor;
    // _egressReactor 监听本实例作为客户端的sockets连接
    std::shared_ptr<ASIOReactor> _egressReactor;
    // _acceptorReactor 监听_acceptors中的所有sockets连接
    std::shared_ptr<ASIOReactor> _acceptorReactor;
    std::vector<std::pair<SockAddr, GenericAcceptor>> _acceptors;
    
    void _acceptConnection(GenericAcceptor& acceptor);
    void _runListener() noexcept;
    
    struct Listener {
        stdx::thread thread;
        stdx::condition_variable cv;
        bool active = false;
    };
    Listener _listener;
    
    ServiceEntryPoint* const _sep = nullptr;
}
```

#### 接收连接

TransportLayerASIO最主要的职责就是监听端口，建立新连接，具体看代码的时候可以先看setup函数再看start函数。setup函数初始化所有的acceptors；start函数启动一个名为listener的线程，执行_runListener函数。

```cpp
Status TransportLayerASIO::setup() {
    ...
    // endpoints 是要监听的地址，默认是127.0.0.1：27017
    for (auto& addr : endpoints) {
        // 将acceptor绑定在_acceptorReactor中
        GenericAcceptor acceptor(*_acceptorReactor);
        // 绑定socket地址
        acceptor.bind(*addr, ec);
        _acceptors.emplace_back(SockAddr(addr->data(), addr->size()), std::move(acceptor));
    }
    ...
}

void TransportLayerASIO::_runListener() noexcept {
    auto acceptCb = [this, &acceptor](const std::error_code& ec, GenericSocket peerSocket) mutable {
        if (ec) {
            log() << "Error accepting new connection on "
                  << endpointToHostAndPort(acceptor.local_endpoint()) << ": " << ec.message();
            _acceptConnection(acceptor);
            return;
        }

        try {
            // 接收到新连接之后，新建并启动一个session
            std::shared_ptr<ASIOSession> session(new ASIOSession(this, std::move(peerSocket), true));
            // _sep为ServiceEntryPointMongod
            _sep->startSession(std::move(session));
        } catch (const DBException& e) {
            warning() << "Error accepting new connection " << e;
        }

        _acceptConnection(acceptor);
    };
    
    for (auto& acceptor : _acceptors) {
        // 设置回调。新连接建立之后会调用acceptCb，并将socket连接加入到_ingressReactor
        acceptor.second.async_accept(*_ingressReactor, std::move(acceptCb));
    }
    
    while (!_isShutdown) {
        // io_context执行accept操作
        _acceptorReactor->run();
    }
}
```

注意，startSession方法是由listener线程执行的。

### ServiceEntryPoint

startSession方法是在ServiceEntryPointImpl中实现的，会创建一个ServiceStateMachine，然后调用它的start方法，start方法只是去调用它的`_scheduleNextWithGuard`方法。

```cpp
void ServiceStateMachine::_scheduleNextWithGuard(ThreadGuard guard,
                                                 transport::ServiceExecutor::ScheduleFlags flags,
                                                 transport::ServiceExecutorTaskName taskName,
                                                 Ownership ownershipModel) {
    auto func = [ssm = shared_from_this(), ownershipModel] {
        ThreadGuard guard(ssm.get());
        if (ownershipModel == Ownership::kStatic)
            guard.markStaticOwnership();
        ssm->_runNextInGuard(std::move(guard));
    };
    // 调度执行_runNextInGuard方法
    // _serviceExecutor默认是ServiceExecutorSynchronous实例，后面会介绍
    Status status = _serviceExecutor->schedule(std::move(func), flags, taskName);
    ...
}

void ServiceStateMachine::_runNextInGuard(ThreadGuard guard) {
    ...
    switch (curState) {
        case State::Source:
            // 接收消息
            _sourceMessage(std::move(guard));
            break;
        case State::Process:
            // 处理消息，然后给客户端答复
            _processMessage(std::move(guard));
            break;
        case State::EndSession:
            // 清理session
            _cleanupSession(std::move(guard));
            break;
        default:
            MONGO_UNREACHABLE;
    }
    ...
}
```

`_processMessage`中会调用ServiceEntryPointMongod中的`handleRequest`来处理请求。

`_sourceMessage`、`_processMessage`、`_sinkMessage`等函数结尾处都会再次调用`_scheduleNextWithGuard`或者`_runNextInGuard`函数来维持一个客户端的IO loop。

### ServiceExecutor

像前面所说，ServiceExecutor是一个总的调度器，负责消息的收取、回复和处理。

mongodb支持两种ServiceExecutor：
- ServiceExecutorAdaptive：使用一个worker pool来执行任务，根据负载调整线程数
- ServiceExecutorSynchronous：一个connection占有一个线程

#### ServiceExecutorAdaptive

ServiceExecutorAdaptive的start方法启动一个名为worker-controller的线程，和默认为cpu核数一半的名为worker-xxxx的工作线程。

worker-controller线程主要用来控制worker pool中的线程数量，当线程池中线程不够用、线程卡死或者线程数低于最小值就创建新线程。

worker线程一直调用`_reactorHandle`的runFor方法，来等待事件发生并调用callback。

在前面的createWithConfig函数中可以看到，当设置ServiceExecutor为adaptive时，这个`_reactorHandle`其实就是transportLayerASIO中的`_ingressReactor`。在说transportLayerASIO的时候也说了，所有已经接收的socket连接都会被放到`_ingressReactor`中。

listener线程接收到新连接之后，会创建一个session，然后调用ServiceExecutorAdaptive的schedule方法。schedule方法会调度这个客户端的相关动作，如上面代码中`ServiceStateMachine::_scheduleNextWithGuard()`函数所示。


#### ServiceExecutorSynchronous

ServiceExecutorSynchronous 的start函数什么也没做。

只用看schedule函数，schedule函数比较简单，只是判断`_localWorkQueue`变量是否为空，如果为空就创建一个线程，这个线程运行的任务就是`ServiceStateMachine::_scheduleNextWithGuard()`中schedule的那个func。因为`_localWorkQueue`是thread local变量，而运行schedule函数的是listener线程，listener线程的`_localWorkQueue`一定是空的，所以会创建新线程。

从接收连接到创建线程都是在listener线程中执行，整个调用栈如下：
```
TransportLayerASIO::_runListener()
    ServiceEntryPointImpl::startSession()
        ServiceStateMachine::start()
            ServiceStateMachine::_scheduleNextWithGuard()
                ServiceExecutorSynchronous::schedule()
```



