---
layout: post
title: 神奇的shell(2)
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

seq命令用来输出一个等差队列，man相关内容:  

 * -f, --format=FORMAT  

use printf style floating-point FORMAT  

 * -s, --separator=STRING  

use STRING to separate numbers (default: \n)  

 * -w, --equal-width  

equalize width by padding with leading zeroes        

如果用shell批量创建1000个mysql表，表名的唯一区别是后缀：000-999：  

~~~~
for i in `seq -w 0 999`;do  
tableName="table_"$i  
mysql -uroot -p123456 database -e "sqlstatement"  
done  
~~~~

**awk 分析日志的利器**

* awk调用的基本方式：  

~~~~
awk [-F fild-separator] 'command' input-file(s)
~~~~

>任何awk语句都是由模式和动作组成。模式部分决定动作语句何时触发及触发事件。处理即对数据进行的操作。如果省略模式部分，动作将时刻保持执行状态。模式可以是任何条件语句或复合语句或正则表达式。模式包括两个特殊字段B E G I N和E N D。使用B E G I N语句设置计数和打印头。B E G I N语句使用在任何文本浏览动作之前，之后文本浏览动作依据输入文件开始执行。E N D语句用来在a w k完成文本浏览动作后打印输出文本总数和结尾状态标志。如果不特别指明模式， a w k总是匹配或打印行数。
实际动作在大括号{ }内指明。{}中的多个语句用分号隔开，动作大多数用来打印，但是还有些更长的代码诸如i f和循环（l o o p i n g）语句及循环退出结构。如果不指明采取动作， a w k将打印出所有浏览出来的记录。

* awk匹配  
awk支持条件表达式：<, >, ==, !=，测试demo： 

~~~~
afred@afred:~/script/data$ cat awk.test  
1 2 hello   
2 4 world    
~~~~

选出第一域大于1的行，也就是第二行：  

~~~~
afred@afred:~/script/data$ awk '$1>1{print}' awk.test  
2 4 world  
~~~~

也可以： 

~~~~
afred@afred:~/script/data$ awk '{if($1>1)print}' awk.test  
2 4 world  
~~~~

awk的if-else还有for、while、do-while等流程控制其实跟c差不多，也支持break，continue和exit。  
awk可以用正则表达式过滤不需要的日志记录，awk用“～”表示匹配正则表达式，“！～”表示不匹配正则表达式。 

~~~~
afred@afred:~/script/data$ awk '$3~/hello/{print}' awk.test  
1 2 hello  
~~~~

* awk获取shell参数  

awk除了简单的过滤日志外，还可以分析日志，也可以和其他shell命令一起用来数据迁移。  
awk可以接受shell传递的参数，通过`-v`可以达到目的：  

~~~~
#!/bin/sh
from="table1"
awk -F'\t' -v from=${table1} '{print from}'
~~~~

* awk调用系统命令  

在awk的{}中使用`system`调用系统命令：  

~~~~
#!/bin/sh
awk '{
    sql="mysql  -uroot -p123456 -Ddb -BN -e \"select * from table1\""
    system(sql)
}'  
~~~~

* awk中打印单引号和双引号  

  awk做数据迁移时，会需要在`insert`语句的字段值上添加单引号：  

~~~~
awk '{print "'\''"}'
~~~~

双引号： 

~~~~
awk  '{print "\""}'  
~~~~

在awk中拼接sql语句其实就是类似拼接字符串，比如：  

~~~~
val="mysql  -uroot -p123456 -Ddb -BN -e \"insert into "t" (imei,pkg_name,model,app_id,update_time) values('\''"imei"'\'','\''"pkg"'\'','\''"model"'\'','\''"appid"'\'','\''"time"'\'')\""

~~~~

可以将上面的sql类似java的+操作拼接如下：    

~~~~
String str1 = "mysql  -uroot -p123456 -Ddb -BN -e \"insert into ";  
String str2 = t;    // t为变量名  
String str3 = " (imei,pkg_name,model,app_id,update_time) values('\''";  
String str4 = imei;   
String str5 = "'\'','\''";   
String str6 = pkg;   
String str7 = "'\'','\''";   
String str8 = model;   
String str9 = "'\'','\''";   
String str10 = appid;   
String str11 = "'\'','\''";   
String str12 = time;   
String str13 = "'\'')\"";    
~~~~

终于写完了，居然是12个字符串组合成的。

**快速编辑shell的一些命令**

 * shell命令行快速跳转

>[Ctrl + a] 跳转至命令行首 Ahead of line  
[Ctrl + e] 跳转至命令行尾 End of line  
[Ctrl + f] 向前跳转一个字符 jump Forward one character  
[Ctrl + b] 向后跳转一个字符 jump Backward one character  
[Alt + f] 向前跳转到下一个字的第一个字符  
[Alt + b] 向后跳转到下一个字的第一个字符  


 * 编辑命令的快捷键

>[Ctrl + w] 向后删除一个字，用来对付刚刚输入的错误字很有用  
[Ctrl + u] 从光标当前位置删除所有字符至行首  
[Ctrl + k] 从光标当前位置删除所有字符至行尾  
[Ctrl + d] 删除光标当前位置的字符  
[Ctrl + y] 粘贴最后一个被删除的字  
[Alt + d] 删除从光标当前位置，到当前字的结尾字符  

 * shell脚本调试

>-x 选项

来自[酷壳](http://coolshell.cn/articles/1379.html)

* 查看指定java进程的线程数量

>ps -eLF \| grep -c [PID]

 * 比较两个文件的不同

 >comm
逐行比较两个已排序文件的不同，选项比较少，只能比较一行的数据。

>awk
使用awk的内置变量`NR`和`FNR`，也可以比较两个文件的不同，并且选项更多，可以处理不同的需求。

>awk -F'\t' 'NR==FNR{arr[$0]}NR>FNR{if ($3 in arr) print $0}'  imei loginStat.0.log | tee result.txt

**crontab定时任务**

Linux可以通过crond服务执行定时任务，而具体的任务设定需要修改crontab。

```
usage:  crontab [-u user] file
        crontab [-u user] [ -e | -l | -r ]
                (default operation is replace, per 1003.2)
        -e      (edit user's crontab)
        -l      (list user's crontab)
        -r      (delete user's crontab)
        -i      (prompt before deleting user's crontab)
        -s      (selinux context)
```

crontab的书写规则：

><minute> <hour> <day> <month> <dow> <command>
第1列      第2列   3       4       5      6
第1列表示分钟1～59 每分钟用*或者 */1表示
第2列表示小时1～23（0表示0点）
第3列表示日期1～31
第4列表示月份1～12
第5列标识号星期0～6（0表示星期天）
第6列要运行的命令

需要注意的是，`%`在crontab中是换行符，所以如果执行命令中有`%`，需要用`\`来escape才行。比如在crontab中需要使用`date`命令：

```
*/2 * * * * /var/job/show-busy-java-threads.sh -p 2873 -c 10 >> /var/job/"cpuStat2873-"$(date +\%Y\%m\%\d\%H\%M\%S) 2>&1
```

另外，如果命令是执行一个脚本，需要在脚本中明确引用环境变量比如：

```
export PATH=/usr/local/bin:/usr/bin:/bin:/usr/games:/usr/local/jdk/bin:/usr/local/mysql/bin
```