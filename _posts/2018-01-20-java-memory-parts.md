---
layout: post
title:  "JVM运行时内存数据区域"
date:   2018-01-20 21:49:08 +0800
categories: JVM
tags: 虚拟机栈 方法区 Java堆 运行时常量池 
author: Tommy.Tesla
mathjax: true
---

## 1 讨论背景

周志明老师写的《深入理解Java虚拟机》应该很多程序员都读过，第二章中阐述了Java虚拟机在执行Java程序的过程中是如何管理内存的，以及这些内存是如何被划分成更细的逻辑区域的。如下图所示，按照书中的论述JVM运行时数据区域包含以下几个数据区[1]。

![](/image/java-memory-parts/parts-in-book.png)

按照《Java虚拟机规范（Java SE 7版）》，各区域的功能简要介绍如下：
* **程序计数器**：各线程私有。用于记录每个线程当前执行到的字节指令行号以及相关信息。这是唯一的不会抛出OOM异常的区域。
* **Java虚拟机栈**：各线程私有。虚拟机栈由一个个的栈帧组成，每个栈帧包含了对应方法执行所需要的信息，具体包括：局部变量表、操作数栈（类似于编译型语言体系下的数据寄存器）、动态链接（某些接口符号可能会动态的指向不同的目标方法）、函数返回地址以及其他一些相关信息。理论上当函数调用链超过栈的深度时就会触发StackOverflow，当该区域设置为动态扩展时，虚拟机无法为栈申请到更多内存时就会触发OOM。事实中基本上不管哪种情况，结果都很可能会是StackOverflow，因为栈容量和栈帧的大小决定了栈的深度（栈帧大小*深度<=栈容量），所以当OOM时，栈深度一定也已经不够用了，所以抛出StackOverflow异常也无可厚非。可以通过“-Xss”来配置虚拟机栈固定大小。
* **Java堆**：各线程公有。虚拟机工作的主要内存区域（大部分情况下也是最大的），绝大部分对象实例的内存分配都在这里进行。Java 7和之前的Java堆细分为：新生代（伊甸区、存活区0、存活区1）、年老代和永久代。Java 8去除了永久代，替换以Metaspace。在JVM的运行中，大部分情况下，GC主要就发生在堆区域，
* **方法区**：各线程公有。用于存放类定义、常量池、静态变量（static修饰）、编译后的字节码等。方法区实际上是从堆上划分出来的一块区域，但是其GC机制是单独的，与堆不同，所以为了区分方法区和堆，通常又把方法区叫做“非堆”。方法区对应了堆中的永久代。因此在Java8以及之后版本中，永久代被抹除了，方法区也移到了元数据空间（metaspace）中。
* **运行时常量池**：各线程公有。用于存放类信息中的常量（字面量、符号引用等），每个类编译后的信息中的都有一个常量池，可以通过javap -vebose xxxx.class命令来查看。
* **直接内存**：进程间公有。直接内存不属于Java虚拟机运行时数据区的一部分，它是指操作系统分配给虚拟机以及其他进程所运行的那块内存区域，之所以这么说，是因为很多服务器都是虚拟机（操作系统级别），对于物理机来说，这块内存就是指操作系统所管控的物理内存。通过在堆中创建一个DirectByteBuffer实例来对直接内存进行访问。

很多读者了解完这些后还是云里雾里，各论坛还是会出现各种没有定论的问题，比如
1. 字符串常量池属于哪个数据区？书中对字符串常量池和运行时常量池描述的相当晦涩和模糊。
2. Java6、Java7和Java8的运行时内存数据区域到底有何不一样？
3. 什么是字面量，什么又是字符串常量？
4. 什么是**本地内存**？他和**直接内存**相同嘛？什么又是**堆外内存**？

下面我们围绕这几个问题做一些讨论和引申，从而帮助我们更好的理解运行时数据区域划分。

## 2 字符串常量池

我们先来回答第一和第二个问题。

### 2.1 字符串常量池在哪 

在不同的Java版本中，规范规定的字符串常量池的位置也不一样。以下三张图分别代表了Java6、Java7和Java8体系下的Java虚拟机与运行时数据区域划分，哪些是线程私有，哪些是线程公有，哪些又是进程间公有都比较清晰了。

#### 2.1.1 Java 6 虚拟机运行数据区

![Java 6 内存数据区域划分](/image/java-memory-parts/java6.png)

当我们听到“字符串常量池也是方法区的一部分”的时候，我们要知道他大概暗指的是Java 6或者之前的版本。如上图所示，在Java 6虚拟机规范中，字符串常量池确实是方法区的一部分，受永久代内存区大小的限制。当频繁使用Spring.intern()时，可能会引发OOM（PermGen space)。

#### 2.1.2 Java 7 虚拟机运行数据区

![Java 7 内存数据区域划分](/image/java-memory-parts/java7.png)

从Java 7 开始，规范将**字符串常量池**迁移到了**Java堆**中，受Java堆大小的限制。当频繁大量使用String.intern()时，可能会引发OOM(Java heap space)。

#### 2.1.3 Java 8 虚拟机运行数据区

