## 防卫式声明

防止**同一个头文件** 在 **同一个编译单元**中被重复包含。

```cpp
#ifndef __COMPLEX__
#define __COMPLEX__

...

#endif
```



---


<br/>


<br/>


<br/>



## 构造函数

C++ 编译器也会自动生成一些“特殊成员函数”，其中就包括**拷贝构造函数**和**拷贝赋值运算符**。

```cpp
complex a(1, 2);
complex b（a）; //拷贝构造函数
b = a;     //拷贝赋值运算符
```

`=` 是可以重载的，但只能重载为**成员函数**。

```cpp
class complex {
private:
    double real;
    double imag;

public:
    complex(double i = 0, double j = 0)
        : real(i), imag(j) {}

    complex& operator=(const complex& other) {
        if (this != &other) {
            real = other.real;
            imag = other.imag;
        }
        return *this;
    }
};
```

**对于函数， `const` 在可以添加时，必须添加**。加了就可以操作 const 对象。

<div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-05-22_15-51-53.png" loading="lazy" style="max-width: 100%; height: auto; width: 500px;"/> </div>


---

<br/>


<br/>


<br/>



## 友元

**同一个 class 的多个实例之间相互为友元**。


---

<br/>


<br/>


<br/>


## 按引用返回

**需要考虑按值还是按引用返回**。对于大对象，且没有修改操作，使用按引用返回效率更好。



发送者无需知道接收者是以引用形式接受的


<div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-05-22_15-08-41.png" loading="lazy" style="max-width: 100%; height: auto; width: 400px;"/> </div>


内联 inline 和内部链接的区别

<div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-05-22_15-21-41.png" loading="lazy" style="max-width: 100%; height: auto; width: 700px;"/> </div>



---


<br/>


<br/>


<br/>


## 拷贝构造函数、拷贝赋值运算符重载、析构函数


涉及指针内存操作的，**一定要写**拷贝构造函数、拷贝赋值运算符重载、析构函数。

<div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-05-22_16-14-50.png" loading="lazy" style="max-width: 100%; height: auto; width: 600px;"/> </div>

拷贝赋值比拷贝构造，多了 **释放内存** 和 **检测自我赋值** 操作。

---


<br/>


<br/>


<br/>


## 堆、栈和内存管理

<div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-05-22_16-28-14.png" loading="lazy" style="max-width: 100%; height: auto; width: 500px;"/> </div>

array new 一定要搭配 array delete。


----



<br/>


<br/>


<br/>

## 类模板、函数模板


**static 成员一定要在类外进行定义**。

**而静态函数没有 this 指针**，只能处理 static 数据成员。

<div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-05-23_08-10-50.png" loading="lazy" style="max-width: 100%; height: auto; width: 500px;"/> </div>

Sigleton 设计模式

<div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-05-23_08-15-59.png" loading="lazy" style="max-width: 100%; height: auto; width: 500px;"/> </div>


---

<br/>


<br/>


<br/>


## 组合与继承


**复合**

<div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-05-23_08-39-38.png" loading="lazy" style="max-width: 100%; height: auto; width: 500px;"/> </div>


**委托**

<div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-05-23_08-40-38.png" loading="lazy" style="max-width: 100%; height: auto; width: 600px;"/> </div>

**继承**

<div style="text-align: center;"><img src="https://picture-in-md.oss-cn-guangzhou.aliyuncs.com/2026-05-23_09-17-22.png" loading="lazy" style="max-width: 100%; height: auto; width: 600px;"/> </div>



---
