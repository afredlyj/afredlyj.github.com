---
layout: post
title: LinkedHashMap Դ�����
category: program
---

��Java�У�HashMap������ģ���LinkedHashMap��TreeMap���򣬱�ƪ��������һ��LinkedHashMap��Դ�롣LinkedHashMap��HashMap�����࣬�����ڹ��캯��������`accessOrder`�趨�������������������·������ǻ���jdk1.7.0_51����������

�ȿ���Ķ��壺
```java
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V> {
    // ˫�������ͷ���
	private transient Entry<K,V> header;
	// ָ����LinkedHashMap�Ƿ��շ������򣬷����ղ�������ע�������Ϊfinal��Ĭ��Ϊfalse
	private final boolean accessOrder;

	@Override
    void init() {
		 // ��ʼ������
	     // ... 
    }
    
    @Override
    void transfer(HashMap.Entry[] newTable, boolean rehash) {
	    // ������������
        // ...
    }
}
```

`LinkedHashMap`��`HashMap`�����࣬��д��`init`��`transfer`������

### init ����
ǰ������HashMap�ĳ�ʼ�����ɸ���Ĺ��캯�����ã�

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
		 // ��ʼ������
        header = new Entry<>(-1, null, null, null);
        header.before = header.after = header;
    }
```

��ʼ��ʱ����˫�������ͷ��㡣

### transfer ����

transfer������HashMap rehash��ʱ����ã����ڽ�bucket�е�����ת�Ƶ��µ������С�

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

����HashMap��transfer���̱Ƚϼ򵥣�
1. �������еĽ�㣻
2. ���¼����������index��Ȼ�󽫸ý����뵽bucket list��ͷ���λ�á�
������LinkedHashMap transfer������ϵ��header��ʹ��˫�������������δ�����Ҫ�ȷ���get��put������

### get����

��HashMapһ����LinkedHashMap��key��value������Ϊnull����ȡ��key��Ӧ��valueΪnullʱ�����ܱ�ʾkey�����ڣ����߸�key��ֵΪnull���������������ͨ��`containsKey`���֡�

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

���`accessOrder`Ϊ`true`�������շ����������Entry�ӵ�ǰλ��ɾ���������뵽ͷ����С�LinkedHashMap�е�Entry�������£�

```java
private static class Entry<K,V> extends HashMap.Entry<K,V> {
	// ˫��������ָ�򱾽��ǰ��������������
	// ����Entry�������������ã�HashMap.Entry��next���ã�
	Entry<K,V> before, after;

	// ��˫��������ɾ����ǰEntry
	 private void remove() {
	     before.after = after;
         after.before = before;
	  }

	// ����ǰEntry������existingEntry���֮ǰ
	private void addBefore(Entry<K,V> existingEntry) {
        after  = existingEntry;
        before = existingEntry.before;
        before.after = this;
        after.before = this;
    }
}
```

�ڻص�`recordAccess`��������`accessOrder`Ϊ`true`ʱ��get�����ĵ��þ�һĿ��Ȼ�ˡ�

### put ����

put������get����Ҫ���ӣ����HashMap�����������������ã�

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
                // ���accessOrderΪtrue����Ὣ�ý����뵽ͷ���λ��
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

������д��`addEntry`����������ִ�и��෽���е�ԭ���߼��������ж��Ƿ���Ҫɾ��Entry��`super.addEntry`����ǰkey-value��ӵ�Entry�����У�����ռ���ʣ�࣬�����`createEntry`���Entry���͸��಻ͬ���ǣ��������Ὣ�ý����ӵ�˫����е�ͷ���λ�á�����������ʱ��ִ��rehash���������ԣ�����ص��������е�`transfer`������
1. ����˫�������������Entry���飻
2. ���������и����µ�Entry���飬���¼���ÿ������index������bucket����
3. ˫������Ľṹ������Ҫ�仯��
�ɴ˿��Կ�����LinkedHashMap��HashMap rehash���������ڣ�ǰ��ͨ��˫����������ɱ�����������������Ҫ����Entry�����е�Entry��Ҳ�����������bucket��

������rehash���̣�������`addEntry`������ɽ�����֮��LinkedHashMap���ж��Ƿ���Ҫɾ�����ϵ�Entry������ǣ����ִ��ɾ����������δ��룬��������ʵ��LRU��





