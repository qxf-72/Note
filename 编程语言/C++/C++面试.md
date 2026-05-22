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


---

# Day 2：STL 容器与迭代器 

  

> 覆盖内容：`string`、`vector`、迭代器、数组与指针、`map`、`unordered_map`、`set`、`vector` 扩容、迭代器失效、`string::c_str()` 生命周期、`resize` 和 `reserve` 区别。

  

---

  

## 一、STL 基础

  

### 1. STL 是什么？

  

**答：**

  

STL 是 C++ 标准模板库，主要由以下几部分组成：

  

- 容器：例如 `vector`、`string`、`map`、`set`、`unordered_map`

- 迭代器：用于遍历容器

- 算法：例如 `sort`、`find`、`lower_bound`

- 函数对象：例如比较器、哈希函数

- 适配器：例如 `stack`、`queue`、`priority_queue`

  

面试中可以这样回答：

  

> STL 是 C++ 提供的一套泛型数据结构和算法库。容器负责存储数据，迭代器负责连接容器和算法，算法通过迭代器操作容器中的元素。

  

---

  

### 2. STL 容器大致分为哪几类？

  

**答：**

  

常见分类如下：

  

| 类型 | 常见容器 | 特点 |

|---|---|---|

| 序列式容器 | `vector`、`deque`、`list`、`array`、`string` | 按元素顺序存储 |

| 关联式容器 | `map`、`set`、`multimap`、`multiset` | 通常基于红黑树，有序 |

| 无序关联式容器 | `unordered_map`、`unordered_set` | 通常基于哈希表，无序 |

| 容器适配器 | `stack`、`queue`、`priority_queue` | 基于其他容器封装而来 |

  

---

  

## 二、`string`

  

### 3. `string` 的底层是什么？

  

**答：**

  

`std::string` 可以理解为对动态字符数组的封装，底层通常是一段连续的字符存储空间。

  

从 C++11 开始，`std::string` 的字符存储要求是连续的，因此可以通过：

  

```cpp

string s = "hello";

const char* p = s.c_str();

```

  

获得一个 C 风格字符串指针。

  

面试可以这样说：

  

> `string` 底层类似动态字符数组，负责自动管理内存，支持动态扩容，并且提供了比 C 风格字符串更安全、更方便的接口。

  

---

  

### 4. `string` 和 C 风格字符串有什么区别？

  

**答：**

  

| 对比点 | `std::string` | C 风格字符串 |

|---|---|---|

| 本质 | 类对象 | 字符数组或字符指针 |

| 内存管理 | 自动管理 | 需要程序员自己管理 |

| 是否记录长度 | 记录长度 | 通常依赖 `\0` 判断结尾 |

| 安全性 | 更高 | 容易越界、悬空 |

| 使用便利性 | 支持拼接、比较、查找等成员函数 | 需要使用 `strlen`、`strcpy` 等函数 |

  

例如：

  

```cpp

string s = "hello";

s += " world";

cout << s.size() << endl;

```

  

C 风格字符串：

  

```cpp

char s[20] = "hello";

strcat(s, " world");

cout << strlen(s) << endl;

```

  

---

  

### 5. `string::c_str()` 的作用是什么？

  

**答：**

  

`c_str()` 返回一个指向字符串内部字符数组的 `const char*`，通常用于和 C 风格接口交互。

  

```cpp

string s = "hello";

const char* p = s.c_str();

```

  

它返回的字符串以 `\0` 结尾。

  

常见用途：

  

```cpp

string filename = "test.txt";

FILE* fp = fopen(filename.c_str(), "r");

```

  

---

  

### 6. `string::c_str()` 返回的指针生命周期是怎样的？

  

**答：**

  

`c_str()` 返回的指针指向 `string` 对象内部的存储空间，所以它的有效性依赖于原来的 `string` 对象。

  

只要满足以下条件，指针通常有效：

  

1. 原 `string` 对象还活着；

2. 原 `string` 没有发生可能导致重新分配内存的修改操作。

  

例如：

  

```cpp

string s = "hello";

const char* p = s.c_str();

  

cout << p << endl; // 正确

  

s += " world";     // 可能触发扩容

cout << p << endl; // 危险，p 可能已经失效

```

  

错误示例：

  

```cpp

const char* getName() {

    string s = "hello";

    return s.c_str(); // 错误，s 离开函数后被销毁

}

```

  

因为函数返回后，局部变量 `s` 已经销毁，返回的指针变成悬空指针。

  

面试重点：

  

> `c_str()` 返回的指针不能长期保存。只要原 `string` 被销毁，或者发生可能重新分配内存的修改操作，这个指针就可能失效。

  

---

  

### 7. `string` 可以直接和 `char*` 相互转换吗？

  

**答：**

  

C 风格字符串可以构造 `string`：

  

```cpp

const char* p = "hello";

string s = p;

```

  

`string` 转 C 风格字符串需要使用 `c_str()`：

  

```cpp

string s = "hello";

const char* p = s.c_str();

```

  

注意，`c_str()` 返回的是 `const char*`，不能直接修改其内容：

  

```cpp

*p = 'H'; // 错误

```

  

---

  

## 三、`vector`

  

### 8. `vector` 的底层原理是什么？

  

