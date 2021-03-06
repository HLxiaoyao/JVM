
首先需要了解一下什么是Java虚拟机的内存结构，其实从某一角度来说，Java虚拟机的内存结构就是运行时数据区，在《Java 虚拟机规范》中用的是运行时数据区术语的，并没有内存结构的概念。内存结构只是听着更加贴切，更加形象。

![](/images/JVM架构图.png)

JVM被分为三个主要的子系统：类加载器子系统、运行时数据区和执行引擎。而本文主要介绍其中的运行时数据区（Runtime Data Areas）。在Java虚拟机规范中，定义了五种运行时数据区，分别是Java堆、方法区、虚拟机栈、本地方法区、程序计数器（运行时常量池也会进入方法区，即方法区中就已经包含了常量池）。

## 线程共享：Java堆、方法区
### Java堆
Java堆是所有线程共享的，它在虚拟机启动时就会被创建。Java堆是内存空间占据的最大一块区域了，Java堆是用来存放对象实例及数组，基本上new出来的对象都放在这里，所以这也是垃圾回收器的主要工作区域。并且单个JVM进程有且仅有一个Java堆。根据垃圾回收器的规则，可以对Java堆进行进一步的划分，具体Java堆内存结构如下图所示：

![](/images/JVM堆划分.png)

如上图所示，Java堆并不是单纯的一整块区域，实际上java堆是根据对象存活时间的不同，Java堆还被分为年轻代、老年代两个区域，年轻代还被进一步划分为Eden区、From Survivor 0、To Survivor 1区。并且默认的虚拟机配置比例是Eden：from ：to = 8:1:1。

    （1）Java堆 = 老年代 + 新生代
    （2）新生代 = Eden + S0 + S1
    （3）默认Eden：from ：to = 8:1:1

那么JVM堆内存溢出后，其他线程是否可继续工作呢？发生OOM的线程一般情况下会死亡，也就是会被终结掉，该线程持有的对象占用的heap都会被gc了，释放内存。因为发生OOM之前要进行gc，就算其他线程能够正常工作，也会因为频繁gc产生较大的影响。

### 方法区
以HotSpot虚拟机为例，在JDK1.7的时候，方法区被称作为永久代；从JDK1.8开始，Metaspace（元空间）也就是所谓的方法区。

方法区（Method Area）也是各个线程共享的，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。虽然Java虚拟机规范把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫做Non-Heap（非堆），目的应该是与Java堆区分开来。

Java虚拟机规范中是这样定义方法区的，它存储了每个类的结构信息，例如运行时常量池、字段、方法数据、构造函数和普通方法的字节码内容，还包括一些在类、实例、接口初始化时用到的特殊方法。

#### JDK1.8之前的方法区
以HotSpot虚拟机为例，在JDK1.8之前，方法区也被称作为永久代，这个方法区会发生常见的java.lang.OutOfMemoryError: PermGen space异常，注意是永久代异常信息，可以通过启动参数来控制方法区的大小：

    （1）-XX:PermSize  设置方法区最小空间
    （2）-XX:MaxPermSize  设置方法区最大空间

在JDK7之前的HotSpot虚拟机中，纳入字符串常量池的字符串被存储在永久代中，因此导致了一系列的性能问题和内存溢出错误。特别突出的例子就是String的intern()方法

![](/images/JDK8之前堆空间.png)

#### JDK1.8 之后的方法区
JDK8之后就没有永久代了，变成叫做元空间（meta space），而且将老年代与元空间剥离。元空间放置于本地的内存中，因此元空间的最大空间就是系统的内存空间了，从而不会再出现像永久代的内存溢出错误了，也不会出现泄漏的数据移到交换区这样的事情。用户可以为元空间设置一个可用空间最大值，不设置默认根据类的元数据大小动态增加元空间的容量。对于一个64位的服务器端JVM来说，其默认的–XX:MetaspaceSize值为21MB。也就是说默认的元空间大小是21MB。只要类加载器还存活，其加载的类的元数据也是存活的，不会被回收掉。

![](/images/JDK8堆空间.png)

#### JDK1.8之后的方法区为何变化如此之大？
做这个改变主要是基于以下两点原因：

    （1）由于永久代（PermGen）内存经常会溢出，抛出java.lang.OutOfMemoryError: PermGen，因此JVM的开发者希望这一块内存可以更灵活地被管理，不要再经常出现这样的OOM错误。
    （2）移除永久代（PermGen）可以促进HotSpot JVM与JRockit VM的融合，因为JRockit没有永久代。

但是要注意是，永久代的移除并不代表自定义的类加载器泄露问题就解决了。因此，还必须监控你的内存消耗情况，因为一旦发生泄漏，会占用你的大量本地内存，并且还可能导致交换区交换更加糟糕。

### 常量池
### 运行时常量池
运行时常量池是方法区的一部分。Class 文件中除了有类的版本、字段、方法、接口等描述信息外，还有常量池信息（用于存放编译期生成的各种字面量和符号引用）

既然运行时常量池时方法区的一部分，自然受到方法区内存的限制，当常量池无法再申请到内存时会抛出 OutOfMemoryError 异常。

