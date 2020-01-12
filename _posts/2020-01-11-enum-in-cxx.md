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
enum { A = 1, B = 2, ... }; // (1)匿名枚举
enum EnumA {  A = 1, B = 2, ... }; //(2) 具名枚举
enum EnumA: int { A = 1, B = 2, ... }; //(3) c++11， 指定枚举底层类型

// 使用枚举
void foo(Enum A);

foo(A);
```

# 2. 有作用域枚举(强类型枚举)

```
enum class|struct EnumA { A = 1, B = 2, ... }; //(4)c++11，强类型枚举
enum class|struct EnumA: uint8_t { A = 1, B = 2, ...}; //(5)c++11，指定枚举底层类型的强类型枚举

// 使用枚举
void foo(EnumA);

foo(EnumA::A);
```

# 3. enum 字节大小

* c++11提供`underlying_type`获取enum的底层类型。

* 无作用域且不指定底层类型的枚举和c语言枚举一样，底层类型不固定，由具体平台决定。若枚举项数值不超过`int`表示范围，一般编译器实现底层类型为`int`(比如msvc)或者`unsigned int`。若超过`int`表示范围，则底层类型为能容纳该值的整形类型，比如gcc，底层类型可能为`unsigned int`；而msvc的底层类型只能是`int`，数值溢出按截断处理。

* 强类型枚举，若不指定底层类型(4)，则底层类型为`int`，数值溢出则编译出错。

# 4. gcc特殊行为

* gcc 可通过编译选项`-fshort-enums`和类型属性`__attribute__((__packed__))`来优化无作用域enum字节大小，根据enum最大枚举值来特殊处理底层类型，可以为`unsigned char`和`unsighed short`。

* `-fshort-enums`是一种非常危险的编译选项，会导致两个模块无作用域enum字节大小不一致。从这个层面来将，强类型枚举比较安全。

# 5. 获取enum底层类型代码

```c++
#include <iostream>
#include <typeinfo>
#include <type_traits>

enum EnumA
{
    A = 1,
};

int main()
{
    std::clog << typeid(std::underlying_type<EnumA>::type).name() << std::endl;
}

// gcc 打印类型之后，使用c++filt -t查看类型名
```

# 6. 参考

* <https://en.cppreference.com/w/cpp/language/enum>