**答：**

  

`vector` 底层是一段连续的动态数组。

  

它通常维护三个核心信息：

  

- 起始位置；

- 当前元素结束位置；

- 当前容量结束位置。

  

可以抽象理解为：

  

```cpp

begin      end        capacity end

  |         |              |

  v         v              v

[ 已使用元素 ][ 未使用但已分配空间 ]

```

  

所以：

  

```cpp

v.size();     // 当前已有元素个数

v.capacity(); // 当前已经分配的空间能容纳多少元素

```

  

面试回答：

  

> `vector` 底层是连续内存，因此支持 O(1) 随机访问。但在中间插入或删除元素需要移动后续元素，时间复杂度是 O(n)。当容量不足时，`vector` 会重新分配更大的内存，并把原来的元素移动或拷贝过去。

  

---

  

### 9. `vector` 的优点和缺点是什么？

  

**答：**

  

优点：

  

- 支持随机访问，`v[i]` 是 O(1)；

- 内存连续，缓存友好；

- 尾部插入平均效率高；

- 使用方便，能自动扩容。

  

缺点：

  

- 中间插入、删除需要移动元素，效率较低；

- 扩容时可能重新分配内存，导致迭代器、指针、引用失效；

- 如果频繁扩容，会有额外开销。

  

---

  

### 10. `vector` 的 `size()` 和 `capacity()` 有什么区别？

  

**答：**

  

`size()` 表示当前实际元素个数。

  

`capacity()` 表示当前已经分配的内存最多能容纳多少个元素。

  

例如：

  

```cpp

vector<int> v;

v.reserve(10);

  

cout << v.size() << endl;     // 0

cout << v.capacity() << endl; // 至少 10

```

  

此时 `vector` 已经预留空间，但还没有真正创建元素。

  

---

  

### 11. `vector` 是如何扩容的？

  

**答：**

  

当 `vector` 的元素数量超过当前容量时，会触发扩容。

  

一般过程是：

  

1. 申请一块更大的连续内存；

2. 将原来的元素移动或拷贝到新内存；

3. 销毁旧内存中的元素；

4. 释放旧内存；

5. 新元素插入到新空间。

  

示例：

  

```cpp

vector<int> v;

  

for (int i = 0; i < 10; ++i) {

    v.push_back(i);

    cout << "size = " << v.size()

         << ", capacity = " << v.capacity() << endl;

}

```

  

常见实现中，`vector` 扩容一般按 1.5 倍或 2 倍增长，但标准并没有规定必须是多少倍。

  

面试重点：

  

> `vector` 扩容会重新分配内存，因此原来的迭代器、指针、引用可能全部失效。

  

---

  

### 12. 为什么 `vector` 扩容通常不是每次只增加 1 个空间？

  

**答：**

  

如果每插入一个元素都重新分配一次内存，插入 n 个元素可能需要大量拷贝或移动，效率很低。

  

使用倍增扩容策略可以让尾插 `push_back` 的均摊时间复杂度保持为 O(1)。

  

简单说：

  

> 扩容多申请一些空间，是用空间换时间，减少重新分配内存的次数。

  

---

  

### 13. `push_back` 和 `emplace_back` 有什么区别？

  

**答：**

  

`push_back` 是把一个已经存在的对象加入 `vector`。

  

`emplace_back` 是在 `vector` 尾部直接构造对象。

  

例如：

  

```cpp

vector<string> v;

  

string s = "hello";

v.push_back(s);              // 拷贝或移动已有对象

v.emplace_back("world");     // 直接在容器中构造 string

```

  

面试回答：

  

> 对于复杂对象，`emplace_back` 可能减少一次临时对象构造或移动的开销。但在现代编译器优化和移动语义下，二者性能差距不一定明显。普通类型如 `int`，差别可以忽略。

  

---

  

### 14. `vector` 中间插入和删除为什么慢？

  

**答：**

  

因为 `vector` 底层是连续数组。

  

在中间插入元素时，需要把插入位置后面的元素整体向后移动。

  

```cpp

vector<int> v = {1, 2, 4, 5};

v.insert(v.begin() + 2, 3);

```

  

插入后：

  

```cpp

1 2 3 4 5

```

  

原来的 `4`、`5` 都需要后移。

  

删除同理：

  

```cpp

v.erase(v.begin() + 1);

```

  

删除后，后面的元素需要前移。

  

所以中间插入和删除的时间复杂度通常是 O(n)。

  

---

  

## 四、`resize` 和 `reserve`

  

### 15. `resize` 和 `reserve` 有什么区别？

  

**答：**

  

这是 STL 面试高频问题。

  

| 函数 | 改变 `size` | 改变 `capacity` | 是否创建元素 | 用途 |

|---|---|---|---|---|

| `resize(n)` | 会 | 可能会 | 会 | 改变元素个数 |

| `reserve(n)` | 不会 | 可能会 | 不会 | 预留容量 |

  

示例：

  

```cpp

vector<int> v;

  

v.reserve(10);

cout << v.size() << endl;     // 0

cout << v.capacity() << endl; // 至少 10

  

v.resize(10);

cout << v.size() << endl;     // 10

cout << v.capacity() << endl; // 至少 10

```

  

