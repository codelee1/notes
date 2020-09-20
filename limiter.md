## goland time/rate限流

##简单使用

### 创建限流器
可以用如下方法来创建一个限流器：

limiter := NewLimiter(10, 1);

第一个参数是r Limit，每秒可向token桶中产生多少个token，Limit其实是float64别名。

第二个参数是b int，b代表桶的容量大小。

那么对于上面的例子，含义为：该限流器的令牌桶大小为1，并每秒往令牌通中放置10个token。

上述例子为每秒产生10个token，即产生token的间隔为100ms，除此之外，还可以用Every自定义产生token的间隔。

limit := rate.Every(100 * time.Millisecond) // 间隔为100ms，也是每秒产生10个token

limiter := NewLimiter(limit, 1); 

### 消费token
1、Allow & AllowN

func (lim *Limiter) Allow() bool

func (lim *Limiter) AllowN(now time.Time, n int) bool

Allow 即为 AllowN(lim, 1)

AllowN含义为：截止到当前时间内，令牌桶内是否有n个令牌，有，则消费之，返回true，无或不足则不消费，返回false，

常用于线上场景突然流量增大，对服务器无力处理的流量请求直接丢弃。


2、Wait & WaitN

func (lim *Limiter) Wait(ctx context.Context) (err error)

func (lim *Limiter) WaitN(ctx context.Context, n int) (err error)

Wait 即为 WaitN(ctx, 1)

WaitN含义为，截止到当前时间内,令牌桶内是否有n个令牌，有，则消费之，返回nil，无或不足则等待到n个token后，再消费返回，

可以设置context的Deadline或者Timeout，来决定此次Wait的最长时间。


3、Reserve & ReserveN（储量）

func (lim *Limiter) Reserve() *Reservation

func (lim *Limiter) ReserveN(now time.Time, n int) *Reservation

Reserve 即为ReserveN(time.Now(), 1)

调用返回的是*Reservation，可以看成是当前时间戳下的限制器，可以看成是WaitN的增强版，可操作性更多

可以调用对象的Delay方法获得还需多长时间才可产生足够的token，如果足够，则返回0。

可以调用对象的CancelAt(time.Time)来指定取消时间，取消后会返还token

引用源码例子：
````
        r := lim.ReserveN(time.Now(), 1)
        if !r.OK() {
             // Not allowed to act! Did you remember to set lim.burst to be > 0 ?
             return
        }
        time.Sleep(r.Delay())
        Act()
````
### 动态调整

Limiter支持修改token生成速率和token桶容量

SetLimit(Limit) 修改生成token的速率

SetBurst(int) 修改桶大小


## 源码分析

### 原理
源码参考思想为维基百科：https://en.wikipedia.org/wiki/Token_bucket

原理图为: 

