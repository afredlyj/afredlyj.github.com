---
layout: post
title: mysql socket closed异常 
category: bug
---

@[mysql|database]

最近刮刮卡mysql线上每隔几天就会报出相同的异常，并且根据log来看，都是凌晨2:33左右，让人很费解，具体的错误日志如下：  

~~~~  

The last packet successfully received from the server was 138,859 milliseconds ago.  The last packet sent successfully to the server was 1 milliseconds ago.
### The error may exist in com/rom/scratchcard/conf/mybatis/prize.xml
### The error may involve com.rom.querprize-Inline
### The error occurred while setting parameters
### SQL: {call   querprize(?,?,?,?,?,?,?,?,?)   }
### Cause: com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure

The last packet successfully received from the server was 138,859 milliseconds ago.  The last packet sent successfully to the server was 1 milliseconds ago.
; SQL []; Communications link failure

The last packet successfully received from the server was 138,859 milliseconds ago.  The last packet sent successfully to the server was 1 milliseconds ago.; 
nested exception is com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure

The last packet successfully received from the server was 138,859 milliseconds ago.  The last packet sent successfully to the server was 1 milliseconds ago.
        at org.springframework.jdbc.support.SQLExceptionSubclassTranslator.doTranslate(SQLExceptionSubclassTranslator.java:98) ~
        [org.springframework.jdbc-3.0.5.RELEASE.jar:3.0.5.RELEASE]
        at org.springframework.jdbc.support.AbstractFallbackSQLExceptionTranslator.translate(AbstractFallbackSQLExceptionTranslator.java:72) ~
        [org.springframework.jdbc-3.0.5.RELEASE.jar:3.0.5.RELEASE]
        at org.springframework.jdbc.support.AbstractFallbackSQLExceptionTranslator.translate(AbstractFallbackSQLExceptionTranslator.java:80) ~
        [org.springframework.jdbc-3.0.5.RELEASE.jar:3.0.5.RELEASE]
        at org.mybatis.spring.MyBatisExceptionTranslator.translateExceptionIfPossible(MyBatisExceptionTranslator.java:71) ~[mybatis-spring-1.1.1.jar:1.1.1]
        at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:365) ~[mybatis-spring-1.1.1.jar:1.1.1]
        at com.sun.proxy.$Proxy34.selectOne(Unknown Source) ~[na:na]
        at org.mybatis.spring.SqlSessionTemplate.selectOne(SqlSessionTemplate.java:160) ~[mybatis-spring-1.1.1.jar:1.1.1]
        at com.rom.scratchcard.cache.MybatisCache.getCache(MybatisCache.java:79) ~[classes:na]
        at com.nearme.base.cache.AbstractCache2.get(AbstractCache2.java:45) ~[OBaseTools.jar:na]
        at com.nearme.base.cache.AbstractCache2.get(AbstractCache2.java:69) ~[OBaseTools.jar:na]
        at com.rom.scratchcard.cache.CacheManager.getObject(CacheManager.java:66) ~[classes:na]
        at com.rom.scratchcard.business.PrizeBusiness.checkUser(PrizeBusiness.java:135) [classes:na]
        at com.rom.scratchcard.business.PrizeBusiness.querPrize(PrizeBusiness.java:83) [classes:na]
        at com.rom.scratchcard.action.QuerPrize.processRequest(QuerPrize.java:55) [classes:na]
        at com.rom.scratchcard.action.QuerPrize.doPost(QuerPrize.java:38) [classes:na]
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:158) [javaee-16.jar:na]
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:96) [javaee-16.jar:na]
        at com.caucho.server.dispatch.ServletFilterChain.doFilter(ServletFilterChain.java:109) [resin.jar:4.0.15]
        at com.caucho.server.webapp.WebAppFilterChain.doFilter(WebAppFilterChain.java:184) [resin.jar:4.0.15]
        at com.caucho.server.dispatch.ServletInvocation.service(ServletInvocation.java:287) [resin.jar:4.0.15]
        at com.caucho.server.hmux.HmuxRequest.handleInvocation(HmuxRequest.java:468) [resin.jar:4.0.15]
        at com.caucho.server.hmux.HmuxRequest.handleRequestImpl(HmuxRequest.java:373) [resin.jar:4.0.15]
        at com.caucho.server.hmux.HmuxRequest.handleRequest(HmuxRequest.java:339) [resin.jar:4.0.15]
        at com.caucho.network.listen.TcpSocketLink.dispatchRequest(TcpSocketLink.java:729) [resin.jar:4.0.15]
        at com.caucho.network.listen.TcpSocketLink.handleRequest(TcpSocketLink.java:688) [resin.jar:4.0.15]
        at com.caucho.network.listen.TcpSocketLink.handleRequestsImpl(TcpSocketLink.java:668) [resin.jar:4.0.15]
        at com.caucho.network.listen.TcpSocketLink.handleRequests(TcpSocketLink.java:616) [resin.jar:4.0.15]
        at com.caucho.network.listen.AcceptTask.doTask(AcceptTask.java:104) [resin.jar:4.0.15]
        at com.caucho.network.listen.ConnectionReadTask.runThread(ConnectionReadTask.java:98) [resin.jar:4.0.15]
        at com.caucho.network.listen.ConnectionReadTask.run(ConnectionReadTask.java:81) [resin.jar:4.0.15]
        at com.caucho.network.listen.AcceptTask.run(AcceptTask.java:67) [resin.jar:4.0.15]
        at com.caucho.env.thread.ResinThread.runTasks(ResinThread.java:164) [resin.jar:4.0.15]
        at com.caucho.env.thread.ResinThread.run(ResinThread.java:130) [resin.jar:4.0.15]
