# 线程管控

## 基本线程管理

**启动线程：构建 thread 对象**
- 函数符号
- 函数指针（仿函数）
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
    thread t1((background_task())); //函数指针，加多一对括号，或者改为大括号，来避免被编译器识别为函数声明
    hread t2(do_something); //函数符
    thread t3([](){   // lambda表达式
        do_something();
    }); 
    
    t1.join();

    return 0;
}
```

**需要显式调用 join 或者 detach**，否则，thread 析构函数会调用 terminate 终止程序，即便有异常。

> terminate called without an active exception


**RAII**

分离线程直接在启动之后调用 detach 即可。但是 join 不可以直接调用，否则就变成了同步函数了。为了确保能够显式调用 join，使用 RAII
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

<div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-05-26_17-01-24.png" loading="lazy" style="max-width: 100%; height: auto; width: 500px;"/> </div>

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


# 线程间共享数据
## 线程间共享数据的问题

**竞态条件**

程序执行的结果依赖于多个线程的执行顺序，并且这种顺序不受程序控制，如果某些执行顺序会导致错误结果、数据损坏或未定义行为，这就是有害静态条件。

**data race**

多个线程同时访问同一内存位置，其中至少一个是写操作，并且没有使用 mutex、atomic 等同步手段。

**为什么 `if (!s.empty()) s.pop();` 在多线程中不一定安全？**

因为 empty 和 pop 是两个独立的操作，线程 a 在判断 empty 之后，切换到线程 b 执行 pop 操作，如果在切换回线程 a 执行 pop 操作就会出错。所以在并发环境下，多个相关操作需要被作为同一个整体进行保护。

**避免有害竞态条件的方法：**
- 不同享数据
- 同享数据，但是用锁保护
- 使用 atomic 类型


---
## 用 mutex 保护共享数据

多个线程访问同一份数据时，可能发生数据竞争。mutex 用来限制同一时间只有一个线程进入临界区。

- mutex 保护的是共享数据；
- lock_guard 用 RAII 管理加锁和解锁；
- 所有访问共享数据的路径都要加同一把锁；
- 不要返回受保护数据的引用或指针；
- 接口设计也会产生竞态条件；
- 多把锁要注意死锁；
- 不要持锁做耗时操作。


```cpp
std::mutex mtx;

void func()
{
    std::lock_guard<std::mutex> lock(mtx);
    // 访问共享数据
}
```


---