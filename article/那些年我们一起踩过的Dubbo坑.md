**前言**

 

微服务架构在如今的9102年已经不是什么新鲜的话题了，但是怎么做好微服务架构，却又是一个永恒的话题。比如服务粒度的划分，怎么控制好粗细？服务划分后，对于项目的部署会有什么改变？...  这会是一个很大的话题，以后可以分开篇章探讨一翻，但是我们本篇并不打算聊这个，而是讨论一下具体的实现技术--dubbo。

 

**dubbo历史**

 

2011 年末，阿里巴巴在 GitHub 上开源了基于 Java 的分布式服务治理框架 Dubbo，之后它成为了国内该类开源项目的佼佼者，许多开发者对其表示青睐。同时，先后有不少公司在实践中基于 Dubbo 进行分布式系统架构，目前在 GitHub 上，它的 fork、star 数均已破万。2014 年 10 月 30 号发布版本 dubbo-2.4.11，修复了一个小 Bug，版本又陷入漫长的停滞到2017年九月份。

在dubbo停滞的期间呢，当当网 Fork 了阿里的一个 Dubbo 版本开始维护，并命名为 dubbox-2.8.0。值得注意的是，当当网扩展 Dubbo 服务框架支持 REST 风格远程调用，并且跟随着 ZooKeepe 和 Spring 升级了对应的版本。之后 Dubbox 一直在小版本维护，2015 年 3 月 31 号发布了最后一个版本 dubbox-2.8.4。笔者公司用的也是这个版本，并稍微改造了下源码，下面会有提及。

其实在当前说到微服务，可能大家第一反应是springcloud，spring全家桶带来的便捷是显而易见的，然而为什么我们这里聊的是dubbo呢？原因之一是因为笔者公司只用了dubbo（别扔鸡蛋....）,其二呢其实rpc框架很多原理是相通的，当我们理解了其中一个，再去看其他的框架，会有一种似曾相识的感觉，最后也没必要去争论XX框架的好与坏，选择最适合自己业务的就是最好的。

先交代下背景，我们这边是从2016年开始使用dubbo，使用的是dubbox-2.8.4 版本，然后因为一些场景不合适改了下代码，重新打包成2.8.5提交至公司的私服使用。好了，接下来就开始进入正文，聊聊这几年在dubbo使用过程中遇到坑，以及需要注意的地方吧。

 

**正文**

 

**1、超时重试**

 

这是一个很经典的坑，当时由于刚使用dubbo，很多配置都是基于默认的。刚好此时在项目中，有一个机器人送礼的逻辑比较复杂，当遇到某些特定的条件时，该逻辑的耗时会比正常情况下变长，这时候就出现了一个很神奇的现象，为何我只触发了一次送礼的请求，而线上却送了三次？

 

刚遇到这种情况可我惊呆了，重新审视了代码，发现并无问题。这就奇怪了，哪里来的3次？后来掉了几根头发以后，才在dubbo的文档中发现了服务这块有timeout跟retry属性，默认timeout=1000ms,retry=2。这下就豁然开朗，原来是第一次调用超时，导致又重试了2次，一共就是3次了。

 

找到问题的原因，我们就有办法解决了。由于我们这个接口不是幂等性的，而且也不用返回什么信息给调用者，所以我们可以通过一个线程池来执行这段耗时的逻辑，让rpc调用可以比较快的返回给调用者。这样就不存在超时的问题了。或者可以配合增加timeout时间跟retry=0也能实现，具体的业务逻辑需要自己找到合适的解决方案。

 

**2、dubbo使用内网ip**

 

正常情况下，我们的服务调用推荐走内网连接的方式，效率是比较高的。但是有些特殊的情况，我们需要dubbo注册服务的时候使用外网ip，该怎么修改呢？这时候就需要修改我们的服务器上 /etc/hosts 文件了，新增一条 “外网ip  主机名”的记录，restart我们的服务即可。

 

**3、docker里面注册宿主机内网ip**

 

