---
layout: post
title: 神奇的shell
category: shell
---

毕业一年多了，接触shell也只是平时看看服务器的日志，cp，top，tail等等一些基本命令，深度学习还是在毕业那会儿跟着ChinaUnix的几篇帖子[Shell基础二十篇](http://bbs.chinaunix.net/thread-452942-1-1.html) 和 [shell十三问？](http://bbs.chinaunix.net/thread-218853-1-1.html)，水平可想而知，所以就把一些基本的使用方法记下来。

最近消息中心改用Golang实现，要做一些数据迁移和详细的日志跟踪，才渐渐明白*where there is a shell, there is a way*。

**shell连接mysql**

先来最基本的，之前连接mysql，使用select时，使用了here-document，感觉很麻烦，后来发现其实可以很简单：  

>mysql -uroot -p123456 -BN -e "select * from userinfo"   

这样就可以获得数据，当然，我一般都是把获取的数据重定向到临时文件。  

**top命令**

在linux下，敲入top回车，可以看到当前系统的进程，当然还包括一些系统资源占用情况，由于是动态的，并不利于跟踪某一个特定的进程，可以使用-p选项：  

> top -pPID1[,PID2]  

以此查看特定的进程。top命令的一些参数，可以看[这里](http://os.51cto.com/art/201108/285581.htm)。  

**wget调试接口**

每次调试接口，都得用HttpClient自己写代码，执行、测试，部门大牛过来一句话，原来wget也可以测试接口，好吧，立马觉得，linux真TM牛逼，wget提供了HTTP请求的一些选项，基本够用了：  

 * --header=header-line，设置请求头  
 * --post-data=string，post请求的请求字符串  
 * --post-file=file，如果不想在命令行添加post请求的字符串，也可以将请求数据放到文件中  

如果端口有返回数据，默认会在当前目录新建一个文件，至于文件名，试试就知道啦~~  

**grep精确匹配**

写服务器重启脚本的时候，需要先用PID kill掉原来的进程，那么怎么得出PID的呢，我是用  

> ps aux \| grep -w SERVER_NAME  

或者  

> ps aux \| grep -w '\<SERVER_NAME\>'  

达到精确匹配的目的，万一服务器上刚好有个名字类似的服务，直接kill掉了，别人会找我干仗的~~  

**用linux删除多个redis缓存**

昨天帮别人删除服务器上某个用户的登陆缓存，一查数据库，居然有上百条数据，删除数据库数据倒是方便，但是删除redis缓存可就不能用一条sql能搞定了（起码当时是这么认为的），程序猿一定要学会偷懒，这种需求必然后前辈遇到过，故google之，果然。所以将数据库中的缓存key拷贝出来，存入文件，然后：  

> cat tmp \| xargs /usr/local/bin/redis-cli DEL  


**删除万恶的^M**

如果将windows下编辑的文件上传到linux下，有可能在linux下看到很多^M转义符号，这些符号在windows下是看不到的，如果在linux下用vim打开，有看到文件中含有这个转义符号，那么在shell处理这个文件时就有可能出问题。我知道的，有两种解决办法：  

 * 用vim打开文件，直接将^M替换掉  

 ~~~~
:%s/^M//g
 ~~~~

 注意，这里的^M并不是直接输入'^'和'M'两个字符，而是`ctrl+v`和`ctrl+m`，这样就可以解决大部分情况下的^M，起码到昨天为止我都是这么做的，so，如果还是不行，那就试试下面的办法。  

 * 用col命令  

 ~~~~~  
cat file|col -b>file.col
 ~~~~~  

 如果不放心，处理完成之后可以通过cat验证处理结果：    

 ~~~~  
 cat -e file.col  
 ~~~~  

 正常情况下，每行都是以$结尾，并且不含有^M字符。  

OK！

