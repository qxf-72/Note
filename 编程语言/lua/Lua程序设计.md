<div style="text-align: center;">
  <img 
    src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-09-28_17-33-23.png"
    alt="封面"
    loading="lazy"
    style="max-width: 100%; height: auto; width: 500;"
  />
</div>

# 入门

## Lua 解释器

- `lua`：启动 Lua 交互式解释器。
- `lua prog.lua`：执行 Lua 程序 `prog.lua`。
- `lua -i prog.lua`：先执行 `prog.lua`，执行结束后进入交互模式。
- `lua -e '代码'`：直接执行命令行中给出的 Lua 代码。
    


```
lua -e 'print("hello lua")'
```

---
## 加载 Lua 文件

```
dofile("main.lua")
```

- 加载并执行 `main.lua`。
    
- `dofile` 不只可以在交互模式中使用，在普通 Lua 程序中同样可以调用。
    
- 每调用一次 `dofile`，文件通常都会重新加载并执行一次。
    
---
## 注释

单行注释：

```
-- 这是一行注释
```

多行注释：

```
--[[
这是
多行注释
]]
```

可以利用 Lua 的语法方便地临时启用/禁用一段代码：

```
--[[
print("不会执行")
--]]
```

在开头再增加一个 `-`：

```
---[[
print("会执行")
--]]
```

此时第一行和最后一行都变成普通单行注释，中间代码恢复执行。

---

## 类型

Lua 是动态类型语言，变量本身没有固定类型，**值具有类型**。

使用：

```
type(value)
```

可以得到值对应的类型名称，例如：

```
type(nil)        --> "nil"
type(true)       --> "boolean"
type(10)         --> "number"
type("hello")    --> "string"
type({})         --> "table"
type(print)      --> "function"
```

--

## 布尔值

Lua 中只有两个值表示逻辑假：

```
false
nil
```

其他所有值都表示真，包括：

```
0
""
{}
```

> Lua 与 C/C++ 不同：`0` 在 Lua 中是真值。

## `and` 和 `or`

`and` 和 `or` 不一定返回 boolean，而是直接返回操作数本身。

### `and`

```
a and b
```

规则：

- 如果 `a` 为假，返回 `a`。
- 如果 `a` 为真，返回 `b`。

### `or`

```
a or b
```

规则：

- 如果 `a` 为真，返回 `a`。
    
- 如果 `a` 为假，返回 `b`。

因此可以写出类似默认值的代码：

```
x = x or 10
```

---

<br/>


<br/>


# 数值

## Number 类型

在 Lua 5.3 中，`number` 有两种内部表示：

- **integer**：通常为 64 位有符号整数。
    
- **float**：通常为双精度浮点数（double）。
    

它们都属于：

```
type(x) == "number"
```

**具有相同算术值的 integer 和 float 在数值比较时相等：**

```
10 == 10.0    --> true
```
---
## `math.type`

可以使用：

```
math.type(x)
```

进一步区分一个 number 的内部表示：

```
math.type(10)      --> "integer"
math.type(10.0)    --> "float"
```
---
## 普通除法 `/`

`/` 执行浮点除法：

```
5 / 2    --> 2.5
```

即使两个操作数都是 integer，结果通常也是 float。

---
## Floor 除法 `//`

```
a // b
```

执行 **floor division**，结果向负无穷取整。

例如：

```
5 // 2      --> 2
-5 // 2     --> -3
```

注意：

```
-2.5 向负无穷取整 = -3
```

而不是 `-2`。

这与 C/C++ 的整数除法尤其需要区分：

```
Lua:      -5 // 2 = -3
C/C++:    -5 / 2  = -2   （整数除法向 0 截断）
```

---

## 取模 `%`

Lua 中：

```
a % b
```

与 floor division 满足：

```
a = (a // b) * b + (a % b)
```

因此可以利用取模操作把一个数截取到某个步长的整数倍：

```
x - x % 0.01
```

表示将 `x` **向负无穷方向截取到 0.01 的整数倍**。

例如，对正数而言可以近似理解为截掉两位小数之后的部分：

```
x = 3.14159

x - x % 0.01
-- 约为 3.14
```

类似地：

```
x - x % 0.001
```

是截取到 **0.001 的整数倍，即三位小数的粒度**，不是两位小数。

注意：由于浮点数存在表示误差，不应该把这种写法理解成可靠的十进制定点数处理方案。

---

## 取整函数

### `math.floor`

向负无穷取整：

```
math.floor(3.8)     --> 3
math.floor(-3.8)    --> -4
```

### `math.ceil`

向正无穷取整：

```
math.ceil(3.2)      --> 4
math.ceil(-3.2)     --> -3
```

### `math.modf`

将一个数拆分成：

```
整数部分 + 小数部分
```

例如：

```
local i, f = math.modf(3.14)

print(i)    --> 3
print(f)    --> 0.14
```

对于负数：

```
local i, f = math.modf(-3.14)

print(i)    --> -3
print(f)    --> -0.14
```

其中整数部分相当于**向 0 截断**。

因此可以记为：

```
math.floor  → 向 -∞
math.ceil   → 向 +∞
math.modf   → 整数部分向 0 截断，同时返回小数部分
```

---

<br/>


<br/>

# 字符串

## 字符串常量

- lua 中字符串是不可变值的，不可修改字符串中某个字符。
- 使用 `#` 获取字符串长度。
- 使用 `..` 连接字符串。
- 使用 `u{  }` 声明 utf-8 字符。

---




## 多行字符串

- 使用 `[==[   ]==]` 来声明多行字符串变量，其中 `=` 个数任意，但必须左右对称。
	- 这种机制对于注释依然有效，可以使用 `--[==[  ]==]` 进行多行注释。


