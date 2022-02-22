---
title: go的context使用
tags:
  - null
categories:
  - technical
  - null
toc: true
declare: true
date: 2022-02-10 19:03:31
---

> 学习来源：
>
> * https://juejin.cn/post/6844904070667321357

# 一、context的作用/为什么需要context

核心作用就是主动关闭多个子协程的执行(`cancel`)，避免采用`channel`的方式

<!-- more -->

首先考虑一个简单的场景：主进程在一定时间后通知子进程关闭:

```go
func main() {
    messages := make(chan int, 10)
    done := make(chan bool)

    defer close(messages)
    // consumer
    go func() {
        ticker := time.NewTicker(1 * time.Second)
        for _ = range ticker.C {
            select {
            case <-done:				// 子进程结束
                fmt.Println("child process interrupt...")
                return
            default:
                fmt.Printf("send message: %d\n", <-messages)
            }
        }
    }()

    // producer
    for i := 0; i < 10; i++ {
        messages <- i
    }
    time.Sleep(5 * time.Second)
    close(done)										// 等待5秒，通知结束
    time.Sleep(1 * time.Second)
    fmt.Println("main process exit!")
}
```

这样看似没什么问题，但是如果场景变得复杂，就不是这么简单了。

**如果我们可以在简单的通知上附加传递额外的信息来控制取消：为什么取消，或者有一个它必须要完成的最终期限，更或者有多个取消选项，我们需要根据额外的信息来判断选择执行哪个取消选项。**

如果又有一些父进程t1、t2\...，此时希望这个消费者/子进程再等待这些t1、t2…进程的关闭信号,并且这些期限也是不相同的，那么每个对应关系都创建一个`channel`这是非常麻烦的。

**我们需要一种主动结束子协程的机制：**

* **上层取消后，所有下层协程都会取消**
* **中间某一层取消后，只会影响其所有下层而不会影响上层**

# 二、context是什么

是一个接口，具体形式如下：

```go
type Context interface {

    Deadline() (deadline time.Time, ok bool)

    Done() <-chan struct{}

    Err() error

    Value(key interface{}) interface{}
}
```

`Context`接口包含四个方法：

- `Deadline`返回绑定当前`context`的任务被取消的**截止时间**；如果没有设定期限，将返回`ok == false`。

  > <font color='#39b54a'>用于返回当前设置取消的截止时间</font>

- `Done` 当绑定当前`context`的任务被取消时，将返回一个关闭的`channel`；如果当前`context`不会被取消，将返回`nil`。

  > <font color='#39b54a'>用于在协程内部判断是否已经取消，与上面例子自定义的channel相同功能</font>

- `Err` 如果`Done`返回的`channel`没有关闭，将返回`nil`;如果`Done`返回的`channel`已经关闭，将返回非空的值表示**任务结束的原因**。

  如果是`context`被取消，`Err`将返回`Canceled`；如果是`context`超时，`Err`将返回`DeadlineExceeded`。

  > <font color='#39b54a'>用于解释当前协程被取消的原因</font>

- `Value` 返回`context`存储的键值对中当前`key`对应的值，如果没有对应的`key`,则返回`nil`。

  > <font color='#39b54a'>用于获取当前任务树的额外信息，获取key的过程就是当前协程构成的树（每个节点/协程都可能携带一组kv）不断向上寻找，找到根节点都没有就返回nil</font>

# 三、类别

## 1、emptyCtx类

`emptyCtx`是一个`int`类型的变量，但实现了`context`的接口。`emptyCtx`没有超时时间，不能取消，也不能存储任何额外信息，<font color='#39b54a'>**所以`emptyCtx`用来作为`context`树的根节点。**</font>

但我们一般不会直接使用`emptyCtx`，而是使用由`emptyCtx`实例化的两个变量，分别可以通过调用`Background`和`TODO`方法得到，但这两个`context`在实现上是一样的。

`Backgroud`与`TODO`两个emptyctx的区别：

