####  前言

在我们平时的工作或者面试中，都会经常遇到“反射”这个知识点，通过“反射”我们可以动态的获取到对象的信息以及灵活的调用对象方法等，但是在使用的同时又伴随着另一种声音的出现，那就是“反射”很慢，要少用。难道反射真的很慢？那跟我们平时正常创建对象调用方法比慢多少? 估计很多人都没去测试过，只是”道听途说“。下面我们就直接通过一些测试用例来直观的感受一下”反射“。

#### 正文

##### 准备测试对象

下面先定义一个测试的类*TestUser*，只有*id*跟*name*属性，以及它们的getter/setter方法，另外还有一个自定义的*sayHi*方法。

```java
public class TestUser {
    private Integer id;
    private String name;
    
    public String sayHi(){
        return "hi";
    }

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

##### 测试创建100万个对象

```java
// 通过普通方式创建TestUser对象
@Test
public void testCommon(){
    long start = System.currentTimeMillis();
    TestUser user = null;
    int i = 0;
    while(i<1000000){
        ++i;
        user = new TestUser();
    }
    long end = System.currentTimeMillis();
    System.out.println("普通对象创建耗时："+(end - start ) + "ms");
}

//普通对象创建耗时：10ms
```

```java
// 通过反射方式创建TestUser对象
@Test
public void testReflexNoCache() throws Exception {
    long start = System.currentTimeMillis();
    TestUser user = null;
    int i = 0;
    while(i<1000000){
        ++i;
        user = (TestUser) Class.forName("ReflexDemo.TestUser").newInstance();
    }
    long end = System.currentTimeMillis();
    System.out.println("无缓存反射创建对象耗时："+(end - start ) + "ms");
}

//无缓存反射创建对象耗时：926ms
```

在上面这两个测试方法中，笔者各自测了5次，把他们消耗的时间取了一个平均值，在输出结果中可以看到一个是10ms，一个是926ms，在创建100W个对象的情况下，反射居然慢了90倍左右。wtf？差距居然这么大？难道反射真的这么慢？下面笔者换一种反射的姿势，继续测试一下，看看结果如何？

```java
// 通过缓存反射方式创建TestUser对象
@Test
public void testReflexWithCache() throws Exception {
    long start = System.currentTimeMillis();
    TestUser user = null;
    Class rUserClass = Class.forName("RefleDemo.TestUser");
    int i = 0;
    while(i<1000000){
        ++i;
        user = (TestUser) rUserClass.newInstance();
    }
    long end = System.currentTimeMillis();
    System.out.println("通过缓存反射创建对象耗时："+(end - start ) + "ms");
}

//通过缓存反射创建对象耗时：41ms
```

咦？这种操作只需要41ms了，大大提高了反射创建对象的效率。为什么会快这么多呢？

其实通过代码我们可以发现，是Class.forName这个方法比较耗时，它实际上调用了一个本地方法，通过这个方法来要求JVM查找并加载指定的类。所以我们在项目中使用的时候，可以把Class.forName返回的Class对象缓存起来，下一次使用的时候直接从缓存里面获取，这样就极大的提高了获取Class的效率。同理，在我们获取Constructor、Method等对象的时候也可以缓存起来使用，避免每次使用时再来耗费时间创建。



##### 测试反射调用方法

```java
@Test
public void testReflexMethod() throws Exception {
    long start = System.currentTimeMillis();
    Class testUserClass = Class.forName("RefleDemo.TestUser");
    TestUser testUser = (TestUser) testUserClass.newInstance();
    Method method = testUserClass.getMethod("sayHi");
    int i = 0;
    while(i<100000000){
        ++i;
        method.invoke(testUser);
    }
    long end = System.currentTimeMillis();
    System.out.println("反射调用方法耗时："+(end - start ) + "ms");
}

//反射调用方法耗时：330ms
```
```java
@Test
public void testReflexMethod() throws Exception {
    long start = System.currentTimeMillis();
    Class testUserClass = Class.forName("RefleDemo.TestUser");
    TestUser testUser = (TestUser) testUserClass.newInstance();
    Method method = testUserClass.getMethod("sayHi");
    int i = 0;
    while(i<100000000){
        ++i;
        method.setAccessible(true);
        method.invoke(testUser);
    }
    long end = System.currentTimeMillis();
    System.out.println("setAccessible=true 反射调用方法耗时："+(end - start ) + "ms");
}

//setAccessible=true 反射调用方法耗时：188ms
```

这里我们反射调用sayHi方法1亿次，在调用了method.setAccessible(true)后，发现快了将近一半。查看API可以了解到，jdk在设置获取字段，调用方法的时候会执行安全访问检查，而此类操作会比较耗时，所以通过setAccessible(true)的方式可以关闭安全检查，从而提升反射效率。

##### 极致的反射

除了上面的手段，还有没有什么办法可以更极致的使用反射呢？这里介绍一个高性能反射工具包ReflectASM。它是通过字节码生成的方式来实现的反射机制，下面是一个跟java反射的性能比较。


![](https://user-gold-cdn.xitu.io/2019/4/25/16a55008b98e4347?w=700&h=62&f=png&s=5265)
![](https://user-gold-cdn.xitu.io/2019/4/25/16a55001783d17d9?w=700&h=62&f=png&s=5259)
![](https://user-gold-cdn.xitu.io/2019/4/25/16a5500b5ccbdfbd?w=700&h=62&f=png&s=5551)

这里就不介绍它的用法了，有兴趣的朋友可以直接传送过去：https://github.com/EsotericSoftware/reflectasm

#### 结语

最后总结一下，为了更好的使用反射，我们应该在项目启动的时候将反射所需要的相关配置及数据加载进内存中，在运行阶段都从缓存中取这些元数据进行反射操作。大家也不用惧怕反射，虚拟机在不断的优化，只要我们方法用的对，它并没有”传闻“中的那么慢，当我们对性能有极致追求的时候，可以考虑通过三方包，直接对字节码进行操作。


---

#### 推荐阅读

《[Java日志正确使用姿势](https://mp.weixin.qq.com/s/aQx2ROajH2SqgHL77yxW3Q)》  
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
</p>805b08e1?w=200&h=200&f=png&s=7078)