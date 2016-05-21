---
layout: post
title: LinkedHashMap 源码分析
category: program
---

在Java中，HashMap是无序的，而LinkedHashMap和TreeMap有序，本篇就来分析一下LinkedHashMap的源码。LinkedHashMap是HashMap的子类，可以在构造函数中配置`accessOrder`设定访问有序或插入有序。以下分析都是基于jdk1.7.0_51分析得来。

先看类的定义：

```java
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V> {
    // 双向链表的头结点
	private transient Entry<K,V> header;
	// 指定该LinkedHashMap是否按照访问排序，否则按照插入排序，注意该属性为final，默认为false
	private final boolean accessOrder;

	@Override
    void init() {
		 // 初始化代码
	     // ... 
    }
    
    @Override
    void transfer(HashMap.Entry[] newTable, boolean rehash) {
	    // 具体代码见下文
        // ...
    }
}
```

`LinkedHashMap`是`HashMap`的子类，重写了`init`和`transfer`方法。

### init 方法
前者用于HashMap的初始化，由父类的构造函数调用：

```java
// HashMap.java
   public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);

        this.loadFactor = loadFactor;
        threshold = initialCapacity;
        init();
    }

// LinkedHashMap.java
	@Override
    void init() {
		 // 初始化代码
        header = new Entry<>(-1, null, null, null);
        header.before = header.after = header;
    }
```

初始化时创建双向链表的头结点。

### transfer 方法

transfer方法在HashMap rehash的时候调用，用于将bucket中的数据转移到新的数组中。

```java
// LinkedHashMap.java
    @Override
    void transfer(HashMap.Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e = header.after; e != header; e = e.after) {
            if (rehash)
                e.hash = (e.key == null) ? 0 : hash(e.key);
            int index = indexFor(e.hash, newCapacity);
            e.next = newTable[index];
            newTable[index] = e;
        }
    }

// HashMap.java
    void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }
```

父类HashMap的transfer过程比较简单：
1. 遍历所有的结点；
2. 重新计算结点的数组index，然后将该结点插入到bucket list的头结点位置。
而子类LinkedHashMap transfer函数关系到header，使用双向链表遍历，这段代码需要先分析get和put方法。

### get方法

和HashMap一样，LinkedHashMap的key和value都可以为null，当取出key对应的value为null时，可能表示key不存在，或者该key的值为null，这两种情况可以通过`containsKey`区分。

```java
// LinkedHashMap
    public V get(Object key) {
        Entry<K,V> e = (Entry<K,V>)getEntry(key);
        if (e == null)
            return null;
        e.recordAccess(this);
        return e.value;
    }

//Entry
   void recordAccess(HashMap<K,V> m) {
        LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;
        if (lm.accessOrder) {
            lm.modCount++;
            remove();
            addBefore(lm.header);
        }
    }
```

如果`accessOrder`为`true`，即按照访问排序，则把Entry从当前位置删除，并插入到头结点中。LinkedHashMap中的Entry定义如下：

```java
private static class Entry<K,V> extends HashMap.Entry<K,V> {
	// 双向链表中指向本结点前后两个结点的引用
	// 所以Entry包含了三个引用（HashMap.Entry的next引用）
	Entry<K,V> before, after;

	// 从双向链表中删除当前Entry
	 private void remove() {
	     before.after = after;
         after.before = before;
	  }

	// 将当前Entry结点插入existingEntry结点之前
	private void addBefore(Entry<K,V> existingEntry) {
        after  = existingEntry;
        before = existingEntry.before;
        before.after = this;
        after.before = this;
    }
}
```

在回到`recordAccess`方法，当`accessOrder`为`true`时，get方法的调用就一目了然了。

### put 方法

put方法比get方法要复杂，相比HashMap，多了两个函数调用：

```java
// HashMap.java
   public V put(K key, V value) {
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);
        }
        if (key == null)
            return putForNullKey(value);
        int hash = hash(key);
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                // 如果accessOrder为true，则会将该结点插入到头结点位置
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        // 
        addEntry(hash, key, value, i);
        return null;
    }
    
    // LinkedHashMap.java
   void addEntry(int hash, K key, V value, int bucketIndex) {
        super.addEntry(hash, key, value, bucketIndex);

        // Remove eldest entry if instructed
        Entry<K,V> eldest = header.after;
        if (removeEldestEntry(eldest)) {
            removeEntryForKey(eldest.key);
        }
    }
```

子类重写了`addEntry`方法，除了执行父类方法中的原有逻辑，还会判断是否需要删除Entry。`super.addEntry`将当前key-value添加到Entry数组中，如果空间有剩余，则调用`createEntry`添加Entry，和父类不同的是，子类最后会将该结点添加到双向队列的头结点位置。当容量不足时，执行rehash操作，所以，这里回到了上文中的`transfer`方法：
1. 利用双向链表遍历整个Entry数组；
2. 遍历过程中根据新的Entry数组，重新计算每个结点的index，设置bucket链表；
3. 双向链表的结构并不需要变化。
由此可以看到，LinkedHashMap和HashMap rehash的区别在于，前者通过双向链表即可完成遍历操作，而后者需要遍历Entry数组中的Entry，也就是逐个遍历bucket。

分析完rehash流程，继续看`addEntry`，在完成结点添加之后，LinkedHashMap会判断是否需要删除最老的Entry，如果是，则会执行删除操作，这段代码，可以用来实现LRU。





