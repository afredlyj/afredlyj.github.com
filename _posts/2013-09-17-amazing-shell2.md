---
layout: post
title: 神奇的shell2
category: shell
---

继续记录神奇的shell，越来越觉得，如果从事服务器开发，不懂shell，那就真的是蛋疼了，项目部署需要shell，分析日志需要shell...

**shell连接mysql**

上一篇就提到了shell怎样连接mysql，但仅仅是一条命令，在shell调用mysql中还有其他几个选项需要使用的，这次一并记录下来：  
* -h，--host  
连接指定host上的mysql server  
* -P，--port  
TCP/IP端口号  
* -B，--batch  
输出结果，以tab为字段分割符，每条记录以换行符分割  
* -N，--skip-column-names  
输出结果不输出字段名称  

**seq输出数字序列**

seq命令用来输出一个等差队列，man相关内容：  
* -f, --format=FORMAT  
use printf style floating-point FORMAT  
* -s, --separator=STRING  
use STRING to separate numbers (default: \n)  
* -w, --equal-width  
equalize width by padding with leading zeroes        

如果用shell批量创建1000个mysql表，表名的唯一区别是后缀：000-999：  
```
for i in `seq -w 0 999`;do  
tableName="table_"$i  
mysql -uroot -p123456 database -e "sqlstatement"  
done  
```

**awk分析日志的利器**

* awk调用的基本方式：  
```
awk [-F fild-separator] 'command' input-file(s)
```

>任何awk语句都是由模式和动作组成。模式部分决定动作语句何时触发及触发事件。处理即对数据进行的操作。如果省略模式部分，动作将时刻保持执行状态。模式可以是任何条件语句或复合语句或正则表达式。模式包括两个特殊字段B E G I N和E N D。使用B E G I N语句设置计数和打印头。B E G I N语句使用在任何文本浏览动作之前，之后文本浏览动作依据输入文件开始执行。E N D语句用来在a w k完成文本浏览动作后打印输出文本总数和结尾状态标志。如果不特别指明模式， a w k总是匹配或打印行数。
实际动作在大括号{ }内指明。{}中的多个语句用分号隔开，动作大多数用来打印，但是还有些更长的代码诸如i f和循环（l o o p i n g）语句及循环退出结构。如果不指明采取动作， a w k将打印出所有浏览出来的记录。

* awk过滤日志  
awk支持条件表达式：<, >, ==, !=，测试demo：  
```
afred@afred:~/script/data$ cat awk.test  
1 2 hello  
2 4 world  
```

选出第一域大于1的行，也就是第二行：  
```
afred@afred:~/script/data$ awk '$1>1{print}' awk.test
2 4 world
```

也可以：  
```
afred@afred:~/script/data$ awk '{if($1>1)print}' awk.test
2 4 world
```

awk的if-else还有for、while、do-while等流程控制其实跟c差不多，也支持break，continue和exit。  
awk可以用正则表达式过滤不需要的日志记录，awk用“～”表示匹配正则表达式，“！～”表示不匹配正则表达式。  
```
afred@afred:~/script/data$ awk '$3~/hello/{print}' awk.test
1 2 hello
```
