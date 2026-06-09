
# CMake 基础

- 指定 CMake 的最低版本要求：
	- `cmake_minimum_required(VERSION 3.10)`
- 定义项目的名称和使用的编程语言：
	- `project(<project_name> [<language>...])`
- 指定要生成的可执行文件和其源文件：
	- `add_executable(<target> <source_files>...)`
- 添加头文件搜索路径：
	- `target_include_directories(my_target PRIVATE include)`
- 设置变量的值：
	- `target_compile_features(my_target PRIVATE cxx_std_11)`
- 链接目标文件与其他库：
	- `target_link_libraries(<target> <libraries>...)`

## 单文件项目

```cmake
cmake_minimum_required(VERSION 3.10)

project(cpp_demo LANGUAGES CXX)

add_executable(cpp_demo main.cpp)

target_compile_features(cpp_demo PRIVATE cxx_std_11)
```

---

## 多文件项目

```cmake
cmake_minimum_required(VERSION 3.10)

project(cpp_demo LANGUAGES CXX)

add_executable(cpp_demo
    src/main.cpp
    src/Logger.cpp
)

target_include_directories(cpp_demo PRIVATE
    ${PROJECT_SOURCE_DIR}/include
)

target_compile_features(cpp_demo PRIVATE cxx_std_11)
```

---