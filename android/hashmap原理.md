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


“你用过HashMap吗？” “什么是HashMap？你为什么用到它？”

几乎每个人都会回答“是的”，然后回答HashMap的一些特性，譬如HashMap可以接受null键值和值，而Hashtable则不能；HashMap是非synchronized;HashMap很快；以及HashMap储存的是键值对等等。这显示出你已经用过HashMap，而且对它相当的熟悉。但是面试官来个急转直下，从此刻开始问出一些刁钻的问题，关于HashMap的更多基础的细节。面试官可能会问出下面的问题：

“你知道HashMap的工作原理吗？” “你知道HashMap的get()方法的工作原理吗？”

你也许会回答“我没有详查标准的Java API，你可以看看Java源代码或者Open JDK。”“我可以用Google找到答案。”

但一些面试者可能可以给出答案，“HashMap是基于hashing的原理，我们使用put(key, value)存储对象到HashMap中，使用get(key)从HashMap中获取对象。当我们给put()方法传递键和值时，我们先对键调用hashCode()方法，返回的hashCode用于找到bucket位置来储存Entry对象。”这里关键点在于指出，HashMap是在bucket中储存键对象和值对象，作为Map.Entry。这一点有助于理解获取对象的逻辑。如果你没有意识到这一点，或者错误的认为仅仅只在bucket中存储值的话，你将不会回答如何从HashMap中获取对象的逻辑。这个答案相当的正确，也显示出面试者确实知道hashing以及HashMap的工作原理。但是这仅仅是故事的开始，当面试官加入一些Java程序员每天要碰到的实际场景的时候，错误的答案频现。下个问题可能是关于HashMap中的碰撞探测(collision detection)以及碰撞的解决方法：

“当两个对象的hashcode相同会发生什么？” 从这里开始，真正的困惑开始了，一些面试者会回答因为hashcode相同，所以两个对象是相等的，HashMap将会抛出异常，或者不会存储它们。然后面试官可能会提醒他们有equals()和hashCode()两个方法，并告诉他们两个对象就算hashcode相同，但是它们可能并不相等。一些面试者可能就此放弃，而另外一些还能继续挺进，他们回答“因为hashcode相同，所以它们的bucket位置相同，‘碰撞’会发生。因为HashMap使用链表存储对象，这个Entry(包含有键值对的Map.Entry对象)会存储在链表中。”这个答案非常的合理，虽然有很多种处理碰撞的方法，这种方法是最简单的，也正是HashMap的处理方法。但故事还没有完结，面试官会继续问：

“如果两个键的hashcode相同，你如何获取值对象？” 面试者会回答：当我们调用get()方法，HashMap会使用键对象的hashcode找到bucket位置，然后获取值对象。面试官提醒他如果有两个值对象储存在同一个bucket，他给出答案:将会遍历链表直到找到值对象。面试官会问因为你并没有值对象去比较，你是如何确定确定找到值对象的？除非面试者直到HashMap在链表中存储的是键值对，否则他们不可能回答出这一题。

其中一些记得这个重要知识点的面试者会说，找到bucket位置之后，会调用keys.equals()方法去找到链表中正确的节点，最终找到要找的值对象。完美的答案！

许多情况下，面试者会在这个环节中出错，因为他们混淆了hashCode()和equals()方法。因为在此之前hashCode()屡屡出现，而equals()方法仅仅在获取值对象的时候才出现。一些优秀的开发者会指出使用不可变的、声明作final的对象，并且采用合适的equals()和hashCode()方法的话，将会减少碰撞的发生，提高效率。不可变性使得能够缓存不同键的hashcode，这将提高整个获取对象的速度，使用String，Interger这样的wrapper类作为键是非常好的选择。

如果你认为到这里已经完结了，那么听到下面这个问题的时候，你会大吃一惊。“如果HashMap的大小超过了负载因子(load factor)定义的容量，怎么办？”除非你真正知道HashMap的工作原理，否则你将回答不出这道题。默认的负载因子大小为0.75，也就是说，当一个map填满了75%的bucket时候，和其它集合类(如ArrayList等)一样，将会创建原来HashMap大小的两倍的bucket数组，来重新调整map的大小，并将原来的对象放入新的bucket数组中。这个过程叫作rehashing，因为它调用hash方法找到新的bucket位置。

如果你能够回答这道问题，下面的问题来了：“你了解重新调整HashMap大小存在什么问题吗？”你可能回答不上来，这时面试官会提醒你当多线程的情况下，可能产生条件竞争(race condition)。

当重新调整HashMap大小的时候，确实存在条件竞争，因为如果两个线程都发现HashMap需要重新调整大小了，它们会同时试着调整大小。在调整大小的过程中，存储在链表中的元素的次序会反过来，因为移动到新的bucket位置的时候，HashMap并不会将元素放在链表的尾部，而是放在头部，这是为了避免尾部遍历(tail traversing)。如果条件竞争发生了，那么就死循环了。这个时候，你可以质问面试官，为什么这么奇怪，要在多线程的环境下使用HashMap呢？：）

热心的读者贡献了更多的关于HashMap的问题：

为什么String, Interger这样的wrapper类适合作为键？ String, Interger这样的wrapper类作为HashMap的键是再适合不过了，而且String最为常用。因为String是不可变的，也是final的，而且已经重写了equals()和hashCode()方法了。其他的wrapper类也有这个特点。不可变性是必要的，因为为了要计算hashCode()，就要防止键值改变，如果键值在放入时和获取时返回不同的hashcode的话，那么就不能从HashMap中找到你想要的对象。不可变性还有其他的优点如线程安全。如果你可以仅仅通过将某个field声明成final就能保证hashCode是不变的，那么请这么做吧。因为获取对象的时候要用到equals()和hashCode()方法，那么键对象正确的重写这两个方法是非常重要的。如果两个不相等的对象返回不同的hashcode的话，那么碰撞的几率就会小些，这样就能提高HashMap的性能。
我们可以使用自定义的对象作为键吗？ 这是前一个问题的延伸。当然你可能使用任何对象作为键，只要它遵守了equals()和hashCode()方法的定义规则，并且当对象插入到Map中之后将不会再改变了。如果这个自定义对象时不可变的，那么它已经满足了作为键的条件，因为当它创建之后就已经不能改变了。
我们可以使用CocurrentHashMap来代替Hashtable吗？这是另外一个很热门的面试题，因为ConcurrentHashMap越来越多人用了。我们知道Hashtable是synchronized的，但是ConcurrentHashMap同步性能更好，因为它仅仅根据同步级别对map的一部分进行上锁。ConcurrentHashMap当然可以代替HashTable，但是HashTable提供更强的线程安全性。看看这篇博客查看Hashtable和ConcurrentHashMap的区别。

HashMap与HashTable的哈希算法不同点：

hashtable: hash%h



hashmap: 哈希坐标计算方法是： (n - 1) & hash


   
    
