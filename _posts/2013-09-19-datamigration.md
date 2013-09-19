---
layout: post
title: 记消息中心改版
category: program
---

最近消息中心改版，从之前java + Netty改成现在的Golang实现，代码实现方面的东西不是这篇文章的重点，这边博客谈谈新旧改版需要做的一些缓存设计。

**说说旧版的设计**

1. 数据库  
新旧版本都用mysql，旧版有13个表，其中有10个表存放客户端注册信息，即分表处理，规则很简单，由于是java实现，直接用客户端的ID求哈希，然后对10取余，其他两个表分别存放消息记录和后台服务器信息。  

2. NOSQL  
缓存用到数据库，主要是考虑到redis有多种数据结构，并且缓存的内容不多，不需要分布式。  

3. 简单介绍工作流程  
客户端（以下简称Client）先要注册某个服务器，之后携带注册服务器ID拉取消息，消息中心（以下简称Server）经过一系列的验证规则验证请求合法，然后根据上传的服务器ID，获取该ID对应的消息ID（msgId），获取消息之后并不是立即返回，起码消息内容还没有获取到，但是在这之前，还有一件重要事情要做，由于消息拉取规则有多种，比如：包含、排除、所有。  
所以，Server从redis获取到一个msgId的List之后，需要判断是否需要下发本条消息，当然，还需要判断该Client ID对应的历史消息List中是否存在该msgId。等这一切都完成之后，通过msgId获取消息内容。  

4. 旧版本的问题  
第一次独立上一个项目，并且在学校对这种高并发量完全没有接触，在设计的时候很多方面都没有考虑到，导致上线之后问题很多。  
* mysql没有设置索引，只设置了主键，并且实现的时候想得太多，把客户端的所有注册记录都保存了下来，导致客户端在获取注册记录的时候要获取最新的注册标记，然后在某个阳光明媚的周末早上，服务挂了，害得运维的同事临时给我加了索引。  
* 没有考虑mysql的存储引擎，直接就用了InnoDB，其实整个服务对事物安全没有要求，都是一句sql就搞定了，实在不行，大不了insert ignore。新版的改用默认MyISAM。  
* 注册信息表数量少了，新版直接改为1000个。
* 从上面的工作流程可以看到，整个处理略显复杂，简单的处理逻辑，居然需要多大5到6次请求redis操作，这对于高并发的服务器来说，有点致命，前期这么设计的原因，除了经验不足，还考虑到如果将Client ID作为redis缓存key，可以避免Client ID重复，而ID的量又是比较大的。这样设计用了redis的hash，Client ID为key，各个后台服务器ID为field，对应的msgId列表用#连接成一个字符串，是value。现在看来，确实设计得有点奇葩。  

旧版本大概就差不多了，其实我觉得如果当时有建索引，没有宕机的话，java + Netty的组合还是能用一阵子的，虽然整个结构看起来有点怪异。

**新版怎么做**

针对以上旧版的问题，新版对症下药（我能力有限，显然不是主刀的，由部门大牛帮忙重新设计整个结构）。

1. 用高并发的Golang
Golang最近比较火，从语言级别上支持高并发，并且部门已经有了项目经验，代码量确实比java要少，比较适合消息中心这种轻量级的服务端应用。但是也存在为数不多的几个坑，我就被坑得很惨：  
* 比如定义了一个全局变量globalVar，却在函数内部用  
```  
    globalVar := make(map[int32]string, 0)
```  
上面代码，其实是重新定义了局部变量globalVar，正确的做法是  
```
    globalVar = make(map[int32]string, 0)  
```
* 指针和append，Golang是支持指针的，并且当数组作为函数参数时，是值传递。并且，要注意了，append可能会新建一个数组，所以需要把append的返回结果保存。  

2. 缓存结构重新设计，用了进程内缓存  
改版之后，并没有把消息设置到redis中，而是直接插入mysql，那怎么取消息呢？这就是大牛的牛逼之处了，在程序里面跑一个goroutine，负责定期取可用的消息，并保证消息没有过期。这步操作有两个好处：  
* 客户端拉取消息时，不需要请求redis，直接读取进程内缓存即可  
* 旧版获取到msgId之后，需要再请求redis才能获取到消息的完整信息，包括有效期，而新版通过一个简单的sql就能根据消息的startTime和endTime保证消息有效并且不会提前下发  

同一个时间段内，其实需要下发的消息总数不会很多，占用进程内缓存也会不会很大，至于旧版考虑的Client ID重复存储导致redis内存过大的问题，其实新版的存储策略是这样的：

 >  后台设置消息时，如果是包含和排除这两种规则，用redis的Set数据结构，将提前生成的msgId和规则名称组成key，Client ID组成value，根据消息的startTime和endTime设置expire时间，只要消息过期，这部分缓存就会清掉，所以缓存也不会居高不下。  