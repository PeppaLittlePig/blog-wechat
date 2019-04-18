#### 前言



万丈高楼平地起，今天的聊点基础而又经常让人忽视的话题，比如“==”与“equals()”区别？为何当我们重写完"equals()"后也要有必要去重写"hashcode()"呢？ ... 带着这些问题，我们一起来探究一下。



#### 概念



"==":它主要是判断符号两边的“对象”的值是否相等，而这里的“值“”又有所区分了。

基础数据类型：比较的就是自身的值，这个跟我们常规的理解是基本一致的。

引用数据类型：比较的对象的内存地址。



“equals()”:它也是用来判断两个对象是否相等，所以也得分不同的情况来说明。

在当前类中，没有重写equals方法的话，默认的实现跟"=="的实现是一样的。下面是Object类的equals方法实现。





在当前类中，重写了equals方法，此时判断的依据就是你重写的逻辑。





####  怎样重写equals()方法？



* 1、自反性：对于任何非空引用x，x.equals(x)应该返回true。
* 2、对称性：对于任何引用x和y，如果x.equals(y)返回true，那么y.equals(x)也应该返回true。
* 3、传递性：对于任何引用x、y和z，如果x.equals(y)返回true，y.equals(z)返回true，那么x.equals(z)也应该返回true。
* 4、一致性：如果x和y引用的对象没有发生变化，那么反复调用x.equals(y)应该返回同样的结果。
* 5、非空性：对于任意非空引用x，x.equals(null)应该返回false。



由此可以看出，重写一个equals()方法，需要注意的点还是比较多的，这里给出一个参考的事例。


```
public class EqualsDemo {
    private String name;
    private String info;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;

        EqualsDemo that = (EqualsDemo) o;

        if (name != null ? !name.equals(that.name) : that.name != null) return false;
        return info != null ? info.equals(that.info) : that.info == null;
    }

    @Override
    public int hashCode() {
        int result = name != null ? name.hashCode() : 0;
        result = 31 * result + (info != null ? info.hashCode() : 0);
        return result;
    }
}
```

有些读者可能会感到奇怪，不是说重写equals()方法吗，为什么这里又出现了一个hashcode()?所以这里又引出了我们的另一个主角hashcode()方法，当我们重写了equals()方法后，它就一定会出现，也会“吵着“自己也要被重写。



####  什么是hashcode()？



hashCode() 的作用是获取哈希码，也称为散列码；它返回的一个int整数。这个哈希码的作用是确定该对象在哈希表中的索引位置。hashCode方法的主要作用是为了配合基于散列的集合一起正常运行，这样的散列集合包括HashSet、HashMap、HashTable等。它定义在JDK的Object.java中，这就意味着Java中的任何类都包含有hashCode() 函数。



当我们在上面的集合插入对象的时候，java是怎么知道里面是否有重复的对象呢？可能大家第一反应是equals方法，没错这方法可以实现这个功能，但是当集合里面有成千上万个元素的时候，效率会如何呢？答案当然是比较差了，所以才会出现了哈希码。


```
public V put(K key, V value) {
    //判断当前数组是否等于{}，若是则初始化数组
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        //判断 key 是否等于 null，是则将把当前键值对添加进table[0]中，遍历table[0]链表
        //如果已经有null为key的Entry，则修改值，返回旧值，若无则直接添加。
        if (key == null)
            return putForNullKey(value);
        //key不为null则计算hash
        int hash = hash(key);
        //搜索对应hash所在的table中的索引
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        //修改次数
        modCount++;
        addEntry(hash, key, value, i);
        return null;
}
```

这里是jdk7中 Hashmap put()方法的实现，通过源码的注释可以看出执行的流程，需要更详细的了解HashMap可以参考我之前发在开源中国的博客《Java7  HashMap全面解读! 》，链接：https://my.oschina.net/19921228/blog/752073



经过概念的介绍，知道为什么重写完equals()后要接着重写hashcode()了吧？


```
People p1=new People("小明",18);
People p2=new People("小明",18);
```


此时重写了equals方法，p1.equals(p2)一定返回true，假如只重写equals而不重写hashcode，那么Student类的hashcode方法就是Object默认的hashcode方法，由于默认的hashcode方法是根据对象的内存地址经哈希算法得来的，显然此时s1!=s2,故两者的hashcode不一定相等。所以在一些集合的使用当中会出现问题。



#### 总结 



小小的几个方法，没想到却有这么多“坑”，而且在面试中也会经常被问到，在金三银四的时候，但愿各位不会陷在这里。


<p align="center">
有收获的话，就分享给更多的朋友吧<br/>
<b>关注「深夜里的程序猿」，分享最干的干货</b>
</p>
<p align="center">
<img src="/resource/qrcode.png" alt="Sample"  width="200" height="200">
</p>