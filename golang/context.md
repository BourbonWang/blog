# 理解Go context

## 场景

1. 一个goroutine派生了多个子goroutine，通常使用WaitGroup来等待所有goroutine结束。而如果派生的子goroutine又派生出新的goroutine，这种情况下使用WaitGroup就不太容易，因为子goroutine个数不容易确定。

2. 上层任务取消后，所有的下层任务都会被取消；

   中间某一层的任务取消后，只会将当前任务的下层任务取消，而不会影响上层的任务以及同级任务。

   比如，一个获取用户信息的request，设置了10秒的超时时间。它创建了3个协程，分别获取好友列表，动态列表，订阅列表。而获取动态列表的协程，又创建了另外的协程分别获取评论动态和点赞动态，并为它们设置了5秒的超时时间。在request的10秒超时后，这些所有的子协程都应当退出。

3. 父子协程间往往可以共享一些变量。比如上面例子中，各个子任务都需要用户token等。

## context 

context用来在goroutine之间传递上下文，包括：取消信号，生存时间，关键的变量等。创建协程时，将context作为函数参数传入goroutine，就可以完成对goroutine的控制。

context是一个接口，定义如下：

```go
type Context interface {
    Deadline() (deadline time.Time, ok bool)
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{}
}
```

- `Deadline`会返回一个超时时间，如果没有设置超时，返回的 ok 为 false 。
- `Done`方法返回一个channel，当 Context 被撤销或过期时，该信道关闭，据此goroutine可以收到关闭请求。
- 当`Done`信道关闭后，`Err`方法表明 Context 退出的原因。
- `Value`可以让Goroutine共享一些数据，当然获得数据是协程安全的。但使用这些数据的时候要注意同步，比如返回了一个map，而这个map的读写则要加锁。

context包提供了几种context结构体，它们有不同的功能。

- emptyCtx：空context
- cancelCtx：可以被取消的context
- timerCtx：可以设置超时时间的context
- valueCtx：

