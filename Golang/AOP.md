AOP(Aspect Oriented Programming) 面向切面编程。核心在于将横向关注点从业务中剥离出来。

横向关注点： 就是那些跟你业务没啥关系，但是每个业务又必须要处理的。常见的有几类：

可观测性：logging， metric， tracing

安全相关：登录、鉴权与权限控制

错误处理：例如错误页面支持，比如404 要重定向到一个404的页面

可用性保证：熔断限流和降级等



AOP: 用闭包来实现责任链
主要讲的是：不侵入web框架核心，不侵入业务，
也叫：middleware， interceptor, 本质是一样的

AOP 切面编程，横向关注点(也就是所有的接口都需要处理的)，一般用来解决Log, tracing, metric，熔断，限流，跨域CORS，权限校验等等

作用：我们希望请求在真正被处理之前能够经过一大堆的filter
```
+---------+            +---------+             +---------+
| filter1 | -request-> | filter2 |  -request-> | filter3 |
+---------+            +---------+             +---------+
												 request
													 |
							                    +----v----+
							                    | 业务代码 |
							                    +---------+
												  response
                                                     |
+---------+            +---------+              +----v----+
| filter6 | <-response-| filter5 |  <-response- | filter4 |
+---------+            +---------+              +---------+
```
**核心点：**

-  定义Filter的时候输入和输出参数都为同一个参数
```go
type FilterBuilder func(next Filter) Filter
type Filter func(c *Context)
```
-  在和真正业务逻辑组合起来的时候是先从最后添加的filter开始的，因为要把最先添加的放在最外层，才能最先执行，当然，真正业务逻辑代码肯定是要最先组合起来。**要组合成只有一个函数**
```go
for i := len(builder) - 1; i >= 0; i-- {
    b := builder[i]
    root = b(root)
}
```
#### 简单实现

```go
type FilterBuilder func(next Filter) Filter
type Filter func(c *Context)

type Context struct {
}

func do(builder ...FilterBuilder) {
   var root Filter = func(c *Context) {
      // 真正处理的内容
      fmt.Println("这是我真正处理的内容。")
   }

   // 和真正业务逻辑组合起来，组合成一个函数，把最后添加的filter包在最里面，当然最后执行的是我们的业务代码也就是上面的root
   for i := len(builder) - 1; i >= 0; i-- {
      b := builder[i]
      root = b(root)
   }
   root(&Context{})
}

func MetricsFilterBuilder(next Filter) Filter {
   return func(c *Context) {
      start := time.Now()
      fmt.Println("start time")
      next(c)
      fmt.Println("total time:", start.Sub(start))
   }
}

func main() {
   do(MetricsFilterBuilder)
}
```
```go
// filter.go
// 输入和输出参数都一样
type FilterBuilder func(next Filter) Filter
type Filter func(c *Context)


// server.go

type sdkHttpHandle struct {
	Name    string
	handler Handler
	root    Filter
}

func newSdkHttpHandle(name string, builder ...FilterBuilder) *sdkHttpHandle {
	var root Filter = func(c *Context) {
		// http真正处理的内容
		ServeHTTP(c)
	}

	for i := len(builder) - 1; i > 0; i-- {
		b := builder[i]
		root = b(root)
	}
	return &sdkHttpHandle{
		Name:    name,
		root:    root,
	}
}

func (s *sdkHttpHandle) Start(address string) error {
	http.HandleFunc("/", func(writer http.ResponseWriter, request *http.Request) {
		ctx := NewContext(writer, request)
		s.root(ctx)
	})
	return http.ListenAndServe(address, nil)
}

// 运行
func MetricsFilterBuilder(next Filter) Filter {
	return func(c *Context) {
		start := time.Now()
		next(c)
		fmt.Println(start.Sub(start))
	}
}
srv := newSdkHttpHandle("test", MetricsFilterBuilder)
srv.Start(":8080")
```
### 面试要点
- **什么是AOP ?** AOP就是面向切面编程，用于解决横向关注点问题，如可观测性问题、安全问题等。
- **什么是洋葱模式?**形如洋葱，拥有一个核心，这个核心一般就是业务逻辑。而后在这个核心外面层层包裹，每一层就是一个Middleware。一般用洋葱模式来**无侵入式**地增强核心功能，或者解决AOP问题。
- **什么是责任链模式? **不同的Handler组成一条链，链条上的每一环都有自己功能。一方面可以用责任链模式将复杂逻辑分成链条上的不同步骤，另外一方面也可以灵活地在链条上添加新的 Handelr。
- **怎么实现?** 最简单的方案就是我们课程上讲的这种函数式方案，还有一种是集中调度的模式。