* `Background`通常被用于主函数、初始化以及测试中，作为一个顶层的`context`，也就是说一般我们创建的`context`都是基于`Background`；
* `TODO`是在不确定使用什么`context`的时候才会使用。

## 2、valueCtx类

结构体：

```go
type valueCtx struct {
    Context
    key, val interface{}
}

func (c *valueCtx) Value(key interface{}) interface{} {
    if c.key == key {
        return c.val
    }
    return c.Context.Value(key)
}
```

`valueCtx`利用一个`Context`类型的变量来表示父节点`context`，**所以当前`context`继承了父`context`的所有信息；**

`valueCtx`类型还携带一组键值对，也就是说这种`context`可以携带额外的信息。`valueCtx`实现了`Value`方法，用以在`context`链路上获取`key`对应的值，如果当前`context`上不存在需要的`key`,会沿着`context`链向上寻找`key`对应的值，直到根节点。

方法：

`witchValue`：添加键值对

```go
func WithValue(parent Context, key, val interface{}) Context {
    if key == nil {
        panic("nil key")
    }
    if !reflect.TypeOf(key).Comparable() {
        panic("key is not comparable")
    }
    return &valueCtx{parent, key, val}		// 以父节点，重新创建一个子节点, 添加在子节点上
}
```

这里添加键值对不是在原`context`结构体上直接添加，**而是以此`context`作为父节点，重新创建一个新的`valueCtx`子节点**，将键值对添加在子节点上，由此形成一条`context`链。获取`value`的过程就是在这条`context`链上由尾部上前搜寻：

