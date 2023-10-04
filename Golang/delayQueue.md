## 延迟队列
延迟队列就是我把元素按优先级排序 ，最先过期的排在前面（过期时间短的），后过期的排后面。只需要有一个goroutine从该队列里面取元素，每一次取的时候，队首的元素如果已经过期了就返回给我，如果没有过期就阻塞。

https://github.com/gotomicro/ekit/pull/111

延迟队列的大概设计。
有两个设计，第一个设计在 queue_1.go 中，第二个设计在 queue_2.go 中。

两个设计的最大问题都是 sync.Condition 并没有提供一个 Wait(time) 的 API，即我希望有一个 Wait 的API，它接收一个时间参数，到点了就会自己醒来。

但是以为 sync.Condition 没有，所以我们必须要要利用 sleep 或者 ticker 来达成类似的问题。

两个实现的基本理念都是类似的：

入队的时候，如果新的元素被放过去了队首，那么就唤醒等待的 goroutine
出队的时候，如果队首的元素时间还没到，那么就 sleep。但是以为在 sleep 的期间可能有元素入队，所以实际上这个地方需要监听三个：
ctx 超时
sleep 到期
Signal 信号
注意的是，sleep 到期，或者收到 signal 信号，都需要考虑再次检测队首元素，确保元素的确已经过期了。

两者实现的差异在于：

- 第一种实现很难解决 ctx 超时之后， 调用者之前开启的 gorroutine 被唤醒的问题，以及 Dequeue 里面的 channel 关闭问题
- 第二种实现的问题在于开启了一个 goroutine，那么意味着我们可能需要引入 Close 方法，否则这个 goroutine 难以退出。
** 总结就是：**一切的根源都在于 sync.Condition 并不友好。所以另外一个思路是能不能利用 cgo 之类的工具，为 sync.Condition 提供额外的 Wait(time) 的方法



DelayQueue需求澄清

假定T的Dealy方法返回1s,请问这1s是指从T,Enqueue成功到Dequeue成功这整个过程1s,这1s包含了T在优先级队列中排队、以及Dequeue的阻塞过程.还是说只是在Dequeue T的时候阻塞1s,这个时间就没法预测了,假定T前面有10个元素每个都1s那调度到T的时候至少10s后了.

我认为应该是前者,从入队到出队这整个过程1s.. 从使用者的角度,我在t1时刻Enqueue,在t1+1s去Dequeue时应该能拿到T(即便有插队元素,多次调用也能快速拿到T),或者Enqueue后立即Dequeue能够阻塞住直到t1+1s返回T

延时队列和调用者被阻塞多久没有关系，也和入队出队多久没有关系。我举个例子，我设计了一个本地缓存，那么我可以将每一个元素丢进去优先级队列里面，Delay 返回的就是还有多久我这个缓存的键值对会过期。

因此这样一来，如果我们的队列本身一直阻塞，以至于这个键值对其实已经过期很久了，但是因为一直没有入队成功，也不能出队，那么就会导致无法及时删除过期键值对的问题。

另外一方面来说，严格的时间控制基本上是不现实的。如果用户希望避免阻塞引起这种问题，那么可以考虑使用无界的延时队列。但是即便如此，依旧会有时间不精确的问题，比如说我过期，但是过了几毫秒之后，才把我从队头取出来。但是这种不精确我觉得可以忍受。

