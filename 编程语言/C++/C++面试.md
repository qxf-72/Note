# Day 1：C++ 基础语义

  

> 主题：引用、指针、`const`、顶层 `const` / 底层 `const`、`auto`、`decltype`

  

---

  

## 一、引用

  

### 1. C++ 中引用和指针有什么区别？

  

**答：**

  

引用可以理解为对象的别名，指针是一个保存地址的对象。

  

| 对比点 | 引用 | 指针 |

|---|---|---|

| 是否必须初始化 | 必须初始化 | 可以不初始化，但危险 |

| 是否能改变绑定对象 | 不能重新绑定 | 可以改指向 |

| 是否可以为空 | 正常引用不能为空 | 指针可以是 `nullptr` |

| 使用方式 | 像普通对象一样用 | 需要解引用 `*p` |

| 本身是否是对象 | 引用不是独立对象 | 指针本身是对象 |

  

示例：

  

```cpp

int a = 10;

int b = 20;

  

int& r = a;

r = b;      // 不是让 r 绑定 b，而是把 b 的值赋给 a

  

int* p = &a;

p = &b;     // p 可以改为指向 b

```

  

**面试重点：**

  

> 引用一旦绑定，就不能再改绑；对引用赋值，本质是给它绑定的对象赋值。

  

---

  

### 2. 引用为什么必须初始化？

  

**答：**

  

因为引用不是一个独立对象，而是某个对象的别名。没有绑定对象的引用是没有意义的。

  

```cpp

int& r; // 错误，引用必须初始化

```

  

正确写法：

  

```cpp

int x = 10;

int& r = x;

```

  

---

  

### 3. 引用可以绑定字面值吗？

  

**答：**

  

普通左值引用不可以绑定普通字面值：

  

```cpp

int& r = 10; // 错误

```

  

但是 `const` 引用可以绑定临时对象：

  

```cpp

const int& r = 10; // 正确

```

  

原因是编译器会创建一个临时 `int` 对象，然后让 `r` 绑定它，并且延长这个临时对象的生命周期。

  

---

  

### 4. 函数参数为什么常用引用？

  

**答：**

  

因为引用传参可以避免拷贝，提高效率，同时还能修改实参。

  

```cpp

void modify(int& x) {

    x = 100;

}

  

int a = 10;

modify(a);

// a 变成 100

```

  

如果只想避免拷贝，但不想修改对象，常用 `const` 引用：

  

```cpp

void print(const string& s) {

    cout << s << endl;

}

```

  

**面试常见回答：**

  

> 大对象传参一般使用 `const T&`，既避免拷贝，又保证函数内部不会修改实参。

  

---

  

### 5. 可以返回局部变量的引用吗？

  

**答：**

  

不能。

  

```cpp

int& func() {

    int x = 10;

    return x; // 错误，危险

}

```

  

因为 `x` 是局部变量，函数结束后就销毁了，返回它的引用会形成悬空引用。

  

正确情况是返回生命周期足够长的对象引用，例如静态变量或外部传入的对象：

  

```cpp

int& get(int& x) {

    return x;

}

```

  

---

  

## 二、指针

  

### 6. 指针是什么？

  

**答：**

  

指针是一个对象，它保存另一个对象的地址。

  

```cpp

int x = 10;

int* p = &x;

```

  

其中：

  

```cpp

p   // 存的是 x 的地址

*p  // 访问 p 指向的对象，也就是 x

```

  

---

  

### 7. 空指针、野指针、悬空指针有什么区别？

  

**答：**

  

**空指针：**

  

```cpp

int* p = nullptr;

```

  

它不指向任何对象，是安全的“无指向状态”。

  

**野指针：**

  

```cpp

int* p;

*p = 10; // 危险

```

  

未初始化的指针，里面是随机地址。

  

**悬空指针：**

  

```cpp

int* p = nullptr;

  

{

    int x = 10;

    p = &x;

}

// x 已销毁，p 仍保存 x 原来的地址

```

  

指向的对象已经不存在了。

  

**面试回答重点：**

  

> 指针使用前要初始化，不再指向有效对象时应设为 `nullptr`。

  

---

  