`reserve(10)` 只是预留空间，不创建元素。

  

`resize(10)` 会让容器真的拥有 10 个元素。

  

---

  

### 16. `reserve` 后可以直接用下标访问吗？

  

**答：**

  

不可以。

  

```cpp

vector<int> v;

v.reserve(10);

  

v[0] = 1; // 错误，未定义行为

```

  

因为此时：

  

```cpp

v.size() == 0

```

  

虽然容量够了，但元素还没有被创建。

  

正确写法：

  

```cpp

vector<int> v;

v.reserve(10);

v.push_back(1);

```

  

或者：

  

```cpp

vector<int> v;

v.resize(10);

v[0] = 1;

```

  

面试重点：

  

> `reserve` 只管容量，不管元素个数；只有 `size()` 范围内的下标访问才是合法的。

  

---

  

### 17. `resize` 变大和变小时分别发生什么？

  

**答：**

  

如果 `resize(n)` 中的 `n` 大于当前 `size()`：

  

- 新增元素会被默认初始化；

- 如果容量不够，可能触发扩容。

  

```cpp

vector<int> v = {1, 2, 3};

v.resize(5);

// v 变成 {1, 2, 3, 0, 0}

```

  

如果 `n` 小于当前 `size()`：

  

- 多余元素会被销毁；

- `size()` 变小；

- `capacity()` 通常不会减少。

  

```cpp

vector<int> v = {1, 2, 3, 4, 5};

v.resize(3);

// v 变成 {1, 2, 3}

```

  

---

  

## 五、迭代器

  

### 18. 迭代器是什么？

  

**答：**

  

迭代器是 STL 中用于访问容器元素的对象。它的作用类似指针，但不完全等同于指针。

  

例如：

  

```cpp

vector<int> v = {1, 2, 3};

  

for (vector<int>::iterator it = v.begin(); it != v.end(); ++it) {

    cout << *it << endl;

}

```

  

也可以用 `auto` 简化：

  

```cpp

for (auto it = v.begin(); it != v.end(); ++it) {

    cout << *it << endl;

}

```

  

面试回答：

  

> 迭代器是连接容器和算法的桥梁。算法不需要关心容器底层结构，只需要通过迭代器访问元素。

  

---

  

### 19. `begin()` 和 `end()` 分别表示什么？

  

**答：**

  

`begin()` 指向第一个元素。

  

`end()` 指向最后一个元素的后一个位置，也叫尾后迭代器。

  

```cpp

vector<int> v = {1, 2, 3};

  

v.begin(); // 指向 1

v.end();   // 指向 3 后面的一个位置

```

  

注意：

  

```cpp

*v.end(); // 错误，不能解引用 end()

```

  

`end()` 只是边界标记，不指向有效元素。

  

---

  

### 20. 常见迭代器类型有哪些？

  

**答：**

  

常见迭代器能力从弱到强大致如下：

  

| 迭代器类型 | 典型容器 | 能力 |

|---|---|---|

| 输入迭代器 | 输入流 | 只读、单向 |

| 输出迭代器 | 输出流 | 只写、单向 |

| 前向迭代器 | `forward_list` | 多次遍历、单向 |

| 双向迭代器 | `list`、`map`、`set` | 可前进、可后退 |

| 随机访问迭代器 | `vector`、`deque`、`array` | 支持 `+`、`-`、`[]` |

  

常见面试点：

  

```cpp

vector<int>::iterator

```

  

支持：

  

```cpp

it + 3

it - 1

it1 - it2

```

  

但是：

  

```cpp

map<int, int>::iterator

```

  

不支持 `it + 1`，只能使用：

  

```cpp

++it;

--it;

```

  

---

  

### 21. `iterator` 和 `const_iterator` 有什么区别？

  

**答：**

  

`iterator` 可以修改元素。

  

`const_iterator` 不能修改元素。

  

```cpp

vector<int> v = {1, 2, 3};

  

vector<int>::iterator it = v.begin();

*it = 10; // 正确

  

vector<int>::const_iterator cit = v.cbegin();

*cit = 20; // 错误

```

  

面试回答：

  

> 如果只遍历不修改元素，建议使用 `const_iterator`，可以表达只读语义，也能提高代码安全性。

  

---

  

### 22. 范围 `for` 循环中什么时候用引用？

  

**答：**

  

如果只读取小对象，可以直接值拷贝：

  

```cpp

for (int x : nums) {

    cout << x << endl;

}

```

  

如果要修改元素，要用引用：

  

```cpp

for (int& x : nums) {

    x *= 2;

}

```

  

如果对象比较大，并且只读，建议用 `const` 引用：

  

```cpp

for (const string& s : words) {

    cout << s << endl;

}

```

  

面试总结：

  

> 小对象只读可以值传递；要修改用引用；大对象只读用 `const` 引用。

  

---

  

## 六、迭代器失效

  

### 23. 什么是迭代器失效？

  

**答：**

  

迭代器失效指的是：原来指向容器元素的迭代器，由于容器发生修改，已经不再指向有效元素，继续使用会产生未定义行为。

  

例如：

  

```cpp

vector<int> v = {1, 2, 3};

auto it = v.begin();

  

v.push_back(4); // 可能触发扩容

  

cout << *it << endl; // 危险，it 可能已经失效

```

  

