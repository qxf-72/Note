<div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-06-09_09-31-15.png" loading="lazy" style="max-width: 100%; height: auto; width: 500px;"/> </div>

# 线程管控

## 基本线程管理

### 启动线程：构建 thread 对象

- 函数符号
- 仿函数
- lambda 表达式

```cpp
void do_something()
{
    cout << "do something" << endl;
}

class background_task {
public:
    void operator()() const
    {
        do_something();
    }
};
int main()
{
    thread t1{background_task()}; //仿函数，加多一对括号，或者改为大括号，来避免被编译器识别为函数声明
    hread t2(do_something); //函数符
    thread t3([](){   // lambda表达式
        do_something();
    }); 
    
    t1.join();

    return 0;
}
```

### 需要显式调用 join 或者 detach

否则，thread 析构函数会调用 terminate 终止程序，即便有异常。

> terminate called without an active exception


### RAII

分离线程直接在启动之后调用 detach 即可。但是 join 不可以直接调用，否则就变成了同步函数了。为了确保能够显式调用 join，使用 RAII

因为不能对同一个线程 join 多次，所以需要判断 joinable 。

```cpp
void do_something()
{
    cout << "do something" << endl;
}

class thread_guard {
    thread &t;
public:
    explicit thread_guard(thread &t_) : t(t_){}
    ~thread_guard()
    {
        if (t.joinable()) {
            t.join();
        }
    }
    thread_guard(thread_guard const &) = delete;
    thread_guard &operator=(thread_guard const &) = delete;
};

int main()
{
    thread t(do_something);
    thread_guard t_g(t);
    
    return 0;
}
```


---
## 向线程传递参数

线程具有内部存储空间，参数会按照默认方式先**复制**到该处，新创建的执行线程才能直接访问它们。然后，这些副本被当成**临时变量**，以**右值**形式传给新线程上的函数或可调用对象


std::thread 创建线程时，会把参数保存起来再在线程中调用函数。  
  
- 默认是拷贝。  
- 要引用，用 std::ref。  
- 要移动，用 std::move。  
- 要调成员函数，传 &Class::func 和对象地址。  
- 只要传引用、指针、this，就必须保证对象活得比线程久。

<br/>


- 传指针时，线程保存的可能只是地址，不是内容本身。 

<div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-05-26_17-01-24.png" loading="lazy" style="max-width: 100%; height: auto; width: 400px;"/> </div>

- <div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-05-26_17-02-48.png" loading="lazy" style="max-width: 100%; height: auto; width: 700px;"/> </div>

- <div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-05-26_17-04-48.png" loading="lazy" style="max-width: 100%; height: auto; width: 700px;"/> </div>





---

## 识别线程


- 通过 thread 对象的 `get_id()` 获取
- 通过 `std::this_thread::get_di()` 获取当前线程的 id

---

<br/>


<br/>


<br/>


# 共享数据
## 线程间共享数据的问题

### 竞态条件

程序执行的结果依赖于多个线程的执行顺序，并且这种顺序不受程序控制，如果某些执行顺序会导致错误结果、数据损坏或未定义行为，这就是有害静态条件。

### data race

多个线程同时访问同一内存位置，其中至少一个是写操作，并且没有使用 mutex、atomic 等同步手段。

### 为什么 `if (!s.empty()) s.pop();` 在多线程中不一定安全？

因为 empty 和 pop 是两个独立的操作，线程 a 在判断 empty 之后，切换到线程 b 执行 pop 操作，如果在切换回线程 a 执行 pop 操作就会出错。所以在并发环境下，多个相关操作需要被作为同一个整体进行保护。

### 避免有害竞态条件的方法

- 不同享数据
- 同享数据，但是用锁保护
- 使用 atomic 类型


---
## 用 mutex 保护共享数据

### 不要手动 lock/unloc

推荐使用 `std::lock_guard<std::mutex> lock(m)` RAII 方式管理锁：构造时自动 lock，析构时自动 unlock。即便抛出异常，也能释放锁。

```cpp
std::mutex m;
void func()
{
    m.lock();
    do_something(); // 如果这里抛异常
    m.unlock();    // 这一句不会执行
}
```


### 把 mutex 和数据封装在同一个类里

因为 lock/unlock 会改变锁的状态，为了能在 const 成员函数中使用锁，需要把锁声明为 **mutable**。

```cpp
#include <vector>
#include <mutex>

class SafeVector {
private:
    std::vector<int> data;
    mutable std::mutex m;

public:
    void push_back(int x)
    {
        std::lock_guard<std::mutex> lock(m);
        data.push_back(x);
    }

    bool empty() const
    {
        std::lock_guard<std::mutex> lock(m);
        return data.empty();
    }

    std::size_t size() const
    {
        std::lock_guard<std::mutex> lock(m);
        return data.size();
    }
};
```

### 不要把受保护数据的引用或指针传出去


受 mutex 保护的数据，不要把裸指针或引用暴露给外部。否则外部可以绕过 mutex 来修改数据。

### 不能在持锁时调用外部传进来的函数

保护共享数据时，不要让受保护的数据逃逸到锁的作用域之外。调用外部函数可以会保存数据的地址，外部可以绕开 mutex 访问数据。

### 接口本身也可能有竞态问题

```cpp
if (!s.empty()) {
    int value = s.top();
    s.pop();
}
```