![image-20220219231328224](http://xwjpics.gumptlu.work/qinniu_uPic/image-20220219231328224.png)

## 3、cancelCtx类

结构体：

```go
type cancelCtx struct {
    Context

    mu       sync.Mutex            // protects following fields
    done     chan struct{}         // created lazily, closed by first cancel call
    children map[canceler]struct{} // set to nil by the first cancel call
    err      error                 // set to non-nil by the first cancel call
}

type canceler interface {
    cancel(removeFromParent bool, err error)
    Done() <-chan struct{}
}
```

跟`valueCtx`类似，`cancelCtx`中也有一个`context`变量作为父节点；变量`done`表示一个`channel`，用来表示传递关闭信号；`children`表示一个`map`，存储了当前`context`节点下的子节点；`err`用于存储错误信息表示任务结束的原因。

cancelCtx实现了canceler接口的`cancele()`、`Done()`方法以及额外的`Err()`方法：

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

func (c *cancelCtx) Err() error {
    c.mu.Lock()
    err := c.err
    c.mu.Unlock()
    return err
}

func (c *cancelCtx) cancel(removeFromParent bool, err error) {
    if err == nil {
        panic("context: internal error: missing cancel error")
    }
    c.mu.Lock()
    if c.err != nil {
        c.mu.Unlock()
        return // already canceled
    }
    // 设置取消原因, 传入的err就是取消原因
    c.err = err
    // 设置一个关闭的channel或者将done channel关闭，用以发送关闭信号
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

`cancelCtx`类型的`context`在调用`cancel`方法时会设置取消原因，将`done channel`设置为一个关闭`channel`或者关闭`channel`，然后将子节点`context`依次取消，如果有需要还会将当前节点从父节点上移除。

方法：

### WithCancel	

 `WithCancel`函数用来创建一个可取消的`context`，即`cancelCtx`类型的`context`。

`WithCancel`返回一个`context`和一个`CancelFunc`，调用`CancelFunc`即可触发`cancel`操作。直接看源码：

```go
type CancelFunc func()

// Canceled is the error returned by Context.Err when the context is canceled.
var Canceled = errors.New("context canceled")

func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
    c := newCancelCtx(parent)				// 创建一个新的CancelCtx
    propagateCancel(parent, &c)			// 链接上一个祖先节点
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
      	// 开启一个协程判断祖先节点是否取消
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

之前说到`cancelCtx`取消时，会将后代节点中所有的`cancelCtx`都取消，`propagateCancel`即用来建立当前节点与祖先节点这个取消关联逻辑。

1. 如果`parent.Done()`返回`nil`，表明父节点以上的路径上没有可取消的`context`，不需要处理；
2. 如果在`context`链上找到到`cancelCtx`类型的祖先节点，则判断这个祖先节点是否已经取消，如果已经取消就取消当前节点；否则将当前节点加入到祖先节点的`children`列表。
3. 否则开启一个协程，监听`parent.Done()`和`child.Done()`，一旦`parent.Done()`返回的`channel`关闭，即`context`链中某个祖先节点`context`被取消，则将当前`context`也取消。

这里或许有个疑问，为什么是祖先节点而不是父节点？这是因为当前`context`链可能是这样的：

<img src="http://xwjpics.gumptlu.work/qinniu_uPic/image-20220219235230794.png" alt="image-20220219235230794" style="zoom:50%;" />

当前`cancelCtx`的父节点`context`并不是一个可取消的`context`，也就没法记录`children`。

## 4、timerCtx类

`timerCtx`是一种基于`cancelCtx`的`context`类型，从字面上就能看出，这是一种可以定时取消的`context`。

```go
type timerCtx struct {
    cancelCtx
    timer *time.Timer // Under cancelCtx.mu.

    deadline time.Time
}

func (c *timerCtx) Deadline() (deadline time.Time, ok bool) {
    return c.deadline, true
}

// 会自动定时取消
func (c *timerCtx) cancel(removeFromParent bool, err error) {
    // 将内部的cancelCtx取消
    c.cancelCtx.cancel(false, err)
    if removeFromParent {
        // Remove this timerCtx from its parent cancelCtx's children.
        removeChild(c.cancelCtx.Context, c)
    }
    c.mu.Lock()
    if c.timer != nil {
        // 取消计时器
        c.timer.Stop()
        c.timer = nil
    }
    c.mu.Unlock()
}
```

`timerCtx`内部使用`cancelCtx`实现取消，另外使用定时器`timer`和过期时间`deadline`实现定时取消的功能。`timerCtx`在调用`cancel`方法，会先将内部的`cancelCtx`取消，如果需要则将自己从`cancelCtx`祖先节点上移除，最后取消计时器。

创建方法：

### WithDeadline

 `WithDeadline`返回一个基于`parent`的可取消的`timerCtx`，并且其过期时间`deadline`不晚于所设置时间`d`。

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

1. 如果父节点`parent`有过期时间并且过期时间早于给定时间`d`，那么新建的子节点`context`无需设置过期时间，使用`WithCancel`创建一个可取消的`context`即可；
2. 否则，就要利用`parent`和过期时间`d`创建一个定时取消的`timerCtx`，并建立新建`context`与可取消`context`祖先节点的取消关联关系，接下来判断当前时间距离过期时间`d`的时长`dur`：
3. 如果`dur`小于0，即当前已经过了过期时间，则直接取消新建的`timerCtx`，原因为`DeadlineExceeded`；
4. 否则，为新建的`timerCtx`设置定时器，一旦到达过期时间即取消当前`timerCtx`。

### WithTimeout

与`WithDeadline`类似，`WithTimeout`也是创建一个定时取消的`context`，只不过`WithDeadline`是**接收一个过期时间点**，而`WithTimeout`接收一个相对当前时间的过期时长`timeout`:

```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
    return WithDeadline(parent, time.Now().Add(timeout))
}
```

# 四、context的使用

首先使用`context`实现文章开头`done channel`的例子来示范一下如何更优雅实现协程间取消信号的同步：

```go
func main() {
    messages := make(chan int, 10)

    // producer
    for i := 0; i < 10; i++ {
        messages <- i
    }

    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)

    // consumer
    go func(ctx context.Context) {
        ticker := time.NewTicker(1 * time.Second)
        for _ = range ticker.C {
            select {
            case <-ctx.Done():
                fmt.Println("child process interrupt...")
                return
            default:
                fmt.Printf("send message: %d\n", <-messages)
            }
        }
    }(ctx)

    defer close(messages)
    defer cancel()

    select {
    case <-ctx.Done():
        time.Sleep(1 * time.Second)
        fmt.Println("main process exit!")
    }
}
```

阅读过`net/http`包源码的朋友可能注意到在实现`http server`时就用到了`context`, 下面简单分析一下。

1、首先`Server`在开启服务时会创建一个`valueCtx`,存储了`server`的相关信息，之后每建立一条连接就会开启一个协程，并携带此`valueCtx`。

```go
func (srv *Server) Serve(l net.Listener) error {

    ...

    var tempDelay time.Duration     // how long to sleep on accept failure
    baseCtx := context.Background() // base is always background, per Issue 16220
    ctx := context.WithValue(baseCtx, ServerContextKey, srv)
    for {
        rw, e := l.Accept()

        ...

        tempDelay = 0
        c := srv.newConn(rw)
        c.setState(c.rwc, StateNew) // before Serve can return
        go c.serve(ctx)
    }
}
```

2、建立连接之后会基于传入的`context`创建一个`valueCtx`用于存储本地地址信息，之后在此基础上又创建了一个`cancelCtx`，然后开始从当前连接中读取网络请求，每当读取到一个请求则会将该`cancelCtx`传入，用以传递取消信号。一旦连接断开，即可发送取消信号，取消所有进行中的网络请求。

```go
func (c *conn) serve(ctx context.Context) {
    c.remoteAddr = c.rwc.RemoteAddr().String()
    ctx = context.WithValue(ctx, LocalAddrContextKey, c.rwc.LocalAddr())
    ...

    ctx, cancelCtx := context.WithCancel(ctx)
    c.cancelCtx = cancelCtx
    defer cancelCtx()

    ...

    for {
        w, err := c.readRequest(ctx)

        ...

        serverHandler{c.server}.ServeHTTP(w, w.req)

        ...
    }
}
```

3、读取到请求之后，会再次基于传入的`context`创建新的`cancelCtx`,并设置到当前请求对象`req`上，同时生成的`response`对象中`cancelCtx`保存了当前`context`取消方法。

```go
func (c *conn) readRequest(ctx context.Context) (w *response, err error) {

    ...

    req, err := readRequest(c.bufr, keepHostHeader)

    ...

    ctx, cancelCtx := context.WithCancel(ctx)
    req.ctx = ctx

    ...

    w = &response{
        conn:          c,
        cancelCtx:     cancelCtx,
        req:           req,
        reqBody:       req.Body,
        handlerHeader: make(Header),
        contentLength: -1,
        closeNotifyCh: make(chan bool, 1),

        // We populate these ahead of time so we're not
        // reading from req.Header after their Handler starts
        // and maybe mutates it (Issue 14940)
        wants10KeepAlive: req.wantsHttp10KeepAlive(),
        wantsClose:       req.wantsClose(),
    }

    ...
    return w, nil
}
```

这样处理的目的主要有以下几点：

- 一旦请求超时，即可中断当前请求；
- 在处理构建`response`过程中如果发生错误，可直接调用`response`对象的`cancelCtx`方法结束当前请求；
- 在处理构建`response`完成之后，调用`response`对象的`cancelCtx`方法结束当前请求。

在整个`server`处理流程中，使用了一条`context`链贯穿`Server`、`Connection`、`Request`，不仅将上游的信息共享给下游任务，同时实现了上游可发送取消信号取消所有下游任务，而下游任务自行取消不会影响上游任务。

# 总结

`context`主要用于父子任务之间的同步取消信号，本质上是一种协程调度的方式。另外在使用`context`时有两点值得注意：

* **`context`主要用于父子协程之间的取消信号的传递，并且实现了当前层的取消只会影响所有下层节点而不影响上层**
* **上游任务仅仅使用`context`通知下游任务不再需要，但不会直接干涉和中断下游任务的执行，由下游任务自行决定后续的处理操作，也就是说`context`的取消操作是无侵入的；**
* **`context`是线程安全的，因为`context`本身是不可变的（`immutable`），因此可以放心地在多个协程中传递使用。**