---

  

### 24. `vector` 哪些操作会导致迭代器失效？

  

**答：**

  

`vector` 底层是连续数组，所以迭代器失效比较常见。

  

#### 1. 扩容导致全部失效

  

```cpp

vector<int> v = {1, 2, 3};

auto it = v.begin();

  

v.push_back(4); // 如果触发扩容，所有迭代器、指针、引用都失效

```

  

#### 2. 插入导致部分失效

  

如果没有扩容：

  

```cpp

v.insert(pos, x);

```

  

插入位置及其后面的迭代器失效。

  

如果发生扩容，则全部失效。

  

#### 3. 删除导致部分失效

  

```cpp

v.erase(pos);

```

  

删除位置及其后面的迭代器失效。

  

---

  

### 25. 如何正确边遍历边删除 `vector` 元素？

  

**答：**

  

错误写法：

  

```cpp

for (auto it = v.begin(); it != v.end(); ++it) {

    if (*it == 0) {

        v.erase(it); // it 失效，后续 ++it 危险

    }

}

```

  

正确写法：

  

```cpp

for (auto it = v.begin(); it != v.end(); ) {

    if (*it == 0) {

        it = v.erase(it); // erase 返回下一个有效位置

    } else {

        ++it;

    }

}

```

  

如果只是按条件删除，推荐使用 erase-remove 惯用法：

  

```cpp

v.erase(remove(v.begin(), v.end(), 0), v.end());

```

  

或者删除满足条件的元素：

  

```cpp

v.erase(remove_if(v.begin(), v.end(), [](int x) {

    return x % 2 == 0;

}), v.end());

```

  

---

  

### 26. `map` 和 `set` 删除元素会导致哪些迭代器失效？

  

**答：**

  

`map` 和 `set` 通常基于红黑树实现，删除一个元素时，只有被删除元素对应的迭代器失效，其他迭代器通常不失效。

  

正确边遍历边删除：

  

```cpp

map<int, int> mp = {{1, 10}, {2, 20}, {3, 30}};

  

for (auto it = mp.begin(); it != mp.end(); ) {

    if (it->first == 2) {

        it = mp.erase(it);

    } else {

        ++it;

    }

}

```

  

---

  

### 27. `unordered_map` 的迭代器什么时候会失效？

  

**答：**

  

`unordered_map` 通常基于哈希表。

  

常见失效场景：

  

1. 删除元素：被删除元素的迭代器失效；

2. 插入元素：如果触发 rehash，所有迭代器可能失效；

3. 手动调用 `rehash()` 或 `reserve()`：可能导致所有迭代器失效。

  

面试重点：

  

> `unordered_map` 插入时如果触发 rehash，会导致迭代器失效，因为元素会被重新分布到新的桶数组中。

  

---

  

## 七、数组与指针

  

### 28. 数组和指针有什么关系？

  

**答：**

  

数组名在很多表达式中会退化为指向首元素的指针。

  

```cpp

int arr[3] = {1, 2, 3};

int* p = arr;

```

  

这里 `arr` 退化成 `&arr[0]`。

  

所以：

  

```cpp

p[0] == arr[0]

p[1] == arr[1]

```

  

但是数组和指针不是同一个东西。

  

```cpp

sizeof(arr); // 整个数组大小，假设 int 为 4 字节，则是 12

sizeof(p);   // 指针大小，64 位系统通常是 8

```

  

---

  

### 29. `arr` 和 `&arr` 有什么区别？

  

**答：**

  

```cpp

int arr[3] = {1, 2, 3};

```

  

`arr` 在表达式中通常退化为：

  

```cpp

int*

```

  

指向首元素。

  

`&arr` 的类型是：

  

```cpp

int (*)[3]

```

  

表示指向整个数组的指针。

  

示例：

  

```cpp

cout << arr << endl;     // 首元素地址

cout << &arr << endl;    // 整个数组地址，数值上通常相同

  

cout << arr + 1 << endl;  // 跳过一个 int

cout << &arr + 1 << endl; // 跳过整个数组

```

  

面试重点：

  

> `arr` 和 `&arr` 打印出来的地址值可能一样，但类型不同，指针运算的步长也不同。

  

---

  

### 30. 数组作为函数参数时会发生什么？

  

**答：**

  

数组作为函数参数时会退化为指针。

  

```cpp

void func(int arr[]) {

    cout << sizeof(arr) << endl;

}

```

  

这里的 `arr` 实际类型是：

  

```cpp

int*

```

  

所以 `sizeof(arr)` 得到的是指针大小，不是数组大小。

  

等价写法：

  

```cpp

void func(int* arr) {

}

```

  

如果想保留数组长度，可以使用数组引用：

  

```cpp

void func(int (&arr)[3]) {

    cout << sizeof(arr) << endl; // 整个数组大小

}

```

  

---

  

### 31. `vector` 和数组有什么区别？

  

**答：**

  

| 对比点 | 数组 | `vector` |

|---|---|---|

| 大小 | 固定 | 可动态变化 |

| 内存管理 | 手动或栈上自动管理 | 自动管理动态内存 |