### 例子

#### Tracing

Tracing:  踪迹，它记录从收到请求到返回响应的整个过程。在分布式环境下，一般代表着请求从Web收到，沿着后续微服务链条传递，得到响应再返回到前端的过程。

![](.\images\aop\1.png)


#### Tracing —链路追踪
上边是一个简单的例子。
需要注意的是，trace不仅仅包含服务调用信息，实际上整条链路上的信息都可能包含在内，包括:
- RPC调用
- HTTP 调用
- 数据库查询
- 丢消息(把消息丢到mq)
- 业务步骤
这些步骤，也可以进一步被细分，分成更加细的span。
![](.\images\aop\2.png)

这里有两个空隙，如果一个空隙很长的话，那么说明打点不够详细。

Tracing相关的几个概念:

- tracer:表示用来记录 trace（踪迹)的实例,一般来说，tracer 会有一个对应的接口。
- span:代表trace中的一段。因此trace本身也可以看做是一个span。span 本身是一个层级概念，因此有父子关系。**一个trace 的 span可以看做是多叉树。**





#### Tracing—用什么tracing工具
目前在业界里面,tracing工具还处于百花齐放阶段，有很多开源实现。例如 SkyWalking、Zipkin、Jeager等。

那么我们该怎么支持这些开源框架?
![](.\images\aop\3.png)
接下来我们来实现第二种，只写一个TracintMiddleware，我们要把TracintMiddleware 抽象出来，然后注入对应的tracing工具
一般来说，如果一个中间件设计者想要摆脱对任何第三方的依赖，**都会定义自己的 API**，常见有:
- 定义LOg API
- 定义Config API
- 定义Tracing API
- 定义Metrics API
![](.\images\aop\4.png)
框架操作的只是我的实现api，我要求你注入实现
这种设计的缺点:
- 过度设计:有些时候这些API只有一个默认实现，而且也没有人愿意提供扩展实现
- API设计得并不咋样
如果自己设计不好API，就别用这种设计方案

那么我们采用下面这种方案：
#### Tracing ——用OpenTelemetry API
虽然我也想定义APl，但是我定义的 API肯定不如OpenTelemetry  APl，所以我们采用OpenTelemetry API作为抽象层。
那么所有支持OpenTelemetry API的框架，用户都可以使用。
![](.\images\aop\5.png)
OpenTelemetry简介
OpenTelemetry是 OpenTracing和 OpenCensus合并而来。
- OpenTelemetry同时支持了logging.tracing和metrics
- OpenTelemetry 提供了各种语言的SDK
- OpenTelemetry 适配了各种开源框架，如Zipkin、Jeager、Prometheus
新时代的可观测性平台。
![](.\images\aop\6.png)

##### OpenTelemetry GO SDK入门
- TracerProvider: 用于构造tracer 实例。
Provider 在平时开发中也很常见，有时候可以看作是轻量级的工厂模式。
- tracer: 追踪者，也就是用于构造 trace的东西·构造tracer 需要一个
instrumentationName，一般来说就是你构造tracer的地方的包名（其实主要保证唯一就好了)
- span: 调用tracer上的Start方法。如果传入的context里面已经有一个span 了，那么新创建的span 就是老的span的儿子。span要记住调用End。
![](.\images\aop\7.png)
![](.\images\aop\8.png)
其实这就是适配器的应用



### Metrics—- Prometheus
Prometheus Metrics类型:
- Counter: 计数器，统计次数，比如说某件事发生了多少次
- Gauge: 度量，它可以增加也可以减少，比如说当前正在处理的请求数
- Histogram: 柱状图，对观察对象进行采样,然后分到一个个桶里面
- Summary:采样点按照百分位进行统计，比如99线、999线等
- 	99线，99%的请求会在这个响应时间段内给你返回，
- 	999线，99.9%的请求会在这个响应时间段内给你返回
Prometheus一般是和Grafana结合使用，Grafana作为数据展示平台。
![](.\images\aop\9.png)

#### Prometheus Verctor 用法
创建一个Vector（向量)，设置ConstLabels和Labels
- 使用WithLabelValues来获得具体的收集器
这种用法会更加普遍。
![](.\images\aop\10.png)
