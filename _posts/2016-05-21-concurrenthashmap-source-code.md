---
layout: post
title: ConcurrentHashMap 源码分析
category: program
---

>本来打算自己写一篇分析ConcurrentHashMap源码的文章，粗读一次源码之后，发现还有很多无法理解的点，google之，找到了很多优秀的分析文章，所以用摘抄的形式完成这篇分析。

### 初识
从字面理解，`ConcurrentHashMap`是支持并发的Map，这是与`HashMap`最本质的区别，后者在多线程中`resize`方法会出现无限循环，从而导致CPU 100%的问题，解决这个问题的办法，要么使用`HashTable`，要么使用`ConcurrentHashMap`。相比`HashTable`，本文的主角效率上会优秀很多，因为采用分段锁的机制，配合哈希算法，保证数据尽可能离散分布，减少锁的竞争，从而提升效率。

想要说明分段锁，需要借用两张图：
![图片无法显示](../assets/images/concurrenthashmap1.png "")  

第一张是类图，`ConcurrentHashMap`实现了`ConcurrentMap`接口（图中没有体现），定义了如下四个方法，都是原子性的：
```java
// 如果key不存在，则将key和value关联，否则返回key对应的value
V putIfAbsent(K key, V value);

// 当且仅当map中的key对应的值为value时，删除该key，否则返回false
boolean remove(Object key, Object value);

// 如果当前map中的key对应的值为oldValue，则将其替换为newValue
boolean replace(K key, V oldValue, V newValue);

// 如果当前map中有key，则将其值替换为value
V replace(K key, V value);
```

如果不能从上图中看出整个Map的存储结构，下图则表现得更加明显：
![图片无法显示](../assets/images/concurrenthashmap3.png "")  

Map中默认包含16个Segment数组，每个Segment数组是一个Hash 表，Hash表的结构和`HashMap`中的类似。从上图所知定位一个元素，需要两次Hash操作，第一次Hash定位到Segment，第二次Hash定位到元素所在链表的头部，所以其Hash过程要比普通的HashMap要长，但正是由于这种结构，使得写入时只需要对写入对应的Segment加锁，提供了并发效率。

### 代码分析
`ConcurrrentHashMap`可以做到读取数据不加锁，并且其内部的结构可以让其在进行写操作的时候能够将锁的粒度保持地尽量地小，不用对整个ConcurrentHashMap加锁。接下来分析两个主要方法`get`和`put`。在分析这两个方法之前，需要关注其内部的数据结构：

#### Segment

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {
	// 当前Segment中元素的个数
	transient volatile int count;
	// 改变Segment大小的操作次数，比如put或remove
	transient int modCount;
	// 阈值，Segment里面元素的数量超过这个值依旧就会对Segment进行扩容
	transient int threshold;
	// 
	transient volatile HashEntry<K,V>[] table;
	//
	final float loadFactor;
}
```

#### HashEntry

```java
static final class HashEntry<K,V> {
	final K key;
	final int hash;
	volatile V value;
	final HashEntry<K,V> next;
	HashEntry(K key, int hash, HashEntry<K,V> next, V value) {
		this.key = key;
		this.hash = hash;
		this.next = next;
		this.value = value;
	}
}
```
可以看到HashEntry的一个特点，除了value以外，其他的几个变量都是final的，next为final，也就是说，新节点只能在链表的表头处插入。

#### get方法

get方法不需要加锁，通过`count`和`value`域保证可见性，除非读取到空，才会加锁重读。

```java
// ConcurrentHashMap
public V get(Object key) {
    int hash = hash(key.hashCode());
    return segmentFor(hash).get(key, hash);
}

// Segment
V get(Object key, int hash) {
     if (count != 0) { // read-volatile
           HashEntry<K,V> e = getFirst(hash);
           while (e != null) {
                if (e.hash == hash && key.equals(e.key)) {
                    V v = e.value;
                    if (v != null)
                        return v;
                    return readValueUnderLock(e); // recheck
                }
                e = e.next;
            }
         }
        return null;
   }