| 是否记录长度 | 原生数组本身不记录 | 通过 `size()` 获取 |

| 越界检查 | 普通下标无检查 | `at()` 有检查，`[]` 无检查 |

| 使用便利性 | 较低 | 较高 |

  

面试回答：

  

> 原生数组轻量，但大小固定且容易退化为指针；`vector` 是动态数组，封装了内存管理和长度信息，更适合一般工程开发。

  

---

  

## 八、`map`

  

### 32. `map` 的底层原理是什么？

  

**答：**

  

`map` 通常基于红黑树实现。

  

特点：

  

- 键值对存储；

- 按 key 自动排序；

- key 不允许重复；

- 插入、删除、查找时间复杂度通常是 O(log n)。

  

示例：

  

```cpp

map<string, int> mp;

mp["Tom"] = 18;

mp["Jerry"] = 20;

```

  

遍历时会按照 key 的顺序输出：

  

```cpp

for (const auto& [name, age] : mp) {

    cout << name << " " << age << endl;

}

```

  

---

  

### 33. `map` 的 key 为什么不能修改？

  

**答：**

  

`map` 中的元素类型本质上类似：

  

```cpp

pair<const Key, Value>

```

  

key 是 `const` 的，不能修改。

  

原因是 `map` 依赖 key 维持红黑树的有序结构。如果随意修改 key，会破坏树的排序关系。

  

错误示例：

  

```cpp

map<int, string> mp;

mp[1] = "one";

  

auto it = mp.begin();

it->first = 2; // 错误，key 不能修改

```

  

可以修改 value：

  

```cpp

it->second = "new value"; // 正确

```

  

---

  

### 34. `map[key]` 和 `map.find(key)` 有什么区别？

  

**答：**

  

`mp[key]`：

  

- 如果 key 存在，返回对应 value；

- 如果 key 不存在，会插入一个默认值。

  

`mp.find(key)`：

  

- 只查找；

- 找不到返回 `mp.end()`；

- 不会插入新元素。

  

示例：

  

```cpp

map<string, int> mp;

  

cout << mp["abc"] << endl; // 插入 {"abc", 0}

  

if (mp.find("def") == mp.end()) {

    cout << "not found" << endl;

}

```

  

面试重点：

  

> 如果只是判断 key 是否存在，不建议用 `operator[]`，因为它可能意外插入新元素。应该使用 `find` 或 C++20 的 `contains`。

  

---

  

### 35. `map` 的 `insert` 和 `operator[]` 有什么区别？

  

**答：**

  

`operator[]` 会在 key 不存在时插入默认 value，然后返回 value 的引用。

  

```cpp

mp["a"] = 1;

```

  

`insert` 只有在 key 不存在时才插入，key 已存在则不会覆盖原值。

  

```cpp

mp.insert({"a", 100}); // 如果 "a" 已存在，不会覆盖

```

  

如果想插入或覆盖，可以使用：

  

```cpp

mp["a"] = 100;

```

  

或者 C++17：

  

```cpp

mp.insert_or_assign("a", 100);

```

  

---

  

## 九、`unordered_map`

  

### 36. `unordered_map` 的底层原理是什么？

  

**答：**

  

`unordered_map` 通常基于哈希表实现。

  

它通过哈希函数把 key 映射到桶中。

  

特点：

  

- key 无序；

- key 唯一；

- 平均查找、插入、删除是 O(1)；

- 最坏情况下可能退化到 O(n)。

  

示例：

  

```cpp

unordered_map<string, int> mp;

mp["Tom"] = 18;

mp["Jerry"] = 20;

```

  

遍历顺序不保证和插入顺序一致，也不保证有序。

  

---

  

### 37. 什么是哈希冲突？

  

**答：**

  

不同 key 经过哈希函数计算后，可能落到同一个桶中，这就叫哈希冲突。

  

例如：

  

```cpp

hash(key1) % bucket_count == hash(key2) % bucket_count

```

  

解决冲突的常见方式包括：

  

- 链地址法；

- 开放寻址法。

  

C++ 标准没有强制规定具体实现，但常见标准库实现通常使用类似链地址法的结构。

  

---

  

### 38. `unordered_map` 为什么平均 O(1)，最坏 O(n)？

  

**答：**

  

理想情况下，哈希函数分布均匀，每个桶中的元素很少，查找时可以快速定位到目标桶，所以平均 O(1)。

  

但如果大量 key 哈希到同一个桶中，桶内元素很多，查找时可能需要线性遍历，所以最坏 O(n)。

  

面试回答：

  

> `unordered_map` 的性能依赖哈希函数质量和负载因子。如果哈希冲突严重，性能会下降。

  

---

  

### 39. 什么是负载因子？

  

**答：**

  

负载因子表示：

  

```cpp

load_factor = 元素数量 / 桶数量

```

  

可以通过：

  

```cpp

mp.load_factor();

mp.max_load_factor();

```

  

查看或设置。

  

当负载因子过高时，哈希冲突可能增多，容器可能触发 rehash，重新分配桶并重新放置元素。

  

---

  

### 40. `unordered_map` 的 `reserve` 和 `rehash` 有什么区别？

  

**答：**

  

`reserve(n)` 表示：希望容器至少能容纳 n 个元素而不 rehash。

  

