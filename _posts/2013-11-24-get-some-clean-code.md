---
layout: post
title: 代码摘抄
category: reading 
---

#### 用Future实现高速缓存   

在服务器编程时，会存在多个请求需要同一个资源的情况，但是获取这个资源消耗比较大（比如需要http请求partner的信息），这时可以在本地缓存结果。下面的例子，用`ConcurrentHashMap`缓存计算结果，想比如普通的HashMap+Synchronized的组合，效率要提高很多。另外，如果仅仅是普通的获取之后再存到Map中，可能会出现这种情况：Thread1判断缓存中没有值，则进入计算，此时Thread2请求到达，由于Thread1还没有完成计算，Thread2也会获取失败，进入计算，导致相同计算进行了两次，还有可能其他更严重的问题。  
如果缓存Future，Thread1获取失败，设置Future，并放入Map中，再run（注意这里的步骤，一定是先put，再run，否则还是会出现上面的问题），Thread2请求到达，判断时候存在Future（由于Thread1已经将Future存入Map，所以是存在的），然后通过get获取计算的值。
~~~~
/**  
 * @Title: Memoizer3.java
 * @Prject: TestConcurrent
 * @Package: com.demo.buildingblocks
 * @Description: TODO
 * @author: Administrator  
 * @date: 2013-11-24 上午10:33:44
 * @version: V1.0  
 */
package com.demo.buildingblocks;

import java.util.Map;
import java.util.concurrent.Callable;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;
import java.util.concurrent.FutureTask;

/**
 * @author Administrator
 *
 */
public class Memoizer3<A, V> implements Computable<A, V> {

	private final Map<A, Future<V>> cache = new ConcurrentHashMap<A, Future<V>>();
	
	private final Computable<A, V> c;
	
	public Memoizer3(Computable<A, V> c) {
		this.c = c;
	}
	
	/* (non-Javadoc)
	 * @see com.demo.buildingblocks.Computable#compute(java.lang.Object)
	 */
	@Override
	public V compute(final A arg) throws InterruptedException {
		System.out.println("compute:" + Thread.currentThread().getId());
		Future<V> f = cache.get(arg);
		if (f == null) {
			// 用Callable而不用Runnable，是因为前者有返回值
			Callable<V> eval = new Callable<V>() {
				@Override
				public V call() throws Exception {
					System.out.println("call:" + Thread.currentThread().getId());
					return c.compute(arg);
				}
			};
			
			FutureTask<V> ft = new FutureTask<V>(eval);
			f = ft;
			cache.put(arg, ft);
			ft.run();
		}
		
		try {
			return f.get();
		} catch (ExecutionException e) {
			throw new RuntimeException(e);
		}
	}
}
~~~~  

这段代码存在一个问题，即检查再运行（check-then-act），两个线程同时到达，并且ConcurrentHashMap中还没有缓存相应的值时，两个线程都会调用compute。  
下面的改版就不会出现这样的问题：  

~~~~  
/**  
 * @Title: Memoizer.java
 * @Prject: TestConcurrent
 * @Package: com.demo.buildingblocks
 * @Description: TODO
 * @author: Administrator  
 * @date: 2013-11-24 下午1:54:39
 * @version: V1.0  
 */
package com.demo.buildingblocks;

import java.util.concurrent.Callable;
import java.util.concurrent.CancellationException;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;
import java.util.concurrent.FutureTask;

/**
 * @author Administrator
 *
 */
public class Memoizer<A, V> implements Computable<A, V> {

	private final ConcurrentHashMap<A, Future<V>> cache = new ConcurrentHashMap<A, Future<V>>();
	
	private final Computable<A, V> c;
	
	public Memoizer(Computable<A, V> c) {
		this.c = c;
	}
	
	/* (non-Javadoc)
	 * @see com.demo.buildingblocks.Computable#compute(java.lang.Object)
	 */
	@Override
	public V compute(final A arg) throws InterruptedException {
		
		while (true) {
			Future<V> f = cache.get(arg);
			if (f == null) {
				// 用Callable而不用Runnable，是因为前者有返回值
				Callable<V> eval = new Callable<V>() {
					@Override
					public V call() throws Exception {
						return c.compute(arg);
					}
				};
				FutureTask<V> ft = new FutureTask<V>(eval);
				// 原子操作，ConcurrentHaspMap保证不会出现两个线程同时执行返回的情况：
				// Segment的put方法有加锁操作
				f = cache.putIfAbsent(arg, ft);
				if (f == null) {
					f = ft;
					ft.run();
				}
			}
			try {
				return f.get();
			} catch (CancellationException e) {
				cache.remove(arg, f);
			} catch (ExecutionException e) {
				launderThrowable(e);
			}
		}
	}
	
    public static RuntimeException launderThrowable(Throwable t) {
        if (t instanceof RuntimeException)
            return (RuntimeException) t;
        else if (t instanceof Error)
            throw (Error) t;
        else
            throw new IllegalStateException("Not unchecked", t);
    }
}
~~~~
