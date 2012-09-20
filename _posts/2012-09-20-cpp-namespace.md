---
layout: default
published: true
---

# 命名空间小记
  
 一直以为在.h里面写 namespace xxxx \{ …… \};， 在.cpp里面要在每个函数或者类前直接加命名空间全称，原来不是，在.cpp文件里面也可以像.h里面那样写。而且！对于gcc编译的模板特化，只有这样才能通过编译，否则总是提特化跟模板不在同一个命名空间里面，例如：  

    .h
    namespace A {
    template<typename T>
    void Func(T t);
    };

    .cpp
    template<>
    void A::Func(int n) {
     ……
    }  

会触发编译错误，必须是：  
    .cpp
    namespace A {
    template<>
    void Func(int n) {
     ...
    }
    };