---


## 类型转换



- 在需要字符串的地方出现了数值，会自动转换为字符串
```lua
print(10 .. 20)  -->1020 
```

- 算术运算中，只有两个操作数都是整型结果才会是整型，**任何字符串参与的算术运算都会被自动当成浮点数处理**。
```lua
print("10"+1)  -->11.0
```

- 字符串显式转化为数值：
	- `tonumber("-3")  --> -3`
	- `tonumber("10e3") -->1000.0`
	- 当不能进行转化时返回 nil
	- `tonumber("10",2)  -->返回2`，这是指定二进制的情况。
- 数值显式转化为字符串：
	- `print(tostring(10)=="10")  -->true`，这种转化总是有效的。

---

## 字符串标准库

**字符串标准库默认处理的是 1 byte 的字符**，例如 ascii 字符：
- `string.len("10")`：等价于 `#`
- `string.rep(s,n)`：将字符串 s 重复 n 次
- `string.reverse(s)`
- `string.upper(s)`
- `string.lower(s)`
- `string.sub(s,i,j)`：提取 s 中第 i 到第 j 个字符构成的字符串，**索引从 1 开始**。
	- 支持负数索引：-1 代指最后一个字符。
- `string.char（）`：将内部数值转化为字符表示。

```lua
print(string.char(97))  --> a
print(string.char(97,98,99))  --> abc
```

- `string.byte(s)`：返回 s 第一个字符的内部数值表示。
```lua
print(string.byte("abc",2))  -->98
print(string.byte("abc",1,3))  --> 97 98 99
```

- `string.format()`：和 C 语言中的 printf 类似，用于控制格式化输出。
- 模式匹配：

```lua
string.find("hello,world","wor")  --> 7 9
string.find("hello,world","war")  --> nil
```

- 字符串替换：
```lua
string.gsub("hello,world","l",".")  -->he..o,wor.d     3
```

---

## Unicode 编码

**以编码字符为单位**，而非字节个数：
- `utf8.len()`
```lua
utf8.len("你好")  --> 2
utf8.len("hi")  --> 2
```
- `utf8.char()`：类似于 `string.char`。
- `utf8.codepont()`：类似于 `string.byte`。


- 提取偏移量：
```lua
s = "你好，世界"
string.sub(s,utf8.offset(s,-2))  -->"世界"
```


---


<br/>


<br/>


# 表

表是 lua 语言中唯一的数据结构。

使用构造器表达式创建表：
```lua
a = {}
a.x = 10
print(a["x"])  --> 10
```


**当使用一个表给另一个表赋值时，不会进行深拷贝**：
```lua
a = {}
b = a  -- a和b是同一张表
```


## 表索引

- `a.x` 等价于 `a["x"]`。
- 当被用作索引时，任何能够转化成整型的浮点数都会转化为整型。所以 1 和 1.0 被用作索引时，指向同一个元素。

---

## 表构造器


- 使用初始化列表构造表时，索引从 1 开始。
```lua
numbers = {"1","2","3","4","5"}
print(numbers[1]) --> 1 
```
- 初始化记录式：
```lua
a = {x=10,y=20}
```
- 以上两种方式可以混用。
- 显式指定索引：
```lua
a = {["x"]=10, ["y"]=20}
```

---

## 序列


序列：由 n 个正整数类型键组成的集合 {1,..., n } 形成的表。

使用 `#` 获取序列长度：
```lua
print(a[#a])  --输出最后一个元素
a[#a] = nil  --移除最后一个元素
a[#a+1] = 4  --在序列末尾插入元素
```

---

## 遍历表

使用 `pairs` 遍历表中键值对：

```lua
t = { 10, print, x = 12, k = "hi" }
for k, v in pairs(t) do
    print(k, v)
end
```


对于**序列**，使用 `ipairs` 顺序遍历表中键值对：

```lua
t = { 10, print, 12, "hi" }
for k, v in ipairs(t) do
    print(k, v)
end
```

使用数值型 for 循环：

```lua
t = { 10, print, 12, "hi" }
for k = 1, #t do
    print(k, t[k])
end
```


---

## 表标准库

`table.insert()`：向指定位置插入元素，不指定位置默认在最后插入。

`table.remove()`：删除指定位置元素，然后剩余元素向前填充空洞，不指定位置默认序列最后一个元素。

`table.move(a,f,e,t)`：将表 a 中 f 到 e 的元素移动到 t 上面。


---


<br/>


<br/>

# 函数

当输入参数个数和函数定义时的参数数量不一致时，lua 会抛弃多余参数，或者将不足的参数设置为 nil。

## 多返回值

当函数作为一条单独语句调用时，其所有返回值都会被丢弃。

**当函数作为表达式调用时，只保留第一个返回值。如果函数是一系列表达式最后一个或者唯一一个时，才会保留所有返回值**。

```lua
function func()
    return "a", "b"
end

print(func())       --> a b
print(1, func())    --> 1 a b
print(func(), 1)    --> a 1

```

**将函数调用用括号括起来，可以强制返回第一个结果**。
```lua
print((func()))    --> a
```


---

## 可变长参数函数


收集参数的方式：
- 表达式 `{...}`：返回一个由参数组成的表。
- `table.pack(...)`：同样是返回一个参数组成的表，但是多了一个键 `n` 表示参数数量。
- `select(n,...)`：返回第 `n` 个参数以及之后的所有参数，当 `n` 为 `#` 时返回参数数量。

```lua
function add(...)
    local s = 0
    for _, v in ipairs { ... } do
        s = s + v
    end
    return s
end
```

```lua
function add(...)
    local s = 0
    local arg = table.pack(...)
    for i = 1, arg.n do
        s = s + arg[i]
    end
    return s
end
```


