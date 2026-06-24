---
tags: [c++, AVL树, 平衡二叉树, 二叉搜索树]
created: 2026-06-24
status: 需要补充
---

# AVL树

## 一句话理解
AVL树是一种严格自平衡的二叉搜索树，通过限制任何节点的左右子树高度差不超过1来保证查找效率稳定在 O(log n)。

## 核心原理

### 平衡因子
AVL树在二叉搜索树的基础上引入了**平衡因子**（Balance Factor，bf），定义为：
```
平衡因子 = 右子树高度 - 左子树高度
```
每个节点的平衡因子绝对值不能超过1，即 bf ∈ {-1, 0, 1}。当插入或删除导致 |bf| > 1 时，需要通过旋转恢复平衡。

### 解决的核心问题
二叉搜索树在插入有序数据时会退化为链表（歪脖子树），查找效率从 O(log n) 降到 O(n)。AVL 树通过严格的平衡约束彻底解决了这个问题，但也带来了频繁旋转的代价。

### AVL树 vs 红黑树
| 特性 | AVL树 | 红黑树 |
|------|-------|--------|
| 平衡性 | 严格平衡（\|bf\| ≤ 1） | 近似平衡（最长路径≤2倍最短路径） |
| 查找效率 | 更高（更矮） | 略低 |
| 插入/删除效率 | 频繁旋转，效率较低 | 旋转次数少，效率更高 |
| 实现复杂度 | 中等 | 较复杂 |
| 应用场景 | 查询多、修改少的场景 | 增删改查较均衡的场景 |

## 底层实现

### 四种旋转情况

插入节点破坏平衡后，需要根据插入位置决定旋转方式。设"最近受影响节点"为从插入节点向上回溯时第一个 |bf| > 1 的祖先节点。

#### 1. 左左（LL）- 右单旋
**场景**：插入到最近受影响节点**左子树的左边**

**操作**：对最近受影响节点执行右单旋——该节点接收左子树的右子节点，然后下降为左子树的右子节点，降低整层高度。

```cpp
void RotateR(Node* parent) {
    Node* subL = parent->_left;
    Node* subLR = subL->_right;

    parent->_left = subLR;
    if (subLR) subLR->_parent = parent;

    Node* ppnode = parent->_parent;
    subL->_right = parent;
    parent->_parent = subL;

    if (parent == _root) {
        _root = subL;
        _root->_parent = nullptr;
    } else {
        if (ppnode->_left == parent) ppnode->_left = subL;
        else ppnode->_right = subL;
        subL->_parent = ppnode;
    }
    subL->_bf = parent->_bf = 0;
}
```

#### 2. 右右（RR）- 左单旋
**场景**：插入到最近受影响节点**右子树的右边**

**操作**：对最近受影响节点执行左单旋——该节点接收右子树的左子节点，然后下降为右子树的左子节点。

```cpp
void RotateL(Node* parent) {
    Node* subR = parent->_right;
    Node* subRL = subR->_left;

    parent->_right = subRL;
    if (subRL) subRL->_parent = parent;

    Node* ppnode = parent->_parent;
    subR->_left = parent;
    parent->_parent = subR;

    if (ppnode == nullptr) {
        _root = subR;
        _root->_parent = nullptr;
    } else {
        if (ppnode->_left == parent) ppnode->_left = subR;
        else ppnode->_right = subR;
        subR->_parent = ppnode;
    }
    parent->_bf = subR->_bf = 0;
}
```

#### 3. 左右（LR）- 左右双旋
**场景**：插入到最近受影响节点**左子树的右边**

**操作**：先对左子节点执行左单旋（变为LL情况），再对最近受影响节点执行右单旋。需要根据 subLR 的平衡因子更新最终平衡因子。

```cpp
void RotateLR(Node* parent) {
    Node* subL = parent->_left;
    Node* subLR = subL->_right;
    int bf = subLR->_bf;

    RotateL(parent->_left);
    RotateR(parent);

    if (bf == 1) {
        subL->_bf = -1;
    } else if (bf == -1) {
        parent->_bf = 1;
    }
}
```

#### 4. 右左（RL）- 右左双旋
**场景**：插入到最近受影响节点**右子树的左边**

**操作**：先对右子节点执行右单旋（变为RR情况），再对最近受影响节点执行左单旋。需要根据 subRL 的平衡因子更新最终平衡因子。

```cpp
void RotateRL(Node* parent) {
    Node* subR = parent->_right;
    Node* subRL = subR->_left;
    int bf = subRL->_bf;

    RotateR(parent->_right);
    RotateL(parent);

    if (bf == 1) {
        parent->_bf = -1;
    } else if (bf == -1) {
        subR->_bf = 1;
    }
}
```

### 插入操作完整流程
```cpp
bool InSert(const K& key, const V& val) {
    // 1. 按 BST 规则插入
    // 2. 更新平衡因子
    // 3. 检查 |bf| > 1 并选择对应旋转

    // 平衡因子更新逻辑：
    if (parent->_bf == 0) return;         // 子树高度未变，停止更新
    else if (parent->_bf == 2 || parent->_bf == -2) {
        if (parent->_bf == -2 && cur->_bf == -1) RotateR(parent);       // LL
        else if (parent->_bf == -2 && cur->_bf == 1) RotateLR(parent);  // LR
        else if (parent->_bf == 2 && cur->_bf == 1) RotateL(parent);    // RR
        else if (parent->_bf == 2 && cur->_bf == -1) RotateRL(parent);  // RL
    }
    // 否则向上继续更新
}
```

## 常见应用场景
- 数据库索引（查找密集型场景）
- 需要快速查找的有序数据集合
- 编译器中的符号表管理
- 需要频繁查询、较少修改的场景

## 容易踩坑的地方
1. **平衡因子更新方向容易混淆**：bf = 右子树高度 - 左子树高度。插入在左边时 bf--，插入在右边时 bf++
2. **双旋后的平衡因子更新**：LR 和 RL 双旋后，不能简单将平衡因子设为0，需要根据原始 subLR/subRL 的 bf 值分别处理三种情况（bf=-1,0,1）
3. **根节点旋转**：当 parent 是根节点时，旋转后需要更新 `_root` 指针，并将根节点的 `_parent` 置空
4. **与红黑树的选择**：不要认为 AVL 树比红黑树"更好"，两者各有适用场景。AVL 树查询更快但修改更慢

## 面试高频问题
1. AVL 树的平衡因子定义是什么？如何维护？
2. AVL 树的四种旋转分别发生在什么场景？（LL/RR/LR/RL）
3. 如何在 AVL 树中找到最小不平衡子树？
4. LR 和 RL 双旋时如何正确更新平衡因子？
5. 手写 AVL 树的左单旋和右单旋代码。
6. AVL 树和红黑树的对比：查找、插入、删除的时间复杂度常数区别？

## 关联知识
- [[二叉搜索树]]
- [[红黑树]]
- [[平衡二叉树]]
- [[旋转操作]]
