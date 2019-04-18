#### 前言

对于大部分程序员来说，主要工作都是进行编码以及一些简单的中间件安装，这就导致了很多人对于“运维”相关的工作会比较生疏。例如当我们拥有一台自己的服务器以后，可能会在上面跑一跑自己blog程序，mysql，nginx等等。当程序越来越多了没有一个统一的入口管理启停，也可能会遇到一些特殊的原因导致程序被kill掉了，这时候又没装相关的监控程序或者脚本（太麻烦了懒得装，机器配置差不想装），所以只能当我们访问自己程序发现异常的时候才会登上服务器查找原因。

这些状况对我们来说是比较麻烦的，那么这就需要一个“神器”来解放我们的双手，铛铛铛！！**Supervisor** 就来了。


#### 正文

##### Supervisor 介绍

Supervisor是用Python开发的一套通用的进程管理程序，能将一个普通的命令行进程变为后台daemon，并监控进程状态，异常退出时能自动重启。它是通过fork/exec的方式把这些被管理的进程当作supervisor的子进程来启动，这样只要在supervisor的配置文件中，把要管理的进程的可执行文件的路径写进去即可。也实现当子进程挂掉的时候，父进程可以准确获取子进程挂掉的信息的，可以选择是否自己启动和报警。


##### supervisor 安装
简单粗暴

yum install supervisor -y

##### supervisor 配置说明

通过这种形式安装的supervisor，其配置文件的目录位于：  
/etc/supervisord.conf (主配置文件，下面会详细介绍)  
/etc/supervisor.d/  (默认子进程配置文件，也就是需要我们根据程序配置的地方)

---

supervisord.conf 基本配置项说明，由于其参数比较多，这些只贴出一些常用的配置项，详细内容可参阅官网。温馨提示  **“;”** 符号是表示该行配置被注释。

```
[unix_http_server]
file=/home/supervisor/supervisor.sock   ; supervisorctl使用的 socket文件的路径
;chmod=0700                 ; 默认的socket文件权限0700
;chown=nobody:nogroup       ; socket文件的拥有者

[inet_http_server]         ; 提供web管理后台管理相关配置
port=0.0.0.0:9001          ; web管理后台运行的ip地址及端口，绑定外网需考虑安全性 
;username=root             ; web管理后台登录用户名密码
;password=root

[supervisord]
logfile=/var/log/supervisord.log ; 日志文件，默认在$CWD/supervisord.log
logfile_maxbytes=50MB        ; 日志限制大小，超过会生成新文件，0表示不限制
logfile_backups=10           ; 日志备份数量默认10，0表示不备份
loglevel=info                ; 日志级别
pidfile=/home/supervisor/supervisord.pid ; supervisord pidfile; default supervisord.pid              ; pid文件
nodaemon=false               ; 是否在前台启动，默认后台启动false
minfds=1024                  ; 可以打开文件描述符最小值
minprocs=200                 ; 可以打开的进程最小值

[supervisorctl]
serverurl=unix:///home/supervisor/supervisor.sock ; 通过socket连接supervisord,路径与unix_http_server->file配置的一致

[include]
files = supervisor.d/*.conf ;指定了在当前目录supervisor.d文件夹下配置多个配置文件

```

##### 准备测试项目

这里我打包了一个简单的spring-boot程序，存放与“/opt/project/”下。

![](https://user-gold-cdn.xitu.io/2019/4/11/16a0b4473b2c1d04?w=573&h=40&f=png&s=5166)

在这个程序中我们只是简单的定义了一个rest接口，主要用于演示作用。

```java
@RestController
public class HelloController {

    @RequestMapping("/hello")
    public String hello(){
        return "hello world";
    }
}
```

如果不用supervisor的话，我们启动程序一般用一下命令  
> nohup java -jar springboot-hello-sample.jar & 

这是以后台的方式启动jar包，程序运行相关输出会在nohup.out中，我们我们就不再赘述了，那么我们来看一下，切换到supervisor的方式，我们是怎么配置项目，以及管理的呢？

##### 定义supervisor管理进程配置文件

从上面的配置文件[include]->files配置项我们可以知道，supervisor会把supervisor.d/下以conf结尾的配置文件都加载进来，那么我们在这个目录下面新建一个sboot.conf,内容如下：  
```

[program:sboot] ;[program:xxx] 这里的xxx是指的项目名字
directory = /opt/project  ;程序所在目录
command =  java -jar springboot-hello-sample.jar ;程序启动命令
autostart=true ;是否跟随supervisord的启动而启动
autorestart=true; 程序退出后自动重启,可选值：[unexpected,true,false]，默认为unexpected，表示进程意外杀死后才重启
stopasgroup=true;进程被杀死时，是否向这个进程组发送stop信号，包括子进程
killasgroup=true;向进程组发送kill信号，包括子进程
stdout_logfile=/var/log/sboot/supervisor.log;该程序日志输出文件，目录需要手动创建
stdout_logfile_maxbytes = 50MB;日志大小
stdout_logfile_backups  = 100;备份数

```

可以看到，在配置文件里面已经有配置该程序名，启动路径等，这样一来，supervisor就可以完全的掌管程序的生死了。接着我们执行

service supervisord restart

重启supervisord。

##### 观察效果

浏览器输入:服务器ip:9001 （这里的web管理页面端口是在配置文件配置好的。）


![](https://user-gold-cdn.xitu.io/2019/4/11/16a0bc67e116a660?w=971&h=287&f=png&s=25254)

如图所示，我们可以可以观察到springboot程序已经是running状态了，pid是27517，我们可以点击Tail -f拉力观察输出日志，它的作用跟我们在服务器直接“tail -f”是类似的。


![](https://user-gold-cdn.xitu.io/2019/4/11/16a0bd046846e991?w=1293&h=138&f=png&s=19805)

这里我们为了方便延时就没有添加验证了，如果大家是在公网的使用环境，需要配置文件里面的用户名密码验证注释打开。不然别人扫出你的后台管理页面就可以随意控制程序了。

除了web管理页面，还有一些简单的命令也是需要我们掌握的。

直接在命令行输入supervisorctl会展示当前已配置好的项目信息。
```
[root@wangzh supervisor.d]# supervisorctl 
sboot                            RUNNING   pid 27517, uptime 0:18:04
supervisor> 

```

然后可以执行

```
start/stop/restart sboot 来简单控制项目的启停等
```


```
supervisorctl update #更新配置文件
supervisorctl reload #重新启动配置的程序
supervisorctl stop all #停止全部管理进程
```




#### 结语

只需要一点简单的配置，就可以统一的管理我们的程序了，同时也可以在进程意外死掉的时候自动重启，这些工作以后就交给supervisor了，我们只要掌握一点简单的命令就可以“为所欲为”。

supervisor官网：http://www.supervisord.org/


<p align="center">
有收获的话，就分享给更多的朋友吧<br/>
<b>关注「深夜里的程序猿」，分享最干的干货</b>
</p>
<p align="center">
<img src="/resource/qrcode.png" alt="Sample"  width="200" height="200">
</p>