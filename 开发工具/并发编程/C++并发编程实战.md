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

**需要显式调用 join 或者 detac**h，否则，thread 析构函数会调用 terminate 终止程序，即便有异常。

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

- <div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-05-26_17-01-24.png" loading="lazy" style="max-width: 100%; height: auto; width: 500px;"/> </div>

- <div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-05-26_17-02-48.png" loading="lazy" style="max-width: 100%; height: auto; width: 700px;"/> </div>

- <div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-05-26_17-04-48.png" loading="lazy" style="max-width: 100%; height: auto; width: 700px;"/> </div>


std::thread 创建线程时，会把参数保存起来再在线程中调用函数。  
  
- 默认是拷贝。  
- 要引用，用 std::ref。  
- 要移动，用 std::move。  
- 要调成员函数，传 &Class::func 和对象地址。  
- 只要传引用、指针、this，就必须保证对象活得比线程久。


---