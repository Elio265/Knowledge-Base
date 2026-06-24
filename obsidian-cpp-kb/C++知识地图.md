---
tags: [MOC, c++]
created: 2026-06-24
---

# C++ 知识地图 (MOC)

> 从 CSDN 博客重构的个人 C++ 知识库，按知识领域组织，支持 Obsidian 双向链接。

---

## 🏗️ 核心语言

- [[内存管理]] — new/delete、RAII、智能指针、内存布局
- [[模板]] — 函数模板、类模板、模板特化、可变参数模板

## 🧬 面向对象

- [[类和对象]] — 构造/析构、this、static、friend、运算符重载
- [[继承]] — 单继承、多继承、虚继承、菱形问题
- [[多态]] — 虚函数、vtable、override、final、抽象类
- [[C++异常]] — try/catch/throw、异常安全、栈展开

## 📦 STL 容器

- [[string]] — std::string 实现、SSO、COW
- [[vector]] — 动态数组、扩容策略、迭代器失效
- [[list]] — 双向链表、splice、sort
- [[map-set]] — 有序关联容器、红黑树实现
- [[unordered_map-unordered_set]] — 哈希容器、bucket、负载因子
- [[stack-queue-priority_queue]] — 容器适配器、堆实现

## 🌲 数据结构

- [[二叉搜索树]] — BST 定义、增删查、复杂度分析
- [[红黑树]] — 红黑性质、旋转、插入/删除修正
- [[AVL树]] — 平衡因子、四种旋转
- [[布隆过滤器]] — 位图、哈希函数、误判率

## ⚡ C++11 新特性

- [[C++11新特性总览]] — auto、lambda、move、constexpr 等
- [[智能指针]] — unique_ptr、shared_ptr、weak_ptr

## 🎨 设计模式

- [[C++设计模式]] — 单例、工厂、观察者、策略、RAII

## 🌐 网络编程

- [[httplib库]] — HTTP 客户端/服务器、REST API

## 🔧 工程实践

- [[不定参数解析]] — va_list、可变参数模板
- [[JSONCPP库]] — JSON 解析与序列化

## 📝 其他

- [[C语言杂谈]] — C 语言零散知识点

---

## 📊 掌握程度总览

| 状态 | 数量 | 知识点 |
|------|------|--------|
| ✅ 已掌握 | 8 | 类和对象、继承、string、vector、list、map-set、stack-queue-priority_queue、二叉搜索树 |
| 📖 需要补充 | 12 | C++异常、AVL树、布隆过滤器、内存管理、模板、C++11新特性总览、智能指针、C++设计模式、httplib库、不定参数解析、JSONCPP库、C语言杂谈 |
| 🔬 深入学习 | 3 | 多态、红黑树、unordered_map-unordered_set |

---

## 🔗 外部资源

- 原始博客：[CSDN C++专栏](https://blog.csdn.net/wzh18907434168/category_12239750.html)
- 参考：[cppreference.com](https://en.cppreference.com/)
- 参考：[C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/)