JDK1.7及之后版本的JVM已经将运行时常量池从方法区中移了出来，在Java堆（Heap）中开辟了一块区域存放运行时常量池。

![](/images/运行时常量池.jpg)

## 线程私有：程序计数器、Java 虚拟机栈、本地方法栈
Java堆以及方法区的数据是共享的，但是有一些部分则是线程私有的。线程私有部分可以分为：程序计数器、Java 虚拟机栈、本地方法栈三大部分。
### Java 虚拟机栈（JVM Stacks）

![](/images/Java虚拟机栈.jpeg)

Java虚拟机的每一条线程都有自己私有的Java虚拟机栈，这个Java虚拟机栈跟线程同时创建，所以它跟线程有相同的生命周期。Java虚拟机栈描述的是Java 方法执行的内存模型：每一个方法在执行的同时都会创建一个栈帧，用于存储局部变量表、操作数栈、动态链接、方法出口等信息，每一个方法从调用直至执行完成的过程，就对应着一个栈帧在Java虚拟机栈中的入栈到出栈的过程。

其中局部变量表存放了编译期可知的各种基本数据类型、对象引用和returnAddress类型。

    （1）基本类型：八种基本类型
    （2）对象引用：reference 类型，它不等同于对象本身，根据不同的虚拟机实现，它可能是一个指向对象起始地址的引用指针，也可能指向一个代表对象的句柄或者其他与此对象相关的位置。
    （3）returnAddress 类型：指向了一条字节码指令的地址。

其中64位长度的long和double类型的数据会占用2个局部变量空间（Slot），其余的数据类型只占用1个。局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这个方法需要在帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小。
Java虚拟机栈既允许被实现成固定的大小，也允许根据计算动态来扩展和收缩，如果采用固定大小的话，每一个线程的Java虚拟机栈容量可以在线程创建的时候独立选定。在Java虚拟机栈中会发生两种异常，这个在虚拟机规范中有指出：

    （1）如果线程请求分配的栈容量超过Java虚拟机栈允许的最大容量，Java虚拟机将会抛出StackOverflowError异常；也就是栈溢出错误。方法递归调用产生StackOverflowError异常这种结果。
    （2）如果Java虚拟机栈可以动态扩展，并且在尝试扩展的时候无法申请到足够的内存或者在创建新的线程时没有足够的内存去创建对应的Java虚拟机栈，那么虚拟机将会抛出 OutOfMemoryError异常。也就是OOM内存溢出错误！(线程启动过多)

### 本地方法栈（Native Method Stacks）
虚拟栈相似，只不过它服务于Native方法，也是线程私有。当Java虚拟机使用其他语言（例如 C 语言）来实现指令集解释器时，也会使用到本地方法栈。如果Java虚拟机不支持natvie方法，并且自己也不依赖传统栈的话，可以无需支持本地方法栈。
与Java虚拟机栈一样，本地方法栈区域也会抛出StackOverflowError和 OutOfMemoryError异常。HotSpot虚拟机直接就把本地方法栈和虚拟机栈合二为一。
### 程序计数器
程序计数器是一块较小的内存空间，可以看作是当前线程所执行的字节码的行号指示器。字节码解释器工作时通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等功能都需要依赖这个计数器来完。

另外，为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各线程之间计数器互不影响，独立存储，我们称这类内存区域为“线程私有”的内存。从上面的介绍中知道程序计数器主要有两个作用：

    （1）字节码解释器通过改变程序计数器来依次读取指令，从而实现代码的流程控制，如：顺序执行、选择、循环、异常处理。
    （2）在多线程的情况下，程序计数器用于记录当前线程执行的位置，从而当线程被切换回来的时候能够知道该线程上次运行到哪儿了。

注意：程序计数器是唯一一个不会出现 OutOfMemoryError 的内存区域，它的生命周期随着线程的创建而创建，随着线程的结束而死亡

## 直接内存

![](/images/直接内存-数据拷贝.jpg)

直接内存并不是虚拟机运行时数据区的一部分，也不是虚拟机规范中定义的内存区域，但是这部分内存也被频繁地使用。而且也可能导致 OutOfMemoryError 异常出现。

JDK1.4 中新加入的 NIO(New Input/Output) 类，引入了一种基于通道（Channel） 与缓存区（Buffer） 的 I/O 方式，它可以直接使用 Native 函数库直接分配堆外内存，然后通过一个存储在 Java 堆中的 DirectByteBuffer 对象作为这块内存的引用进行操作。这样就能在一些场景中显著提高性能，因为避免了在 Java 堆和 Native 堆之间来回复制数据。

本机直接内存的分配不会收到 Java 堆的限制，但是，既然是内存就会受到本机总内存大小以及处理器寻址空间的限制。

## 问题
2、栈内存分配越大越好吗？
解答：不是。栈内存过大意味着java程序中单个线程的内存过大，这样会影响整个java程序的线程并发数。

3、方法内的局部变量是否是线程安全的?
解答: 方法中的局部变量如果被方法返回了，并且返回值是一个引用数据类型，则可能导致线程安全问题。

