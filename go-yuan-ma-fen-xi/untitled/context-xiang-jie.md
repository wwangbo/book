# context详解

### 介绍

在Go服务器中，每个传入请求都在其自己的goroutine中进行处理。 请求处理程序通常会启动其他goroutine来访问后端，例如数据库和RPC服务。 处理请求的goroutine集合通常需要访问特定于请求的值，例如最终用户的身份，授权令牌和请求的期限。 当一个请求被取消或超时时，处理该请求的所有goroutine应该迅速退出，以便系统可以回收他们正在使用的任何资源。

在Google，我们开发了一个上下文包，可以轻松地跨API边界将请求范围的值、取消信号和截止日期传递给处理请求的所有goroutine。 该软件包可作为上下文公开使用。

### context

Context是一个接口，它的定义如下：

```text
type Context interface {
    Done() <-chan struct{}
​
    Err() error
​
    Deadline() (deadline time.Time, ok bool)
​
    Value(key interface{}) interface{}
}
```

```text
type emptyCtx int //使用int是为了减轻垃圾回收的压力
​
func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
  return
}
​
func (*emptyCtx) Done() <-chan struct{} {
  return nil
}
​
func (*emptyCtx) Err() error {
  return nil
}
​
func (*emptyCtx) Value(key interface{}) interface{} {
  return nil
}
​
func (e *emptyCtx) String() string {
  switch e {
  case background:
    return "context.Background"
  case todo:
    return "context.TODO"
  }
  return "unknown empty Context"
}
```

```text
func Background() Context
func TODO() Context
```

```text
type cancelCtx struct {
  Context
​
  mu       sync.Mutex            // protects following fields
  done     chan struct{}         // created lazily, closed by first cancel call
  children map[canceler]struct{} // set to nil by the first cancel call
  err      error                 // set to non-nil by the first cancel call
}
```

```text
type timerCtx struct {
  cancelCtx
  timer *time.Timer // Under cancelCtx.mu.
​
  deadline time.Time
}
```

```text
type valueCtx struct {
  Context
  key, val interface{}
}
```

```text
background = new(emptyCtx)
func Background() Context {
  return background
}
```

```text
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
  c := newCancelCtx(parent)
  // 向parent context注册cancel
  propagateCancel(parent, &c)
  return &c, func() { c.cancel(true, Canceled) }
}
func newCancelCtx(parent Context) cancelCtx {
  return cancelCtx{Context: parent}
}
```

```text
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc) {
  return WithDeadline(parent, time.Now().Add(timeout))
}
func WithDeadline(parent Context, d time.Time) (Context, CancelFunc) {
  if cur, ok := parent.Deadline(); ok && cur.Before(d) {
    // 如果parent有deadline，并且比自己的更早，直接cancel
    return WithCancel(parent)
  }
  // 创建自己的timerCtx并设置deadline
  c := &timerCtx{
    cancelCtx: newCancelCtx(parent),
    deadline:  d,
  }
  // 注册cancel到最近的父节点
  propagateCancel(parent, c)
  dur := time.Until(d) //获取deadline时间
  if dur <= 0 { //deadline已经触发则执行cancel
    c.cancel(true, DeadlineExceeded) // deadline has already passed
    return c, func() { c.cancel(false, Canceled) }
  }
  c.mu.Lock()
  defer c.mu.Unlock()
  if c.err == nil {
    // 设置过dur时间后自动cancel
    c.timer = time.AfterFunc(dur, func() {
      c.cancel(true, DeadlineExceeded)
    })
  }
  return c, func() { c.cancel(true, Canceled) }
}
```

```text
func WithValue(parent Context, key, val interface{}) Context {
  if key == nil {
    panic("nil key")
  }
  if !reflectlite.TypeOf(key).Comparable() {//key必须是可比较的
    panic("key is not comparable")
  }
  return &valueCtx{parent, key, val}
}
func (c *valueCtx) Value(key interface{}) interface{} {
  if c.key == key {// 如果当前 context 的 key 和要找寻的 key 一样，则返回 value
    return c.val
  }
  return c.Context.Value(key)// 如果不是，则调用 parent 的 Value()
}
```

```text
// 向parent context注册cancel，当parent执行cancel时，同时会进行child cancel
func propagateCancel(parent Context, child canceler) {
  if parent.Done() == nil {
    return // parent is never canceled
  }
  if p, ok := parentCancelCtx(parent); ok {
    p.mu.Lock()
    if p.err != nil {
      // parent has already been canceled，则标记child取消
      child.cancel(false, p.err)
    } else {
      if p.children == nil {//parent没有child，先初始化
        p.children = make(map[canceler]struct{})
      }
      p.children[child] = struct{}{} //加到parent中
    }
    p.mu.Unlock()
  } else {
  // 如果没有找到最近的cancelContext，那么说明自己时parent，启动协程，等待当前节点收到取消信号，取消child
    go func() {
      select {
      case <-parent.Done():
        child.cancel(false, parent.Err())
      case <-child.Done():
      }
    }()
  }
}
```

```text
func parentCancelCtx(parent Context) (*cancelCtx, bool) {//找到最近的parent cancelCtx
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

```text
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

