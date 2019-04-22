#### 前言

关于日志，在大家的印象中都是比较简单的，只须引入了相关依赖包，剩下的事情就是在项目中“尽情”的打印我们需要的信息了。但是往往越简单的东西越容易让我们忽视，从而导致一些不该有的bug发生，作为一名严谨的程序员，怎么能让这种事情发生呢？所以下面我们就来了解一下关于日志的那些正确使用姿势。

#### 正文

##### 日志规范

###### 命名

首先是日志文件的命名，尽量要做到见名知意，团队里面也必须使用统一的命名规范，不然“脏乱差”的日志文件会影响大家排查问题的效率。这里推荐以“projectName_logName_logType.log”来命名，这样通过名字就可以清晰的知道该日志文件是属于哪个项目，什么类型，有什么作用。例如在我们MessageServer项目中**监控**Rabbitmq 消费者相关的日志文件名可以定义成“messageserver_rabbitmqconsumer_monitor.log”。

###### 保存时间

关于日志保存的时间，普通的日志文件建议保留15天，若比较重要的可根据实际情况延长，具体请参考各自服务器磁盘空间以及日志文件大小作出最优选择。

###### 日志级别

常见的日志级别有以下：

* DEBUG级别:记录调试程序相关的信息。
* INFO级别:记录程序正常运行有意义的信息。
* WARN级别:记录可能会出现潜在错误的信息。
* ERROR级别:记录当前程序出错的信息，需要被关注处理。
* Fatal级别：表示出现了严重错误，程序将会中断执行。

建议在项目中使用这四种级别， ERROR、WARN、INFO 、DEBUG。
##### 正确姿势

1、提前判断日志级别
```java
//条件判断
if(logger.isDebugEnabled){
    logger.debug("server info , id : " + id + ", user : " + user);
}

//使用占位符
logger.debug("server info , id : {}, user : {}",id,user);
```
对于DEBUG,INFO级别的日志，在我们的程序中是比较高频的存在，当我们的项目大了，日志变多了，这时候为了程序运行的效率，我们必须以条件判断或者占位符的方式来打印日志。为什么呢？假如我们项目中配置的日志级别为WARN,那么对于我们下面的日志输出语句‘ logger.debug("server info , id : " + id + ", user : " + user);’，虽然该日志不会被打印，但是却会执行字符串拼接的操作，这里我们的user是一个实例对象，所以还会执行toString方法，这样就白白浪费了不少系统的资源。

2、避免多余日志输出

在我们的生产环境中，一般禁止DEBUG日志的输出，其打印的频率是非常高的，容易对正常运行的程序造成严重的影响，在我们最近的项目中就有遇到过类似的情况。

那么这时候该学会使用additivity属性
```
<logger name="xx" additivity="true">
```
在这边配置成true的话，也就是默认的情况，这时候当前Logger会继承父Logger的Appender，说白了就是当前日志的输出除了输出在当前日志文件以外，还会输出至父文件里。所以一般情况下，我们为了避免重复打印，会将这个参数设置成false，以减少不必要的输出。

3、保证日志记录信息完整

在我们的代码中，日志记录的内容要包含异常的堆栈，请勿随意输出“XX出错”等简单的日志，这对于错误的调试毫无帮助。所以我们在记录异常的时候一定要带上堆栈信息，例如
```
logger.error("rabbitmq consumer error,cause : "+e.getMessage(),e);
```
切记在输出对象实例的时候，须确保对象重写了toString方法，否则只会输出其hashCode值。

4、定义logger变量为static

```
private static final Logger logger = LoggerFactory.getLogger(XX.class);
```
确保一个对象只使用一个Logger对象，避免每次都重新创建，否则可能会导致OOM。

5、正确使用日志级别

```java
try{
    //..
}catch(xx){
    logger.info(..);
}
```
这样一来，本来是ERROR的信息，全都打印在INFO日志文件里了，不知情的同事还会在死盯着错误日志，而且还找不出问题，多影响工作效率是吧?


6、推荐使用slf4j+logback组合

logback库里自身就已经实现了slf4j的接口，就无需引入多余的适配器了，而且logback也具有更多的优点，建议新项目可以使用这个组合。
还有一点需要注意，当引入slf4j后，要注意其实际使用的日志库是否是由我们引入的，也有可能会使用了我们第三方依赖包所带入的日志库，这样就可能会导致我们的日志失效。

7、日志的聚合分析

日志的聚合可以把位于不同服务器之间的日志统一起来分析处理，如今ELK技术栈亦或者的EFG（fluentd+elasticsearch+grafana）等都是一些比较成熟的开源解决方案。  

拿ELK来说，可以在我们的服务器上直接通过logstash来读取应用打印的日志文件，或者也可以在我们项目中的日志配置文件里配置好相关的socket信息，打印的时候直接把日志信息输出至logstash。再交由elasticsearch存储，kibana展示。

#### 结语

好了，关于日志先聊这么多~ 大家有需要补充或者交流的可以在下方留言哦。

---

#### 推荐阅读

《[使用ConcurrentHashMap一定线程安全吗？](https://mp.weixin.qq.com/s/IT13vku21IMPv4aHuHG9lQ)》  
《[大白话搞懂什么是同步/异步/阻塞/非阻塞](https://mp.weixin.qq.com/s/TW82I31CVRbKOwJGnTTP8A)》  
《[Java异常处理最佳实践及陷阱防范](https://mp.weixin.qq.com/s/zeGqY0ZcrU_oOHpVW9V3zQ)》    
《[论JVM爆炸的几种姿势及自救方法](https://mp.weixin.qq.com/s/2oLX-i5zbTNayjJzAOSN8A)》    


<p align="center">
有收获的话，就分享给更多的朋友吧<br/>
<b>关注「深夜里的程序猿」，分享最干的干货</b>
</p>
<p align="center">
<img src="/resource/qrcode.png" alt="Sample"  width="200" height="200">
</p>