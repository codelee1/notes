# go-kit

### go-kit简介

go-kit是一个由一系列Go(golang)的包/资源库组成的工具集，可以让我们构建出健壮、可靠、可维护的微服务。

go-kit的最初构思者：Peter Bourgon是这样定义go-kit的:
    
    Go + Microservices = Go kit
    
其中起源思想参见：https://peter.bourgon.org/applied-go-kit    

### go-kit架构
go-kit架构主要分为三层，由外到内分别是：Transports、Endpoints和Services。

Transports：通信层，在这里可以采用不同的通信方式，如HTTP协议的REST接口、gRPC协议的RPC接口，这是很灵活和强大的，可以方便的切换成不同的通信协议。

Endpoints：终端层，类似于Controller，里面实现了各种接口的Handler，负责req/resp格式的转换，同时也是被人吐糟繁杂的原因所在。

Services：服务层，即实现业务的地方。

从架构图来看，简单明了。需要注意的是除了上述的主要三层结构外，还加入了不同的中间件，主要在Endpoints和Service中实现。大多中间件其实是业务无关的，
比如记录日志、访问限制频率、负载均衡分布式追踪等。

![images](https://gokit.io/faq/onion.png "onion")

### 模块组件间的关系-依赖注入

go-kit鼓励我们的服务应该设计可相互的组件，尽管是单一职责的中间件也应如此。
在main函数中实现所有模块的组装，组件之间的依赖需要通过参数传入。

1、通过此方法严格保持组件生命周期处于main域中，可以避免使用全局变量作为捷径，这对可测试性至关重要。

2、在作用域为main的组件中，则将它们作为作为其他组件的依赖项提供的唯一方法就是将它们显式的传递个构造函数，这使依赖关系保持明确，从而在启动之前就消除了很多技术债务。


例如，有如下组件：

    Logger
    TodoService， 实现 Service 接口
    LoggingMiddleware， 实现 Service 接口，依赖 Logger 和 TodoService 的实现
    Endpoints，依赖一个实现了 Service 的组件
HTTP (transport)，依赖Endpoints

我们的主方法应该这样写：

    logger = log.NewLogger(...)
    var service todo.Service // interface
    service = todo.NewService  // 具体实现
    service = todo.NewLoggerMiddleware(logger)(service)
    
    endpoints := todo.NewEndpoints(service)
    transport := todo.NewHTTPTransport(endpoints)


### Why go-kit
 
严格来说，go-kit不是一个框架，而是一个工具集（toolkit），它不会像其微服务他框架，如micro加入太多限制，因为框架本身就包含了设计者的思想与理念。

而go-kit则是一个框架的底层，如同简介中：Micro wants to be a platform; Go kit, in contrast, wants to integrate into your platform.

框架总是在尝试做适应任何人的业务平台，而不是专门做适应你自己的平台，相反，我们可以用go-kit来做适应自己平台的框架，相比来说灵活很多。

放眼望去，其实没有适应所有业务的框架，因为每个业务都是在变化的，这也是为什么公司大了，就会弄自己的框架，其真正的原因很有可能当前框架已经不能很好满足公司的业务发展需求了。

go-kit还有一个优点，如果要更换成其他框架，在go-kit良好的基础上，我们只需要把Transports层和Endpoints层剥离，留下Services就可以很方便的集成到新的框架下面了。


### go-kit缺点

1、框架太繁琐，每个接口的代码太多，太啰嗦；
（利用工具生成代码）

2、interface{}API太蛋疼，在Endpoint层，每个Endpoint都需要重复类似的转换代码。
（同1，而且可以将公共代码抽离出来）



