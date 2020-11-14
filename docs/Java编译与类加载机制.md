# Java编译与类加载机制
为什么需要类加载机制呢？Java源码经过编译后成为字节码(Byte Code)存储在Class文件中，在Class文件中包含的各类信息都需要加载到虚拟机后才能被运行和使用
## 什么是类加载器
类加载器(ClassLoader)用来加载Java类到Java虚拟机中。一般来说Java虚拟机使用Java类的方式如下：Java源程序(.java 文件)在经过Java编译器编译之后就被转换成Java字节代码(.class 文件)。

    （1）类加载器负责读取Java字节代码，并转换成java.lang.Class类的一个实例。
    （2）每个这样的实例用来表示一个Java类。通过此实例的Class#newInstance(...) 方法，就可以创建出该类的一个对象。
        实际的情况可能更加复杂，比如Java字节代码可能是通过工具动态生成的，也可能是通过网络下载的。
## 虚拟机在何时加载类
关于在什么情况下需要开始类加载的第一个阶段，《Java虚拟机规范》中并没有进行强制约束，留给虚拟机自由发挥。但对于初始化阶段，虚拟机规范则严格规定：当且仅当出现以下六种情况时，必须立即对类进行初始化，而加载、验证、准备自然需要在此之前进行。虚拟机规范中对这六种场景中的行为称为对一个类型进行主动引用。除此之外，所有引用类型的方式都不会触发初始化，称为被动引用。接下来通过多个实例来介绍这六种情况。
### 遇到指定指令时
在程序执行过程中，遇到new、getstatic、putstatic、invokestatic这4条字节码执行时，如果类型没有初始化，则需要先触发其初始化阶段。能够生成这四条指令的典型场景有：

    （1）new：使用new关键字创建对象，肯定会触发该类的初始化
    （2）getstatic与putstatic：当访问某个类或接口的静态变量，或对该静态变量进行赋值时，会触发类的初始化。
        但是注意：static final int a = 2;若是这种情况。
        a不再是一个静态变量，而变成了一个常量，这种情况下访问该变量是不会触发初始化流程。
        因为常量在编译阶段会存入到调用这个常量的方法所在类的常量池中。本质上调用类并没有直接引用定义常量的类，因此并不会触发定义常量的类的初始化。
        即这里已经将常量a=2存入到Demo类的常量池中，这之后Demo类与Bird类已经没有任何关系，甚至可以直接把Bird类生成的class文件删除，Demo仍然可以正常运行。
        
        例如：static final String a = UUID.randomUUID().toString();
        在本例中，常量a的值在编译时不能确定，这种情况下，编译后会产生getstatic指令，同样会触发类的初始化。
        
    （3）invokestatic：调用类的静态方法时，也会触发该类的初始化。

### 反射调用时
使用java.lang.reflect包的方法对类型进行反射调用的时候，如果类型没有进行过初始化，则需要先触发其初始化。