Caused by: com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure

The last packet successfully received from the server was 138,859 milliseconds ago.  The last packet sent successfully to the server was 1 milliseconds ago.
        at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method) ~[na:1.6.0_41]
        at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:39) ~[na:1.6.0_41]
        at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:27) ~[na:1.6.0_41]
        at java.lang.reflect.Constructor.newInstance(Constructor.java:513) ~[na:1.6.0_41]
        at com.mysql.jdbc.Util.handleNewInstance(Util.java:409) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at com.mysql.jdbc.SQLError.createCommunicationsException(SQLError.java:1118) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:3055) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:2941) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at com.mysql.jdbc.MysqlIO.checkErrorPacket(MysqlIO.java:3489) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at com.mysql.jdbc.MysqlIO.sendCommand(MysqlIO.java:1959) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at com.mysql.jdbc.MysqlIO.sqlQueryDirect(MysqlIO.java:2113) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2568) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at com.mysql.jdbc.PreparedStatement.executeInternal(PreparedStatement.java:2113) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at com.mysql.jdbc.PreparedStatement.execute(PreparedStatement.java:1364) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at com.mysql.jdbc.CallableStatement.execute(CallableStatement.java:879) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at sun.reflect.GeneratedMethodAccessor114.invoke(Unknown Source) ~[na:na]
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25) ~[na:1.6.0_41]
        at java.lang.reflect.Method.invoke(Method.java:597) ~[na:1.6.0_41]
        at org.logicalcobwebs.proxool.ProxyStatement.invoke(ProxyStatement.java:100) ~[proxool-0.9.1.jar:na]
        at org.logicalcobwebs.proxool.ProxyStatement.intercept(ProxyStatement.java:57) ~[proxool-0.9.1.jar:na]
        at $java.sql.Statement$$EnhancerByProxool$$4dd9dfad.execute(<generated>) ~[proxool-cglib.jar:na]
        at sun.reflect.GeneratedMethodAccessor44.invoke(Unknown Source) ~[na:na]
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25) ~[na:1.6.0_41]
        at java.lang.reflect.Method.invoke(Method.java:597) ~[na:1.6.0_41]
        at org.apache.ibatis.logging.jdbc.PreparedStatementLogger.invoke(PreparedStatementLogger.java:58) ~[mybatis-3.1.1.jar:3.1.1]
        at com.sun.proxy.$Proxy39.execute(Unknown Source) ~[na:na]
        at org.apache.ibatis.executor.statement.CallableStatementHandler.query(CallableStatementHandler.java:63) ~[mybatis-3.1.1.jar:3.1.1]
        at org.apache.ibatis.executor.statement.RoutingStatementHandler.query(RoutingStatementHandler.java:70) ~[mybatis-3.1.1.jar:3.1.1]
        at org.apache.ibatis.executor.SimpleExecutor.doQuery(SimpleExecutor.java:57) ~[mybatis-3.1.1.jar:3.1.1]
        at org.apache.ibatis.executor.BaseExecutor.queryFromDatabase(BaseExecutor.java:267) ~[mybatis-3.1.1.jar:3.1.1]
        at org.apache.ibatis.executor.BaseExecutor.query(BaseExecutor.java:141) ~[mybatis-3.1.1.jar:3.1.1]
        at org.apache.ibatis.executor.CachingExecutor.query(CachingExecutor.java:105) ~[mybatis-3.1.1.jar:3.1.1]
        at org.apache.ibatis.executor.CachingExecutor.query(CachingExecutor.java:81) ~[mybatis-3.1.1.jar:3.1.1]
        at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:101) ~[mybatis-3.1.1.jar:3.1.1]
        at org.apache.ibatis.session.defaults.DefaultSqlSession.selectList(DefaultSqlSession.java:95) ~[mybatis-3.1.1.jar:3.1.1]
        at org.apache.ibatis.session.defaults.DefaultSqlSession.selectOne(DefaultSqlSession.java:59) ~[mybatis-3.1.1.jar:3.1.1]
        at sun.reflect.GeneratedMethodAccessor64.invoke(Unknown Source) ~[na:na]
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:25) ~[na:1.6.0_41]
        at java.lang.reflect.Method.invoke(Method.java:597) ~[na:1.6.0_41]
        at org.mybatis.spring.SqlSessionTemplate$SqlSessionInterceptor.invoke(SqlSessionTemplate.java:355) ~[mybatis-spring-1.1.1.jar:1.1.1]
        ... 28 common frames omitted
