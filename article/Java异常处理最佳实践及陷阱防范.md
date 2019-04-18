
#### 前言

不管在我们的工作还是生活中，总会出现各种“错误”，各种突发的“异常”。无论我们做了多少准备，多少测试，这些异常总会在某个时间点出现，如果处理不当或是不及时，往往还会导致其他新的问题出现。所以我们要时刻注意这些陷阱以及需要一套“最佳实践”来建立起一个完善的异常处理机制。

#### 正文

##### 异常分类

![](https://user-gold-cdn.xitu.io/2019/4/13/16a1586e46901104?w=902&h=508&f=png&s=24726)

首先，这里我画了一个异常分类的结构图。

在JDK中，Throwable是所有异常的父类，其下分为”Error“和”Exception“。Error意味着出现了不可控的严重错误，例如OutOfMemoryError。Exception则细分为两类，受检异常（check）需要我们手动try/catch或者在方法定义中throws，编译器在编译的时候会检查其合法性。非受检异常（uncheck）则不需要我们提前处理。这些简单的概念对于开发人员来说都是必须掌握的，这里就展示个图例，不做详细的描述了，我们的”正餐“还在后面。

##### 重新认识try/catch/finally

 说到异常处理，这里就不得不提try/catch/finally。try不可以单独存在，要么搭配catch，要么搭配finally，或者三者并存。  
 1、try代码块：监视代码块的执行，发现对应的的异常则跳转至catch，若无catch则直接到finally块。  
 2、catch代码块：发生对应的异常会执行里面的代码，要么处理，要么向上抛出。  
 3、finally代码块:不管是否有异常，都必执行，一般用来清理资源，释放连接等。然而有以下几种情况不会执行到这里的代码。  
 * 代码执行流程未进入try代码块。
 * 代码在try代码块中发生死循环、死锁等状态。
 * 在try代码块中执行了System.exit()操作。
 

##### try/catch/finally陷阱

下面介绍两个我们在使用tcf的时候可能会遇到的陷阱。  

代码1

```java
public class TCFDemo {
    public static void main(String[] args) {
        //11
        System.out.println(returnVal());
    }

    static int returnVal(){
        int a = 1;
        int b = 10;
        try{
            return ++a;
        }finally {
            return ++b;
        }
    }
}

```

陷阱1：在finally中添加return语句，这样会覆盖掉try代码return的值，假如业务逻辑比较复杂，这里是很容易掉坑的，不利于排查错误。

代码2


```java
public class TCFDemo {
    public static void main(String[] args) {
        Lock lock = new ReentrantLock();
       try{
            //有可能加锁失败
            lock.lock();
            //dost
       }finally {
           lock.unlock();
       }
    }
}
```
陷阱2：由于lock方法在加锁的时候有可能会抛出Uncheck异常，如果在try代码块中，必然会执行unlock方法，此时由于并没有加锁成功，所以会抛出IllegalMonitorStateException，这样一来后者的异常就覆盖掉了前者加锁失败的异常信息，所以我们应该把加锁的方法挪至try代码块外面。


##### 最佳实践

好了，前面简单介绍了异常的分类以及try/catch/finally的注意事项，现在可以总结一下我们在异常处理的时候有哪些”最佳实践“了。   

1. 当需要向上抛出异常的时候，需根据当前业务场景定义具有业务含义的异常，优先使用行业内定义的异常或者团队内部定义好的。例如在使用dubbo进行远程服务调用超时的时候会抛出DubboTimeoutException，而不是直接把RuntimeException抛出。
2. 请勿在finally代码块中使用return语句，避免返回值的判断变得复杂。
3. 捕获异常具体的子类，而不是Exception，更不是throwable。这样会捕获所有的错误，包括JVM抛出的无法处理的严重错误。
4. 切记更别忽视任何一个异常（catch住了不做任何处理），即使现在能确保不影响逻辑的正常运行，但是对于将来谁都无法保证代码会如何改动，别给自己挖坑。
5. 不要使用异常当作控制流程来使用，这是一个很奇葩也很影响性能的做法。
6. 清理资源，释放连接等操作一定要放在finally代码块中，防止内存泄漏，如果finally块处理的逻辑比较多且模块化，我们可以封装成工具方法调用，代码会比较简洁。


#### 结尾

小小的异常，有大大的学问，你觉得呢？


<p align="center">
        有收获的话，就分享给更多的朋友把<br/>
        <b>关注「深夜里的程序猿」，分享最干的干货</b>
        <img src="/resource/qrcode.png" alt="Sample"  width="200" height="200">
</p>