```go
package queue

import (
	"errors"

	"github.com/gotomicro/ekit/internal/slice"

	"github.com/gotomicro/ekit"
)

var (
	ErrOutOfCapacity = errors.New("ekit: 超出最大容量限制")
	ErrEmptyQueue    = errors.New("ekit: 队列为空")
)

// PriorityQueue 是一个基于小顶堆的优先队列
// 当capacity <= 0时，为无界队列，切片容量会动态扩缩容
// 当capacity > 0 时，为有界队列，初始化后就固定容量，不会扩缩容
type PriorityQueue[T any] struct {
	// 用于比较前一个元素是否小于后一个元素
	compare ekit.Comparator[T]
	// 队列容量
	capacity int
	// 队列中的元素，为便于计算父子节点的index，0位置留空，根节点从1开始
	data []T
}

func (p *PriorityQueue[T]) Len() int {
	return len(p.data) - 1
}

// Cap 无界队列返回0，有界队列返回创建队列时设置的值
func (p *PriorityQueue[T]) Cap() int {
	return p.capacity
}

func (p *PriorityQueue[T]) IsBoundless() bool {
	return p.capacity <= 0
}

func (p *PriorityQueue[T]) isFull() bool {
	return p.capacity > 0 && len(p.data)-1 == p.capacity
}

func (p *PriorityQueue[T]) isEmpty() bool {
	return len(p.data) < 2
}

func (p *PriorityQueue[T]) Peek() (T, error) {
	if p.isEmpty() {
		var t T
		return t, ErrEmptyQueue
	}
	return p.data[1], nil
}

func (p *PriorityQueue[T]) Enqueue(t T) error {
	if p.isFull() {
		return ErrOutOfCapacity
	}

	p.data = append(p.data, t)
	node, parent := len(p.data)-1, (len(p.data)-1)/2
	for parent > 0 && p.compare(p.data[node], p.data[parent]) < 0 {
		p.data[parent], p.data[node] = p.data[node], p.data[parent]
		node = parent
		parent = parent / 2
	}

	return nil
}

func (p *PriorityQueue[T]) Dequeue() (T, error) {
	if p.isEmpty() {
		var t T
		return t, ErrEmptyQueue
	}

	pop := p.data[1]
	p.data[1] = p.data[len(p.data)-1]
	p.data = p.data[:len(p.data)-1]
	p.shrinkIfNecessary()
	p.heapify(p.data, len(p.data)-1, 1)
	return pop, nil
}

func (p *PriorityQueue[T]) shrinkIfNecessary() {
	if p.IsBoundless() {
		p.data = slice.Shrink[T](p.data)
	}
}

func (p *PriorityQueue[T]) heapify(data []T, n, i int) {
	minPos := i
	for {
		if left := i * 2; left <= n && p.compare(data[left], data[minPos]) < 0 {
			minPos = left
		}
		if right := i*2 + 1; right <= n && p.compare(data[right], data[minPos]) < 0 {
			minPos = right
		}
		if minPos == i {
			break
		}
		data[i], data[minPos] = data[minPos], data[i]
		i = minPos
	}
}

// NewPriorityQueue 创建优先队列 capacity <= 0 时，为无界队列，否则有有界队列
func NewPriorityQueue[T any](capacity int, compare ekit.Comparator[T]) *PriorityQueue[T] {
	sliceCap := capacity + 1
	if capacity < 1 {
		capacity = 0
		sliceCap = 64
	}
	return &PriorityQueue[T]{
		capacity: capacity,
		data:     make([]T, 1, sliceCap),
		compare:  compare,
	}
}
```

