### 1. 常见限流算法

### 漏斗算法

漏桶算法思路很简单，水（请求）先进入到漏桶里，漏桶以一定的速度出水，当水流入速度过大会直接溢出，可以看出漏桶算法能强行限制数据的传输速率。

### 令牌筒算法

令牌桶算法的原理是系统会以一个恒定的速度往桶里放入令牌，而如果请求需要被处理，则需要先从桶里获取一个令牌，当桶里没有令牌可取时，则拒绝服务。对于很多应用场景来说，除了要求能够限制数据的平均传输速率外，还要求允许某种程度的突发传输。这时候漏桶算法可能就不合适了，令牌桶算法更为适合。

---

## 2. RateLimiter使用与原理

Google开源工具包Guava提供了限流工具类RateLimiter，该类基于令牌桶算法实现流量限制

### pom

```
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>28.2-jre</version>
        </dependency>	
```

---

### easyDemo

```java
package src.rate_limit;

import com.google.common.util.concurrent.RateLimiter;

import java.util.concurrent.TimeUnit;

/**
 * @author lianjunjie
 * @date 2021-08-31 14:27:40
 */
public class RateLimitDemo {
    public static void main(String[] args) {
        rateLimitEasyDemo();
    }

    public static void rateLimitEasyDemo() {
        // generate 1 token per second
        RateLimiter limiter = RateLimiter.create(1);
        for (int i = 1; i < 10; i += 2) {
            // acquire i token, accquireTime is blocking time
//            double acquireTime = limiter.acquire(i);
//            System.out.println("acquire " + i + " token need acquire time: " + acquireTime);

            // try to acquire i token in 2 seconds, fail will return false
            boolean acquireFlag = limiter.tryAcquire(i, 2, TimeUnit.SECONDS);
            System.out.println("acquire " + i + " token " + (acquireFlag ? "success" : "false"));
        }
    }
}
```

---

### 相关设计

参考地址：https://www.jianshu.com/p/5d4fe4b2a726

---

#### guava两种限流模式

Guava有两种限流模式，一种为稳定模式(SmoothBursty:令牌生成速度恒定)，一种为渐进模式(SmoothWarmingUp:令牌生成速度缓慢提升直到维持在一个稳定值)。默认为SmoothBursty

---

#### setRate()设计

- setRate

```java
public final void setRate(double permitsPerSecond) {
  checkArgument(
      permitsPerSecond > 0.0 && !Double.isNaN(permitsPerSecond), "rate must be positive");
  synchronized (mutex()) {
    doSetRate(permitsPerSecond, stopwatch.readMicros());
  }
}
```

- doSetRate

```java
@Override
  final void doSetRate(double permitsPerSecond, long nowMicros) {
    resync(nowMicros);
    double stableIntervalMicros = SECONDS.toMicros(1L) / permitsPerSecond;
    this.stableIntervalMicros = stableIntervalMicros;
    doSetRate(permitsPerSecond, stableIntervalMicros);
  }
```

- resync

```java
void resync(long nowMicros) {
    // if nextFreeTicket is in the past, resync to now
    if (nowMicros > nextFreeTicketMicros) {
      double newPermits = (nowMicros - nextFreeTicketMicros) / coolDownIntervalMicros();
      storedPermits = min(maxPermits, storedPermits + newPermits);
      nextFreeTicketMicros = nowMicros;
    }
}
```

- doSetRate in SomoothRateLimiter

```java
    @Override
    void doSetRate(double permitsPerSecond, double stableIntervalMicros) {
      double oldMaxPermits = this.maxPermits;
      maxPermits = maxBurstSeconds * permitsPerSecond;
      if (oldMaxPermits == Double.POSITIVE_INFINITY) {
        // if we don't special-case this, we would get storedPermits == NaN, below
        storedPermits = maxPermits;
      } else {
        storedPermits =
            (oldMaxPermits == 0.0)
                ? 0.0 // initial state
                : storedPermits * maxPermits / oldMaxPermits;
      }
    }
```

**以上是持续生成令牌的逻辑**

