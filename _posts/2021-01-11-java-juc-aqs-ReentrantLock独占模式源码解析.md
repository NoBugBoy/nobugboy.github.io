## Aqs源码解析
从ReentrantLock入手看AQS独占模式

> 实例化ReentrantLock

在实例化ReentrantLock时，同时实例化了内部的非公平Aqs的类NonfairSync（***默认实现为非公平同步类***）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111115619528.png)

> lock方法

当我们想要进行同步操作时，会调用lock方法，lock方法实际调用的为Aqs中的抽象方法Lock，这里的具体实现为NonfairSync的lock方法
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111113612536.png)
接下来我们来看一下，非公平的lock方法具体做了什么，如下图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111113716845.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
当我们尝试调用NonfairSync的lock时，他会CAS直接比较state，如果是0就更新为1 不关心是否有人排队（***不讲武德模式***），更新成功就将当前线程设置为独占状态，言外之意就是当前线程持有锁了（ ***这里的state可以理解为一个计数器，当有人占有锁时 state > 0*** ）此时如果其他线程尝试lock，则进入acquire方法
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111130930328.png)


> nonfairTryAcquire方法

当不讲武德操作失败或有人持有锁的时候，就会进入acquire方法，当前锁如果无人占用，直接cas修改state成功后将当前线程设置为独占返回 ***True***，否则返回**False**，如果当前线程已经持有锁，再次获取锁（***也就是重入***）那么state加1,返回**True**
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111130330924.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)

> addWaiter

当nonfairTryAcquire方法返回**True**时，代表获取锁，或重入成功，流程结束，如果返回为False时，表示获取锁失败，进入排队状态，从这里就开始引入了Node的概念，Node是CLH队列中的节点,也是Aqs抽象类中定义的内部类
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111131700567.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)

```java
      Wait queue node class.
           +------+  prev +-----+       +-----+
      head |      | <---- |     | <---- |     |  tail
           +------+       +-----+       +-----+
```
这里传入了Node.EXCLUSIVE，此时EXCLUSIVE如上图，为null，在这里创建了第一个节点并将当前线程绑定在node上，当tail为空时，进入enq方法，***否则将tail节点设置为新节点的前一个节点，并cas将tail节点改为新的节点，（tail = node无锁会有并发问题，你懂得）*** ，如果cas设置失败则也需要进入enq方法
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111131320948.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)

> enq

addWaiter，如果tail是空或者cas设置尾节点失败时会进入该方法，这里分两种情况

 1. 当尾节点为空时（一般是第一次进入等待的时候），会创建一个空node作为head节点（
      CLH queues need a dummy header node to get started. But
     we don't create them on construction, because it would be wasted
     effort if there is never contention. Instead, the node
     is constructed and head and tail pointers are set upon first
     contention.），并且设置 tail  就是 head ，第二次循环进入else，将tail设置为当前节点的前一个节点，并cas设置尾节点为当前节点（***也就是将自己排在队尾***），设置失败了再次循环直到成功为止  
 2. 已经有尾结点就直接cas去设置尾节点为当前节点的头节点（***也就是将自己排在队尾***），设置失败了再次循环直到成功为止  


![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111132931925.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)

>  acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) , arg = 1

此时addWaiter会返回一个形成了CLH链表状态的的node，node.predecessor() 返回当前节点的前置节点（***如果是第一次排队则是空的head节点***），此时又有两种状态

 1. 如果前置节点就是head的话，直接就去nonfairTryAcquire（***言外之意就是我前面是空头节点，我就是排在第一位的有效节点直接尝试获取锁***）如果获取成功，改node的线程持有独占锁，***并将该第一个有效节点设置为头节点*** 并设置next为空，错误标识为False，***并返回打断状态为False***
 2. 如果当前节点的前置节点不是头节点（***说明已经开始排长队了***) , 翻到下面继续看

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111134303987.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)

> acquireQueued的第二种情况

此时已经排长队的情况下，会调用shouldParkAfterFailedAcquire方法，该方法会传递 ***{ 当前节点的前置节点 }*** 和 ***{当前节点 }*** 在这又引入了waitStatus
```java
   /** waitStatus value to indicate thread has cancelled */
    static final int CANCELLED =  1;
    /** waitStatus value to indicate successor's thread needs unparking */
    static final int SIGNAL    = -1;
    /** waitStatus value to indicate thread is waiting on condition */
    static final int CONDITION = -2;
    /**
     * waitStatus value to indicate the next acquireShared should
     * unconditionally propagate
     */
    static final int PROPAGATE = -3;
    
    //waitStatus默认初始化值为0,除非在构造时传入指定的值否则均为0
    volatile int waitStatus;
    The field is initialized to 0 for normal sync nodes, and
    CONDITION for condition nodes.  It is modified using CAS
    (or when possible, unconditional volatile writes).
```

  由于是waitStatus是0，所以进入else里，会cas将当前节点的前置节点的waitStatus ***从0修改为-1*** 返回False ，这里会不断的循环查看每个节点前置节点的状态，***直到找到前置节点为头节点***
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111135520460.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
接下来又调用了parkAndCheckInterrupt方法，这里就是核心的上锁方法,底层是通过LockSupport去控制的，***这里会锁上当前的线程等待unPark释放***
```java
  LockSupport.park(this);
  # if interrupted while waiting
  Thread.interrupted();
```
如果在等待时被打断返回True，此时acquireQueued返回True或False。

***此时回到下图这里*** ，nonfairTryAcquire为False，acquireQueued为True时会打断自身 Thread.currentThread().interrupt(); ，（***言外之意,没有获取到锁，并且线程已经被中断***），cancelAcquire只有在抛异常时会执行这里先不看

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111144131789.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)

FairSync和NonfairSync的区别就在于NonfairSync会不排队，先去抢占锁，如果没抢到才去乖乖排队，FairSync则不先去抢锁。

lock方法到这就结束了，接下里是unlock。

> unlock

这里会调用释放方法
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021011115064816.png)

> tryRelease

tryRelease返回True，则获取当前头节点，如果不是空并且不是新建状态，unparkSuccessor会唤醒下一个节点
```java
 Node s = node.next;
 LockSupport.unpark(s.thread)
```
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111150730884.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)
这里的state就是开头说的计数器，在释放锁时会在这里减1，如果减1后是0证明该线程的锁全部释放完毕，设置独占锁为空，并且设置state为0，返回True,否则还需要继续unlock释放该线程的独占锁资源
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210111150828626.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0RheV9EYXlfTm9fQnVn,size_16,color_FFFFFF,t_70)