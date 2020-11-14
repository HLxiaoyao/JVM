# Java引用类型
## 强引用(Strong Reference)
强引用就是最常见的普通对象引用，类似Object obj = new Object()这类的引用，只要还有强引用指向一个对象，垃圾收集器永远不会回收这个对象。对于一个普通的对象，如果没有其他的引用关系，只要超过了引用的作用域或者显式地将相应的(强)引用赋值为null，就是可以被JVM回收的，当然具体回收时机还是要看垃圾收集策略。

因此不使用的对象应手动赋值为null，有利于GC更早回收内存，减少内存占用。强引用具有以下特点：

    （1）强引用可以直接访问目标对象。
    （2）强引用所指向的对象在任何时候都不会被系统回收。JVM宁愿抛出OOM异常，也不会回收强引用所指向的对象。
    （3）强引用可能导致内存泄漏。
    
## 软引用(Soft Reference)
软引用一般用于描述一些有用但非必需的对象，它相对强引用来说，引用的关系更弱一些。当JVM认为内存不足时，才会去尝试回收软引用指向的对象，如果回收以后，还没有足够的内存，才会抛出内存溢出错误。因此，JVM会确保在抛出内存溢出错误之前，回收软引用指向的对象。

﻿软引用通常用来实现内存敏感的缓存，当有足够内存时，保留缓存，反之则清理掉部分缓存，这样在使用缓存的同时，尽量避免耗尽内存。来看一个简单的示例：
```
class SoftRefObject { 
    public void m() { 
        System.out.println("I'm Soft Reference Object"); 
    } 
} 
  
public class Example { 
    public static void main(String[] args) { 
        // 强引用
        SoftRefObject obj = new SoftRefObject();    
        obj.m(); 
        // 创建一个软引用指向SoftRefObject类型的实例对象'obj'
        SoftReference<SoftRefObject> softRef = new SoftReference<SoftRefObject>(obj);   
        // 去掉 SoftRefObject 对象上面的强引用，这时，对象可以被回收
        obj = null;  
        // 返回弱引用指向的对象，对象的可达性状态发生改变
        obj = softRef.get(); 
        if (obj != null) {
            obj.m();
        }
    } 
} 

// 内存足够的情况下，输出结果
I'm Soft Reference Object
I'm Soft Reference Object
```
如上代码，首先创建一个强引用obj指向堆中一个SoftRefObject实例对象，然后创建一个软引用，软引用中的referent指向堆中的SoftRefObject实例，其内存结构示意图如下图所示。

![](/images/软引用示例1.jpeg)

当去掉强引用时，这时候obj指向的对象是可以被内存回收的，当内存充足时，可以通过get()方法，得到堆中的实例对象，其内存示意图如下图所示。

![](/images/软引用示例2.jpeg)

所有引用类型，都是抽象类java.lang.ref.Reference的子类，它提供了get()方法，其部分代码如下所示。其中referent指向具体的实例对象，因此如果referent指向的对象还没有被回收，都可以通过get方法获取原有对象。这意味着利用软引用 (弱引用也类似)，可以将访问到的对象，重新指向强引用(obj=softRef.get())，也就是人为的改变了对象的可达性状态。
```
public abstract class Reference<T> {
    private T referent;
    public T get() {
        return this.referent;
    }
}
```
当软引用对象被JVM标记为可回收状态时，仍然可以通过get方法，让其重新被强引用关联，这时候就需要JVM进行第二次确认，以确保正在使用的对象不会被回收，这也是部分对象真正死亡至少需要经历两次标记的原因。

### 软引用的内存回收策略
软引用的内存结构示意图如下所示。这种情况下，虽然堆中对应的实例对象已经没有强引用指向它，但softRef作为强引用指向referent，而referent则指向SoftRefObject type object，看起来堆中的对象还是被强引用关联着，如下图所示，JVM到底是如何回收这部分内存的呢？

![](/images/软引用内存回收.jpeg)

JVM在进行垃圾回收的时候，首先会遍历引用列表，判断列表中每个软引用中的referent是否存活 (存活的条件是referent指向的对象不为空且被GC Roots可达)，比如前面的例子中，如果obj还指向SoftRefObject的话，则说明SoftRefObject还活着，JVM不会对其作任何处理。而如果已经没有其它对象引用SoftRefObject，就如上图所示，表示该对象已死，JVM会尝试回收该对象。

