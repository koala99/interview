---
title: hashmap详解
date: 2014-08-03 21:30:30
categories: [数据结构,java]
tags: [java,hashmap]
---
hashmap在java中，我们一般面试中会经常问到:</br>
1.什么时候会使用HashMap？他有什么特点？</br>
2.你知道HashMap的工作原理吗？</br>
3.你知道get和put的原理吗？equals()和hashCode()的都有什么作用？</br>
4.如果HashMap的大小超过了负载因子(load factor)定义的容量，怎么办？</br>
 为什么运行下面代码块：
 <pre>
  HashMap<String, Integer> map = new HashMap<String, Integer>();
        map.put("语文", 1);
        map.put("数学", 2);
        map.put("英语", 3);
        map.put("历史", 4);
        map.put("政治", 5);
        map.put("地理", 6);
        map.put("生物", 7);
        map.put("化学", 8);
        for(Map.Entry<String, Integer> entry : map.entrySet()) {
            System.out.println(entry.getKey() + ": " + entry.getValue());
        }
 </pre>
 结果却并非我们想象的那样：
 ![](http://ol76vrjwp.bkt.clouddn.com/aa.png)

这篇文章就带你去解决这些问题。HashMap最新代码：<a herf="http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/8u40-b25/java/util/HashMap.java#HashMap">点击查看 </a> <br/>
首先解决上面的问题，为什么排序不一致呢？看看源代码的描述。
![](http://ol76vrjwp.bkt.clouddn.com/a1.png)
简单翻译最后一句就是：它不确保顺序。不信，你再添加元素看看运行的结果。</br>
hashmap中有两个重要的参数，容量(Capacity)和负载因子(Load factor)，容量就是hashmap的大小，负载因子就是填满的比例。
再来看我们常用的put函数，添加元素，它的具体代码如下：
<pre>
610     public V More ...put(K key, V value) {
611         return putVal(hash(key), key, value, false, true);
612     }
inal V More ...putVal(int hash, K key, V value, boolean onlyIfAbsent,
625                    boolean evict) {
626         Node<K,V>[] tab; Node<K,V> p; int n, i;
627         if ((tab = table) == null || (n = tab.length) == 0)
628             n = (tab = resize()).length;
629         if ((p = tab[i = (n - 1) & hash]) == null)
630             tab[i] = newNode(hash, key, value, null);
631         else {
632             Node<K,V> e; K k;
633             if (p.hash == hash &&
634                 ((k = p.key) == key || (key != null && key.equals(k))))
635                 e = p;
636             else if (p instanceof TreeNode)
637                 e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
638             else {
639                 for (int binCount = 0; ; ++binCount) {
640                     if ((e = p.next) == null) {
641                         p.next = newNode(hash, key, value, null);
642                         if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
643                             treeifyBin(tab, hash);
644                         break;
645                     }
646                     if (e.hash == hash &&
647                         ((k = e.key) == key || (key != null && key.equals(k))))
648                         break;
649                     p = e;
650                 }
651             }
652             if (e != null) { // existing mapping for key
653                 V oldValue = e.value;
654                 if (!onlyIfAbsent || oldValue == null)
655                     e.value = value;
656                 afterNodeAccess(e);
657                 return oldValue;
658             }
659         }
660         ++modCount;
661         if (++size > threshold)
662             resize();
663         afterNodeInsertion(evict);
664         return null;
665     }

</pre>
具体流程，可以总结为：<br/>
对key的hashCode()做hash，然后再计算index;<br/>
如果没碰撞直接放到bucket里；<br/>
如果碰撞了，以链表的形式存在buckets后；<br/>
如果碰撞导致链表过长(大于等于TREEIFY_THRESHOLD)，就把链表转换成红黑树；<br/>
如果节点已经存在就替换old value(保证key的唯一性)<br/>
如果bucket满了(超过load factor*current capacity)，就要resize。<br/>
再看看我们用的get方法：
<pre>
   public V More ...get(Object key) {
555         Node<K,V> e;
556         return (e = getNode(hash(key), key)) == null ? null : e.value;
557     }
566     final Node<K,V> More ...getNode(int hash, Object key) {
567         Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
568         if ((tab = table) != null && (n = tab.length) > 0 &&
569             (first = tab[(n - 1) & hash]) != null) {
570             if (first.hash == hash && // always check first node
571                 ((k = first.key) == key || (key != null && key.equals(k))))
572                 return first;
573             if ((e = first.next) != null) {
574                 if (first instanceof TreeNode)
575                     return ((TreeNode<K,V>)first).getTreeNode(hash, key);
576                 do {
577                     if (e.hash == hash &&
578                         ((k = e.key) == key || (key != null && key.equals(k))))
579                         return e;
580                 } while ((e = e.next) != null);
581             }
582         }
583         return null;
584     }
</pre>
这代码看起来就简单多了，bucket里的第一个节点，直接命中；
如果有冲突，则通过key.equals(k)去查找对应的entry
若为树，则在树中通过key.equals(k)查找，O(logn)；
若为链表，则在链表中通过key.equals(k)查找，O(n)。<br/>
超过阀值后，我们需要进行扩容，这时就需要调用resize方法，代码如下：
<pre>
 final Node<K,V>[] More ...resize() {
677         Node<K,V>[] oldTab = table;
678         int oldCap = (oldTab == null) ? 0 : oldTab.length;
679         int oldThr = threshold;
680         int newCap, newThr = 0;
681         if (oldCap > 0) {
682             if (oldCap >= MAXIMUM_CAPACITY) {
683                 threshold = Integer.MAX_VALUE;
684                 return oldTab;
685             }
686             else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
687                      oldCap >= DEFAULT_INITIAL_CAPACITY)
688                 newThr = oldThr << 1; // double threshold
689         }
690         else if (oldThr > 0) // initial capacity was placed in threshold
691             newCap = oldThr;
692         else {               // zero initial threshold signifies using defaults
693             newCap = DEFAULT_INITIAL_CAPACITY;
694             newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
695         }
696         if (newThr == 0) {
697             float ft = (float)newCap * loadFactor;
698             newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
699                       (int)ft : Integer.MAX_VALUE);
700         }
701         threshold = newThr;
702         @SuppressWarnings({"rawtypes","unchecked"})
703             Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
704         table = newTab;
705         if (oldTab != null) {
706             for (int j = 0; j < oldCap; ++j) {
707                 Node<K,V> e;
708                 if ((e = oldTab[j]) != null) {
709                     oldTab[j] = null;
710                     if (e.next == null)
711                         newTab[e.hash & (newCap - 1)] = e;
712                     else if (e instanceof TreeNode)
713                         ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
714                     else { // preserve order
715                         Node<K,V> loHead = null, loTail = null;
716                         Node<K,V> hiHead = null, hiTail = null;
717                         Node<K,V> next;
718                         do {
719                             next = e.next;
720                             if ((e.hash & oldCap) == 0) {
721                                 if (loTail == null)
722                                     loHead = e;
723                                 else
724                                     loTail.next = e;
725                                 loTail = e;
726                             }
727                             else {
728                                 if (hiTail == null)
729                                     hiHead = e;
730                                 else
731                                     hiTail.next = e;
732                                 hiTail = e;
733                             }
734                         } while ((e = next) != null);
735                         if (loTail != null) {
736                             loTail.next = null;
737                             newTab[j] = loHead;
738                         }
739                         if (hiTail != null) {
740                             hiTail.next = null;
741                             newTab[j + oldCap] = hiHead;
742                         }
743                     }
744                 }
745             }
746         }
747         return newTab;
748     }
</pre>
在resize的过程，简单的说就是把bucket扩充为2倍，之后重新计算index，把节点再放到新的hashmap中。
现在我们就可以回答上面的问题了：<br/>
1. 什么时候会使用HashMap？他有什么特点？<br/>
是基于Map接口的实现，存储键值对时，它可以接收null的键值，是非同步的，HashMap存储着Entry(hash, key, value, next)对象。<br/>
2. 你知道HashMap的工作原理吗？<br/>
通过hash的方法，通过put和get存储和获取对象。存储对象时，我们将K/V传给put方法时，它调用hashCode计算hash从而得到bucket位置，进一步存储，HashMap会根据当前bucket的占用情况自动调整容量(超过Load Facotr则resize为原来的2倍)。获取对象时，我们将K传给get，它调用hashCode计算hash从而得到bucket位置，并进一步调用equals()方法确定键值对。如果发生碰撞的时候，Hashmap通过链表将产生碰撞冲突的元素组织起来，在Java 8中，如果一个bucket中碰撞冲突的元素超过某个限制(默认是8)，则使用红黑树来替换链表，从而提高速度。<br/>
3. 你知道get和put的原理吗？equals()和hashCode()的都有什么作用？<br/>
通过对key的hashCode()进行hashing，并计算下标( n-1 & hash)，从而获得buckets的位置。如果产生碰撞，则利用key.equals()方法去链表或树中去查找对应的节点<br/>
4. 如果HashMap的大小超过了负载因子(load factor)定义的容量，怎么办？<br/>
如果超过了负载因子(默认0.75)，则会重新resize一个原来长度两倍的HashMap，并且重新调用hash方法。


   
    
