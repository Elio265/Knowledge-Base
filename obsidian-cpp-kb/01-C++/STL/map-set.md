---
tags: [cpp, map, set, 关联容器]
created: 2026-06-24
updated: 2026-06-25
status: 已掌握
---

# map 和 set

## 一句话理解
map 和 set 是 C++ 基于红黑树实现的有序关联容器，map 存储键值对、set 存储唯一键，均按严格弱排序自动排序，支持 O(log n) 的插入、删除和查找。

## 核心原理

### set——集合

- 存储唯一的元素（value 即 key），元素不能在容器中直接修改（否则会破坏排序）。
- 默认按 `<` 升序排序（`less<T>`）。
- 底层使用**红黑树**，保证插入、删除、查找均为 O(log n)。

```cpp
set<int> s;                           // 升序
set<int, greater<int>> s_desc;        // 降序
set<int> s = {5, 3, 8, 1, 3};        // {1, 3, 5, 8}，重复 3 被忽略

s.insert(val);
s.erase(val);
s.find(val);          // 返回迭代器，未找到返回 end()
s.count(val);         // 返回 0 或 1（因为唯一）
s.empty();
s.size();
s.clear();
```

### map——映射

- 存储**键值对**（`pair<const Key, T>`），每个键唯一，按键升序排列。
- 支持 `operator[]`：通过键访问值，**键不存在时会自动插入一个默认值**。
- 底层使用**红黑树**，所有操作 O(log n)。

```cpp
map<string, int> m;
m["apple"] = 5;                       // 插入键值对
m.insert(make_pair("banana", 3));
m.insert(pair<string,int>("cat",7));

m.find(key);                          // 返回迭代器
m.count(key);                         // 返回 0 或 1
m.erase(key);                         // 删除

// 遍历（有序输出）
for (auto& [k, v] : m) {
    cout << k << ": " << v << endl;
}
```

### multiset / multimap

- multiset：允许重复元素，有序。
- multimap：允许重复键，有序。**不支持 `operator[]`**（因为一个键可能对应多个值）。

### 自定义比较

```cpp
// 根据字符串长度排序的 set
struct CompareLength {
    bool operator()(const string& a, const string& b) const {
        return a.size() < b.size();
    }
};
set<string, CompareLength> s;
```

## 底层实现——红黑树

- **红黑树**是一种**近似平衡**的二叉搜索树，保证最长路径不超过最短路径的 2 倍。
- 性质：
  1. 每个节点是红色或黑色。
  2. 根节点是黑色。
  3. 叶子节点（NIL）是黑色。
  4. 红色节点的子节点必须是黑色（不能有连续红色节点）。
  5. 从任意节点到其所有叶子节点的路径上，黑色节点数目相同。
- **插入和删除后通过旋转和变色恢复平衡**：
  - 左旋 / 右旋：调整节点位置。
  - 变色：改变节点颜色以满足红黑树性质。
- 与 AVL 树的对比：红黑树要求更宽松（不要求绝对平衡），因此插入/删除的旋转次数更少（O(1) 次旋转），适合频繁插入删除的场景。

### 为什么选择红黑树而不是哈希表或 AVL 树？

| 对比项 | 红黑树 (map/set) | 哈希表 (unordered_map/set) |
|--------|------------------|----------------------------|
| 时间复杂度 | O(log n) | 平均 O(1)，最坏 O(n) |
| 元素有序 | 自动排序 | 无序 |
| 范围查询 | 支持（lower_bound/upper_bound） | 不支持 |
| 内存占用 | 更少（节点有颜色位但无需哈希表桶） | 更多（桶+链表） |

红黑树 vs AVL 树：AVL 更严格平衡（查询更快），但红黑树插入删除旋转更少。C++ 标准库选择红黑树是**读写均衡**的取舍。

## 常见应用场景

