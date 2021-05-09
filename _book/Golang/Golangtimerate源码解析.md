## Golang time rate 源码解析

### Limiter对象解析

```go
// Limit 定义了某些行为的最大频率，也就是每秒发生的次数。0代表不允许任何行为
type Limit float64

// Inf 代表不限制数量
const Inf = Limit(math.MaxFloat64)

// 输入时间，返回一个Limit，这个Limit代表这段时间内只允许发生一次行为
func Every(interval time.Duration) Limit {
	if interval <= 0 {
		return Inf
	}
	return 1 / Limit(interval.Seconds())
}

// Limiter 控制了行为发生的频率
// 它实现了一个容量为b的令牌桶, 初始为满的，并且以r个令牌/s的速度重新装满
// 在一个足够长的时间里， limiter会控制令牌入桶的速度
// 特殊地，
// 1.如果令牌桶设置为无限容量，那么b将被忽视。
// 2.如果令牌桶的容量被设置为0，那么它会拒绝所有events
// 3.我们可以使用NewLimiter来创建一个非空的Limiter
//
// Limiter有三个主要的方法:Allow, Reserve, 和 Wait.
// 大多数调用者应该使用Wait.
//
// 三个方法都会消费一个token（令牌）
// 他们只在没有令牌可以消费的时候有一些差别
// Allow：如果没有可用的令牌，调用allow会返回false
// Reserve：返回未来令牌的一个预定（一个对象）和需要等待的时间
// Wait：一直等待直到获得令牌，或者相关的上下文被取消。
//
// 方法 AllowN, ReserveN, and WaitN 一次消费n个令牌
type Limiter struct {
	limit Limit
	burst int

	mu     sync.Mutex
	tokens float64
	// last是指上次令牌桶令牌更新的时间
	// lastEvent是指未来或者以前的最新的频率限制行为
	lastEvent time.Time
}

// Limit 返回总体的最大限制频率
func (lim *Limiter) Limit() Limit {
	lim.mu.Lock()
	defer lim.mu.Unlock()
	return lim.limit
}

// Burst 返回能被 Allow, Reserve, or Wait 消费的令牌最大数量
// Burst values allow more events to happen at once.
// Burst为0代表不允许任何events 除非 limit==Inf
func (lim *Limiter) Burst() int {
	return lim.burst
}

// NewLimiter 初始化一个恢复速率为r和最大容量为b的Limiter对象
func NewLimiter(r Limit, b int) *Limiter {
	return &Limiter{
		limit: r,
		burst: b,
	}
}

// Allow是AllowN的特殊情况(time.Now(), 1).
func (lim *Limiter) Allow() bool {
	return lim.AllowN(time.Now(), 1)
}

// AllowN 代表当前消费n个tokens的events是否可以发生
// 如果你想跳过使用超过这个限制的方法，请使用它。否则请使用Reserve or Wait
func (lim *Limiter) AllowN(now time.Time, n int) bool {
	return lim.reserveN(now, n, 0).ok
}
```

### Reservation解析

