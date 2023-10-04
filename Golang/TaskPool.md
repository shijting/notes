https://github.com/ecodeclub/ekit/issues/175
https://github.com/ecodeclub/ekit/pull/80
## 使用场景

在有些时候，我们需要控制住 goroutine 的数量。例如在实际应用中，我们可能有快慢两种请求。对于慢请求来说，如果我们不限制它所能占据的 goroutine 数量，那么慢请求占据了一个 goroutine 之后一直不会释放，那么就会导致越来越多的 goroutine 被慢任务占据。

极端情况下，可能超过一半的 goroutine 都被慢任务所占据。这部分 goroutine 一直占据着资源在缓慢运行。

因此我们可能希望引入一种隔离机制，将慢任务和快任务分离，让慢任务在数量有限的 goroutine 上运行，而其它快任务则没有这种限制。

例如说我们在做数据迁移的时候，开启 10 个 goroutine 并发迁移不同表的数据。

因此我们需要设计一个任务池，用户提交任务，并且可以控制住并发执行任务的 goroutine 数量。

## 行业分析

> 如果你知道有框架提供了类似功能，可以在这里描述，并且给出文档或者例子

这方面的典型设计，就是 Java 的 [ExecutorService](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ExecutorService.html)，也就是 Java 中线程池的公共接口，它的方法可以分成好几个部分：







## 可行方案

### TaskPool 接口定义

