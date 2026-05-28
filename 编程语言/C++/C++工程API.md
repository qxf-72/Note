# 文件操作

**打开目录**

```cpp
DIR* dir = opendir(path.c_str());
if (dir == nullptr) {
    std::cerr << "Cannot open directory: " << path << std::endl;
    return 0;
}

// ...

closedir(dir);
```

**读取目录信息**

```cpp
dirent* entry = nullptr;
// 在目录中一个一个的读取文件名
while ((entry = readdir(dir)) != nullptr) {
	// ...
}
```


**判断文件类型**

```cpp
struct stat st;	//Linux 系统提供的一个结构体，用来保存文件信息

// Linux 提供的函数，用来获取某个路径的文件信息
// 第一个参数：文件路径  
// 第二个参数：stat 结构体的地址
lstat(fullPath.c_str(), &st) ;
	
}
// 是目录类型
if (S_ISDIR(st.st_mode)) {
            
}
// 是普通文件类型
else if (S_ISREG(st.st_mode)) {
    
}
```

---