![](https://oscimg.oschina.net/oscnet/b94500dd55067bc048c8e96a0f3492978e6.jpg)

## emptyCtx

`emptyCtx`是一个`int`类型的变量，但实现了`context`的接口。`emptyCtx`没有超时时间，不能取消，也不能存储任何额外信息，所以`emptyCtx`用来作为`context`树的根节点。定义如下：

```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}
```

但我们不会直接使用emptyCtx，而是使用`Background()`和`TODO()`方法。`Background`通常被用于主函数、初始化以及测试中，作为一个顶层的`context`，也就是说一般我们创建的`context`都是基于`Background`；而`TODO`是在不确定使用什么`context`的时候才会使用。定义如下：

```go
var (
    background = new(emptyCtx)
    todo       = new(emptyCtx)
)

func Background() Context {
    return background
}

func TODO() Context {
    return todo
}
```

## cancelCtx

### 使用

`WithCancel()`函数用来创建一个可取消的`context`，即`cancelCtx`。它会返回一个cancelCtx，以及一个取消函数 cancel，调用这个函数就可以让context退出。

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
```

`cancelCtx`取消时，会将后代节点中所有的`cancelCtx`都取消。

```go
func main() {
	ctx := context.Background()
	father(ctx)
}

func father(ctx context.Context) {
	cctx, cancel := context.WithCancel(ctx)
	for i := 0; i < 5; i++ {
		go son(cctx, i)
	}
	
    // do something
	time.Sleep(5 * time.Second)
    
	cancel()
    
	//do something
	time.Sleep(5 * time.Second)
}

func son(ctx context.Context, idx int) {
	for {
		select {
		case <-ctx.Done():
			fmt.Printf("goroutine %d return \n", idx)
			return
		default:
			// do something
		}
	}
}
```

### 实现 

**cancelCtx**

```go 
type cancelCtx struct {
	Context

	mu       sync.Mutex            // protects following fields
	done     chan struct{}         // created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```

children中记录了由此context派生的所有后代cancelCtx，此context被 cancel 时会把其中的所有后代都cancel掉。

cancelCtx与deadline和value无关，所以只需要实现Done()和Err()。

**Done()**

只需要返回成员变量即可。这里的成员变量done是懒加载的。

```go
func (c *cancelCtx) Done() <-chan struct{} {
	c.mu.Lock()
	if c.done == nil {
		c.done = make(chan struct{})
	}
	d := c.done
	c.mu.Unlock()
	return d
}
```

**Err()**

```go 
func (c *cancelCtx) Err() error {
	c.mu.Lock()
	err := c.err
	c.mu.Unlock()
	return err
}
```

**cancel()**

```go 
var closedchan = make(chan struct{})

func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    if err == nil {
        panic("context: internal error: missing cancel error")
    }
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return // already canceled
    }
    // 设置取消原因
    c.err = err
    //设置一个关闭的channel或者将done channel关闭，用以发送关闭信号
    if c.done == nil {
        c.done = closedchan
    } else {
        close(c.done)
    }
    // 将子节点context依次取消
    for child := range c.children {
        // NOTE: acquiring the child's lock while holding parent's lock.
        child.cancel(false, err)
    }
    c.children = nil
    c.mu.Unlock()

    if removeFromParent {
        // 将当前context节点从父节点上移除
        removeChild(c.Context, c)
    }
}
```

cancel() 作用是将自己与后代都撤销掉，后代存储在map中，自己存储在祖先节点的map中。

**WithCancel()**

```go
type CancelFunc func()

func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    c := newCancelCtx(parent)
    propagateCancel(parent, &c)
    return &c, func() { c.cancel(true, Canceled) }
}

// newCancelCtx returns an initialized cancelCtx.
func newCancelCtx(parent Context) cancelCtx {
    // 将parent作为父节点context生成一个新的子节点
    return cancelCtx{Context: parent}
}

func propagateCancel(parent Context, child canceler) {
    if parent.Done() == nil {
        // parent.Done()返回nil表明父节点以上的路径上没有可取消的context
        return // parent is never canceled
    }
    // 获取最近的类型为cancelCtx的祖先节点
    if p, ok := parentCancelCtx(parent); ok {
        p.mu.Lock()
        if p.err != nil {
            // parent has already been canceled
            child.cancel(false, p.err)
        } else {
            if p.children == nil {
                p.children = make(map[canceler]struct{})
            }
            // 将当前子节点加入最近cancelCtx祖先节点的children中
            p.children[child] = struct{}{}
        }
        p.mu.Unlock()
    } else {
        go func() {
            select {
            case <-parent.Done():
                child.cancel(false, parent.Err())
            case <-child.Done():
            }
        }()
    }
}

func parentCancelCtx(parent Context) (*cancelCtx, bool) {
    for {
        switch c := parent.(type) {
        case *cancelCtx:
            return c, true
        case *timerCtx:
            return &c.cancelCtx, true
        case *valueCtx:
            parent = c.Context
        default:
            return nil, false
        }
    }
}
```

调用CancelFunc()将执行上面的cancel( )。之前说到`cancelCtx`取消时，会将后代节点中所有的`cancelCtx`都取消，`propagateCancel`即用来将当前节点添加到祖先的cancelCtx的map中。

1. 如果`parent.Done()`返回`nil`，表明父节点以上的路径上没有 cancelCtx，不需要处理；
2. 如果在context链上找到`cancelCtx`类型的祖先节点，则判断这个祖先节点是否已经取消，如果已经取消就取消当前节点；否则将当前节点加入到祖先节点的`children`列表。
3. 如果没有`cancelCtx`类型的祖先节点，则开启一个协程，监听`parent.Done()`和`child.Done()`，一旦parent 被关闭，则将当前`context`取消。

## timerCtx

timerCtx内部使用cancelCtx和一个定时器实现，可以定时取消context。使用`WithDeadline()`返回一个timerCtx，并且设置过期时间 d，d应早于parent的过期时间。

```go 
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)
```

也可以使用`WithTimeout()`创建，不同的是，它是设置相当于当前时间的timeout

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) 
```

例子：

```go
func main() {
	tctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	for i := 0; i < 5; i++ {
		go do(tctx, i)
	}

	defer cancel()
	
	select {
	case <-tctx.Done():
		time.Sleep(1 * time.Second)
		fmt.Println("main process exit!")
	}
}

func do(ctx context.Context, idx int) {
	for {
		select {
		case <-ctx.Done():
			fmt.Printf("goroutine %d return \n", idx)
			return
		default:
			// do something
		}
	}
}
```

设置了5秒的超时时间，然后将context传入goroutine。5秒后，context自动撤销，goroutine也退出。

```
// output:
goroutine 2 return 
goroutine 1 return 
goroutine 0 return 
goroutine 4 return 
goroutine 3 return 
main process exit!
```

### 实现

**timerCtx**

```go
type timerCtx struct {
    cancelCtx
    timer *time.Timer // Under cancelCtx.mu.

    deadline time.Time
}

func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
    return c.deadline, true
}

