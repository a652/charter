---
title: "go-redis 连接池的实现"
date: 2020-06-15T11:36:16+08:00
draft: false
tags: ["go"]
categories: [""]
---

> 基于版本v7.4.0
> 连接池的实现位于/internal/pool/pool.go


文件一开始先是一些变量和类型的定义。首先是用对象池存储timer定时器，用于控制连接池超时。

```go
var timers = sync.Pool{
	New: func() interface{} {
		t := time.NewTimer(time.Hour)
		t.Stop()
		return t
	},
}
```

然后是一个Stats结构体，用于存储连接池自身的一些状态。

```go
// Stats contains pool state information and accumulated stats.
type Stats struct {
	Hits     uint32 // number of times free connection was found in the pool
	Misses   uint32 // number of times free connection was NOT found in the pool
	Timeouts uint32 // number of times a wait timeout occurred

	TotalConns uint32 // number of total connections in the pool
	IdleConns  uint32 // number of idle connections in the pool
	StaleConns uint32 // number of stale connections removed from the pool
}
```

接下来是连接池的接口定义。

```go
type Pooler interface {
	NewConn(context.Context) (*Conn, error)
	CloseConn(*Conn) error

	Get(context.Context) (*Conn, error)
	Put(*Conn)
	Remove(*Conn, error)

	Len() int
	IdleLen() int
	Stats() *Stats

	Close() error
}
```

然后是一些可配置选项和一个struct，是对上述接口的具体实现。

```go
type Options struct {
	Dialer  func(context.Context) (net.Conn, error)
	OnClose func(*Conn) error

	PoolSize           int   // 连接池大小
	MinIdleConns       int   // 最小空闲连接
	MaxConnAge         time.Duration  // 连接最大存活时间
	PoolTimeout        time.Duration  // 连接池超时
	IdleTimeout        time.Duration  // 空闲超时
	IdleCheckFrequency time.Duration  // 检查空闲超时的时间间隔
}

type ConnPool struct {
	opt *Options

	dialErrorsNum uint32 // atomic

	lastDialErrorMu sync.RWMutex
	lastDialError   error

	queue chan struct{} //我将其理解为一个令牌桶，在初始化的时候桶的大小与连接池大小一致，只有拿到令牌后才能获得连接 

	connsMu      sync.Mutex
	conns        []*Conn  // 连接池中的所有连接
	idleConns    []*Conn  // 连接池中的空闲连接
	poolSize     int
	idleConnsLen int

	stats Stats

	_closed  uint32 // atomic
	closedCh chan struct{}
}
```

其中一行比较特殊的写法是在编译阶段就检查ConnPool是否实现了Pooler接口。

```go
var _ Pooler = (*ConnPool)(nil)
```

如果设置了Options中的MinIdleConns，IdleTimeout和IdleCheckFrequency，在新建连接池的时候会做两件事情：

1. 新建空闲连接，直到达到连接池容量或者达到最小空闲连接数。
2. 启动一个goroutine，定时清理连接池中过期的空闲连接。

```go
func NewConnPool(opt *Options) *ConnPool {
	p := &ConnPool{
		opt: opt,

		queue:     make(chan struct{}, opt.PoolSize),
		conns:     make([]*Conn, 0, opt.PoolSize),
		idleConns: make([]*Conn, 0, opt.PoolSize),
		closedCh:  make(chan struct{}),
	}

	p.connsMu.Lock()
	p.checkMinIdleConns()
	p.connsMu.Unlock()

	if opt.IdleTimeout > 0 && opt.IdleCheckFrequency > 0 {
		go p.reaper(opt.IdleCheckFrequency)
	}

	return p
}
```

### 从连接池获取连接的逻辑

先检查连接池是否已关闭，如果已关闭则直接返回err。

waitTurn()等待获取令牌，获取之后是循环从空闲连接slice里取连接，每次取slice里最后一个元素，如果取到已经过期的连接就丢弃，直到取到正常的连接。如果空闲连接为nil会再新建一个。

此时拿到连接后令牌不会释放，直到连接用完再Put回连接池的时候才会释放。

```go
// Get returns existed connection from the pool or creates a new one.
func (p *ConnPool) Get(ctx context.Context) (*Conn, error) {
	if p.closed() {
		return nil, ErrClosed
	}

	err := p.waitTurn(ctx)
	if err != nil {
		return nil, err
	}

	for {
		p.connsMu.Lock()
		cn := p.popIdle()
		p.connsMu.Unlock()

		if cn == nil {
			break
		}

		if p.isStaleConn(cn) {
			_ = p.CloseConn(cn)
			continue
		}

		atomic.AddUint32(&p.stats.Hits, 1)
		return cn, nil
	}

	atomic.AddUint32(&p.stats.Misses, 1)

	newcn, err := p.newConn(ctx, true)
	if err != nil {
		p.freeTurn()
		return nil, err
	}

	return newcn, nil
}
```