Caused by: java.net.SocketException: Socket closed
        at java.net.SocketInputStream.socketRead0(Native Method) ~[na:1.6.0_41]
        at java.net.SocketInputStream.read(SocketInputStream.java:129) ~[na:1.6.0_41]
        at com.mysql.jdbc.util.ReadAheadInputStream.fill(ReadAheadInputStream.java:114) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at com.mysql.jdbc.util.ReadAheadInputStream.readFromUnderlyingStreamIfNecessary(ReadAheadInputStream.java:161) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at com.mysql.jdbc.util.ReadAheadInputStream.read(ReadAheadInputStream.java:189) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at com.mysql.jdbc.MysqlIO.readFully(MysqlIO.java:2499) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:2952) ~[mysql-connector-java-5.1.13-bin.jar:na]
        ... 61 common frames omitted

~~~~  

看了半天，没看出问题，google后发现有可能是mysql的一个连接配置的问题：  

~~~~
autoReconnect=true
~~~~

更新到正式环境后并没有解决问题，没别的办法，只能再看一次日志，避免有遗漏的点，幸亏项目刚上不久，还有debug日志，在报异常的这个点，发现了一个WARN：  

~~~~
[WARN ] 2013-12-29 02:33:39 [HouseKeeper][org.logicalcobwebs.proxool.account_read:149] - 
#0154 was active for 130395 milliseconds and has been removed automaticaly. 
The Thread responsible was named 'server://127.0.0.1:6809-489',
and the last SQL it performed is '{call
                querprize('860813021676809','V1.00_1311210.W16110_131223','460026995709861','X909T','X909TROM_12_131227_Beta','2','19978463','0',9)
                }; '.
~~~~  

再结合上面的错误日志：  

~~~~
Caused by: java.net.SocketException: Socket closed
        at java.net.SocketInputStream.socketRead0(Native Method) ~[na:1.6.0_41]
        at java.net.SocketInputStream.read(SocketInputStream.java:129) ~[na:1.6.0_41]
        at com.mysql.jdbc.util.ReadAheadInputStream.fill(ReadAheadInputStream.java:114) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at com.mysql.jdbc.util.ReadAheadInputStream.readFromUnderlyingStreamIfNecessary(ReadAheadInputStream.java:161) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at com.mysql.jdbc.util.ReadAheadInputStream.read(ReadAheadInputStream.java:189) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at com.mysql.jdbc.MysqlIO.readFully(MysqlIO.java:2499) ~[mysql-connector-java-5.1.13-bin.jar:na]
        at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:2952) ~[mysql-connector-java-5.1.13-bin.jar:na]
        ... 61 common frames omitted
~~~~  

基本可以看出，一个mysql链接执行时间太长，被proxool house keeper线程直接关闭，应用程序以后该链接仍然可用，企图获取存储过程返回值（这里可能表述有问题）时报了`Socket closed`，那么，接下来就需要在测试环境验证重现这个异常，所以在存储过程里面添加了一个耗时操作（for循环）：  

~~~~
while i < 1000000 do
set i = i + 1;
end while;
~~~~

OK，复现成功，确实是由于存储过程耗时太长，导致house keeper将链接回收，但是存储过程本身都是一些很简单的insert, update操作，存储引擎用的MyISAM，应该不会导致死锁，最后的可能就只能是这个时间段机器I/O负载太高，导致mysql性能受到影响，找运维同事确认，确实这个时间段再跑mysql备份，好吧。  
解决办法：  

 > 设置连接池链接最长活跃时间:maximum-active-time

搞定，继续观察~~
