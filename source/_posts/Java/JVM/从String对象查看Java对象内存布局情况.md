---
title: 从String对象查看Java对象内存布局情况
date: 2018/6/1
updated: 2018/6/7
tags:
categories:
   - Programming Language
   - Java
   - 语言
   - JVM
---
## 编码知识
### unicode
`Unicode`编码相关的文章网上已经是汗牛充栋，这里直接粘贴维基百科的一段描述:
>Unicode（中文：万国码、国际码、统一码、单一码）是计算机科学领域里的一项业界标准。它对世界上大部分的文字系统进行了整理、编码，使得电脑可以用更为简单的方式来呈现和处理文字。
Unicode是为了解决传统的字符编码方案的局限而产生的，例如ISO 8859-1所定义的字符虽然在不同的国家中广泛地使用，可是在不同国家间却经常出现不兼容的情况。很多传统的编码方式都有一个共同的问题，即容许电脑处理双语环境（通常使用拉丁字母以及其本地语言），但却无法同时支持多语言环境（指可同时处理多种语言混合的情况
在文字处理方面，统一码为每一个字符而非字形定义唯一的代码（即一个整数）。换句话说，统一码以一种抽象的方式（即数字）来处理字符，并将视觉上的演绎工作（例如字体大小、外观形状、字体形态、文体等）留给其他软件来处理，例如网页浏览器或是文字处理器

所以简单的说，`Unicode`编码就是把世界上很多的文字的字符都统一给编码为了一个唯一数字。

<!--more-->

### UCS-2
`Unicode`编码是由`Xerox, Apple`等公司再1988年组成的统一码联盟所指定的字符集标准。在差不多的时间，国际标准化组织(ISO) ISO 10646 创建了统一字符码集合(`Universal Coded Character Set (UCS) `)。
> UCS不仅给每个字符分配一个代码，而且赋予了一个正式的名字。表示一个UCS或Unicode值的十六进制数通常在前面加上“U+”，例如“U+0041”代表字符“A”

而UCS-2是ISO 10646标准的一种编码方式，它对每个字节都使用2 byte来保存，类似的还有UCS-4 - 即使用4 byte来保存每个字符。 而UCS-2使用2 byte来保存字符，那么范围就在U+0000 - U+FFFF了，也就是最多65536个。 这个区间也被称为基本多文种平面(`the Basic Multilingual Plane`)。

后面`Unicode`和`ISO`统一了字符编码，使得同一个字符在`Unicode`和`ISO 10664`的编码都是一致的。

### UTF-8, UTF-16
上面讲的`Unicode`它指明了一个字符对应的编码是多少，但是在实际传输或者存储过程中，由于不同系统平台的设计不一定一致、节省空间等目的，所以有了`Unicode`的实现方式，称为**Unicode转换格式(`Unicode Transformation Format, UTF`)**
比如说，一个仅包含7位(bit)的ASCII字符，也是用`Unicode`编码为2 byte长度，明显浪费了太多空间。所以有了UTF8, UTF16

#### UTF8
`UTF8`是一种变长字节编码，它将基本7位ASCII字符仍用7位表示（占用1 byte, 首位补0），而对于其他字符则使用一定算法转换位1-3 byte来编码，并且利用首位为0或者1进行识别。

#### UTF-16
`UTF-16`也是一种变长字节编码，它使用2 byte或者4 byte来表示字符。上面说过，基本多语言平面的范围是U+0000 - U+FFFF，所以在UTF-16中，基本多语言平面范围内的字符被编码为2 byte,而其他范围的则是4 byte。

## Java中字符串的处理
### JVM规范
在[JVM规范中数据类型定义](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.3)中可以看到:
> char, whose values are 16-bit unsigned integers representing Unicode code points in the Basic Multilingual Plane, encoded with UTF-16, and whose default value is the null code point ('\u0000')
char, 值为16位无符号整形，代表BMP中的`Unicode`编码， 使用UTF-16进行编码。默认值为`\u0000`

所以这里可以很明显的看到，JVM对于`char`，保存为2 byte, 16 bit. 但是，`char`可以保存的字符范围是BMP, U+0000 - U+FFFF， 所以对于超出这个范围的字符，则需要上层语言自行处理。

