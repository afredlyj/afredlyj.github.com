---
layout: post
title: ConcurrentHashMap Դ�����
category: program
---

>���������Լ�дһƪ����ConcurrentHashMapԴ������£��ֶ�һ��Դ��֮�󣬷��ֻ��кܶ��޷����ĵ㣬google֮���ҵ��˺ܶ�����ķ������£�������ժ������ʽ�����ƪ������

### ��ʶ
��������⣬`ConcurrentHashMap`��֧�ֲ�����Map��������`HashMap`��ʵ����𣬺����ڶ��߳���`resize`�������������ѭ�����Ӷ�����CPU 100%�����⣬����������İ취��Ҫôʹ��`HashTable`��Ҫôʹ��`ConcurrentHashMap`�����`HashTable`�����ĵ�����Ч���ϻ�����ܶ࣬��Ϊ���÷ֶ����Ļ��ƣ���Ϲ�ϣ�㷨����֤���ݾ�������ɢ�ֲ����������ľ������Ӷ�����Ч�ʡ�

��Ҫ˵���ֶ�������Ҫ��������ͼ��
![ͼƬ�޷���ʾ](../assets/images/concurrenthashmap1.png "")  

��һ������ͼ��`ConcurrentHashMap`ʵ����`ConcurrentMap`�ӿڣ�ͼ��û�����֣��������������ĸ�����������ԭ���Եģ�
```java
// ���key�����ڣ���key��value���������򷵻�key��Ӧ��value
V putIfAbsent(K key, V value);

// ���ҽ���map�е�key��Ӧ��ֵΪvalueʱ��ɾ����key�����򷵻�false
boolean remove(Object key, Object value);

// �����ǰmap�е�key��Ӧ��ֵΪoldValue�������滻ΪnewValue
boolean replace(K key, V oldValue, V newValue);

// �����ǰmap����key������ֵ�滻Ϊvalue
V replace(K key, V value);
```

������ܴ���ͼ�п�������Map�Ĵ洢�ṹ����ͼ����ֵø������ԣ�
![ͼƬ�޷���ʾ](../assets/images/concurrenthashmap3.png "")  

Map��Ĭ�ϰ���16��Segment���飬ÿ��Segment������һ��Hash ��Hash��Ľṹ��`HashMap`�е����ơ�����ͼ��֪��λһ��Ԫ�أ���Ҫ����Hash��������һ��Hash��λ��Segment���ڶ���Hash��λ��Ԫ�����������ͷ����������Hash����Ҫ����ͨ��HashMapҪ�����������������ֽṹ��ʹ��д��ʱֻ��Ҫ��д���Ӧ��Segment�������ṩ�˲���Ч�ʡ�

### �������
`ConcurrrentHashMap`����������ȡ���ݲ��������������ڲ��Ľṹ���������ڽ���д������ʱ���ܹ����������ȱ��ֵؾ�����С�����ö�����ConcurrentHashMap����������������������Ҫ����`get`��`put`���ڷ�������������֮ǰ����Ҫ��ע���ڲ������ݽṹ��

#### Segment

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {
	// ��ǰSegment��Ԫ�صĸ���
	transient volatile int count;
	// �ı�Segment��С�Ĳ�������������put��remove
	transient int modCount;
	// ��ֵ��Segment����Ԫ�ص������������ֵ���ɾͻ��Segment��������
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
���Կ���HashEntry��һ���ص㣬����value���⣬�����ļ�����������final�ģ�nextΪfinal��Ҳ����˵���½ڵ�ֻ��������ı�ͷ�����롣

#### get����

get��������Ҫ������ͨ��`count`��`value`��֤�ɼ��ԣ����Ƕ�ȡ���գ��Ż�����ض���

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
����Java �ڴ�ģ�͵�ƫ��ԭ��happens-beforeԭ�򣩣���volatile������д�����happens-before�Ը�volatile�����Ķ���������������������ֵ��������put��remove�����и��£������ͱ�֤��д������Զ������ɼ�������`count`��`value`������ֵ����Ҫ˵����
>���иı�bin�ṹ��HashEntry����д��������дһ��count�����Ա�֤HashEntry�Ŀɼ���(��Ϊ��������ӻ���ɾ����bin��ʼ��HashEntry���ᷢ���仯������HashEntry��next���ǲ���ģ�����ɾ��ʱ��Ҫ��Ŀ��HashEntry֮ǰ��Entry������һ��)�������Ǿ�ֵ������²���дcount����ΪHashEntry��value����Ҳ��volatile�ģ����Ա�֤����Ŀɼ��ԡ�
       
�ڱ�����ȡ��key֮�������ǰvalueΪ�գ�������`readValueUnderLock`�������÷����������ȡvalueֵ����㻹������⣬���߽������£�
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

ԭ��[�ڴ�](http://stackoverflow.com/questions/5002428/concurrenthashmap-reorder-instruction)�����ڵ���ȡ��valueΪnull�����ж���Ҫ�����ض�������ConcurrentHashMap��value��֧��null������HashMap��ͬ������֧��key-value��Ϊnull�Ĳ�����

#### put����

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
                // д����ֵ�����Ǿ�ֵ��volatile������д����
                e.value = value;
        }
         else {
            oldValue = null;
            // ���ڻ�ı�Segment��Ԫ�ظ���������modCount���������ñ������÷�������size�����п���
            ++modCount;
            tab[index] = new HashEntry<K,V>(key, hash, first, value);
            // volatile������д����
            count = c; // write-volatile
        }
        return oldValue;
    } finally {
        unlock();
    }
}
```

��һ��Hash��λ��Segment�󣬽���Segment��put���������Ȼ������
put�������ڼ����µ�HashEntryǰ���ж��Ƿ���Ҫrehash����HashMap�����ڼ���֮���жϣ����ܵ���һ�����õ�rehash�����⣬ConcurrentHashMap�����������Map���ݣ���ֻ�Ƕ�Segmetn���ݡ�


#### ��һ����

�ο�[ΪʲôConcurrentHashMap����һ�µ�](http://ifeve.com/concurrenthashmap-weakly-consistent/)��

### �ο��ĵ�

* http://ifeve.com/concurrenthashmap/comment-page-1/#comment-26817
* http://goldendoc.iteye.com/blog/1103980
* https://www.ibm.com/developerworks/cn/java/java-lo-concurrenthashmap/
* http://brokendreams.iteye.com/blog/2253345
