#### 前言

如今不管是在面试还是在我们的工作中，OOM总是不断的出现在我们的视野中，所以我们有必要去了解一下导致OOM的原因以及一些基本的调整方法，大家可以通过下面的事例来了解一下什么样的代码会导致OOM，帮助我们以后在工作中能够通过异常信息来判断是JVM里面哪个区域出现了问题。

先介绍一下笔者的相关编码环境。

jdk：java version "1.8.0_121"

ide：IntelliJ IDEA 2019.1 (Community Edition)

---

#### 正文 

##### 1.Java堆溢出

Java中的堆存储的都是对象实例，当我们不断的创建对象，而GC的时候又不能回收，当存储的对象大小超过了-Xmx的值，这时候则会出现OutOfMemoryError.[-XX:+HeapDumpOnOutOfMemoryError]参数可以让jvm出现内存溢出的时候dump出内存堆转储快照。

```
/**
 * VM Args: -Xms10m -Xmx10m -XX:+HeapDumpOnOutOfMemoryError
 * @author wangzenghuang
 */
public class HeapOOMDemo {
    public static void main(String[] args) {
        List<String> stringList = new ArrayList<>();
        while(true){
            stringList.add("str");
        }
    }
}

```

运行结果，发生OOM，并且在我们项目的根目录dump出当前的内存堆快照

```
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid1376.hprof ...
Heap dump file created [7972183 bytes in 0.047 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3210)
	at java.util.Arrays.copyOf(Arrays.java:3181)
	at java.util.ArrayList.grow(ArrayList.java:261)
	at java.util.ArrayList.ensureExplicitCapacity(ArrayList.java:235)
	at java.util.ArrayList.ensureCapacityInternal(ArrayList.java:227)
	at java.util.ArrayList.add(ArrayList.java:458)
	at HeapOOMDemo.main(HeapOOMDemo.java:12)

Process finished with exit code 1
```

简单解决思路

那么发生这个问题以后我们的解决思路有哪些呢？我们可以利用一些工具（例如Eclipse Memory Analyzer
）来分析dump出的文件，一般来说，当生产环境发生OOM，比较常见的一个原因是发生了**内存泄漏**，用工具可以分析出泄露的对象到GC Root的引用链，从而定位到问题代码。假如经过分析后发现内存中的对象都是“必须存活”的对象，这时候就要思考下项目中是否把“-Xms跟-Xmx”设置得太小了（当然这里也不是随意调大，需要结合机器的物理内存情况），再者需要留意代码中是否有一些长生命周期的对象，从代码中优化内存消耗。

##### 2.方法区溢出

在jvm的方法区中，它主要存放了类的信息，常量，静态变量等。在jdk8以前是通过“-XX:PermSize，-XX:MaxPermSize”来调整这个区域的值，但是从8开始呢，永久代的概念被MetaSpace（元空间）代替了，对应的参数也变成了“-XX:MetaspaceSize，-XX:MaxMetaspaceSize”。在这个例子中使用CGLib来动态生成一些类，方便我们实验操作。

```
/**
 * VM Args: -XX:MetaspaceSize=5m -XX:MaxMetaspaceSize=5m
 * @author wangzenghuang
 */
public class MethodAreaOOMDemo {
    public static void main(String[] args) {
        while(true){
            Enhancer enhancer = new Enhancer();
            enhancer.setSuperclass(OOMObject.class);
            enhancer.setUseCache(false);
            enhancer.setCallback(new MethodInterceptor() {
                public Object intercept(Object obj, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
                    return methodProxy.invokeSuper(obj,objects);
                }
            });
            enhancer.create();
        }
    }
    static class OOMObject{}
}
```

运行结果

```
Exception in thread "main" java.lang.OutOfMemoryError: Metaspace
```

简单解决方法

这个问题的话，一般来说根据情况调整方法区的大小就行了，网上也有人说可以去掉MetaSpace的的大小限制，但是不建议这么干，毕竟不可控的事情我们要少点干，很容易给自己埋雷。


##### 3.栈溢出

对于我们来说，还有一个熟悉的错误，那就是“StackOverflowError”，它是由线程请求的栈深度超过了jvm允许的最大范围而产生的。“-Xss”参数可以设置栈容量。


```
/**
 * VM Args: -Xss128k
 * @author wangzenghuang
 */
public class StackOFDemo {
    private static int stackLength = 1;

    public void stackLeak(){
        stackLength++;
        stackLeak();
    }

    public static void main(String[] args) {
        StackOFDemo stackOFDemo = new StackOFDemo();
        try {
            stackOFDemo.stackLeak();
        }catch (Throwable e){
            System.out.println("length : "+ stackLength);
            throw e;
        }
    }
}
```

运行结果

```
length : 983
Exception in thread "main" java.lang.StackOverflowError
	at StackOFDemo.stackLeak(StackOFDemo.java:10)
	at StackOFDemo.stackLeak(StackOFDemo.java:10)
    ...
```

简单解决思路

一般来说此类问题多出现在存在递归的地方，要从代码里重新审视递归未结束的原因，若递归的方法没问题可以根据实际情况调整“-Xss”参数的大小。还有一些代码的循坏依赖也会造成此类情况，

##### 4.直接内存溢出

本机直接内存默认与“-Xmx”设定的值一样大，可以通过“-XX:MaxDirectMemorySize”修改。

```
/**
 * VM Args: -Xmx20m -XX:MaxDirectMemorySize=10
 * @author wangzenghuang
 */
public class DirectMemoryOOMDemo {
    private static final int _1MB = 1024 * 1024;

    public static void main(String[] args) throws IllegalAccessException {
        Field field = Unsafe.class.getDeclaredFields()[0];
        field.setAccessible(true);
        Unsafe unsafe = (Unsafe) field.get(null);
        while (true){
            unsafe.allocateMemory(_1MB);
        }
    }
}

```

运行结果

呃，一运行这段代码idea直接闪退了，查阅其他资料可以得知当DirectMemory导致内存溢出时，Heap Dump文件是很小的，如果程序中有使用NIO的情况可以检查一下。

##### 总结 


这里所展示的代码只是可以触发jvm的各种错误，但是并不代表这是唯一的触发错误的方方式，假如我们的代码比较复杂，有时候遇到类似错误的时候还是需要耐心分析。

<p align="center">
有收获的话，就分享给更多的朋友吧<br/>
<b>关注「深夜里的程序猿」，分享最干的干货</b>
</p>
<p align="center">
<img src="/resource/qrcode.png" alt="Sample"  width="200" height="200">
</p>