### Java中String的实现
首先必须提到的是，Java中的`char`基本类型与JVM的`char`是一一对应的。
上面提到，对于范围在U+0000-U+FFFF的范围，`char`可以直接保存，而超出这个范围的则不行。大部分中文都在这个BMP范围，所以`char`是可以保存一个中文的。

如果需要保存超出BMP范围的字符或者多个BMP字符，则需要用到`String`类型。`String`在Java中是一个类，而对于JVM来说是不存在的，Java中的任何类在JVM中都对应着一个`reference`。
`String`使用`char[]`来保存当前对象代表的字符串。如果字符串中由字符超过BMP了是怎么处理的？

String源码中从Unicode编码到字符串的代码如下:

```java
public String(int[] codePoints, int offset, int count) {
    //忽略边界检查
    final int end = offset + count;

    // Pass 1: Compute precise size of char[]
    int n = count;
    for (int i = offset; i < end; i++) {
        int c = codePoints[i];
        //判断是否是BMP字符，如果不是长度+1
        if (Character.isBmpCodePoint(c))
            continue;
        else if (Character.isValidCodePoint(c))
            n++;
        else throw new IllegalArgumentException(Integer.toString(c));
    }

    // Pass 2: Allocate and fill in char[]
    final char[] v = new char[n];

    for (int i = offset, j = 0; i < end; i++, j++) {
        int c = codePoints[i];
        //如果是BMP字符，则素组索引处直接设置为该值
        if (Character.isBmpCodePoint(c))
            v[j] = (char)c;
        else
            //如果不是，则调用了另外一个方法并且把下标+1
            Character.toSurrogates(c, v, j++);
    }

    this.value = v;
}
```

调用的函数为:

```java
static void toSurrogates(int codePoint, char[] dst, int index) {
    dst[index+1] = lowSurrogate(codePoint);
    dst[index] = highSurrogate(codePoint);
}
public static char highSurrogate(int codePoint) {
    return (char) ((codePoint >>> 10)
                   + (MIN_HIGH_SURROGATE - (MIN_SUPPLEMENTARY_CODE_POINT >>> 10)));
}
public static char lowSurrogate(int codePoint) {
    return (char) ((codePoint & 0x3ff) + MIN_LOW_SURROGATE);
}
```

所以可以看到，对于超过了BMP范围的字符，`String`使用2个`char`来保存，也就是2*2=4 byte来保存，否则就是2 byte

### 内存占用分析
#### `char`
前面说过，Java中`char`与JVM的一一对应，所以它占用的就是2 byte, 16 bit。对于它不能保存的则使用`String`来保存。

#### `String`
`String`是一个对象，分析Java中对象的内存占用必须考虑到Java对象的内存布局。
在JVM的规范中并没有规定一个对象具体要如何存储，对象存储结构的细节属于JVM实现。这里我们分析最为流行的`Hotspot JVM`.

