
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