```lua
function add(...)
    local s = 0
    for i = 1, select("#", ...) do
        s = s + select(i, ...)
    end
    return s
end
```


---


## `table.unpack`

- `table.pack()`：将参数转化为表。
- `table.unpack()`：将表转化为参数。常用于将表转化为一系列参数传递给变长参数函数。
	- `table.unpack(a,i)`：返回列表中第 i 个以及之后的元素。
	- `table.unpack(a,i,j)`：返回列表中第 i 到第 j 个元素。

```lua
local a = { "1", "2", "3" }
print(table.unpack(a))
```

---

## 尾调用


尾调用消除：lua 在尾调用时不使用额外的栈空间。

需要注意的是，以下情况都不是尾调用：
- `function f(x) g(x) end`：原因是在调用完 g 之后需要返回 f 中丢弃所有结果。
- `return (g(x))`：丢弃部分结果，返回值限制为 1 个。



---


<br/>


<br/>


# 闭包


## 函数是第一类值


Lua 中的函数是一等值，可以像其他值一样赋值、传参和返回。Lua 的函数值本身没有固定的名字，通常所谓的“函数名”只是保存或引用函数值的变量名，例如 `function f() ... end` 本质上可以理解为 `f = function() ... end` 的语法糖。

从使用方式上看，Lua 的函数与 C++ lambda 有一定相似性：它们都可以形成闭包并捕获外部变量。不过二者的底层实现不同。C++ lambda 会由编译器生成一个匿名的闭包类型，其对象保存捕获的数据；Lua 则由虚拟机通过 closure 和 upvalue 等机制保存函数及其捕获的外部变量。



---

## 非全局函数

函数可以存储在表字段中，有一类特殊语法定义此类函数：

```lua
Lib = {}
function Lib.foo(x, y)
    return x + y
end

```


---



<br/>


<br/>

# 编译、执行和错误



## 编译


`dofile` 的实现可以视作：

```lua
function dofile(filename)
    local f = assert(loadfile(filename))
    return f()
end
```

其中 `loadfile` 编译代码，然后将编译之后的代码段作为一个函数返回，**不运行代码**。

函数 `load` 和 `loadfile` 类似，不过是从字符串或者函数中读取代码段。

```lua
local f = load ("i = i + 1")
```

**函数 `load` 在编译时不涉及词法定界，总是在全局环境下编译代码。**

```lua
i = 42
local i = 0
f = load("i=i+1;print(i)")
f()   --43
```

---


## 预编译


lua 语言会在运行代码之前，对源代码进行预编译。

生成预编译文件：`luac -o prog.lc prog.lua`

函数 `loadfile` 可以接受预编译代码文件。

使用预编译代码的好处之一是可以避免意外修改源代码。


---

## 错误处理



在 lua 中处理错误，使用 `pcall` 函数：将需要处理错误的代码封装成一个函数，然后作为参数传递给 `pcall`。

`pcall` 会运行第一个参数（即函数），无论函数是否引发错误其都不会引发错误。
- 当有错误发生时，其返回 false；
- 如果没有错误发生，返回 true 和函数的所有返回值。

```lua
function func()
    return 1, 2, 3
end

print(pcall(func))  -- true 1 2 3
```

---


<br/>


<br/>


# 模块和包


## `require` 到底干了什么？

```text
寻找模块
  ↓
加载代码
  ↓
执行代码
  ↓
取得模块返回值
  ↓
缓存
  ↓
返回给调用者
```

---

## 模块内部如何实现 `private`？

使用 `local` 即可隐藏接口，如下所示：

```lua
local M = {}

local function check_number(x)
    if type(x) ~= "number" then
        error("expected number")
    end
end

function M.square(x)
    check_number(x)
    return x * x
end

return M
```

另外一种最后导出出口的写法：

```lua
local function add(a, b)
    return a + b
end

local function sub(a, b)
    return a - b
end

local function check(x)
    -- private
end

return {
    add = add,
    sub = sub
}
```


---

## lua 在哪里寻找模块？

`package.path` 可查看默认的寻找路径，`package.cpath` 则是在寻找不到 lua 文件之后寻找 C 动态库的路径。

可以使用 `package.searchpath` 排查模块是否存在的问题：

```lua
local path, err =
    package.searchpath("mymath", package.path)
print(path)
```

另外当输入的模块名称带有 `.` 号时，会自动将 `.` 替换成 `/` 号以便执行路径搜索。

---


## 模块和包的区别？

一个包可以组织多个相关的模块。

`init.lua` 可以作为包的入口，当搜索不到 `./mynet.lua` 时，会搜索 `./mynet/init.lua`。而 `init.lua` 文件中的内容如下：

```lua
-- mynet/init.lua

local M = {}

M.http = require("mynet.http")
M.tcp  = require("mynet.tcp")

return M
```


---


## `require` 完整执行流程

```text
local M = require("net.http")
                  │
                  ▼
        package.loaded["net.http"] ?
             │              │
            有             没有
             │              │
             ▼              ▼
          直接返回       package.searchers
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
        preload        Lua module      C module
                          │               │
                    package.path      package.cpath
                          │               │
                          └───────┬───────┘
                                  ▼
                              找到 loader
                                  │
                                  ▼
                               执行它
                                  │
                                  ▼
                              得到返回值
                                  │
                                  ▼
                    package.loaded[name]
                                  │
                                  ▼
                                return
```



---


<br/>


<br/>


# 迭代器

## 泛型 for


**lua 中的迭代器本质上是一个函数。**

```lua
function values(t)
    local i = 0
    return function()
        i = i + 1
        return t[i]
    end
end
```

