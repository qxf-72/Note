# 前提

编译时生成调试信息，并且不优化：
```bash
g++ main.cpp -g -O0 -o main
```

用 CMake 构建 C++ 项目，最常用的方式是：

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build
```

---

<br/>


<br/>


<br/>


# 执行

- 设置断点：`b`
	- 查看所有断点 `i b`
	- 删除断点：`delete`
	- 禁止断点：`disable`
	- 启用断点：`enable`
	- 条件断点：`break 文件：行号 if 条件 `
- 下一行：`n`
- 进入函数 ：`s`
- 继续运行：`c`


---

<br/>


<br/>


<br/>

# 查看

- 查看变量：`p`
- 查看源码：`list`
- 查看调用栈：`bt`
- 切换栈帧：`frame`


---