![Java 8 内存数据区域划分](/image/java-memory-parts/java8.png)

Java 8 虚拟机规范彻底移除了永久代（-XX:Permsize和-XX:MaxPermsize均已失效），替而代之的则是**元空间(Metaspace)**。**字符串常量池**仍然在**Java堆**中，但方法区已经迁移到了元空间中。这时候由于滥用 String.intern()引发的OOM依旧在Java堆中。

### 2.2 字符串常量池是啥

那么字符串常量池的数据结构是怎么实现的呢？答案是HashMap，每个字符串常量池对应了一个StringTable的数据结构，其本质并不是Table，而是一个HashMap。这个HashMap的容量是固定的（默认1009），可以通过**-XX:StringTableSize**来设置，注意这个值是指哈希表中桶的数量，不是占用内存的大小。所以这个值最好是一个质数，并且要大于默认的1009[2]。


## 3 字面量和字符串常量

如以下代码：
```
String str = "123";
```
其中"123"就是我们经常看到的“字面量”。字面量是随着Class信息等在类被加载完毕后一起进入**运行时常量池**的。
而
```
String str2 = str.intern();
```
这句代码则尝试将str的值放入字符串常量池，然而"123"已经在类信息的常量池中了，所以StringTable实际记录的是类信息常量池中该字符串的引用。

对于语句：
``` 
String str = new StringBuilder("hello").append(" world").toString().intern();
```
这会将新创建的“hello world”的堆内对象引用（str）放入到**字符串常量池**中，因为这是第一次出现，没有其他地方存在该值的引用。


## 4 本地内存和直接内存

首先需要说明的是，**本地内存**(Native Memory)和**堆外内存**(Off-heap Memory)的含义是一样的。而关于**直接内存**和**本地内存**的关系，StackOverflow上也没有说清楚的帖子，第二部分中的三张图已经可以很好的说明直接内存和本地内存的关系了，所谓的本地内存是操作系统分配给JVM虚拟机（作为一个进程）使用的内存块中除去堆的那一部分。而直接内存则是所有进程共享的操作系统所控制的内存。所以可以这么说：本地内存和直接内存的关系就像“苹果”和“水果”的关系，苹果属于水果，是水果更具体的限定。Java8中的元空间就属于本地内存空间，而他们都是直接内存的一部分。
通过DirectByteBuffer分配的内存区域一定在本地内存中,它也受直接内存大小的限制。本地内存的大小也有限制，比如Window中对每个程序运行所需的内存大小做了2G的默认限制，这只时候其上运行的JVM的本地内存大小≈2G-JVM堆内存大小。


## 5 字符串常量池所属数据区的具体说明

下面我们举2个例子讨论下在Java6和Java7（含之后版本）下字符串常量池迁移带来的变化

### 5.1 例子1

请给出以下代码抛出异常的类型：
``` 
import java.util.ArrayList;
import java.util.List;

public class Test {  
	  public static void main(String[] args){  
		  List<String> list = new ArrayList<String>();
		  int i = 0;
		  while(true) { 
			   list.add( String.valueOf(i++).intern());
		  }
	  }
}

```
然后启动参数中我们加上：
``` 
-XX:PermSize=10M -XX:MaxPermSize=10M
```
分析下这个代码，其意图在于不断的产生新的字符串，并且放入**字符串常量池**中，试图撑爆永久代。然而这只会在Java 6 中发生，对于Java7和Java8来说，字符串常量池已经迁移到了**Java堆**中，如果这时候我们添加以下虚拟机参数：
``` 
-Xms10M -Xmx10M
```
则会引发：java.lang.OutOfMemoryError: GC overhead limit exceeded 这样的错误，这个异常的本质与 OOM(Heap space)一直，都是堆内存溢出。


### 5.2 例子2

以下代码在Java6和Java7中输出也不相同：
``` 

public class TestStringConstantPool {

	public static String hello = "Hello Java";
	
	public static void main(String[] args) {
		 
		String str1 = new StringBuilder("Hello ").append("World").toString();
		System.out.println(str1.intern() == str1);
		
		String str2 = new StringBuilder("Hello ").append("Java").toString();
		System.out.println(str2.intern() == str2); 
	}
} 

```
在Java6中会输出：
``` 
false
false
```
 在Java7中则输出：
 ``` 
 true
 false
 ```
 首先我们分析下Java6中的场景，Java6中字符串常量池还是运行时常量池的一部分，所以使用String.intern()时，会把堆中的字符串复制到方法区中，返回的是方法区中的对象引用。所以不管如何，堆中对象和方法区中对象应用都不会想等。
 而在Java7中，这个情况发生了变化，字符串常量池转移到了堆中，对于**str1**来说，字符串常量池StringTable会记录其在堆中的引用（即str1）。所以**str1.intern() == str1**成立。而**str2**情况则不一样了，因为**“Hello Java”**字符串已经存在于方法区的运行时常量池中，所以intern()返回的是方法区中的对象引用。所以**str2.intern() == str2**不成立。
 

## 参考文献
1. 深入理解Java虚拟机。
2. http://java-performance.info/string-intern-in-java-6-7-8/

* content
{:toc}