其中 values 是迭代器工厂，只运行一次，返回值函数才是迭代器函数，每访问一次元素就运行一次。

迭代器函数使用方式：

```lua
for v in values({10, 20, 30}) do
    print(v)
end
```

其本质上和以下是等价的：

```lua
local iter = values({10, 20, 30})

while true do
    local v = iter()
    if v == nil then
        break
    end
    print(v)
end
```

**泛型 for 实际上保持的内容**：
- 迭代函数：即迭代器函数，**泛型 for 是否结束，只看迭代器返回的第一个值是不是 nil。**
- 恒定状态：需要遍历的对象
- 控制变量：可以理解为 index。

以上讨论的都是有状态迭代器，即状态都保持在闭包里面，例如 i 和 t。

---

## 无状态迭代器

迭代器函数本身闭包中不保持状态，状态由泛型 for 保存。例如 ipairs：

```lua
local function iter(t, i)
    i = i + 1

    local v = t[i]

    if v ~= nil then
        return i, v
    end
end

function my_ipairs(t)
    return iter, t, 0
end
```

```text
迭代函数      iter
恒定状态      t
控制变量初值   0
```

---

## 小结

Lua 的 generic `for` 本质上反复执行 `iterator(state, control)`，把第一个返回值作为新的 control，并在它变成 `nil` 时结束；迭代状态既可以保存在 closure 的 upvalue 中，也可以交给 generic `for` 的 state/control 来保存。


```text
                   iterator factory
                        │
                        │ 初始化一次
                        ▼
         ┌────────────────────────────┐
         │ iterator function          │
         │ invariant state            │
         │ initial control value      │
         └─────────────┬──────────────┘
                       │
                       ▼
                  generic for
                       │
            ┌──────────┴──────────┐
            ▼                     │
iterator(state, control)           │
            │                     │
            ▼                     │
    new_control, value             │
            │                     │
            ├── nil → 结束         │
            │                     │
            └── 非 nil ────────────┘
```

---


<br/>


<br/>

# 元表和元方法

元表决定一个值遇到某些特殊操作时应该表现成什么样；元方法就是元表中负责这些特殊操作的字段。

## 元表和元方法的使用


```lua
local a = { x = 1, y = 2 }
local b = { x = 3, y = 4 }

local mt = {}
mt.__add = function(a, b)
    return { x = a.x + b.x, y = a.y + b.y }
end

setmetatable(a, mt)
setmetatable(b, mt)

print(getmetatable(a))


local c = a + b
```

操作符对应的元方法：

```text
操作符           元方法

a + b          __add
a - b          __sub
a * b          __mul
a / b          __div
a // b         __idiv
a % b          __mod
-a             __unm
==             __eq
<              __lt
<=             __le
```

---

## `__tostring`

`print` 会使用 `tostring`，而 `tostring` 会检查对象是否提供了 `__tostring`。使用元方法 `__tostring` 可以使对象更方便打印。

---

## `__index` 和 `__newindex`

- `__index`：访问不存在字段时触发。
- `__newindex`：给原来不存在的字段赋值时触发。

### `__index` 不一定是函数

```lua
local prototype = {
    x = 0,
    y = 0,
    width = 100,
    height = 100
}

local mt = {
    __index = prototype
}

local window = {
    x = 10,
    y = 20
}

setmetatable(window, mt)
print(window.x)       -- 10
print(window.width)   -- 100
```

### `rawget` 和 `rawset`

当 `__newindex` 拦截赋值之后，不能直接赋值，否则会再次调用 `__newindex` 陷入死循环。应该**使用 `rawset` 跳过元表机制**。


### 代理对象

将真实数据放在 data 对象中，然后通过一个空的 proxy 对象访问。可以利用该机制实现只读对象。

```lua
local data = {
    x = 10,
    y = 20
}
local proxy = {}
local mt = {}

mt.__index = function(_, key)
    print("read:", key)
    return data[key]
end

mt.__newindex = function(_, key, value)
    print("write:", key, value)
    data[key] = value
end

setmetatable(proxy, mt)
```


### `__metatable`：保护元表


```lua
mt.__metatable = "protected"
```

执行上面语句之后 `getmetatable` 只会返回 protected，而 `setmetatable` 会失败。

---




<br/>


<br/>

# 垃圾回收

Lua 使用垃圾回收器自动管理大部分 Lua 对象的内存。

Lua 5.3 使用 **增量式 Mark-and-Sweep GC（标记-清除垃圾回收）**。

GC 判断对象是否应该回收的核心依据是：

> **可达性（reachability）**

当一个对象无法再从 Lua 程序的根对象出发访问到时，它就成为不可达对象，可以被 GC 回收。

> **对象变成不可达，并不意味着立即释放。**

真正什么时候释放由垃圾回收器决定。

字符串、table、Lua 函数、closure、thread、full userdata 等由 Lua 管理的对象，都可能由 GC 管理生命周期。

---

## GC 并不能完全避免“内存泄漏”

GC 只能判断：

> 这个对象还能不能访问到？

它无法判断：

> 这个对象从业务逻辑上还有没有意义？

例如：

```lua
local cache = {}

while true do
    local obj = create_object()
    cache[#cache + 1] = obj
end
```

即使以前创建的对象业务上已经没有任何用途：

```
cache
  ├── obj1
  ├── obj2
  ├── obj3
  └── ...
```

由于 `cache` 始终强引用这些对象，GC 会认为它们仍然存活，因此不会回收。

所以：

> **Lua 有 GC，但仍然可能发生逻辑上的内存泄漏。**

常见原因包括：

- 全局 table 不断增长。
- cache 永远不删除旧对象。
- 回调、监听器长期保存对象引用。
- 不再需要的对象仍然被其他 table 引用。