![token-bucket](https://raw.githubusercontent.com/codelee1/uploads/master/limiter/token-bucket.jpg "token-bucket")


简单的原理为：令牌桶就是想象有一个固定大小的桶，系统会以恒定速率向桶中放 Token，桶满则暂时不放。

而用户则从桶中取 Token，如果有剩余 Token 就可以一直取。如果没有剩余 Token，则需要等到系统中被放置了 Token 才行。

直观的实现方式为：有一个Timer和一个Queue，其中Timer定时往Queue中放置token，用户则从Queue中取token。

但这个方式需要维护Timer和一个Queue，还需要额外的内存。

Golang time/rate则采用 Lazy loading 的方式，直到需要消费之前，才根据时间差来更新token数量，而且也不用Queue来存放token，仅通过计数方式实现。


### 源码实现

#### 一、token的生成与消费

time/rate的简单使用中，介绍了三种消费token的方法，分别是Allow、Wait和Reserve，但是三个方法内部都调用了reserveN来生成和消费token。

在正式分析reserveN函数实现之前，我们先了解一些简单概念。

首先我们通过原理知道，token的生成速率是一定的，比如每秒10个，即每个token的生成间隔时间为100ms/个。

有了间隔时间，我们就可以确定两个数据：

1、在给定的一段时间中，可生成token数目，time/rate对应的实现函数为：tokensFromDuration

2、生成N个token所需要的时间，time/rate对应的函数为：durationFromTokens

有了这两个转换函数，reserveN函数实现整体过程就很清晰了：

1、计算上次取token的时间到现在，期间一共产生了多少token，以及当前token数量：
    
因为只在取token的时候生成token，也就意味着每次取token的时间戳，也是生成token的时间间隔（这里有坑，后面填），

于是我们可以用tokensFromDuration(now.Sub(lastTakeTime))算出这段时间内产生的token数量，

于是当前的token数量 = 之前剩余token数量 + 新产生token数量 - 要消费的token数量。

2、如果当前token数量大于等于0，则说明桶中token数量充足，调用测无序等待，
如果小于零，则可通过durationFromTokens，计算出生成所需token数量的等待时间。

3、过程中更新令牌通相关数据都添加mutex锁保证线程安全。

再抽象的来看，上述reserveN的实现过程其实就是token数量和时间之间的相互转换过程，如果token数量为正，则token数量充足；
如果token数量为负，则返回生成对应token数量所需的等待时间，且随着负值越来越小，等待的时间就越长。

模拟一下token数量变化：

某一时刻，桶内token数量为2，此时线程A请求5个token，那么桶内token数量不足，则线程A需要等3个token时间，且此时更新token数量为-3，

同时，线程B请求3个token，此时token数量为-3，那么线程B需要等待3+3=6个token时间，且又更新token数量为-6。

对于Allow函数来说，只需要判断等待时间是否0，如果大于0则说明需要等待，返回false，否则返回true。

对于Wait来说，直接等待：t := time.NewTimer(delay) 即可。


## 数值溢出与优化

在讲reserveN函数实现时，计算从当前时间到上一次取token时间的时间段，一共产生了多少token为：
````    
delta = tokensFromDuration(now.Sub(lim.last))

tokens := lim.tokens + delta
if(token> lim.burst){
    token = lim.burst
}
````
计算当前时间到上一次取token的时间段，作为生成token的依据，好像是没有什么问题。

但是，如果上一次取token的时间lim.last 是很久之前（或者初始化为0），那么根据时间差生成token就会有可能导致token巨大至数值溢出。

time/rate的实现方式是：
````
last := lim.last
if now.Before(last) {
    last = now
}

maxElapsed := lim.limit.durationFromTokens(float64(lim.burst) - lim.tokens)
elapsed := now.Sub(last)
if elapsed > maxElapsed {
    elapsed = maxElapsed
}

delta := lim.limit.tokensFromDuration(elapsed)
tokens := lim.tokens + delta
if burst := float64(lim.burst); tokens > burst {
    tokens = burst
}
````

我们看到，这里先是用了：lim.limit.durationFromTokens(float64(lim.burst) - lim.tokens)，求得当前填满桶的时间maxElapsed，
然后再取 elapsed 和 maxElapsed 较小的值，这样就既能避免数值溢出，又能规避时间差太长而生成过多token。


## float精度问题

在原理讲述中，token和时间的相互转换函数durationFromTokens和tokensFromDuration涉及到float64的乘除，这时候就需要注意精度问题了

如在tokensFromDuration方法中，官方也犯了一个错误：golang.org/issues/34861。

这是tokensFromDuration的早期实现版本
````
func (limit Limit) tokensFromDuration(d time.Duration) float64 {
    return d.Seconds() * float64(limit)
}
````

乍一看好像没有什么问题，但是 d.Seconds()的值已经是float64小数，两个小数相乘会导致精度问题。

修改后的为
````
func (limit Limit) tokensFromDuration(d time.Duration) float64 {
	// Split the integer and fractional parts ourself to minimize rounding errors.
	// See golang.org/issues/34861.
	sec := float64(d/time.Second) * float64(limit)
	nsec := float64(d%time.Second) * float64(limit)
	return sec + nsec/1e9
}
````
time.Duration 代表纳秒，是int64的别名，修改后分别将秒的整数部分和小数部分求出，然后再相乘，这样就可以得到最精确的值。

## token归还


对于token的归还，有两个地方用到，一个是Reserve函数中返回的*Reservation，它可调用Cancel()，或是CancelAt(time.Time)
来将消费的token尽可能的返还。还有一个是Wait函数的Context可取消等待，其实Wait内部实现也是通过 *Reservation.Cancel()
来完成token归还。

然而，在归还token时，并不是简单的要归还的token数量累加到现有的token数量，而是：
````
// calculate tokens to restore
// The duration between lim.lastEvent and r.timeToAct tells us how many tokens were reserved
// after r was obtained. These tokens should not be restored.
restoreTokens := float64(r.tokens) - r.limit.tokensFromDuration(r.lim.lastEvent.Sub(r.timeToAct))
if restoreTokens <= 0 {
    return
}
````

其中：
r.tokens 代表本次消耗的token数。

r.timeToAct 代表此次满足此次token消费的时间戳，也就是此次的等待时长。

r.lim.lastEvent 代表最近一次消耗token的timeToAct的值


r.lim.lastEvent.Sub(r.timeToAct)代表后续的申请时间，如果等于0，则没有申请过；如果大于0，则说明除了当次申请的，还有别的地方也申请消耗
token，那么在执行此次消费的等待的过程中，如果在某一时刻取消消费，那么此次执行的结束时间到下次执行消耗的时间段所产生token则不归还(这里还需深究)


### 总结
token bucket其实很适合互联网突发性请求，其限制请求并不是严格的速率限制，而是有一个桶作为缓冲。
只要桶中有数据，请求就可以一直执行，但当突发性流量的时候，也可以按照预定速率来进行消费。


此外在维基百科中，也提到了分层 Token Bucket(HTB) 作为传统 Token Bucket 的进一步优化，Linux 内核中也用它进行流量控制。
