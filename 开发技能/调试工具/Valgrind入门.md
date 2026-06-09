# 前提

使用 debugg 模式编译

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build
```

---

<br/>


<br/>


<br/>

# 常用指令

```bash
valgrind --tool=memcheck --leak-check=full --show-leak-kinds=all --track-origins=yes ./your_program
```

```text
--tool=memcheck  
使用内存检查工具，这是 Valgrind 最常用的工具。  
  
--leak-check=full  
详细检查内存泄漏。  
  
--show-leak-kinds=all  
显示所有类型的泄漏。  
  
--track-origins=yes  
追踪未初始化变量从哪里来的，虽然会更慢，但非常有用。
```

---

<br/>


<br/>


<br/>

# 输出

## definitely lost

确定内存泄漏。必须进行修复。

---
## indirectly lost

间接泄漏：泄漏了一个对象，而这个对象里面又管理了其他内存。

---

## possibly lost

可能泄漏。可以结合上下文判断。

---
## still reachable

程序结束时还有内存能被访问到。不一定是严重错误，比如某些全局对象、单例、标准库内部缓存，程序退出时没有主动释放，但操作系统会回收。


---

## Invalid read

非法读取。常见原因：

```text
数组越界读取  
访问已经释放的内存  
访问非法地址
```

---

## Invalid write

非法写入，例如数组越界写入。

---

## Use of uninitialised value

使用了未初始化变量。

---

