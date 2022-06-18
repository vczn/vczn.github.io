---
title: dyn_cast
date: 2022-06-16 22:01:08
tags: 
 - C++
 - LLVM
 - Type
categories:
 - C++
---





分析 LLVM `isa` 和 `dyn_cast` 实现原理

<!-- more -->



## 引言

在 C++ 中，`dynamic_cast` 是四种类型转换运算符之一，表达式 `dynamic_cast<T>(v)` 的结果是将 `v` 转换为 `T` 类型的结果。`T` 应该是一个完整类型的指针或引用，或是可以带有 `cv` 限定的 `void` 指针，类型转换不应丢弃限定 `const`。

`dynamic_cast` 主要用来表示动态类型转换语义，沿继承层级向上、向下以及侧向(菱形继承)，安全转换到其他类型的指针或引用。如果转型失败，`T` 是指针则返回 `nullptr`，`T` 是引用则抛出 `std::bad_cast` 异常。

但是某些形式的 `dynamic_cast` 需要检查表达式的动态类型信息(依赖 RTTI)，会造成运行时开销，而且开启 RTTI 会导致二进制文件变大。在 LLVM/Clang 中，为了减小二进制体积，关闭了 RTTI，但是仍然有动态类型转型的需求，LLVM 实现了 `llvm::dyn_cast` 来进行动态类型转换。



## 原理

LLVM 中的 `isa` 和 `dyn_cast` 并不是通用的类型判断以及类型转换，对于大多数的类型我们不需要生成 RTTI 信息，仅在需要类型信息的继承层级中添加即可。

考虑以下的继承层级：

```cpp
struct Stmt {};
struct ReturnStmt : Stmt {};
struct Expr : Stmt {};
struct UnaryOperator : Expr {};
struct BinaryOperator : Expr {};
```

我们需要拿到该继承层级下某个对象的动态类型，可以给这个继承体系添加一个字段。比如：

```cpp
struct Stmt {
  enum StmtClass {
    ReturnStmtClass,
    UnaryOperatorClass,
    BinaryOperatorClass,
    firstExprClass = UnaryOperatorClass,
    lastExprClass = BinaryOperatorClass,
  };

  unsigned stmtClass;

  Stmt(StmtClass SC) : stmtClass(SC) {}
  StmtClass getStmtClass() const {
    return static_cast<StmtClass>(stmtClass);
  }
};
```

我们可以通过 `getStmtClass` 方法获取某个对象的运行时类型标识 ID，在继承体系 isa 语义下，给定某个类型 `StmtT` 和 某个对象 `S`，为了 `S` 的动态类型是否为 `StmtT`，我们可以给各个派生类添加 `classof` 方法

```cpp
struct ReturnStmt : Stmt {
  ReturnStmt() : Stmt(ReturnStmtClass) {}
  static bool classof(const Stmt *S) {
    return S->getStmtClass() == ReturnStmtClass;
  }
};

struct Expr : Stmt {
  Expr(StmtClass SC) : Stmt(SC) {}
  static bool classof(const Stmt *S) {
    return firstExprClass <= S->getStmtClass() &&
           S->getStmtClass() <= lastExprClass;
  }
};

struct UnaryOperator : Expr {
  UnaryOperator() : Expr(UnaryOperatorClass) {}
  static bool classof(const Stmt *S) {
    return S->getStmtClass() == UnaryOperatorClass;
  }
};

struct BinaryOperator : Expr {
  BinaryOperator() : Expr(BinaryOperatorClass) {}
  static bool classof(const Stmt *S) {
    return S->getStmtClass() == BinaryOperatorClass;
  }
};
```

然后使用下面的测试用例进行测试：

```cpp
UnaryOperator UO;
std::cout << "UO is a ReturnStmt: " << ReturnStmt::classof(&UO) << "\n";         // 0
std::cout << "UO is a Expr: " << Expr::classof(&UO) << "\n";                     // 1
std::cout << "UO is a UnaryOperator: " << UnaryOperator::classof(&UO) << "\n";   // 1
std::cout << "UO is a BinaryOperator: " << BinaryOperator::classof(&UO) << "\n"; // 0
```

这样我们就可以在运行时判断对象动态类型了



## 简化版本

接下来我们先自己实现一个简化版本的 `isa` 和 `dyn_cast`