---

## Weak Table

普通 Lua table 中保存的引用通常是：

```
strong reference
```

即强引用。

只要存在强引用，对象就不会因为 GC 而被回收。

有些场景中，我们希望：

> 可以通过这个 table 找到对象，但这个 table 本身不应该决定对象是否继续存活。

这时可以使用：

**weak table（弱表）**

Lua 通过元表中的：

```
__mode
```

指定弱引用方式。

```text
__mode = "k"     weak key
__mode = "v"     weak value
__mode = "kv"    key 和 value 都是 weak
```


### Weak Value

例如：

```
local cache = {}

setmetatable(cache, {
    __mode = "v"
})

local obj = {
    name = "hello"
}

cache[1] = obj
```

此时：

```
obj ─────────────→ object
                     ↑
cache[1] - - - - - -┘
           weak
```

`obj` 是强引用，而 `cache[1]` 是弱引用。

执行：

```
obj = nil
```

之后：

```
cache[1] - - - → object
          weak
```

如果没有其他强引用，对象就可以被 GC 回收。

也就是说：

> weak table 可以引用对象，但这种引用本身不会保证对象继续存活。

### Weak Value 特别适合 Cache

```lua
function load_image(name)
    -- 加载图片，可能十分耗时
end

local cache = {}

setmetatable(cache, {
    __mode = "v"
})

function get_image(name)
    local img = cache[name]

    if img ~= nil then
        return img
    end

    img = load_image(name)
    cache[name] = img

    return img
end
```

其生命周期关系是：

```text
程序正在使用 image

业务代码 ────────→ image
                     ↑
cache[name] - - - - -┘
               weak
```

对象仍然存活。

当业务代码不再引用 image：

```
cache[name] - - - → image
               weak
```

cache 不会阻止 GC 回收它。

因此 weak-value cache 很适合：

> **有缓存就复用，没有缓存就重新计算或加载。**

这样可以避免 cache 自己成为所有缓存对象永久存活的原因。

---

## Weak Key

也可以让 table 的 key 成为弱引用：

```
local attrs = {}

setmetatable(attrs, {
    __mode = "k"
})

local obj = {}

attrs[obj] = {
    color = "red",
    size = 10
}
```

这里可以把：

```
attrs[obj]
```

理解成：

> 给 `obj` 保存额外的 metadata。

关系类似：

```
attrs
  │
  └ - - → obj ───→ metadata
       weak key
```

如果程序其他地方仍然有：

```
strong reference ───→ obj
```

那么记录继续存在。

当程序其他地方彻底不再引用 `obj` 后：

```
obj = nil
```

weak key 不会阻止对象被 GC 回收。

`obj` 被回收后：

```
attrs
```

中对应的 key-value 记录也会消失。

因此 weak key 很适合：

> **给对象附加额外信息，但不希望保存这些信息的 table 决定对象本身的生命周期。**

例如，对象由别的模块创建，不方便直接修改：

```
local socket = get_socket()
```

可以：

```
local socket_info = {}

setmetatable(socket_info, {
    __mode = "k"
})

socket_info[socket] = {
    last_active = os.time()
}
```

当 `socket` 在其他地方不再被引用并被 GC 后，对应的：

```
socket_info[socket]
```

也会自动消失。

这种模式常用于：

```
对象 metadata
对象关联信息
监听器注册表
Lua/C 对象关联
辅助映射表
```

---

## Weak Key 和 Weak Value 的典型用途

可以简单记成：

```
weak value
    ↓
“value 不应该因为 cache 而继续活着”
    ↓
cache / memoization

weak key
    ↓
“metadata 不应该让 key 对象继续活着”
    ↓
object metadata / object association
```

具体工程中也可以使用：

```
__mode = "kv"
```

让 key 和 value 都是弱引用。

---

## Weak Table 的特殊情况

并不是 Lua 中的所有值都会因为出现在 weak table 中而“消失”。

例如：

```
number
boolean
light userdata
light C function
```

这些值本身不是普通的 GC 对象，因此不存在“因为只有弱引用而被垃圾回收”的问题。

### String 是一个特殊情况

Lua string 虽然由垃圾回收器管理，但 Lua 对 weak table 中的 string 有特殊处理。

可以粗略理解为：

> string 是值语义的对象，没有程序能够观察到的独立对象身份，因此 Lua 不会像处理 table、closure、thread、userdata 那样，因为它只存在于 weak table 中就删除对应项。

因此测试 weak table 时，更适合使用：

```
table
closure
thread
full userdata
```

这类具有明显对象身份的 GC 对象。

---

## Ephemeron Table

当：

```
__mode = "k"
```

即：

```
weak key
strong value
```

时，这种 table 具有特殊的 **ephemeron** 语义。

考虑：

```lua
local t = setmetatable({}, {
    __mode = "k"
})

local key = {}

local value = {
    owner = key
}

t[key] = value

key = nil
```

引用关系为：

```
          weak
t - - - - - - → key
│                ▲
│                │
└──────→ value ──┘
 strong          strong
```

也就是说：

```
t --weak--> key

t --strong--> value

value --strong--> key
```

如果按照普通强引用规则：

```
t → value → key
```

那么 `key` 会一直保持存活。

这样：

```
weak key
```

就失去了意义。

因此 ephemeron table 有特殊规则：

> **value 是否因为 table 而被认为可达，要取决于对应的 key 是否已经通过其他途径可达。**

换句话说：

> 如果 key 只能通过与它关联的 value 反过来找到，那么这个 value 不能靠这种关系把 key “救活”。

因此在上面的例子中，如果没有其他外部强引用：

```
key
```