采用了延迟计算的方法，如上`resync`函数。该函数会在每次获取令牌之前调用，其实现思路为，若当前时间晚于nextFreeTicketMicros，则计算该段时间内可以生成多少令牌，将生成的令牌加入令牌桶中并更新数据。这样一来，只需要在获取令牌时计算一次即可。

---

#### acquire()

- acquire(int permits)

```java
  @CanIgnoreReturnValue
  public double acquire(int permits) {
    long microsToWait = reserve(permits);
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

/* notice here resync,其实就是在acquire的时候，进行了令牌筒的补充 */
  @Override
  final long reserveEarliestAvailable(int requiredPermits, long nowMicros) {
    resync(nowMicros);
    long returnValue = nextFreeTicketMicros;
    double storedPermitsToSpend = min(requiredPermits, this.storedPermits);
    double freshPermits = requiredPermits - storedPermitsToSpend;
    long waitMicros =
        storedPermitsToWaitTime(this.storedPermits, storedPermitsToSpend)
            + (long) (freshPermits * stableIntervalMicros);

    this.nextFreeTicketMicros = LongMath.saturatedAdd(nextFreeTicketMicros, waitMicros);
    this.storedPermits -= storedPermitsToSpend;
    return returnValue;
  }

/*首先通过resync生成令牌以及同步nextFreeTicketMicros时间戳，freshPermits从令牌桶中获取令牌后还需要的令牌数量，通过storedPermitsToWaitTime计算出获取freshPermits还需要等待的时间，在稳定模式中，这里就是(long) (freshPermits * stableIntervalMicros) ，然后更新nextFreeTicketMicros以及storedPermits，这次获取令牌需要的等待到的时间点， reserveAndGetWaitLength返回需要等待的时间间隔。

从`reserveEarliestAvailable`可以看出RateLimiter的预消费原理，以及获取令牌的等待时间时间原理（可以解释示例结果），再获取令牌不足时，并没有等待到令牌全部生成，而是更新了下次获取令牌时的nextFreeTicketMicros，从而影响的是下次获取令牌的等待时间。

 `reserve`这里返回等待时间后，`acquire`通过调用`stopwatch.sleepMicrosUninterruptibly(microsToWait);`进行sleep操作，这里不同于Thread.sleep(), 这个函数的sleep是uninterruptibly的，内部实现：
 */

public static void sleepUninterruptibly(long sleepFor, TimeUnit unit) {
    //sleep 阻塞线程 内部通过Thread.sleep()
  boolean interrupted = false;
  try {
    long remainingNanos = unit.toNanos(sleepFor);
    long end = System.nanoTime() + remainingNanos;
    while (true) {
      try {
        // TimeUnit.sleep() treats negative timeouts just like zero.
        NANOSECONDS.sleep(remainingNanos);
        return;
      } catch (InterruptedException e) {
        interrupted = true;
        remainingNanos = end - System.nanoTime();
        //如果被interrupt可以继续，更新sleep时间，循环继续sleep
      }
    }
  } finally {
    if (interrupted) {
      Thread.currentThread().interrupt();
      //如果被打断过，sleep过后再真正中断线程
    }
  }
}
```

---

#### tryAcquire()

```java
public boolean tryAcquire(int permits) {
  return tryAcquire(permits, 0, MICROSECONDS);
}

public boolean tryAcquire() {
  return tryAcquire(1, 0, MICROSECONDS);
}

public boolean tryAcquire(int permits, long timeout, TimeUnit unit) {
  long timeoutMicros = max(unit.toMicros(timeout), 0);
  checkPermits(permits);
  long microsToWait;
  synchronized (mutex()) {
    long nowMicros = stopwatch.readMicros();
    if (!canAcquire(nowMicros, timeoutMicros)) {
      return false;
    } else {
      microsToWait = reserveAndGetWaitLength(permits, nowMicros);
    }
  }
  stopwatch.sleepMicrosUninterruptibly(microsToWait);
  return true;
}

private boolean canAcquire(long nowMicros, long timeoutMicros) {
  return queryEarliestAvailable(nowMicros) - timeoutMicros <= nowMicros;
}

@Override
final long queryEarliestAvailable(long nowMicros) {
  return nextFreeTicketMicros;
}
```

---

