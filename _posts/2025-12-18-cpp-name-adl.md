---
layout: post
title: C++的名称查找与参数依赖查找(ADL)
date: 2025-12-18 21:44 +0800
categories: [技术经验, 现代C++]
tags: [C++] # TAG 名称应始终为小写
mermaid: true
---

经常看到ADL这个名词，之前一直不知道它的意思，后来才知道是参数依赖查找。于是想彻底弄懂C++的名称查找，浅薄之见留在这里，供参考。如有错误或纰漏敬请谅解，可以去[我网站的github](https://github.com/Mike-Sagiri/myblogs)提issue。

参考资料：
1. [csdn](https://blog.csdn.net/weixin_43862847/article/details/132281951)，一篇很好的文章，之前还不是vip的。
2. [知乎](https://zhuanlan.zhihu.com/p/338917913)，讲ADL不错。
3. 自己动手试试！但是注意，本文代码均基于gcc-14,c++23。

## 基本概念
1. * **qualified name：** 一个名称所属的作用域被显式的指明，例如`::`、`->`或者`.`。`this->count`就是一个 qualified name，但 `count` 不是，因为它的作用域没有被显示的指明，即使它和this->count是等价的。
    * **dependent name：**依赖于模板参数的名称，也就是访问运算符左面的表达式类型依赖于模板参数。例如：`std::vector<T>::iterator` 是一个 Dependent Name，但假如 `T` 是一个已知类型的别名（`using T = int`），那就不是 Dependent Name。
2. * **限定作用符：** `::`
    * **限定名**：指出现在`::`(限定作用符)右侧的名字，他可以是：类、命名空间、枚举，也就是qualified name。
    * **无限定作用域**：没有限定作用符在左侧的名字

## 基本规则
1. 如果`::`限定作用符左侧留空，只会在全局命名空间查找；
2. 存在多级限定作用符级联时，只有对左侧名查找完了，再在找到的作用域里对右侧进行查找；有限定和无限定和可以混用
3. 对`::`左侧的符号进行查找，会忽略变量、函数、枚举项的声明
4. 当在声明符中使用限定名，该声明符中的非限定名查找在限定名的类或者命名空间中查找
5. 无限定查找，查找之前声明的部分，如果存在命名空间嵌套则会一级一级往外查找
6. 友元声明的无限定名查找（非模板实参），先在友元的限定域里面查找，再在友元的声明所在域查找
7. 模板中的非依赖名，立即绑定，并且模板的非限定名查找只可见模板定义点前的命名
8. 如果某个基类取决于某个模板形参，那么无限定名字查找不会检查它的作用域
9. 非限定名的ADL

我们一个一个规则来看。

### 规则1
> 如果`::`限定作用符左侧留空，只会在全局命名空间查找；
{: .prompt-info }

```cpp
#include <iostream>
#include <print>

namespace X {
    int max(int a, int b){
        std::println("X::max called!");
        return 0;
    }
}

struct Y {
    int max(int a, int b){
        std::println("Y::max called!");
        return 0;
    }
};

int max(int a, int b){
    std::println("::max called!");
    return 0;
}

int main(){
    int max = 0;
    ::max(1,2);//规则1，输出::max called
}
```

### 规则2
> 存在多级限定作用符级联时，只有对左侧名查找完了，再在找到的作用域里对右侧进行查找；有限定和无限定和可以混用
{: .prompt-info }

```cpp
int main()
{
	struct std {};
	::std::cout << "\n"; // 规则1，忽略std{}, 规则2 多级限定
}
```

### 规则3
> 对`::`左侧的符号进行查找，会忽略变量、函数、枚举项的声明
{: .prompt-info }

```cpp
namespace A{
    int n = 1;
}

int main(){
    int A = 0;
    A::n = 2; // 规则3，忽略变量A，查找到命名空间的A。A是无限定名，忽略变量A的定义，n是限定名
}
```

### 规则4
> 当在声明符中使用限定名，该声明符中的非限定名查找在限定名的类或者命名空间中查找
{: .prompt-info }

```cpp
const int DD = 100;
struct N{
    static const int DD = 50;
    static int S;
};

int N::S {DD};

int main(){
    std::cout<<N::S<<std::endl;// 规则4 输出50，使用N::DD。因为int N::S {DD}是进行初始化，在声明符中使用。
    N::S=DD;
    std::cout<<N::S<<std::endl;// 输出100，会用到下面的规则5，也就是非限定名的查找。
}
```

### 规则5
> 无限定查找，查找之前声明的部分，如果存在命名空间嵌套则会一级一级往外查找
{: .prompt-info }

```cpp
int n = 1;
namespace N{
    int m = 2;
    namespace Y{
        int x = n; // 规则5，全局的n，等价于int x = ::n;
        int y = m; // 规则5，N::m。
    };
};

namespace X {
    int max(int a, int b){
        std::println("X::max called!");
        return 0;
    }
}

struct Y {
    int max(int a, int b){
        std::println("Y::max called!");
        return 0;
    }
};

int max(int a, int b){
    std::println("::max called!");
    return 0;
}

int main(){
    int max = 0;
    {
    int max = 2;
    ::max(1,2); //规则1，输出::max called
    std::cout<<max<<std::endl; //规则5，输出2，找的的是本代码块的max。
    }
}
```

### 规则6
> 友元声明的无限定名查找（非模板实参），先在友元的限定域里面查找，再在友元的声明所在域查找
{: .prompt-info }

补充几个基本概念
1. **模板形参：**分为类型形参和非类型形参。
```cpp
template<typename T>        // 类型形参
template<int N>             // 非类型形参
```
2. **模板实参：**实际传给模板的参数。
```cpp
A<double, 10> a;
```
3. **声明**：声明说明符 + 声明符
```cpp
int x;// 声明说明符：int 声明符：x
```
```cpp
int* (*f)(double);//声明说明符：int 声明符：*(*f)(double)
//声明符描述了：f 是一个指向函数的指针，该函数接收 double，返回 int*
```
4. **声明符的标识符**：声明符的标识符是声明符中真正被引入到作用域中的那个名字。
```cpp
int* (*f)(double);//声明符的标识符：f
```

现在来看例子：
```cpp
template<class T>
struct S{

};

struct A
{
    typedef int AT;
 
    void f1(AT);
    void f2(float);
 
    template<class T>
    void f3();
 
    void f4(S<AT>);
};
 
// 这个类为 f1，f2 和 f3 授予友元关系
struct B
{
    typedef char AT;
    typedef float BT;
 
    friend void A::f1(AT);    // 规则6 对 AT 的查找找到的是 A::AT（在 A 中找到 AT）
    friend void A::f2(BT);    // 规则6 对 BT 的查找找到的是 B::BT（在 A 中找不到 AT）
    friend void A::f3<AT>();  // 规则6 对 AT 的查找找到的是 B::AT （不在 A 中进行查找，
                              //     因为 AT 在声明符中的标识符 A::f3<AT> 中）
};
 // 这个类模板为 f4 授予友元关系
template<class AT>
struct C
{
    friend void A::f4(S<AT>); //规则6 对 AT 的查找找到的是 A::AT 
                              // （AT 不在声明符中的标识符 A::f4 中）
};
```

### 规则7
> 模板中的非依赖名，立即绑定，并且模板的非限定名查找只可见模板定义点前的命名
{: .prompt-info }

这里也有叫待决名的，我个人认为和依赖名是一回事，都是依赖于模板参数的名。

```cpp
#include <iostream>

void f(char); // f 的第一个声明
 
template<class T> 
void g(T t)
{
    f(1);    //  规则7 非依赖名：名字查找找到了 ::f(char) 并在此时绑定
    f(t);    // 依赖名：查找推迟（但是只可见定义点之前的声明）
//  dd++;    // 非依赖名：名字查找未找到声明（因为只可见之前的定义）
}
 
void f(int); // 函数模板g不可见
double dd;
 
void h()
{
 
    g(32); // 实例化 g<int>，此处
           // 对 'f' 的第二次使用
           // 进行了查找仅找到了 ::f(char)
           // 然后重载解析选择了 ::f(char)
           // 这两次调用了 f(char)，规则7 ，不可见f(int)
           
}

void f(char x){
    std::cout<<"char f called"<<std::endl;
}

void f(int x){
    std::cout<<"int f called"<<std::endl;
}

int main(){
    h(); // 应该输出两次char f called
}
```

### 规则8
> 如果某个基类取决于某个模板形参，那么无限定名字查找不会检查它的作用域
{: .prompt-info }

```cpp
typedef double AA;
 
//template<class T>
struct BB
{
    typedef int AA;
};
 
template<class T>
struct XX : BB//<T>
{
    AA a = 3.3; //  规则8  对 AA 的查找找到了 BB::AA ，而不是 ::AA (double)。因为此时基类BB不依赖模板形参
};


int main(){
    std::cout<<XX<int>().a<<std::endl;// 输出3
}
```

```cpp
typedef double AA;
 
template<class T>
struct BB
{
    typedef int AA;
    // typedef T AA;
};
 
template<class T>
struct XX : BB<T>
{
    AA a = 3.3; //  规则8  对 AA (double) ，而不是 BB<T>::AA。
                // 此时基类依赖于模板形参，如果检查BB的作用域，会使得类型绑定是隐式依赖的，AA的解析到底是一个普通非限定名，还是依赖于基类模板参数的。
                // 目的是为了避免隐式依赖造成的不确定性。
                // 流程是：AA是非限定名->需要立即进行名字查找并绑定->依赖基类不能保证AA良定义
};


int main(){
    std::cout<<XX<int>().a<<std::endl;// 输出3.3
}
```

```cpp
typedef double AA;
 
template<class T>
struct BB
{
    typedef int AA;
};
 
template<class T>
struct XX : BB<T>
{
    BB<T>::AA a = 3.3; //  此时AA是正常的限定名，自然应该去BB<T>里寻找。同时显示指明了它是依赖于模板参数T的。
};


int main(){
    std::cout<<XX<int>().a<<std::endl;// 输出3
}
```

```cpp
typedef double AA;
 
template<int T>
struct BB
{
    typedef int AA;
};
 
template<class T>
struct XX : BB<10>
{
    AA a = 3.3; //  这里基类是非类型模板参数，所以实际上它的AA定义是确定的，也不存在隐式依赖。即BB不再是依赖基类。
};


int main(){
    std::cout<<XX<int>().a<<std::endl;// 输出3
}
```
```cpp
typedef double AA;
 
template<class T>
struct BB
{
    typedef int AA;
};
 

struct XX : BB<double>
{
    AA a = 3.3; //  同上，这里BB<double>是一个明确的类模板实例化后的类，不是依赖基类。因此AA找到了BB<double>里的。
};


int main(){
    std::cout<<XX().a<<std::endl;// 输出3
}
```

理解原因比较复杂，总而言之，依赖基类不保证良定义。请看下面的例子：
```cpp
template<class T>
struct BB;              // 主模板，不定义 AA

template<>
struct BB<int> {
    typedef int AA;     // 只有 int 才有 AA
};

template<>
struct BB<double> {
    // 没有 AA
};
```

这种情况下，继承了类模板的派生类如果用`AA`，编译器无法在定义时检查类型，只有派生类实例化的时候才知道，那么不能保证派生类是良定义的。现代C++中的`concept`可以施加约束，来保证`AA`良定义，但这一切发生在模板实例化期间，而名字查找是一切的基础。如果等到实例化再进行名字查找，实现起来非常复杂（但不是不可能实现）。注意区分“绑定”和“查找”。绑定是可以延迟的，但是查找名字是谁以及是否存在，是不能延迟的。

### 规则9
> 非限定名的ADL（参数依赖查找）
{: .prompt-info }

简而言之，**函数**名字的查找，根据它的参数类型进行。结合下面的例子：
```cpp
#include <iostream>
int main()
{
    std::cout << "测试\n"; // 全局命名空间中没有 operator<<，但 ADL 检验 std 命名空间，
                           // 因为左实参在 std 命名空间中
                           // 并找到 std::operator<<(std::ostream&, const char*)
    operator<<(std::cout, "测试\n"); // 同上，用函数调用记法
 
    // 然而，
    std::cout << endl; // 错误：'endl' 未在此命名空间中声明。
                       // 这不是对 endl() 的函数调用，所以不适用 ADL
 
    endl(std::cout); // OK：这是函数调用：ADL 检验 std 命名空间，
                     // 因为 endl 的实参在 std 中，并找到了 std::endl
 
    (endl)(std::cout); // 错误：'endl' 未在此命名空间声明。
                       // 子表达式 (endl) 不是函数调用表达式
}
```

如果没有参数依赖查找，模板max无法找到后定义的类的函数（规则7），就无法调用类重载的运算符了。
```cpp
template<typename T>
T max (T a, T b) {
    return b < a ? a : b;
}

namespace BigMath {
  class BigNumber {
    ...
};

  bool operator < (BigNumber const&, BigNumber const&);  //对模板 MAX 不可见，若无ADL
  ...
}

using BigMath::BigNumber;

void g (BigNumber const& a, BigNumber const& b) {
  ...
  BigNumber x = ::max(a,b);
  ...
}
```

### 忽略ADL的情况

#### 1、成员函数声明
```cpp
namespace NS1{
    using A = char;
    void foo(A){
        std::println("NS1::foo");
    }
}

class MyClass {
    public:
        void foo(int) {
            // 不考虑参数类型的命名空间，直接在类的作用域中查找
            std::cout << "MyClass::foo" << std::endl;
        }
    };
    
int main(){
    NS1::A a;
    MyClass obj;
        
    obj.foo(a); // 不会触发ADL，直接在 MyClass 的作用域中查找 foo，因此输出MyClass::foo
}
```
#### 2、块作用域中的函数声明（非using-declaration）
```cpp
namespace NS1 {
    using A = char;
    void foo(A) {
        std::cout << "NS1::foo" << std::endl;
    }
}

int main() {
    void foo(int); // 在 main 函数作用域内声明一个函数
    NS1::A a = 1;
    foo(a); // 不会触发ADL，直接在 main 函数作用域内查找 foo，因此输出main,foo
}

void foo(int){
    std::cout<<"main,foo"<<std::endl;
}
```
什么是using declaration呢，请看下面，X::f会被ADL忽略。
```cpp
#include <iostream>

namespace X {
  template <typename T> void f(T);
}

namespace N {
  using namespace X;
  enum E { e1 };
  void f(E) { std::cout << "N::f(N::E) called\n"; }
}    // namespace N

void f(int) { std::cout << "::f(int) called\n"; }

int main() {
  ::f(N::e1);    // qualified function name: no ADL
  f(N::e1);     // ordinary lookup finds ::f() and ADL finds N::f(), the latter is preferred, but X::f ignored
}
```
#### 3、非函数、函数模板的声明（重载了`()`运算符）
```cpp
namespace NS1 {
    struct MyStruct {
        void operator()() {
            std::cout << "MyStruct operator()" << std::endl;
        }
    };
}

int main() {
    NS1::MyStruct foo; // 函数对象

    foo(); // 不会触发ADL，直接在当前作用域内查找 foo
}
```