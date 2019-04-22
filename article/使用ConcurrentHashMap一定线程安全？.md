#### 前言
老王为何半夜惨叫？几行代码为何导致服务器爆炸？说好的线程安全为何还是出问题？让我们一起收看今天的《走进IT》

#### 正文

##### CurrentHashMap出现背景

说到ConcurrentHashMap的出现背景，还得从HashMap说起。


老王是某公司的苦逼Java开发，在互联网行业中，业务总是迭代得非常快。体现在代码中的话，就是v1.0的模块是单线程执行的，这时候使用HashMap是一个不错的选择。然而到了v1.5的版本，为了性能考虑，老王觉得把这段代码改成多线程会更有效率，那么说改就改，然后就愉快的发布上线了。

直到某天晚上，突然收到线上警报，服务器CPU占用100%。这时候惊醒起来一顿排查（百度，谷歌），结果发现原来是HashMap 在并发的环境下进行rehash的时候会造成链表的闭环，因此在进行get()操作的时候导致了CPU占用100%。喔，原来HashMap不是线程安全的类，在当前的业务场景中会有问题。那么你这时候又想到了Hashtable，没错，这是个线程安全的类，那我先用这个类替换不就行了，一顿commit，push，部署上去了，观察了一段时间，完美~再也没出现过类似的问题了。

但是好日子过的并不长久，运营的同事又找上门了，老王啊，XX功能怎么慢了这么多啊？这时候老王就纳闷了，我没改代码啊？不就上次替换了一个Hashtable，难道这里会有效率的问题？然后又是一顿排查（百度、谷歌），我去，果不其然，原来它线程安全的原因是因为在方法上都加了synchronized，导致我们全部操作都串行化了，难怪这么慢。

经过了2次掉陷阱的经验，这次的老王已经是非常谨慎的去寻求更好的解决方案了,这时他找到ConcurrentHashMap，而且为了避免再次掉坑他也去提前了解了实现原理，原来这个类是使用了Segment分段锁，每一个Segment都有自己的锁，这样冲突的的范围就变小了，效率也能提高不少。经过调研发现确实不错，于是他就放心的把Hashtable给替换掉了，从此运营再也没来吐槽了，老王又过上了幸福的日子。

经过一段时间紧张的业务开发，此时的项目已经去到了v2.0，之前的ConcurrentHashMap相关的代码已经被改的面目全非，逻辑也复杂了很多，但项目还是按时顺利的上线了。在项目在运行了一段时间以后，居然再次出现线程安全的问题，其根源竟然是ConcurrentHashMap，老王叕陷入了沉思...



##### 为何会出问题?
抛开复杂的例子，我们用一个多线程并发获取map中的值并加1，看看最后输出的数字如何

```java
public class CHMDemo {
    public static void main(String[] args) throws InterruptedException {
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<String,Integer>();
        map.put("key", 1);
        ExecutorService executorService = Executors.newFixedThreadPool(100);
        for (int i = 0; i < 1000; i++) {
            executorService.execute(new Runnable() {
                @Override
                public void run() {
                    int key = map.get("key") + 1; //step 1
                    map.put("key", key);//step 2
                }
            });
        }
        Thread.sleep(3000); //模拟等待执行结束
        System.out.println("------" + map.get("key") + "------");
        executorService.shutdown();
    }
}
```


此时我们看看多次执行输出的结果

```
------790------
------825------
------875------
```

通过观察输出结果可以发现，这段使用ConcurrentHashMap的代码，产生了线程安全的问题。我们来分析一下为什么会发生这种情况。在step1跟step2中，都只是调用ConcurrentHashMap的方法，各自都是原子操作，是线程安全的。但是他们组合在一起的时候就会有问题了，A线程在进入方法后，通过map.get("key")拿到key的值，刚把这个值读取出来还没有加1的时候，线程B也进来了，那么这导致线程A和线程B拿到的key是一样的。不仅仅是在

ConcurrentHashMap，在其他的线程安全的容器比如Vector之类的也会出现如此情况，所以在使用这些容器的时候还是不能大意。

##### 如何解决？
1、可以用synchronized
```java
synchronized(this){
    //step1
    //step2
}

```
但是用这种方法的话，我们要考虑一下效率的问题，会不会对当前的业务影响很大?

2、用原子类
```java
public class CHMDemo {
    public static void main(String[] args) throws InterruptedException {
        ConcurrentHashMap<String, AtomicInteger> map = new ConcurrentHashMap<String,AtomicInteger>();
        AtomicInteger integer = new AtomicInteger(1);
        map.put("key", integer);
        ExecutorService executorService = Executors.newFixedThreadPool(100);
        for (int i = 0; i < 1000; i++) {
            executorService.execute(new Runnable() {
                @Override
                public void run() {
                    map.get("key").incrementAndGet();
                }
            });
        }
        Thread.sleep(3000); //模拟等待执行结束
        System.out.println("------" + map.get("key") + "------");
        executorService.shutdown();
    }
}
```

```
------1001------
```
此时的输出结果就正确了，效率上也比第一种解决方案提高很多。

#### 结语

人生处处是陷阱，写代码也是如此，多思考，多留心。

---

#### 推荐阅读

《[大白话搞懂什么是同步/异步/阻塞/非阻塞](https://mp.weixin.qq.com/s/IT13vku21IMPv4aHuHG9lQ)》
《[Java异常处理最佳实践及陷阱防范](https://mp.weixin.qq.com/s/zeGqY0ZcrU_oOHpVW9V3zQ)》  
《[论JVM爆炸的几种姿势及自救方法](https://mp.weixin.qq.com/s/2oLX-i5zbTNayjJzAOSN8A)》   
《[解放程序员双手之Supervisor](https://mp.weixin.qq.com/s/cMvrhqSJrNBYoq5NJqTT5w)》  


<p align="center">
有收获的话，就分享给更多的朋友吧<br/>
<b>关注「深夜里的程序猿」，分享最干的干货</b>
</p>
<p align="center">
<img src="/resource/qrcode.png" alt="Sample"  width="200" height="200">
</p>