我们对 `isa` 进行简化，`isa<To>(fromVal)` 如果 `To::classof(fromVal)` 或 `To` 是 `From` 的基类则返回 `true`，否则返回 `false`。我们这里不考虑传入参数类型，假定 `To` 为非指针，非引用类型，而 `From` 是一个指针类型。

```cpp
template <typename To, typename From>
inline bool isa(const From& Val) {
  return std::is_base_of<To, From>::value || To::classof(Val);
}

// test
std::cout << "UO is a ReturnStmt: " << isa<ReturnStmt>(&UO) << "\n";
std::cout << "UO is a Expr: " << isa<Expr>(&UO) << "\n";
std::cout << "UO is a UnaryOperator: " << isa<UnaryOperator>(&UO) << "\n";
std::cout << "UO is a BinaryOperator: " << isa<BinaryOperator>(&UO) << "\n";
```

和 `isa` 类似， `dyn_cast` 暂时不考虑参数类型和返回类型，返回类型固定为 `const To *`

```cpp
template <typename To, typename From>
inline const To* dyn_cast(const From& Val) {
  return isa<To>(Val) ? static_cast<const To*>(Val) : nullptr;
}
```



基本原理到这里就完结了，但是上面的版本没有考虑各种参数的情况，因为 `llvm` 实现比较复杂，而且实现比较类似，我们仅对 `llvm::isa` 进行一下源码分析

## LLVM 实现

首先看 `isa` 定义

```C++
template <class X, class Y> LLVM_NODISCARD inline bool isa(const Y &Val) {
  return isa_impl_wrap<X, const Y,
                       typename simplify_type<const Y>::SimpleType>::doit(Val);
}

template <typename First, typename Second, typename... Rest, typename Y>
LLVM_NODISCARD inline bool isa(const Y &Val) {
  return isa<First>(Val) || isa<Second, Rest...>(Val);
}
```

`isa` 第二个版本主要为了使用更方便一些，我们继续看 `isa_impl_wrap`。

```cpp
template<typename To, typename From, typename SimpleFrom>
struct isa_impl_wrap {
  // When From != SimplifiedType, we can simplify the type some more by using
  // the simplify_type template.
  static bool doit(const From &Val) {
    return isa_impl_wrap<To, SimpleFrom,
      typename simplify_type<SimpleFrom>::SimpleType>::doit(
                          simplify_type<const From>::getSimplifiedValue(Val));
  }
};

template<typename To, typename FromTy>
struct isa_impl_wrap<To, FromTy, FromTy> {
  // When From == SimpleType, we are as simple as we are going to get.
  static bool doit(const FromTy &Val) {
    return isa_impl_cl<To,FromTy>::doit(Val);
  }
};
```

这里 `isa_impl_wrap` 只是为了特殊处理 `const From`，最终都会走到 `is_impl_cl` 上，`is_impl_cl` 分别处理第二个模板参数为 `From, const From, std::unique<From>, From*, From *const, const From *, const From *const` 的情况

```cpp
template <typename To, typename From> struct isa_impl_cl {
  static inline bool doit(const From &Val) {
    return isa_impl<To, From>::doit(Val);
  }
};

template <typename To, typename From> struct isa_impl_cl<To, const From> {
  static inline bool doit(const From &Val) {
    return isa_impl<To, From>::doit(Val);
  }
};

template <typename To, typename From>
struct isa_impl_cl<To, const std::unique_ptr<From>> {
  static inline bool doit(const std::unique_ptr<From> &Val) {
    assert(Val && "isa<> used on a null pointer");
    return isa_impl_cl<To, From>::doit(*Val);
  }
};

template <typename To, typename From> struct isa_impl_cl<To, From*> {
  static inline bool doit(const From *Val) {
    assert(Val && "isa<> used on a null pointer");
    return isa_impl<To, From>::doit(*Val);
  }
};

// From *const, const From *, const From *const
```

最后就是具体的实现部分，如果 `To` 是 `From` 的基类或 `To::classof(&Val)`，则为 `true`

```cpp
template <typename To, typename From, typename Enabler = void>
struct isa_impl {
  static inline bool doit(const From &Val) {
    return To::classof(&Val);
  }
};

/// Always allow upcasts, and perform no dynamic check for them.
template <typename To, typename From>
struct isa_impl<To, From, std::enable_if_t<std::is_base_of<To, From>::value>> {
  static inline bool doit(const From &) { return true; }
};
```

