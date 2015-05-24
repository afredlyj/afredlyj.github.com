---
layout: post
title: sleep和wait的区别
category: program
---

看了一篇stackoverflow上关于sleep和wait方法的区别，发现自己对这两个方法理解不彻底，所以纪录下来，原文地址在[这里](http://stackoverflow.com/questions/1036754/difference-between-wait-and-sleep)。

只有获取到对象的监视器后，才能调用wait和notify方法。

> A thread becomes the owner of the object's monitor in one of three ways: 
> 
>  * By executing a synchronized instance method of that object. 
>  * By executing the body of a <code>synchronized</code> statement  that synchronizes on the object. 
>  * For objects of type <code>Class,</code> by executing a synchronized static method of that class. 
>
>
> A thread can also wake up without being notified, interrupted, or timing out, a so-called <i>spurious wakeup</i>.  While this will rarely occur in practice, applications must guard against it by testing for the condition that should have caused the thread to be awakened, and continuing to wait if the condition is not satisfied.  In other words, waits should always occur in loops, like this one:
><pre>
     synchronized (obj) {
            while (&lt;condition does not hold&gt;)
                  obj.wait(timeout);
              ... // Perform action appropriate to condition
      } </pre>

当前线程可以通过如下四种方式结束休眠状态：  

* 其他线程调用监视器的`notify`方法，之后当前线程恰好被cpu选择被唤醒；
* 其他线程调用监视器的`notifyAll`方法；
* 其他线程调用`Thread.interrupt`方法中断；
* wait方法超时。

而sleep方法只能通过两种方式结束休眠： 
 
* 其他线程调用`Thread.interrupt`方法中断；
* sleep方法超时。

另外考虑到虚假唤醒（spurious wakeup），在实际项目中，wait的正常方法是放在循环体中，并判断条件是否合适。

