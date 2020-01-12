---
layout: post
title:  "c++从虚表看问题"
date:   2020-01-12 19:37:00
categories: cpp
tags: cpp
excerpt: c++编译器使用虚表来实现类的虚函数的动态多态行为，虚函数的诸多问题需通过虚表解释
mathjax: true
author: superman
---

* content
{:toc}

# 1. 虚表如何植入

* 为了更快的访问速度，c++将虚表地址放在类的起始位置。

* 构造函数在执行完所有基类构造函数之后执行初始化列表之前植入当前类的虚表。析构函数执行用户代码之前植入当前类的虚表。

* vs 通过命令行`/d1 reportAllClassLayout`或者`/d1 reportSingleClassLayout XX`查看所有类布局包括虚表所在，`gdb`使用命令`info vtbl var`查看类实例的虚表。

# 2. 不要在构造或析构函数中调用虚函数

* 构造或析构函数中直接或间接调用虚函数，无多态行为发生，而调用当前类的最终覆盖函数。比如执行基类构造函数时，此时派生类不存在，当前无法调用派生类的虚函数。标准规定：
```
During construction and destruction
When a virtual function is called directly or indirectly from a constructor or from a destructor (including during the construction or destruction of the class’s non-static data members, e.g. in a member initializer list), and the object to which the call applies is the object under construction or destruction, the function called is the final overrider in the constructor’s or destructor’s class and not one overriding it in a more-derived class. In other words, during construction or destruction, the more-derived classes do not exist.
```
* 构造或析构函数中直接或间接调用纯虚函数，编译器一般会警告。构造或析构函数中直接调用纯虚函数，编译器会调用当前类的虚函数，最终链接器会报缺少的符号的错误。

* 构造或析构函数中间接调用纯虚函数，行为未定义。大多编译器会将虚表中对应的纯虚函数地址换成一个提前定义好的函数地址XX(vs为`purecall`, gcc为`cxa_pure_virtual`)，方便hook处理。XX默认会调用`std::abort`结束程序。vs上可使用`_set_purecall_handler`(包含头文件stdlib.h)设置在`std::abort`调用之前调用函数。

# 3. 编译器如何实现构造函数调用虚函数的行为

我们反汇编一段c++代码来看编译器如何实现：
```c++
class A;

void test(A* a);

class A
{
public:
    A()
    {
        // test(this); // ---> 此处会间接调用纯虚函数 程序会崩溃
    }

    virtual ~A()
    {
    }

    virtual void foo() = 0;
};

class B : public A
{
public:
    void foo() override {}
    
    ~B() override
    {
    }
};

void test(A* a)
{
    a->foo();
}

int main()
{
    B a;
}
```

1. vs2019
```c++
// A::A
00007FF7163D2770  mov         qword ptr [rsp+8],rcx  
00007FF7163D2775  push        rdi  
00007FF7163D2776  mov         rax,qword ptr [this]  
00007FF7163D277B  lea         rcx,[A::`vftable' (07FF7163DB110h)]  
00007FF7163D2782  mov         qword ptr [rax],rcx
00007FF7163D2785  mov         rax,qword ptr [this]  
00007FF7163D278A  pop         rdi  
00007FF7163D278B  ret  