默认情况下，当内存不足时，如果软引用的存活时间到达一定时长就会被回收，这个时间限制可以通过参数SoftRefLRUPolicyMSPerMB来设置。-XX:SoftRefLRUPolicyMSPerMB可以影响软引用的存活时间，在其他因素不变的情况下，VM参数的值越大，软引用对象存活越久，同样地，如果应用已使用堆内存不变的情况下，设置的堆内存越大，软引用对象也存活的更久。而如果想要在内存不足时回收所有软引用，则把SoftRefLRUPolicyMSPerMB设置为0即可。

### 软引用的应用场景
可以利用软引用来实现缓存，比如一些图片缓存框架中，均大量使用到软引用。由于软引用和弱引用均可以应用在缓存的实现，所以具体的实现原理在介绍弱引用的时候详细说明。此外软引用还有另外一个应用场景：内存熔断。

在分布式系统中也大量运用熔断机制，以实现快速失败，防止服务间调用的雪崩效应。当应用大量使用内存时，容易造成内存溢出错误，甚至程序崩溃，这种情况下可以使用软引用来避免OutOfMemoryError，以实现自我保护的目的。举例说明：
```
// 去掉资源关闭，异常处理等细节
public static List<Map<String, Object>> processResults(ResultSet rs) {
    List<Map<String, Object>> list = Lists.newArrayList(); 
    ResultSetMetaData meta = rs.getMetaData(); 
    int colCount = meta.getColumnCount(); 
    while (rs.next()) { 
        Map<String, Object> map = new HashMap<String, Object>(); 
        // 每行数据放入一个map中
        for (int i = 0; i < colCount; i++) { 
            map.put(meta.getColumnName(i), rs.getObject(i)); 
        } 
        list.add(map); 
    } 
}
```
这是JDBC查询常用的一段代码，但它有一个小的缺陷：如果查询返回一百万行而你没有可用内存来存储它们会发生什么？ 现在使用软引用在完善上面这段代码：
```
// 去掉资源关闭，异常处理等细节
public static List<Map<String, Object>> processResults(ResultSet rs)  { 
    ResultSetMetaData meta = rs.getMetaData(); 
    int colCount = meta.getColumnCount(); 
    // 软引用指向最终返回的List
    SoftReference<List<Map<String, Object>>> ref = new SoftReference<>(new LinkedList<Map<String, Object>>()); 
    while (rs.next()) { 
        Map<String, Object> map = new HashMap<>(); 
        for (int i = 0; i < colCount; i++) { 
            map.put(meta.getColumnName(i), rs.getObject(i)); 
        } 
        // 如果List已经被回收，那么说明内存不足，直接返回自定义的异常通知上层服务
        List<Map<String, Object>> result = ref.get(); 
        if (result == null) { 
            throw new TooManyResultsException(); 
        } else { 
            result.add(map); 
        } 
    } 
    return ref.get(); 
} 
```
在整个过程中，内存分配都集中在两个地方：调用next()和将行数据存储在自己的列表中。调用next()时，ResultSet通常会检索包含多行数据库数据的大块二进制数据，以判断数据是否取完。当需要存储数据时，调用getObject()方法提取数据并包装成Java对象，然后在扔到列表中。

﻿当进行这些操作的时候，如果发现内存不足，GC会回收列表占用的内存，这时候再通过软引用获取列表对象，得到的就是null。当上层捕获到自定义异常时，可以进行相关处理：再次检索或者减少获取数据的行数。需要注意的是，这里的列表使用的是LinkedList而不是ArrayList，这是因为ArrayList在扩容的时候会创建新的数组，占用更多的内存。

但是这仅是一个示例而已，提供另外一个应用场景和解决问题的思路，并不是建议在操作JDBC时要使用软引用。

