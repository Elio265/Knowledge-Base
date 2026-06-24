---
tags: [c++, stack, queue, priority_queue, 容器适配器]
created: 2026-06-24
status: 已掌握
---

# stack、queue、priority_queue

## 一句话理解
stack、queue、priority_queue 都是**容器适配器**——它们封装底层容器，提供特定的数据访问接口（LIFO、FIFO、优先级堆），而非从头实现数据结构。

## 核心原理

容器适配器的设计模式：**封装一个现有容器（默认 deque 或 vector），只对外暴露符合语义的接口**，隐藏底层细节。

### stack（栈）——后进先出 LIFO

- 元素从栈顶（即底层容器尾部）压入和弹出。
- **默认底层容器**：`deque`（也可用 `vector`、`list`）。
- 底层容器需支持：`empty`、`back`、`push_back`、`pop_back`。

```cpp
stack<int> s;                 // 默认 deque
stack<int, vector<int>> sv;   // 使用 vector
stack<int, list<int>> sl;     // 使用 list

s.push(x);    // 入栈
s.pop();      // 出栈（不返回元素）
s.top();      // 返回栈顶元素引用
s.size();     // 元素个数
s.empty();    // 判空
```

### queue（队列）——先进先出 FIFO

- 队尾入队，队头出队。
- **默认底层容器**：`deque`（也可用 `list`，不能用 vector 因为没有 pop_front）。
- 底层容器需支持：`empty`、`size`、`front`、`back`、`push_back`、`pop_front`。

```cpp
queue<int> q;

q.push(x);    // 队尾入队
q.pop();      // 队头出队
q.front();    // 返回队头引用
q.back();     // 返回队尾引用
q.size();
q.empty();
```

### priority_queue（优先队列）——最大堆/最小堆

- 每次取出的元素都是当前优先级最高的（默认最大）。
- **默认底层容器**：`vector`（也可用 `deque`，必须支持随机访问迭代器）。
- 底层容器需支持随机访问迭代器，因为内部使用堆算法（`make_heap`、`push_heap`、`pop_heap`）。
- **默认是大堆**（最大堆），通过仿函数 `less<T>` 实现；小堆需指定 `greater<T>`。

```cpp
priority_queue<int> pq;                            // 大堆（默认）
priority_queue<int, vector<int>, greater<int>> minHeap;  // 小堆

pq.push(x);   // 插入元素
pq.pop();     // 弹出堆顶元素
pq.top();     // 返回堆顶元素引用（最大/最小）
pq.size();
pq.empty();
```

### 自定义比较

priority_queue 的模板参数是 `priority_queue<T, Container, Compare>`。

```cpp
// 小堆
priority_queue<int, vector<int>, greater<int>> minHeap;

// 自定义类型，重载 operator< 实现大堆
struct Task {
    int priority;
    bool operator<(const Task& t) const { return priority < t.priority; }
};
priority_queue<Task> tasks;
```

## 底层实现

### stack / queue 的底层：deque

- **deque（双端队列）** 是 stack 和 queue 的默认底层容器。
- deque 底层由多个连续缓冲区（buffer）组成，由中控器（map）管理，支持 O(1) 的头尾插入删除，且**不会因扩容导致迭代器全部失效**（只影响特定缓冲区）。
- 选择 deque 作为默认底层容器是因为：它兼顾 `push_back`、`pop_back` 和（对 queue）`pop_front` 的高效，比 vector 更灵活（vector 没有 `pop_front`）。

### priority_queue 的底层：堆（heap）

- priority_queue 本质上是一个**堆**（默认最大堆），使用 `vector` 存储数据。
- C++ 标准库通过以下算法维护堆结构：
  1. **`make_heap`**：建堆，将一段乱序数据调整为堆。
  2. **`push_heap`**：在 `push_back` 后调整，将新元素上浮到正确位置。
  3. **`pop_heap`**：将堆顶与堆尾交换，然后下沉调整，再 `pop_back`。
- **堆的物理存储**：用数组模拟完全二叉树——节点 i 的左子节点为 `2i+1`，右子节点为 `2i+2`，父节点为 `(i-1)/2`。

## 常见应用场景

| 适配器 | 应用场景 |
|--------|----------|
| **stack** | 函数调用栈、括号匹配、表达式求值（逆波兰式）、浏览器的前进后退、迷宫 DFS |
| **queue** | BFS 广度优先搜索、任务队列、消息队列、缓存（FIFO 淘汰） |
| **priority_queue** | 任务调度（按优先级）、Dijkstra 最短路径、哈夫曼编码、Top-K 问题、合并 K 个有序链表 |

## 容易踩坑的地方

- **stack / queue 不提供迭代器**：因为容器适配器的设计目的就是不允许随机遍历，只能按特定顺序访问。
- **`pop()` 不返回元素**：`s.pop()` 只是删除栈顶/队头元素，不返回它。必须先 `top()` / `front()` 再 `pop()`。
  
  ```cpp
  // 正确用法
  int val = s.top();
  s.pop();
  
  // 错误：没有 top() 直接 pop()
  s.pop();  // 不知道 pop 了什么
  ```

- **`top()` 前必须判空**：空容器调用 `top()` 是未定义行为。
  
  ```cpp
  if (!s.empty()) { int val = s.top(); }
  ```

- **priority_queue 不支持 `operator[]`**：不能像 vector 那样随机访问堆中元素。
- **priority_queue 的 `top()` 返回 const 引用**：不能通过 top 修改元素值，否则会破坏堆结构。
- **默认是大堆**：需要小堆时别忘记 `greater<T>`。
- **自定义类型的比较**：priority_queue 需要自定义比较时，比较严格弱排序必须满足（非自反、反对称、传递性）。
- **时间复杂度**：
  - stack/queue：所有操作 O(1)。
  - priority_queue：`push` O(log n)，`pop` O(log n)，`top` O(1)。

## 面试高频问题

1. **stack、queue 为什么被称为容器适配器？**
   - 它们不直接实现数据结构，而是封装现有容器（deque/vector/list），只暴露符合特定数据访问语义的接口。

2. **为什么 stack 和 queue 默认使用 deque 而不是 vector？**
   - deque 同时支持 `push_back` 和 `pop_front`（queue 需要），且头尾插入效率均为 O(1)。vector 没有 `pop_front`。

3. **priority_queue 的底层实现原理？**
   - 基于堆（默认最大堆），用 vector 存储，通过 `make_heap`、`push_heap`、`pop_heap` 维护堆结构。

4. **如何创建最小堆的 priority_queue？**
   - `priority_queue<int, vector<int>, greater<int>> minHeap;`

5. **priority_queue 排序的稳定性？**
   - priority_queue 基于堆排序，是不稳定的。如果两个元素优先级相同，它们的出队顺序不确定。

6. **如果让你实现一个支持动态更新优先级的 priority_queue，你会怎么做？**
   - STL 的 priority_queue 不支持。可以用 `multiset` 或手写堆 + 索引哈希表（如 Dijkstra 中的可更新堆）。

## 关联知识
- [[vector]]
- [[list]]
- [[deque]]
- [[红黑树]]
- [[堆与堆排序]]