最终仍然可以被回收，同时这一整条 key-value 记录也会被删除。

可以简单记成：

```
metadata[key] = value

即使：
value → key

也不能让已经没有外部引用的 key
因为自己的 metadata 而永远存活。
```

---

## Finalizer

Lua 可以通过：

```
__gc
```

定义 finalizer。

finalizer 的主要用途通常不是释放普通 Lua 内存，因为普通 Lua 对象的内存本来就由 GC 管理。

它更适合负责清理：

> **Lua GC 本身并不了解的外部资源。**

例如：

```
FILE*
socket fd
数据库连接
C/C++ malloc 出来的对象
GPU resource
OS handle
```

例如：

```
local mt = {
    __gc = function(obj)
        -- 清理外部资源
    end
}

local obj = setmetatable({}, mt)
```

当 `obj` 变成不可达，并且 Lua 进行相应的垃圾回收时，会在对象真正被释放前执行它的 finalizer。

### `__gc` 与 C++ 析构函数不同

不要简单理解为：

```
__gc == C++ destructor
```

C++ RAII：

```
{
    Foo foo;
}
```

自动存储期对象离开作用域时：

```
立即、确定地调用析构函数
```

类似：

```
{
    std::unique_ptr<Foo> foo;
}
```

RAII owner 离开作用域，也会确定性释放资源。

Lua：

```
do
    local obj = ...
end
```

离开作用域只意味着：

> `obj` 对对象的引用消失了。

如果对象因此变成不可达，它什么时候执行：

```
__gc
```

仍然取决于：

```
GC 什么时候真正运行
```

所以：

```
C++ RAII / destructor
        ↓
确定性资源释放

Lua __gc
        ↓
非确定性的 finalization
```

因此对于：

```
socket
文件
数据库连接
锁
```

等资源，最好仍然提供显式的：

```
obj:close()
```

或类似操作。

`__gc` 更适合作为最后一道保险，而不是依赖它及时释放关键系统资源。

### `__gc` 的一个注意点

如果希望某个对象被注册为需要 finalization 的对象，通常应该在给对象设置元表之前就准备好：

```
__gc
```

例如：

```
local mt = {
    __gc = function(obj)
        ...
    end
}

local obj = setmetatable({}, mt)
```

不要依赖这种方式：

```
local mt = {}

local obj = setmetatable({}, mt)

-- 之后才添加
mt.__gc = function(obj)
end
```

因为 Lua 在对象设置元表时决定是否将其标记为需要 finalization。

---

## `collectgarbage`

Lua 提供：

```
collectgarbage(option [, arg])
```

用于查询或控制垃圾回收器。


### `"collect"`

```
collectgarbage("collect")
```

要求执行一次完整的垃圾回收周期。

通常用于：

```
测试
调试
内存实验
```

不应该在普通业务代码中高频调用。



### `"count"`

```
local mem = collectgarbage("count")
```

返回 Lua 当前使用内存的大致大小，单位为：

```
KB
```

例如：

```
print(collectgarbage("count"))
```

可以用于观察程序的大致 Lua 内存变化。



### `"stop"`

```
collectgarbage("stop")
```

停止垃圾回收器的自动运行。


### `"restart"`

```
collectgarbage("restart")
```

重新启动自动 GC。

### `"step"`

```
collectgarbage("step")
```

让增量式垃圾回收器向前执行一个步骤。

也可以给出一个参数：

```
collectgarbage("step", n)
```

控制这一次 GC step 的工作量。

---

## Lua 5.3 的增量式 GC

最简单的垃圾回收方式可以想象成：

```
程序运行
程序运行
程序运行
    ↓
整个程序暂停
    ↓
完成大量 GC 工作
    ↓
继续程序
```

如果堆非常大，可能造成明显停顿。

Lua 5.3 使用增量式 GC，将垃圾回收工作拆成多个步骤：

```
程序运行一点
     ↓
GC 做一点
     ↓
程序继续
     ↓
GC 再做一点
     ↓
……
```

主要是在两个目标之间权衡：

```
GC 总体 CPU 开销
        ↕
单次 GC 停顿时间
```

---

## `setpause`

调用形式：

```
collectgarbage("setpause", value)
```

例如：

```
collectgarbage("setpause", 200)
```

它控制：

> **完成一轮 GC 后，需要等待内存增长到什么程度，再开始下一轮 GC。**

可以粗略理解为：

```
pause 大
    ↓
下一轮 GC 开始得更晚
    ↓
GC 相对不积极
    ↓
允许使用更多内存

pause 小
    ↓
下一轮 GC 更早开始
    ↓
GC 更积极
```

Lua 5.3 的默认值通常是：

```
200
```

它大致意味着内存增长到上一轮 GC 结束时的约两倍时，开始下一轮 GC。

---

## `setstepmul`

调用形式：

```
collectgarbage("setstepmul", value)
```

例如：

```
collectgarbage("setstepmul", 200)
```

它控制：

> **垃圾回收器相对于 Lua 内存分配速度工作的积极程度。**

可以粗略理解为：

```
stepmul 大
    ↓
GC 每次工作更多
    ↓
回收推进得更快

stepmul 小
    ↓
GC 每次工作较少
    ↓
回收推进得更慢
```

因此可以这样记：

```
pause
    → 什么时候开始下一轮 GC

step multiplier
    → GC 开始以后工作得多积极
```

---

## 手动控制 GC

在某些对延迟非常敏感的短时间段，可以暂时停止 GC：

```
collectgarbage("stop")

-- 延迟非常敏感的一小段代码

collectgarbage("restart")
```

也可以在相对空闲的时候主动执行：

```
collectgarbage("step")
```

把一部分 GC 工作放在较合适的时间执行。