HotSpot JVM的源码中关于内存布局有如下的[注释](http://hg.openjdk.java.net/jdk8/jdk8/hotspot/file/87ee5ee27509/src/share/vm/oops/markOop.hpp):
```cpp
//  32 bits:
//  --------
//             hash:25 ------------>| age:4    biased_lock:1 lock:2 (normal object)
//             JavaThread*:23 epoch:2 age:4    biased_lock:1 lock:2 (biased object)
//             size:32 ------------------------------------------>| (CMS free block)
//             PromotedObject*:29 ---------->| promo_bits:3 ----->| (CMS promoted object)
//
//  64 bits:
//  --------
//  unused:25 hash:31 -->| unused:1   age:4    biased_lock:1 lock:2 (normal object)
//  JavaThread*:54 epoch:2 unused:1   age:4    biased_lock:1 lock:2 (biased object)
//  PromotedObject*:61 --------------------->| promo_bits:3 ----->| (CMS promoted object)
//  size:64 ----------------------------------------------------->| (CMS free block)
//
//  unused:25 hash:31 -->| cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && normal object)
//  JavaThread*:54 epoch:2 cms_free:1 age:4    biased_lock:1 lock:2 (COOPs && biased object)
//  narrowOop:32 unused:24 cms_free:1 unused:4 promo_bits:3 ----->| (COOPs && CMS promoted object)
//  unused:21 size:35 -->| cms_free:1 unused:7 ------------------>| (COOPs && CMS free block)
```

上面的注释中几个名词的解释:
> - hash包含了唯一Hash编码。最长为31 bit。
- `biased_lock` 偏心锁。用来给指定的线程指定偏心锁，当低3位的这个标志被设置的时候，锁就要么是针对某个线程要么是针对“匿名”偏心。
- `OOP`,`ordinary object pointer`,普通对象指针，它用来表示对象的实例信息。正常情况下就是一个本地机器指针。这就意味着在[LP32](https://docs.oracle.com/cd/E19620-01/805-3024/lp64-1/index.html)上这个指针就是32位，[LP64](https://docs.oracle.com/cd/E19620-01/805-3024/lp64-1/index.html) JDK上这个指针就是64位。 所以，这也意味着在64位上相同的代码会是32位上占用的堆空间的1.5倍左右。32位机器最大可用堆差不多4g，而64位
- `Compressed OOPS`, 压缩普通对象指针，这就是为了解决上面的指针导致占用空间膨胀，并且一般情况下用不到64位的堆范围，所以采用了一些手段堆这个字段进行压缩。原理简单的说就是把64位指针地址改为64位基础地址加上一个32位偏移量，[中文参考](https://blog.csdn.net/lqp276/article/details/52231261),[openJDK文档](https://wiki.openjdk.java.net/display/HotSpot/CompressedOops)

由于压缩对象指针这个优化的影响，我们将分别查看开启和关闭该选项情况下的资源占用情况。
`Oracle`提供了一个[JOL](http://openjdk.java.net/projects/code-tools/jol/)工具可以帮助分析内存布局，我们使用该工具，分析如下代码:

```java
//注意加入JOL包的依赖
public class LayOutDemo {
    public static void main(String[] args) throws Exception {
        out.println(VM.current().details());
        String ins = "哈哈哈哈哈哈哈哈";
        out.println(ClassLayout.parseClass(java.lang.String.class).toPrintable());
        out.println(ClassLayout.parseInstance(ins).toPrintable());
    }
}
```

使用`-XX:-UseCompressedOops`关闭指针优化的输出:

```
# Running 64-bit HotSpot VM.
# Objects are 8 bytes aligned.
# Field sizes by type: 8, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
# Array element sizes: 8, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]

java.lang.String object internals:
 OFFSET  SIZE     TYPE DESCRIPTION                               VALUE
      0    16          (object header)                           N/A
     16     8   char[] String.value                              N/A
     24     4      int String.hash                               N/A
     28     4          (loss due to the next object alignment)
Instance size: 32 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.String object internals:
 OFFSET  SIZE     TYPE DESCRIPTION                               VALUE
      0     4          (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4          (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4          (object header)                           78 8f cd 1b (01111000 10001111 11001101 00011011) (466456440)
     12     4          (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
     16     8   char[] String.value                              [哈, 哈, 哈, 哈, 哈, 哈, 哈, 哈]
     24     4      int String.hash                               0
     28     4          (loss due to the next object alignment)
Instance size: 32 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

所以我们看到在关闭优化的64位JDK上，String对象头长度为16个字节，然后紧接是两个成员变量:
- `hash` int类型,长度4个字节
- `value[]` char类型，长度8个字节（因为它实际上是个指针，所以在64位系统上就是8字节。该指针指向的堆地址才是这个字符串字面量的实际位置）

此时，对象长度为28，此时需要[对齐](https://en.wikipedia.org/wiki/Data_structure_alignment)，所以有了额外的4字节，这个`String`对象的总大小为32字节。 但是还不包括这个字符串数组。
数组对象在内存中的结构与上面的内存布局描述没有差别，只是多了一个`int`型的`size`成员变量标识数组长度。所以可以知道这个字符串数组的大小为16（头）+4（`size`）+8*2 (`char`长度为2) = 36字节。

所以，为了保存一个8个字符的中文，需要36字节的字符串数组加上32字节的管理这个数组的`String`对象。总空间占用比（总共占用空间除以字符串本身大小）为68 / 16 = 4.25. 相较于在C中使用`w_char*`，空间浪费的却较为严重。

那么，再查看下开启指针压缩的情况,使用`-XX:+UseCompressedOops`开启这个优化:

```
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# Objects are 8 bytes aligned.
# Field sizes by type: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
# Array element sizes: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]

java.lang.String object internals:
 OFFSET  SIZE     TYPE DESCRIPTION                               VALUE
      0    12          (object header)                           N/A
     12     4   char[] String.value                              N/A
     16     4      int String.hash                               N/A
     20     4          (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

java.lang.String object internals:
 OFFSET  SIZE     TYPE DESCRIPTION                               VALUE
      0     4          (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4          (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4          (object header)                           da 02 00 f8 (11011010 00000010 00000000 11111000) (-134216998)
     12     4   char[] String.value                              [哈, 哈, 哈, 哈, 哈, 哈, 哈, 哈, 哈, 哈, 哈, 哈, 哈, 哈, 哈, 哈, 哈, 哈, 哈, 哈, 哈, 哈, 哈, 哈]
     16     4      int String.hash                               0
     20     4          (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

可以看到，对象大小由没有压缩的32字节变为压缩后的24字节。最终的空间占用比为(24+36)/16=3.75。

对象指针压缩(`CompressedOops`)仅在[LP64](https://docs.oracle.com/cd/E19620-01/805-3024/lp64-1/index.html)上有效，开启过后使得64位上的空间占用和32位一样了，并且没有损失堆可访问空间（32位默认只有最大4g, 指针压缩的时候尽管偏移指针是32位，但是对象头上有3位可以作为附加偏移量，所以最终可以访问2*35=32G空间）
IBM上有篇[文章](https://www.ibm.com/developerworks/library/j-codetoheap/index.html)对这个也做了详细的介绍。

所以到这里可以清晰的看到Java对象在HotSpot JVM中的**内存布局情况：**
- 12-16个字节（多种情况）的对象头。 对象头里保存有对象Hash,锁，线程，GC分代等信息
- 对象的成员变量。 如果成员变量是指针，则是对应系统的指针的长度（32位或者64位，默认在64位开启了指针压缩后是32位）
- 如果有对齐的需要（8的倍数），则会添加合适的空bit已满足对齐


下面分别具体查看下这两种情况:

- 对象头与成员变量

比如我们查看`HashSet`的内存布局情况：

```java
public class LayoutDemo1 {
    public static void main(String[] args) {
        out.println(VM.current().details());
        out.println(ClassLayout.parseClass(java.util.HashSet.class).toPrintable());
    }
}
```

使用JDK 1.8.0.6_161的默认参数运行，输出:

```
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# Objects are 8 bytes aligned.
# Field sizes by type: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
# Array element sizes: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]

java.util.HashSet object internals:
 OFFSET  SIZE                TYPE DESCRIPTION                               VALUE
      0    12                     (object header)                           N/A
     12     4   java.util.HashMap HashSet.map                               N/A
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

- 补齐:
如下代码：

```java

public class LayoutDemo1 {
    public static void main(String[] args) {
        out.println(VM.current().details());
        out.println(ClassLayout.parseClass(Demo.class).toPrintable());
    }

    private class Demo{
        int a;
        boolean b; //boolean的长度为1，所以会补3bit
        HashSet c;
    }
}
```

输出：

```java
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 3-bit shift.
# Objects are 8 bytes aligned.
# Field sizes by type: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
# Array element sizes: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]

org.doubleysoft.testpersonal.LayoutDemo1$Demo object internals:
 OFFSET  SIZE                                       TYPE DESCRIPTION                               VALUE
      0    12                                            (object header)                           N/A
     12     4                                        int Demo.a                                    N/A
     16     1                                    boolean Demo.b                                    N/A
     17     3                                            (alignment/padding gap)
     20     4                          java.util.HashSet Demo.c                                    N/A
     24     4   org.doubleysoft.testpersonal.LayoutDemo1 Demo.this$0                               N/A
     28     4                                            (loss due to the next object alignment)
Instance size: 32 bytes
Space losses: 3 bytes internal + 4 bytes external = 7 bytes total
```