1. **字典 / 词频统计**：map 存储单词到次数的映射。
2. **去重并排序**：set 自动去重并排序。
3. **范围查找**：使用 `lower_bound` / `upper_bound` 在有序数据中查询区间。
4. **配置项管理**：键值对存储配置。
5. **图论邻接表**：`map<int, set<int>>` 表示图的邻接关系。

```cpp
// 范围查找示例
set<int> s = {1, 3, 5, 7, 9, 11};
auto low = s.lower_bound(4);   // 指向第一个 >= 4 的元素，即 5
auto up = s.upper_bound(8);    // 指向第一个 > 8 的元素，即 9
// 遍历 [5, 9)  -> 5, 7
for (auto it = low; it != up; ++it) cout << *it << " ";
```

## 容易踩坑的地方

- **set 的元素不可修改**：因为修改 = 改变排序顺序 = 破坏红黑树结构。正确做法是先 erase 再 insert 新值。
- **map 的 `operator[]` 会自动插入**：这是最常见的坑。查询不存在键时，会创建默认值插入。
  
  ```cpp
  map<int, string> m;
  if (m[100] == "abc") { }       // ！此时 m 中已经有 {100, ""}
  // 正确做法：用 find
  if (auto it = m.find(100); it != m.end() && it->second == "abc") { }
  ```

- **map 的迭代器指向的是 `pair<const Key, T>`**：`it->first` 为 const，不能修改；`it->second` 可修改。
- **`multimap` 不能使用 `[]`**：因为有多个相同键，无法确定访问哪个。
- **`count()` 在 map/set 中只返回 0/1**：只在 multiset/multimap 中才可能 >1。
- **自定义比较必须满足严格弱排序**：
  - `!comp(a, b) && !comp(b, a)` 意味着 a == b。
  - 不能只定义 `operator<` 而没有 `operator>`。
  - 比较函数必须一致，否则会导致未定义行为。
- **迭代器失效**：map/set 的插入和删除**不会使其他迭代器失效**（只有被删除元素的迭代器失效），这与 vector 不同。
- **红黑树不擅长随机访问**：不能通过下标（位置）访问元素，只能通过键。

### 时间复杂度表

| 操作 | 时间复杂度 |
|------|-----------|
| 插入 | O(log n) |
| 删除 | O(log n) |
| 查找 | O(log n) |
| `lower_bound` / `upper_bound` | O(log n) |
| 遍历（迭代器递增） | O(n) 均摊 O(1) |

## 面试高频问题

1. **map 的底层为什么用红黑树而不是哈希表或 AVL 树？**
   - 红黑树保证 O(log n) 操作 + 有序性 + 范围查询能力；AVL 树插入删除旋转太多；哈希表无序且最坏 O(n)。

2. **map 的 `operator[]` 与 `at()` 有什么区别？**
   - `operator[]` 键不存在时会自动插入默认值，`at()` 键不存在时抛 `out_of_range` 异常。

3. **map 和 set 的迭代器为什么不会因为插入而失效（与 vector 不同）？**
   - 红黑树节点独立分配（new），插入不涉及内存搬迁。只要被删除的节点的迭代器，其他全部有效。

4. **红黑树的特性是什么？为什么它是近似平衡的？**
   - 5 条性质保证最长路径不超过最短路径的 2 倍，因此查找 O(log n) 有上限保障，同时插入/删除的旋转代价可控。

5. **map 如何自定义排序方式？**
   - 在模板参数中传入比较器：`map<Key, T, Compare>`，Compare 必须是可调用对象，实现严格弱排序。
   ```cpp
   map<int, string, greater<int>> m;  // 降序
   ```

6. **`lower_bound` 和 `upper_bound` 在 set/map 中怎么用？**
   - `lower_bound(k)`：指向第一个 >= k 的元素。
   - `upper_bound(k)`：指向第一个 > k 的元素。
   - 用于范围查找，如统计 [a, b) 区间内的元素。

## 关联知识
- [[unordered_map-unordered_set]]
- [[红黑树]]
- [[二叉搜索树]]
- [[stack-queue-priority_queue]]
- [[迭代器]]