```text
var closedchan = make(chan struct{})
​
func init() {
  close(closedchan)
}
​
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
  if err == nil {
    panic("context: internal error: missing cancel error")
  }
  c.mu.Lock()
  if c.err != nil {
    c.mu.Unlock()
    return // already canceled
  }
  c.err = err
  if c.done == nil {
    c.done = closedchan
  } else {
    close(c.done)
  }
  for child := range c.children {
    // NOTE: acquiring the child's lock while holding parent's lock.
    child.cancel(false, err)
  }
  c.children = nil
  c.mu.Unlock()
​
  if removeFromParent {
    removeChild(c.Context, c)
  }
}
```

```text
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
  if err == nil {
    panic("context: internal error: missing cancel error")
  }
  c.mu.Lock()
  if c.err != nil {
    c.mu.Unlock()
    return // already canceled
  }
  c.err = err
  if c.done == nil {
    c.done = closedchan
  } else {
    close(c.done)
  }
  for child := range c.children {
    // NOTE: acquiring the child's lock while holding parent's lock.
    child.cancel(false, err)
  }
  c.children = nil
  c.mu.Unlock()
​
  if removeFromParent {
    removeChild(c.Context, c)
  }
}
```

```text
func (c *timerCtx) cancel(removeFromParent bool, err error) {
  c.cancelCtx.cancel(false, err)
  if removeFromParent {
    // Remove this timerCtx from its parent cancelCtx's children.
    removeChild(c.cancelCtx.Context, c)
  }
  c.mu.Lock()
  if c.timer != nil {
    c.timer.Stop()
    c.timer = nil
  }
  c.mu.Unlock()
}
```

对于WithValue来说，不可以设置nil key，key的类型必须是可比较的，否则会panic

对于WithTimeout（WithDeadline）来说，只是基于WithCancel基础上增加了定时cancel

* 初始化context
* 将自己注册到parent上
* 如果没有parent，自己作为parent，并监听cancel信号

创建一个cancel需要一下一下步骤

### 总结

我们知道，timeCtx的cancel和cancelCtx的不同在于，timeCtx是通过定时器调用的，而cancel是程序员自己调用的，timeCtx的实现如下:

timeCtx取消是WithTimeout中取消的，而WithTimeout是通过WithDeadline来实现的，。

cancelCtx取消是自顶向下的，遍历所有children context，依次取消，其实现如下:

### context是如何取消的

其中parent调用的cancel方法实现如下：

可以看到，如果cancelCtx的done channel里面有值的话，则会进行相应的操作。（当cancel的时候会往done里面写东西），此处为了保证并发安全加了锁。

对于非cancelCtx或timerCtx的context来说，调用parent和child的done方法，其实现如下：

parentCancelCtx根据parent context的类型，返回相应的bool值，bool值为真时需要建立parent对应的children，并保存parent-&gt;child映射关系。valueCtx类型会一直向上寻找

对于parentCancelCtx函数来说，如果传入的context是cancelCtx或者timerCtx，返回true和cancelCtx对象.对于valueCtx来说，其返回值跟传入valueCtx的context的类型有关。

这里的注释容易引起误解，并不是说parent永远不会被取消，而是如果parent被取消了，说明你当前parent的所有子节点肯定已经被cancel掉了，自然不需要继续执行了（当然，root节点永远不会被cancel）。

下面分析一下propagateCancel函数干了哪些事：

无论是WithCancel、WithTimeout还是WithValue，实际上都是在父子context之间进行相应的信息传递。他们实际上都是使用propagateCancel进行相应信息的传递。

### context是如何传递的

WithValue实现如下:

WithTimeout实现如下:

可以看到WithCancel方法先去new一个cancelCtx的对象，把父context传入到cancelCtx对象里，然后去执行Can cel信号的传播。

其中WithCancel实现如下:

WithCancel和WithTimeout返回派生的Context值，该值可以比父Context早被取消。 请求处理程序返回时，通常会取消与传入请求关联的上下文。 使用多个副本时，WithCancel对于取消冗余请求也很有用。 WithTimeout对于设置对后端服务器的请求的截止日期很有用：

Background是Context树的root节点，永远不会被取消，其定义如下

context包提供了从已有的Context对象中生成子Context对象，他们构成一棵树，当Context被取消，所有Context的子节点也会被cancel掉。

### context的生成

1. valueCtx

   valueCtx用来进行值的传递

timeCtx内部使用的cancelCtx,另外加上了定时器以实现定时cancel的功能，代码如下:

1. timeCtx

其使用sync.Mutex保证并发安全，使用done用来实现cancel通知

cancelCtx的cancel是手工cancel、超时cancel的内部实现,代码如下:

1. cancelCtx

没有考虑是否传递、怎么传递的时候使用TODO，其他情况下使用Background

context包提供了两个初始化emptyCtx的方式，分别是：

1. emptyCtx是context的源头，它一个永远不会被取消，没有值，没有deadline的非结构体类型，它的定义与实现如下：

context共有emptyCtx,cancelCtx,timeCtx,valueCtx四种

### context有几种