```go
import (
	"context"
	"fmt"
	"sync"
	"time"
	//"github.com/gotomicro/ekit/internal/queue"
)

// DelayQueue 延时队列
// 每次出队的元素必然都是已经到期的元素，即 Delay() 返回的值小于等于 0
// 延时队列本身对时间的精确度并不是很高，其时间精确度主要取决于 time.Timer
// 所以如果你需要极度精确的延时队列，那么这个结构并不太适合你。
// 但是如果你能够容忍至多在毫秒级的误差，那么这个结构还是可以使用的
type DelayQueue[T Delayable] struct {
	q             queue.PriorityQueue[T]
	mutex         *sync.Mutex
	dequeueSignal *cond
	enqueueSignal *cond
}

func NewDelayQueue[T Delayable](c int) *DelayQueue[T] {
	m := &sync.Mutex{}
	res := &DelayQueue[T]{
		q: *queue.NewPriorityQueue[T](c, func(src T, dst T) int {
			srcDelay := src.Delay()
			dstDelay := dst.Delay()
			if srcDelay > dstDelay {
				return 1
			}
			if srcDelay == dstDelay {
				return 0
			}
			return -1
		}),
		mutex:         m,
		dequeueSignal: newCond(m),
		enqueueSignal: newCond(m),
	}
	return res
}

func (d *DelayQueue[T]) Enqueue(ctx context.Context, t T) error {
	for {
		select {
		// 先检测 ctx 有没有过期
		case <-ctx.Done():
			return ctx.Err()
		default:
		}
		d.mutex.Lock()
		err := d.q.Enqueue(t)
		switch err {
		case nil:
			d.enqueueSignal.broadcast()
			return nil
		case queue.ErrOutOfCapacity:
			signal := d.dequeueSignal.signalCh()
			select {
			case <-ctx.Done():
				return ctx.Err()
			case <-signal:
			}
		default:
			d.mutex.Unlock()
			return fmt.Errorf("ekit: 延时队列入队的时候遇到未知错误 %w，请上报", err)
		}
	}
}

func (d *DelayQueue[T]) Dequeue(ctx context.Context) (T, error) {
	var timer *time.Timer
	defer func() {
		if timer != nil {
			timer.Stop()
		}
	}()
	for {
		select {
		// 先检测 ctx 有没有过期
		case <-ctx.Done():
			var t T
			return t, ctx.Err()
		default:
		}
		d.mutex.Lock()
		val, err := d.q.Peek()
		switch err {
		case nil:
			delay := val.Delay()
			if delay <= 0 {
				val, err = d.q.Dequeue()
				d.dequeueSignal.broadcast()
				// 理论上来说这里 err 不可能不为 nil
				return val, err
			}
			signal := d.enqueueSignal.signalCh()
			if timer == nil {
				timer = time.NewTimer(delay)
			} else {
				timer.Reset(delay)
			}
			select {
			case <-ctx.Done():
				var t T
				return t, ctx.Err()
			case <-timer.C:
				// 到了时间
				d.mutex.Lock()
				// 原队头可能已经被其他协程先出队，故再次检查队头
				val, err := d.q.Peek()
				if err != nil || val.Delay() > 0 {
					d.mutex.Unlock()
					continue
				}
				// 验证元素过期后将其出队
				val, err = d.q.Dequeue()
				d.dequeueSignal.broadcast()
				return val, err
			case <-signal:
				// 进入下一个循环。这里可能是有新的元素入队，也可能是到期了
			}
		case queue.ErrEmptyQueue:
			signal := d.enqueueSignal.signalCh()
			select {
			case <-ctx.Done():
				var t T
				return t, ctx.Err()
			case <-signal:
			}
		default:
			d.mutex.Unlock()
			var t T
			return t, fmt.Errorf("ekit: 延时队列出队的时候遇到未知错误 %w，请上报", err)
		}
	}
}

type Delayable interface {
	Delay() time.Duration
}

type cond struct {
	signal chan struct{}
	l      sync.Locker
}

func newCond(l sync.Locker) *cond {
	return &cond{
		signal: make(chan struct{}),
		l:      l,
	}
}

// broadcast 唤醒等待者
// 如果没有人等待，那么什么也不会发生
// 必须加锁之后才能调用这个方法
// 广播之后锁会被释放，这也是为了确保用户必然是在锁范围内调用的
func (c *cond) broadcast() {
	signal := make(chan struct{})
	old := c.signal
	c.signal = signal
	c.l.Unlock()
	close(old)
}

// signalCh 返回一个 channel，用于监听广播信号
// 必须在锁范围内使用
// 调用后，锁会被释放，这也是为了确保用户必然是在锁范围内调用的
func (c *cond) signalCh() <-chan struct{} {
	res := c.signal
	c.l.Unlock()
	return res
}
```