`rehash(n)` 表示：直接调整桶数量至少为 n。

  

```cpp

unordered_map<int, int> mp;

  

mp.reserve(1000); // 为大约 1000 个元素做准备

mp.rehash(1000);  // 桶数量至少调整到 1000

```

  

实际工程中，如果已知大概元素数量，通常用：

  

```cpp

mp.reserve(n);

```

  

这样可以减少 rehash 次数。

  

---

  

## 十、`map` 和 `unordered_map` 区别

  

### 41. `map` 和 `unordered_map` 有什么区别？

  

**答：**

  

| 对比点 | `map` | `unordered_map` |

|---|---|---|

| 底层结构 | 红黑树 | 哈希表 |

| 元素顺序 | 按 key 有序 | 无序 |

| 查找复杂度 | O(log n) | 平均 O(1)，最坏 O(n) |

| key 要求 | 需要能比较大小 | 需要能哈希和判断相等 |

| 迭代器稳定性 | 插入通常不影响已有迭代器 | rehash 会导致迭代器失效 |

| 适用场景 | 需要有序、范围查询 | 只需要快速查找 |

  

面试回答：

  

> 如果需要按 key 有序遍历，或者需要 `lower_bound`、`upper_bound` 这类范围操作，选 `map`。如果只关心快速查找，一般选 `unordered_map`。

  

---

  

### 42. 什么时候用 `map`，什么时候用 `unordered_map`？

  

**答：**

  

用 `map` 的情况：

  

- 需要按照 key 有序遍历；

- 需要范围查询；

- 需要稳定的 O(log n) 复杂度；

- key 没有合适的哈希函数，但可以比较大小。

  

用 `unordered_map` 的情况：

  

- 只需要快速查找、插入、删除；

- 不关心遍历顺序；

- key 有合适的哈希函数；

- 希望平均 O(1)。

  

例如：

  

```cpp

// 需要有序输出

map<int, string> ordered;

  

// 只需要快速查找

unordered_map<string, int> counter;

```

  

---

  

### 43. 为什么有时 `unordered_map` 不一定比 `map` 快？

  

**答：**

  

虽然 `unordered_map` 平均复杂度是 O(1)，但实际性能还受很多因素影响：

  

- 哈希计算本身有开销；

- 哈希冲突会降低效率；

- rehash 有额外成本；

- 内存布局可能不如连续结构缓存友好；

- 小数据量时，`map` 可能并不慢；

- 如果需要有序结果，`unordered_map` 还需要额外排序。

  

面试回答：

  

> 算法复杂度只是一个维度，实际性能还要看数据规模、哈希函数、内存访问、是否需要有序等因素。

  

---

  

## 十一、`set`

  

### 44. `set` 的底层原理是什么？

  

**答：**

  

`set` 通常基于红黑树实现。

  

特点：

  

- 元素唯一；

- 自动排序；

- 插入、删除、查找复杂度通常是 O(log n)。

  

示例：

  

```cpp

set<int> s;

s.insert(3);

s.insert(1);

s.insert(2);

s.insert(2);

  

for (int x : s) {

    cout << x << " ";

}

```

  

输出：

  

```cpp

1 2 3

```

  

重复的 `2` 不会插入。

  

---

  

### 45. `set` 和 `map` 有什么区别？

  

**答：**

  

| 对比点 | `set` | `map` |

|---|---|---|

| 存储内容 | 只存 key | 存 key-value |

| key 是否唯一 | 唯一 | 唯一 |

| 是否有序 | 有序 | 按 key 有序 |

| 底层结构 | 红黑树 | 红黑树 |

  

可以理解为：

  

```cpp

set<Key>

```

  

只关心 key 是否存在。

  

```cpp

map<Key, Value>

```

  

既关心 key，也关心 key 对应的 value。

  

---

  

## 46. `set` 中的元素可以修改吗？

  

**答：**

  

通常不能直接修改。

  

因为 `set` 中的元素本身就是 key，容器依赖元素值维持有序结构。如果直接修改元素，可能破坏红黑树的排序关系。

  

错误示例：

  

```cpp

set<int> s = {1, 2, 3};

auto it = s.find(2);

*it = 5; // 错误

```

  

正确做法是先删除，再插入：

  

```cpp

s.erase(2);

s.insert(5);

```

  

---

  

## 47. `set`、`unordered_set` 有什么区别？

  

**答：**

  

| 对比点 | `set` | `unordered_set` |

|---|---|---|

| 底层结构 | 红黑树 | 哈希表 |

| 是否有序 | 有序 | 无序 |

| 查找复杂度 | O(log n) | 平均 O(1)，最坏 O(n) |

| 元素要求 | 能比较大小 | 能哈希和判断相等 |

  

如果需要有序，选 `set`。

  

如果只判断是否存在，且不关心顺序，选 `unordered_set`。

  

---

  

# 十二、常见代码题

  

## 48. 判断下面代码有什么问题

  

```cpp

vector<int> v;

v.reserve(10);

v[0] = 1;

```

  

**答：**

  

有问题。

  

`reserve(10)` 只是预留容量，并没有创建元素。此时：

  

```cpp

v.size() == 0

```

  

访问 `v[0]` 是未定义行为。

  

