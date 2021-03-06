# jvm内存结构

## 1. 程序计数器

* 作用：记录下一条jvm指令执行地址
* 特点：
  * 线程私有
  * 不存在内存溢出

## 2. 虚拟机栈

* 定义：

  * 每个线程运行时所需要的内存，称为**虚拟机栈**
  * 每个栈由多个**栈帧（Frame）**组成，对应着每次方法调用时所占用的内存，**每个栈帧存储局部变量表、操作数栈、常量池引用等信息**
  * 每个线程**只能有个一个活动栈帧**，对应着当前正在执行的那个方法

* 注意：

  * 垃圾回收不会涉及栈内存，因为栈线程私有，每个栈帧的空间会在弹栈后被回收
  * 方法的局部变量是否安全？
    * 如果方法内变量没有逃离方法的作用范围，他是线程安全的
    * 如果逃离了，则是不安全的

* 栈内存溢出

  * 当线程请求的栈深度超过最大值，会抛出 StackOverflowError 异常；
  * 栈进行动态扩展时如果无法申请到足够内存，会抛出 OutOfMemoryError 异常

* 排查问题：

  先使用top、ps命令，查看出现问题的进程和线程ID，然后根据PID和TID，使用jstack命令查看指定PID的具体情况



## 3. Heap 堆

new 创建对象存放的区域，也是垃圾回收的主要区域

* 特点：

  * 线程共享的，堆中的对象是要考虑线程安全问题
  * 有垃圾回收机制 

* 堆内存溢出：

  *  OutOfMemoryError 异常，当堆内存中空间不足，而且无法再增加时，就会抛出该异常。

* 堆内存诊断

  * jps 工具

    查看当前系统中有哪些Java进程

  * jmap工具

    查看堆内存中占用情况 `jmap -heap 进程id`

  * jconsole 工具

    可以连续监测

  

## 4. 方法区

* 定义：https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html

  用于存放已被加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。

  和堆一样不需要连续的内存，并且可以动态扩展，动态扩展失败一样会抛出 OutOfMemoryError 异常。

  对这块区域进行垃圾回收的主要目标是对常量池的回收和对类的卸载，但是一般比较难实现。

  HotSpot 虚拟机把它当成永久代来进行垃圾回收。但很难确定永久代的大小，因为它受到很多因素影响，并且每次 Full GC 之后永久代的大小都会改变，所以经常会抛出 OutOfMemoryError 异常。为了更容易管理方法区，从 JDK 1.8 开始，移除永久代，并把方法区移至元空间，它位于本地内存中，而不是虚拟机内存中。

  方法区是一个 JVM 规范，永久代与元空间都是其一种实现方式。在 JDK 1.8 之后，原来永久代的数据被分到了堆和元空间中。元空间存储类的元信息，静态变量和常量池等放入堆中。

* ![image-20200917141916693](C:/Users/公维信/AppData/Roaming/Typora/typora-user-images/image-20200917141916693.png)

### 运行时常量池

* 常量池：一张表，虚拟机指令根据这张常量表找到执行的类名、方法名、参数类型、字面量（字面量是字符串、数字类型、布尔类型之类的）
* 运行时常量池，常量池是 *.class 文件中的，当该类被加载，他的常量池信息就会放入到运行时常量池中，并把里面的符号变为真实地址

运行时常量池是在jvm里面的，而常量池是在class文件中的，常量池中的常量会在加载到jvm的运行时常量池中。



### StringTable（字符串常量池，在Heap中）特性

* 常量池中的字符串仅仅是一个符号，第一次用到时才变为对象
* 利用字符串常量池的机制，来避免重复创建字符串对象
* 字符串变量拼接的原理是StringBuilder
* 字符串常量拼接的原理是编译期优化
* 可以使用intern方法，主动将字符串池中还没有的字符串对象放入字符串常量池

自己总结的：

* ``String str1 = new String("abc");``

  这一句话，字符串abc首先从常量池中添加到字符串常量池中。（如果字符串常量池中没有该字符串，进行添加，有的话，就不用添加）然后将字符串常量池中的该字符串生成在堆内存中，创建一个新的字符串对象abc。

  总之，如果是创建新的字符串对象，字符串常量池中会先于堆创建一个字符串常量，然后字符串对象会根据字符串常量创建新的对象。

* `String str2 = "abc;"`

  这句话，引用的是字符串常量池中的字符串常量abc，如果字符串常量池中没有该字符串，则创建一个常量存入字符串常量池中。

  