### 8. `*p` 表达式的类型是什么？

  

**答：**

  

这个问题经常和 `decltype` 一起考。

  

```cpp

int x = 10;

int* p = &x;

```

  

表达式：

  

```cpp

*p

```

  

表示“通过指针访问它指向的对象”。它是一个**左值表达式**。

  

所以：

  

```cpp

decltype(*p)

```

  

结果是：

  

```cpp

int&

```

  

注意，不是 `int`。

  

---

  

## 三、`const`

  

### 9. `const` 的作用是什么？

  

**答：**

  

`const` 表示对象不能被修改。

  

```cpp

const int x = 10;

x = 20; // 错误

```

  

它常用于：

  

```cpp

const int n = 100;          // 常量

void func(const string& s); // 防止函数修改参数

const int* p;               // 不能通过 p 修改指向的值

```

  

---

  

### 10. `const` 对象必须初始化吗？

  

**答：**

  

通常必须初始化。

  

```cpp

const int x; // 错误

```

  

因为 `const` 对象创建之后不能再赋值，如果不初始化，就永远没有合法的值。

  

正确写法：

  

```cpp

const int x = 10;

```

  

---

  

## 四、`const int*`、`int* const`、`const int* const`

  

这是面试高频中的高频。

  

### 11. `const int* p` 是什么意思？

  

**答：**

  

```cpp

const int* p;

```

  

也可以写成：

  

```cpp

int const* p;

```

  

含义是：

  

> `p` 是一个指针，指向 `const int`。

  

也就是：

  

```cpp

int a = 10;

int b = 20;

  

const int* p = &a;

  

*p = 30; // 错误，不能通过 p 修改 a

p = &b;  // 正确，p 可以改指向

```

  

总结：

  

```cpp

const int* p;

```

  

是**指向常量的指针**。

  

指针本身可以变，指向的内容不能通过它改。

  

---

  

### 12. `int* const p` 是什么意思？

  

**答：**

  

```cpp

int* const p = &a;

```

  

含义是：

  

> `p` 是一个 const 指针，指向 int。

  

也就是：

  

```cpp

int a = 10;

int b = 20;

  

int* const p = &a;

  

*p = 30; // 正确，可以修改 a

p = &b;  // 错误，p 本身不能改指向

```

  

总结：

  

```cpp

int* const p;

```

  

是**常量指针**。

  

指针本身不能变，指向的内容可以改。

  

---

  

### 13. `const int* const p` 是什么意思？

  

**答：**

  

```cpp

const int* const p = &a;

```

  

含义是：

  

> `p` 是一个 const 指针，并且它指向 const int。

  

所以：

  

```cpp

const int* const p = &a;

  

*p = 20; // 错误，不能修改指向的内容

p = &b;  // 错误，不能修改 p 的指向

```

  

总结：

  

```cpp

const int* const p;

```

  

是**指向常量的常量指针**。

  

指针本身不能变，指向的内容也不能通过它修改。

  

---

  

### 14. 如何快速判断复杂指针声明？

  

**答：**

  

看 `const` 在 `*` 的哪边。

  

```cpp

const int* p;

```

  

`const` 在 `*` 左边，说明修饰的是指向的对象：

  

```cpp

*p 不能改

p 可以改

```

  

```cpp

int* const p;

```

  

`const` 在 `*` 右边，说明修饰的是指针本身：

  

```cpp

p 不能改

*p 可以改

```

  

```cpp

const int* const p;

```

  

两边都有：

  

```cpp

p 不能改

*p 也不能改

```

  

---

  

## 五、顶层 `const` 和底层 `const`

  

### 15. 什么是顶层 `const`？

  

**答：**

  

顶层 `const` 表示对象本身是常量。

  

```cpp

const int x = 10;

```

  

这里 `x` 本身不能被修改，所以 `const` 是顶层 `const`。

  

指针中：

  

```cpp

int* const p = &a;

```

  

`p` 本身不能改，所以这个 `const` 是顶层 `const`。

  

---

  

### 16. 什么是底层 `const`？

  

**答：**

  

底层 `const` 表示指针或引用所指向、所绑定的对象是常量。

  