正确写法：

  

```cpp

vector<int> v;

v.resize(10);

v[0] = 1;

```

  

或者：

  

```cpp

vector<int> v;

v.reserve(10);

v.push_back(1);

```

  

---

  

## 49. 判断下面代码有什么问题

  

```cpp

vector<int> v = {1, 2, 3};

auto it = v.begin();

v.push_back(4);

cout << *it << endl;

```

  

**答：**

  

`push_back(4)` 可能触发扩容。如果发生扩容，`vector` 会重新分配内存，原来的迭代器 `it` 失效。

  

继续使用：

  

```cpp

*it

```

  

是未定义行为。

  

更安全的写法是插入后重新获取迭代器：

  

```cpp

v.push_back(4);

auto it = v.begin();

cout << *it << endl;

```

  

---

  

## 50. 判断下面代码有什么问题

  

```cpp

const char* foo() {

    string s = "hello";

    return s.c_str();

}

```

  

**答：**

  

错误。

  

`s` 是局部变量，函数结束后会被销毁，`s.c_str()` 返回的指针指向 `s` 内部空间。

  

函数返回后，该指针变成悬空指针。

  

正确写法：

  

```cpp

string foo() {

    string s = "hello";

    return s;

}

```

  

如果确实需要 C 风格字符串，应确保底层存储的生命周期足够长。

  

---

  

## 51. 判断下面代码有什么问题

  

```cpp

map<string, int> mp;

  

if (mp["Tom"] == 0) {

    cout << "not found" << endl;

}

```

  

**答：**

  

问题是：

  

```cpp

mp["Tom"]

```

  

如果 `Tom` 不存在，会自动插入：

  

```cpp

{"Tom", 0}

```

  

所以这段代码不是单纯的查找，它会改变容器内容。

  

如果只是判断是否存在，应该写：

  

```cpp

if (mp.find("Tom") == mp.end()) {

    cout << "not found" << endl;

}

```

  

C++20 可以写：

  

```cpp

if (!mp.contains("Tom")) {

    cout << "not found" << endl;

}

```

  

---

  

## 52. 判断下面代码有什么问题

  

```cpp

map<int, string> mp = {{1, "a"}, {2, "b"}};

  

for (auto it = mp.begin(); it != mp.end(); ++it) {

    if (it->first == 1) {

        mp.erase(it);

    }

}

```

  

**答：**

  

`erase(it)` 后，`it` 已经失效，循环末尾还会执行 `++it`，这是危险的。

  

正确写法：

  

```cpp

for (auto it = mp.begin(); it != mp.end(); ) {

    if (it->first == 1) {

        it = mp.erase(it);

    } else {

        ++it;

    }

}

```

  

---

  

## 53. 判断下面代码有什么问题

  

```cpp

int arr[5] = {1, 2, 3, 4, 5};

  

void func(int arr[]) {

    cout << sizeof(arr) << endl;

}

```

  

**答：**

  

函数参数中的 `int arr[]` 会退化为 `int* arr`。

  

所以：

  

```cpp

sizeof(arr)

```

  

得到的是指针大小，不是数组大小。

  

如果要获得数组长度，可以使用模板引用：

  

```cpp

template <size_t N>

void func(int (&arr)[N]) {

    cout << N << endl;

}

```

  

---

  

# 十三、面试总结版

  

## 1. `string`

  

> `string` 底层类似动态字符数组，自动管理内存。`c_str()` 返回内部字符数组指针，生命周期依赖原 `string` 对象，不能在原对象销毁后继续使用，也要注意修改字符串后指针可能失效。

  

## 2. `vector`

  

> `vector` 底层是连续动态数组，支持 O(1) 随机访问，尾插均摊 O(1)，中间插入删除 O(n)。扩容时会重新分配内存，导致原有迭代器、指针、引用失效。

  

## 3. 迭代器

  

> 迭代器是访问容器元素的工具，是容器和算法之间的桥梁。`begin()` 指向首元素，`end()` 是尾后迭代器，不能解引用。

  

## 4. 迭代器失效

  

> `vector` 扩容会导致所有迭代器失效；`erase` 会导致删除位置及其后面的迭代器失效。边遍历边删除时，要使用 `erase` 返回的新迭代器。

  

## 5. 数组与指针

  

> 数组名在很多表达式中会退化为指向首元素的指针，但数组和指针不是同一个东西。`sizeof(arr)` 是整个数组大小，`sizeof(p)` 是指针大小。数组作为函数参数时会退化为指针。

  

## 6. `map`

  

> `map` 通常基于红黑树，按 key 有序，查找、插入、删除复杂度是 O(log n)。`operator[]` 在 key 不存在时会插入默认值，单纯查找应使用 `find` 或 `contains`。

  

## 7. `unordered_map`

  

> `unordered_map` 通常基于哈希表，无序，平均查找、插入、删除 O(1)，最坏 O(n)。如果插入触发 rehash，迭代器可能全部失效。

  

## 8. `set`

  

> `set` 通常基于红黑树，元素唯一且有序，元素本身就是 key，不能直接修改。需要修改时应先删除再插入。

  

## 9. `map` 和 `unordered_map`

  