```
由于Java 内存模型的偏序原则（happens-before原则），对volatile变量的写入操作happens-before对该volatile变量的读操作。而这两个变量的值，都会在put和remove操作中更新，这样就保证了写入操作对读操作可见。关于`count`和`value`这两个值，需要说明：
>所有改变bin结构（HashEntry链表）写操作都会写一下count，可以保证HashEntry的可见性(因为无论是添加还是删除，bin起始的HashEntry都会发生变化，由于HashEntry的next域是不变的，所以删除时需要将目标HashEntry之前的Entry都拷贝一下)。而覆盖旧值的情况下不会写count，因为HashEntry的value本身也是volatile的，可以保证自身的可见性。
       
在遍历获取到key之后，如果当前value为空，则会进入`readValueUnderLock`方法，该方法会加锁读取value值，这点还不能理解，作者解释如下：
>For those interested in an answer from the Doug Lea on this topic, he recently exlpained the reason for readValueUnderLock
>
>This is in response to someone who had the question:
>
>In the ConcurrentHashMap the get method does not require "readValueUnderLock" because a racing remove does not make >the value null. The value never becomes null on the from the removing thread. this means it is possible for get to return a value >for key even if the removing thread (on the same key) has progressed till the point of cloning the preceding parts of the list. This is fine so long as it is the desired effect.
>
>But this means "readValueUnderLock" is not required for NEW memory model.
>
>However for the OLD memory model a put may see the value null due to reordering(Rare but possible).
>
>Is my understanding correct.
>
>Response:
>
>Not quite. You are right that it should never be called. However, the JLS/JMM can be read as not absolutely forbidding it from being called because of weaknesses in required ordering relationships among finals vs volatiles set in constructors (key is final, value is volatile), wrt the reads by threads using the entry objects. (In JMM-ese, ordering constraints for finals fall outside of the synchronizes-with relation.) That's the issue the doc comment (pasted below) refers to. No one has ever thought of any practical loophole that a processor/compiler might find to produce a null value read, and it may be provable that none exist (and perhaps someday a JLS/JMM revision will fill in gaps to clarify this), but Bill Pugh once suggested we put this in anyway just for the sake of being conservatively pedantically correct. In retrospect, I'm not so sure this was a good idea, since it leads people to come up with exotic theories.

原文[在此](http://stackoverflow.com/questions/5002428/concurrenthashmap-reorder-instruction)。由于当读取到value为null，则判断需要加锁重读，所以ConcurrentHashMap的value不支持null，这与HashMap不同，后者支持key-value均为null的操作。

#### put方法

```java
// ConcurrentHashMap
public V put(K key, V value) {
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key.hashCode());
    return segmentFor(hash).put(key, hash, value, false);
    }
   
// Segment
V put(K key, int hash, V value, boolean onlyIfAbsent) {
    lock();
    try {
        int c = count;
        if (c++ > threshold) // ensure capacity
            rehash();
        HashEntry<K,V>[] tab = table;
        int index = hash & (tab.length - 1);
        HashEntry<K,V> first = tab[index];
        HashEntry<K,V> e = first;
        while (e != null && (e.hash != hash || !key.equals(e.key)))
            e = e.next;

        V oldValue;
        if (e != null) {
            oldValue = e.value;
            if (!onlyIfAbsent)
                // 写入新值，覆盖旧值，volatile变量的写操作
                e.value = value;
        }
         else {
            oldValue = null;
            // 由于会改变Segment的元素个数，所以modCount会自增，该变量的用法可以在size方法中看到
            ++modCount;
            tab[index] = new HashEntry<K,V>(key, hash, first, value);
            // volatile变量的写操作
            count = c; // write-volatile
        }
        return oldValue;
    } finally {
        unlock();
    }
}
```

第一次Hash定位到Segment后，进入Segment的put方法，首先会加锁。
put方法会在加入新的HashEntry前，判断是否需要rehash，而HashMap则是在加入之后判断，可能导致一次无用的rehash。另外，ConcurrentHashMap并不会对整个Map扩容，而只是对Segmetn扩容。


#### 弱一致性

参考[为什么ConcurrentHashMap是弱一致的](http://ifeve.com/concurrenthashmap-weakly-consistent/)。

### 参考文档

* http://ifeve.com/concurrenthashmap/comment-page-1/#comment-26817
* http://goldendoc.iteye.com/blog/1103980
* https://www.ibm.com/developerworks/cn/java/java-lo-concurrenthashmap/
* http://brokendreams.iteye.com/blog/2253345