```cpp

const int* p = &a;

```

  

这里 `p` 本身可以改，但是不能通过 `p` 修改它指向的对象。

  

所以这个 `const` 是底层 `const`。

  

引用中：

  

```cpp

const int& r = a;

```

  

这里 `r` 绑定的对象不能通过 `r` 修改，所以也是底层 `const`。

  

---

  

### 17. 举例说明顶层和底层 `const`

  

**答：**

  

```cpp

int a = 10;

  

const int x = 10;        // 顶层 const

int* const p1 = &a;      // 顶层 const

  

const int* p2 = &a;      // 底层 const

const int& r = a;        // 底层 const

  

const int* const p3 = &a;

```

  

其中：

  

```cpp

const int* const p3 = &a;

```

  

第一个 `const` 是底层 `const`，第二个 `const` 是顶层 `const`。

  

也就是：

  

```cpp

const int* const p3

^^^^^      ^^^^^

底层       顶层

```

  

---

  

### 18. 拷贝时顶层 `const` 和底层 `const` 有什么区别？

  

**答：**

  

顶层 `const` 通常会被忽略。

  

```cpp

const int x = 10;

int y = x; // 正确

```

  

因为这是把 `x` 的值拷贝给 `y`，不会修改 `x`。

  

但是底层 `const` 不能随便忽略。

  

```cpp

const int x = 10;

const int* p = &x;

  

int* q = p; // 错误

```

  

因为如果允许这样写，就可以通过 `q` 修改 `const int` 对象，破坏 `const` 限制。

  

正确写法：

  

```cpp

const int* q = p;

```

  

---

  

## 六、`auto`

  

### 19. `auto` 是什么？

  

**答：**

  

`auto` 让编译器根据初始化表达式自动推导变量类型。

  

```cpp

auto x = 10;       // int

auto y = 3.14;     // double

auto s = "hello";  // const char*

```

  

注意：

  

`auto` 必须有初始化表达式。

  

```cpp

auto x; // 错误

```

  

---

  

### 20. `auto` 会不会保留 `const`？

  

**答：**

  

要分情况。

  

#### 情况一：普通对象，顶层 `const` 会被忽略

  

```cpp

const int ci = 10;

  

auto x = ci;

```

  

这里 `x` 的类型是：

  

```cpp

int

```

  

不是：

  

```cpp

const int

```

  

因为 `ci` 的顶层 `const` 被忽略了。

  

所以：

  

```cpp

x = 20; // 正确

```

  

#### 情况二：如果想保留顶层 `const`，需要自己写 `const auto`

  

```cpp

const int ci = 10;

  

const auto x = ci;

```

  

此时 `x` 是：

  

```cpp

const int

```

  

#### 情况三：底层 `const` 会被保留

  

```cpp

const int ci = 10;

const int* p = &ci;

  

auto q = p;

```

  

这里 `q` 的类型是：

  

```cpp

const int*

```

  

底层 `const` 被保留。

  

所以：

  

```cpp

*q = 20;    // 错误

q = nullptr; // 正确

```

  

---

  

### 21. `auto` 和引用有什么关系？

  

**答：**

  

默认情况下，`auto` 会忽略引用。

  

```cpp

int x = 10;

int& r = x;

  

auto y = r;

```

  

这里 `y` 的类型是：

  

```cpp

int

```

  

不是：

  

```cpp

int&

```

  

所以：

  

```cpp

y = 20;

```

  

只会修改 `y`，不会修改 `x`。

  

如果想让 `auto` 推导出引用，要写：

  

```cpp

auto& y = r;

```

  

此时：

  

```cpp

y = 20;

```

  

会修改 `x`。

  

---

  

### 22. `auto` 推导数组时会发生什么？

  

**答：**

  

```cpp

int arr[3] = {1, 2, 3};

  

auto p = arr;

```

  

这里 `p` 的类型是：

  

```cpp

int*

```

  

数组名会退化为指针。

  

如果想保留数组类型，要用引用：

  

```cpp

auto& r = arr;

```

  

此时 `r` 的类型是：

  

```cpp

int (&)[3]

```

  

---

  

## 七、`decltype`

  

### 23. `decltype` 是什么？

  

