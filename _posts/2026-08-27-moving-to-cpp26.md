---
layout: post
title: 迈向C++26
date: 2026-08-27 13:31 +0800
categories: [技术经验, 现代C++]
tags: [C++] # TAG 名称应始终为小写
---

参考：
1. [Moving to C++26: How to Build and Set Up GCC 15.1 on Ubuntu](https://medium.com/@xersendo/moving-to-c-26-how-to-build-and-set-up-gcc-15-1-on-ubuntu-f52cc9173fa0)
2. [CPP23](https://zh.cppreference.com/cpp/23)
3. [CPP26](https://zh.cppreference.com/cpp/26)

本文不多讲如何安装GCC或者其他支持最新标准的编译器。主要是一些新标准的理解。
代码样例来自于参考文章1.

许多人*声称*：cppreference并非官方委员会的内容，而是社区维护的，并为了`main`函数到底该不该写上`return 0;`而争论不休。本文无意讨论这些，就像人类说话最重要的是表达清晰有逻辑，而非每个用词都和字典一样精准。

```cpp
#include <iostream>
#include <format>

// ===================================================
// Feature 1: Pack Indexing
// ===================================================

template<typename... Types>
void PrintFirstArg(Types... Args)
{
    std::cout << std::format("{}\n\n", Args...[0]);
}

void TestPackIndexing()
{
    puts("=== Feature 1: Pack Indexing ===");
    PrintFirstArg('a', "two", 33);
}

// ===================================================
// Feature 2: Attributes for Structured Bindings
// ===================================================

void TestAttributesForStructuredBindings()
{
    puts("=== Feature 2: Attributes for Structured Bindings ===");
    struct StructS1 { int A = 10, B = 20; } S;
    auto [A [[maybe_unused]], B [[deprecated]]] = S;
    std::cout << std::format("A = {}, B = {}\n\n", A, B);
}

// ===================================================
// Feature 3: Variadic Friends
// ===================================================

template<typename... Ts>
class Passkey
{
    friend Ts...;
    Passkey() { }
};

class ClassA;
class ClassB;

struct Widget
{
    void Secret(Passkey<ClassA, ClassB>) const
    {
        puts("Secret accessed!");
    }
};

struct ClassA
{
    void Access(const Widget& W) const { W.Secret({}); } 
};

struct ClassB
{
    void Access(const Widget& W) const { W.Secret({}); }
};

struct ClassD
{
    // Compilation error: ClassD is not a friend
    // void Access(Widget& W) { W.Secret({}); }
};

void TestVariadicFriends()
{
    puts("=== Feature 3: Variadic Friends ===");
    Widget W;
    ClassA A, B;
    A.Access(W);
    B.Access(W);
    puts("");
}

// ===================================================
// Feature 4: Constexpr Placement New
// ===================================================

constexpr int ConstexprPlacementNew()
{
    std::allocator<int> Allocator;
    int* P = Allocator.allocate(16);
    new (P) int(33);
    int Value = *P;
    Allocator.deallocate(P, 16);
    return Value;
}

void TestConstexprPlacementNew()
{
    puts("=== Feature 4: Constexpr Placement New ===");
    constexpr int Result = ConstexprPlacementNew();
    std::cout << std::format("Value constructed with placement new: {}\n\n", Result);
}

// ===================================================
// Feature 5: Structured Binding as Condition
// ===================================================

struct StructS2
{
    int A, B;
    explicit operator bool() const noexcept
    {
        return A != B;
    }
};

void Use(int A, int B)
{
    std::cout << std::format("Using A = {}, B = {}\n", A, B);
}

void TestStructuredBindingAsCondition()
{
    puts("=== Feature 5: Structured Binding as Condition ===");
    StructS2 S{10, 20};
    if (auto [A, B] = S)
    {
        Use(A, B);
    }
    puts("");
}

// ===================================================
// Feature 6: Deleted Function with Reason
// ===================================================

void OldFunction(char*) = delete("unsafe, use NewFunction instead");
void NewFunction(char*) { }

void TestDeletedFunctionWithReason()
{
    // error: use of deleted function ‘void OldFunction(char*)’: unsafe, use NewFunction instead
    // OldFunction("test");
}

// ===================================================
// Feature 7: Deleting Pointer to Incomplete Type
// ===================================================

struct IncompleteType;

void TestDeletingIncompleteType(IncompleteType* Pointer)
{
    // delete Pointer; // Compilation error in C++26
}

// ===================================================
// Feature 8: Oxford Variadic Comma
// ===================================================

void DeprecatedFunction(auto......) { } // omission of ‘,’ before varargs ‘...’ is deprecated in C++26
void OkayFunction(auto..., ...) { }

void DeprecatedG(int...) { } // Deprecated in C++26
void OkayG(int, ...) { }

void DeprecatedH(int X...) { } // Deprecated in C++26
void OkayH(int X, ...) { }

// ===================================================
// Main Function
// ===================================================

int main()
{
    TestPackIndexing();
    TestAttributesForStructuredBindings();
    TestVariadicFriends();
    TestConstexprPlacementNew();
    TestStructuredBindingAsCondition();
    TestDeletedFunctionWithReason();
    TestDeletingIncompleteType(nullptr);
}
```