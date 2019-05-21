### 前言

在我们的程序开发过程中，如果涉及到多线程环境，那么对于集合框架的使用就必须更加谨慎了，因为大部分的集合类在不施加额外控制的情况下直接在并发环境中直接使用可能会出现数据不一致的问题，所以为了解决这个潜在的问题，我们要么在自己的业务逻辑中加上一些额外的控制，例如`锁`，或者我们直接使用Java提供的可在并发环境中使用的集合类,这是一个简便而且高效的方法。那么我们下面就来了解下Java提供了哪些“神器”可以让我们安全的使用集合。

### 正文

####  非阻塞式安全列表 - ConcurrentLinkedDeque

ConcurrentLinkedDeque可以在并发环境中直接使用，所谓的非阻塞，就是当列表为空的时候，我们还继续从列表中取数据的话，它会直接返回null或者抛出异常。下面列出来一些常用的方法。

* `peekFirst()`、`peekLast()` ：返回列表中首位跟末尾元素，如果列表为空则返回null。返回的元素不从列表中删除。
* `getFirst()`、`getLast()` ：返回列表中首位跟末尾元素，如果列表为空则抛出`NoSuchElementExceotion`异常。返回的元素不从列表中删除。
* `removeFirst()`、`removeLast()` ：返回列表中首位跟末尾元素，如果列表为空则抛出`NoSuchElementExceotion`异常。【返回的元素会从列表中删除】。



####  阻塞式安全列表 - LinkedBlockingDeque

LinkedBlockingDeque是一个阻塞式的线程安全列表，它跟 ConcurrentLinkedDeque最大的区别就是，当列表中元素满了或者为空的时候，我们对该列表的操作不会立即返回，而是阻塞当前操作，直到该操作可以执行时才返回。我们对比着上面ConcurrentLinkedDeque的常用方法，来看下LinkedBlockingDeque会有哪些不一致的地方呢？

* `put()` ：插入元素至列表中，当表中元素已满的时候，该操作将会被阻塞，直到表中存在空余空间。
* `take()` : 从列表中获取元素，当列表为空，该操作会被阻塞，直到列表不为空。
* `peekFirst()`、`peekLast()` ：返回列表中首位跟末尾元素，如果列表为空则返回null。返回的元素不从列表中删除。
* `getFirst()`、`getLast()` ：返回列表中首位跟末尾元素，如果列表为空则抛出`NoSuchElementExceotion`异常。返回的元素不从列表中删除。
* `addFirst()`、`addLast()` ：将元素添加至首位跟末尾，如果列表已满，则会抛出`IllegalStateException`

可以看出不管是从获取还是插入元素，都多了不少“花样”，其差别就在于是否阻塞，不满足条件是否返回null，不满足条件是否抛异常这几个方面来区分。


#### 优先级排序阻塞式安全列表 - PriorityBlockingQueue

相信大家都写过把某个列表元素按照特定的规则来排序之类的代码，在`PriorityBlockingQueue`中，存放进去的元素必须要实现Comparable接口。在这个接口中，有一个`compareTo()`方法,当执行该方法的对象跟参数传入的对象进行比较的时候，这个方法会返回一个数字值，如果值小于0，则当前对象小于参数传入对象。大于0则相反，等于0就表示两个对象相等。

```java
public class DemoObj implements Comparable<DemoObj> {
   
    private int priority;
    
    @Override
    public int compareTo(DemoObj do){
        if(this.getPriority() > do.getPriority()){
            return -1;
        }else if(this.getPriority() < do.getPriority()){
            return 1;
        }
        return 0;
    }
    
    //省略getset ...
}


//==== use ===================

PriorityBlockingQueue<DemoObj> queue = new PriorityBlockingQueue()<>;
queue.put(DemoObj);
queue.peek();
```

其常用方法跟上面提到的类基本都差不多大家可以动手实现一下，简单对比的话，可以说是LinkedBlockingDeque的增强版，多了元素排序功能。



#### 延迟元素线程安全列表 - DelayQueue

DelayQueue 里面存放着带有日期的元素，当我们从列表获取数据的时候，未到时间的元素将会被忽略。因此，存放进来的元素必须实现Delayed接口，使之成为一个延迟对象。

```java
/**
 *  compareTo方法与getDelay方法需排序一致
 */
class Order implements Delayed{

    private String name ;
    private long start = System.currentTimeMillis();
    private long time ;

    public MyDelayedTask(String name,long time) {
        this.name = name;
        this.time = time;
    }

    /**
     * 需要实现的接口，获得延迟时间   用过期时间-当前时间
     * @param unit
     * @return
     */
    @Override
    public long getDelay(TimeUnit unit) {
        return unit.convert((start+time) - System.currentTimeMillis(),TimeUnit.MILLISECONDS);
    }

    /**
     * 延迟队列内部排序   当前对象延迟时间 - 入参对象的延迟时间
     * @param o
     * @return
     */
    @Override
    public int compareTo(Delayed o) {
        Order o1 = (Order) o;
        return (int) (this.getDelay(TimeUnit.MILLISECONDS) - o1.getDelay(TimeUnit.MILLISECONDS));
    }
}
```

使用方式如下

```java
    private static DelayQueue delayQueue  = new DelayQueue();
    public static void main(String[] args) throws InterruptedException {

        new Thread(new Runnable() {
            @Override
            public void run() {

                delayQueue.offer(new Order("t3000",3000));
                delayQueue.offer(new Order("t4000",4000));
                delayQueue.offer(new Order("t2000",2000));
                delayQueue.offer(new Order("t6000",6000));
                delayQueue.offer(new Order("t1000",1000));

            }
        }).start();

        while (true) {
            Delayed take = delayQueue.take();
        }
    }
```

关于结果的输出大家可以动手尝试一下~


### 结语

这里仅仅是介绍了几种常用的并发集合，其目的主要是让大家对这些集合有一个直观的认识，在使用的时候可以思考下自己的场景用哪种更合适，如果当前介绍的类没合适的，那么是否还有其他并发集合会更有用呢？这里就当做抛砖引玉吧，有兴趣的朋友可以多去了解一下相关技术，相信你会有不少收获的。



