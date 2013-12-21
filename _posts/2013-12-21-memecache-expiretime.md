---
layout: post
title: java memcache客户端过期时间设置
category: bug 
---

使用`com.danga.MemCached`需要注意`set`方法，通过查看[源码](http://code.livejournal.org/trac/memcached/browser/trunk/api/java/com/danga/MemCached/MemCachedClient.java?rev=143)发现一直以来的误区，之前用MemcacheClient时，直接使用:  

~~~~
  public boolean set(String key, Object value, Integer hash) {
	        return set("set",key,value, null, hash);
}
~~~~

原来int只是hash值而已，真正存到memcache确是永不过期，正确用法是调用：

~~~~
 public boolean set(String key, Object value, Date expiry) {
        return set("set",key,value,expiry,null);
}
~~~~

最后，所有的操作都是调用另外的重载`set`：

~~~~
try {
	 String cmd = cmdname + " " + key + " " + flags + " " +
	 expiry.getTime() / 1000 + " " + val.length + "\r\n";
	 sock.out().writeBytes(cmd);
	 sock.out().write(val);
	 sock.out().writeBytes("\r\n");
	 sock.out().flush();
	           
	 String tmp = sock.in().readLine();
	 if (tmp.equals("STORED")) {
	  if (debug) {
	   System.out.println("MemCache: " + cmdname + " " + key + " = " + val);
	           }
	        return true;
	  } else {
	   System.out.println("MemCache:" + cmd + tmp);
	            }
	           
	        } catch (IOException e) {
	            sock.close();
	  }
~~~~

这里可以看到`expiry.getTime() / 1000`可就是说expriy的单位是毫秒，但是这个时间是相对时间还是绝对时间，现在从客户单api看不出来了，只能通过看memcache服务端代码：

~~~~
#define REALTIME_MAXDELTA 60*60*24*30
/*
 * given time value that's either unix time or delta from current unix time, return
 * unix time. Use the fact that delta can't exceed one month (and real time value can't
 * be that low).
 */
static rel_time_t realtime(const time_t exptime) {
    /* no. of seconds in 30 days - largest possible delta exptime */
    if (exptime == 0) return 0; /* 0 means never expire */
    if (exptime > REALTIME_MAXDELTA) {
        /* if item expiration is at/before the server started, give it an
           expiration time of 1 second after the server started.
           (because 0 means don't expire).  without this, we'd
           underflow and wrap around to some large value way in the
           future, effectively making items expiring in the past
           really expiring never */
        if (exptime <= process_started)
            return (rel_time_t)1;
        return (rel_time_t)(exptime - process_started);
    } else {
        return (rel_time_t)(exptime + current_time);
    }
}
~~~~  
memcache的最长过期时间是30天，并且相对时间和绝对时间都是可以的。