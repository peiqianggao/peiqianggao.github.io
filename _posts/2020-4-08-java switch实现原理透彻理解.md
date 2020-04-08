---
layout:     post
title:      Java switch实现原理透彻理解
subtitle:   深入分析作用于 switch 下各种数据类型的实现原理
date:       2020-04-08
author:     gaopq
header-img: img/post-bg-map.jpg
catalog: true
tags:
    - Java
    - 关键字分析
    - 理解系列
---
Java 官方文档描述:

A switch works with the byte, short, char, and int primitive data types. It also works with enumerated types, the String class, and a few special classes that wrap certain primitive types: Character, Byte, Short, and Integer
 
#### 在基本数据类型 byte / char 下
```java
public class SwitchBasicTest {
    
    private int switchByte(byte i) {
        int ret;
        switch (i) {
            case 0:
                ret = 10;
                break;
            case 1:
                ret = 11;
                break;
            default:
                ret = 0;
        }

        return ret;
    }
    private int switchChar(char i) {
        int ret;
        switch (i) {
            case 0:
                ret = 10;
                break;
            case 1:
                ret = 11;
                break;
            case 'a':
                ret = 20;
                break;
            case '写':
                ret = 21;
                break;
            default:
                ret = 0;
        }

        return ret;
    }
}
```
用 javap -v 或其他工具反编译 `SwitchBasicTest.class`，本文使用 IDEA 插件 `jclasslib Bytecode viewer`

反编译得到 `switchByte` 方法的字节码：
```
 0 iload_1
 1 lookupswitch 2
        0:  28 (+27)
        1:  34 (+33)
        default:  40 (+39)
28 bipush 10
30 istore_2
31 goto 42 (+11)
34 bipush 11
36 istore_2
37 goto 42 (+5)
40 iconst_0
41 istore_2
42 iload_2
43 ireturn

```
`switchChar` 方法的字节码：
```
 0 iload_1
 1 lookupswitch 4
        0:  44 (+43)
        1:  50 (+49)
        97:  56 (+55)
        20889:  62 (+61)
        default:  68 (+67)
44 bipush 10
46 istore_2
47 goto 70 (+23)
50 bipush 11
52 istore_2
53 goto 70 (+17)
56 bipush 20
58 istore_2
59 goto 70 (+11)
62 bipush 21
64 istore_2
65 goto 70 (+5)
68 iconst_0
69 istore_2
70 iload_2
71 ireturn

```
可知对于 `byte` 等基础数据类型，`switch` 直接使用整型做比较
- byte, short 可直接转整整值
- char 取 `unicode` 整型码值
- Integer 等包装类型，会调用 `intValue` 方法获得变量的整值

#### 作用在 String 下(JDK1.7及之后版本)
```java
public class SwitchBasicTest {
    private int switchString(String i) {
            int ret;
            switch (i) {
                case "0":
                    ret = 10;
                    break;
                case "1":
                    ret = 11;
                    break;
                case "a":
                    ret = 20;
                    break;
                default:
                    ret = 0;
            }
    
            return ret;
        }
}
```
使用 IDEA 编辑器打开 .class 文件看到反编译代码：
```java
public class SwitchBasicTest {
    private int switchString(String i) {
        byte var4 = -1;
        switch(i.hashCode()) {
        case 48:
            if (i.equals("0")) {
                var4 = 0;
            }
            break;
        case 49:
            if (i.equals("1")) {
                var4 = 1;
            }
            break;
        case 97:
            if (i.equals("a")) {
                var4 = 2;
            }
        }

        byte ret;
        switch(var4) {
        case 0:
            ret = 10;
            break;
        case 1:
            ret = 11;
            break;
        case 2:
            ret = 20;
            break;
        default:
            ret = 0;
        }

        return ret;
    }
}
```
反编译得到 `switchString` 的字节码：
```
  0 aload_1
  1 astore_3
  2 iconst_m1
  3 istore 4
  5 aload_3
  6 invokevirtual #8 <java/lang/String.hashCode>
  9 lookupswitch 3
        48:  44 (+35)
        49:  59 (+50)
        97:  74 (+65)
        default:  86 (+77)
 44 aload_3
 45 ldc #9 <0>
 47 invokevirtual #10 <java/lang/String.equals>
 50 ifeq 86 (+36)
 53 iconst_0
 54 istore 4
 56 goto 86 (+30)
 59 aload_3
 60 ldc #11 <1>
 62 invokevirtual #10 <java/lang/String.equals>
 65 ifeq 86 (+21)
 68 iconst_1
 69 istore 4
 71 goto 86 (+15)
 74 aload_3
 75 ldc #12 <a>
 77 invokevirtual #10 <java/lang/String.equals>
 80 ifeq 86 (+6)
 83 iconst_2
 84 istore 4
 86 iload 4
 88 tableswitch 0 to 2	
        0:  116 (+28)
        1:  122 (+34)
        2:  128 (+40)
        default:  134 (+46)
116 bipush 10
118 istore_2
119 goto 136 (+17)
122 bipush 11
124 istore_2
125 goto 136 (+11)
128 bipush 20
130 istore_2
131 goto 136 (+5)
134 iconst_0
135 istore_2
136 iload_2
137 ireturn
```
可以看到 `switch` 作用在 `String` 上最终拆分成两个针对整型 `switch` 语句，具体流程为：
- 计算并比较 hashcode
- 如果 hashcode 相同，则进一步使用 `equals` 确定字符串内容相同，处理两个字符串 hashcode 相同的情况
 
