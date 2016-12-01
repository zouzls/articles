---
title: 深入理解JVM之前端编译器（一）
date: 2016-09-06 23:50:44
tags:
  - JVM
categories: JVM
---
前两天在leetcode做了算法题，惊讶的发现用java实现的时间复杂度，竟然跻身于C/C++同列，甚至偶尔会超过后两者，虽然知道JVM功不可没，但还是很好奇在VM编译过程中到底发生了什么，翻出《深入理解java虚拟机》一探究竟，算是有所收获，记录如下。
<!--more-->
### 概述
java语言的“编译期”其实是一段“不确定”的操作过程，因为可能是下面三种：
- 前端编译器
叫编译器的前端可能更合适，**主要是把*.java文件转变成*.class文件。**主要种类有：Sun的Javac、Eclipse JDT的增量式编译器（ECJ）。
- JIT编译器
就是可能指虚拟机的后端运行期编译器（JIT编译器：Just In Time Compiler），**把「字节码」变成「机器码」**，主要有：Hotspot VM的C1、C2编译器。
- AOT编译器
上面两个可能关心更多，这个是指用静态提前编译器（AOT编译器：Ahead Of Time Compiler）直接把*.java变成本地机器码的过程。

本文后面所指的编译器和编译期都表示第一种。
关于优化的两点解释：
1. Javac这类编译器对代码的运行效率几乎没有任何优化措施。只是做了一些针对java语言「编码过程」的“优化”措施来改善程序员的「编码风格和编码效率」（俗称“语法糖”，并没有涉及虚拟机底层的改进）。
2. 虚拟机设计团队把对性能的优化集中到了后端的即时编译中，这样可以让那些不是有javac产生的Class文件（如JRuby、Groovy等语言的Class）也同样能享受到编译器优化所带来的好处。

也就是说java中即时编译器的在运行期的优化对程序运行更重要，而前端编译器在编译期的优化对于程序编码更重要。注意此处的「编译期」和「运行期」、「程序编码」和「程序运行」的区别。

### Javac编译器
Javac不像是Hotspot虚拟机本身是CPP（少量C）写的，本身是java语言编写的程序，这样java程序员就很方便了解它的编译过程了。关于源码环境搭建和调试不做详述，大致说下Javac的编译过程，主要分为3个过程，分别是：
- 解析与填充符号表过程
- 插入式注解处理器的注解处理过程
- 分析与字节码生成过程

### Java语法糖
语法糖虽不能带来实质性的功能的改进，但是它们或能提高效率，或能提升语法严谨性，或能减少编码出错机会。java中常见的语法糖如下：
#### 泛型与泛型擦除
java的泛型规则只在程序源码中存在，在编译后就已经替换为原来的原生类型（RawType，裸类型），并且在相应的地方插入了强制转换。
#### 自动装箱、拆箱与遍历循环、变长参数
演示代码如下（以下jdk环境1.8）：
```java
public class AutoBoxCompile {

    public static void main(String[] args) {
        List<Integer> list =Arrays.asList(1,2,3,4);
        int sum=0;
        for (Integer i : list) {
            sum+=i;
        }
        System.out.println(sum);
    }
}
```
用的本地jd-gui.exe工具反编译之后变成：
```java
public class AutoBoxCompile
{
  public static void main(String[] args)
  {
    List list = Arrays.asList(new Integer[] { Integer.valueOf(1), Integer.valueOf(2), Integer.valueOf(3), Integer.valueOf(4) });
    int sum = 0;
    for (Integer i : list) {
      sum += i.intValue();
    }
    System.out.println(sum);
  }
}
```
foreach明明是一颗语法糖特性，猜想是工具的问题，找了一个[在线反编译工具](http://www.ludaima.cn/java.html)，得到下列代码，可以发现变长参数、自动装箱和遍历循环的语法糖特性：
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/QQ%E5%9B%BE%E7%89%8720160907102754.png)
#### 条件编译
类似于下面这段if代码，在编译过程中就会被“运行”。
```java
public class Ifcompiler {
    public static void main(String[] args) {
        if (true) {
            System.out.println(1111);
        }else {
            System.out.println(2222);
        }
    }
}
```
生成的字节码只包括System.out.println(1111);，将上述编译之后产生的class反编译如下图：
![](http://oaewlsdmg.bkt.clouddn.com/image/jpg/QQ%E6%88%AA%E5%9B%BE20160907095715.png)
只有使用条件为常量的if语句才能有上述效果，否则会被拒绝编译比如while(false){};。
还有很多其他的语法糖，比如内部类、枚举、断言、switch以及try中关闭资源等等，有时间再去尝试，重要的是明白语法糖是怎么回事。

### 最后
「之所以把从java文件到字节码文件的编译器叫做前段编译器，是因为它只完成了从程序到抽象语法树或者中间字节码的转变，而在此之后还有一组内置于虚拟机内部的“后端编译器”完成了从字节码到本地机器码的过程，即前面提到的即时编译器或JIT编译器，这个编译器的编译速度及结果的优劣，是衡量虚拟机性能的很重要的指标。」
JVM还有很多的东西没有去挖掘，希望这是个开端，能不断的去探索java虚拟机更深处的东西。

-EOF-