> `map` 有序，适合范围查询和有序遍历；`unordered_map` 无序，适合快速查找。需要有序用 `map`，只关心平均查找效率用 `unordered_map`。

  

## 10. `resize` 和 `reserve`

  

> `resize` 改变元素个数，会创建或销毁元素；`reserve` 只预留容量，不改变 `size`，也不会创建元素。`reserve` 后不能直接通过下标访问未创建的元素。

  

---

  

# 十四、最高频必背题

  

## 必背 1：`vector` 底层原理

  

`vector` 是连续动态数组，支持 O(1) 随机访问。尾部插入均摊 O(1)，中间插入删除 O(n)。容量不足时会扩容，重新申请更大内存并移动或拷贝元素，原迭代器、指针、引用可能失效。

  

## 必背 2：`map` 和 `unordered_map` 区别

  

`map` 基于红黑树，有序，查找 O(log n)，支持范围查询。`unordered_map` 基于哈希表，无序，平均查找 O(1)，最坏 O(n)，插入时可能 rehash 导致迭代器失效。

  

## 必背 3：迭代器失效场景

  

`vector` 扩容导致全部迭代器失效；`insert` 如果不扩容，插入位置及其后面的迭代器失效；`erase` 使删除位置及其后面的迭代器失效。`map`、`set` 删除时一般只有被删除元素的迭代器失效。`unordered_map` rehash 会导致迭代器失效。

  

## 必背 4：`resize` 和 `reserve` 区别

  

`resize` 改变 `size`，会真正创建或销毁元素；`reserve` 只改变 `capacity`，不创建元素。`reserve` 后 `size` 仍然可能是 0，因此不能直接使用下标访问。

  

## 必背 5：`string::c_str()` 生命周期

  

`c_str()` 返回指向 `string` 内部字符数组的 `const char*`。原 `string` 销毁后，或者字符串被修改并触发重新分配后，该指针可能失效。不能返回局部 `string` 的 `c_str()`。

  

---

  

# 十五、建议你下午写的 demo

  

## Demo 1：观察 `vector` 扩容

  

```cpp

#include <iostream>

#include <vector>

using namespace std;

  

int main() {

    vector<int> v;

  

    for (int i = 0; i < 20; ++i) {

        v.push_back(i);

        cout << "i = " << i

             << ", size = " << v.size()

             << ", capacity = " << v.capacity()

             << endl;

    }

  

    return 0;

}

```

  

观察重点：

  

- `size` 每次加 1；

- `capacity` 不是每次都加 1；

- 容量不足时会跳跃式增长。

  

---

  

## Demo 2：观察迭代器失效

  

```cpp

#include <iostream>

#include <vector>

using namespace std;

  

int main() {

    vector<int> v;

    v.reserve(3);

  

    v.push_back(1);

    v.push_back(2);

    v.push_back(3);

  

    auto it = v.begin();

  

    cout << "before push_back: " << *it << endl;

  

    v.push_back(4); // 可能触发扩容

  

    // 这里不要继续使用 it，因为它可能已经失效

    it = v.begin();

    cout << "after push_back: " << *it << endl;

  

    return 0;

}

```

  

---

  

## Demo 3：`map` 和 `unordered_map` 遍历顺序区别

  

```cpp

#include <iostream>

#include <map>

#include <unordered_map>

using namespace std;

  

int main() {

    map<int, string> mp;

    unordered_map<int, string> ump;

  

    for (auto [k, v] : {pair{3, "three"}, pair{1, "one"}, pair{2, "two"}}) {

        mp[k] = v;

        ump[k] = v;

    }

  

    cout << "map:" << endl;

    for (const auto& [k, v] : mp) {

        cout << k << " " << v << endl;

    }

  

    cout << "unordered_map:" << endl;

    for (const auto& [k, v] : ump) {

        cout << k << " " << v << endl;

    }

  

    return 0;

}

```

  

观察重点：

  

- `map` 会按 key 有序输出；

- `unordered_map` 输出顺序不确定。

  

---

  

## Demo 4：`string::c_str()` 生命周期

  

```cpp

#include <iostream>

#include <string>

using namespace std;

  

int main() {

    string s = "hello";

    const char* p = s.c_str();

  

    cout << p << endl;

  

    s += " world, this may cause reallocation";

  

    // p 可能已经失效，不应该继续依赖它

    const char* q = s.c_str();

    cout << q << endl;

  

    return 0;

}

```

  

---

  

## Demo 5：`resize` 和 `reserve` 区别

  

```cpp

#include <iostream>

#include <vector>

using namespace std;

  

int main() {

    vector<int> a;

    a.reserve(5);

  

    cout << "after reserve:" << endl;

    cout << "size = " << a.size() << endl;

    cout << "capacity = " << a.capacity() << endl;

  

    vector<int> b;

    b.resize(5);

  

    cout << "after resize:" << endl;

    cout << "size = " << b.size() << endl;

    cout << "capacity = " << b.capacity() << endl;

  

    b[0] = 100;

    cout << b[0] << endl;

  

    return 0;

}

```

  

观察重点：

  

- `reserve` 后 `size` 仍然是 0；

- `resize` 后 `size` 变为指定大小；

- 只有 `resize` 后才可以访问对应下标范围内的元素。