#### 作用在枚举下
```java
public enum SeasonEnum {
    SPRING, SUMMER, AUTUMN, WINTER
}

public class SwitchEnumTest {
   
    private static int testSpring(SeasonEnum seasonEnum) {
        int ret;
        switch (seasonEnum) {
            case AUTUMN:
                ret = 1;
                break;
            case SPRING:
                ret = 2;
                break;
            case SUMMER:
                ret = 3;
                break;
            case WINTER:
                ret = 4;
                break;
            default:
                ret = 5;
        }
        return ret;
    }
}
```
反编译得到 `testSpring` 方法的字节码：
```
 0 getstatic #6 <xx/SwitchEnumTest$1.$SwitchMap$xx$enumt$SeasonEnum>
 3 aload_0
 4 invokevirtual #7 <xx/enumt/SeasonEnum.ordinal>
 7 iaload
 8 tableswitch 1 to 4	1:  40 (+32)
        2:  45 (+37)
        3:  50 (+42)
        4:  55 (+47)
        default:  60 (+52)
40 iconst_1
41 istore_1
42 goto 62 (+20)
45 iconst_2
46 istore_1
47 goto 62 (+15)
50 iconst_3
51 istore_1
52 goto 62 (+10)
55 iconst_4
56 istore_1
57 goto 62 (+5)
60 iconst_5
61 istore_1
62 iload_1
63 ireturn
```   
可以看到：
`4 invokevirtual #7 <xx/enumt/SeasonEnum.ordinal>`

即 `switch` 作用在枚举上时，调用枚举的 `ordinal` 方法 ，即最终还是 `int` 值比较

#### lookupswitch 和 tableswitch
- `tableswitch` 使用数组数据结构，通过下标可直接定位到跳转目标行
- `lookupswitch` 维护了一个key-value的关系，通过逐个比较索引来查找匹配的待跳转的行数
- `tableswitch` 比 `lookupswitch` 查找性能更佳
- `switch`语句中的 `case` 分支条件值比较稀疏时，`tableswitch`指令的空间使用率偏低，这种情况下可能使用`lookupswitch` 指令

#### 结论
- `switch` 语法参数实际上只对整型有效
- `break` 在字节码层面上会生成一条 `goto` 语句，`case` 下不带 `break` 时直接进入下一个 `case` 逻辑
- `switch` 作用在枚举上实际使用了枚举的 `ordinal` 方法返回值作为比较的依据
- `switch` 作用在字符串上时使用字符串的 `hashcode` 作为比较的依据，`hashcode` 相同时进一步使用 `equals` 比较字符串内容
- 根据java虚拟机规范，java虚拟机的 `tableswitch` 和 `lookupswitch` 指令都只能支持int类型的条件值，所以 `switch` 不支持 `long` 长整型

#### 资料参考
- [Java Document - The switch Statement](https://docs.oracle.com/javase/tutorial/java/nutsandbolts/switch.html)
- [深入理解Java的switch...case...语句](https://www.cnblogs.com/wuyuegb2312/p/11172440.html)
- [语言设计者的笔记本 - 首先，不要造成伤害](https://www.ibm.com/developerworks/cn/java/j-ldn2/index.html)
- [深入理解java之关于switch的探究](https://www.debugger.wiki/article/html/1554908400407220)
- [jvm中的switch和String为条件时的思考](https://www.javatt.com/p/49424)