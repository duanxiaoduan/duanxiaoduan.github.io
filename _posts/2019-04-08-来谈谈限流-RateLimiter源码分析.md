---
layout:     post
title:      来谈谈限流-RateLimiter源码分析
subtitle:   java基础
date:       2019-04-08
author:     duanxiaoduan
header-img: img/post-bg-design-linux.jpg
catalog: true
tags:
    - java基础
---

>本文将分析guava限流类`RateLimiter`的实现。

`RateLimiter`有两个实现类：`SmoothBursty`和`SmoothWarmingUp`，其都是令牌桶算法的变种实现，区别在于`SmoothBursty`加令牌的速度是恒定的，而`SmoothWarmingUp`会有个预热期，在预热期内加令牌的速度是慢慢增加的，直到达到固定速度为止。其适用场景是，对于有的系统而言刚启动时能承受的QPS较小，需要预热一段时间后才能达到最佳状态。

### 基本使用

`RateLimiter`的使用很简单：

    //create方法传入的是每秒生成令牌的个数
    RateLimiter rateLimiter= RateLimiter.create(1);
    for (int i = 0; i < 5; i++) {
        //acquire方法传入的是需要的令牌个数，当令牌不足时会进行等待，该方法返回的是等待的时间
        double waitTime=rateLimiter.acquire(1);
        System.out.println(System.currentTimeMillis()/1000+" , "+waitTime);
    }

输出如下：

    1548070953 , 0.0
    1548070954 , 0.998356
    1548070955 , 0.998136
    1548070956 , 0.99982

需要注意的是，当令牌不足时，`acquire`方法并不会阻塞本次调用，而是**会算在下次调用的头上**。比如第一次调用时，令牌桶中并没有令牌，但是第一次调用也没有阻塞，而是在第二次调用的时候阻塞了1秒。也就是说，**每次调用欠的令牌（如果桶中令牌不足）都是让下一次调用买单**。

    RateLimiter rateLimiter= RateLimiter.create(1);
    double waitTime=rateLimiter.acquire(1000);
    System.out.println(System.currentTimeMillis()/1000+" , "+waitTime);
    waitTime=rateLimiter.acquire(1);
    System.out.println(System.currentTimeMillis()/1000+" , "+waitTime);

输出如下：

    1548072250 , 0.0
    1548073250 , 999.998773
    

这样设计的目的是：

     Last, but not least: consider a RateLimiter with rate of 1 permit per second, currently completely unused, and an expensive acquire(100) request comes. It would be nonsensical to just wait for 100 seconds, and /then/ start the actual task. Why wait without doing anything? A much better approach is to /allow/ the request right away (as if it was an acquire(1) request instead), and postpone /subsequent/ requests as needed. In this version, we allow starting the task immediately, and postpone by 100 seconds future requests, thus we allow for work to get done in the meantime instead of waiting idly.
    

简单的说就是，如果每次请求都为本次买单会有不必要的等待。比如说令牌增加的速度为每秒1个，初始时桶中没有令牌，这时来了个请求需要100个令牌，那需要等待100s后才能开始这个任务。所以更好的办法是先放行这个请求，然后延迟之后的请求。

另外，RateLimiter还有个`tryAcquire`方法，如果令牌够会立即返回true，否则立即返回false。

### 源码分析

本文主要分析`SmoothBursty`的实现。

首先看`SmoothBursty`中的几个关键字段:

// 桶中最多存放多少秒的令牌数
final double maxBurstSeconds;
//桶中的令牌个数
double storedPermits;
//桶中最多能存放多少个令牌，=maxBurstSeconds*每秒生成令牌个数
double maxPermits;
//加入令牌的平均间隔，单位为微秒，如果加入令牌速度为每秒5个，则该值为1000*1000/5
double stableIntervalMicros;
//下一个请求需要等待的时间
private long nextFreeTicketMicros = 0L; 

#### RateLimiter的创建

先看创建RateLimiter的create方法。

    // permitsPerSecond为每秒生成的令牌数
    public static RateLimiter create(double permitsPerSecond) {
        return create(permitsPerSecond, SleepingStopwatch.createFromSystemTimer());
    }
    
    //SleepingStopwatch主要用于计时和休眠
    static RateLimiter create(double permitsPerSecond, SleepingStopwatch stopwatch) {
        //创建一个SmoothBursty
        RateLimiter rateLimiter = new SmoothBursty(stopwatch, 1.0 /* maxBurstSeconds */);
        rateLimiter.setRate(permitsPerSecond);
        return rateLimiter;
    }

