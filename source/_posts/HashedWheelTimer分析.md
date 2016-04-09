---
title: HashedWheelTimer分析
date: 2016-03-21 22:30:07
tags: [Java,Netty]
---

## HashedWheelTimer 基本原理

1. 建立一个时间wheel，平均分成诺干个slot，比如512个。
1. 每个slot代表一个时间间隔，所以slot一般叫做tick，时间间隔就是tickduration，比如100ms。
1. 像时钟上的时钟一样，wheel上有个指针随着时间旋转指向当前的时间点，每隔一个tickduration一个tick。
1. 当有新的任务添加时，根据过期时间点放置这个任务到合适的slot上. 下面的代码展示了如何计算slot位置。
1. 当目前的指针移动到新的slot上，检查该slot上的过期任务。在实现中，一般由独立的线程来完成。

```java
	long relativeIndex = (delay + this.tickDuration - 1L) / this.tickDuration;
	long remainingRounds = relativeIndex / this.wheel.length;
	int stopIndex = (int)(this.wheelCursor + relativeIndex & this.mask);
	timeout.stopIndex = stopIndex;
	timeout.remainingRounds = remainingRounds;

	this.wheel[stopIndex].add(timeout);
```
![HashedWheel](wheel.png)

原则上HashedWheelTimer以损失时间控制精度来换取高性能，对于大部分超时任务而言，一秒以内的误差一般都是可以接受的。

HashedWheelTimer 在netty中有现成的实现。不过在netty3和netty4中略有区别。

## Netty 3

每个tick里面维护的timeout任务列表用一个concurrentHashMap(准确的说是ConcurrentIdentityHashMap)保存，虽然这个数据结构支持并发，但是无法保证强数据一致性。 也就是说，多个timeout任务可以同时在同一个tick中add或者cancel，但是不能保证timeout线程同时遍历整个map务并处理其中的到期任务。

所以在netty3中，当外部线程添加timeout任务的时候，会用read lock加锁，但是当timeout线程对某个tick开始检查到期任务时就用write lock，防止同时有新的任务加入。

schedule the timeout task:
```java
  void scheduleTimeout(HashedWheelTimeout timeout, long delay)
  {
    long relativeIndex = (delay + this.tickDuration - 1L) / this.tickDuration;
    if (relativeIndex < 0L) {
      relativeIndex = delay / this.tickDuration;
    }
    if (relativeIndex == 0L) {
      relativeIndex = 1L;
    }
    if ((relativeIndex & this.mask) == 0L) {
      relativeIndex -= 1L;
    }
    long remainingRounds = relativeIndex / this.wheel.length;
    
    this.lock.readLock().lock();
    try
    {
      if (this.workerState.get() == 2) {
        throw new IllegalStateException("Cannot enqueue after shutdown");
      }
      int stopIndex = (int)(this.wheelCursor + relativeIndex & this.mask);
      timeout.stopIndex = stopIndex;
      timeout.remainingRounds = remainingRounds;
      
      this.wheel[stopIndex].add(timeout);
    }
    finally
    {
      this.lock.readLock().unlock();
    }
  }

```
process the expired tasks:
```java

      HashedWheelTimer.this.lock.writeLock().lock();
      try
      {
      	...
        fetchExpiredTimeouts(expiredTimeouts, i, deadline);
      }
      finally
      {
        HashedWheelTimer.this.lock.writeLock().unlock();
      }


```

因为这个writelock的存在，在运行过程中可能会短时间阻塞添加timeout任务的业务线程。影响取决于slot上存在的任务数量。

以前有个同事使用HashedWheelTimer来处理request timeout，正常情况下，大部分情况下response会正常返回，原理的timeout任务应该会被cancel，但是这位同事不是直接cancel这个任务，而是用在任务里面去判断cancel状态
```java
if( response != null){
	return
}
// process the timeout logic
```
虽然function没问题，但是在performance test中，connector的性能不是很稳定，原因就是任务没有主动cancel，导致slot里面的任务列表太长，writelock持有时间太长，影响了正常的request发送。虽然修改同时代码解决了这个问题，但是当时个人认为这个lock的粒度应该在slot层面而不是在整个wheel上，这样可以减少锁竞争，但是netty4给出了跟好的方案。

## Netty 4
在netty4中，任务不是直接添加到wheel上，而是先放到一个队列里面。然后由wheel线程把任务从队列挪到wheel上(如果任务已经取消了，就直接扔掉了)。这样子整个wheel上的操作都变成了单线程，上面的问题就没有了。

## 使用小结
1. 无论用什么scheduler，当任务应该取消时，尽早取消。其实这个无关scheduler实现，尽早取消可以释放内存减小GC的压力，防止对象进入old区。
2. tickduration 取决与时间的控制精度，一般默认配置就可以了。
3. 至于tick的数量，如果大部分任务的超时时间差不多，个人建议设置合适的tick数量使得 tickduration*ticknumber 略大于超时时间。