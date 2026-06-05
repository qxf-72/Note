# auto

## lambda 表达式初探

**lambda 表达式底层实现原理**：编译器构建一个类——可调用的类，并且重载了（）运算符。其  实就是仿函数。

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

## 优先考虑 `auto` 而非显式声明











---