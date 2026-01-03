---
layout: post
title: static magic
date: 2024-09-17 17:39 +0800
---

`Magic statics` 或者说-函数内的静态变量初始化。就是在函数作用域内直接声明一个变量并初始化。
就像这样：
```c++
std::string& get_string() {
    static std::string magic_static("Hello");
    return magic_static;
}
```
在C++11之后，Magic statics的初始化保证为线程安全的。
