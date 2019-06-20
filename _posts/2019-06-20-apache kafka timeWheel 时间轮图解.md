---
layout:     post
title:      apache kafka timeWheel 时间轮简单图解
subtitle:   了解kafka时间轮工作流程
date:       2019-06-20
author:     LC
header-img: /img/post-bg-map.jpg
catalog: true  
---

### 目录
- 回顾：什么是时间轮
- 为什么环形队列高效
- kafka时间轮流程
- 时间轮如何添加任务
- 推动时间指针移动
- 思考

## 前言
时间轮(timeWheel)由于其特殊的数据结构，使得天生具备了处理定时任务的高效特性。时间轮被用作很多知名项目中，例如kafka、netty等。
这篇文章将带领我们了解kafka是如何运用时间轮的。

### 回顾：什么是时间轮

![图一](https://ws3.sinaimg.cn/large/005BYqpggy1g3rilywrkgj30vo0h80u1.jpg)
图一

如图，一排格子组成环形，每个格子有自己时间范围，当时间指针按顺序移到某个格子，该格子里的任务队列开始执行。
时间轮（TimingWheel）是一个存储定时任务的环形队列，底层采用数组实现，
数组中的每个元素可以存放一个定时任务列表（TimerTaskList）。
TimerTaskList是一个环形的双向链表，链表中的每一项表示的都是定时任务项（TimerTaskEntry），
其中封装了真正的定时任务TimerTask。

### 为什么环形队列高效：
1. 任务的添加与移除，都是O(1)级的复杂度；(因为可以快速定位到任务所在的桶位置，且桶里的任务是双向链表，易增删)
2. 不会占用大量的资源；
3. 只需要有一个线程去推进时间轮就可以工作了;
4. 不存在hash冲突;()
5. 表示的时间有限。


### kafka时间轮流程

![图二](https://ws3.sinaimg.cn/large/005BYqpggy1g3rj31ejj8j30zw0ptdhd.jpg)
图二 

上图是kafka时间轮的简单流程。

下面将配合代码进行分解。

```java
   public TimingWheel(Long tickMs, Integer wheelSize, Long startMs, AtomicInteger taskCounter, DelayQueue<TimerTaskList> queue) {
           this.tickMs = tickMs;                 //每个时间桶的时间粒度
           this.wheelSize = wheelSize;           // 一个环形层里包含多少个时间桶
           this.startMs = startMs;               // 时间轮的开始时间
           this.taskCounter = taskCounter; 
           this.queue = queue;                   //该队列负责推动指针
           this.interval = tickMs * wheelSize;   //时间指针转一圈的用时
           this.currentTime = startMs - (startMs % tickMs);
           this.buckets = new TimerTaskList[wheelSize];  // 构建数组，时间轮的数据结构
           for (int i = 0; i < buckets.length; i++) {    //每个桶里存储的任务以链表的形式存在
               buckets[i] = new TimerTaskList(taskCounter);
           }
       }
```
上述代码是kafka定义的时间轮构造方法。

### 时间轮添加任务
时间轮构造结束后，我们看看当试图向这个时间轮加一个定时任务时，代码是如何工作的。
```java
    private void addTimerTaskEntry(TimerTaskEntry timerTaskEntry) {
        if (!timingWheel.add(timerTaskEntry)) {
            // Already expired or cancelled
            if (!timerTaskEntry.cancelled()) {
                // 对于已经取消或者过期的任务，都放入线程池中
                taskExecutor.submit(timerTaskEntry.getTimerTask());
            }
        }
    }

TimingWheel.class

   public boolean add(TimerTaskEntry timerTaskEntry) {
           long expiration = timerTaskEntry.getExpirationMs();
           if (timerTaskEntry.cancelled()) {
               // Cancelled
               return false;
           } else if (expiration < currentTime + tickMs) {
               // Already expired
               return false;
           } else if (expiration < currentTime + interval) {
               long virtualId = expiration / tickMs;
               TimerTaskList bucket = buckets[(int) (virtualId % wheelSize)];
               bucket.add(timerTaskEntry);
               // Set the bucket expiration time
               if (bucket.setExpiration(virtualId * tickMs)) {
                   // 如果设置桶的触发时间成功，说明该桶中之前无定时任务
                   // 将有任务的桶放入队列，该队列负责推动时针，采用堆排序，决定何时触某个桶
                   queue.offer(bucket);
               }
               return true;
           } else {
               // 如果时间溢出，创建下一个时间轮
               if (overflowWheel == null) {
                   addOverflowWheel();
               }
               return overflowWheel.add(timerTaskEntry);
           }
       } 
       
TimerTaskList.class       
       // Add a timer task entry to this list
       public void add(TimerTaskEntry timerTaskEntry) {
           boolean done = false;
           while (!done) {
               timerTaskEntry.remove();
   
               synchronized (this) {
                   synchronized (timerTaskEntry) {
                       if (timerTaskEntry.getList() == null) {
                           // put the timer task entry to the end of the list. (root.prev points to the tail entry)
                           TimerTaskEntry tail = root.prev;
                           timerTaskEntry.next = root;
                           timerTaskEntry.prev = tail;
                           timerTaskEntry.setList(this);
                           tail.next = timerTaskEntry;
                           root.prev = timerTaskEntry;
                           taskCounter.incrementAndGet();
                           done = true;
                       }
                   }
               }
           }
       }
```
上述代码是时间轮添加新任务的过程，分为几个步骤

1. 创添加任务时.
  判断该任务的开始时间。如果这个任务已经取消或者当前时间到期，则返回false。
  
2. 返回false的任务，会被直接放入线程池中taskExecutor.submit(timerTaskEntry.getTimerTask())，等待执行。结合图中的线程池处理task，应该不难理解。

3. 对于时间未到期的任务，返回对于true。未到期的任务，又分两步骤;

4. 首先判断该任务是否在该层级的时间轮中expiration < currentTime + interval，
如果该任务所在的时间轮层级，就要判断该任务所在的桶，并加入到该桶bucket.add(timerTaskEntry)。
   如果该桶之前没有任务，此时将该桶加入任务队列中queue.offer(bucket);

5. 如果不在，放入下一层时间轮中overflowWheel.add(timerTaskEntry)，再循环以上从第一步开始。

6. 我们注意，在桶具体添加任务的过程中，使用了加锁机制。说明这个bucket在应对高并发时存在一定的瓶颈。
为什么作为大数据处理的kafka使用这么重的安全机制，我们稍后讨论。

### 推动时间指针
我们将任务存储在时间轮的桶中，那么如何知道何时该哪个桶里的任务执行，这就涉及到时间指针;

时间推动的方式有多种，事实上，kafka与netty就采用不同的时间指针推动的方法。

在图1中，时间指针被形象的画出来的;在图2中，时间指针是被标记红色的currentTime.
事实上，指针是跳跃走动了，不是连续平稳的向前划。当它跳跃到哪个桶的，那个桶就开始执行桶内的任务。
kafka的指针推动方式是利用了JDK自动的delayQueue延迟队列.该队列采用堆排序，感兴趣的可以去了解。

我们结合delayQueue和图2来了解kafka时间推动方式。

如图2,有一个“待处理buckets[i]”的队列。
我们在添加任务的第4条中提到过queue.offer(bucket)，这个queue就是delayQueue延迟堆队列，
存储的便是已经有任务的bucket.
队列里的序号从2、5开始，说明队列中的桶不是连续的，因为delayQueue仅加载存在任务的buckets。

delayQueue队列会判断何时将堆顶到期的桶pop出去，被pop出去的桶，会遍历桶桶中的任务，再次调用添加任务的方法。
再这次的添加中，任务会被加入到线程池队列中被执行。
```java
//遍历
         while (bucket != null) {
               timingWheel.advanceClock(bucket.getExpiration());
               bucket.flush(this);
               bucket = delayQueue.poll();
         }

   // Remove all task entries and apply the supplied function to each of them
    public void flush(Function<TimerTaskEntry, Void> f) {
        synchronized (this) {
            TimerTaskEntry head = root.next;
            while (head != root) {
                remove(head);
                f.apply(head);
                head = root.next;
            }
            expiration.set(-1L);
        }
    }

// 添加任务
    public Void apply(TimerTaskEntry timerTaskEntry) {
        addTimerTaskEntry(timerTaskEntry);
        return null;
    }
```

### 思考
1. 为什么处理大数据的kafka在添加任务过程中，使用可能影响效率的加锁来完成操作？

 A: 虽然kafka具有快速处理的数据的能力而常用来处理大数据任务。 但是kafka使用时间轮来实现的延时任务，
并不对外提供，producer和consumer不能要求自己的任务延迟到指定时间。 事实上，这个时间轮延迟任务是
供kakfa集群内部自己使用的，例如 延迟加入，延迟心跳，延迟生产，延迟拉取。

