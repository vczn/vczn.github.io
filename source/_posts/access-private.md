---
title: C++ 访问私有成员
date: 2022-05-19 23:18:34
tags:
- C++
categories:
- C++
---



C++ 在类外访问私有成员

<!-- more -->



最近被问到一个问题，在类外如何访问类的私有成员？

比如 `class Foo { char padding; int data; };` 在类外访问私有成员变量  `data`

如果再加上一个条件，对于带有虚函数的类呢？



## 1. `friend` 

正常的方法声明友元函数或友元类，比如我们可以声明 `main` 函数作为类 `Foo` 的友元函数

```cpp
#include <iostream>

int main(int, char **);

class Foo {
  char padding;
  int data = 0;
  friend int ::main(int, char **);
};

int main(int argc, char **argv) {
  Foo foo;
  foo.data = 42; // access private member
  std::cout << foo.data << "\n"; // access private member
  return 0;
}
```



## 2. 显式实例化模板

*ISO/IEC 14882:2014* C++14(C++11 也同样适用) 标准文档第 14.7.2.12 小节对于显式实例化有一条特殊的规则：

> The usual access checking rules do not apply to names used to specify explicit instantiations. [Note: In particular, the template arguments and names used in the function declarator (including parameter types, return types and exception specifications) may be private types or objects which would normally not be accessible and the template may be a member template or member function which would not normally be accessible. — end note ]

直译过来就是：

> 访问限定检测规则不适用于指定在显式实例化中指定的名称

利用这条规则我们可以做一些事情，把访问私有成员放到显式实例化参数上，例如：

```c++
#include <iostream>

class Foo { char padding; int data = 0; };

template <typename T, typename Obj, T Obj::* Ptr>
struct Access {
  friend T &getData(Obj &obj) { return obj.*Ptr; }
};

// explicit instantiation
template struct Access<int, Foo, &Foo::data>;
int &getData(Foo &foo);

int main(int argc, char **argv) {
  Foo foo;
  getData(foo) = 42; // access private member
  std::cout << getData(foo) << "\n"; // access private member
  return 0;
}
```