接口定义只需要描述 TaskPool 所要遵循的规范，查看 [TaskPool](https://github.com/gotomicro/ekit/blob/dev/pool/task_pool.go)

每一个方法的设计理由是：

- Submit 要考虑一个额外的阻塞因素。即如果任务池本身有容量的限制，那么用户在提交任务的时候可以被阻塞一会，在阻塞的这段时间，或许能够提交任务成功，否则就返回错误。影响 Submit 的还有 Start 方法和 Shutdown, ShutdownNow 两个方法。对于前者来说，实现者可以自由决定 Start 之后还是否允许继续提交任务——这本质上是一个线程安全的问题；Shutdown 的两个方法，则明确在调用之后就不再允许提交任务了；
- Shutdown 和 ShutdownNow：Shutdown 返回了一个 channel，那么用户监听这个 channel，就可以得知所有任务什么时候被执行完毕，同时自己也可以控制等待所有任务执行完毕的时间。而 ShutdownNow 则可以立刻关闭线程池，并且返回所有剩下未执行的任务
- Start 启动任务：用户可以在更加确切的地方开始调度任务的执行



目的: 队列中不会有太多等待任务，goroutine也不会有太多



问：
我举个例子,看看我是否正确理解了你的需求, 以及需求不明确的地方.

初始=10, 核心 = 20, 最大=30
Start后立即创建10个协程 —— 类似“固定创建”模式
当发现此时len(b.queue) > X, X范围[0, queueSize)且当前协程数 < 核心协程数即20, 创建一个协程, 类似“按需创建“模式
当发现此时当前协程数>=20且<30且len(b.queue) > Y, Y范围(0, queueSize). 创建一个协程, 类似“按需创建“模式
当前协程数 = 最大协程数即30后,不再创建任何协程
X和Y的具体值是多少?

当协程数在(10, 20] 和 (20, 30]范围内时, 这些按需创建出的协程是常驻的与初始创建那10个等效,还是完成任务后自动退出?虽然你说核心协程数：这个我在想可能不需要关掉协程。如果需要关掉的但上面也提到用户指定“空闲时间”让协程自动退出.假设现在有了这个空闲时间,协程号在(10, 30]按需创建出来的协程需要监听空闲时间,如果达到就自动退出吗?

空闲时间是否要有一个检查,至少为多少吗?传0, -1等非法值在创建TaskPool阶段就报错吗?还是不管

当参数关系不合理时如何处理? 比如:初始>核心=最大; 初始>核心>最大; 初始>最大>核心; 核心>最大>初始; 等
一刀切,只要不符合初始<=核心<=最大 这个关系,就按初始=核心=最大处理 还是分情况处理?
初始 ==核心==最大, 我觉得合法, 相当于现有实现的“固定个数”模式
初始<=核心==最大, 我觉得合法, 相当于现有实现的“按需创建”模式

是否允许在TaskPool运行中,动态地改变核心协程数及最大协程数?还是创建后不可改!

假设上面的需求已经实现,现在还有必要分出两个TaskPool实现吗?
我觉得只需要一个TaskPool实现,让用户通过控制初始,核心,最大,空闲时间等参数就可以模拟出已实现的“按需创建”“固定个数”两个模式

使用Option还是公开字段?Option可以不破坏现有API(可选参数),但需要提供默认值,初始,核心,最大,空闲时间的默认值是多少?
初始 ==核心==最大, 固定个数模式
初始<=核心==最大, 初始<=核心<=最大, 固定个数+按需创建
空闲时间默认值为多少合适呢?1h?

**回复：**

目的：队列中不会有太多等待任务，goroutine也不会有太多

我思考了一下怎么控制这个东西，我觉得类比 Java 线程池和 go 连接池两个设计，能取得一个很不错的效果。我来描述一下这个过程，也就是 10， 20， 30 这个例子：

- 最开始的时候创建 10 个协程；
- 用户提交任务。如果此时 10 个协程都在繁忙中，那么用户提交任务，就直接开启一个 goroutine，直到达到 20（核心数）；
- 如果这个时候用户还继续提交任务，那么就会放到队列中，接下来我们会采用一个算法来判断要不要新建一个 goroutine:
- - 算法1：队列满了，我们直接创建一个 goroutine，直到达到 30
- - 算法2：队列满一半了，以 size/cap 的概率创建一个新的 goroutine
- - 算法3：直接创建新 goroutine，直到达到 30
- 当 goroutine 数量在 20-30 的时候，一个 Goroutine 发现队列已经空了，那么它会直接退出，不需要等待空闲时间；
- 当 goroutine 降低到 10-20 以下的时候，我们超过最大空闲时间，就关掉 goroutine
- 当 goroutine 降低到 10 的时候，我们将不会关闭，即便一直空闲

这里面，有些地方是可以进一步讨论的：

- 队列满一半了，以 size/cap 的概率创建一个新的 goroutine。这个我们甚至可以固定一个比较低的概率，比如说 0.3， 0.4 什么的。理论上来说，我们可以通过压测来观测队列中等待任务数量，和最终运行的 goroutine 数量来取一个概率，使得队列中不会有太多等待任务，goroutine 也不会有太多。注意，假如说我们的概率是 p，那么连续两次都没有触发创建 goroutine 的概率是 (1-p)^2
- goroutine 降低到 10 的时候，可以一直保留，因为少量 goroutine 阻塞之后，只是占据了一点点内存。也可以达到最大空闲时间，就关掉；
- 参数问题，你可以自由决定校验还是不校验，以及是否兼容；
- - 暂时不允许。从业务角度来说，用户可能希望能够监控队列，如果队列满了，他可能希望在运行期间动态调整。但是这个的前提是我们暴露了观测接口
- - 你说得对，没必要，合并为一个就可以了
- - 采用 Option 模式，默认值你可以根据自己的经验来设置，这方面其实我也没特别好的建议——我也是第一次设计这个。

**关于被创建出的协程的分类及退出策略的总结:**
- 在(0, 初始]即(0, 10]区间的协程为同一类,退出时机受Shutdown和ShutdownNow方法影响.
- 在(初始, 核心]即(10, 20]区间的协程为同一类,退出时机受Shutdown和ShutdownNow方法、“最大空闲时间“参数、”队列中是否有任务“的影响.
完成任务后,如果队列为空则等待“最大空闲时间”后退出,如果等待期间拿到任务,停掉计时器->执行任务,结束后—>队列不空,取任务运行;队列为空,则重置计时器.
- 在(核心,最大]即(20, 30] 区间的协程为同一类,退出时机受到Shutdown和ShutdownNow、“队列中是否有任务”的影响
如果队列有任务则拿任务运行,运行任务结束后,检查发现队列为空,则立即退出;队列不为空,重复前面的操作;

这意味着我们会有三类协程,它们内部监听着queue及不同退出策略.我需要给它们命名,你有什么好的建议吗?

- 第一类(0, 10]叫它们,常驻协程(resident goroutine)
Resident Goroutine
- 第二类(10, 20]叫它们,带有超时时间的临时协程
Temporary Goroutine With Timeout
- 第三类(20, 30]叫它们,临时协程
Temporary Goroutine


**最终我没有采取在讨论2提到的按协程分类创建策略.原因如下:**

- 代码结构上的重复,只有微小差别,还不好抽出方法来复用.
- 沿用 WIP 实现复用Go协程的调度模式 #80 中的例子,当协程数达到最大协程数即30时,此时初始协程数的协程们(10个)恰好运行任务结束,它们既没有任务运行也不能退出.当初始数、核心数及最大数的数值很大时更是一个问题.
本次重构采用的是在协程运行任务结束后,根据当前协程总数(totalNumOfGo)与初始数(10, initGo),核心数(20, coreGo)的关系来动态随机划分协程.并保证最终稳定在“初始数”即10个. 算法如下:

1. 如果当前协程满足`b.coreGo < b.totalNumOfGo && (len(b.queue) == 0 || int32(len(b.queue)) < b.totalNumOfGo)` 则更新协程总数并直接退出
2. 如果当前协程满足b.initGo < b.totalNumOfGo-b.timeoutGroup.size()那么当前协程会被设置定时器,在“最大空闲时间”内没接到任务会因为超时而退出.
3 . 其他情况要么因为队列中有任务,要么因为当前协程处于“初始数”组中不处理.


timeoutGroup是超时组,有两个作用:

- 通过协程id将其标记为“核心组”/超时组,使b.totalNumOfGo-b.timeoutGroup.size()能够准确反应结果保证动态分类的正确性
- timeoutGroup明确表示意图,而不是用idleTimer来表示这个逻辑分组.不再依赖于idleTimer.Stop这个不确定性大的方法



按照我的理解：

- 要不要开 goroutine：应该是在 submit task 的时候。每次 submit 之后看一下队列和goroutine 的数量，然后决定开不开；
- 要不要关掉 goroutine 是在每个 goroutine 从队列里面拿 task 的时候。相当于最大空闲时间创建一个 context，作为一个 case，和队列里面拿一个 task 作为一个 case，进行 select
- 在开 goroutine 的时候，每次应该最多开一个。而 initGo 应该是在 Start 的时候就创建起来了
- 在关 goroutine 的时候，则需要判断有没有下降到 initGo。

### Task 设计

Task 本身的设计并没有考虑任何返回值的问题，后面我们会提供一个允许返回值的任务的装饰器。为了简化用户的操作，即用户不希望自己的任务也需要实现 Task 接口，那么就可以使用我们的 TaskFunc 进行一个简单的转换：

```go
t := TaskFunc(func(ctx context.Context) {
    //.. 做些事情
})
```

Task 的 Run 方法被设计为接收一个 ctx 参数，这意味着用户应该考虑任务需要进行超时控制的问题，即使是慢任务，用户可能也预期这个任务能够在一小时或者两小时内释放掉 goroutine，即便此时它还没有执行完毕。

### BlockQueueTaskPool

基于队列的阻塞式的任务池实现。
核心在于通过 concurrency 和 queueSize 来控制 goroutine 数量和等待任务。
而如何控制 queueSize 和 concurrency 则有很多方案：

- 使用 channel
- 读写锁
- 原子变量

# 一个完整的生产级任务池应该长什么样子

## 控制住 goroutine 的数量

- 隔离快慢任务,让慢任务在数量有限的 goroutine 上运行，而其它快任务则没有这种限制

## 优雅关闭

- 用户希望在关闭的时候，可能希望尽量执行已经提交的任务，但是又不希望等待太久
- api设计

```
Shutdown 开始优雅关闭
 
ShutdownNow 立即关闭

awaitingShutdown 等待优雅关闭
```

## 任务池设计

- 用户希望支持任务池运行时变量的传递
- 用户希望支持任务超时控制
- 用户希望支持任务依赖有序
- 用户希望支持业务指定任务池

## 感知任务池运行情况

- 任务执行停止，怀疑发生死锁或执行耗时操作
- 任务执行时间超过平均执行周期
- 任务提交了但长时间没执行

## 运行指标（百分比+数量+top）

- 任务运行时长
- 任务等待时长
- 任务池活跃度
- 队列容量

## 运行时报警策略

- 任务运行超时
- 任务池活跃度
- 阻塞队列容量
- 触发拒绝策略
- 主流软件告警（微信，钉钉，飞书）

## 修改运行时生效

- 修改任务池 某一实例配置，或者修改 集群全部实例

## 监控运行可视化

- Prometheus + Grafana第三方采集监控
- 内置数据池数据采集 + 监控

## 代码演示

### option.go

```go
package option

// Option 是用于 Option 模式的泛型设计，
// 避免在代码中定义很多类似这样的结构体
// 一般情况下 T 应该是一个结构体
type Option[T any] func(t *T)

// Apply 将 opts 应用在 t 之上
func Apply[T any](t *T, opts ...Option[T]) {
	for _, opt := range opts {
		opt(t)
	}
}

// OptionErr 形如 Option，但是会返回一个 error
// 你应该优先使用 Option，除非你在设计 option 模式的时候需要进行一些校验
type OptionErr[T any] func(t *T) error

// ApplyErr 形如 Apply，它将 opts 应用在 t 之上，
// 如果 opts 中任何一个返回 error，那么它会中断并且返回 error
func ApplyErr[T any](t *T, opts ...OptionErr[T]) error {
	for _, opt := range opts {
		if err := opt(t); err != nil {
			return err
		}
	}
	return nil
}


```

### task_pool.go
```go
package pool

import (
	"context"
	"errors"
	"fmt"
	"runtime"
	"sync"
	"sync/atomic"
	"time"

	"github.com/gotomicro/ekit/bean/option"
)

var (
	stateCreated int32 = 1
	stateRunning int32 = 2
	stateClosing int32 = 3
	stateStopped int32 = 4
	stateLocked  int32 = 5

	errTaskPoolIsNotRunning = errors.New("ekit: TaskPool未运行")
	errTaskPoolIsClosing    = errors.New("ekit：TaskPool关闭中")
	errTaskPoolIsStopped    = errors.New("ekit: TaskPool已停止")
	errTaskPoolIsStarted    = errors.New("ekit：TaskPool已运行")
	errTaskIsInvalid        = errors.New("ekit: Task非法")
	errTaskRunningPanic     = errors.New("ekit: Task运行时异常")

	errInvalidArgument = errors.New("ekit: 参数非法")

	_            TaskPool = &OnDemandBlockTaskPool{}
	panicBuffLen          = 2048

	defaultMaxIdleTime = 10 * time.Second
)

// TaskPool 任务池
type TaskPool interface {
	// Submit 执行一个任务
	// 如果任务池提供了阻塞的功能，那么如果在 ctx 过期都没有提交成功，那么应该返回错误
	// 调用 Start 之后能否继续提交任务，则取决于具体的实现
	// 调用 Shutdown 或者 ShutdownNow 之后提交任务都会返回错误
	Submit(ctx context.Context, task Task) error

	// Start 开始调度任务执行。在调用 Start 之前，所有的任务都不会被调度执行。
	// Start 之后，能否继续调用 Submit 提交任务，取决于具体的实现
	Start() error

	// Shutdown 关闭任务池。如果此时尚未调用 Start 方法，那么将会立刻返回。
	// 任务池将会停止接收新的任务，但是会继续执行剩下的任务，
	// 在所有任务执行完毕之后，用户可以从返回的 chan 中得到通知
	// 任务池在发出通知之后会关闭 chan struct{}
	Shutdown() (<-chan struct{}, error)

	// ShutdownNow 立刻关闭线程池
	// 任务池能否中断当前正在执行的任务，取决于 TaskPool 的具体实现，以及 Task 的具体实现
	// 该方法会返回所有剩下的任务，剩下的任务是否包含正在执行的任务，也取决于具体的实现
	ShutdownNow() ([]Task, error)
}

// Task 代表一个任务
type Task interface {
	// Run 执行任务
	// 如果 ctx 设置了超时时间，那么实现者需要自己决定是否进行超时控制
	Run(ctx context.Context) error
}

// TaskFunc 一个可执行的任务
type TaskFunc func(ctx context.Context) error

// Run 执行任务
// 超时控制取决于衍生出 TaskFunc 的方法
func (t TaskFunc) Run(ctx context.Context) error { return t(ctx) }

// taskWrapper 是Task的装饰器
type taskWrapper struct {
	t Task
}

func (tw *taskWrapper) Run(ctx context.Context) (err error) {
	defer func() {
		// 处理 panic
		if r := recover(); r != nil {
			buf := make([]byte, panicBuffLen)
			buf = buf[:runtime.Stack(buf, false)]
			err = fmt.Errorf("%w：%s", errTaskRunningPanic, fmt.Sprintf("[PANIC]:\t%+v\n%s\n", r, buf))
		}
	}()
	return tw.t.Run(ctx)
}

type group struct {
	mp map[int]int
	n  int32
	mu sync.RWMutex
}

func (g *group) isIn(id int) bool {
	g.mu.RLock()
	defer g.mu.RUnlock()
	_, ok := g.mp[id]
	return ok
}

func (g *group) add(id int) {
	g.mu.Lock()
	defer g.mu.Unlock()
	if _, ok := g.mp[id]; !ok {
		g.mp[id] = 1
		g.n++
	}
}

func (g *group) delete(id int) {
	g.mu.Lock()
	defer g.mu.Unlock()
	if _, ok := g.mp[id]; ok {
		g.n--
	}
	delete(g.mp, id)
}

func (g *group) size() int32 {
	g.mu.RLock()
	defer g.mu.RUnlock()
	return g.n
}

// OnDemandBlockTaskPool 按需创建goroutine的并发阻塞的任务池
type OnDemandBlockTaskPool struct {
	// TaskPool内部状态
	state int32

	queue             chan Task
	numGoRunningTasks int32

	totalGo int32
	mutex   sync.RWMutex

	// 初始协程数
	initGo int32
	// 核心协程数
	coreGo int32
	// 最大协程数
	maxGo int32
	// 超时组
	timeoutGroup *group
	// 最大空闲时间
	maxIdleTime time.Duration
	// 队列积压率
	queueBacklogRate float64
	shutdownOnce     sync.Once

	// 协程id方便调试程序
	id int32

	// 外部信号
	shutdownDone chan struct{}
	// 内部中断信号
	shutdownNowCtx    context.Context
	shutdownNowCancel context.CancelFunc
}

// NewOnDemandBlockTaskPool 创建一个新的 OnDemandBlockTaskPool
// initGo 是初始协程数
// queueSize 是队列大小，即最多有多少个任务在等待调度
// 使用相应的Option选项可以动态扩展协程数
func NewOnDemandBlockTaskPool(initGo int, queueSize int, opts ...option.Option[OnDemandBlockTaskPool]) (*OnDemandBlockTaskPool, error) {
	if initGo < 1 {
		return nil, fmt.Errorf("%w：initGo应该大于0", errInvalidArgument)
	}
	if queueSize < 0 {
		return nil, fmt.Errorf("%w：queueSize应该大于等于0", errInvalidArgument)
	}
	b := &OnDemandBlockTaskPool{
		queue:        make(chan Task, queueSize),
		shutdownDone: make(chan struct{}, 1),
		initGo:       int32(initGo),
		coreGo:       int32(initGo),
		maxGo:        int32(initGo),
		maxIdleTime:  defaultMaxIdleTime,
	}

	b.shutdownNowCtx, b.shutdownNowCancel = context.WithCancel(context.Background())
	atomic.StoreInt32(&b.state, stateCreated)

	option.Apply(b, opts...)

	if b.coreGo != b.initGo && b.maxGo == b.initGo {
		b.maxGo = b.coreGo
	} else if b.coreGo == b.initGo && b.maxGo != b.initGo {
		b.coreGo = b.maxGo
	}
	if !(b.initGo <= b.coreGo && b.coreGo <= b.maxGo) {
		return nil, fmt.Errorf("%w : 需要满足initGo <= coreGo <= maxGo条件", errInvalidArgument)
	}

	b.timeoutGroup = &group{mp: make(map[int]int)}

	if b.queueBacklogRate < float64(0) || float64(1) < b.queueBacklogRate {
		return nil, fmt.Errorf("%w ：queueBacklogRate合法范围为[0,1.0]", errInvalidArgument)
	}
	return b, nil
}

func WithQueueBacklogRate(rate float64) option.Option[OnDemandBlockTaskPool] {
	return func(pool *OnDemandBlockTaskPool) {
		pool.queueBacklogRate = rate
	}
}

func WithCoreGo(n int32) option.Option[OnDemandBlockTaskPool] {
	return func(pool *OnDemandBlockTaskPool) {
		pool.coreGo = n
	}
}

func WithMaxGo(n int32) option.Option[OnDemandBlockTaskPool] {
	return func(pool *OnDemandBlockTaskPool) {
		pool.maxGo = n
	}
}

func WithMaxIdleTime(d time.Duration) option.Option[OnDemandBlockTaskPool] {
	return func(pool *OnDemandBlockTaskPool) {
		pool.maxIdleTime = d
	}
}

// Submit 提交一个任务
// 如果此时队列已满，那么将会阻塞调用者。
// 如果因为 ctx 的原因返回，那么将会返回 ctx.Err()
// 在调用 Start 前后都可以调用 Submit
func (b *OnDemandBlockTaskPool) Submit(ctx context.Context, task Task) error {
	if task == nil {
		return fmt.Errorf("%w", errTaskIsInvalid)
	}
	// todo: 用户未设置超时，可以考虑内部给个超时提交
	for {

		if atomic.LoadInt32(&b.state) == stateClosing {
			return fmt.Errorf("%w", errTaskPoolIsClosing)
		}

		if atomic.LoadInt32(&b.state) == stateStopped {
			return fmt.Errorf("%w", errTaskPoolIsStopped)
		}

		task = &taskWrapper{t: task}

		ok, err := b.trySubmit(ctx, task, stateCreated)
		if ok || err != nil {
			return err
		}

		ok, err = b.trySubmit(ctx, task, stateRunning)
		if ok || err != nil {
			return err
		}
	}
}

func (b *OnDemandBlockTaskPool) trySubmit(ctx context.Context, task Task, state int32) (bool, error) {
	// 进入临界区
	if atomic.CompareAndSwapInt32(&b.state, state, stateLocked) {
		defer atomic.CompareAndSwapInt32(&b.state, stateLocked, state)

		// 此处b.queue <- task不会因为b.queue被关闭而panic
		// 代码执行到trySubmit时TaskPool处于lock状态
		// 要关闭b.queue需要TaskPool处于RUNNING状态，Shutdown/ShutdownNow才能成功
		select {
		case <-ctx.Done():
			return false, fmt.Errorf("%w", ctx.Err())
		case b.queue <- task:
			if state == stateRunning && b.allowToCreateGoroutine() {
				b.increaseTotalGo(1)
				id := int(atomic.AddInt32(&b.id, 1))
				go b.goroutine(id)
				// log.Println("create go ", id)
			}
			return true, nil
		default:
			// 不能阻塞在临界区,要给Shutdown和ShutdownNow机会
			return false, nil
		}
	}
	return false, nil
}

func (b *OnDemandBlockTaskPool) allowToCreateGoroutine() bool {
	b.mutex.RLock()
	defer b.mutex.RUnlock()

	if b.totalGo == b.maxGo {
		return false
	}

	// 这个判断可能太苛刻了，经常导致开协程失败，先注释掉
	// allGoShouldBeBusy := atomic.LoadInt32(&b.numGoRunningTasks) == b.totalGo
	// if !allGoShouldBeBusy {
	// 	return false
	// }

	rate := float64(len(b.queue)) / float64(cap(b.queue))
	if rate == 0 || rate < b.queueBacklogRate {
		// log.Println("rate == 0", rate == 0, "rate", rate, " < ", b.queueBacklogRate)
		return false
	}

	// b.totalGo < b.maxGo && rate != 0 && rate >= b.queueBacklogRate
	return true
}

// Start 开始调度任务执行
// Start 之后，调用者可以继续使用 Submit 提交任务
func (b *OnDemandBlockTaskPool) Start() error {

	for {

		if atomic.LoadInt32(&b.state) == stateClosing {
			return fmt.Errorf("%w", errTaskPoolIsClosing)
		}

		if atomic.LoadInt32(&b.state) == stateStopped {
			return fmt.Errorf("%w", errTaskPoolIsStopped)
		}

		if atomic.LoadInt32(&b.state) == stateRunning {
			return fmt.Errorf("%w", errTaskPoolIsStarted)
		}

		if atomic.CompareAndSwapInt32(&b.state, stateCreated, stateLocked) {

			n := b.initGo

			allowGo := b.maxGo - b.initGo
			needGo := int32(len(b.queue)) - b.initGo
			if needGo > 0 {
				if needGo <= allowGo {
					n += needGo
				} else {
					n += allowGo
				}
			}

			b.increaseTotalGo(n)
			for i := int32(0); i < n; i++ {
				go b.goroutine(int(atomic.AddInt32(&b.id, 1)))
			}
			atomic.CompareAndSwapInt32(&b.state, stateLocked, stateRunning)
			return nil
		}
	}
}

func (b *OnDemandBlockTaskPool) goroutine(id int) {

	// 刚启动的协程除非恰巧赶上Shutdown/ShutdownNow被调用，否则应该至少执行一个task
	idleTimer := time.NewTimer(0)
	if !idleTimer.Stop() {
		<-idleTimer.C
	}

	for {
		// log.Println("id", id, "working for loop")
		select {
		case <-b.shutdownNowCtx.Done():
			// log.Printf("id %d shutdownNow, timeoutGroup.Size=%d left\n", id, b.timeoutGroup.size())
			b.decreaseTotalGo(1)
			return
		case <-idleTimer.C:
			b.mutex.Lock()
			b.totalGo--
			b.timeoutGroup.delete(id)
			// log.Printf("id %d timeout, timeoutGroup.Size=%d left\n", id, b.timeoutGroup.size())
			b.mutex.Unlock()
			return
		case task, ok := <-b.queue:

			// log.Println("id", id, "running tasks")
			if b.timeoutGroup.isIn(id) {
				// timer只保证至少在等待X时间后才发送信号而不是在X时间内发送信号
				b.timeoutGroup.delete(id)
				// timer的Stop方法不保证一定成功
				// 不加判断并将信号清除可能会导致协程下次在case<-idleTimer.C处退出
				if !idleTimer.Stop() {
					<-idleTimer.C
				}
				// log.Println("id", id, "out timeoutGroup")
			}

			atomic.AddInt32(&b.numGoRunningTasks, 1)
			if !ok {
				// b.numGoRunningTasks > 1表示虽然当前协程监听到了b.queue关闭但还有其他协程运行task，当前协程自己退出就好
				// b.numGoRunningTasks == 1表示只有当前协程"运行task"中，其他协程在一定在"拿到b.queue到已关闭"，这一信号的路上
				// 绝不会处于运行task中
				if atomic.CompareAndSwapInt32(&b.numGoRunningTasks, 1, 0) && atomic.LoadInt32(&b.state) == stateClosing {
					// 在b.queue关闭后，第一个检测到全部task已经自然结束的协程
					b.shutdownOnce.Do(func() {
						// 状态迁移
						atomic.CompareAndSwapInt32(&b.state, stateClosing, stateStopped)
						// 显示通知外部调用者
						b.shutdownDone <- struct{}{}
						close(b.shutdownDone)
					})

					b.decreaseTotalGo(1)
					return
				}

				// 有其他协程运行task中，自己退出就好。
				atomic.AddInt32(&b.numGoRunningTasks, -1)
				b.decreaseTotalGo(1)
				return
			}

			// todo handle error
			_ = task.Run(b.shutdownNowCtx)
			atomic.AddInt32(&b.numGoRunningTasks, -1)

			b.mutex.Lock()
			// log.Println("id", id, "totalGo-mem", b.totalGo-b.timeoutGroup.size(), "totalGo", b.totalGo, "mem", b.timeoutGroup.size())
			if b.coreGo < b.totalGo && (len(b.queue) == 0 || int32(len(b.queue)) < b.totalGo) {
				// 协程在(coreGo,maxGo]区间
				// 如果没有任务可以执行，或者被判定为可能抢不到任务的协程直接退出
				// 注意：一定要在此处减1才能让此刻等待在mutex上的其他协程被正确地分区
				b.totalGo--
				// log.Println("id", id, "exits....")
				b.mutex.Unlock()
				return
			}

			if b.initGo < b.totalGo-b.timeoutGroup.size() /* && len(b.queue) == 0 */ {
				// log.Println("id", id, "initGo", b.initGo, "totalGo-mem", b.totalGo-b.timeoutGroup.size(), "totalGo", b.totalGo)
				// 协程在(initGo，coreGo]区间，如果没有任务可以执行，重置计时器
				// 当len(b.queue) != 0时，即便协程属于(coreGo,maxGo]区间，也应该给它一个定时器兜底。
				// 因为现在看队列中有任务，等真去拿的时候可能恰好没任务，如果不给它一个定时器兜底此时就会出现当前协程总数长时间大于始协程数（initGo）的情况。
				// 直到队列再次有任务时才可能将当前总协程数准确无误地降至初始协程数，因此注释掉len(b.queue) == 0判断条件
				idleTimer = time.NewTimer(b.maxIdleTime)
				b.timeoutGroup.add(id)
				// log.Println("id", id, "add timeoutGroup", "size", b.timeoutGroup.size())
			}

			b.mutex.Unlock()
		}
	}
}

func (b *OnDemandBlockTaskPool) increaseTotalGo(n int32) {
	b.mutex.Lock()
	b.totalGo += n
	b.mutex.Unlock()
}

func (b *OnDemandBlockTaskPool) decreaseTotalGo(n int32) {
	b.mutex.Lock()
	b.totalGo -= n
	b.mutex.Unlock()
}

// Shutdown 将会拒绝提交新的任务，但是会继续执行已提交任务
// 当执行完毕后，会往返回的 chan 中丢入信号
// Shutdown 会负责关闭返回的 chan
// Shutdown 无法中断正在执行的任务
func (b *OnDemandBlockTaskPool) Shutdown() (<-chan struct{}, error) {

	for {

		if atomic.LoadInt32(&b.state) == stateCreated {
			return nil, fmt.Errorf("%w", errTaskPoolIsNotRunning)
		}

		if atomic.LoadInt32(&b.state) == stateStopped {
			return nil, fmt.Errorf("%w", errTaskPoolIsStopped)
		}

		if atomic.LoadInt32(&b.state) == stateClosing {
			return nil, fmt.Errorf("%w", errTaskPoolIsClosing)
		}

		if atomic.CompareAndSwapInt32(&b.state, stateRunning, stateClosing) {
			// 目标：不但希望正在运行中的任务自然退出，还希望队列中等待的任务也能启动执行并自然退出
			// 策略：先将队列中的任务启动并执行（清空队列），再等待全部运行中的任务自然退出

			// 先关闭等待队列不再允许提交
			// 同时工作协程能够通过判断b.queue是否被关闭来终止获取任务循环
			close(b.queue)
			return b.shutdownDone, nil
		}

	}
}

// ShutdownNow 立刻关闭任务池，并且返回所有剩余未执行的任务（不包含正在执行的任务）
func (b *OnDemandBlockTaskPool) ShutdownNow() ([]Task, error) {

	for {

		if atomic.LoadInt32(&b.state) == stateCreated {
			return nil, fmt.Errorf("%w", errTaskPoolIsNotRunning)
		}

		if atomic.LoadInt32(&b.state) == stateClosing {
			return nil, fmt.Errorf("%w", errTaskPoolIsClosing)
		}

		if atomic.LoadInt32(&b.state) == stateStopped {
			return nil, fmt.Errorf("%w", errTaskPoolIsStopped)
		}

		if atomic.CompareAndSwapInt32(&b.state, stateRunning, stateStopped) {
			// 目标：立刻关闭并且返回所有剩下未执行的任务
			// 策略：关闭等待队列不再接受新任务，中断工作协程的获取任务循环，清空等待队列并保存返回

			close(b.queue)

			// 发送中断信号，中断工作协程获取任务循环
			b.shutdownNowCancel()

			// 清空队列并保存
			tasks := make([]Task, 0, len(b.queue))
			for task := range b.queue {
				tasks = append(tasks, task)
			}
			return tasks, nil
		}
	}
}

// internalState 用于查看TaskPool状态
func (b *OnDemandBlockTaskPool) internalState() int32 {
	for {
		state := atomic.LoadInt32(&b.state)
		if state != stateLocked {
			return state
		}
	}
}

// numOfGo 用于查看TaskPool中有多少工作协程
func (b *OnDemandBlockTaskPool) numOfGo() int32 {
	var n int32
	b.mutex.RLock()
	n = b.totalGo
	b.mutex.RUnlock()
	return n
}

```


### task_pool_test.go
```go
package pool

import (
	"context"
	"errors"
	"fmt"
	"sync"
	"testing"
	"time"

	"github.com/gotomicro/ekit/bean/option"
	"github.com/stretchr/testify/assert"
	"golang.org/x/sync/errgroup"
)

/*
TaskPool有限状态机
                                                       Start/Submit/ShutdownNow() Error
                                                                \     /
                                               Shutdown()  --> CLOSING  ---等待所有任务结束
         Submit()nil--执行中状态迁移--Submit()      /    \----------/ \----------/
           \    /                    \   /      /
New() --> CREATED -- Start() --->  RUNNING -- --
           \   /                    \   /       \           Start/Submit/Shutdown() Error
  Shutdown/ShutdownNow()Error      Start()       \                \    /
                                               ShutdownNow() ---> STOPPED  -- ShutdownNow() --> STOPPED
*/

func TestOnDemandBlockTaskPool_In_Created_State(t *testing.T) {
	t.Parallel()

	t.Run("New", func(t *testing.T) {
		t.Parallel()

		pool, err := NewOnDemandBlockTaskPool(1, -1)
		assert.ErrorIs(t, err, errInvalidArgument)
		assert.Nil(t, pool)

		pool, err = NewOnDemandBlockTaskPool(1, 0)
		assert.NoError(t, err)
		assert.NotNil(t, pool)

		pool, err = NewOnDemandBlockTaskPool(1, 1)
		assert.NoError(t, err)
		assert.NotNil(t, pool)

		pool, err = NewOnDemandBlockTaskPool(-1, 1)
		assert.ErrorIs(t, err, errInvalidArgument)
		assert.Nil(t, pool)

		pool, err = NewOnDemandBlockTaskPool(0, 1)
		assert.ErrorIs(t, err, errInvalidArgument)
		assert.Nil(t, pool)

		pool, err = NewOnDemandBlockTaskPool(1, 1)
		assert.NoError(t, err)
		assert.NotNil(t, pool)

		t.Run("With Options", func(t *testing.T) {
			t.Parallel()

			initGo := 10
			pool, err := NewOnDemandBlockTaskPool(initGo, 10)
			assert.NoError(t, err)

			assert.Equal(t, int32(initGo), pool.initGo)
			assert.Equal(t, int32(initGo), pool.coreGo)
			assert.Equal(t, int32(initGo), pool.maxGo)
			assert.Equal(t, defaultMaxIdleTime, pool.maxIdleTime)

			coreGo, maxGo, maxIdleTime := int32(20), int32(30), 10*time.Second
			pool, err = NewOnDemandBlockTaskPool(initGo, 10, WithCoreGo(coreGo), WithMaxGo(maxGo), WithMaxIdleTime(maxIdleTime))
			assert.NoError(t, err)

			assert.Equal(t, int32(initGo), pool.initGo)
			assert.Equal(t, coreGo, pool.coreGo)
			assert.Equal(t, maxGo, pool.maxGo)
			assert.Equal(t, maxIdleTime, pool.maxIdleTime)

			pool, err = NewOnDemandBlockTaskPool(initGo, 10, WithCoreGo(coreGo))
			assert.NoError(t, err)
			assert.Equal(t, pool.coreGo, pool.maxGo)

			initGo, coreGo = 30, 20
			pool, err = NewOnDemandBlockTaskPool(initGo, 10, WithCoreGo(coreGo))
			assert.Nil(t, pool)
			assert.ErrorIs(t, err, errInvalidArgument)

			pool, err = NewOnDemandBlockTaskPool(initGo, 10, WithMaxGo(maxGo))
			assert.NoError(t, err)
			assert.Equal(t, pool.maxGo, pool.coreGo)

			initGo, maxGo = 30, 10
			pool, err = NewOnDemandBlockTaskPool(initGo, 10, WithMaxGo(maxGo))
			assert.Nil(t, pool)
			assert.ErrorIs(t, err, errInvalidArgument)

			initGo, coreGo, maxGo = 30, 20, 10
			pool, err = NewOnDemandBlockTaskPool(initGo, 10, WithCoreGo(coreGo), WithMaxGo(maxGo))
			assert.Nil(t, pool)
			assert.ErrorIs(t, err, errInvalidArgument)

			initGo, coreGo, maxGo = 30, 10, 20
			pool, err = NewOnDemandBlockTaskPool(initGo, 10, WithCoreGo(coreGo), WithMaxGo(maxGo))
			assert.Nil(t, pool)
			assert.ErrorIs(t, err, errInvalidArgument)

			initGo, coreGo, maxGo = 20, 10, 30
			pool, err = NewOnDemandBlockTaskPool(initGo, 10, WithCoreGo(coreGo), WithMaxGo(maxGo))
			assert.Nil(t, pool)
			assert.ErrorIs(t, err, errInvalidArgument)

			initGo, coreGo, maxGo = 20, 30, 10
			pool, err = NewOnDemandBlockTaskPool(initGo, 10, WithCoreGo(coreGo), WithMaxGo(maxGo))
			assert.Nil(t, pool)
			assert.ErrorIs(t, err, errInvalidArgument)

			initGo, coreGo, maxGo = 10, 30, 20
			pool, err = NewOnDemandBlockTaskPool(initGo, 10, WithCoreGo(coreGo), WithMaxGo(maxGo))
			assert.Nil(t, pool)
			assert.ErrorIs(t, err, errInvalidArgument)

			pool, err = NewOnDemandBlockTaskPool(initGo, 10, WithQueueBacklogRate(-0.1))
			assert.Nil(t, pool)
			assert.ErrorIs(t, err, errInvalidArgument)

			pool, err = NewOnDemandBlockTaskPool(initGo, 10, WithQueueBacklogRate(1.0))
			assert.NotNil(t, pool)
			assert.NoError(t, err)

			pool, err = NewOnDemandBlockTaskPool(initGo, 10, WithQueueBacklogRate(1.1))
			assert.Nil(t, pool)
			assert.ErrorIs(t, err, errInvalidArgument)

		})
	})

	// Start()导致TaskPool状态迁移，测试见TestTaskPool_In_Running_State/Start

	t.Run("Submit", func(t *testing.T) {
		t.Parallel()

		t.Run("提交非法Task", func(t *testing.T) {
			t.Parallel()

			pool, _ := NewOnDemandBlockTaskPool(1, 1)
			assert.Equal(t, stateCreated, pool.internalState())
			assert.ErrorIs(t, pool.Submit(context.Background(), nil), errTaskIsInvalid)
			assert.Equal(t, stateCreated, pool.internalState())
		})

		t.Run("正常提交Task", func(t *testing.T) {
			t.Parallel()

			pool, _ := NewOnDemandBlockTaskPool(1, 3)
			assert.Equal(t, stateCreated, pool.internalState())
			testSubmitValidTask(t, pool)
			assert.Equal(t, stateCreated, pool.internalState())
		})

		t.Run("阻塞提交并导致超时", func(t *testing.T) {
			t.Parallel()

			pool, _ := NewOnDemandBlockTaskPool(1, 1)
			assert.Equal(t, stateCreated, pool.internalState())
			testSubmitBlockingAndTimeout(t, pool)
			assert.Equal(t, stateCreated, pool.internalState())
		})
	})

	t.Run("Shutdown", func(t *testing.T) {
		t.Parallel()

		pool, err := NewOnDemandBlockTaskPool(1, 1)
		assert.NoError(t, err)
		assert.Equal(t, stateCreated, pool.internalState())

		done, err := pool.Shutdown()
		assert.Nil(t, done)
		assert.ErrorIs(t, err, errTaskPoolIsNotRunning)
		assert.Equal(t, stateCreated, pool.internalState())
	})

	t.Run("ShutdownNow", func(t *testing.T) {
		t.Parallel()

		pool, err := NewOnDemandBlockTaskPool(1, 1)
		assert.NoError(t, err)
		assert.Equal(t, stateCreated, pool.internalState())

		tasks, err := pool.ShutdownNow()
		assert.Nil(t, tasks)
		assert.ErrorIs(t, err, errTaskPoolIsNotRunning)
		assert.Equal(t, stateCreated, pool.internalState())
	})
}

func TestOnDemandBlockTaskPool_In_Running_State(t *testing.T) {
	t.Parallel()

	t.Run("Start —— 使TaskPool状态由Created变为Running", func(t *testing.T) {
		t.Parallel()

		pool, _ := NewOnDemandBlockTaskPool(1, 1)

		// 与下方 testSubmitBlockingAndTimeout 并发执行
		errChan := make(chan error)
		go func() {
			time.Sleep(1 * time.Millisecond)
			errChan <- pool.Start()
		}()

		assert.Equal(t, stateCreated, pool.internalState())

		testSubmitBlockingAndTimeout(t, pool)

		assert.NoError(t, <-errChan)
		assert.Equal(t, stateRunning, pool.internalState())

		// 重复调用
		assert.ErrorIs(t, pool.Start(), errTaskPoolIsStarted)
		assert.Equal(t, stateRunning, pool.internalState())
	})

	t.Run("Start —— 在TaskPool启动前队列中已有任务，启动后不再Submit", func(t *testing.T) {

		t.Run("WithCoreGo,WithMaxIdleTime，所需要协程数 <= 允许创建的协程数", func(t *testing.T) {

			initGo, coreGo, maxIdleTime := 1, 3, 3*time.Millisecond
			queueSize := coreGo

			needGo, allowGo := queueSize-initGo, coreGo-initGo
			assert.LessOrEqual(t, needGo, allowGo)

			pool, err := NewOnDemandBlockTaskPool(initGo, queueSize, WithCoreGo(int32(coreGo)), WithMaxIdleTime(maxIdleTime))
			assert.NoError(t, err)

			assert.Equal(t, int32(0), pool.numOfGo())

			done := make(chan struct{}, coreGo)
			wait := make(chan struct{}, coreGo)

			for i := 0; i < coreGo; i++ {
				err := pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error {
					wait <- struct{}{}
					<-done
					return nil
				}))
				assert.NoError(t, err)
			}

			assert.Equal(t, int32(0), pool.numOfGo())

			assert.NoError(t, pool.Start())

			for i := 0; i < coreGo; i++ {
				<-wait
			}
			assert.Equal(t, int32(coreGo), pool.numOfGo())
		})

		t.Run("WithMaxGo, 所需要协程数 > 允许创建的协程数", func(t *testing.T) {
			initGo, maxGo := 3, 5
			queueSize := maxGo + 1

			needGo, allowGo := queueSize-initGo, maxGo-initGo
			assert.Greater(t, needGo, allowGo)

			pool, err := NewOnDemandBlockTaskPool(initGo, queueSize, WithMaxGo(int32(maxGo)))
			assert.NoError(t, err)

			assert.Equal(t, int32(0), pool.numOfGo())

			done := make(chan struct{}, queueSize)
			wait := make(chan struct{}, queueSize)

			for i := 0; i < queueSize; i++ {
				err := pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error {
					wait <- struct{}{}
					<-done
					return nil
				}))
				assert.NoError(t, err)
			}

			assert.Equal(t, int32(0), pool.numOfGo())

			assert.NoError(t, pool.Start())

			for i := 0; i < maxGo; i++ {
				<-wait
			}
			assert.Equal(t, int32(maxGo), pool.numOfGo())
		})
	})

	t.Run("Start —— 与Submit并发调用,WithCoreGo,WithMaxIdleTime,WithMaxGo，所需要协程数 < 允许创建的协程数", func(t *testing.T) {

		initGo, coreGo, maxGo, maxIdleTime := 2, 4, 6, 3*time.Millisecond
		queueSize := coreGo

		needGo, allowGo := queueSize-initGo, maxGo-initGo
		assert.Less(t, needGo, allowGo)

		pool, err := NewOnDemandBlockTaskPool(initGo, queueSize, WithCoreGo(int32(coreGo)), WithMaxGo(int32(maxGo)), WithMaxIdleTime(maxIdleTime))
		assert.NoError(t, err)

		assert.Equal(t, int32(0), pool.numOfGo())

		done := make(chan struct{}, queueSize)
		wait := make(chan struct{}, queueSize)

		// 与下方阻塞提交并发调用
		errChan := make(chan error)
		go func() {
			time.Sleep(10 * time.Millisecond)
			errChan <- pool.Start()
		}()

		// 模拟阻塞提交
		for i := 0; i < maxGo; i++ {
			err := pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error {
				wait <- struct{}{}
				<-done
				return nil
			}))
			assert.NoError(t, err)
		}

		assert.NoError(t, <-errChan)

		for i := 0; i < maxGo; i++ {
			<-wait
		}

		assert.Equal(t, int32(maxGo), pool.numOfGo())
	})

	t.Run("Submit", func(t *testing.T) {
		t.Parallel()

		t.Run("提交非法Task", func(t *testing.T) {
			t.Parallel()

			pool := testNewRunningStateTaskPool(t, 1, 1)
			assert.ErrorIs(t, pool.Submit(context.Background(), nil), errTaskIsInvalid)
			assert.Equal(t, stateRunning, pool.internalState())
		})

		t.Run("正常提交Task", func(t *testing.T) {
			t.Parallel()

			pool := testNewRunningStateTaskPool(t, 1, 3)
			testSubmitValidTask(t, pool)
			assert.Equal(t, stateRunning, pool.internalState())
		})

		t.Run("阻塞提交并导致超时", func(t *testing.T) {
			t.Parallel()

			pool := testNewRunningStateTaskPool(t, 1, 1)
			testSubmitBlockingAndTimeout(t, pool)
			assert.Equal(t, stateRunning, pool.internalState())
		})
	})

	// Shutdown()导致TaskPool状态迁移，TestTaskPool_In_Closing_State/Shutdown

	// ShutdownNow()导致TaskPool状态迁移，TestTestPool_In_Stopped_State/ShutdownNow

	t.Run("工作协程", func(t *testing.T) {
		t.Parallel()

		t.Run("保持在初始数不变", func(t *testing.T) {
			t.Parallel()

			initGo, queueSize := 1, 3
			pool := testNewRunningStateTaskPool(t, initGo, queueSize)

			n := queueSize
			done1 := make(chan struct{}, n)
			wait := make(chan struct{}, n)

			// 队列中有积压任务
			for i := 0; i < n; i++ {
				err := pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error {
					wait <- struct{}{}
					<-done1
					return nil
				}))
				assert.NoError(t, err)
			}

			// initGo个tasks在运行中
			for i := 0; i < initGo; i++ {
				<-wait
			}

			assert.Equal(t, int32(initGo), pool.numOfGo())

			// 使运行中的tasks结束
			for i := 0; i < initGo; i++ {
				done1 <- struct{}{}
			}

			// 积压在队列中的任务开始运行
			for i := 0; i < n-initGo; i++ {
				<-wait
				assert.Equal(t, int32(initGo), pool.numOfGo())
				done1 <- struct{}{}
			}

		})

		t.Run("从初始数达到核心数", func(t *testing.T) {
			t.Parallel()

			t.Run("核心数比初始数多1个", func(t *testing.T) {
				t.Parallel()

				initGo, coreGo, maxIdleTime, queueBacklogRate := int32(2), int32(3), 3*time.Millisecond, 0.1
				queueSize := int(coreGo)
				testExtendGoFromInitGoToCoreGo(t, initGo, queueSize, coreGo, maxIdleTime, WithQueueBacklogRate(queueBacklogRate))
			})

			t.Run("核心数比初始数多n个", func(t *testing.T) {
				t.Parallel()

				initGo, coreGo, maxIdleTime, queueBacklogRate := int32(2), int32(5), 3*time.Millisecond, 0.1
				queueSize := int(coreGo)
				testExtendGoFromInitGoToCoreGo(t, initGo, queueSize, coreGo, maxIdleTime, WithQueueBacklogRate(queueBacklogRate))
			})

			t.Run("在(初始数,核心数]区间的协程运行完任务后，在等待退出期间再次抢到任务", func(t *testing.T) {
				t.Parallel()

				initGo, coreGo, maxIdleTime := int32(1), int32(6), 100*time.Millisecond
				queueSize := int(coreGo)

				pool := testNewRunningStateTaskPool(t, int(initGo), queueSize, WithCoreGo(coreGo), WithMaxIdleTime(maxIdleTime))

				assert.Equal(t, initGo, pool.numOfGo())
				t.Log("1")
				done := make(chan struct{}, queueSize)
				wait := make(chan struct{}, queueSize)

				for i := 0; i < queueSize; i++ {
					i := i
					err := pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error {
						wait <- struct{}{}
						<-done
						t.Log("task done", i)
						return nil
					}))
					assert.NoError(t, err)
				}
				t.Log("2")
				for i := 0; i < queueSize; i++ {
					t.Log("wait ", i)
					<-wait
				}
				assert.Equal(t, coreGo, pool.numOfGo())

				close(done)
				t.Log("3")
				err := pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error {
					<-done
					t.Log("task done [x]")
					return nil
				}))
				assert.NoError(t, err)
				t.Log("4")
				// <-time.After(maxIdleTime * 100)
				for pool.numOfGo() > initGo {
					t.Log("loop", "numOfGo", pool.numOfGo(), "timeoutGroup", pool.timeoutGroup.size())
					time.Sleep(maxIdleTime)
				}
				assert.Equal(t, initGo, pool.numOfGo())
			})
		})

		t.Run("从核心数到达最大数", func(t *testing.T) {
			t.Parallel()

			t.Run("最大数比核心数多1个", func(t *testing.T) {
				t.Parallel()

				initGo, coreGo, maxGo, maxIdleTime, queueBacklogRate := int32(2), int32(4), int32(5), 3*time.Millisecond, 0.1
				queueSize := int(maxGo)
				testExtendGoFromInitGoToCoreGoAndMaxGo(t, initGo, queueSize, coreGo, maxGo, maxIdleTime, WithQueueBacklogRate(queueBacklogRate))
			})

			t.Run("最大数比核心数多n个", func(t *testing.T) {
				t.Parallel()

				initGo, coreGo, maxGo, maxIdleTime, queueBacklogRate := int32(1), int32(3), int32(6), 3*time.Millisecond, 0.1
				queueSize := int(maxGo)
				testExtendGoFromInitGoToCoreGoAndMaxGo(t, initGo, queueSize, coreGo, maxGo, maxIdleTime, WithQueueBacklogRate(queueBacklogRate))
			})
		})
	})

}

func testExtendGoFromInitGoToCoreGo(t *testing.T, initGo int32, queueSize int, coreGo int32, maxIdleTime time.Duration, opts ...option.Option[OnDemandBlockTaskPool]) {

	opts = append(opts, WithCoreGo(coreGo), WithMaxIdleTime(maxIdleTime))
	pool := testNewRunningStateTaskPool(t, int(initGo), queueSize, opts...)

	assert.Equal(t, initGo, pool.numOfGo())

	assert.LessOrEqual(t, initGo, coreGo)

	done := make(chan struct{})
	wait := make(chan struct{}, coreGo)

	// 稳定在initGo
	t.Log("XX")
	for i := int32(0); i < initGo; i++ {
		err := pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error {
			wait <- struct{}{}
			<-done
			return nil
		}))
		assert.NoError(t, err)
		t.Log("submit ", i)
	}

	t.Log("YY")
	for i := int32(0); i < initGo; i++ {
		<-wait
	}

	// 至少initGo个协程
	assert.GreaterOrEqual(t, pool.numOfGo(), initGo)

	t.Log("ZZ")

	// 逐步添加任务
	for i := int32(1); i <= coreGo-initGo; i++ {
		err := pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error {
			wait <- struct{}{}
			<-done
			return nil
		}))
		assert.NoError(t, err)
		<-wait
		t.Log("after wait coreGo", coreGo, i, pool.numOfGo())
	}

	t.Log("UU")

	assert.Equal(t, pool.numOfGo(), coreGo)
	close(done)

	// 等待最大空闲时间后稳定在initGo
	for pool.numOfGo() > initGo {
	}

	assert.Equal(t, initGo, pool.numOfGo())
}

func testExtendGoFromInitGoToCoreGoAndMaxGo(t *testing.T, initGo int32, queueSize int, coreGo, maxGo int32, maxIdleTime time.Duration, opts ...option.Option[OnDemandBlockTaskPool]) {

	opts = append(opts, WithCoreGo(coreGo), WithMaxGo(maxGo), WithMaxIdleTime(maxIdleTime))
	pool := testNewRunningStateTaskPool(t, int(initGo), queueSize, opts...)

	assert.Equal(t, initGo, pool.numOfGo())

	assert.LessOrEqual(t, initGo, coreGo)
	assert.LessOrEqual(t, coreGo, maxGo)

	done := make(chan struct{})
	wait := make(chan struct{}, maxGo)

	// 稳定在initGo
	t.Log("00")
	for i := int32(0); i < initGo; i++ {
		err := pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error {
			wait <- struct{}{}
			<-done
			return nil
		}))
		assert.NoError(t, err)
		t.Log("submit ", i)
	}
	t.Log("AA")
	for i := int32(0); i < initGo; i++ {
		<-wait
	}

	assert.GreaterOrEqual(t, pool.numOfGo(), initGo)

	t.Log("BB")

	// 逐步添加任务
	for i := int32(1); i <= coreGo-initGo; i++ {
		err := pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error {
			wait <- struct{}{}
			<-done
			return nil
		}))
		assert.NoError(t, err)
		<-wait
		t.Log("after wait coreGo", coreGo, i, pool.numOfGo())
	}

	t.Log("CC")

	assert.GreaterOrEqual(t, pool.numOfGo(), coreGo)

	for i := int32(1); i <= maxGo-coreGo; i++ {

		err := pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error {
			wait <- struct{}{}
			<-done
			return nil
		}))
		assert.NoError(t, err)
		<-wait
		t.Log("after wait maxGo", maxGo, i, pool.numOfGo())
	}

	t.Log("DD")

	assert.Equal(t, pool.numOfGo(), maxGo)
	close(done)

	// 等待最大空闲时间后稳定在initGo
	for pool.numOfGo() > initGo {
	}
	assert.Equal(t, initGo, pool.numOfGo())
}

func TestOnDemandBlockTaskPool_In_Closing_State(t *testing.T) {
	t.Parallel()

	t.Run("Shutdown —— 使TaskPool状态由Running变为Closing", func(t *testing.T) {
		t.Parallel()

		initGo, queueSize := 2, 4
		pool := testNewRunningStateTaskPool(t, initGo, queueSize)

		// 模拟阻塞提交
		n := initGo + queueSize*2
		eg := new(errgroup.Group)
		waitChan := make(chan struct{}, n)
		taskDone := make(chan struct{})
		for i := 0; i < n; i++ {
			eg.Go(func() error {
				return pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error {
					waitChan <- struct{}{}
					<-taskDone
					return nil
				}))
			})
		}
		for i := 0; i < initGo; i++ {
			<-waitChan
		}
		done, err := pool.Shutdown()
		assert.NoError(t, err)
		// Closing过程中Submit会报错间接证明TaskPool处于StateClosing状态
		assert.ErrorIs(t, eg.Wait(), errTaskPoolIsClosing)

		// 第二次调用
		done2, err2 := pool.Shutdown()
		assert.Nil(t, done2)
		assert.ErrorIs(t, err2, errTaskPoolIsClosing)
		assert.Equal(t, stateClosing, pool.internalState())

		assert.Equal(t, int32(initGo), pool.numOfGo())

		close(taskDone)
		<-done
		assert.Equal(t, stateStopped, pool.internalState())

		// 第一个Shutdown将状态迁移至StateStopped
		// 第三次调用
		done3, err := pool.Shutdown()
		assert.Nil(t, done3)
		assert.ErrorIs(t, err, errTaskPoolIsStopped)
	})

	t.Run("Shutdown —— 协程数按需扩展至maxGo，调用Shutdown成功后，所有协程运行完任务后可以自动退出", func(t *testing.T) {
		t.Parallel()

		initGo, coreGo, maxGo, maxIdleTime, queueBacklogRate := int32(1), int32(3), int32(5), 10*time.Millisecond, 0.1
		queueSize := int(maxGo)
		pool := testNewRunningStateTaskPool(t, int(initGo), queueSize, WithCoreGo(coreGo), WithMaxGo(maxGo), WithMaxIdleTime(maxIdleTime), WithQueueBacklogRate(queueBacklogRate))

		assert.LessOrEqual(t, initGo, coreGo)
		assert.LessOrEqual(t, coreGo, maxGo)

		taskDone := make(chan struct{})
		wait := make(chan struct{})

		for i := int32(0); i < maxGo; i++ {
			err := pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error {
				wait <- struct{}{}
				<-taskDone
				return nil
			}))
			assert.NoError(t, err)
		}

		// 提交任务后立即Shutdown
		shutdownDone, err := pool.Shutdown()
		assert.NoError(t, err)

		// 已提交的任务应该正常运行并能扩展至maxGo
		for i := int32(0); i < maxGo; i++ {
			<-wait
		}
		assert.Equal(t, maxGo, pool.numOfGo())

		// 让所有任务结束
		close(taskDone)
		<-shutdownDone

		// 用循环取代time.After/time.Sleep
		for pool.numOfGo() != 0 {

		}

		// 最终全部退出
		assert.Equal(t, int32(0), pool.numOfGo())
	})

	t.Run("Start", func(t *testing.T) {
		t.Parallel()

		pool, wait := testNewRunningStateTaskPoolWithQueueFullFilled(t, 2, 4)

		done, err := pool.Shutdown()
		assert.NoError(t, err)

		select {
		case <-done:
		default:
			assert.ErrorIs(t, pool.Start(), errTaskPoolIsClosing)
		}

		close(wait)
		<-done
		assert.Equal(t, stateStopped, pool.internalState())
	})

	// Submit()在状态中会报错，因为Closing是一个中间状态，故在上面的Shutdown间接测到了

	t.Run("ShutdownNow", func(t *testing.T) {
		t.Parallel()

		pool, wait := testNewRunningStateTaskPoolWithQueueFullFilled(t, 2, 4)

		done, err := pool.Shutdown()
		assert.NoError(t, err)

		select {
		case <-done:
		default:
			tasks, err := pool.ShutdownNow()
			assert.ErrorIs(t, err, errTaskPoolIsClosing)
			assert.Nil(t, tasks)
		}

		close(wait)
		<-done
		assert.Equal(t, stateStopped, pool.internalState())
	})
}

func TestOnDemandBlockTaskPool_In_Stopped_State(t *testing.T) {
	t.Parallel()

	t.Run("ShutdownNow —— 使TaskPool状态由Running变为Stopped", func(t *testing.T) {
		t.Parallel()

		initGo, queueSize := 2, 4
		pool, wait := testNewRunningStateTaskPoolWithQueueFullFilled(t, initGo, queueSize)

		// 模拟阻塞提交
		eg := new(errgroup.Group)
		for i := 0; i < queueSize; i++ {
			eg.Go(func() error {
				return pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error {
					<-wait
					return nil
				}))
			})
		}

		// 并发调用ShutdownNow
		result := make(chan ShutdownNowResult, 1)
		go func() {
			tasks, err := pool.ShutdownNow()
			result <- ShutdownNowResult{tasks: tasks, err: err}
			close(wait)
		}()

		// 阻塞的Submit在ShutdownNow后会报错间接证明TaskPool处于StateStopped状态
		assert.ErrorIs(t, eg.Wait(), errTaskPoolIsStopped)
		assert.Equal(t, stateStopped, pool.internalState())

		r := <-result
		assert.NoError(t, r.err)
		assert.Equal(t, queueSize, len(r.tasks))

		// 重复调用
		tasks, err := pool.ShutdownNow()
		assert.Nil(t, tasks)
		assert.ErrorIs(t, err, errTaskPoolIsStopped)
		assert.Equal(t, stateStopped, pool.internalState())
	})

	t.Run("ShutdownNow —— 工作协程数扩展至maxGo后，调用ShutdownNow成功，所有协程能够接收到信号", func(t *testing.T) {
		t.Parallel()

		initGo, coreGo, maxGo, maxIdleTime, queueBacklogRate := int32(1), int32(3), int32(5), 10*time.Millisecond, 0.1
		queueSize := int(maxGo)
		pool := testNewRunningStateTaskPool(t, int(initGo), queueSize, WithCoreGo(coreGo), WithMaxGo(maxGo), WithMaxIdleTime(maxIdleTime), WithQueueBacklogRate(queueBacklogRate))

		assert.LessOrEqual(t, initGo, coreGo)
		assert.LessOrEqual(t, coreGo, maxGo)

		taskDone := make(chan struct{})
		wait := make(chan struct{}, queueSize)

		for i := 0; i < queueSize; i++ {
			err := pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error {
				wait <- struct{}{}
				<-taskDone
				return nil
			}))
			assert.NoError(t, err)
		}

		tasks, err := pool.ShutdownNow()
		assert.NoError(t, err)
		assert.GreaterOrEqual(t, len(tasks), 0)

		// 让所有任务结束
		close(taskDone)

		// 用循环取代time.After/time.Sleep
		for pool.numOfGo() != 0 {
		}

		assert.Equal(t, int32(0), pool.numOfGo())
	})

	t.Run("Start", func(t *testing.T) {
		t.Parallel()

		pool := testNewStoppedStateTaskPool(t, 1, 1)
		assert.ErrorIs(t, pool.Start(), errTaskPoolIsStopped)
		assert.Equal(t, stateStopped, pool.internalState())
	})

	t.Run("Submit", func(t *testing.T) {
		t.Parallel()

		pool := testNewStoppedStateTaskPool(t, 1, 1)
		err := pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error { return nil }))
		assert.ErrorIs(t, err, errTaskPoolIsStopped)
		assert.Equal(t, stateStopped, pool.internalState())
	})

	t.Run("Shutdown", func(t *testing.T) {
		t.Parallel()

		pool := testNewStoppedStateTaskPool(t, 1, 1)
		done, err := pool.Shutdown()
		assert.Nil(t, done)
		assert.ErrorIs(t, err, errTaskPoolIsStopped)
		assert.Equal(t, stateStopped, pool.internalState())
	})
}

func testSubmitBlockingAndTimeout(t *testing.T, pool *OnDemandBlockTaskPool) {
	done := make(chan struct{})
	err := pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error {
		<-done
		return nil
	}))
	assert.NoError(t, err)

	n := cap(pool.queue) + 2
	eg := new(errgroup.Group)

	for i := 0; i < n; i++ {
		eg.Go(func() error {
			ctx, cancel := context.WithTimeout(context.Background(), time.Millisecond)
			defer cancel()
			return pool.Submit(ctx, TaskFunc(func(ctx context.Context) error {
				<-done
				return nil
			}))
		})
	}
	assert.ErrorIs(t, eg.Wait(), context.DeadlineExceeded)
	close(done)
}

func testSubmitValidTask(t *testing.T, pool *OnDemandBlockTaskPool) {

	err := pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error { return nil }))
	assert.NoError(t, err)

	err = pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error { panic("task panic") }))
	assert.NoError(t, err)

	err = pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error { return errors.New("fake error") }))
	assert.NoError(t, err)
}

type ShutdownNowResult struct {
	tasks []Task
	err   error
}

func testNewRunningStateTaskPool(t *testing.T, initGo int, queueSize int, opts ...option.Option[OnDemandBlockTaskPool]) *OnDemandBlockTaskPool {
	pool, _ := NewOnDemandBlockTaskPool(initGo, queueSize, opts...)
	assert.Equal(t, stateCreated, pool.internalState())
	assert.NoError(t, pool.Start())
	assert.Equal(t, stateRunning, pool.internalState())
	return pool
}

func testNewStoppedStateTaskPool(t *testing.T, initGo int, queueSize int) *OnDemandBlockTaskPool {
	pool := testNewRunningStateTaskPool(t, initGo, queueSize)
	tasks, err := pool.ShutdownNow()
	assert.NoError(t, err)
	assert.Equal(t, 0, len(tasks))
	assert.Equal(t, stateStopped, pool.internalState())
	return pool
}

func testNewRunningStateTaskPoolWithQueueFullFilled(t *testing.T, initGo int, queueSize int) (*OnDemandBlockTaskPool, chan struct{}) {
	pool := testNewRunningStateTaskPool(t, initGo, queueSize)
	wait := make(chan struct{})
	for i := 0; i < initGo+queueSize; i++ {
		err := pool.Submit(context.Background(), TaskFunc(func(ctx context.Context) error {
			<-wait
			return nil
		}))
		assert.NoError(t, err)
	}
	return pool, wait
}

func TestGroup(t *testing.T) {
	n := 10

	// g := &sliceGroup{members: make([]int, n, n)}
	g := &group{mp: make(map[int]int)}

	for i := 0; i < n; i++ {
		assert.False(t, g.isIn(i))
		g.add(i)
		assert.True(t, g.isIn(i))
		assert.Equal(t, int32(i+1), g.size())
	}

	assert.Equal(t, int32(n), g.size())

	for i := 0; i < n; i++ {
		g.delete(i)
		assert.Equal(t, int32(n-i-1), g.size())
	}

	assert.Equal(t, int32(0), g.size())

	assert.False(t, g.isIn(n+1))

	id := 100
	g.add(id)
	assert.Equal(t, int32(1), g.size())
	assert.True(t, g.isIn(id))
	g.delete(id)
	assert.Equal(t, int32(0), g.size())
}

func ExampleNewOnDemandBlockTaskPool() {
	p, _ := NewOnDemandBlockTaskPool(10, 100)
	_ = p.Start()
	// wg 只是用来确保任务执行的，你在实际使用过程中是不需要的
	var wg sync.WaitGroup
	wg.Add(1)
	_ = p.Submit(context.Background(), TaskFunc(func(ctx context.Context) error {
		fmt.Println("hello, world")
		wg.Done()
		return nil
	}))
	wg.Wait()
	// Output:
	// hello, world
}
```