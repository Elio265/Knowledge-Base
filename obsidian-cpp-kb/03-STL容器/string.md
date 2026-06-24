---
tags: [cpp, string]
created: 2026-06-24
status: 已掌握
---

# string

## 一句话理解
string 是 C++ 标准库中用来管理动态字符串的类，封装了 C 风格字符串的操作，自动管理内存，是 `basic_string<char>` 的类型别名。

## 核心原理

- string 底层是 `basic_string<char>` 的模板特化：`typedef basic_string<char> string`。
- 使用连续的内存空间存储字符，支持随机访问（`operator[]`）。
- 自动管理内存分配与释放，避免 C 字符串手动管理 `\0` 结尾和缓冲区溢出的问题。
- 接口设计与 STL 容器保持一致（如 `size()`、`begin()`、`end()`），同时增加字符串专用操作（如 `find()`、`substr()`、`c_str()`）。
- **深拷贝**：拷贝构造和赋值运算符进行深拷贝，每个对象拥有独立的资源，避免多个对象共享同一块内存导致的析构重复释放问题。

### 常用构造方式

```cpp
string s1;                  // 空字符串
string s2("hello");         // 从 C 字符串构造
string s3(10, 'c');         // 10 个 'c'
string s4(s2);              // 拷贝构造
```

### 常用操作

| 分类 | 函数 | 说明 |
|------|------|------|
| 容量 | `size() / length()` | 返回有效字符个数（二者等价） |
| 容量 | `capacity()` | 返回当前底层空间总大小 |
| 容量 | `reserve(n)` | 预留空间，不改变 size |
| 容量 | `resize(n, c)` | 改变有效字符个数，可初始化新空间 |
| 容量 | `empty()` | 检测是否为空 |
| 容量 | `clear()` | 清空有效字符，不改 capacity |
| 修改 | `push_back(c)` | 尾插字符 |
| 修改 | `append(str)` | 追加字符串 |
| 修改 | `operator+=` | 追加字符串/字符 |
| 修改 | `insert(pos, str)` | 插入 |
| 修改 | `erase(pos, len)` | 删除 |
| 访问 | `operator[]` | 随机访问字符 |
| 访问 | `c_str()` | 返回 C 风格 `const char*` |
| 查找 | `find(c, pos)` | 从 pos 开始正向查找 |
| 查找 | `rfind(c, pos)` | 从 pos 开始反向查找 |
| 查找 | `substr(pos, len)` | 截取子串 |
| 非成员 | `operator+` | 拼接字符串（传值返回，深拷贝开销大，慎用） |
| 非成员 | `operator>> / <<` | 输入输出流操作 |
| 非成员 | `operator< <= > >= == !=` | 比较操作 |

## 底层实现

- **连续动态数组**：string 底层通过 `char*` 指针管理堆上的连续内存，类似 vector<char>。
- **SSO（Small String Optimization）**：现代实现（如 GCC 的 `std::string`）对于短字符串（通常 15 字节以内）直接存储在栈上的固定大小数组中，避免堆分配。这是最常见的优化手段。
- **COW（Copy-on-Write）**：旧版实现（如 GCC 5 之前的 `std::string`）使用写时复制技术——多个 string 共享同一块内存，仅在写入时才真正拷贝。C++11 之后因线程安全等问题已被废弃。
- **扩容策略**：类似于 vector，string 在空间不足时会重新申请更大的内存（通常 1.5~2 倍增长），拷贝原数据后释放旧空间。
- **`c_str()` 保证**：返回的 `const char*` 以 `\0` 结尾，保证与 C 字符串兼容。

## 常见应用场景

1. **替代 C 字符串**：所有需要处理文本的地方，避免手动 `malloc`/`free` 和缓冲区溢出。
2. **文本解析**：配合 `find()`、`substr()` 解析配置文件、URL、日志等。
3. **字符串拼接**：使用 `+=` 或 `append`，需要大量拼接时建议先 `reserve` 预分配空间。
4. **与 C 代码交互**：使用 `c_str()` 将 string 传递给 C API。
5. **文件读写**：配合 `ifstream` / `ofstream` 读写文本文件。

## 容易踩坑的地方

- **`size()` 与 `length()` 等价**：二者底层实现完全一样，`size()` 是为与其它容器接口保持一致。
- **`clear()` 不释放内存**：只将有效字符清空为 0，`capacity()` 不变。想释放空间需用 `shrink_to_fit()`（C++11）。
- **`reserve()` 不会缩小空间**：当参数小于当前 capacity 时，`reserve()` 不一定会缩小。
- **`resize(n)` 会修改 `size()`**：若 n 大于当前 size，会填充 `\0`；若 n 小于当前 size，会截断。
- **`operator[]` 不检查越界**：越界访问是未定义行为；使用 `at()` 会抛 `out_of_range` 异常。
- **`c_str()` 返回值可能失效**：在调用非常量成员函数（如 `append`、`operator[]` 非 const 版本）后，之前获取的 `c_str()` 指针可能被释放。
- **`operator+` 传值返回**：每次调用都会产生深拷贝，大量拼接时性能差，应使用 `+=` 或 `ostringstream`。
- **迭代器失效**：任何引起底层空间重新分配的操作（`reserve`、`resize`、`insert`、`push_back`、`append` 等）都会使已有迭代器失效。
- **`find` 返回值是 `size_t`**：查找失败返回 `npos`（即 `static_cast<size_t>(-1)`），不要直接与 `-1` 比较。

## 面试高频问题

1. **讲一下 string 的 SSO 和 COW 的区别？为什么 COW 被废弃了？**
   - SSO 对小字符串使用栈上数组，COW 通过引用计数共享内存。COW 在 C++11 的多线程环境下加锁开销大，且赋值操作不一定是真正"写"，因此被弃用。
2. **string 的 `operator[]` 和 `at()` 有什么区别？**
   - `operator[]` 不检查越界，`at()` 越界时抛 `out_of_range` 异常。
3. **string 如何高效地拼接大量字符串？**
   - 先用 `reserve()` 预分配足够空间，再使用 `+=` 或 `append()`，避免频繁扩容。
4. **深拷贝与浅拷贝在 string 中是如何体现的？**
   - 浅拷贝使多个对象共享同一块 `char*`，析构时重复释放导致崩溃；深拷贝为每个对象分配独立内存，通过自定义拷贝构造函数和赋值运算符实现。
5. **`c_str()` 返回的指针什么时候会失效？**
   - 任何可能引起 string 内部重新分配的非 const 操作之后都会失效。

## 关联知识
- [[vector]]
- [[C++内存管理]]
- [[迭代器]]