即便每个接口都加了锁，但是组合在一起时操作不是原子的。更合理的做法是把 `top()` 和 `pop()` 合并成一个操作。更工程化的 try_pop 写法：

```cpp
bool try_pop(T& value)
{
    td::lock_guard<std::mutex> lock(m);
    if (data.empty()) {
        return false;
    }
	value = std::move(data.top());
    data.pop();
    return true;
}
```

### 死锁问题

以下例子，两个现在**加锁顺序不一致**就可以导致死锁。

```cpp
std::mutex m1;
std::mutex m2;
void thread_a()
{
    std::lock_guard<std::mutex> lock1(m1);
    std::lock_guard<std::mutex> lock2(m2);
}
void thread_b()
{
    std::lock_guard<std::mutex> lock2(m2);
    std::lock_guard<std::mutex> lock1(m1);
}
```

避免死锁的方法：
- 固定加锁顺序
- <mark style="background:#fff88f">使用 `std::lock` </mark>：尝试以一种避免死锁的方式同时锁住两个 mutex。

```cpp
std::mutex m1;
std::mutex m2;

void func()
{
    std::lock(m1, m2);
    // 以下只是负责释放锁
    std::lock_guard<std::mutex> lock1(m1, std::adopt_lock);
    std::lock_guard<std::mutex> lock2(m2, std::adopt_lock);

    // 同时持有 m1 和 m2
}
```

<mark style="background:#fff88f">在 C++17 可以替换为：`std::scoped_lock lock(m1, m2);`</mark>


### 锁的粒度要合适

- 不要在持锁时做太多事情，否则会占用锁过长时间，其他线程长时间等待。
- 锁的范围也不能太小，否则保护不完整


### unique_lock

- 允许提前解锁
```cpp
std::unique_lock<std::mutex> lock(m);

// 访问共享数据
do_something_with_shared_data();

lock.unlock();

// 后面执行不需要锁的耗时操作
do_something_expensive();
```

- 允许延迟加锁

```cpp
std::mutex m;
std::unique_lock<std::mutex> lock(m,std::defer_lock);
// 此时没有加锁
lock.lock();
//现在才加锁
```

- **搭配条件变量使用**

---

## 保护数据的其他方式

### 只初始化一次

某个对象一开始不创建，等第一次用到时再创建。  但只能创建一次，而且多线程下也必须安全。

```cpp
std::mutex m;
Logger* logger = nullptr;

Logger* getLogger()
{
    std::lock_guard<std::mutex> lock(m);

    if (!logger) {
        logger = new Logger();
    }

    return logger;
}
```

即使 `logger` 已经初始化完成了，以后每次调用 `getLogger()` 仍然要加锁。如果 `getLogger()` 被频繁调用，这个锁就显得有点浪费。

C++ 提供了专门解决一次性初始化的工具：
```cpp
std::once_flag
std::call_once
```

```cpp
class Logger {
public:
    void log(const std::string &msg)
    {
        // ...
    }
    std::shared_ptr<Logger> logger;
    std::once_flag logger_flag;

    std::shared_ptr<Logger> get_logger()
    {
        std::call_once(logger_flag, [this]()->void{
            logger = std::make_shared<Logger>();
        });
        return logger;
    }
};
```

初始化函数正常返回：once_flag 标记为完成，以后不再执行。**初始化函数抛异常：once_flag 不标记完成，下次还会再试**。

其实有更简单的方式：

```cpp
class Logger {
public:
    static Logger& instance()
    {
        static Logger logger;
        return logger;
    }

    void log(const std::string& msg)
    {
        // ...
    }

private:
    Logger() = default;
};
```

从 C++11 开始，函数内局部静态变量的初始化是线程安全的。以上就是单例模式中的懒汉模式。
- 懒初始化
- 线程安全
- 不需要手动 new/delete

### 读多写少

对于读多写少的场景，即使是读，也需要排队等待取得锁，虽然不会破坏数据结构，真正需要注意的：
- 读多时候有人写
- 写的时候有人写

C++17 提供了 `std::shared_mutex` ，允许两种加锁方式：
- 共享锁 `std::shared_lock<std::shared_mutex>`：多个线程可以同时持有，用于读操作
- 独占锁 `std::unique_lock<std::shared_mutex>`：同一时刻只能一个线程持有，用于写操作

```cpp
class Config {
private:
    std::map<std::string, std::string> data;
    mutable std::shared_mutex m;

public:
    std::string get(const std::string &key) const
    {
        std::shared_lock<std::shared_mutex> lock(m);
        auto it = data.find(key);
        if (it == data.end()) {
            return "";
        }
        return it->second;
    }
    void set(const std::string &key, const std::string &val)
    {
        std::unique_lock<std::shared_mutex> lock(m);
        data[key] = val;
    }
};
```

shared_mutex 适用场景：
- 读操作频繁
- 写操作少
- 读操作耗时较长
- 数据结构比较大




### 同一线程重复加锁

同一个线程可以重复加锁：

```cpp
std::recursive_mutex m;

void f()
{
    std::lock_guard<std::recursive_mutex> lock(m);
    g();
}

void g()
{
    std::lock_guard<std::recursive_mutex> lock(m);
}
```


recursive_mutex 使用场景：
- 维护旧代码，类中成员函数相互调用，而且各自加锁，短期内重构成本高，可以作为过度方案。


---


## 并发操作的同步