create方法主要就是创建了一个`SmoothBursty`实例，并调用了其`setRate`方法。注意这里的`maxBurstSeconds`写死为1.0。

    @Override
    final void doSetRate(double permitsPerSecond, long nowMicros) {
        resync(nowMicros);
        double stableIntervalMicros = SECONDS.toMicros(1L) / permitsPerSecond;
        this.stableIntervalMicros = stableIntervalMicros;
        doSetRate(permitsPerSecond, stableIntervalMicros);
    }
    
    void resync(long nowMicros) {
        // 如果当前时间比nextFreeTicketMicros大，说明上一个请求欠的令牌已经补充好了，本次请求不用等待
        if (nowMicros > nextFreeTicketMicros) {
          // 计算这段时间内需要补充的令牌，coolDownIntervalMicros返回的是stableIntervalMicros
          double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
         // 更新桶中的令牌，不能超过maxPermits
          storedPermits = min(maxPermits, storedPermits + newPermits);
          // 这里先设置为nowMicros
          nextFreeTicketMicros = nowMicros;
        }
    }

    @Override
    void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
        double oldMaxPermits = this.maxPermits;
        maxPermits = maxBurstSeconds * permitsPerSecond;
        if (oldMaxPermits == Double.POSITIVE_INFINITY) {
            // if we don't special-case this, we would get storedPermits == NaN, below
            storedPermits = maxPermits;
        } else {
            //第一次调用oldMaxPermits为0，所以storedPermits（桶中令牌个数）也为0
            storedPermits =
                    (oldMaxPermits == 0.0)
                            ? 0.0 // initial state
                            : storedPermits * maxPermits / oldMaxPermits;
        }
    }

`setRate`方法中设置了`maxPermits=maxBurstSeconds * permitsPerSecond`;而`maxBurstSeconds` 为1，所以`maxBurstSeconds` 只会保存1秒中的令牌数。

需要注意的是`SmoothBursty`是非public的类，也就是说只能通过`RateLimiter.create`方法创建，而该方法中的`maxBurstSeconds` 是写死1.0的，也就是说我们只能创建桶大小为permitsPerSecond*1的`SmoothBursty`对象（当然反射的方式不在讨论范围），在guava的github仓库里有好几条issue（[issue1](https://github.com/google/guava/issues/3112),[issue2](https://github.com/google/guava/issues/1707),[issue3](https://github.com/google/guava/issues/1974),[issue4](https://github.com/google/guava/issues/1581)）希望能由外部设置`maxBurstSeconds` ，但是并没有看到官方人员的回复。而在唯品会的开源项目vjtools中，有人提出了这个[问题](https://github.com/vipshop/vjtools/issues/110)，唯品会的同学对guava的RateLimiter进行了[拓展](https://github.com/vipshop/vjtools/commit/9eacb861960df0c41b2323ce14da037a9fdc0629)。

对于guava的这样设计我很不理解，有清楚的朋友可以说下~

到此为止一个`SmoothBursty`对象就创建好了，接下来我们分析其`acquire`方法。

#### acquire方法

    public double acquire(int permits) {
        // 计算本次请求需要休眠多久（受上次请求影响）
        long microsToWait = reserve(permits);
        // 开始休眠
        stopwatch.sleepMicrosUninterruptibly(microsToWait);
        return 1.0 * microsToWait / SECONDS.toMicros(1L);
    }
     
    final long reserve(int permits) {
        checkPermits(permits);
        synchronized (mutex()) {
          return reserveAndGetWaitLength(permits, stopwatch.readMicros());
        }
    }
    
    final long reserveAndGetWaitLength(int permits, long nowMicros) {
        long momentAvailable = reserveEarliestAvailable(permits, nowMicros);
        return max(momentAvailable - nowMicros, 0);
    }

    final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
        // 这里调用了上面提到的resync方法，可能会更新桶中的令牌值和nextFreeTicketMicros
        resync(nowMicros);
        // 如果上次请求花费的令牌还没有补齐，这里returnValue为上一次请求后需要等待的时间，否则为nowMicros
        long returnValue = nextFreeTicketMicros;
        double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
        // 缺少的令牌数
        double freshPermits = requiredPermits - storedPermitsToSpend;
        // waitMicros为下一次请求需要等待的时间；SmoothBursty的storedPermitsToWaitTime返回0
        long waitMicros =
            storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
                + (long) (freshPermits * stableIntervalMicros);
        // 更新nextFreeTicketMicros
        this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
        // 减少令牌
        this.storedPermits -= storedPermitsToSpend;
        return returnValue;
    }

`acquire`中会调用`reserve`方法获得当前请求需要等待的时间，然后进行休眠。`reserve`方法最终会调用到`reserveEarliestAvailable`，在该方法中会先调用上文提到的`resync`方法对桶中的令牌进行补充（如果需要的话），然后减少桶中的令牌，以及计算这次请求欠的令牌数及需要等待的时间（由下次请求负责等待）。

如果上一次请求没有欠令牌或欠的令牌已经还清则返回值为`nowMicros`，否则返回值为上一次请求缺少的令牌个数*生成一个令牌所需要的时间。

### End

本文讲解了`RateLimiter`子类`SmoothBursty`的源码，对于另一个子类`SmoothWarmingUp`的原理大家可以自行分析。相对于传统意义上的令牌桶，`RateLimiter`的实现还是略有不同，主要体现在一次请求的花费由下一次请求来承担这一点上。