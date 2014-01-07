---
layout: post
title: mybatis jdbcType引起的异常
category: bug 
---

今天给sso添加删除记录功能的时候，mybatis报了异常，部分异常日志如下：  

~~~~
org.apache.ibatis.exceptions.PersistenceException: 
### Error updating database.  Cause: org.apache.ibatis.type.TypeException: Error setting null parameter.  Most JDBC drivers require that the JdbcType must be specified for all nullable parameters. Cause: java.sql.SQLException: Invalid column type
### The error may involve User.addDeleteRecord-Inline
### The error occurred while setting parameters
### Cause: org.apache.ibatis.type.TypeException: Error setting null parameter.  Most JDBC drivers require that the JdbcType must be specified for all nullable parameters. Cause: java.sql.SQLException: Invalid column type
        at org.apache.ibatis.exceptions.ExceptionFactory.wrapException(ExceptionFactory.java:8) ~[mybatis-3.0.4.jar:3.0.4]
        at org.apache.ibatis.session.defaults.DefaultSqlSession.update(DefaultSqlSession.java:120) ~[mybatis-3.0.4.jar:3.0.4]
        at org.apache.ibatis.session.defaults.DefaultSqlSession.insert(DefaultSqlSession.java:107) ~[mybatis-3.0.4.jar:3.0.4]
        at com.oppo.base.cache.DBConnect.insertCache(DBConnect.java:68) ~[OBaseTools.jar:na]
        at com.oppo.base.cache.AbstractCache.insert(AbstractCache.java:84) ~[OBaseTools.jar:na]
        at com.oppo.sso.dataaccess.user.UserManager.addDeleteRecord(UserManager.java:187) ~[classes:na]
        at com.oppo.sso.business.user.RegisterService.delete(RegisterService.java:419) [classes:na]
        at com.oppo.sso.controller.user.DeleteUserServlet.handle(DeleteUserServlet.java:22) [classes:na]
        at com.oppo.sso.controller.SSOCommonServlet.doPost(SSOCommonServlet.java:55) [classes:na]
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:158) [javaee-16.jar:na]
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:96) [javaee-16.jar:na]
        at com.caucho.server.dispatch.ServletFilterChain.doFilter(ServletFilterChain.java:109) [resin.jar:4.0.15]
        at com.oppo.sso.IPFilter.doFilter(IPFilter.java:43) [classes:na]

~~~~

原因在于，如果在配置sqlmap的`insert`元素时，没有指定jdbcType类型的参数,mybatis默认jdbcType.OTHER，这时如果插入的属性值为null，就会报这个异常，抛这个异常的代码：  

~~~~
if (parameter == null) {
      if (jdbcType == null) {
        try {
          ps.setNull(i, JdbcType.OTHER.TYPE_CODE);
        } catch (SQLException e) {
          throw new TypeException("Error setting null parameter.  Most JDBC drivers require that the JdbcType must be specified for all nullable parameters. Cause: " + e, e);
        }
      } else {
        ps.setNull(i, jdbcType.TYPE_CODE);
      }
    } else {
      setNonNullParameter(ps, i, parameter, jdbcType);
    }
~~~~

解决办法就是为每个属性都配置`jdbcType`， 要注意大小写，否则也会报异常。 jdbc和java类型对应如下：  


jdbcType      |     Java Type             |
--------------|---------------------------|
CHAR          | String                    |
VARCHAR       | String                    |
LONGVARCHAR   | String                    |
NUMERIC       | java.math.BigDecimal      |
DECIMAL       | java.math.BigDecimal      |
BIT           | boolean                   |
BOOLEAN       | boolean                   |
TINYINT       | byte                      |
SMALLINT      | short                     |
INTEGER       | int                       |
BIGINT        | long                      |
REAL          | float                     |
FLOAT         | double                    |
DOUBLE        | double                    |
BINARY        | byte[]                    |
VARBINARY     | byte[]                    |
LONGVARBINARY | byte[]                    |
DATE          | java.sql.Date             |
TIME          | java.sql.Time             |
TIMESTAMP     | java.sql.Timestamp        |
CLOB          | Clob                      |
BLOB          | Blob                      |
ARRAY         | Array                     |
DISTINCT      | mapping of underlying type|
STRUCT        |Struct                     |
REF           | Ref                       |
DATALINK      | java.net.URL              |
JAVA_OBJECT   | underlying Java class     |