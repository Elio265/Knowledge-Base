---
tags: [cpp, list, 顺序容器]
created: 2026-06-24
updated: 2026-06-25
status: 已掌握
---

# list

## 一句话理解
list 是 C++ STL 中的双向链表容器，支持在任意位置进行 O(1) 的插入和删除，但不支持随机访问。

## 核心原理

- 元素按照链表顺序存储，每个节点包含数据和指向前后节点的指针。
- 插入和删除操作只需调整指针指向，不影响其他元素的位置和迭代器有效性（被删除节点的迭代器除外）。
- 不支持随机访问，只能通过遍历来访问元素。
- 与 vector 不同，list **不需要连续内存**，因此插入元素不会导致整体搬迁。
- 迭代器是**双向迭代器**（bidirectional iterator），支持 `++it` 和 `--it`，但不支持 `it + n`。

### 构造方式

```cpp
list<int> ls;                    // 空链表
list<int> ls(10, 6);             // 10 个 6
list<int> ls(ls2);               // 拷贝构造
list<int> ls(begin, end);        // 迭代器范围构造
```

### 常用接口

| 分类 | 函数 | 说明 |
|------|------|------|
| 元素访问 | `front()` | 首元素引用 |
| 元素访问 | `back()` | 尾元素引用 |
| 容量 | `size()` | 元素个数 |
| 容量 | `empty()` | 判空 |
| 容量 | `max_size()` | 系统允许的最大元素数 |
| 修改 | `push_back(val)` | 尾部插入 |
| 修改 | `push_front(val)` | 头部插入 |
| 修改 | `pop_back()` | 尾部删除 |
| 修改 | `pop_front()` | 头部删除 |
| 修改 | `insert(pos, val)` | 在 pos 位置插入 |
| 修改 | `erase(pos)` | 删除 pos 位置元素 |
| 修改 | `clear()` | 清空所有元素 |
| 修改 | `swap(ls)` | 交换两个链表内容 |

### list 专属操作

| 函数 | 说明 |
|------|------|
| `splice(pos, ls)` | 将另一个 list 的元素转移到当前 list 的 pos 位置，O(1) 完成 |
| `remove(val)` | 删除所有等于 val 的元素 |
| `unique()` | 删除连续重复元素（只保留第一个） |
| `sort()` | 排序（list 不能使用 std::sort，因为需要随机访问迭代器） |
| `reverse()` | 反转链表 |
| `merge(ls)` | 合并两个有序链表 |
| `l.splice(it, l2)` | 将 l2 的元素插入到 l 的 it 位置前，l2 变为空 |

### sort 注意事项
list 提供了自己的 `sort()` 成员函数，因为 `std::sort` 要求随机访问迭代器。list 的 sort 使用归并排序，时间复杂度 O(n log n)。

```cpp
list<int> ls = {5, 3, 1, 4, 2};
ls.sort();                    // 升序
ls.sort(greater<int>());      // 降序
```

## 底层实现

- **双向链表结构**：每个节点包含三个字段——`_prev`（前驱指针）、`_next`（后继指针）、`_data`（存储的数据）。
  ```cpp
  struct ListNode {
      ListNode* _prev;
      ListNode* _next;
      T _data;
  };
  ```
- **哨兵节点（头节点）**：list 内部通常有一个不存储数据的哨兵节点（`_head`），使链表操作统一，避免边界判断。
- **迭代器封装**：list 的迭代器是对节点指针的封装，`++` 操作本质是 `node = node->_next`，`--` 操作本质是 `node = node->_prev`。
- **无 capacity 概念**：list 动态分配节点内存，没有 capacity / reserve 的概念。
- 与 vector 不同，list 没有 `operator[]`，没有随机访问能力。

## 常见应用场景

1. **频繁在中间插入/删除**：如任务调度队列、LRU 缓存的双向链表结构。
2. **大对象存储**：避免 vector 扩容时的大规模拷贝。
3. **需要常数时间头部插入**：vector 头部插入 O(n)，list 头部插入 O(1)。
4. **用 splice 实现 O(1) 区间转移**：将一个链表的节点转移到另一个链表。

## 容易踩坑的地方

- **迭代器失效**
  - **`erase()` 会使被删除节点的迭代器失效**，但其他节点的迭代器保持有效（这与 vector 不同）。
  - **不要在遍历时直接 erase**：删除后迭代器无法自增，需先保存下一个节点的迭代器。
  
  ```cpp
  // 错误：erase 后 it 已失效
  for (auto it = ls.begin(); it != ls.end(); ++it) {
      if (*it == val) ls.erase(it);
  }
  
  // 正确方式 1：利用 erase 返回值
  for (auto it = ls.begin(); it != ls.end(); ) {
      if (*it == val) it = ls.erase(it);
      else ++it;
  }
  
  // 正确方式 2：先保存下一个迭代器
  for (auto it = ls.begin(); it != ls.end(); ) {
      if (*it == val) {
          auto tmp = it++;
          ls.erase(tmp);
      } else {
          ++it;
      }
  }
  ```

- **不要将外部迭代器长期保存**：对 list 进行插入/删除后，保存在外部的迭代器可能失效。
- **没有随机访问**：不能使用 `it + n` 或 `it[n]`，访问第 n 个元素需要遍历 O(n)。
- **无法使用 `std::sort`**：必须使用 list 自己的 `sort()` 成员函数。
- **`splice` 不复制元素**：只是调整指针指向，原链表不再包含这些元素。

## 面试高频问题

1. **list 和 vector 的核心区别？**
   - 存储结构：list 非连续内存（链表），vector 连续内存（动态数组）。
   - 随机访问：list 不支持 O(n)，vector 支持 O(1)。
   - 插入删除：list 任意位置 O(1)（已知迭代器），vector 非尾部 O(n)。
   - 迭代器：list 双向迭代器，vector 随机访问迭代器。
   - 迭代器失效：list 只使被删除节点失效，vector 扩容后全部失效。

2. **list 的迭代器为什么不支持 `it + n`？**
   - 因为 list 是链表，内存不连续，无法通过指针偏移直接跳到第 n 个元素。

3. **list 有哪些区别于其他容器的操作？**
   - `splice`（O(1) 转移节点）、`unique`（去重）、`merge`（合并有序链表）、`reverse`（反转）、`sort`（归并排序）。

4. **为什么 list 的 `sort()` 和 `std::sort` 不同？**
   - `std::sort` 需要随机访问迭代器，list 的迭代器是双向的，因此实现不了。list 内部使用归并排序。

5. **`splice` 有什么特殊之处？**
   - splice 只是调整指针，不分配或拷贝元素，时间复杂度 O(1)，这是 vector 做不到的。

### 时间复杂度表

| 操作 | 时间复杂度 |
|------|-----------|
| 头部/尾部插入删除 | O(1) |
| 中间插入删除（已知迭代器） | O(1) |
| 随机访问 | 不支持 |
| 查找 | O(n) |
| sort | O(n log n) |

## 关联知识
- [[vector]]
- [[stack-queue-priority_queue]]