但一般情况下：

> **优先使用 Lua 默认的 GC 参数。**

只有实际测量发现：

```
GC pause 过大
内存占用过高
延迟出现明显抖动
```

之后，才有必要调整 GC。

原则仍然是：

```
Measure first.
Optimize second.
```

---


<br/>


<br/>

# 协程

**Lua 协程（coroutine）可以理解为一种能够暂停，并在之后从暂停位置继续执行的执行流。**

一个 coroutine 会保存自己的：

```text
调用栈
局部变量
当前执行位置
```

因此，coroutine.yield() 暂停后，再次 coroutine.resume(co) 并不是重新执行协程函数，而是从上一次暂停的位置继续执行。

---

## Coroutine 与 Closure

闭包和协程都可以保存状态，但二者保存的东西不同：

```text
Closure
    ↓
函数 + 捕获的外部变量

Coroutine
    ↓
完整的执行状态
    ├── 调用栈
    ├── 局部变量
    └── 当前执行位置
```

例如闭包通常需要显式保存：

```lua
local i = 0
```

而 coroutine 可以直接暂停整个函数调用过程。因此 coroutine 特别适合：

```text
generator
复杂 iterator
producer-consumer
异步流程
事件驱动程序
```

---

## Coroutine 与线程

Lua 中：

```lua
type(coroutine.create(function() end))
```

返回：

```
"thread"
```

但这里的 `thread` 是 **Lua coroutine 对象**，并不是操作系统线程或 `pthread`。

Lua coroutine 使用：**协作式调度（cooperative scheduling）**：也就是说，一个正在运行的 coroutine 不会像 OS 线程那样被调度器在任意位置抢占。

控制权通常在以下情况交还出去：

```text
coroutine.yield()
协程函数正常 return
协程发生错误
```

因此：

```text
OS Thread
    ↓
通常由 OS 抢占式调度
    ↓
可能真正并行

Lua Coroutine
    ↓
由程序主动控制切换
    ↓
单个 Lua state 中通常一次只运行一个 coroutine
```

所以：

> **Coroutine 提供并发控制流，但本身不提供 CPU 并行。**

如果某个 coroutine 长时间运行而不 `yield`：

```text
while true do
    -- 一直计算
end
```

其他 coroutine 就没有机会运行。

---

## 协程的基本使用

### `coroutine.create`

创建 coroutine：

```lua
local co = coroutine.create(function(...)
    print("hello, coroutine")
    print(...)
end)
```

`create` 只是创建协程：

```
create
  ↓
suspended
```

并不会立即执行其中的函数。


### `coroutine.resume`

使用：

```lua
coroutine.resume(co, 1, 2, 3)
```

第一次 `resume` 时，额外参数会传递给 coroutine 的主函数：

```lua
local co = coroutine.create(function(a, b, c)
    print(a, b, c)
end)

coroutine.resume(co, 1, 2, 3)
```

相当于：

```
resume(co, 1, 2, 3)
          │
          ▼
function(1, 2, 3)
```

---

## `resume` 与 `yield`

### `yield` 暂停协程

例如：

```lua
local co = coroutine.create(function()
    print("A")

    coroutine.yield()

    print("B")
end)
```

第一次：

```
coroutine.resume(co)
```

输出：

```
A
```

此时 coroutine 暂停在：

```
coroutine.yield()
```

再次：

```
coroutine.resume(co)
```

从 `yield` 后继续执行：

```
B
```

---

## `resume` 与 `yield` 可以双向传递数据

这是 coroutine 最重要的机制之一。

### `yield` → `resume`


```lua
local co = coroutine.create(function()
    coroutine.yield(100)
end)

print(coroutine.resume(co))  -- true    100
```


```text
coroutine.yield(100)
          │
          ▼
coroutine.resume(co)
返回：
true, 100
```



### `resume` → `yield`

反过来：

```lua
local co = coroutine.create(function()
    local x, y = coroutine.yield()

    print(x, y)
end)

coroutine.resume(co)

coroutine.resume(co, 10, 20)  --10, 20
```

---

## `resume` 的返回值与错误处理

`coroutine.resume` 的第一个返回值表示执行是否成功：

```lua
local ok, value = coroutine.resume(co)
```

成功：

```
true, ...
```

失败：

```
false, error
```


因此实际代码中应该注意检查：

```lua
local ok, value = coroutine.resume(co)

if not ok then
    error(value)
end
```

---

## Coroutine 状态

可以使用：

```lua
coroutine.status(co)
```

查看状态。

主要有：

```text
suspended
    尚未启动，或者已经 yield

running
    当前正在运行

normal
    自己暂时没有运行，但它 resume 的另一个 coroutine 正在运行

dead
    已经正常结束，或者因为错误终止
```

`dead` coroutine 不能再次成功 `resume`。

---

## `coroutine.wrap`

`coroutine.wrap` 可以把 coroutine 包装成一个普通函数：

```lua
local f = coroutine.wrap(function()
    coroutine.yield(10)
    coroutine.yield(20)

    return 30
end)

print(f())    -- 10
print(f())    -- 20
print(f())    -- 30
```

可以理解为：

```
coroutine.create + coroutine.resume
              ↓
包装成普通函数调用形式
```

---

## 生产者—消费者

Coroutine 很适合 producer-consumer 模型。

基本关系：

```text
consumer
    │
    │ resume
    ▼
producer
    │
    │ yield(data)
    ▼
consumer
```

可以先封装：

```lua
function send(x)
    coroutine.yield(x)
end

function receive(prod)
    local ok, value = coroutine.resume(prod)

    if not ok then
        error(value)
    end

    return value
end
```

Producer：

