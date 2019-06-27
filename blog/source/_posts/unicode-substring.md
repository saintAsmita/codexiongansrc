---
title: unicode-substring
date: 2019-06-27 20:49:38
tags: java
---

本文将提供一种在UTF-8编码下，能够安全截取子串的方法。
UTF-8采用变长进行编码，如果仅仅使用subString，很有可能截断多个字节的一个完整字符，导致字符串没法用，涉及到的概念有代码点（本文不详细介绍）

<!--more-->
首先，考虑这么一个字符串
```java
abc\uD83C\uDDEB\uD83C\uDDF7测试字符串\uD83C\uDDF7
```
普通字符串长度函数`long charCount = s.length();` 得到的字符的数量是14。但是，第3、4和5、6，还有12、13个字符均表示一个unicode字符。

通过代码点数量函数`long realLen = s.codePointCount(0, s.length());` 得到上面那个字符串一共有11个unicode个字符。8个普通的 + 3个2字节的 = **11**。

如果执行`s.subString(0, 4)`那么就是会返回字符0、1、2、3：`abc\\uD83C`，显然，第3、4个字符构成的unicode码被截断了，这是不期望出现的。为了不截断，希望返回是0、1、2、3、4个字符，即`abc\uD83C\uDDEB`。下文的`safeSubString`能够安全截取UTF-8编码的字符串。

`s = s.safeSubString(s, 0, 4)`，返回了5个字符，实际的UTF-8字符数量是4个，即` s.codePointCount(0, s.length()) == 4`，没有从第3个字符截断，而是自动定位到了第4个字符。

```java
/**
 * 用于处理unicode下多字节表示一个字符的情况
 * begin 与 end 均是字符串中char的位置，截取的时候，会适当的向后延长到安全的代码点处，防止截断unicode字符
 * @param s
 * @param begin char的起始索引
 * @param end char的结束索引
 * @return
 */
public static String safeSubString(String s, int begin, int end){
        if(s == null){
            return null;
        }

        int realLen = stringRealLen(s);
        if(end > realLen){
            end = realLen;
        }
        String newData = null;
        try{
            int codePointStartIndex = s.offsetByCodePoints(0, begin);
            int codePointEndIndex = s.offsetByCodePoints(0, end);
            newData = s.substring(codePointStartIndex, codePointEndIndex);
        } catch (Exception e){
            newData = " ";
        }

        return newData;
}

```
