> 转载：[class文件常量池、运行时常量池、字符串常量池](https://focusss.github.io/2018/10/13/class%E6%96%87%E4%BB%B6%E5%B8%B8%E9%87%8F%E6%B1%A0%E3%80%81%E8%BF%90%E8%A1%8C%E6%97%B6%E5%B8%B8%E9%87%8F%E6%B1%A0%E3%80%81%E5%AD%97%E7%AC%A6%E4%B8%B2%E5%B8%B8%E9%87%8F%E6%B1%A0/)

## 1. 简介

常量池是为了避免频繁的创建和销毁对象而影响系统性能，其实现了对象的共享。例如字符串常量池，在编译阶段就把所有的字符串文字放到一个常量池中。

> 节省内存空间：常量池中所有相同的字符串常量被合并，只占用一个空间。
> 节省运行时间：比较字符串时，==比equals()快。对于两个引用变量，只用==判断引用是否相等，也就可以判断实际值是否相等。

## 2. class文件常量池（静态常量池）

class文件常量池是指编译生成的class字节码文件结构中的一个常量池（Constant Pool Table），用于存放编译期间生成的各种字面量和符号引用，这部分内容将在类加载后，存放于方法区的运行时常量池。

> - 字面量指的是**字符串字面量和声明为final的常量值（基本数据类型）**。
> - 字符串字面量除了类中所有双引号括起来的字符串（包括方法体内的），还包括所有用到的类名、方法名和这些类与方法的字符串描述、字段（成员变量）的名称和描述符
> - 声明为final的常量值指的是成员变量，不包含本地变量，本地变量是属于方法的。
> - 符号引用包括类和接口的全限定名（包括包路径的完整名）、字段的名称和描述符、方法的名称和描述。只不过是以一组符号来描述所引用的目标，和内存并无关，所以称为符号引用，直接指向内存中某一地址的引用称为直接引用

### 2.1 常量池结构

前端的两个字节占有的位置叫做常量池计数器(constant_pool_count)，它记录着常量池的组成元素-常量池项(cp_info)的个数（从１开始，将０表示不引用任何常量）。紧接着会排列着constant_pool_count-1个常量池项(cp_info)。每个常量池项(cp_info) 都会对应记录着class文件中的某中类型的字面量。cp_info项的结构如下：

```java
cp_info{
	u1 tag;
	u1 info[];
}
```

JVM会根据tag的值来确定当前常量池项表示什么类型的字面量。

| Tag  |                             类型 |          描述           |
| ---- | -------------------------------: | :---------------------: |
| 1    |               CONSTANT_utf8_info | UTF-8编码的字符串字面量 |
| 3    |            CONSTANT_Integer_info |       整型字面量        |
| 4    |              CONSTANT_Float_info |      浮点型字面量       |
| 5    |               CONSTANT_Long_info |      长整型字面量       |
| 6    |             CONSTANT_Double_info |   双精度浮点型字面量    |
| 7    |              CONSTANT_Class_info |   类或接口的符号引用    |
| 8    |             CONSTANT_String_info |    字符串类型字面量     |
| 9    |           CONSTANT_Fieldref_info |     字段的符号引用      |
| 10   |          CONSTANT_Methodref_info |   类中方法的符号引用    |
| 11   | CONSTANT_InterfaceMethodref_info |  接口中方法的符号引用   |
| 12   |        CONSTANT_NameAndType_info |  字段或方法的符号引用   |
| 15   |       CONSTANT_MethodHandle_info |      表示方法句柄       |
| 16   |         CONSTANT_MothodType_info |      表示方法类型       |
| 18   |      CONSTANT_InvokeDynamic_info | 表示一个动态方法调用点  |

### 2.2 基本数据类型常量在常量池存储

通过下面的例子来讲解下：

```java
public class Test {  
      
    private final int a = 10;  
    private final int b = 10;  
    private float c = 11f;  
    private float d = 11f;  
    private float e = 11f;  

    private String s1 = "JVM原理";  
    private String s2 = "JVM原理";  
      
}
```

创建完上面的java文件后，执行`javac Test.java`Test.class文件，再执行`javap -v Test` 查看class文件。

![2020-09-10-q2LCrl](https://image.ldbmcs.com/2020-09-10-q2LCrl.jpg)

从上面的结果我们可以看到在常量池中，只有一个常量10 、一个常量11f、一个常量JVM原理。代码中所有用到 int 类型 10 的地方，会使用指向常量池的指针值#16（符号引用）定位到第#16个常量池项(cp_info)，即值为 10的结构体CONSTANT_Integer_info，而用到float类型的11f时，也会指向常量池的指针值#4来定位到第#4个常量池项(cp_info) 即值为11f的结构体CONSTANT_Float_info。我们可以看到CONSTANT_String_info结构体位于常量池的第#8个索引位置，且该结构体直接存储了指针值#37，用来指向第#37个常量池项。而存放”JVM原理”字符串的UTF-8编码格式的字节数组被放到CONSTANT_Utf8_info结构体中，该结构体位于常量池的第#37个索引位置。

### 2.3 类文件中定义的类名和类中使用到的类常量在常量池存储

CONSTANT_Class_info结构体中，存在一个索引值，用于指向存储了[类二进制形式的完全限定名称]字符串的CONSTANT_String_info结构体。该索引值占两个字节大小，所以它能表示的最大索引是65535（2的16次方-1），也就是说常量池中最多能容纳65535个常量项。所在在定义类时要注意类的大小。

> 假设我们定义了一个 ClassTest的类，并把它放到com.focus.jvm 包下，则 ClassTest类的完全限定名为com.focus.jvm.ClassTest，将JVM编译器将类编译成class文件后，此完全限定名在class文件中，是以二进制形式的完全限定名存储的，即它会把完全限定符的”.”换成”/“ ，即在class文件中存储的 ClassTest类的完全限定名称是”com/focus/jvm/ClassTest”。因为这种形式的完全限定名是放在了class二进制形式的字节码文件中，所以就称之为二进制形式的完全限定名。

```java
package com.focus.jvm;  
import  java.util.Date;  
public class ClassTest {  
    private Date date1 =new Date();  
    private Date date2;  
    private Date date3;

    public ClassTest(){  
        date2  = new Date();  
    }  
}
```

![2020-09-10-1bsIJQ](https://image.ldbmcs.com/2020-09-10-1bsIJQ.jpg)

由上图可以看到，在ClassTest.class文件的常量池中，共有3个CONSTANT_Class_info结构体，表示ClassTest中用到的Class信息。一个java/util/Date的CONSTANT_Class_info结构体，它在常量池中的位置是#2，存储的指针值为#19，它指向了常量池的第19个常量池项；一个com/focus/ClassTest的CONSTANT_Class_info结构体，它在常量池中的位置是#6，存储的指针值为#22，它指向了常量池的第22个常量池项；另外一个java/lang/Object的CONSTANT_Class_info结构体，它在常量池中的位置是#7，存储的指针值为#23，它指向了常量池的第23个常量池项。

> Java中规定所有的类都要继承java.lang.Object类，即所有类都是java.lang.Object的子类。JVM在编译类的时候，即使我们没有显示地继承Object，JVM编译器在编译的时候会自动帮我们加上去。所以对于某个类而言，其class文件中至少会有两个CONSTANT_Class_info常量池项，用来表示自己的类信息和其父类信息。如果类声明实现了某些接口，那么接口的信息也会生成对应的CONSTANT_Class_info常量池项。

如果在类中使用到了其他的类，只有真正使用到了相应的类，JDK编译器才会将类的信息组成CONSTANT_Class_info常量池项放置到常量池中。如下面的代码：

```java
package com.focus.jvm;  
import  java.util.Date;  
public class ClassTest {  
     
    private Date date3;

   
}
```

编译后结果：

![2020-09-10-iMsSzu](https://image.ldbmcs.com/2020-09-10-iMsSzu.jpg)

在JDK将其编译成class文件时，常量池中并没有java.util.Date对应的CONSTANT_Class_info常量池项。它认为你只是声明了“Ljava/util/Date”类型的变量，并没有实际使用到Ljava/util/Date类。

### 2.4 总结

- 对于某个类或接口而言，其自身、父类和继承或实现的接口的信息会被直接组装成CONSTANT_Class_info常量池项放置到常量池中；

- 类中或接口中使用到了其他的类，只有在类中实际使用到了该类时，该类的信息才会在常量池中有对应的CONSTANT_Class_info常量池项；

- 类中或接口中仅仅定义某种类型的变量，JDK只会将变量的类型描述信息以UTF-8字符串组成CONSTANT_Utf8_info常量池项放置到常量池中。

## 3. 运行时常量池

运行时常量池是方法区的一部分，是一块内存区域。class文件常量池将在类加载后进入方法区的运行时常量池中存放。**一个类加载到JVM中后对应一个运行时常量池**，运行时常量池相对于class文件常量池来说具备动态性，**即字面量可以动态的添加**。Java语言并不要求常量一定只能在编译期产生，运行期间也可能产生新的常量（基本类型包装类和String），这些常量被放在运行时常量池中。class文件常量池只是一个静态存储结构，里面的引用都是符号引用。而运行时常量池可以在运行期间将符号引用解析为直接引用。可以说运行时常量池就是用来索引、查找字段和方法名称和描述符的。给定任意一个字段或方法的索引，最终可得到该字段或方法所属的类型信息和名称及描述符信息。

### 3.1 基本类型

java中基本类型的包装类的大部分都实现了常量池技术，即Byte,Short,Integer,Long,Character,Boolean。这5种包装类默认创建了数值[-128，127]的相应类型的缓存数据，但是超出此范围仍然会去创建新的对象。两种浮点数类型的包装类Float,Double并没有实现缓存池技术。

## 4. 字符串常量池

字符串常量池是**全局的**，JVM中独此一份，因此也称为全局字符串常量池。运行时常量池中的字符串字面量若是成员的，则在类加载初始化阶段就使用到了字符串常量池；若是本地的，则在使用到的时候才会使用字符串常量池。其实，“使用常量池”对应的字节码是一个ldc指令，在给String类型的引用赋值的时候会先执行这个指令，看常量池中是否存在这个字符串对象的引用，若有就直接返回这个引用，若没有，就在堆里创建这个字符串对象并在字符串常量池中记录下这个引用。String类的intern()方法还可以在运行期间把字符串放到字符串常量池中。

- 在 jdk1.6（含）之前也是方法区的一部分，并且其中存放的是字符串的实例；
- 在 jdk1.7（含）之后是在堆内存之中，存储的是字符串对象的引用，字符串实例是在堆中；
- jdk1.8 已移除永久代，字符串常量池是在本地内存当中，存储的也只是引用

- JVM中除了字符串常量池，8种基本数据类型中除了两种浮点类型外，其它的6种基本数据类型的包装类都使用了缓冲池，但是Byte、Short、Integer、Long、Character这几个包装类只有对应值在[-128,127]时才会使用缓冲池，超出此范围仍会去创建新的对象