### waitTurn获取令牌的逻辑

如果有空闲连接则直接return，继续获取空闲连接；否则按照设置的PoolTimeout进入等待超时机制。如果在ctx结束之前或者timer定时器超时之前取得令牌则正常返回，否则返回err获取令牌失败。

```go
func (p *ConnPool) waitTurn(ctx context.Context) error {
	select {
	case <-ctx.Done():
		return ctx.Err()
	default:
	}

	select {
	case p.queue <- struct{}{}:
		return nil
	default:
	}

	timer := timers.Get().(*time.Timer)
	timer.Reset(p.opt.PoolTimeout)

	select {
	case <-ctx.Done():
		if !timer.Stop() {  // timer.Stop()的推荐用法，目的是保证channel被清空，详见官方注释
			<-timer.C
		}
		timers.Put(timer)
		return ctx.Err()
	case p.queue <- struct{}{}:
		if !timer.Stop() {
			<-timer.C
		}
		timers.Put(timer)
		return nil
	case <-timer.C:
		timers.Put(timer)
		atomic.AddUint32(&p.stats.Timeouts, 1)
		return ErrPoolTimeout
	}
}
```

### 定时清理过期的空闲连接

如果设置了IdleCheckFrequency参数，则在新建连接池时会启动一个goroutine定时检查idleConns中的空闲连接，并将已经过期的清理掉。

如果没有设置IdleCheckFrequency参数，则不会启动这个定时清理的过程，只会在Get()到一个连接时进行判断，这样会导致在系统不忙的时候，一些已超时的空闲连接不能被及时关闭。

```go
func (p *ConnPool) reaper(frequency time.Duration) {
	ticker := time.NewTicker(frequency)
	defer ticker.Stop()

	for {
		select {
		case <-ticker.C:
			// It is possible that ticker and closedCh arrive together,
			// and select pseudo-randomly pick ticker case, we double
			// check here to prevent being executed after closed.
			if p.closed() {
				return
			}
			_, err := p.ReapStaleConns()
			if err != nil {
				internal.Logger.Printf("ReapStaleConns failed: %s", err)
				continue
			}
		case <-p.closedCh:
			return
		}
	}
}
```

这个时候是每次从idleConns slice的头部取空闲连接，当遇到未过期的空闲连接就停止循环。

```go
func (p *ConnPool) ReapStaleConns() (int, error) {
	var n int
	for {
		p.getTurn()

		p.connsMu.Lock()
		cn := p.reapStaleConn()
		p.connsMu.Unlock()
		p.freeTurn()

		if cn != nil {
			_ = p.closeConn(cn)
			n++
		} else {   // 遇到cn是nil，即连接未过期，就停止循环
			break
		}
	}
	atomic.AddUint32(&p.stats.StaleConns, uint32(n))
	return n, nil
}

func (p *ConnPool) reapStaleConn() *Conn {
	if len(p.idleConns) == 0 {
		return nil
	}

	cn := p.idleConns[0]   // 每次从头部取
	if !p.isStaleConn(cn) {
		return nil
	}

	p.idleConns = append(p.idleConns[:0], p.idleConns[1:]...)
	p.idleConnsLen--
	p.removeConn(cn)

	return cn
}
```

### 判断一个连接是否过期的逻辑

取决于IdleTimeout和MaxConnAge参数的设置，如果这两个参数都没有设置，则连接被认为不会过期；

否则当连接的空闲时间超过IdleTimeout时，被认为已过期；

当连接的创建时长超过MaxConnAge时，被认为已过期。

```go
func (p *ConnPool) isStaleConn(cn *Conn) bool {
	if p.opt.IdleTimeout == 0 && p.opt.MaxConnAge == 0 {
		return false
	}

	now := time.Now()
	if p.opt.IdleTimeout > 0 && now.Sub(cn.UsedAt()) >= p.opt.IdleTimeout {
		return true
	}
	if p.opt.MaxConnAge > 0 && now.Sub(cn.createdAt) >= p.opt.MaxConnAge {
		return true
	}

	return false
}
```

实际使用中可以通过调整PoolSize, MinIdleConns, MaxConnAge, PoolTimeout, IdleTimeout, IdleCheckFrequency 这些参数来影响conn pool的行为。