func (c *timerCtx) cancel(removeFromParent bool, err error) {
    将内部的cancelCtx取消
    c.cancelCtx.cancel(false, err)
    if removeFromParent {
        // Remove this timerCtx from its parent cancelCtx's children.
        removeChild(c.cancelCtx.Context, c)
    }
    c.mu.Lock()
    if c.timer != nil {
        取消计时器
        c.timer.Stop()
        c.timer = nil
    }
    c.mu.Unlock()
}
```

`timerCtx`内部使用`cancelCtx`实现取消，另外使用定时器`timer`和过期时间`deadline`实现定时取消的功能。`timerCtx`在调用`cancel`方法，会先将内部的`cancelCtx`取消，如果需要则将自己从`cancelCtx`祖先节点上移除，最后取消计时器。

**WithDeadline**

```go
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
    if cur, ok := parent.Deadline(); ok && cur.Before(d) {
        // The current deadline is already sooner than the new one.
        return WithCancel(parent)
    }
    c := &timerCtx{
        cancelCtx: newCancelCtx(parent),
        deadline:  d,
    }
    // 建立新建context与可取消context祖先节点的取消关联关系
    propagateCancel(parent, c)
    dur := time.Until(d)
    if dur <= 0 {
        c.cancel(true, DeadlineExceeded) // deadline has already passed
        return c, func() { c.cancel(false, Canceled) }
    }
    c.mu.Lock()
    defer c.mu.Unlock()
    if c.err == nil {
        c.timer = time.AfterFunc(dur, func() {
            c.cancel(true, DeadlineExceeded)
        })
    }
    return c, func() { c.cancel(true, Canceled) }
}
```

1. 如果父节点`parent`有过期时间并且过期时间早于给定时间`d`，那么新建的子`context`无需设置过期时间，使用`WithCancel`创建一个可取消的`context`即可；
2. 否则，就要利用`parent`和过期时间`d`创建一个定时取消的`timerCtx`，并建立新建`context`与可取消`context`祖先节点的取消关联关系，接下来判断当前时间距离过期时间`d`的时长`dur`：
3. 如果`dur`小于0，即当前已经过了过期时间，则直接取消新建的`timerCtx`，原因为`DeadlineExceeded`；
4. 否则，为新建的`timerCtx`设置定时器，一旦到达过期时间即取消当前`timerCtx`。

**WithTimeout**

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout))
}
```

## valueCtx

### 使用

valueCtx可以在context里传递数据。它只是在context的基础上增加了一个key-value对。

```go
type valueCtx struct {
	Context
	key, val interface{}
}
```

使用 `WithValue()`来创建一个valueCtx，在创建时，可以输入要存储的key-value

```go
func WithValue(parent Context, key, val interface{}) Context
```

虽然创建时只能设置一组k-v数据，但是，context是链式的，子context基于父context，所以子context也可以访问到所有祖先context链路上存储的数据。

例子：

```go
func main() {
	go gen1(context.Background())

	time.Sleep(5 * time.Second)
}

func gen1(ctx context.Context) {
	vctx := context.WithValue(ctx, "k1", "v1")
	go gen2(vctx)
}

func gen2(ctx context.Context) {
	vctx := context.WithValue(ctx, "k2", "v2")
	go gen3(vctx)
}

func gen3(ctx context.Context) {
    fmt.Println("k1: ", ctx.Value("k1"))  // k1: v1
    fmt.Println("k2: ", ctx.Value("k2"))  // k2: v2
    fmt.Println("kn: ", ctx.Value("kn"))  // kn: <nil>
}
```

### 实现

由于valueCtx既不需要cancel，也不需要deadline，那么只需要实现Value()接口即可。

```go
func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
```

如果当前context查找不到数据时，会向父context查找。

**WithValue()**

```go
func WithValue(parent Context, key, val interface{}) Context {
    if key == nil {
        panic("nil key")
    }
    if !reflect.TypeOf(key).Comparable() {
        panic("key is not comparable")
    }
    return &valueCtx{parent, key, val}
}
```

## 总结

context用于父子协程之间的同步取消信号，是一种协程调度的方式。上游协程仅仅是通知下游协程取消，但不会干涉下游协程的执行。协程在收到ctx参数后，应当监听 context.Done()，然后自行决定后续的处理操作。使用时，应当针对一个请求创建一个Context变量；在请求处理结束后，撤销此ctx变量，释放资源。官方提供的三种context可以互为父节点，从而可以组合成不同的应用形式。

在使用时需要注意的地方：

- 不要将 Context 塞到结构体里。直接将 Context 类型作为函数的第一参数，而且一般都命名为 ctx。
- 不要向函数传入一个 nil 的 context，如果你实在不知道传什么，标准库给你准备好了一个 context：TODO。
- 不要把本应该作为函数参数的类型塞到 context 中，context 存储的应该是一些共同的数据。例如：登陆的 session、cookie 等。
- 同一个 context 可能会被传递到多个 goroutine，别担心，context 是并发安全的。