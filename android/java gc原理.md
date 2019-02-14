---
title: java gc原理
date: 2014-10-13 20:24:30
categories: [数据结构,java]
tags: [java,垃圾回收]
---
   作为android开发人员，了解java的GC工作原理很有必要。我们开发的app需要不断的优化，这就包括如何优化GC的性能、如何与GC进行有限的交互。</br>
   这边就从一个简单的demo展开。在idea 运行中配置如下代码:
   <pre>
   -server -Xms10m -Xmx10m -XX:+DoEscapeAnalysis -XX:+PrintGc
   </pre>
   这边先介绍几个概念：<br/>
   　-Xms 10m，表示JVM Heap(堆内存)最小尺寸10MB，最开始只有 -Xms 的参数，表示 `初始` memory size(m表示memory，s表示size)，属于初始分配10m，-Xms表示的 `初始` 内存也有一个 `最小` 内存的概念（其实常用的做法中初始内存采用的也就是最小内存）。</br>

　　-Xmx 10m，表示JVM Heap(堆内存)最大允许的尺寸10MB，按需分配。如果 -Xmx 不指定或者指定偏小，也许出现java.lang.OutOfMemory错误，此错误来自JVM不是Throwable的，无法用try...catch捕捉。<br/>
　　DoEscapeAnalysis 逃逸模式，前面+代表启用，-号代表关闭<br/>
　　PrintGc 打印gc信息<br/>
   测试代码如下：
   <pre>
   public class Test {
    /**
     * 内分配了两个字节的内存空间
     */
    public static void alloc(){
        int[] array = new int[2];
        array[0] = 1;
    }
    public static void main(String[] args) {
        long startTime = System.currentTimeMillis();
        // 分配 100000000 个 alloc 分配的内存空间
        for(int i = 0; i < 100000000; i++){
            alloc();
        }
        long endTime = System.currentTimeMillis();
        System.out.println(endTime - startTime);
    }
}
   </pre>
   
  可以看到跟踪的gc信息，一连串如下：
  <pre>
  [GC (Allocation Failure)  2472K->424K(9728K), 0.0030868 secs]
  </pre>
   看起来还不够详细，可以继续配置如下命令 -XX:+PrintGCDetails，运行可以看到如下结果：
   <pre>
   [GC (Allocation Failure) [PSYoungGen: 2048K->0K(2560K)] 2488K->440K(9728K), 0.0003446 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
1832
Heap
 PSYoungGen      total 2560K, used 1515K [0x00000007bfd00000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 2048K, 73% used [0x00000007bfd00000,0x00000007bfe7ad80,0x00000007bff00000)
  from space 512K, 0% used [0x00000007bff80000,0x00000007bff80000,0x00000007c0000000)
  to   space 512K, 0% used [0x00000007bff00000,0x00000007bff00000,0x00000007bff80000)
 ParOldGen       total 7168K, used 440K [0x00000007bf600000, 0x00000007bfd00000, 0x00000007bfd00000)
  object space 7168K, 6% used [0x00000007bf600000,0x00000007bf66e050,0x00000007bfd00000)
 Metaspace       used 2991K, capacity 4494K, committed 4864K, reserved 1056768K
  class space    used 327K, capacity 386K, committed 512K, reserved 1048576K
   </pre>
   看最后的heap总结，这里就可以看到划分去了几个generation，PSYoungGen  ParOldGen Metaspace，Metaspace，在jdk1.8中， Permanent Generation已经取代了Metaspace， young gen中就有我们熟悉的伊甸区 eden space，关于各区的介绍，这里详细的概述下：
   JVM在程序运行过程当中，会创建大量的对象，这些对象，大部分是短周期的对象，小部分是长周期的对象，对于短周期的对象，需要频繁地进行垃圾回收以保证无用对象尽早被释放掉，对于长周期对象，则不需要频率垃圾回收以确保无谓地垃圾扫描检测。为解决这种矛盾，Sun JVM的内存管理采用分代的策略。<br/>

1）年轻代(Young Gen)：年轻代主要存放新创建的对象，内存大小相对会比较小，垃圾回收会比较频繁。年轻代分成1个Eden Space和2个Suvivor Space（命名为A和B）。当对象在堆创建时，将进入年轻代的Eden Space。垃圾回收器进行垃圾回收时，扫描Eden Space和A Suvivor Space，如果对象仍然存活，则复制到B Suvivor Space，如果B Suvivor Space已经满，则复制到Old Gen。同时，在扫描Suvivor Space时，如果对象已经经过了几次的扫描仍然存活，JVM认为其为一个持久化对象，则将其移到Old Gen。扫描完毕后，JVM将Eden Space和A Suvivor Space清空，然后交换A和B的角色（即下次垃圾回收时会扫描Eden Space和BSuvivor Space。这么做主要是为了减少内存碎片的产生。<br/>

我们可以看到：Young Gen垃圾回收时，采用将存活对象复制到到空的Suvivor Space的方式来确保尽量不存在内存碎片，采用空间换时间的方式来加速内存中不再被持有的对象尽快能够得到回收。
2）年老代(Tenured Gen)：年老代主要存放JVM认为生命周期比较长的对象（经过几次的Young Gen的垃圾回收后仍然存在），内存大小相对会比较大，垃圾回收也相对没有那么频繁（譬如可能几个小时一次）。年老代主要采用压缩的方式来避免内存碎片（将存活对象移动到内存片的一边，也就是内存整理）。当然，有些垃圾回收器（譬如CMS垃圾回收器）出于效率的原因，可能会不进行压缩。<br/>
3）持久代(Perm Gen)：持久代主要存放类定义、字节码和常量等很少会变更的信息。<br/>
 了解了这些，其实我们也该清楚一些优化的方法，为什么常量不宜太多，年轻区的内存空间应该大，太小了，操作频繁的数据都会进入年老区，垃圾回收不频繁。</br>
 言归正传，继续谈我们的GC，首先讲下我们的垃圾回收算法，古老的引用计数法。它的基本思想很简单，对于一个对象A，只要有任何一个对象引用了A，则A的引用计数器就加1，当引用失效时，引用计数器就减1。只要对象A的引用计数器的值为0，则对象A就不可能再被使用。就可以回收了。它的缺点也很明显，1，每次引用和去引用都需要加减，性能不行；2循环引用问题，可以看下简单代码。
 <pre>
 class A{
  public B b;
   
}
class B{
  public A a;
}
public class Main{
    public static void main(String[] args){
    A a = new A();
    B b = new B();
    a.b=b;
    b.a=a;
    }
}
 </pre>
 这种情况下，谁都没法回收。java的垃圾回收使用的基本的算法思想是标记-清除算法：<br/>
　　标记-清除算法是现代垃圾回收算法的思想基础。将垃圾回收分为两个阶段：标记阶段和清除阶段。</br>
　　一种可行的实现是，在标记阶段，首先通过根节点，标记所有从根节点开始的可达对象（从GC ROOT开始标记引用链——又叫可达性算法）。因此，未被标记的对象就是未被引用的垃圾对象。然后，在清除阶段，清除所有未被标记的对象。这样就不怕循环问题了。</br>
　　具体效果图如下：<br/>
　　![](http://ol76vrjwp.bkt.clouddn.com/lll.png)
　　
　　这种算法的缺点就是容易出现内存碎片。利用率不高。要知道，现代的Java虚拟机都是使用的分代回收的设计，比如在标记-清除算法的基础上做了一些优化的——标记-压缩算法，适合用于存活对象较多的场合，如老年代。<br/>
　　和标记-清除算法一样，标记-压缩算法也首先需要从根节点开始，对所有可达对象做一次标记。但之后，它并不简单的清理未标记的对象，而是将所有的存活对象压缩到内存的一端。之后，清理边界外所有的空间。有效解决内存碎片问题。
　　　![](http://ol76vrjwp.bkt.clouddn.com/ll1.png)
　　　
　　和标记-清除算法相比，复制算法是一种相对高效的回收方法，但是 不适用于存活对象较多的场合如老年代，使用在新生代， 原理是 将原有的内存空间分为两块，两块空间完全相同，每次只用一块，在垃圾回收时，将正在使用的内存中的存活对象复制到未使用的内存块中，之后，清除正在使用的内存块中的所有对象，交换两个内存的角色，完成垃圾回收。同样也没有内存 碎片产生。<br/> 
　　  复制算法的缺点是内存的浪费，因为每次只是使用了一般的空间， 而大多数存活对象都在老年代，故复制算法不用在老年代，老年代是Java堆的空间的担保地区。复制算法主要用在新生代。在垃圾回收的时候，大对象直接从新生代进入了老年代存放，大对象一般不使用复制算法，因为一是太大，复制效率低，二是过多的大对象，会使得小对象复制的时候无地方存放。还有被长期引用的对象也放在了老年代。 Java的垃圾回收机制使用的是分代的思想。 依据对象的存活周期进行分类，短命对象归为新生代，长命对象归为老年代。根据不同代的特点，选取合适的收集算法。少量对象存活（新生代，朝生夕死的特性），适合复制算法，大量对象存活（老年代，生命周期很长，甚至和应用程序存放时间一样），适合标记清理或者标记压缩算法。　
　　  
　
　　
    
    
