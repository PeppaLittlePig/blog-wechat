### 前言

随着项目的迭代，代码中存在的分支判断可能会越来越多，当里面涉及到的逻辑比较复杂或者分支数量实在是多的难以维护的时候，我们就要考虑下，有办法能让这些代码变得更优雅吗？

### 正文


#### 使用枚举

这里我们简单的定义一个表示状态的枚举。

```java
public enum Status {
    NEW(0),RUNNABLE(1),RUNNING(2),BLOCKED(3),DEAD(4);

    public int statusCode;

    Status(int statusCode){
        this.statusCode = statusCode;
    }
}
```

那么我们在使用的时候就可以直接通过枚举调用了。

```java
int statusCode = Status.valueOf("NEW").statusCode;
```

优雅的解决了下面代码赋值的方式

```java
if(param.equals("NEW")){
    statusCode = 0;
}else if(param.equals("RUNNABLE")){
    statusCode = 1;
}
...
```

#### 善用 Optional

在项目中，总少不了一些非空的判断,可能大部分人还是如下的用法

```java
if(null == user){
    //action1
}else{
    //action2
}
```

这时候该掏出Optional这个秘密武器了，它可以让非空校验更加优雅，间接的减少if操作。没了解过Optional的同学可自行Google，这里就不再赘述。

```java
Optional<User> userOptional = Optional.ofNullable(user);
userOptional.map(action1).orElse(action2);
```
上面的代码跟第一段是等效的，通过一些新特性让代码更加紧凑。

#### 表驱动法

来自Google的解释：表驱动法是一种编程模式，它的本质是，从表里查询信息来代替逻辑语句(if,case)。下面看一个案例，通过月份来获取当月的天数（仅作为案例演示，获取2月份的数据不严谨），普通做法：

```java
int getMonthDays(int month){
	switch(month){
		case 1:return 31;break;
		case 2:return 29;break;
		case 3:return 31;break;
		case 4:return 30;break;
		case 5:return 31;break;
		case 6:return 30;break;
		case 7:return 31;break;
		case 8:return 31;break;
		case 9:return 30;break;
		case 10:return 31;break;
		case 11:return 30;break;
		case 12:return 31;break;
		default：return 0;
	}
}
```

表驱动法实现方式

```java
int monthDays[12] = {31, 29, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31};
int getMonthDays(int month){
	return monthDays[--month];
}
```
其实这里的表就是数组而已，通过直接查询数组来获得需要的数据，那么同理，Map之类的容器也可以成为我们编程概念中的表。

```java
Map<?, Function<?> action> actionsMap = new HashMap<>();

// 初试配置对应动作
actionsMap.put(value1, (someParams) -> { doAction1(someParams)});
actionsMap.put(value2, (someParams) -> { doAction2(someParams)});
actionsMap.put(value3, (someParams) -> { doAction3(someParams)});
 
// 省略 null 判断
actionsMap.get(param).apply(someParams);
```

通过Java8的lambda表达式，我们把需要执行东西存进value中，调用的时候通过匹配key的方式进行。

#### 提前判断返回

在之前的文章《优化代码中的“坏味道”》里也有提过，如下语句

```java

if(condition){
   //dost
}else{
   return ;
}
```

改为

```java
if(!condition){
   return ;
}
//dost
```
避免一些不必要的分支，让代码更精炼。

#### 其他方法 

除了上面提到的方法，我们还可以通过一些设计模式，例如策略模式，责任链模式等来优化存在大量if,case的情况，其原理会和表驱动的模式比较相似，大家可以自己动手实现一下，例如我们在Netty的使用过程中，可能会出现需要大量判断不同的命令去执行对应动作的场景。

```java  
ServerHandler.java

if(command.equals("login")){
    //执行登录
}else if(command.equals("chat")){
    //聊天
}else if(command.equals("broadcast")){
    //广播信息
}
....
```
该如何处理呢？这里先卖个关子，大家可以先思考一下，笔记后续会写一些关于Netty实现IM的文章，到时候会详细介绍。




### 结语

最后要明确一点，不是所有的if/else,switch/case都需要优化，当我们发现有“痛点”或者“闻到代码有坏味道”再来优化才是最好的，不然你可能会写了一个从不扩展的可扩展代码，所有的优化都是为了更好的迭代项目，更好的服务于业务，而不是为了优化而优化。

---


#### 推荐阅读

《[如何提高使用Java反射的效率？](https://mp.weixin.qq.com/s/-HXqicBROZU8XDF5YSCHMw)》  
《[Java日志正确使用姿势](https://mp.weixin.qq.com/s/aQx2ROajH2SqgHL77yxW3Q)》   
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
                                                                                                                                                                                                                                                                  
                                                                                                                                                                                                                                                                  《[如何提高使用Java反射的效率？](https://juejin.im/post/5cc16dd25188252d3f7e1712)》  
                                                                                                                                                                                                                                                                  《[Java日志正确使用姿势](https://juejin.im/post/5cbc1309f265da038a147452)》  
                                                                                                                                                                                                                                                                  《[大白话搞懂什么是同步/异步/阻塞/非阻塞](https://juejin.im/post/5cb58010e51d456e4d4de729)》   
                                                                                                                                                                                                                                                                  《[论JVM爆炸的几种姿势及自救方法](https://juejin.im/post/5ca83c5a6fb9a05e5343b6da)》   
                                                                                                                                                                                                                                                                  
                                                                                                                                                                                                                                                                  <center> 有收获的话，就点个赞吧
                                                                                                                                                                                                                                                                  
                                                                                                                                                                                                                                                                  
                                                                                                                                                                                                                                                                  **关注「深夜里的程序猿」，分享最干的干货**</center>
                                                                                                                                                                                                                                                                  
                                                                                                                                                                                                                                                                  ![](https://user-gold-cdn.xitu.io/2019/4/17/16a28c18805b08e1?w=200&h=200&f=png&s=7078)