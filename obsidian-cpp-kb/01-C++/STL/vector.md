---
tags: [cpp, vector, 顺序容器]
created: 2026-06-24
updated: 2026-06-25
status: 已掌握
---

# vector

## 一句话理解
vector 是一个动态数组容器，在连续内存上存储元素，支持随机访问和自动扩容，是 C++ 中最常用的序列容器。

## 核心原理

- vector 维护一段连续的线性内存空间，可以通过下标在 O(1) 时间内随机访问任意元素。
- 大小可动态改变，由容器自动管理内存分配与释放。
- 为减少频繁扩容，vector 会预先分配额外空间（`capacity >= size`）。
- 扩容策略因实现而异：**VS 按 1.5 倍增长，GCC 按 2 倍增长**。

### 核心成员

```cpp
vector<T> v;              // 空 vector
vector<T> v(5, 3);        // 5 个 3
vector<T> v(v2);          // 拷贝构造
vector<T> v(begin, end);  // 迭代器范围构造

v.size();      // 有效元素个数
v.capacity();  // 当前容量（已分配空间能容纳的元素数）
v.empty();     // 是否为空
v.reserve(n);  // 预留容量（不改变 size）
v.resize(n);   // 改变有效元素个数
```

### 常见接口

| 分类 | 函数 | 说明 |
|------|------|------|
| 修改 | `push_back(val)` | 尾部插入元素 |
| 修改 | `pop_back()` | 尾部删除元素 |
| 修改 | `insert(pos, val)` | 在 pos 处插入 |
| 修改 | `erase(pos)` | 删除 pos 处元素 |
| 修改 | `swap(v)` | 交换两个 vector |
| 修改 | `clear()` | 清空所有元素 |
| 访问 | `operator[]` | 下标随机访问 |
| 访问 | `front()` / `back()` | 首/尾元素引用 |
| 算法 | `find(begin, end, val)` | 非成员函数，查找元素 |

### emplace_back vs push_back

- `push_back(val)`：接受已构造好的对象，发生拷贝或移动构造。
- `emplace_back(args...)`：**直接传入构造参数**，在容器内就地构造对象，避免临时对象的拷贝/移动，性能更优。

```cpp
struct Person { Person(string n, int a); };
vector<Person> v;
v.push_back(Person("Alice", 25));    // 先构造临时对象，再拷贝
v.emplace_back("Alice", 25);         // 直接就地构造，无临时对象
```

## 底层实现

- **连续动态数组**：底层为 `T*` 指针，指向堆上连续内存。
- **三个指针管理**：
  - `_start`：指向数据起始位置
  - `_finish`：指向已存储元素的下一个位置
  - `_end_of_storage`：指向已分配空间末尾
- **扩容流程**：当 `_finish == _end_of_storage` 时，申请新的更大内存 -> 拷贝/移动旧元素 -> 释放旧内存 -> 更新指针。
- **扩容复杂度**：均摊 O(1)。每次扩容代价 O(n)，但扩容次数为 O(log n)，均摊到每次插入为常数。

### 迭代器本质
vector 的迭代器就是原生指针 `T*` 的封装，因此支持 `it + n`、`it - n` 等随机访问操作。

## 常见应用场景

1. **代替静态数组**：需要随机访问且大小变化的场景，如存储一组数据。
2. **作为底层容器**：`stack`、`queue`、`priority_queue` 的默认或可选底层容器。
3. **算法配合**：与 `<algorithm>` 配合进行排序、查找、二分等操作。
4. **存储大量元素**：连续内存缓存友好，遍历性能优于 list 和 map。

## 容易踩坑的地方

- **迭代器失效（最重要）**
  - **扩容导致失效**：`resize`、`reserve`、`insert`、`push_back`、`assign` 等引起底层内存重新分配的操作，会使所有迭代器、指针、引用全部失效。
  - **`erase` 导致失效**：删除位置之后的迭代器全部失效。VS 认为删除任意位置的迭代器都应视为失效。
  - **正确做法**：操作后重新获取迭代器。
  ```cpp
  // 错误示例
  vector<int>::iterator it = v.begin();
  v.push_back(100);        // 可能扩容
  cout << *it;             // it 可能已失效
  
  // 正确用法：erase 后使用返回值
  auto it = v.begin();
  it = v.erase(it);        // erase 返回下一个有效迭代器
  ```

- **`reserve` vs `resize` 混淆**
  - `reserve(n)`：只扩容，不改变 `size`，不初始化元素。
  - `resize(n)`：改变 `size`，可能扩容并初始化新元素。
- **`reserve` 不会缩小**：当参数小于当前 capacity 时，不一定生效。
- **`insert` / `erase` 效率差**：非尾部位置插入/删除需要移动大量元素，复杂度 O(n)。
- **越界访问**：`operator[]` 不检查越界，越界是未定义行为（不会抛异常）。使用 `v.at(i)` 会在越界时抛 `out_of_range`。
- **扩容后迭代器失效常考**：在 `for` 循环中向 v 插入元素，遍历可能访问到失效迭代器。

### 时间复杂度表

| 操作 | 时间复杂度 |
|------|-----------|
| 随机访问 `operator[]` | O(1) |
| 尾部插入/删除 | 均摊 O(1) |
| 中间插入/删除 | O(n) |
| 查找（线性） | O(n) |

## 面试高频问题

1. **vector 的扩容机制是什么？为什么是 1.5 倍或 2 倍？**
   - 固定增长倍数保证均摊 O(1) 尾部插入。小于 1.5 倍会导致扩容过于频繁，大于 2 倍可能浪费太多内存。
2. **`reserve(n)` 和 `resize(n)` 的区别？**
   - reserve 只开空间不改 size；resize 开空间 + 初始化 + 改 size。
3. **哪些操作会导致 vector 迭代器失效？如何避免？**
   - 扩容操作（push_back、insert、reserve、resize）和 erase。避免方法：操作后重新获取迭代器，或使用下标。
4. **`emplace_back` 和 `push_back` 有什么区别？什么时候用哪个？**
   - emplace_back 就地构造，避免临时对象拷贝。对自定义类型参数多时优先用 emplace_back。
5. **vector 的 `at()` 和 `operator[]` 的区别？**
   - `at()` 越界抛异常，`operator[]` 越界未定义行为（不检查）。

## 关联知识
- [[string]]
- [[list]]
- [[迭代器]]
- [[stack-queue-priority_queue]]
- [[内存管理]]
