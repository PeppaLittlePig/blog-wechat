### 前言

在之前的文章《[一文彻底搞懂面试中常问的各种“锁”](https://juejin.im/post/5ca2fc1bf265da307d448e00 "标题")》中介绍了Java中的各种“锁”，可能对于不是很了解这些概念的同学来说会觉得有点绕，所以我决定拆分出来，逐步详细的介绍一下这些锁的来龙去脉，那么这篇文章就先来会一会“自旋锁”。

### 正文

#### 出现原因

在我们的程序中，如果存在着大量的互斥同步代码，当出现高并发的时候，系统内核态就需要不断的去挂起线程和恢复线程，频繁的此类操作会对我们系统的并发性能有一定影响。同时聪明的JVM开发团队也发现，在程序的执行过程中锁定“共享资源“的时间片是极短的，如果仅仅是为了这点时间而去不断挂起、恢复线程的话，消耗的时间可能会更长，那就“捡了芝麻丢了西瓜”了。

而在一个多核的机器中，多个线程是可以并行执行的。如果当后面请求锁的线程没拿到锁的时候，不挂起线程，而是继续占用处理器的执行时间，让当前线程执行一个忙循环（自旋操作），也就是不断在盯着持有锁的线程是否已经释放锁，那么这就是传说中的自旋锁了。

#### 自旋锁开启

虽然在JDK1.4.2的时候就引入了自旋锁，但是需要使用“-XX:+UseSpinning”参数来开启。在到了JDK1.6以后，就已经是默认开启了。下面我们自己来实现一个基于CAS的简易版自旋锁。

```java

public class SimpleSpinningLock {

    /**
     * 持有锁的线程，null表示锁未被线程持有
     */
    private AtomicReference<Thread> ref = new AtomicReference<>();

    public void lock(){
        Thread currentThread = Thread.currentThread();
        while(!ref.compareAndSet(null, currentThread)){
            //当ref为null的时候compareAndSet返回true，反之为false
            //通过循环不断的自旋判断锁是否被其他线程持有
        }
    }

    public void unLock() {
        Thread cur = Thread.currentThread();
        if(ref.get() != cur){
            //exception ...
        }
        ref.set(null);
    }
}

```

简简单单几行代码就实现了一个简陋的自旋锁，下面我们来测试一下

```java
public class TestLock {

    static int count  = 0;

    public static void main(String[] args) throws InterruptedException {
       ExecutorService executorService = Executors.newFixedThreadPool(100);
       CountDownLatch countDownLatch = new CountDownLatch(100);
       SimpleSpinningLock simpleSpinningLock = new SimpleSpinningLock();
       for (int i = 0 ; i < 100 ; i++){
           executorService.execute(new Runnable() {
               @Override
               public void run() {
                   simpleSpinningLock.lock();
                   ++count;
                   simpleSpinningLock.unLock();
                   countDownLatch.countDown();
               }
           });

       }
       countDownLatch.await();
       System.out.println(count);
    }
}

// 多次执行输出均为：100 ，实现了锁的基本功能
```

通过上面的代码可以看出，自旋就是在循环判断条件是否满足，那么会有什么问题吗？如果锁被占用很长时间的话，自旋的线程等待的时间也会变长，白白浪费掉处理器资源。因此在JDK中，自旋操作默认10次，我们可以通过参数“-XX:PreBlockSpin”来设置，当超过来此参数的值，则会使用传统的线程挂起方式来等待锁释放。


#### 自适应自旋锁

随着JDK的更新，在1.6的时候，又出现了一个叫做“自适应自旋锁”的玩意。它的出现使得自旋操作变得聪明起来，不再跟之前一样死板。所谓的“自适应”意味着对于同一个锁对象，线程的自旋时间是根据上一个持有该锁的线程的自旋时间以及状态来确定的。例如对于A锁对象来说，如果一个线程刚刚通过自旋获得到了锁，并且该线程也在运行中，那么JVM会认为此次自旋操作也是有很大的机会可以拿到锁，因此它会让自旋的时间相对延长。但是如果对于B锁对象自旋操作很少成功的话，JVM甚至可能直接忽略自旋操作。因此，自适应自旋锁是一个更加智能，对我们的业务性能更加友好的一个锁。


### 结语   

本来想着在一篇文章里面把“自旋锁”，“锁消除”，“锁粗化”等一些锁优化的概念都介绍完成的，但是发现可能篇幅会比较大，对于没怎么接触过这一块的同学来说理解起来会比较吃力，所以决定分开多个章节介绍，希望大家都不懂的地方可以多看几遍，慢慢体会，相信你会有所收获的。

#### 推荐阅读

《[如何优化代码中大量的if/else,switch/case?](https://mp.weixin.qq.com/s/ImswCn_eL4jBleET7fJVdw)》  
《[如何提高使用Java反射的效率？](https://mp.weixin.qq.com/s/-HXqicBROZU8XDF5YSCHMw)》  
《[Java日志正确使用姿势](https://mp.weixin.qq.com/s/aQx2ROajH2SqgHL77yxW3Q)》   
《[Java异常处理最佳实践及陷阱防范](https://mp.weixin.qq.com/s/zeGqY0ZcrU_oOHpVW9V3zQ)》    
《[论JVM爆炸的几种姿势及自救方法](https://mp.weixin.qq.com/s/2oLX-i5zbTNayjJzAOSN8A)》    


<p align="center">
有收获的话，就分享给更多的朋友吧<br/>
<b>关注「深夜里的程序猿」，分享最干的干货</b>
</p>
<p align="center">
<img src="/resource/qrcode.png" alt="Sample"  width="200" height="200">