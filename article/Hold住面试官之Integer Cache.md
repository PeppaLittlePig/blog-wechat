#### 前言



最近跟许多朋友聊了下，在这个“跳槽”的黄金季节，大家都有点蠢蠢欲动，所以最近就多聊聊面试的时候需要注意的一些问题，这些问题不一定多深奥，多复杂，但是一不注意的话却容易掉坑。下面看一下面试的时候经常遇到的一段代码:


```
public class IntegerDemo {
    public static void main(String[] args) {
        Integer numA = 127;
        Integer numB = 127;

        Integer numC = 128;
        Integer numD = 128;

        System.out.println("numA == numB : "+ (numA == numB));
        System.out.println("numC == numD : "+ (numC == numD));
    }
}
```


根据大家以往的经验，会认为上面的代码用“==“符号来比较，对比的是对象的引用，那么ABCD是不同的对象，所以输出当然是false了。我在[《“==”、“equals()”、“hashcode()”之间的秘密](https://mp.weixin.qq.com/s/FCdJN3-1u4u1KK2a2_DvGw)》这篇文章也讨论过。那么事实也是如此吗？下面看一下输出结果：


```
numA == numB : true
numC == numD : false
```

What？这个输出结果怎么跟以往的认知有所出入呢？在我们的代码“Integer numA = 127”中，编译器会把基本数据的“自动装箱”(autoboxing)成包装类,所以这行代码就等价于“Integer numA = Integer.valueOf(127)”了,这样我们就可以进入valueOf方法查看它的实现原理。



```
//Integer valueOf方法    
public static Integer valueOf(int i) {
if (i >= IntegerCache.low && i <= IntegerCache.high)
    return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}

//Integer静态内部类
private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer cache[];

    static {
    // high value may be configured by property
    int h = 127;
    String integerCacheHighPropValue =
        sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
    if (integerCacheHighPropValue != null) {
    try {
          int i = parseInt(integerCacheHighPropValue);
                        i = Math.max(i, 127);
    // Maximum array size is Integer.MAX_VALUE
                        h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                    } catch( NumberFormatException nfe) {
    // If the property cannot be parsed into an int, ignore it.
                    }
                }
                high = h;
                cache = new Integer[(high - low) + 1];
    int j = low;
    for(int k = 0; k < cache.length; k++)
                    cache[k] = new Integer(j++);
    
    // range [-128, 127] must be interned (JLS7 5.1.7)
                assert IntegerCache.high >= 127;
            }
    
    private IntegerCache() {}
        }
```


从上面的源码可以看到，valueOf方法会先判断传进来的参数是否在IntegerCache的low与high之间，如果是的话就返回cache数组里面的缓存值,不是的话就new Integer(i)返回。



那我们再往上看一下IntegerCache，它是Integer的内部静态类，low默认是-128，high的值默认127，但是high可以通过JVM启动参数XX:AutoBoxCacheMax=size来修改（如图），如果我们按照这样修改了，然后再次执行上面代码，这时候2次输出都是true，因为缓存的区间变成-128~200了。



![](https://user-gold-cdn.xitu.io/2019/4/4/169e5f13f1e2bb24?w=462&h=162&f=png&s=45553)




但是如果我们是通过构造器来生成一个Integer对象的话，下面的输出都是false。因为这样不会走ValueOf方法，所以按照正常的对象对比逻辑即可。


```
public class IntegerDemo {
    public static void main(String[] args) {
        Integer numA = new Integer(127);
        Integer numB = new Integer(127);

        Integer numC = new Integer(128);
        Integer numD = new Integer(128);

        System.out.println("numA == numB : "+ (numA == numB));//false
        System.out.println("numC == numD : "+ (numC == numD));//false
    }
}
```

还有需要注意的一点是，此类缓存行为不仅存在于Integer对象。还存在于其他的整数类型Byte，Short，Long，Character。但是能改变缓存范围的就只有Integer了。



#### 结语



有时候往往越简单的知识越容易掉坑里，所以要保持自己的求知欲，不断巩固的基础，才能让自己在面试的时候不会栽跟头。


<p align="center">
有收获的话，就分享给更多的朋友吧<br/>
<b>关注「深夜里的程序猿」，分享最干的干货</b>
</p>
<p align="center">
<img src="/resource/qrcode.png" alt="Sample"  width="200" height="200">
</p>

