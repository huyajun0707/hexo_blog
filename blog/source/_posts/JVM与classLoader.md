---
title: JVM与classLoader加载
tags: 
  - android
categories:
  - [android] 
top_image: https://rmt.dogedoge.com/fetch/fluid/storage/bg/dojm2h.png?w=1920&q=100&fmt=webp
excerpt: Java 虚拟机（Java virtual machine，JVM）是运行 Java 程序必不可少的机制。JVM实现了Java语言最重要的特征：即平台无关性。原理：编译后的 Java 程序指令并不直接在硬件系统的 CPU 上执行，而是由 JVM 执行。JVM屏蔽了与具体平台相关的信息，使Java语言编译程序只需要生成在JVM上运行的目标字节码（.class）,就可以在多种平台上不加修改地运行。Java 虚拟机在执行字节码时，把字节码解释成具体平台上的机器指令执行。因此实现java平台无关性。它是 Java 程序能在多平台间进行无缝移植的可靠保证，同时也是 Java 程序的安全检验引擎（还进行安全检查）。
---

# JVM内存分配
Java 虚拟机（Java virtual machine，JVM）是运行 Java 程序必不可少的机制。JVM实现了Java语言最重要的特征：即平台无关性。原理：编译后的 Java 程序指令并不直接在硬件系统的 CPU 上执行，而是由 JVM 执行。JVM屏蔽了与具体平台相关的信息，使Java语言编译程序只需要生成在JVM上运行的目标字节码（.class）,就可以在多种平台上不加修改地运行。Java 虚拟机在执行字节码时，把字节码解释成具体平台上的机器指令执行。因此实现java平台无关性。它是 Java 程序能在多平台间进行无缝移植的可靠保证，同时也是 Java 程序的安全检验引擎（还进行安全检查）。

JVM 是 编译后的 Java 程序（.class文件）和硬件系统之间的接口 （ 编译后：javac 是收录于 JDK 中的 Java 语言编译器。该工具可以将后缀名为. java 的源文件编译为后缀名为. class 的可以运行于 Java 虚拟机的字节码。）

## JVM加载java文件过程
1. 编译器编译生成.class字节码文件
2. 程序中访问这个类时，需要通过classLoader加载到内存中
3. JVM内存划分不同的数据区域
程序计数器、虚拟机栈、本地方法区、堆、方法区（其中方法区和堆是线程共享数据区）

### 程序计数器
当线程挂起时用于记录当前线程代码执行到的位置，方便CPU重启时知道从哪行指令开始。
1.  没有规定OOM
2. 线程私有，随线程创建和死亡
3. 只能记录java方法，记录的是JVM虚拟机字节码指令，如果是native方法，则计数器值为空

### 虚拟机栈
线程私有，与线程生命周期同步，有两种异常情况
StackOverflowError：当线程请求栈深度超出虚拟机栈所允许的深度时抛出

OutOfMemoryError：当虚拟机动态扩展无法申请足够的内存时
JVM是基于栈的解释器执行的，DVM是基于寄存器解释器执行的。

基于栈指的就是虚拟机栈。虚拟机栈是用来描述java方法执行的内存模型，每个方法被执行时，JVM都会在虚拟机栈中创建一个栈桢。
一个线程包含多个栈帧，而每个栈帧内部包含局部变量表、操作数栈、动态连接、返回地址等。

#### 栈桢

