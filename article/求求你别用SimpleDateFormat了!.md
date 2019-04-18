#### 前言



啊哈哈，标题写的比较随意了，其实呢最近在各种面试以及博客中，SimpleDateFormat出镜率确实是比较高了，为什么？其实聪明的你们肯定知道，那必须是有坑呗，是的，那我们就以案例来分析一下到底会有那些坑，或者还有没有其他更优的替代方案呢？



#### 正文



首先我们来看一下可能会出现在DateUtils中的写法：
```
private static final SimpleDateFormat dayFormat = new SimpleDateFormat("yyyy-MM-dd  HH:mm:ss");

public static Date formatDate(String date) throws ParseException {
    return dayFormat.parse(date);
}
```

当我们在单线程的程序中调用 formatDate(date) ,此时并不会出现任何问题(如果这也出问题那还玩什么...) ,然而当我们的程序在多线程并发执行调用这个方法的时候。




```
ExecuterService es = ExecuterService.newFixedThreadPool(50);
for( ... ){
    es.execute( () -> {
        System.out.println(parse("2018-11-11 10:35:20"));
    })
}
```

此时你会发现打印出来的时间有些是错误，程序甚至会抛出异常NumberFormatException？？为什么会出现这种情况呢？我们可以直接查看SimpleDateFormat.parse() 方法的源码一探究竟。


```
 private StringBuffer format(Date date, StringBuffer toAppendTo,
     FieldDelegate delegate) {
 // Convert input date to time field list

 calendar.setTime(date);

 boolean useDateFormatSymbols = useDateFormatSymbols();

 for (int i = 0; i < compiledPattern.length; ) {

 int tag = compiledPattern\[i\] >>> 8;

 int count = compiledPattern\[i++\] & 0xff;

 if (count == 255) {

 count = compiledPattern\[i++\] << 16;

 count |= compiledPattern\[i++\];

 }
```

从源码可以看到，在多线程的环境中，执行到第五行 calendar进行操作的时候，后面的线程有可能会覆盖上一个线程设置好的值，此时就导致前面线程执行的结果被覆盖而返回了一个错误的值。



那我们该如何避免这个坑呢？



1、使用ThreadLocal，每个线程中返回各自的实例，避免了多线程环境中共用同一个实例而导致的问题。

```
 private static ThreadLocal<SimpleDateFormat> simpleDateFormatThreadLocal = new ThreadLocal<>();

 public static Date formatDate(String date) throws ParseException {
     SimpleDateFormat dayFormat = getSimpleDateFormat();
     return dayFormat.parse(date);
}
private static SimpleDateFormat getSimpleDateFormat() {
    SimpleDateFormat simpleDateFormat = simpleDateFormatThreadLocal.get();
    if (simpleDateFormat == null){
        simpleDateFormat = new SimpleDateFormat("yyyy-mm-dd  HH:mm:ss");
        simpleDateFormatThreadLocal.set(simpleDateFormat);
    }
    return simpleDateFormat;
 }
 ```
 
需要注意一点的是，ThreadLocal的使用过程中也是有小坑需要注意的，大家可以参考一下其他的资料，以后可以抽空聊聊这个话题。



2、推荐升级到JDK8+，使用LocalDateTime，LocalDate，LocalTime来代替，具体的用法请自行参考API，当然JDK8所带来的Lambda，Stream等特性也是值得一试的。



3、使用三方包，推荐Joda-Time，对于日期的增减操作也是相当便捷。



github：https://github.com/JodaOrg/joda-time


<p align="center">
有收获的话，就分享给更多的朋友吧<br/>
<b>关注「深夜里的程序猿」，分享最干的干货</b>
</p>
<p align="center">
<img src="/resource/qrcode.png" alt="Sample"  width="200" height="200">
</p>