```go
// A Reservation 保留了被Limiter运行在一段时间后发生的行为的信息
// A Reservation 可以被取消，这样可以允许Limiter去通过其他的events
type Reservation struct {
	ok        bool
	lim       *Limiter
	tokens    int
	timeToAct time.Time
	// This is the Limit at reservation time, it can change later.
	limit Limit
}

// OK 返回limiter是否能提供需要数量的tokens在给定的最大时间内。
// 如果 OK 是 false, Delay 返回InfDuration,调用Cancel方法不会做任何事
func (r *Reservation) OK() bool {
	return r.ok
}

// Delay是DelayFrom的特殊情况(time.Now()).
func (r *Reservation) Delay() time.Duration {
	return r.DelayFrom(time.Now())
}

// InfDuration 是如果预定不可用的时候返回的等待时间
const InfDuration = time.Duration(1<<63 - 1)

// DelayFrom 返回reserveation在采取预期行动时需要等待的时间。0 duration 代表必须立刻发生
// 返回InfDuration 代表在给定时间内Reservation无法执行
func (r *Reservation) DelayFrom(now time.Time) time.Duration {
	if !r.ok {
		return InfDuration
	}
	delay := r.timeToAct.Sub(now)
	if delay < 0 {
		return 0
	}
	return delay
}

// Cancel是CancelAt的特殊情况(time.Now()).
func (r *Reservation) Cancel() {
	r.CancelAt(time.Now())
	return
}

func (r *Reservation) CancelAt(now time.Time) {
	if !r.ok {
		return
	}

	r.lim.mu.Lock()
	//defer在方法返回的同时释放锁
	defer r.lim.mu.Unlock()

	if r.lim.limit == Inf || r.tokens == 0 || r.timeToAct.Before(now) {
		return
	}

  //恢复的tokens是上次events和这次行动之间生成的token，他们不需要被恢复
	restoreTokens := float64(r.tokens) - r.limit.tokensFromDuration(r.lim.lastEvent.Sub(r.timeToAct))
	if restoreTokens <= 0 {
		return
	}
	// advance time to now
	now, _, tokens := r.lim.advance(now)
	// calculate new number of tokens
	tokens += restoreTokens
	if burst := float64(r.lim.burst); tokens > burst {
		tokens = burst
	}
	// update state
	r.lim.last = now
	r.lim.tokens = tokens
	if r.timeToAct == r.lim.lastEvent {
		prevEvent := r.timeToAct.Add(r.limit.durationFromTokens(float64(-r.tokens)))
		if !prevEvent.Before(now) {
			r.lim.lastEvent = prevEvent
		}
	}

	return
}

// Reserve is shorthand for ReserveN(time.Now(), 1).
func (lim *Limiter) Reserve() *Reservation {
	return lim.ReserveN(time.Now(), 1)
}

// ReserveN 返回一个Reservation对象表明在n个events发生之前，调用者需要等待多少时间
// Limiter会把这个Reservation放到账户里当运行这次events的时候
// ReserveN 返回 false 如果 n 超过了 Limiter'的 最大size.
// 用例:
//   r := lim.ReserveN(time.Now(), 1)
//   if !r.OK() {
//     // 如果执行到这说明可能是limit设置为0了
//     return
//   }
//   time.Sleep(r.Delay())
//   Act()
// 你可以使用这个方法 如果你希望在不删除events的情况下延迟它开始运行的时间
// 而如果你希望维护一个deadline或者取消延迟,请使用Wait.
// 如果希望在生成速率不允许的情况下删除events请使用Allow
func (lim *Limiter) ReserveN(now time.Time, n int) *Reservation {
	r := lim.reserveN(now, n, InfDuration)
	return &r
}
```

### Wait解析

