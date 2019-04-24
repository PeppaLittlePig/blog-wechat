### 前言

对于一名专业的程序员来说，Linux相关知识是必须要掌握的，其中对于文本的处理更是我们常见的操作，比如格式化输出我们需要的数据，这些数据可能会来源于文本文件或管道符，或者统计文本里面我们需要的数据出现的频次以及总数等等。那么这时候**awk**就很值得我们去学习了。

### 正文

在Linux中，awk、sed、grep被称为“三剑客”，都跟文本操作有关，那他们各自有什么特点呢？  

 grep：适合用于单纯的查找与匹配。  
 sed：适合修改匹配到的文本。  
 awk：适合对文本进行复杂的格式化处理。

所以awk是一种文本处理的编程工具语言，它会扫描输入数据的每一行，若与当前的pattern匹配，则执行对应的动作，若不匹配或者当前行的动作已执行完成的话则会继续下一行的处理，直到数据读取完成。

 
 
#### 基本用法

##### awk基本语法
```
awk [option] 'pattern{action}' files

//awk 关键字
//[option] 可以省略的一些参数
//'pattern{action}' pattern 是匹配的条件，可省略。action是具体执行的动作。
//files 是我们操作的文件，可多文件操作。
```

##### awk典型用法
```
awk '{
    BEGIN{action ...} //执行前语句
    {action...} //匹配处理每行数据
    END{action...} //执行后语句
}'
```

##### awk内置变量

| 变量| 作用 |
| ------ | ------ |
| FS | 输入**字段**分割符，默认空白字符 | 
| OFS |输出**字段**分割符，默认空白字符 |
| RS|输入记录也就是**行数据**分隔符，默认换行符|
| ORS|输出记录也就是**行数据**分隔符，默认换行符|
| NF |当前行被分割成多少个字段的数量 |
| NR |当前的行号，从1开始，在多文件中该值也会累加|
| FNR |当前的行号，从1开始，与NR不同，它是对应各自的文件累加 |
| FILENAME |当前的文件名|
| $0 |当前行数据 |
| \$1 ~ \$n |获取该行记录的第N个字段|

示例：
```
[root@wangzh awkdemo]# cat /etc/passwd

root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
...
```
```
//利用FS修改输入字段分割符，然后输出行号以及第1第7个字段的值
[root@wangzh awkdemo]# awk 'BEGIN{FS=":"} {print NR,$1,$7}'  /etc/passwd
1 root /bin/bash
2 bin /sbin/nologin
3 daemon /sbin/nologin
4 adm /sbin/nologin
...
```

```
//跟上一个例子的区别，添加了标题的输出，修改了输出字段的分隔符为"-"
[root@wangzh awkdemo]#awk 'BEGIN{FS=":";print "Result Title"} {print NR,$1}' OFS="-"  /etc/passwd
Result Title
1-root
2-bin
3-daemon
4-adm
...
```

##### 运算符与正则

这块内容的话，跟我们大多数编程语言都比较相似，大伙可以横向对比一下，对于刚接触的同学可以会理解一点。  

算术运算符:```==,>,<,!=,>=,<=,+,-```

逻辑运算符:```&&,||```

正则： 

* /regex/ 该行内容匹配上正则就执行动作
* ! /regex/  该行内容未匹配上正则就执行动作
* $1 ~ /regex/  只在第一个字段匹配正则
* $1 !~ /regex/ 第一个字段不匹配该正则  

案例：
```
//'-F:' 是定义输入字段分割字符的另一种方法，这个匹配第一个字段包含'root'的信息 
[root@wangzh awkdemo]# awk -F: '$1 ~ /root/ {print}' /etc/passwd
root:x:0:0:root:/root:/bin/bash
dockerroot:x:994:991:Docker User:/var/lib/docker:/sbin/nologin
```

```
//输出第一行到第三行的数据
[root@wangzh awkdemo]# ip addr | awk 'NR>=1 && NR<=3 {print}'
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
```

##### if/for/while

```
if(condi1){action1} else if(condi2){action2} else{action3}

---

for(i=1;i<=NR;i++){
    action1;
    action2;
    ...
}

---

while(condi){
    action1;
    ...
}
```


可以看到，用法几乎跟很多编程语言是一致的，下面给出一个简单的示例。

```
[root@wangzh awkdemo]# cat t1.log 
1 aa
3 bb
10 cc
9 dd
5 ee

---

//判断每行第一个字段是否是3-9之间的数字，然后输出对应的结果
[root@wangzh awkdemo]# awk '{if($1 ~ /[3-9]/){print "yes"} else {print "no"}}' t1.log 
no
yes
no
yes
yes
```

##### 内置函数

在awk中，内置函数也不少，帮助我们封装了一些字符操作、数学操作等，具体的用法还需各位查阅帮助手册，下面就先介绍一下比较常用的 sub() 函数的用法。

参考：http://www.cnblogs.com/chengmo/archive/2010/10/08/1845913.html

sub( Ere, Repl, [ string ] )

string参数是需要处理的字符串，默认是```$0```也就是当前行  
把```Ere```正则匹配的字符串用```Repl```的字符串来替换

```
[root@wangzh awkdemo]# awk 'BEGIN{info="this is a test2019test!";sub(/[0-9]+/,"!",info);print info}'   
this is a test!test!
```

#### 实战案例

统计文本中关键字出现的次数

```
[root@wangzh awkdemo]# cat data.txt 
ID NAME
1 xiaom
2 zsan
3 lisi
4 lisi
5 lisi
6 xiaom
7 lisi
8 xiaom
9 xiaoh
10 zsan

---

[root@wangzh awkdemo]# awk 'BEGIN{print "Statistics Result >>>>>"} {if(FNR>1){result[$2]+=1}} END{for(i in result){print i,"count:"result[i]} {print "over >>>>"}}' data.txt 
Statistics Result >>>>>
xiaoh count:1
xiaom count:3
zsan count:2
lisi count:4
over >>>>
```


### 结语

本篇文章的目的是让没接触这块内容的同学对文本处理有一个感性的认识，对于掌握awk绝对不是只看就可以学会的，必须要自己动手实践起来，遇到问题多查手册，相信很快你也是一个文本处理高手。

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
</p>