说到微服务，当然也少不了docker了，我们当前用的是docker+overlay网络一个结构，直接把dubbo服务丢进容器里面跑的话，注册进zk的ip是容器ip。所以我们采取了一种折中的方式。

 

利用docker的特性，我们在创建容器的时候，把宿主机的ip以及需要暴露的端口写进容器的环境变量里面。然后就是修改dubbox的源码了，源码的com.alibaba.dubbo.registry.integration.RegistryProtocol类的getRegistedProviderUrl

方法，此方法用于返回注册到注册中心的URL。
```java
private URL getRegistedProviderUrl(final Invoker<?> originInvoker){
        //targetUrl 注册中心看到的地址
        URL targetUrl;
        URL providerUrl = getProviderUrl(originInvoker);
        //配置的容器环境变量
        String envParameterHost=System.getenv(ENV_HOST_KEY);
        String envParameterPort=System.getenv(ENV_PORT_KEY);
        if (StringUtils.isBlank(envParameterHost)||StringUtils.isBlank(envParameterPort)){//非容器环境：执行原来的注册逻辑
            targetUrl=providerUrl.removeParameters(getFilteredKeys(providerUrl)).removeParameter(Constants.MONITOR_KEY);
        }else {//容器环境，如果环境变量中DOCKER_NAT_HOST和DOCKER_NAT_PORT两个值都不为空则直接将这两个值作为url注册到zk
            //执行重新拼接url的操作，涉及敏感代码这里不展示了
            targetUrl=dockerRegUrlWithHostAndPort;
        }
        return targetUrl;
    }
```
**4、未注意服务重名**

 

其实这是我们开发人员粗心大意出现的情况，开发的时候注册了2个相同签名的服务，但是业务逻辑是完全不同的，这会导致一个之前运行的正常的业务会偶尔调用失败，原因是因为dubbo的负载均衡策略，把一部分流量转移到我们新注册上来的服务上了，但是处理逻辑不同，导致错误。

 

**5、版本的一致性**

 

dubbo当前的releases版本已经去到2.7.1了，项目中要注意一下不同项目间版本的一致性，或者是dubbo跟dubbox的一些差别，最好做到统一，不然出现问题解决的成本会比较高。

 

**6、属性配置的优先级**

 

我们在dubbo的过程中会发现，提供者跟消费者中，很多属性是一样的，我们该怎么配呢？在dubbo的文档当中其实有推荐的用法。

在提供者端尽量多提供消费者端的属性。

参考文档，原因如下：

作服务的提供方，比服务消费方更清楚服务的性能参数，如调用的超时时间、合理的重试次数等

在 Provider 端配置后，Consumer 端不配置则会使用 Provider 端的配置，即 Provider 端的配置可以作为 Consumer 的缺省值 。否则，Consumer 会使用 Consumer 端的全局设置，这对于 Provider 是不可控的，并且往往是不合理的

Provider 端尽量多配置 Consumer 端的属性，让 Provider 的实现者一开始就思考 Provider 端的服务特点和服务质量等问题。

**结语**

其实在dubbo的使用过程中，还有挺多问题这里没列出来的，但是解决方法都差不多，首先文档要熟，做到心中有数，比如dubbo功能的成熟度，有些是不推荐在线上使用的，这时你就要谨慎了。然后文档里面确实是有遗漏的问题，我们有必要可以debug dubbo的源码，这个过程会比较痛苦，但是对于排查问题跟个人能力的提高是有很有帮助的。

大家在dubbo的使用过程中有什么问题也可以交流一下~

github：https://github.com/apache/incubator-dubbo

中文使用手册:http://dubbo.apache.org/zhcn/docs/user/preface/background.html

<p align="center">
有收获的话，就分享给更多的朋友吧<br/>
<b>关注「深夜里的程序猿」，分享最干的干货</b>
</p>
<p align="center">
<img src="/resource/qrcode.png" alt="Sample"  width="200" height="200">
</p>
