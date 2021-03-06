---
title: 一次性搞清楚unicode的代码点和代码单元
date: 2018-09-15 22:40:51
tags:
    - java
    - 编码
categories: 程序人生
---

![4702918-13085078b1f89531](http://image.nianlun.tech/2020/11/07/f87c7e9a9af90c5c80efda27303964c3.jpeg)

最近在处理字符过滤，重新研究了下字符、unicode和代码点的相关知识，首先要说一下编码的基本知识unicode
### unicode
>unicode是计算机科学领域里的一项业界标准，包括字符集、编码方案等。计算机采用八比特一个字节，一个字节最大整数是255，还要表示中文一个字也是不够的，至少需要两个字节，为了统一所有的文字编码，unicode为每种语言中的每个字符设定了统一并且唯一的二进制编码，通常用两个字节表示一个字符，所以unicode每个平面可以组合出65535种不同的字符，一共17个平面。

由于英文符号只需要用到低8位，所以其高8位永远是0，因此保存英文文本时会多浪费一倍的空间。

比如汉子“汉”的unicode,在java中输出

```java
System.out.println("\u5B57");
```
### UTF-8
unicode在计算机中如何存储呢，就是用unicode字符集转换格式，即我们常见的UTF-8、UTF-16等。

UTF-8就是以字节为单位对unicode进行编码，对不同范围的字符使用不同长度的编码。

| Unicode | Utf-8 |
| --- | --- |
| 000000-00007F | 0xxxxxxx |
| 000080-0007FF | 110xxxxx 10xxxxxx |
| 000800-00FFFF | 1110xxxx 10xxxxxx 10xxxxxx |
| 010000-10FFFF | 11110xxx10xxxxxx10xxxxxx10xxxxxx |

Java中的String对象就是一个unicode编码的字符串。

java中想知道一个字符的unicode编码我们可以通过Integer.toHexString()方法

```java
        String str = "编";
        StringBuffer sb = new StringBuffer();
        char [] source_char = str.toCharArray();
        String unicode = null;
        for (int i=0;i<source_char.length;i++) {
            unicode = Integer.toHexString(source_char[i]);
            if (unicode.length() <= 2) {
                unicode = "00" + unicode;
            }
            sb.append("\\u" + unicode);
        }
        System.out.println(sb);
        输出\u7f16
```
对应的utf-8编码是什么呢?

7f16在0800-FFFF之间，所以要用3字节模板：1110xxxx 10xxxxxx 10xxxxxx。
7f16写成二进制是：0111 1111 0001 0110
按三字节模板分段方法分为0111 111100 010110，代替模板中的x，得到11100111 10111100 10010110，即“编”对应的utf-8的编码是e7 bc 96，占3个字节

### codepoint
unicode的范围从000000 - 10FFFF，char的范围只能是在\u0000到\uffff，也就是标准的 2 字节形式通常称作 UCS-2，在Java中，char类型用UTF-16编码描述一个代码单元，但unicode大于0x10000的部分如何用char表示呢，比如一些emoji：😀

java的char类型占两个字节，想要表示😀这个表情就需要2个char，看如下代码

```java
String testCode = "ab\uD83D\uDE03cd";
int length = testCode.length();
int count = testCode.codePointCount(0, testCode.length());
//length=6
//count=5
```
第三个和第四个字符合起来代表😀，是一个代码点,
如果我们想取到每个代码点做一些判断可以这么写

```java
        String testCode = "ab\uD83D\uDE03cd";
        int cpCount = testCode.codePointCount(0, testCode.length());
        for(int index = 0; index < cpCount; ++index) {
            //这里的i是字符的位置
            int i = testCode.offsetByCodePoints(0, index);
            int codepoint = testCode.codePointAt(i);
        }
      //输出
      i:0 index: 0 codePoint: 97
      i:1 index: 1 codePoint: 98
      i:2 index: 2 codePoint: 128515
      i:4 index: 3 codePoint: 99
      i:5 index: 4 codePoint: 100
```
也就是按照codePointindex取字符，0取到a，1取到b，2取到\uD83D\uDE03也就是😀，3取到c，4取到d；
按照String的index取字符，0取到a，1取到b，2取到\uD83D，3取到\uDE03，4取到c，5取到d。
这就是codePointIndex和char的index的区别。

取到codePoint就可以按照unicode值进行字符的过滤等操作。

如果有个需求是既可以按照unicode值过滤字符，也能按照正则表达式过滤字符，并且还有白名单，应该如何实现呢。

其实unicode过滤和正则表达式过滤并不冲突，自己实现自己的过滤就好了，如果需求加入了过滤白名单就会复杂一些，不能直接过滤，需要先检验是否是白名单的index。

我的思路是记录白名单char的index，正则表达式或其他过滤方式可以获得违规char的index，unicode黑名单的codepointIndex可以转换成char的index，在获取codePont的index时可以判断当前字符是单char字符还是双char字符，双char字符需要添加2个下标，方法如下

```java             
        //取到unicode值           
        int codepoint = testCode.codePointAt(i);
        //将unicode值转换成char数组
        char[] chars = Character.toChars(codepoint);
        charIndexs.add(pointIndex);
        if (chars.length > 1) {
            //表示不是单char字符，记录index时同时添加i+1
           charIndexs.add(pointIndex + 1);
        }
     
```
   //例
        String str = "ab\uD83D\uDE03汉字";
想处理emoji，那记录的下标就是2、3，最后和白名单下标比较后统一删除

#### 如何区别char是一对还是单个
就之前的例子ab\uD83D\uDE03cd，换种写法\u0061\u0062\uD83D\uDE0\u0063\u0064
程序是如何将\uD83D\uDE03解析成一个字符的呢。这就需要Surrogate这个概念，来自UTF-16。

UTF-16是16bit最多编码65536，那大于65536如何编码？Unicode 标准制定组想出的办法是，从这65536个编码里，拿出2048个，规定他们是「Surrogates」，让他们两个为一组，来代表编号大于65536的那些字符。
编号为 U+D800 至 U+DBFF 的规定为「High Surrogates」，共1024个。
编号为 U+DC00 至 U+DFFF 的规定为「Low Surrogates」，也是1024个。
他们组合出现，就又可以多表示1048576中字符。

看一下String.codePointAt这个方法

```java
    static int codePointAtImpl(char[] a, int index, int limit) {
        char c1 = a[index];
        if (isHighSurrogate(c1) && ++index < limit) {
            char c2 = a[index];
            if (isLowSurrogate(c2)) {
                return toCodePoint(c1, c2);
            }
        }
        return c1;
    }
```
其中有两个方法isHighSurrogate、isLowSurrogate。
第一个方法判断是否为高代理项代码单元，即在'\uD800'与'\uDBFF'之间，
第二个方法判断是否为低代理项代码单元，即在'\uDC00'与'\uDFFF'之间。

codePointAtImpl方法判断当前char是高代理项代码单元，下一个是低代理项代码单元，则这两个char是一个codepoint。

再来看一下unicode转UTF-16的方法
> 如果U<0x10000，U的UTF-16编码就是U对应的16位无符号整数（为书写简便，下文将16位无符号整数记作WORD）。
如果U≥0x10000，我们先计算U'=U-0x10000，然后将U'写成二进制形式：yyyy yyyy yyxx xxxx xxxx，U的UTF-16编码（二进制）就是：110110yyyyyyyyyy 110111xxxxxxxxxx。

还是以U+1F603这个😃为例子，U'=U-0x10000=F603
写成2进制就是1111011000000011，不足20位前面补0，
变成0000111101-1000000011，替换y和x就是1101100000111101，1101111000000011，最后UTF-16编码就是[d83d，de03] 和上面一样。