**答：**

  

`decltype` 用来推导表达式的类型，但它不会计算表达式。

  

```cpp

int x = 10;

  

decltype(x) y = 20;

```

  

这里：

  

```cpp

decltype(x)

```

  

结果是：

  

```cpp

int

```

  

所以 `y` 是 `int`。

  

---

  

### 24. `decltype(x)` 和 `decltype((x))` 有什么区别？

  

**答：**

  

这是面试非常高频的问题。

  

```cpp

int x = 10;

```

  

那么：

  

```cpp

decltype(x)

```

  

结果是：

  

```cpp

int

```

  

但是：

  

```cpp

decltype((x))

```

  

结果是：

  

```cpp

int&

```

  

原因：

  

`decltype(x)` 中，`x` 是一个**未加括号的变量名**，所以结果就是它声明时的类型。

  

`decltype((x))` 中，`(x)` 是一个表达式，而且它是左值表达式，所以结果是左值引用类型 `int&`。

  

---

  

### 25. `decltype` 的核心规则是什么？

  

**答：**

  

记住这几条：

  

```cpp

decltype(变量名)

```

  

如果是未加括号的变量名，结果就是变量声明时的类型。

  

```cpp

decltype((表达式))

```

  

如果表达式是左值，结果是 `T&`。

  

如果表达式是将亡值，结果是 `T&&`。

  

如果表达式是纯右值，结果是 `T`。

  

例如：

  

```cpp

int x = 10;

int* p = &x;

  

decltype(x) a = x;    // int

decltype((x)) b = x;  // int&

decltype(*p) c = x;   // int&

decltype(10) d = 10;  // int

```

  

重点：

  

```cpp

*p

```

  

是左值，所以：

  

```cpp

decltype(*p)

```

  

是：

  

```cpp

int&

```

  

---

  

### 26. 为什么 `decltype(*p)` 是引用类型？

  

**答：**

  

因为：

  

```cpp

*p

```

  

表示访问 `p` 指向的那个对象。这个表达式可以出现在赋值号左边：

  

```cpp

*p = 20;

```

  

能出现在赋值号左边，说明它是左值。

  

所以：

  

```cpp

decltype(*p)

```

  

结果是：

  

```cpp

int&

```

  

---

  

## 八、`auto` 和 `decltype` 的区别

  

### 27. `auto` 和 `decltype` 有什么区别？

  

**答：**

  

`auto` 根据初始化表达式推导变量类型，通常会发生类型调整，比如忽略顶层 `const`、忽略引用、数组退化为指针。

  

`decltype` 更严格，它根据表达式形式推导类型，尽量保留表达式的真实类型和值类别。

  

例如：

  

```cpp

const int ci = 10;

const int& r = ci;

  

auto a = r;

```

  

`a` 是：

  

```cpp

int

```

  

但是：

  

```cpp

decltype(r) b = ci;

```

  

`b` 是：

  

```cpp

const int&

```

  

所以可以总结为：

  

> `auto` 更像“用这个表达式初始化一个新变量，变量应该是什么类型”；  

> `decltype` 更像“这个表达式本身是什么类型”。

  

---

  

### 28. 什么是 `decltype(auto)`？

  

**答：**

  

`decltype(auto)` 使用 `decltype` 的规则来推导 `auto` 类型，常用于函数返回值。

  

例如：

  

```cpp

int x = 10;

  

int& get() {

    return x;

}

  

decltype(auto) y = get();

```

  

这里 `y` 的类型是：

  

```cpp

int&

```

  

如果用普通 `auto`：

  

```cpp

auto y = get();

```

  

则 `y` 是：

  

```cpp

int

```

  

引用会被忽略。

  

**面试重点：**

  

> `auto` 会丢引用，`decltype(auto)` 可以保留引用和值类别。

  

---

  

## 九、典型代码题

  

### 29. 判断下面代码中变量的类型

  

```cpp

int i = 0;

const int ci = i;

int& r = i;

const int& cr = ci;

```

  

```cpp

auto a = i;   // int

auto b = ci;  // int，顶层 const 被忽略

auto c = r;   // int，引用被忽略

auto d = cr;  // int，引用和顶层 const 被忽略

```

  

