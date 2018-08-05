---
title: 脚踏实地-HashMap
date: 2018-08-01 00:15:06
tags:
- Java
- HashMap
categories:
- 脚踏实地

---

<i>带着问题分析HashMap源码，包括：如何初始化，基础操作的实现，扩容，遍历等。</i>

<!-- more -->

## 前言



前些日子找工作面试的时候暴露了我的一个问题，有些问题知其然而不知其所以然。例如：我们都知道HashMap不是线程安全的，那么多线程中使用HashMap会产生什么问题？ConcurrentHashMap是线程安全的，为了线程安全做了哪些牺牲，有什么局限性？无法回答这些问题的原因是，没有看源码或是看源码时没有进行思考，所以准备脚踏实地带着问题看看源码。

本篇的主角是HashMap,下面会带着问题对源码进行分析。

Java版本是1.8.0_45

#### 目录

* [如何初始化HashMap](#init)

* [如何存储数据](#struct)

* [如何扩容](#resize)

* [如何遍历](#iter)

* [ConcurrentModificationException](#ConcurrentModificationException)

* [线程不安全体现在哪里](#unsafe)

* [如何保证效率](#effect)

* [tips](#tips)


<h2 id="init">如何初始化HashMap</h2>

HashMap提供了四种初始化方法

```
    /* 使用默认参数构建 */
    public HashMap();

    /* 指定初始容量 */
    public HashMap(int initialCapacity)

    /* 指定初始容量和loadFactor */
    public HashMap(int initialCapacity, float loadFactor);

    /* 从另一个Map赋值构建 */
    public HashMap(Map<? extends K, ? extends V> m)         

```
这里目前只讨论第三个构造方法，第一个和第二个与第三个行为相同，第四个方法与putAll方法相同，之后再说。
###### initialCapacity
表示初始化时的容量，如果存储的元素达到阈值之后会进行扩容，所以如果明确Map中的元素数量，最好在构造时就把容量调好，默认是16。
###### loadFactor与threshold
loadFactor用于描述何时对容器进行扩容和重新Hash的值，默认是0.75。threshold用于表示扩容的阈值，当元素数超过threshold时便进行扩容，loadFactor用于生成threshold和在putAll时使用。第一次生成的threshold=capacity * loadFactor，之后每次扩容 threshold = threshold * 2。



#### 接下来看看构造方法的实现



```
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
        this.threshold = tableSizeFor(initialCapacity);
    }
    
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }

```

tableSizeFor方法会返回比cap大的最小的2的指数。



<h2 id="struct">如何存储数据</h2>



先看看HashMap的数据如何储存。



#### 存储的实现



HashMap的数据存储在Node<K,V>[]中。

``` 
    interface Entry<K,V> {
        K getKey();
        V getValue();
        V setValue(V value);
        boolean equals(Object o);
        int hashCode();
    }
    
    static class Node<K,V> implements Map.Entry<K,V> {
        /* 为hash(key)的值 */
        final int hash;        
        final K key;
        V value;
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        /* 方法的实现比较直白，就省略了。 */

    }
    
    
    transient Node<K,V>[] table;
        
    /* 元素数 */
    transient int size;
        
    /* modify count在遍历的时候会说明 */    
    transient int modCount;
    
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

```



值得注意的是Node<K,V>的属性next,也就是说Node其实是一个链表的节点，用于解决hash冲突的情况，hash相同的Node放在同一个链表里。

#### put



接下来看看如何put。



```
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        
        /* 当table还没有初始化的时候使用resize()，进行内存分配。*/
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        
        /*
            n为tab.length也就是capacity，capacity是2的指数，会在resize的时候说到。
            (n - 1) & hash 把 hash均匀映射到了0至n-1的区间内，当Node[i]为空时直接创建Node。
        */  
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            /* Node[i]上已经存在了Node。 */
            Node<K,V> e; K k;
            /* e用于表示被覆盖的Node，如果节点是key与当前key相等，给e赋值，之后覆盖 */
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                /* 如果Node是TreeNode，即使用红黑树存储数据，调用红黑树的put方法*/
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                /* p.key不与key相等，使用链表的next判断剩下的节点。 */
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        /* 判断如果没有响应的key,把生成的新Node放在链表最后。 */
                        p.next = newNode(hash, key, value, null);
                        
                        /* TREEIFY_THRESHOLD = 8，如果链表中的Node 数大于8，把Node链表转换成由TreeNode组成的红黑树。 */
                        if (binCount >= TREEIFY_THRESHOLD - 1)
                            treeifyBin(tab, hash);
                        break;
                    }
                    /* 找到相同的key，break */
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    /* 循环条件 */
                    p = e;
                }
            }
            
            /* e不为空，覆盖 */
            if (e != null) {
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        /* 如果元素数量超过阈值，则进行扩容 */
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }

```

TreeNode是在1.8中新引入的，为了解决hash冲突造成的查询效率降低。当链表里的元素数超过8之后，会把Node数组table中的节点替换成TreeNode,TreeNode组成了一颗红黑树，可以保证在hash冲突的情况下提供O(logN)的查询。需要注意的是红黑树中需要比较两个TreeNode的key来生成树，key的类型需要实现Comparable，否则会影响查询性能。1.8以后的存储结构图如下：


{% asset_img HashMap_Node_1_8.jpeg 1.8下HashMap的Node示意图%}


#### get


```
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }
    
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash &&
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                /* TreeNode使用getTreeNode方法 */
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }

```
比较易懂，会根据是Node还是TreeNode分别不同的策略来寻找。

<h2 id="resize">如何扩容</h2>

扩容需要调用resize()方法，resize方法会把容量扩容为原来的一倍，把原来的所有Node重新通过(capacity-1) & hash(key) 计算新的位置并赋值，重新构造newTable[]。所以扩容是一个比较消耗性能的行为，要减少扩容的次数。

```
    final Node<K,V>[] resize() {
    
        /* 根据是初始化表的情况还是扩容的情况来计算新的capacity和新的threshold */
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        
        /* 遍历原来的table,生成新的节点放入新的table */
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }

```

<h2 id="iter">如何遍历</h2>
HashMap的遍历主要使用以下三个方法，keySet()，values()，entrySet()。这里只看entrySet，keySet与values的实现与enterySet差别不大。

```

    transient Set<Map.Entry<K,V>> entrySet;

    final class EntrySet extends AbstractSet<Map.Entry<K,V>> {
        public final int size()                 { return size; }
        /* 会清除整个HashMap */
        public final void clear()               { HashMap.this.clear(); }
        public final Iterator<Map.Entry<K,V>> iterator() {
            return new EntryIterator();
        }
        public final boolean contains(Object o) {
            if (!(o instanceof Map.Entry))
                return false;
            Map.Entry<?,?> e = (Map.Entry<?,?>) o;
            Object key = e.getKey();
            Node<K,V> candidate = getNode(hash(key), key);
            return candidate != null && candidate.equals(e);
        }
        /* remove会删除真实的Node */
        public final boolean remove(Object o) {
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>) o;
                Object key = e.getKey();
                Object value = e.getValue();
                return removeNode(hash(key), key, value, true, true) != null;
            }
            return false;
        }
        /* Stream并行操作使用的方法，这里不讨论了 */
        public final Spliterator<Map.Entry<K,V>> spliterator() {
            return new EntrySpliterator<>(HashMap.this, 0, -1, 0, 0);
        }
        public final void forEach(Consumer<? super Map.Entry<K,V>> action) {
            Node<K,V>[] tab;
            if (action == null)
                throw new NullPointerException();
            if (size > 0 && (tab = table) != null) {
                int mc = modCount;
                for (int i = 0; i < tab.length; ++i) {
                    for (Node<K,V> e = tab[i]; e != null; e = e.next)
                        action.accept(e);
                }
                if (modCount != mc)
                    throw new ConcurrentModificationException();
            }
        }
    }
    
    final class EntryIterator extends HashIterator
        implements Iterator<Map.Entry<K,V>> {
        public final Map.Entry<K,V> next() { return nextNode(); }
    }

    abstract class HashIterator {
        Node<K,V> next;        // next entry to return
        Node<K,V> current;     // current entry
        int expectedModCount;  // for fast-fail
        int index;             // current slot

        HashIterator() {
            expectedModCount = modCount;
            Node<K,V>[] t = table;
            current = next = null;
            index = 0;
            if (t != null && size > 0) { // advance to first entry
                do {} while (index < t.length && (next = t[index++]) == null);
            }
        }

        public final boolean hasNext() {
            return next != null;
        }

        /* 按照table[]的顺序遍历，优先遍历Node的next节点。
        final Node<K,V> nextNode() {
            Node<K,V>[] t;
            Node<K,V> e = next;
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            if (e == null)
                throw new NoSuchElementException();
            if ((next = (current = e).next) == null && (t = table) != null) {
                do {} while (index < t.length && (next = t[index++]) == null);
            }
            return e;
        }

        public final void remove() {
            Node<K,V> p = current;
            if (p == null)
                throw new IllegalStateException();
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
            current = null;
            K key = p.key;
            removeNode(hash(key), key, null, false, false);
            expectedModCount = modCount;
        }
    }

```
entrySet返回的Set其实是个视图，所有对entrySet的操作会反映到实际的table上。值得注意的是代码中经常会出现类似
```
    if (modCount != mc)
        throw new ConcurrentModificationException();
    
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
```
这样的代码，当modCount和expectModeCount不相等时会抛出ConcurrentModificationException。

<h2 id="ConcurrentModificationException">ConcurrentModificationException</h2>
ConcurrentModificationException是一个在遍历时没写好就会出现的异常，这个异常描述的是，在遍历过程中，元素发生了添加、删除、覆盖。判断是否抛出ConcurrentModificationException是依据modCount是否等于expectModCount来判断的。首先看看哪些地方会抛出ConcurrentModificationException。

```
    Map<K,V> hashMap = new HashMap<>();
    hashMap.keySet().forEach();
    hashMap.values().forEach();
    hashMap.entrySet().forEach();
    hashMap.forEach();
    hashMap.replaceAll();
    
    hashMap.keySet().iterator().next();
    hashMap.values().iterator().next();
    hashMap.entrySet().iterator().next();
       
    hashMap.keySet().iterator().remove();
    hashMap.values().iterator().remove();
    hashMap.entrySet().iterator().remove();
    
    /* 同样没有考虑并行stream的操作 */
```
其中后六个迭代器的方法其实是从他们的基类HashIterator的继承来的。

接下来看看哪些方法会修改modCount的值。

```
    Map<K,V> hashMap = new HashMap<>();
    hashMap.put();
    hashMap.putAll();
    hashMap.putIfAbsent();
    
    hashMap.remove();
    hashMap.keySet().remove();
    hashMap.entrySet().remove();
    hashMap.compute();
    hashMap.computeIfPresent();
    hashMap.computeIfAbsent();
    
    /* 上面也提到了这些方法，这些方法先判断是否抛出异常，然后删除修改modCount，最后更新expectModCount，所以在单线程下这些方法可以拿来在循环中使用*/
    hashMap.keySet().iterator().remove();
    hashMap.values().iterator().remove();
    hashMap.entrySet().iterator().remove();
    
    hashmap.clear();
    
    /* 同样没有考虑并行stream的操作 */
```
在遍历过程中不要使用上面这些方法，否则会抛出ConcurrentModificationException。下面看看为什么要有ConcurrentModificationException。

```
    Map<K,V> hashMap = new HashMap<>();
    K k = new K();
    /* 正确的遍历删除 */
    Iterator<K> it =map.keySet().iterator();
    while(it.hasNext()){
        K key = it.next();
        if(key==K) {
            it.remove();
        }
    }
    
    /* 正确的遍历删除的简写 */
    map.keySet().removeIf(key -> key == K);
    
    /* 更加函数式风格的写法，这个在多线程环境也是OK的，因为没有修改而是复制后赋值。不可变大法好 */
    map = map.entrySet().stream()
                .filter(entry -> entry.getKey() ==K)
                .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
    
    /* 会抛异常的遍历删除 */
    for(Map.Entry<K,V> entry : map.entrySet()){
            if(entry.getKey()==K){
                map.remove(entry.getKey());
       }
    }
        
```
在单线程环境下看一下，会抛异常的遍历删除，如果去掉ConcurrentModificationException会导致什么问题?
在单线程的情况下，如果不存在hash冲突并不会存在什么问题，此时遍历的话会顺序寻找table[]中的下一个非空的元素。但如果存在hash冲突的话，就会产生各种各样的问题了。

先考虑链表的情况：

```

假设hash为K的Node是链表，链表里有两个Node，分别为N1和N2，目前遍历到N1，即Current=N1，Next=N2，N1.next=N2。

此时删除N2，N1.next=null，此时N2还被Next引用，继续调用迭代器的Next就可以遍历到一个已经被删除的Node。

此时删除N1，实际上并不会产生什么问题，因为已经遍历过N1，

```

然后考虑红黑树的情况：

```
情况就非常多变了，删除节点时红黑树为了保证其性质，会进行翻转，改变节点之间的关系。遍历就很容易出现问题了。
```

在多线程环境下，就会有无数种方法出现ConcurrentModificationException，同时还会存在很多其他问题，说到底HashMap就不是应用于多线程的。
 
<h2 id="unsafe">线程不安全体现在哪里</h2>

HashMap的所有方法都没有加锁，多线程使用且不只读会出现各种各样问题，例如：

- 两个线程同时put，(capacity-1) & hash(key)值相等，同时判断当前数组位置为空，预料加入两个Node结果最后只加进去一个。

- 一个线程put，另一个线程remove，i=(capacity-1) & hash(key)值相等，remove线程决定给table[i]制空后，put添加在table[i]的链表后面。预计table[i]存在一个Node结果一个Node也没有。
 
要解决线程安全问题有几个思路：

- 使用同步容器，Collections.synchronizedMap(map)

- 使用并发容器，ConcurrentHashMap

- 保证只读，修改的时候复制一份，在副本上修改，然后使用副本代替原件，保证不存在中间状态，同时使用volatile。

三种方式各有各的优劣：

- 读场景多，写场景非常少，元素少的时候，第三种方式是可以考虑的选择，因为对比前两个查询不需要额外的操作可以保证效率最高。

- 元素多，写场景不太少且线程不多的情况下，第二种方式非常不错，第一种方法每一个操作都需要加锁，效率不高，第三个方法复制时间太长，可能导致数据更新慢。

- 元素多，写场景不太少且线程很多的情况下，第一种方法才有些优势，第二种方法使用了乐观锁，在大量线程去征用的时候可能会长时间阻塞，第一种方法完全同步，在获取锁的速度上优于第二种。

总的来说，在大多数多线程场景下ConcurrentHashMap都可以提供不错的性能，首选。

<h2 id="effect">如何保证效率</h2>

- 通过hash()映射key到数组，实现O(1)查询

- 通过链表或是红黑树解决hash冲突，实现最差O(k)查询（k为同一数组位置下的元素数）

- 通过扩容减少hash冲突的几率

<h2 id="tips">tips</h2>


- 初始化时尽量设置好容量

- key的类型要实现好hashCode和equals方法

- key的类型要实现好Comparable借口

- 注意循环内的操作，避免ConcurrentModificationException

- 不要在多线程下使用