## 弱引用 (Weak Reference)
弱引用的强度比软引用更弱一些，当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。它一般用于维护一种非强制的映射关系，如果获取的对象还在，就是用它，否则就重新实例化，因此，很多缓存框架均基于它来实现。弱引用的内存结果示意图与软引用类似。
```
﻿Object obj = new Object();
WeakReference<Object> weakObj = new WeakReference<Object>(obj);
//去除强引用
obj = null；
```
### 弱引用的应用场景
弱引用的一个应用场景就是缓存，经常使用的数据结构为WeakHashMap。WeakHashMap与其他Map最主要的区别在于它的Key是弱引用类型，每次GC时WeakHashMap的Key均会被回收，进而Value也会被回收，简单的看下其具体的实现：
```
// 构造方法
public WeakHashMap(int initialCapacity, float loadFactor) {
    table = newTable(capacity);
}
// newTable的实现
private Entry<K,V>[] newTable(int n) {
    return (Entry<K,V>[]) new Entry<?,?>[n];
}
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
    V value;
    final int hash;
    Entry<K,V> next;
    Entry(Object key, V value, ReferenceQueue<Object> queue, int hash, Entry<K,V> next) {
        // 调用WeakReference的构造方法，并传入Map的Key和引用队列
        super(key, queue);
        this.value = value;
        this.hash  = hash;
        this.next  = next;
    }
    // other code here ...
}
```
通过源码可以知道，在构造WeakHashMap的Entry时，会将key关联到一个弱引用上，GC发生时，Map的Key会被清理掉，但Map的Value仍然是强引用，但它随后会在WeakHashMap的expungeStaleEntries()方法中被移除数组(Entry[])，这时Value关联的强应用被干掉，即处于可回收状态。

WeakHashMap中缓存的数据其实活不了多久，特别是GC非常频繁的场景下，没两下，缓存的数据就没了，又得重新加载，那它作为缓存的意义何在？如果单纯作为缓存的话，SoftHashMap(表示软引用的HashMap，实际并不存在这样一个类)貌似更合适一些，毕竟当内存够用时，并不希望缓存被GC掉。但WeakHashMap可应用在更为复杂的场景，比如下面的代码：
```
// 代码来自于：org.apache.tomcat.util.collections.ConcurrentCache.java
public final class ConcurrentCache<K,V> {
    private final int size;
    private final Map<K,V> eden;
    private final Map<K,V> longterm;
    public ConcurrentCache(int size) {
        this.size = size;
        this.eden = new ConcurrentHashMap<>(size);
        this.longterm = new WeakHashMap<>(size);
    }
    // 先从ConcurrentHashMap中取值，取不到就在WeakHashMap中取值
    public V get(K k) {
        V v = this.eden.get(k);
        if (v == null) {
            synchronized (longterm) {
                v = this.longterm.get(k);
            }
            // 如果在WeakHashMap取到值以后，在放入ConcurrentHashMap中
            if (v != null) {
                this.eden.put(k, v);
            }
        }
        return v;
    }
    // 如果ConcurrentHashMap已满，则把所有的数据放到WeakHashMap中，并清空自己
    public void put(K k, V v) {
        if (this.eden.size() >= size) {
            synchronized (longterm) {
                this.longterm.putAll(this.eden);
            }
            this.eden.clear();
        }
        // 如果ConcurrentHashMap未满，直接放入ConcurrentHashMap中
        this.eden.put(k, v);
    }
}
```
这其实是一个热点缓存的实现方案，一段时间以后，经常使用的缓存就会在eden这个Map中，而不常用的缓存就会逐渐被清理。在以前，如果让你来设计热点缓存的实现方案，可能会想很多方案，但不可避免的，会写非常多的代码，但使用 WeakHashMap，就变得非常的简单。WeakHashMap 的另外一个应用场景就是 ThreadLocal。
## 虚引用(Phantom Reference)
虚引用也被称为幽灵引用，它是最弱的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。也就是说，通过其get()方法得到的对象永远是null。

﻿那虚引用到底有什么作用？其实虚引用主要被用来跟踪对象被垃圾回收的状态，当目标对象被回收之前，它的引用会被放入一个ReferenceQueue对象中，通过查看引用队列中是否包含对象所对应的虚引用来判断它是否即将被垃圾回收，从而采取行动。因此在创建虚引用的时候，它必须传入一个ReferenceQueue对象，比如：
```
public static void main(String[] args) {
    ReferenceQueue<String> refQueue = new ReferenceQueue<String>();
    PhantomReference<String> referent = new PhantomReference<String>(new String("CSC"), refQueue);
    // null
    System.out.println(referent.get());
    System.gc();
    System.runFinalization();
    //true
    System.out.println(refQueue.poll() == referent);
}
```
其实SoftReference, WeakReference以及PhantomReference的构造函数都可以接收一个 ReferenceQueue对象。当SoftReference以及WeakReference被清空的同时，也就是JVM准备对它们所指向的对象进行回收时，调用对象的finalize()方法之前，它们自身会被加入到这个 ReferenceQueue对象中，此时可以通过ReferenceQueue的poll()方法取到它们。而 PhantomReference只有当Java垃圾回收器对其所指向的对象真正进行回收时，会将其加入到这个ReferenceQueue对象中，这样就可以追综对象的销毁情况。