### 初始化子类时
当初始化类的时候，如果发现其父类还没有进行过初始化，则需要先触发其父类的初始化。比如：
```
public class Demo {
    public static void main(String[] args) throws Exception {
        Pigeon.fly();
    }
}
class Bird {
    static {
        System.out.println(“bird init”);
    }
}
class Pigeon extends Bird {
    static {
        System.out.println(“pigeon init”);
    }
    static void fly() {
        System.out.println(“pigeon fly”);
    }
}
```
本例中，在main方法调用Pigeon类的静态方法，最先初始化的是父类Bird，然后才是子类Pigeon。因此在类初始化时，如果发现其父类并未初始化，则会先触发父类的初始化。
```
public class Demo {
    public static void main(String[] args) throws Exception {
        Pigeon.fly();
    }
}
class Bird {
    static {
        System.out.println(“bird init”);
    }
    static void fly() {
        System.out.println(“bird fly”);
    }
}
class Pigeon extends Bird {
    static {
        System.out.println(“pigeon init”);
    }
}

// 执行后输出：
bird init
Bird fly
```
本例中，由于fly方法是定义在父类中，那么方法的拥有者就是父类，因而使用Pigeno.fly()并不是表示对子类的主动引用，而是表示对父类的主动引用，所以只会触发父类的初始化。
### 遇到启动类时
当虚拟机启动时，如果一个类被标记为启动类(即：包含mian方法），虚拟机会先初始化这个主类。比如：
```
public class Demo {
    static {
        System.out.println(“mian init”);
    }
    public static void main(String[] args) throws Exception {
        Bird.fly();
    }
}

class Bird {
    static {
        System.out.println(“bird init”);
    }
    static void fly() {
        System.out.println(“bird fly”);
    }
}

// 执行后输出：
mian init
bird init
bird fly
```
### 接口（带有默认方法）的实现类被初始化时，接口会被加载
当一个接口中定义了JDK8新加入的默认方法(被default关键字修饰的接口方法)时，如果有这个接口的实现类发生了初始化，那该接口要在其之前被初始化。
```
public class Demo {
    public static void main(String[] args) throws Exception {
        Pigeon pigeon = new Pigeon();
    }
}
interface Bird {
    // 如果接口被初始化,那么这句代码一定会执行
    // 那么Intf类的静态代码块一定会被执行
    public static Intf intf = new Intf();
    default void fly() {
        System.out.println(“bird fly”);
    }
}
class Pigeon implements Bird {
    static {
        System.out.println(“pigeon init”);
    }
}
class Intf {
    {
        System.out.println(“interface init”);
    }
}

// 执行后输出：
interface init
pigeon init
```

## 虚拟机如何加载类 - 类的加载过程
JVM把描述类的数据从Class文件加载到内存，并对数据进行校验、解析和初始化，最终形成可以被虚拟机直接使用的Java类型，这个过程被称为虚拟机的类加载机制。一个类型(类或者接口)从被加载到虚拟机内存开始，到卸载出内存为止，它的整个生命周期将会经历如下7个阶段。

![](/images/虚拟机加载类过程.jpeg)

其中验证、准备、解析阶段统称为连接(Linking)。需要注意的是，加载、验证、准备、初始化和卸载这几个阶段的顺序是确定的，而解析阶段则不一定：它在某些情况下可以在初始化完成后再开始，这是为了支持Java语言的动态绑定特性。

动态绑定是指程序在运行期间判断所引用对象的实际类型，然后再确定具体是哪个实例对象的方法。来看一个最简单的例子，下面的代码中，只有在运行时才知道bird对象所引用实际类型是Pigeon。
```
public class Demo {
    public static void main(String[] args) {
        Bird bird = new Pigeon();
        bird.fly();
    }    
}
class Pigeon extends Bird {
    void fly() {
        System.out.println(“pigeon fly”);
    }
}
class Bird {
    void fly() {
        System.out.println(“bird fly”);
    }
}
```
还有一点需要强调：上述的顺序或者说生命周期是相对于一个类来说的，而对于整个应用，这些阶段通常是相互交叉地混合进行的。

### 加载Loading
Loading是整个“类加载”过程的第一个阶段，该阶段主要完成以下工作：

    （1）通过一个类的全限定名来获取其定义的二进制字节流。
    （2）将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构。
    （3）在Java堆中生成一个代表这个类的java.lang.Class对象，作为对方法区中这些数据的访问入口。
    
相对于类加载的其他阶段而言，加载阶段（准确地说，是加载阶段获取类的二进制字节流的动作）是可控性最强的阶段，因为开发人员既可以使用系统提供的类加载器来完成加载，也可以自定义自己的类加载器来完成加载。

加载阶段完成后，虚拟机外部的二进制字节流就按照虚拟机所需的格式存储在方法区之中，而且在Java堆中也创建一个java.lang.Class类的对象，这样便可以通过该对象访问方法区中的这些数据。

### 验证Verification
验证是连接阶段的首个步骤，其目的是确保被加载的类的正确性，即要确保加载的字节流信息要符合《Java虚拟机规范》的全部约束要求，确保这些信息被当做代码运行后不会危害虚拟机自身的安全。验证阶段大致会完成4个阶段的检验动作：

    （1）文件格式验证：验证字节流是否符合Class文件格式的规范。例如是否以 0xCAFEBABE开头、主次版本号是否在当前虚拟机的处理范围之内、常量池中的常量是否有不被支持的类型。
    （2）元数据验证：对字节码描述的信息进行语义分析（注意：对比javac编译阶段的语义分析），以保证其描述的信息符合Java语言规范的要求。例如：这个类是否有父类，除了 java.lang.Object之外。
    （3）字节码验证：通过数据流和控制流分析，确定程序语义是合法的、符合逻辑的。
    （4）符号引用验证：确保解析动作能正确执行。
    
### 准备Preparation
这一阶段主要是为类的静态变量分配内存，并将其初始化为默认值。这里有两点需要注意：

    （1）这时候进行内存分配的仅包括类变量(static)，而不包括实例变量，实例变量会在对象实例化时随着对象一块分配在Java堆中。
    （2）这里所设置的初始值通常情况下是数据类型默认的零值(如0、0L、null、false 等），而不是被在Java代码中被显式地赋予的值。
        假设一个类变量的定义为：public static int value = 3。那么静态变量value在准备阶段过后的初始值为0，而不是3。
        因为这时候尚未开始执行任何Java方法，而把value赋值为3的 public static指令是在程序编译后，存放于类构造器<clinit>()方法之中的，所以把value 赋值为3的动作将在初始化阶段才会执行。
    （3）如果类字段的字段属性表中存在ConstantValue属性，即同时被final和static修饰，那么在准备阶段变量value就会被初始化为ConstValue属性所指定的值。
        假设上面的类变量value被定义为：public static final int value = 3。编译时javac将会为value生成ConstantValue属性。
        在准备阶段虚拟机就会根据ConstantValue的设置将value赋值为3。可以理解为static final常量在编译期就将其结果放入了调用它的类的常量池中。
        
### 解析Resolution
解析阶段是虚拟机将常量池内的符号引用替换为直接引用的过程。解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符 7 类符号引用进行。

    （1）符号引用，就是一组符号来描述目标，可以是任何字面量。
    （2）直接引用，就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。
    
### 初始化Initialization
初始化阶段主要为类的静态变量赋予正确的初始值，JVM负责对类进行初始化，主要对类变量进行初始化。在Java中对类变量进行初始值设定有两种方式：

    （1）声明类变量是指定初始值。
    （2）使用静态代码块为类变量指定初始值。
    
JVM 初始化步骤：

    （1）若这个类还没有被加载和连接，则程序先加载并连接该类。
    （2）若该类的直接父类还没有被初始化，则先初始化其直接父类。
    （3）若类中有初始化语句，则系统依次执行这些初始化语句。

Java编译器在编译过程中，会自动收集类中所有类变量的赋值动作以及静态代码块，将其合并到类构造器<clinit>()方法，编译器收集的顺序是由语句在源文件中出现的顺序决定的。

而初始化阶段就是执行<clinit>()方法的过程。如果两个类存在父子关系，那么在执行子类的<client>()方法之前，会确保父类的<clinit>方法已执行完毕，因此，父类的静态代码块会优先于子类的静态代码块。

这里有一点需要特别强调，JVM会保证一个类的<clinit>()方法在多线程环境中被正确的加锁同步，如果多个线程同时去初始化一个类，那么只会有其中一个线程去执行这个类的<clinit>()方法，其它线程都需要等待，直到<clinit>()方法执行完毕。如果在一个类的<clinit>()方法中有耗时很长的操作，那么可能会造成多个线程阻塞，在实际应用中这种阻塞往往是很隐蔽的。因此，在实际开发过程中，我们都会强调，不要在类的构造方法中加入过多的业务逻辑，甚至是一些非常耗时的操作。
