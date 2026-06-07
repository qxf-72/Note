# auto

## lambda 表达式初探

**lambda 表达式底层实现原理**：编译器构建一个类——可调用的类，并且重载了（）运算符。其实就是仿函数。

```cpp
size_t sz=0;
auto SizeComp=[sz](const string& a)->bool{return a.size()>sz; };

//编译器构造的类
class SizeComp {
public:
    SizeComp(size_t sz) : sz(sz)
    {}
    bool operator()(const string &a) const
    {
        return a.size() > sz;
    }
private:
    size_t sz;
};
```


**lambda 表达式基础语法**：`[捕获列表](参数列表)可选限定符 异常 -> 返回类型 {函数体}`
- 捕获列表：
	- 广义的捕获（since C++14）：捕获列表可以传入右值 `[r=std::move(x)]`
	- 有一些特殊用法
		- `[this]`：当 lambda 表达式写在类之中时，捕获 this 指针，可以使用 this 类的变量和函数。
		- `[=]`：值捕获全部局部变量，包括 this 指针。
		- `[&]`
- 参数列表：C ++14 之后可以使用 `auto`
	- `auto fun=[](auto a){return a;};`
- 可选限定符：默认是 `const`，可以写 `mutable`，此时可以修改值捕获的变量。
- 异常：`noexcept` 表示不会抛异常


---

## 可调用类型

**lambda 表达式存在的意义**
- 闭包：带有上下文信息的函数
- 闭包的实现方式：重载()；lambda 表达式：std::bind

有一些函数的参数列表不可改变，例如 STL 泛型算法中提供的可调用对象参数，但是可能需要带有上下文信息，此时单纯的函数或者函数指针很难实现。

**可调用对象与 function**
- C++的 5 种可调用对象：函数、函数指针、lambda 表达式、bind 对象、仿函数。
	- 函数和函数指针的区别：函数是代码实体，函数指针是指向代码段的地址。
	- <div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-06-05_16-02-48.png" loading="lazy" style="max-width: 100%; height: auto; width: 600px;"/> </div>
	- 
- 类型擦除：以上 5 种类型都不一样，但是都可以塞入 function 之中。




----

## 条款 5：优先考虑 `auto` 而非显式声明

### 优点

**变量必须初始化**：强迫编程者初始化变量，否则无法进行类型推导。

**避免写又长又复杂的类型**：比如容器的迭代器类型。

**避免手写类型出错**

```cpp
std::vector<int>::size_type
```
在一些平台上，它可能是 64 位无符号整数，而 `unsigned` 可能只有 32 位。使用 `unsigned` 去接可以会出错。

**避免关联容器中的隐藏拷贝**

```cpp
std::unordered_map<std::string, int> m;
for (const std::pair<std::string, int>& p : m) {
    // ...
}
```

因为 key 是 `const std::string`，所以编译器可能会创建一个临时对象，然后让引用绑定到这个临时对象上。

**`auto` 对 lambda 特别重要**：lambda 表达式的真实类型是编译器生成的匿名类型，根本写不出来。

---

### 注意点

**`auto` 默认会丢掉顶层 const 和引用** ：需要显式写出 `const auto&`

---

### 常见场景

- 普通变量
- 迭代器
- lambda
- 范围 for 语句：`auto&`
- 指针指针：`auto p=std::make_shared<int>(10)`


---

## 条款 6：auto 推导若非己愿，使用显式类型初始化器惯用法


**auto 会推导“真实类型”，但真实类型不一定是你想要的**

`std::vector<bool>`：为了节省空间，不会把每个 bool 存成一个字节，而是把多个 bool 压缩到 bit 位里面。

```cpp
std::vector<bool> flags{true, false, true};
auto b = flags[0];
b = false;
```

b 并不是一个独立 bool 副本，而是一个**代理对象**，仍然关联着 `flags[0]`， 表现得像 `bool&`，但实际上不是 `bool&`。

**显式类型初始化器惯用法**

```cpp
auto x = static_cast<T>(expr);
```


---


# 移步现代 C++

## 条款 7：区别使用 `{}` 和 `()` 创建对象

**`{}` 初始化更统一，更安全，但它会优先匹配 `std::initializer_list` 构造函数，有时会导致意料之外的结果。**

**`{}` 初始化的优点**
- 适用范围更广
- 防止窄化转换
- 避免“最令人头疼的解析”

```cpp
class Widget {};
Widget w();
```

这是并不是创建了一个 `Widget` 对象。实际上，它声明了一个函数。解决方法是使用 `{}` 替代 `()`。


**`{}` 会优先匹配 `std::initializer_list` **

```cpp
#include <initializer_list>
#include <iostream>

class Widget {
public:
    Widget(int i, bool b) {
        std::cout << "Widget(int, bool)\n";
    }

    Widget(int i, double d) {
        std::cout << "Widget(int, double)\n";
    }

    Widget(std::initializer_list<long double> il) {
        std::cout << "Widget(initializer_list<long double>)\n";
    }
};
```

```cpp
Widget w1(10, true);  // Widget(int, bool)
Widget w2{10, true};  // Widget(initializer_list<long double>)

Widget w3(10, 5.0);   // Widget(int, double)
Widget w4{10, 5.0};   // Widget(initializer_list<long double>)
```


---

## 条款 8：优先使用 `nullptr` 而不是 `0` 和 `NULL`

`nullptr` 是**真正的空指针字面量**，而 `0` 和 `NULL` 本质上更像整数。

所以在重载函数和模板推导问题中，会将 `NULL` 推导为 int 或者 long 类型，只有 nullptr 会被推导为指针类型。

---

# 智能指针

