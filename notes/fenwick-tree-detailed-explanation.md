# 树状数组 (Fenwick Tree / Binary Indexed Tree) 详解

## 1. 什么是树状数组？

[树状数组](https://en.wikipedia.org/wiki/Fenwick_tree)是一种精巧的数据结构，它主要用于高效地解决以下两类问题：

1.  **单点更新 (Point Update):** 修改数组中某个元素的值。
2.  **区间查询 (Range Query):** 查询数组中某个**前缀**区间的和（例如，求 `A[1] + A[2] + ... + A[i]`）。

基于这两个核心操作，可以方便地扩展支持 **区间和查询**：`sum(l, r) = query(r) - query(l-1)`。通过一些技巧（如差分数组），它也能支持区间更新、单点查询等。

### 核心优势

*   **时间复杂度：** 单点更新和前缀和查询都为 **O(log n)**，其中 n 是数组的大小。
*   **空间复杂度：** 只需要 **O(n)** 的额外空间。

### 性能对比

|       数组类型       | 更新     | 前缀和   |
| :------------------- | :------- | :------- |
| **普通数组**         | O(1)     | O(n)     |
| **预计算前缀和数组** | O(n)     | O(1)     |
| **树状数组**         | O(log n) | O(log n) |

## 2. 核心思想：`lowbit` 运算与管辖范围

树状数组的关键在于其独特的结构。它使用一个辅助数组 `tree`（通常与原数组 `A` 等大，**使用 1-based 索引更方便**），`tree[x]` 存储的是原数组 `A` 中**一个特定区间的和**。

这个区间的右端点是 `x`，区间的长度由一个特殊的计算值 `lowbit(x)` 决定。

### `lowbit(x)` 的计算与含义

`lowbit(x)` 计算的是 `x` 的二进制表示中**最右边的那个 '1' (the rightmost set bit)** 所代表的 **数值**（即 2 的某个幂）。

一个非常高效的计算方法是位运算：

```cpp
lowbit(x) = x & (-x)
```

*   **理解：** 计算机中负数通常用补码表示 (`-x` 等于 `x` 按位取反后加 1)。`x & (-x)` 正好可以分离出 `x` 二进制表示中最低位的 '1' 以及它后面的所有 '0' 组成的数。
*   **示例：**
    *   `x = 12` (二进制 `1100`)。最右边的 '1' 在第 2 位 (0-based)，代表的值是 2<sup>2</sup> = 4。 `lowbit(12) = 12 & (-12) = 1100 & 0100 = 0100` (二进制)，结果是 **4**。
    *   `x = 10` (二进制 `1010`)。最右边的 '1' 在第 1 位，代表的值是 2<sup>1</sup> = 2。 `lowbit(10) = 10 & (-10) = 1010 & 0110 = 0010` (二进制)，结果是 **2**。
    *   `x = 13` (二进制 `1101`)。最右边的 '1' 在第 0 位，代表的值是 2<sup>0</sup> = 1。 `lowbit(13) = 13 & (-13) = 1101 & 0011 = 0001` (二进制)，结果是 **1**。

*   **重要区分：** `lowbit(x)` 计算的是最右边 '1' **所代表的数值**，这与 **最低有效位 (Least Significant Bit, LSB)** 不同。LSB 指的是二进制数最右边的那一位 (2<sup>0</sup> 位)。只有当 `x` 是奇数时，`lowbit(x)` 才等于 1，此时最右边的 '1' 恰好是 LSB。

### `tree[x]` 的管辖范围

`tree[x]` 存储的是原数组 `A` 中下标在区间 `(x - lowbit(x), x]` 内所有元素的和。即：

```cpp
tree[x] = A[x - lowbit(x) + 1] + ... + A[x]
```

**示例 (假设数组大小 n=8):**

| 索引 (x) | lowbit(x) | 节点 (Node) | 管辖范围 (Managed Range) | 存储的和 (Stored Sum)       |
| :------- | :-------- | :---------- | :----------------------- | :-------------------------- |
| 1        | 1         | `tree[1]`   | `(0, 1]`                 | `A[1]`                      |
| 2        | 2         | `tree[2]`   | `(0, 2]`                 | `A[1] + A[2]`               |
| 3        | 1         | `tree[3]`   | `(2, 3]`                 | `A[3]`                      |
| 4        | 4         | `tree[4]`   | `(0, 4]`                 | `A[1] + A[2] + A[3] + A[4]` |
| 5        | 1         | `tree[5]`   | `(4, 5]`                 | `A[5]`                      |
| 6        | 2         | `tree[6]`   | `(4, 6]`                 | `A[5] + A[6]`               |
| 7        | 1         | `tree[7]`   | `(6, 7]`                 | `A[7]`                      |
| 8        | 8         | `tree[8]`   | `(0, 8]`                 | `A[1] + A[2] + ... + A[8]`  |

**可视化（概念上的层级关系）:**

```
           tree[8] (A[1..8])
          /         |      \
   tree[4]        tree[6]   tree[7]
  /       \         |
tree[2]  tree[3]  tree[5]
  |
tree[1]
```

*   更新 `A[3]` 会影响 `tree[3]`, `tree[4]`, `tree[8]`。
*   查询 `sum(1..7)` 会访问 `tree[7]`, `tree[6]`, `tree[4]`。

## 3. 操作实现

### a) 单点更新 (Point Update): `update(i, delta)`

当原数组 `A[i]` 的值增加了 `delta` 时，需要更新所有管辖范围**包含** `A[i]` 的 `tree` 节点。这些节点可以通过从 `i` 开始，不断执行 `x = x + lowbit(x)` 来找到，直到 `x` 超出数组范围 `n`。

*   **过程:**
    1.  从 `x = i` 开始。
    2.  `tree[x] += delta`。
    3.  `x += lowbit(x)`。
    4.  重复 2 和 3，直到 `x > n`。
*   **复杂度:** O(log n)。每次 `x += lowbit(x)` 相当于在 `x` 的二进制表示中，把最右边的 '1' 向左“进位”，最多执行 log n 次。

### b) 前缀和查询 (Prefix Sum Query): `query(i)`

计算 `A[1] + ... + A[i]` 的和。这可以通过累加一系列 `tree` 节点的值实现：`sum(1..i) = tree[i] + tree[i - lowbit(i)] + tree[i - lowbit(i) - lowbit(i - lowbit(i))] + ...` 直到下标变为 0。

*   **过程:**
    1.  初始化 `sum = 0`。
    2.  从 `x = i` 开始。
    3.  `sum += tree[x]`。
    4.  `x -= lowbit(x)` (这相当于移除了 `x` 二进制表示中最右边的 '1')。
    5.  重复 3 和 4，直到 `x = 0`。
    6.  返回 `sum`。
*   **复杂度:** O(log n)。每次 `x -= lowbit(x)` 会消除 `x` 二进制表示中的一个 '1'，最多执行 log n 次。

## 4. 初始化 (构建树状数组)

给定初始数组 `A` (假设为 1-based)，构建 `tree` 数组：

### 方法一 (O(n log n))

初始化 `tree` 全为 0。遍历 `A`，对每个 `i` (从 1 到 n) 调用 `update(i, A[i])`。

### 方法二 (O(n)) (更优，常用)

利用 `tree` 结构的性质进行构建。

1.  可以直接将 `A[i]` 的值**累加**到 `tree[i]` (或者先用 `A` 初始化 `tree`，即 `tree[i] = A[i]`)。
2.  遍历 `i` 从 1 到 `n`，计算 `j = i + lowbit(i)`。
3.  如果 `j <= n`，则将 `tree[i]` 的值累加到 `tree[j]` 上 (`tree[j] += tree[i]`)。
    *   **原理：** 这一步是将 `tree[i]` 所管辖的区间和，正确地贡献给包含它的、更高层级的 `tree[j]`。

## 5. C++ 代码示例 (1-based indexing)

```cpp
#include <vector>
#include <numeric> // 仅用于可能的其他操作，此基础实现不需要

class FenwickTree {
private:
  std::vector<long long> tree; // 使用 long long 防止求和溢出
  int n; // 数组的大小 (最大下标)

  int lowbit(int x) {
    return x & (-x);
  }

public:
  // 构造函数: 初始化大小为 size 的空树状数组 (下标 1 到 size)
  FenwickTree(int size) : n(size) {
    tree.resize(n + 1, 0); // 大小为 n+1，下标从 1 开始使用
  }

  // 构造函数: 从现有数组 A 构建 (O(n log n) 初始化)
  // 假设 A 是 1-based, A.size() == n + 1
  FenwickTree(std::vector<int> const& A) : n(A.size() - 1) {
    tree.resize(n + 1, 0);
    for (int i = 1; i <= n; ++i) {
      update(i, A[i]); // 逐个添加
    }
  }

  // O(n) 初始化方法
  // 假设 A 是 1-based, A.size() == n + 1
  void build(std::vector<int> const& A) {
    // 确保大小正确，并清零
    tree.assign(n + 1, 0);
    // 步骤 1 & 2 结合: 初始化并向上传递贡献
    for (int i = 1; i <= n; ++i) {
      tree[i] += A[i]; // 加上 A[i] 本身的值

      int j = i + lowbit(i); // 找到直接 “父” 节点
      if (j <= n) {
        tree[j] += tree[i]; // 将 i 区间的和贡献给 j
      }
    }
  }

  // 单点更新：将 A[i] 的值增加 delta
  // 时间复杂度: O(log n)
  void update(int i, int delta) { // delta 也可以是 long long
    while (i <= n) {
      tree[i] += delta;
      i += lowbit(i); // 移动到下一个受影响的节点
    }
  }

  // 前缀和查询：计算 A[1] + ... + A[i]
  // 时间复杂度: O(log n)
  long long query(int i) {
    long long sum = 0;
    while (i > 0) {
      sum += tree[i];
      i -= lowbit(i); // 移动到下一个贡献区间
    }
    return sum;
  }

  // 区间和查询：计算 A[l] + ... + A[r]
  // 时间复杂度: O(log n)
  long long queryRange(int l, int r) {
    if (l > r) {
      return 0;
    }
    // 通过前缀和相减得到区间和
    return query(r) - query(l - 1);
  }

  // 获取树的大小 (元素数量)
  int size() const {
    return n;
  }
};

// === 示例用法 ===
#include <iostream>
#include <cassert>

int main() {
  // 原始数据 (下标 0 留空，使用 1-based)
  std::vector<int> A{0, 1, 3, -2, 5, 4};
  int n_elements = 5;

  // 使用 O(n) 方法构建
  FenwickTree ft(n_elements);
  ft.build(A);

  // 查询前缀和 sum(1..3) = A[1]+A[2]+A[3] = 1 + 3 + (-2) = 2
  assert(ft.query(3) == 2);

  // 查询区间和 sum(2..4) = A[2]+A[3]+A[4] = 3 + (-2) + 5 = 6
  assert(ft.queryRange(2, 4) == 6);

  // 更新 A[3]，将 A[3] 增加 3 (从 -2 变为 1)
  ft.update(3, 3);

  // 再次查询前缀和 sum(1..3) = 1 + 3 + 1 = 5
  assert(ft.query(3) == 5);
  // 再次查询区间和 sum(2..4) = 3 + 1 + 5 = 9
  assert(ft.queryRange(2, 4) == 9);

  return 0;
}
```

## 6. 优缺点

### 优点

*   **高效:** 单点更新和前缀和查询都是 O(log n)。
*   **空间高效:** O(n) 额外空间。
*   **实现相对简单:** 比起[线段树 (Segment Tree)](https://en.wikipedia.org/wiki/Segment_tree) 的核心代码更短，常数因子通常更小。

### 缺点

*   **功能限制:** 原生结构直接支持的操作有限（主要是基于加法的区间聚合）。对于更复杂的操作，如区间最大/最小值查询、区间乘法、区间赋值等，[线段树](https://en.wikipedia.org/wiki/Segment_tree)通常更灵活、更直接。
*   **1-based 索引:** 虽然不是强制，但使用 1-based 索引能让 `lowbit` 和更新/查询逻辑更自然简洁。

## 7. 应用场景

*   需要频繁进行单点修改和区间（或前缀）和查询的问题。
*   计算逆序对数量（通常需要结合离散化）。
*   动态维护序列信息，作为更复杂算法的组成部分。
*   扩展到二维：处理二维平面上的单点更新和矩形区域查询（二维树状数组）。

## 8. 总结

树状数组是一种巧妙利用位运算（特别是 `lowbit` 操作）的数据结构，通过让每个节点管理原数组的一个特定长度的后缀区间，高效地实现了单点更新和前缀和查询。它在处理这类基础问题时，具有速度快、空间省、代码相对简洁的优点，是算法竞赛和某些数据处理场景中的有力工具。

---
