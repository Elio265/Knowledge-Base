---
tags: [cpp, unordered_map, unordered_set, 关联容器, 哈希表]
created: 2026-06-24
updated: 2026-07-16
status: 面试重点
---

# unordered_map 和 unordered_set

## 一句话理解
unordered_map 和 unordered_set 是基于哈希表实现的无序关联容器，**平均 O(1)** 的插入、查找和删除性能，但不保证元素顺序，是 C++11 引入的"无序"版 map/set。

## 核心原理

- 与 map/set 的用法基本一致，但**元素不排序**。
- 底层使用**哈希表**，通过哈希函数将键映射到桶（bucket）；冲突节点如何组织属于实现细节。
- 平均常数时间的操作性能远快于红黑树的 O(log n)，但在**最坏情况**下可能退化为 O(n)（大量哈希冲突）。
- C++11 标准引入，头文件 `<unordered_map>` 和 `<unordered_set>`。

### 接口对比（与 map/set 相似）

```cpp
unordered_map<string, int> um;
um.insert(make_pair("apple", 5));
um["banana"] = 3;
auto it = um.find("apple");
if (it != um.end()) { /* found */ }

unordered_set<int> us = {5, 3, 8, 1};
us.insert(10);
us.erase(5);
```

**区别**：
- unordered_map/unordered_set **没有 `lower_bound` / `upper_bound`**（因为无序）。
- 支持 `bucket_count()`、`load_factor()`、`max_load_factor()` 等哈希表特有接口。
- 迭代器是**前向迭代器**（不是双向），但同样可用范围 for 遍历。

### 自定义哈希与相等比较

```cpp
// 自定义类型作为键
struct Person {
    string name;
    int age;
    bool operator==(const Person& other) const {
        return name == other.name && age == other.age;
    }
};

// 自定义哈希函数
struct PersonHash {
    size_t operator()(const Person& p) const {
        return hash<string>()(p.name) ^ (hash<int>()(p.age) << 1);
    }
};

unordered_set<Person, PersonHash> us;
```

如果没有默认的哈希函数（如自定义类型），必须提供自定义哈希仿函数。

## 底层实现——哈希表

### 数据结构：桶与冲突

不同 key 落到同一个桶就是哈希冲突。链地址法（一个桶挂多个节点）是常见实现，但 C++ 标准不强制具体冲突结构。

```cpp
// 简化结构
vector<list<T>> _table;    // 桶数组，每个桶是一个链表
size_t _size;              // 实际元素个数
```

标准库默认使用 `std::hash<Key>`，对常见类型（int、string、指针等）有特化版本。自定义 key 还需提供与相等判断一致的哈希函数。

### 哈希冲突处理

| 方法 | 说明 |
|------|------|
| **链地址法（开散列）** | 每个桶关联多个冲突元素，常见于标准库实现 |
| **开放定址法（闭散列）** | 冲突时探测其他空位，是另一类哈希表实现 |
| 线性探测 | `h(key) + 1, +2, +3...` —— 容易产生主聚集 |
| 二次探测 | `h(key) + 1^2, +2^2, +3^2...` —— 减少聚集 |

具体冲突策略、桶大小和增长规则均为库实现细节；面试回答重点是“冲突会拉长查找，rehash 通过增加桶数降低负载因子”。

### 负载因子（Load Factor）

```
负载因子 = 已存储元素个数 / 总桶数
```

- **负载因子越大**：冲突越多，性能下降。
- **负载因子越小**：空间浪费越多。
- 当 load_factor > max_load_factor 时，哈希表触发**rehash（重新哈希）**：扩大桶数组数量，将所有元素重新映射。

```cpp
unordered_map<int, int> um;
cout << um.load_factor();           // 当前负载因子
um.max_load_factor(0.7f);           // 设置最大负载因子
cout << um.bucket_count();          // 当前桶数量
cout << um.bucket_size(3);          // 第 3 个桶中的元素数
```

### Rehash 过程

1. 选择足够多的新桶。
2. 将所有已存储元素重新分桶。
3. 旧桶结构释放。**此过程会使所有迭代器失效**；指向元素的引用和指针通常仍有效。

## 常见应用场景