* ```java
  String str = "abcabc";
  
  String str1 = new String("abc");
  String str2 = new String("abc");
  
  String str3 = "abc";
  String str4 = "abc";
  
  String s1 = "abc" + "abc";
  String s2 = str1 + st2;
  String s3 = str3 + str4;
  String s4 = str1 + "abc";
  
  sout(str1 == str2); // false
  sout(str3 == str4); // true
  sout(str == s1); // true
  sout(str == s2); // false
  sout(str == s3); // false
  sout(str == s4); // false
  ```

  1. str是一个对字符串常量池中的字符串常量abcabc的引用，str1、str2 均为堆中的String对象，str3、str4均为对字符串常量池中的字符串常量abc的引用

  2. 对于str1、str2 和 str3、str4之间的==比较，没有很大问题，就是堆中对象和字符串常量的引用的问题

  3. 重点是字符串相拼接得到的结果的比较：为什么前两个比较都是false，而后一个比较就是true？

     前两个，s1、s2都是引用变量进行拼接`s1 = str1 + str2` 、`s2 = str3 + str4`，**不论引用变量是引用的堆中的创建的String对象，还是字符串常量池中的字符串常量，都会统一使用StringBuilder进行拼接**。<font color=red>所以：字符串变量拼接的原理是StringBuilder</font>

     后一个，s3是由字符串常量池中的字符串常量进行拼接，编译器会进行优化，直接将拼接得到的字符串存入到字符串常量池中，然后引用的是该字符串常量。<font color=red>字符串常量拼接的原理是编译期优化</font>

  4.  `String s4 = str1 + "abc;"` 

     其实和两个都是引用变量的是一样的。都是使用的StringBuilder进行拼接，尽管有一个字符串常量，但是更主要的是有一个字符串对象引用，所以会创建一个新的字符串对象s4。

* `intern()`方法：

  对一个String对象调用intern方法，尝试将该字符串放入字符串常量池，如果字符串常量池中有该字符串常量，不会放入，如果没有，放入，并且将原来字符串对象的引用由转向字符串常量（或者说是堆中的字符串对象入池）。返回值就是字符串常量池中的该字符串常量引用。

  **如果字符串常量池中有该字符串常量：**

  ```java
  String str = "ab"; // 字符串常量池中有字符串常量 ab
  String s1 = new String("a") + new String("b"); // 对引用字符串进行拼接，得到的是ab的字符串对象
  String s2 = s1.intern(); // 对s1调用intern方法，返回值为字符串常量池中的ab引用，但是由于字符串常量池中已经有该字符串常量了，s1引用的还是堆中的字符串对象
  sout(s1 == str); // false
  sout(s2 == str); // true
  ```

  **如果字符串常量池中没有该字符串常量：**

  ```java
  String s1 = new String("a") + new String("b"); // 对引用字符串进行拼接，得到的是ab的字符串对象
  String s2 = s1.intern(); // 对s1调用intern方法，由于字符串常量池中没有该字符串常量，字符串ab放入字符串常量池中，返回值为ab字符串常量，并且s1也引用字符串常量ab
  String str = "ab";
  
  System.out.println(s1 == str); // true
  System.out.println(s2 == str); // true
  ```

  **与1.6的区别：**

  * 1.8的intern方法：如果字符串常量池中没有，将字符串放入常量池，并且将返回值为字符串常量，原来堆中的字符串对象引用也指向字符串常量

  * 1.6中intern方法：如果字符串常量池中没有，将字符串放入常量池，并且将返回值为字符串常量，原来堆中的字符串对象引用**不会**指向字符串常量，还是原来的字符串对象

  

**StringTable的性能调优：**

1. 由于StringTable本质上是一个HashTable，对于一个HashTable，Bucket数量设定很重要，当Bucket设置足够大，就能够降低哈希冲突，从而提高查找的性能
2. 由于StringTable 的存在，使用字符串的入池操作（intern方法），可以将重复的字符串进行去重，如果有多个相同字符串，同时放入堆中，是不会进行去重的，但是通过将字符串存入字符串常量池，则能达到去重的作用



## 5. 直接内存

**定义：**

* 属于操作系统内存，常见于NIO操作，常见于数据缓冲区
* 分配回收成本高，读写性能高
* 不受jvm内存回收管理

**直接内存的分配和回收：**

* 使用了Unsafe对象完成直接内存的分配回收，并且回收需要主动调用freeMemory方法
* ByteBuffer 的实现类内部，使用了Cleaner（虚引用）来监测ByteBuffer对象，一旦ByteBuffer对象被垃圾回收，那么就会由ReferenceHandler线程通过 Cleaner的clean方法调用freeMemory来释放直接内存