[![ttYYy6.md.png](https://s1.ax1x.com/2020/06/02/ttYYy6.md.png)](https://imgchr.com/i/ttYYy6)

- 局部变量表

入参及方法内创建的局部变量都保存在局部变量表中，java在编译成.class时，就会在方法code属性表中max_locals数据项中，确定该方法需要分配的最大局部变量表的容量。
- 操作数栈

后入先出栈（LIFO），和局部变量一样，也在编译时写入方法Code属性表中max_stacks数据中
- 动态链接

一个方法要调用其他方法，需要将这些方法的符号引用转化为其所在内存地址中的直接引用，而符号引用存在于方法区中。
- 返回地址

方法执行完成后退出方法，用来帮助当前方法恢复它的上层方法执行状态

正常退出：

异常退出：

### 本地方法栈

JNI 针对native方法
### 堆
存放对象实例，GC管理的主要区域。

### 方法区
JVM中运行时数据区，主要存储已经被JVM加载的类信息（版本、字段、方法、接口）、常量、静态变量

[![ttYtOK.md.png](https://s1.ax1x.com/2020/06/02/ttYtOK.md.png)](https://imgchr.com/i/ttYtOK)

JVM 的运行时内存结构中一共有两个“栈”和一个“堆”，分别是：Java 虚拟机栈和本地方法栈，以及“GC堆”和方法区。除此之外还有一个程序计数器。

## 堆的扩展
对象在内存中的布局分为三个部分：对象头、实例数据，对齐填充。

在new一个对象时，JVM会在堆中创建一个instanceOopDesc对象。

[![ttYJQx.md.png](https://s1.ax1x.com/2020/06/02/ttYJQx.md.png)](https://imgchr.com/i/ttYJQx)

_mark 是 markOop 类型数据，一般称它为标记字段（Mark Word），其中主要存储了对象的 hashCode、分代年龄、锁标志位，是否偏向锁等。
_metadata保存了类元数据。

#### Monitor
保存在对象头中的一个对象。monitorObject是Monitor的具体实现，这就是为什么object 对象都可以作为锁的原因。


## 类加载的时机：
程序运行过程中，动态加载相应类到内存中。

主动加载(2种情况)
1. 调用类的构造器
2. 调用静态变量和静态方法

## 类加载器
- 启动类加载器 BootstrapClassLoader
- 扩展器加载器 ExtClassLoader (JDK1.9 后改名为PlatformClassLoader)
- 系统加载器 AppClassLoader

1. AppClassLoader

主要加载系统‘java.class.path’配置下类文件，也就是环境变量CLASS_PATH配置的路径。自己编写的代码以及第三方jar包都是由它来加载的。

2. ExtClassLoader

加载系统属性“java.ext.dirs”配置下类文件，JRE/lib/ext

3. BootstrapClassLoader

它是由c/c++语言编写的，加载系统属性“sun.boot.class.path”配置下类文件，全是 JRE 目录下的 jar 包或者 .class 文件
jre/lib/resources.jar

jre/lib/rt.jar

jre/lib/sunrsasign.jar  等等

### 双亲委派模式
1. 判断该Class是否已加载，如果已加载，则直接将该Class返回。
2. 如果该Class没有被加载过，则判断parent是否为空，如果不为空则将加载的任务委托给parent。
3. 如果 parent == null，则直接调用 BootstrapClassLoader 加载该类。
4. 如果 parent 或者 BootstrapClassLoader 都没有加载成功，则调用当前 ClassLoader 的 findClass 方法继续尝试加载。

加载类过程举例

```bash
Test test = new Test();
//1. AppClassLoader将加载的任务委派给它的父类加载器（parent）--ExtClassLoader
//2. ExtClassLoader的parent为null,所以直接将加载任务委派给BotstrapClassLoader
//3. BootstrapClassLoader在jdk/lib目录下找不到Test类，返回null
//4. parent与bootstrapClassLoader都没有成功加载class,AppClassLoader会调用自身的findClass方法来加载。

```


注意：“双亲委派”机制只是Java推荐的机制，并不是强制的机制。我们可以继承java.lang.ClassLoader类，实现自己的类加载器。如果想保持双亲委派模型，就应该重写findClass(name)方法；如果想破坏双亲委派模型，可以重写loadClass(name)方法。
### Android中的classLoader
#### PathClassLoader
用来加载系统apk和被安装到手机中的apk内的dex文件。

```bash
public class PathClassLoader extends BaseDexClassLoader {
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super((String)null, (File)null, (String)null, (ClassLoader)null);
        throw new RuntimeException("Stub!");
    }

    public PathClassLoader(String dexPath, String librarySearchPath, ClassLoader parent) {
        super((String)null, (File)null, (String)null, (ClassLoader)null);
        throw new RuntimeException("Stub!");
    }
}
```
参数说明：

dexPath：dex 文件路径，或者包含 dex 文件的 jar 包路径；

librarySearchPath：C/C++ native 库的路径。

#### DexClassLoader
对比PathClassLoader只能加载已经安装应用的dex或apk文件，DexClassLoader则没有此限制，可以从SD卡上加载包含class.dex的.jar和.apk文件，这是插件化和热修复的基础。

```bash
public class DexClassLoader extends BaseDexClassLoader {
    public DexClassLoader(String dexPath, String optimizedDirectory, String librarySearchPath, ClassLoader parent) {
        super((String)null, (File)null, (String)null, (ClassLoader)null);
        throw new RuntimeException("Stub!");
    }
}
```
参数说明：

dexPath：包含 class.dex 的 apk、jar 文件路径 ，多个路径用文件分隔符（默认是“:”）分隔。

optimizedDirectory：用来缓存优化的 dex 文件的路径，即从 apk 或 jar 文件中提取出来的 dex 文件。该路径不可以为空，且应该是应用私有的，有读写权限的路径。

总结：
- ClassLoader 就是用来加载 class 文件的，不管是 jar 中还是 dex 中的 class。
- Java 中的 ClassLoader 通过双亲委托来加载各自指定路径下的 class 文件。
- 可以自定义 ClassLoader，一般覆盖 findClass() 方法，不建议重写 loadClass 方法。
- Android 中常用的两种 ClassLoader 分别为：PathClassLoader 和 DexClassLoader。


# class初始化过程
JVM加载class到内存中主要包括三大步骤

[![ttYGS1.md.png](https://s1.ax1x.com/2020/06/02/ttYGS1.md.png)](https://imgchr.com/i/ttYGS1)

### 装载
1. classLoader加载，通过包名+类名查找.class文件，生成二进制流（格式，可以是jar、zip、网络字节流）
2. 把.class文件各个部分解析为JVM内部数据结构，并存储在方法区。
3. 在内存中创建一个java.lang.Class类型的对象，用于对该类的访问（提供给外界的访问接口）。

#### 加载时机
* 隐式加载：在程序运行中，遇到new生成对象时，系统会调用ClassLoader去装裁对应的class到内存中

* 显式加载：主动调用Class.forName()

### 链接
包括三步：验证、准备、解析

**验证：** 验证格式，确保.class文件字节流包含的信息符合当前虚拟机的要求。
1. 文件格式检验，保证当前虚拟机版本可处理
2. 元数据检验：保证元数据符合java语言规范
3. 字节码检验：确定程序语义是合法，符合逻辑的
4. 符号引用检验：对类自身以外（常量池中的各种符号引用）的信息进行匹配性校验
5. 
注意：即例没有java原文件，还是可以对.class字节码进行篡改，因此项目经常使用混淆、加固，来保证代码的安全性。

**准备：** 是链接的第2步，主要目的是为类中的静态变量分配内存，并为其设置‘0’值。

```bash
public static int value = 100;//准备阶段会赋值为0   初始化阶段会设置为100
public static final int value = 100;//静态常量 在准备阶段就会为其分配内存，设置为100
```
Java 中基本类型的默认”0值“如下：

基本类型（int、long、short、char、byte、boolean、float、double）的默认值为 0；

引用类型默认值是 null；

**解析：** 是链接的最后一步，是把常量池中的符号引用转换为直接引用，也就是具体的内存地址。JVM会把常量池中的类、接口名、字段名、方法名等转换为具体的内存地址。
### 初始化
这一阶段会执行类构造器<cinit>方法的过程，真正初始化类变量。

```bash
public static int value = 100;//在此阶段会被真正设置为100
```
#### 初始化的时机（主动引用）
1. 虚拟机启动时，初始化包含main方法的主类；
2. 遇到new指令时，如果目标对象类没有被初始化则进行初始化操作；
3. 当遇到访问静态方法和静态字段时，如果目标对象没有被初始化则进行初始化操作；
4. 子类的初始化过程如发现父类还没进行初始化，则对父类进行初始化；
5. 使用反射API进行反射调用时。。。。
6. 第一次调用java.lang.invoke.MethodHandle实例时

对于静态字段，只有直接定义这个字段的类才会被初始化，因此通过子类Child来引用父类Parent中定义的静态字段，只会触发父类Parent的初始化而不会触发子类Child的初始化。至于是否要触发子类的加载和验证，在虚拟机规范中并未明确规定。
#### class初始化和对象的创建顺序
静态变量/静态代码块 -> 普通代码块 -> 构造函数

1. 父类静态变量和静态代码块；
2. 子类静态变量和静态代码块；
3. 父类普通成员变量和普通代码块；
4. 父类的构造函数；
5. 子类普通成员变量和普通代码块；
6. 子类的构造函数。