1. **需要 O(1) 查找性能**：如缓存系统、字典查找、标记已访问。
2. **数据不需要排序**：只要唯一性，不关心顺序。
3. **大规模数据去重**：比 set 更快。
4. **两数之和 / 三数之和**等算法题：经典哈希表解法。

```cpp
// 两数之和（经典哈希表应用）
vector<int> twoSum(vector<int>& nums, int target) {
    unordered_map<int, int> mp;  // 值 -> 下标
    for (int i = 0; i < nums.size(); ++i) {
        int complement = target - nums[i];
        if (mp.count(complement)) return {mp[complement], i};
        mp[nums[i]] = i;
    }
    return {};
}
```

## 容易踩坑的地方

- **迭代器失效**
  - **insert 可能触发 rehash**，导致所有迭代器失效。
  - **erase 只使被删除元素的迭代器失效**，其他不受影响。
  - 如果确定不会触发 rehash，insert 不使既有迭代器失效。

- **无序性**：遍历顺序不可预测，与插入顺序无关。依赖顺序时要改用 map。
- **自定义类型必须提供哈希函数**：不要忘了 `operator==` 和 `hash` 特化。
- **最坏情况 O(n)**：如果哈希质量差或冲突集中到少数桶，性能会退化。标准库不保证哈希随机化，面向不可信输入时需额外考虑碰撞攻击风险。
- **`operator[]` 行为同 map**：键不存在时自动插入默认值，造成不必要的默认构造。
- **内存开销大**：桶数组通常比元素数量大得多（负载因子 < 1），内存占用高于红黑树。
- **rehash 代价大**：触发 rehash 时需要 O(n) 时间重新映射，不适合对实时性要求极高的场景。可先用 `reserve(n)` 预分配桶数。

### 时间复杂度表

| 操作 | 平均 | 最坏 |
|------|------|------|
| 插入 | O(1) | O(n) |
| 删除 | O(1) | O(n) |
| 查找 | O(1) | O(n) |
| Rehash | O(n) | O(n) |

### unordered_map vs map 对比

| 特性 | unordered_map | map |
|------|--------------|-----|
| 底层结构 | 哈希表 | 红黑树 |
| 时间复杂度 | 平均 O(1)，最坏 O(n) | O(log n) |
| 有序性 | 无序 | 按键排序 |
| 范围查询 | 不支持 | 支持 lower_bound / upper_bound |
| 内存占用 | 取决于桶数、负载因子和实现 | 取决于节点和比较器实现 |
| 自定义哈希 | 需要提供哈希函数 | 只需比较函数 |
| C++ 版本 | C++11 引入 | C++98 |

## 面试高频问题

1. **unordered_map 如何解决哈希冲突？**
   - 不同 key 落同一桶即冲突；链地址法是常见实现，标准不强制具体组织方式。

2. **什么是负载因子？为什么要控制负载因子？**
   - 负载因子 = 元素数 / 桶数。控制负载因子是为了平衡冲突率和空间利用率，超过阈值时 rehash。

3. **rehash 是怎么进行的？迭代器如何受影响？**
   - 扩大桶数组 -> 重新计算所有元素的哈希映射 -> 释放旧表。所有迭代器失效。

4. **什么时候用 unordered_map，什么时候用 map？**
   - 需要 O(1) 查找且不关心顺序 -> unordered_map。
   - 需要有序、范围查找或 O(log n) 稳定的最坏性能 -> map。

5. **如何为自定义类型实现 unordered_map 的键？**
   - 必须提供 `operator==` 和特化的 `std::hash`（或自定义哈希仿函数）。

6. **unordered_map 的 `reserve(n)` 是做什么的？**
   - 预分配足够桶数，使得插入 n 个元素不触发 rehash，提高批量插入性能。

## 我的薄弱点

- 曾将“哈希冲突”说成映射到同一个 key；准确说法是不同 key 落入同一个桶。
- 需要记住 rehash 会使迭代器失效，但通常不使指向元素的引用和指针失效。

## 成长记录

- 2026-07-16：能区分 map 的有序 `O(log n)` 与 unordered_map 的平均 `O(1)`，并根据范围查询需求选择容器。
- 2026-07-16：理解负载因子、rehash、冲突和多重关联容器的边界。

## 关联知识
- [[map-set]]
- [[红黑树]]
