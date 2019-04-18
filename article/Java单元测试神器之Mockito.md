#### 什么是 Mock 测试


Mock测试就是在测试过程中，对于某些不容易构造或者不容易获取的对象，用一个虚拟的对象来创建以便测试的测试方法。什么是不容易构造的对象呢？例如HttpServletRequest，需要在有servlet容器环境中创建获取。那不容易获取的对象呢？如一个JedisCluster，需要准备redis相关环境，然后设置进去等等。



Mock 可以分解在单元测试中耦合的其他类或者接口，它能够帮你模拟这些依赖，并帮你验证所调用的依赖的行为。



#### 场景事例


![](https://user-gold-cdn.xitu.io/2019/4/3/169e0c30405cee2a?w=432&h=380&f=png&s=57074)

当我们需要测试OrderService时，按照我们常规的做法呢，都是要先准备好redis，跟db的环境，然后构造UserService跟CouponService注入进来，此时需要构建完整的依赖树，其过程是比较繁琐的，万一数据库连不上，依赖找不到，服务挂了... 时间一长可能会打击我们对项目进行单测的积极性，所以这时候很有必要寻求一种优雅的方式来解决。



铛铛铛~这时候Mockito出现了(java中Mock框架比较多，但是本篇只介绍这个)，它会把那些繁琐的依赖统统转化为Mock Object，如下图，这样我们就可以专注的进行我们的单测，减少在解决依赖上浪费的时间了。


![](https://user-gold-cdn.xitu.io/2019/4/3/169e0c3396cb3a67?w=435&h=328&f=png&s=56204)

#### 直接开干



关于Mockito的简介这里就不在赘述了，大家有兴趣可以自行去官方文档查阅，这里主要带大家了解一些常用的Mock方法。



maven依赖
```
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>2.23.4</version>
    <scope>test</scope>
</dependency>
```

为了代码测试的方便，直接在测试类中静态导入 import static org.mockito.Mockito.*;



#### 基础方法

```
@Test
   public void testMockBase(){
       //创建ArrayList的Mock对象
       List mockList = mock(ArrayList.class);
       //pass
       Assert.assertTrue(mockList instanceof ArrayList);

       //当我们mockList调用方法去add("张三")的时候会返回true
       when(mockList.add("张三")).thenReturn(true);
       //当我们mockList调用方法size()的时候返回10
       when(mockList.size()).thenReturn(10);
       //pass
       Assert.assertTrue(mockList.add("张三"));
       //pass
       Assert.assertFalse(mockList.add("李四"));
       //pass
       Assert.assertEquals(mockList.size(),10);
       //null
       System.out.println(mockList.get(0));
   }
```

mock静态方法会创建一个Mock对象，由于 Mock对象 并不会真的执行方法中的代码，所以如果未指定返回值的话会返回默认值（如19行）。第九、十行我们指定了mockList在执行特定方法后需要返回的值，所以在assertTrue校验是没问题的，但是add("李四")，我们并没设置，所以是false。



#### 校验方法调用次数

```
 //使用mock
 List mockedList = mock(ArrayList.class);
 mockedList.add("once");

 mockedList.add("twice");
 mockedList.add("twice");

 mockedList.add("three times");
 mockedList.add("three times");
 mockedList.add("three times");
 
 //这里默认是判断该方法调用times(1),同下
 verify(mockedList).add("once");
 verify(mockedList, times(1)).add("once");

 verify(mockedList, times(2)).add("twice");
 verify(mockedList, times(3)).add("three times");
 //从没调用，times(0)
 verify(mockedList, never()).add("never happened");
 //最少一次，最少几次，最多几次
 verify(mockedList, atLeastOnce()).add("three times");
 verify(mockedList, atLeast(2)).add("three times");
 verify(mockedList, atMost(5)).add("three times");
```

其实在上述的代码中，命名是比较直观的，所以我这边就直接注释在代码中了。



#### 校验方法调用时长

```
//方法执行在100ms以内的时候可以通过
   verify(mock, timeout(100)).someMethod();
   //同上
   verify(mock, timeout(100).times(1)).someMethod();

   //方法2次调用均没超过100ms
   verify(mock, timeout(100).times(2)).someMethod();

   verify(mock, timeout(100).atLeast(2)).someMethod();
```

通过超时检测可以校验我们的方法逻辑会不会有出现问题而导致超时的地方。



#### 参数匹配

```
linkedList.add("element");
// anyInt() 任何整数我们都返回 element 
when(linkedList.get(anyInt())).thenReturn("element");

System.out.print(linkedList.get(10));//返回element
```

#### 方法抛出异常

```
 @Test(expected = RuntimeException.class)
    public void doThrow(){
        List list = mock(List.class);
        doThrow(new RuntimeException()).when(list).add(1);
        list.add(1);
    }
```

#### 使用注解注入

```

  public class ArticleManagerTest {

       @Mock private ArticleCalculator calculator;
       @Mock private ArticleDatabase database;
       @Mock private UserProvider userProvider;

       private ArticleManager manager;
```

要注意的是，通过注解的方式用使用的话，我们必须在添加初始化mock的代码，不然即使标注了注解也会是null

```
MockitoAnnotations.initMocks(testClass);
```

关于Mockito更多详细的用法，大家可以直接参考官方文档，因为各种“奇技淫巧”确实比较多，后面也更新对java8 lambda的支持，很多功能还是期待大家去挖掘~


![](https://user-gold-cdn.xitu.io/2019/4/3/169e0c7fc5239a2e?w=1080&h=691&f=png&s=574015)

更多详细用法可直接参考官方文档：

https://static.javadoc.io/org.mockito/mockito-core/2.25.1/org/mockito/Mockito.html#0



相信当你熟练使用Mockito以后，你会爱上写单测的，也会让你代码健壮性有所加强。有些bug能提前发现的话，总比运行的时候被别人半夜叫起来修复舒服是吧?

<p align="center">
有收获的话，就分享给更多的朋友吧<br/>
<b>关注「深夜里的程序猿」，分享最干的干货</b>
</p>
<p align="center">
<img src="/resource/qrcode.png" alt="Sample"  width="200" height="200">
</p>