```lua
function producer()
    return coroutine.create(function()
        while true do
            local x = io.read()

            if x == nil then
                return
            end

            send(x)
        end
    end)
end
```

Filter：

```lua
function filter(prod)
    return coroutine.create(function()
        local line = 1

        while true do
            local x = receive(prod)

            if x == nil then
                return
            end

            x = string.format("%5d %s", line, x)

            line = line + 1

            send(x)
        end
    end)
end
```

Consumer：

```lua
function consumer(prod)
    while true do
        local x = receive(prod)

        if x == nil then
            return
        end

        print(x)
    end
end
```

连接：

```lua
consumer(filter(producer()))
```

形成：

```
producer
    │
    ▼
 filter
    │
    ▼
consumer
```

这种结构可以类比 Unix Pipeline：

```
producer | filter | consumer
```

但二者有本质区别：

```text
Unix pipeline
    ↓
通常由多个进程组成
    ↓
OS 可以并发调度

Lua coroutine pipeline
    ↓
多个 coroutine
    ↓
通过 resume/yield 协作切换
```

---

## Coroutine 作为 Iterator / Generator

Coroutine 保存整个执行状态，因此很适合实现 iterator。

例如：

```lua
function values(t)
    return coroutine.wrap(function()
        for i = 1, #t do
            coroutine.yield(t[i])
        end
    end)
end
```

使用：

```lua
for x in values({10, 20, 30}) do
    print(x)
end
```

执行过程：

```
for 调用 iterator
      ↓
resume coroutine
      ↓
yield(10)
      ↓
for 得到 10

再次调用 iterator
      ↓
从 yield 后继续
      ↓
yield(20)
      ↓
for 得到 20
```

因此：

```
coroutine + yield
        ↓
generator
```

---

## Coroutine 特别适合复杂 Iterator

简单 iterator 可以用闭包保存：

```
local i = 0
```

但复杂遍历可能需要保存：

```text
递归调用栈
当前节点
遍历阶段
多个局部变量
```

例如二叉树中序遍历：

```lua
function traverse(node)
    if node == nil then
        return
    end

    traverse(node.left)

    coroutine.yield(node.value)

    traverse(node.right)
end
```

包装成 iterator：

```lua
function inorder(root)
    return coroutine.wrap(function()
        traverse(root)
    end)
end
```

使用：

```lua
for value in inorder(root) do
    print(value)
end
```

这里最大的好处是：

> 遍历代码仍然可以按照正常递归逻辑编写。

不需要手动维护：

```
stack
current node
state enum
visited flag
```

因为：

> **Coroutine 自己的调用栈就是遍历状态的一部分。**

可以对比：

```text
Closure Iterator
      ↓
显式保存需要的状态

Coroutine Generator
      ↓
整个暂停的执行过程就是状态
```

---

## Event-Driven Programming

Coroutine 与事件驱动程序结合非常重要。

例如 Linux 网络程序：

```
non-blocking socket
        +
      epoll
```

传统事件驱动模型通常是：

```
epoll_wait
    ↓
I/O ready
    ↓
callback
```

业务逻辑可能被拆成很多 callback。

我们真正希望编写的是：

```lua
local request = socket_read(client_fd)

local response = handle(request)

socket_write(client_fd, response)
```

看起来仍然是：

```
read
 ↓
process
 ↓
write
```

Coroutine 可以把异步等待隐藏起来。

概念上：

```lua
function socket_read(fd)
    -- 尝试非阻塞 read

    -- 如果得到 EAGAIN：
    -- 1. 向事件循环注册 fd 的 EPOLLIN
    -- 2. 保存当前 coroutine
    -- 3. 暂停当前 coroutine

    coroutine.yield()

    -- fd readable 后，
    -- event loop 会 resume 当前 coroutine

    return real_read(fd)
end
```

真正的控制过程是：

```
Coroutine

socket_read(fd)
      ↓
read 返回 EAGAIN
      ↓
注册 EPOLLIN
      ↓
yield
      │
      │
      ▼
Event Loop

epoll_wait()
      ↓
处理其他 fd
      ↓
fd readable
      ↓
找到正在等待该 fd 的 coroutine
      ↓
resume(co)
      │
      ▼
Coroutine

从 yield 后继续
      ↓
read(fd)
      ↓
返回数据
```

所以必须注意：

> `yield` **本身并不会自动等待 socket，也不会自动调用** `epoll` **。**

真正完成调度的是：

```
event loop / coroutine scheduler
```

它需要维护类似：

```
fd → waiting coroutine
```

的关系。

---

## Coroutine 与 epoll 的职责

可以这样理解：

```
epoll
    ↓
解决：
“哪个 fd 已经 ready？”

Coroutine
    ↓
解决：
“等待这个 fd 的业务逻辑如何暂停，
并在 ready 后从原位置继续？”

Scheduler
    ↓
负责：
把 epoll event 和对应 coroutine 连接起来
```

因此：

```
epoll
      +
coroutine
      +
scheduler
      ↓
异步底层 + 同步风格业务代码
```

这是 coroutine 在网络编程中非常重要的用途。

---

## Coroutine 适合什么，不适合什么？

Coroutine 特别适合：

```
I/O 等待
网络程序
generator
producer-consumer
复杂 iterator
事件驱动程序
异步工作流
状态机
```

但不应该把：

```
coroutine
```

理解成：

```
多核并行工具
```

例如一个 coroutine：

```
local co = coroutine.create(function()
    expensive_cpu_task()
end)
```

如果 `expensive_cpu_task()` 持续计算很久而不 `yield`：

```
其他 coroutine 一样无法运行
```

因此：

> Coroutine 的主要价值是**管理控制流和并发任务**，而不是提供 CPU 并行能力。

---