如果写成引用：

  

```cpp

auto& e = ci; // const int&

auto& f = r;  // int&

```

  

如果写成 `const auto&`：

  

```cpp

const auto& g = i; // const int&

```

  

---

  

### 30. 判断下面代码中指针的类型

  

```cpp

int i = 0;

const int ci = 10;

  

auto p1 = &i;

auto p2 = &ci;

```

  

答案：

  

```cpp

p1 // int*

p2 // const int*

```

  

因为 `ci` 是 `const int`，所以取地址后得到的是：

  

```cpp

const int*

```

  

底层 `const` 会被 `auto` 保留。

  

---

  

### 31. 判断下面代码是否正确

  

```cpp

const int ci = 10;

int* p = &ci;

```

  

**答：**

  

错误。

  

因为 `&ci` 的类型是：

  

```cpp

const int*

```

  

不能赋给：

  

```cpp

int*

```

  

否则就可以通过 `p` 修改 `ci`，破坏 `const`。

  

正确写法：

  

```cpp

const int* p = &ci;

```

  

---

  

### 32. 判断下面代码是否正确

  

```cpp

int i = 0;

const int* p = &i;

*p = 10;

```

  

**答：**

  

错误。

  

虽然 `i` 本身不是 `const int`，但是 `p` 的类型是：

  

```cpp

const int*

```

  

这表示不能通过 `p` 修改它指向的对象。

  

但是可以直接修改 `i`：

  

```cpp

i = 10; // 正确

```

  

---

  

### 33. 判断下面代码中 `decltype` 的结果

  

```cpp

int x = 10;

int& r = x;

const int ci = 20;

const int& cr = ci;

```

  

```cpp

decltype(x) a = x;    // int

decltype(r) b = x;    // int&

decltype(ci) c = ci;  // const int

decltype(cr) d = ci;  // const int&

```

  

注意：

  

```cpp

decltype(r)

```

  

结果是：

  

```cpp

int&

```

  

因为 `r` 声明时就是引用类型。

  

---

  

### 34. 判断 `decltype((x))` 的结果

  

```cpp

int x = 10;

const int ci = 20;

```

  

```cpp

decltype((x)) a = x;   // int&

decltype((ci)) b = ci; // const int&

```

  

因为：

  

```cpp

(x)

```

  

和：

  

```cpp

(ci)

```

  

都是左值表达式。

  

所以 `decltype((x))` 是引用类型。

  

---

  

## 十、面试总结版

  

### 引用

  

> 引用是对象的别名，必须初始化，不能重新绑定，通常不能为空。对引用赋值就是对它绑定的对象赋值。

  

### 指针

  

> 指针是保存地址的对象，可以改指向，也可以为空。使用指针时要注意空指针、野指针和悬空指针。

  

### `const`

  

> `const` 表示对象不能被修改。指针相关声明中，`const` 在 `*` 左边，表示指向的内容不能改；在 `*` 右边，表示指针本身不能改。

  

### 顶层 `const` / 底层 `const`

  

> 顶层 `const` 修饰对象本身，底层 `const` 修饰指针或引用所指向、所绑定的对象。拷贝时顶层 `const` 通常会被忽略，但底层 `const` 不能随便忽略。

  

### `auto`

  

> `auto` 根据初始化表达式推导类型，通常会忽略顶层 `const` 和引用，但会保留底层 `const`。如果要保留引用，需要写 `auto&`。

  

### `decltype`

  

> `decltype` 根据表达式形式推导类型。`decltype(x)` 如果 `x` 是未加括号的变量名，结果是声明类型；`decltype((x))` 中 `(x)` 是左值表达式，所以结果是 `T&`。

  

重点中的重点是这几个：

  

```cpp

const int* p;        // 指向常量的指针，*p 不能改，p 可以改

int* const p;        // 常量指针，p 不能改，*p 可以改

const int* const p;  // 指向常量的常量指针，p 和 *p 都不能改

```

  

以及：

  

```cpp

int x = 10;

  

decltype(x)    // int

decltype((x))  // int&

```

  

这两组非常容易被面试官拿来考。