```go
func (lim *Limiter) Wait(ctx context.Context) (err error) {
	return lim.WaitN(ctx, 1)
}

// WaitN 阻塞知道limit运行n个events发生
// 他将返回一个error如果上下文取消或者n超过了桶的最大数量或者所需要的运行时间超过上下文所允许的最大时间
func (lim *Limiter) WaitN(ctx context.Context, n int) (err error) {
	lim.mu.Lock()
	burst := lim.burst
	limit := lim.limit
	lim.mu.Unlock()

	if n > burst && limit != Inf {
		return fmt.Errorf("rate: Wait(n=%d) exceeds limiter's burst %d", n, lim.burst)
	}
	// 查看上下文是否完成
	select {
	case <-ctx.Done():
		return ctx.Err()
	default:
	}
	// Determine wait limit
	now := time.Now()
	waitLimit := InfDuration
	if deadline, ok := ctx.Deadline(); ok {
		waitLimit = deadline.Sub(now)
	}
	// Reserve
	r := lim.reserveN(now, n, waitLimit)
	if !r.ok {
		return fmt.Errorf("rate: Wait(n=%d) would exceed context deadline", n)
	}
	// Wait if necessary
	delay := r.DelayFrom(now)
	if delay == 0 {
		return nil
	}
	t := time.NewTimer(delay)
	defer t.Stop()
	select {
	case <-t.C:
		// We can proceed.
		return nil
	case <-ctx.Done():
    //这个代表在我们执行之前上下文已完成，让出token
		r.Cancel()
		return ctx.Err()
	}
}

func (lim *Limiter) SetLimit(newLimit Limit) {
	lim.SetLimitAt(time.Now(), newLimit)
}

// SetLimitAt 为limiter设置一个新的limit. 新的Limit可能违反旧reserve的event
func (lim *Limiter) SetLimitAt(now time.Time, newLimit Limit) {
	lim.mu.Lock()
	defer lim.mu.Unlock()

	now, _, tokens := lim.advance(now)

	lim.last = now
	lim.tokens = tokens
	lim.limit = newLimit
}

func (lim *Limiter) SetBurst(newBurst int) {
	lim.SetBurstAt(time.Now(), newBurst)
}

// 设置新的桶容量
func (lim *Limiter) SetBurstAt(now time.Time, newBurst int) {
	lim.mu.Lock()
	defer lim.mu.Unlock()

	now, _, tokens := lim.advance(now)

	lim.last = now
	lim.tokens = tokens
	lim.burst = newBurst
}

// reserveN is a helper method for AllowN, ReserveN, and WaitN.
// maxFutureReserve 定义为最大等待数
// reserveN returns Reservation, not *Reservation, to avoid allocation in AllowN and WaitN.
func (lim *Limiter) reserveN(now time.Time, n int, maxFutureReserve time.Duration) Reservation {
	lim.mu.Lock()

	if lim.limit == Inf {
		lim.mu.Unlock()
		return Reservation{
			ok:        true,
			lim:       lim,
			tokens:    n,
			timeToAct: now,
		}
	}

	now, last, tokens := lim.advance(now)

	// 计算还剩下多少tokens
	tokens -= float64(n)

	// Calculate the wait duration
	var waitDuration time.Duration
	if tokens < 0 {
		waitDuration = lim.limit.durationFromTokens(-tokens)
	}

	// 只有当拿取的token数量小于桶的容量并且等待数小于最大等待数
	ok := n <= lim.burst && waitDuration <= maxFutureReserve

	// Prepare reservation
	r := Reservation{
		ok:    ok,
		lim:   lim,
		limit: lim.limit,
	}
	if ok {
		r.tokens = n
		r.timeToAct = now.Add(waitDuration)
	}

	// Update state
	if ok {
		lim.last = now
		lim.tokens = tokens
		lim.lastEvent = r.timeToAct
	} else {
		lim.last = last
	}

	lim.mu.Unlock()
	return r
}

// advance 计算返回更新状态
func (lim *Limiter) advance(now time.Time) (newNow time.Time, newLast time.Time, newTokens float64) {
	last := lim.last
	if now.Before(last) {
		last = now
	}

	// Avoid making delta overflow below when last is very old.
	maxElapsed := lim.limit.durationFromTokens(float64(lim.burst) - lim.tokens)
	elapsed := now.Sub(last)
	if elapsed > maxElapsed {
		elapsed = maxElapsed
	}

	// Calculate the new number of tokens, due to time that passed.
	delta := lim.limit.tokensFromDuration(elapsed)
	tokens := lim.tokens + delta
	if burst := float64(lim.burst); tokens > burst {
		tokens = burst
	}

	return now, last, tokens
}

// durationFromTokens is a unit conversion function from the number of tokens to the duration
// of time it takes to accumulate them at a rate of limit tokens per second.
func (limit Limit) durationFromTokens(tokens float64) time.Duration {
	seconds := tokens / float64(limit)
	return time.Nanosecond * time.Duration(1e9*seconds)
}

// tokensFromDuration is a unit conversion function from a time duration to the number of tokens
// which could be accumulated during that duration at a rate of limit tokens per second.
func (limit Limit) tokensFromDuration(d time.Duration) float64 {
	// Split the integer and fractional parts ourself to minimize rounding errors.
	// See golang.org/issues/34861.
	sec := float64(d/time.Second) * float64(limit)
	nsec := float64(d%time.Second) * float64(limit)
	return sec + nsec/1e9
}
```

Tips: 可以参考 [这里](https://en.wikipedia.org/wiki/Token_bucket) 获得令牌桶的更多信息。

