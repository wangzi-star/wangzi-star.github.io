---
layout: post
title:  "c/c++中的枚举类型知多少"
date:   2020-01-11 23:20:00
categories: cpp
tags: cpp enum
excerpt: c/cpp中enum类型是多少字节，我相信绝大部分人都回答不出来
mathjax: true
author: superman
---

* content
{:toc}

# 1. 无作用域枚举

```
enum { ENUM_ITEM1 = $CONST_EXPR0, ... }； // (1)匿名或者无名枚举
enum $ENUM_NAME { ENUM_ITEM1 = $CONST_EXPR0, ... }; //(2) 
enum $ENUM_NAME: $UNDERLY_TYPE { ENUM_ITEM1 = $CONST_EXPR0, ... }; //(3) c++11， 指定枚举底层类型
enum $ENUM_NAME: $UNDERLY_TYPE; //(4)c++11, 枚举类型声明
```

# 2. 有作用域枚举(强类型枚举)

```
enum class|struct $ENUM_NAME { ENUM_ITEM1 = $CONST_EXPR0, ... }; //(5)c++11，强类型枚举
enum class|struct $ENUM_NAME: $UNDERLY_TYPE { ENUM_ITEM1 = $CONST_EXPR0, ... };//(6)c++11，指定枚举底层类型的强类型枚举
enum class|struct $ENUM_NAME;//(7)c++11
enum class|struct $ENUM_NAME: $UNDERLY_TYPE;//(8)c++11
```

# 3. enum 字节大小

* 无作用域枚举和c语言枚举一样，底层类型不固定，由具体平台决定。若枚举项数值不超过`int`表示范围，一般编译器实现底层类型为`int`(比如msvc)或者`unsigned int`。若超过`int`表示范围，则底层类型为能容纳该值的整形类型，比如gcc，底层类型可能为`unsigned long`；而msvc的底层类型只能是`int`，数值溢出按截断处理。

* 强类型枚举，若不指定底层类型(5)，则底层类型为`int`，数值溢出则编译出错。

# 4. gcc特殊行为

* gcc 可通过编译选项`--short_enum`和类型属性`__attribute__((__packed__))`来优化无作用域enum字节大小，根据enum最大枚举值来特殊处理底层类型，可以为`unsigned char`和`unsighed short`。

* `--short_enum`是一种非常危险的编译选项，会导致两个模块无作用域enum字节大小不一致。从这个层面来将，强类型枚举比较安全。

# 5. 参考

[Enumeration declaration](https://en.cppreference.com/w/cpp/language/enum)