// A::~A
00007FF7163D1610  mov         qword ptr [rsp+8],rcx  
00007FF7163D1615  push        rdi  
00007FF7163D1616  mov         rax,qword ptr [this]  
00007FF7163D161B  lea         rcx,[A::`vftable' (07FF7163DB110h)]  
00007FF7163D1622  mov         qword ptr [rax],rcx 

// B::B
00007FF7163D27D0  mov         qword ptr [rsp+8],rcx  
00007FF7163D27D5  push        rdi  
00007FF7163D27D6  sub         rsp,20h  
00007FF7163D27DA  mov         rdi,rsp  
00007FF7163D27DD  mov         ecx,8  
00007FF7163D27E2  mov         eax,0CCCCCCCCh  
00007FF7163D27E7  rep stos    dword ptr [rdi]  
00007FF7163D27E9  mov         rcx,qword ptr [this]  
00007FF7163D27EE  mov         rcx,qword ptr [this]  
00007FF7163D27F3  call        A::A (07FF7163D11BDh)  
00007FF7163D27F8  mov         rax,qword ptr [this]  
00007FF7163D27FD  lea         rcx,[B::`vftable' (07FF7163DB130h)]  
00007FF7163D2804  mov         qword ptr [rax],rcx  
00007FF7163D2807  mov         rax,qword ptr [this]  
00007FF7163D280C  add         rsp,20h  
00007FF7163D2810  pop         rdi  
00007FF7163D2811  ret 

// B::~B
00007FF7163D2840  mov         qword ptr [rsp+8],rcx  
00007FF7163D2845  push        rdi  
00007FF7163D2846  sub         rsp,20h  
00007FF7163D284A  mov         rdi,rsp  
00007FF7163D284D  mov         ecx,8  
00007FF7163D2852  mov         eax,0CCCCCCCCh  
00007FF7163D2857  rep stos    dword ptr [rdi]  
00007FF7163D2859  mov         rcx,qword ptr [this]  
00007FF7163D285E  mov         rax,qword ptr [this]  
00007FF7163D2863  lea         rcx,[B::`vftable' (07FF7163DB130h)]  
00007FF7163D286A  mov         qword ptr [rax],rcx  
00007FF7163D286D  mov         rcx,qword ptr [this]  
00007FF7163D2872  call        A::~A (07FF7163D11D1h)  
00007FF7163D2877  add         rsp,20h  
00007FF7163D287B  pop         rdi  
00007FF7163D287C  ret
```

2. gcc4.8.5
```
00000000004006ea <_ZN1AC1Ev>:
  4006ea:	55                   	push   %rbp
  4006eb:	48 89 e5             	mov    %rsp,%rbp
  4006ee:	48 89 7d f8          	mov    %rdi,-0x8(%rbp)
  4006f2:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  4006f6:	48 c7 00 f0 08 40 00 	movq   $0x4008f0,(%rax)
  4006fd:	5d                   	pop    %rbp
  4006fe:	c3                   	retq   
  4006ff:	90                   	nop

0000000000400700 <_ZN1AD1Ev>:
  400700:	55                   	push   %rbp
  400701:	48 89 e5             	mov    %rsp,%rbp
  400704:	48 83 ec 10          	sub    $0x10,%rsp
  400708:	48 89 7d f8          	mov    %rdi,-0x8(%rbp)
  40070c:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  400710:	48 c7 00 f0 08 40 00 	movq   $0x4008f0,(%rax)
  400717:	b8 00 00 00 00       	mov    $0x0,%eax
  40071c:	85 c0                	test   %eax,%eax
  40071e:	74 0c                	je     40072c <_ZN1AD1Ev+0x2c>
  400720:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  400724:	48 89 c7             	mov    %rax,%rdi
  400727:	e8 44 fe ff ff       	callq  400570 <_ZdlPv@plt>
  40072c:	c9                   	leaveq 
  40072d:	c3                   	retq   

000000000040072e <_ZN1AD0Ev>:
  40072e:	55                   	push   %rbp
  40072f:	48 89 e5             	mov    %rsp,%rbp
  400732:	48 83 ec 10          	sub    $0x10,%rsp
  400736:	48 89 7d f8          	mov    %rdi,-0x8(%rbp)
  40073a:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  40073e:	48 89 c7             	mov    %rax,%rdi
  400741:	e8 ba ff ff ff       	callq  400700 <_ZN1AD1Ev>
  400746:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  40074a:	48 89 c7             	mov    %rax,%rdi
  40074d:	e8 1e fe ff ff       	callq  400570 <_ZdlPv@plt>
  400752:	c9                   	leaveq 
  400753:	c3                   	retq   

0000000000400754 <_ZN1B3fooEv>:
  400754:	55                   	push   %rbp
  400755:	48 89 e5             	mov    %rsp,%rbp
  400758:	48 89 7d f8          	mov    %rdi,-0x8(%rbp)
  40075c:	5d                   	pop    %rbp
  40075d:	c3                   	retq   

000000000040075e <_ZN1BC1Ev>:
  40075e:	55                   	push   %rbp
  40075f:	48 89 e5             	mov    %rsp,%rbp
  400762:	48 83 ec 10          	sub    $0x10,%rsp
  400766:	48 89 7d f8          	mov    %rdi,-0x8(%rbp)
  40076a:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  40076e:	48 89 c7             	mov    %rax,%rdi
  400771:	e8 74 ff ff ff       	callq  4006ea <_ZN1AC1Ev>
  400776:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  40077a:	48 c7 00 b0 08 40 00 	movq   $0x4008b0,(%rax)
  400781:	c9                   	leaveq 
  400782:	c3                   	retq   
  400783:	90                   	nop

0000000000400784 <_ZN1BD1Ev>:
  400784:	55                   	push   %rbp
  400785:	48 89 e5             	mov    %rsp,%rbp
  400788:	48 83 ec 10          	sub    $0x10,%rsp
  40078c:	48 89 7d f8          	mov    %rdi,-0x8(%rbp)
  400790:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  400794:	48 c7 00 b0 08 40 00 	movq   $0x4008b0,(%rax)
  40079b:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  40079f:	48 89 c7             	mov    %rax,%rdi
  4007a2:	e8 59 ff ff ff       	callq  400700 <_ZN1AD1Ev>
  4007a7:	b8 00 00 00 00       	mov    $0x0,%eax
  4007ac:	85 c0                	test   %eax,%eax
  4007ae:	74 0c                	je     4007bc <_ZN1BD1Ev+0x38>
  4007b0:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  4007b4:	48 89 c7             	mov    %rax,%rdi
  4007b7:	e8 b4 fd ff ff       	callq  400570 <_ZdlPv@plt>
  4007bc:	c9                   	leaveq 
  4007bd:	c3                   	retq   

00000000004007be <_ZN1BD0Ev>:
  4007be:	55                   	push   %rbp
  4007bf:	48 89 e5             	mov    %rsp,%rbp
  4007c2:	48 83 ec 10          	sub    $0x10,%rsp
  4007c6:	48 89 7d f8          	mov    %rdi,-0x8(%rbp)
  4007ca:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  4007ce:	48 89 c7             	mov    %rax,%rdi
  4007d1:	e8 ae ff ff ff       	callq  400784 <_ZN1BD1Ev>
  4007d6:	48 8b 45 f8          	mov    -0x8(%rbp),%rax
  4007da:	48 89 c7             	mov    %rax,%rdi
  4007dd:	e8 8e fd ff ff       	callq  400570 <_ZdlPv@plt>
  4007e2:	c9                   	leaveq 
  4007e3:	c3                   	retq   
  4007e4:	66 2e 0f 1f 84 00 00 	nopw   %cs:0x0(%rax,%rax,1)
  4007eb:	00 00 00 
  4007ee:	66 90                	xchg   %ax,%ax
```

# 4. 参考

* <https://en.cppreference.com/w/cpp/language/virtual>