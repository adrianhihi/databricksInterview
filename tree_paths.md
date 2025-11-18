# Find Path in Fibonacci Tree（Pre-order / In-order / Post-order 全解）

> 说明：  
> - 语言：中文讲解  
> - 代码：Java，注释使用英语  
> - 每个版本都自带 1–2 个简单的“单元测试”示例（用 `main + assert`）

---

## 0. 原题说明 & 统一思路

原题 **Basic 版本** 明确说的是：

> “its nodes are labeled in **pre-order traversal** starting from 0…”

所以原始题目默认的编号方式是 **先序遍历（pre-order）**。下面我们先给出 pre-order 版本的完整解法，然后给两个 follow‑up：  
- in-order 编号  
- post-order 编号  

三种版本的**核心套路完全一样**，区别只在于：  
> 给定 `(order, index)` 时，如何根据不同遍历顺序把这个节点定位到「左子树 / 右子树 / 根」。

---

### 0.1 Fibonacci Tree 定义回顾

- 阶数为 `n` 的 Fibonacci tree 记为 `Fn(n)`  
- 若 `n >= 2`：
  - 左子树：`Fn(n-2)`
  - 右子树：`Fn(n-1)`
- `n = 0` 或 `1` 时，树只有一个节点（叶子节点）

记 `size[n]` 为 `Fn(n)` 中的节点个数，则：

- `size[0] = size[1] = 1`
- `size[n] = 1 + size[n-2] + size[n-1]`  （根 + 左 + 右）

---

### 0.2 通用解题框架（对三种遍历都适用）

1. **预处理每个阶数的节点数 `size[0..order]`**  
2. 实现函数 `indexToPath(order, idx)`：  
   - 输入：整棵树的阶数 `order`，以及某种遍历方式下的编号 `idx`（从 0 开始）  
   - 输出：从根到该节点的路径字符串，只包含 `'L'` / `'R'`  

   > 注意：这一部分的逻辑，会随 pre/in/post 顺序不同而不同，其余都一样。

3. 对 `source` 和 `dest` 分别求：
   - `ps = indexToPath(order, source)`   // 根 → source  
   - `pd = indexToPath(order, dest)`     // 根 → dest  
4. 找出 `ps` 和 `pd` 的 **最长公共前缀长度** `k`：  
   - 公共前缀对应的就是从根到 **LCA**（最近公共祖先）的路径  
5. 从 `source` 到 `dest` 的最终路径为：
   - 先从 `source` 一直往上走到 LCA：需要 `ps.length - k` 个 `'U'`  
   - 再从 LCA 按 `pd` 的后缀走到 `dest`：追加 `pd.substring(k)`  

---

### 0.3 复杂度分析（通用）

- 预处理 `size[]`：`O(order)`  
- `indexToPath` 每一层递归只做 O(1) 的切分，递归深度 ≤ `order` → 单次 `O(order)`  
- 整体：两次 `indexToPath` + 一次线性扫描公共前缀 → **时间复杂度 `O(order)`，空间复杂度 `O(order)`**  

如果用总节点数 `N` 来度量，因为 Fibonacci tree 高度大约是 `O(order) = O(log N)`，所以也可以写成 **`O(log N)`**。

---

## 1. Basic：Pre-order 编号版本（原题）

### 1.1 Pre-order 下的编号分布

对于 `Fn(n)`：

- 先序顺序：`[root] [left-preorder] [right-preorder]`
- 记 `leftSize = size[n-2]`，`rightSize = size[n-1]`  
  则整棵树上：
  - `idx == 0`：根节点  
  - `1 .. leftSize`：左子树  
  - `1 + leftSize .. 1 + leftSize + rightSize - 1`：右子树  

所以局部到子树的递推关系如下：

```text
prePath(n, idx):
    if idx == 0:
        return ""               // root

    left = size[n-2]

    if idx - 1 < left:
        // in left subtree
        return 'L' + prePath(n-2, idx-1)
    else:
        // in right subtree
        return 'R' + prePath(n-1, idx-1-left)
```

---

### 1.2 Java 实现（Pre-order）

```java
public class FibonacciTreePathPreorder {

    // Public API: find path from source to dest in a preorder-labeled Fibonacci tree
    public String findPath(int order, int source, int dest) {
        int[] size = computeSizes(order);

        // 1) Path from root to source (L/R only)
        String ps = buildPreorderPath(order, source, size);

        // 2) Path from root to dest (L/R only)
        String pd = buildPreorderPath(order, dest, size);

        // 3) Longest common prefix = path from root to LCA
        int i = 0;
        int minLen = Math.min(ps.length(), pd.length());
        while (i < minLen && ps.charAt(i) == pd.charAt(i)) {
            i++;
        }

        // 4) From source up to LCA: add 'U' steps
        StringBuilder ans = new StringBuilder();
        for (int j = i; j < ps.length(); j++) {
            ans.append('U');
        }

        // 5) From LCA down to dest: append remaining suffix of pd
        ans.append(pd.substring(i));

        return ans.toString();
    }

    // Compute size[k] = number of nodes in Fibonacci tree Fn(k)
    private int[] computeSizes(int order) {
        int[] size = new int[order + 1];
        size[0] = 1;
        if (order >= 1) {
            size[1] = 1;
        }
        for (int i = 2; i <= order; i++) {
            size[i] = 1 + size[i - 2] + size[i - 1];
        }
        return size;
    }

    // Wrapper to build preorder path from root to node with global index idx
    private String buildPreorderPath(int order, int idx, int[] size) {
        StringBuilder sb = new StringBuilder();
        buildPreorderPathHelper(order, idx, size, sb);
        return sb.toString();
    }

    // Recursive helper:
    // In subtree Fn(order), root has local preorder index 0.
    // We append moves from this root to the node with local index idx.
    private void buildPreorderPathHelper(int order, int idx, int[] size, StringBuilder sb) {
        if (idx == 0) {
            // Already at this subtree root
            return;
        }
        if (order < 2) {
            // Leaf Fibonacci trees (order 0 or 1) have only one node
            throw new IllegalArgumentException("Invalid index for leaf Fibonacci tree");
        }

        int leftSize = size[order - 2]; // size of left subtree Fn(order-2)

        if (idx - 1 < leftSize) {
            // Node is in the left subtree
            sb.append('L');                // go to left child
            int childIdx = idx - 1;        // local index inside left subtree
            buildPreorderPathHelper(order - 2, childIdx, size, sb);
        } else {
            // Node is in the right subtree
            sb.append('R');                // go to right child
            int childIdx = idx - 1 - leftSize; // local index inside right subtree
            buildPreorderPathHelper(order - 1, childIdx, size, sb);
        }
    }

    // --- Simple unit tests ---
    public static void main(String[] args) {
        FibonacciTreePathPreorder solver = new FibonacciTreePathPreorder();

        // Test 1: 官方示例（order = 5, source = 5, dest = 7）
        // Expected: "UUURL"
        String p1 = solver.findPath(5, 5, 7);
        System.out.println("Preorder test1: " + p1);
        assert "UUURL".equals(p1);

        // Test 2: 一个更小的例子
        // order = 2 时，结构为：
        //        root
        //       /    \
        //    Fn(0)   Fn(1)
        // Preorder 编号为: root(0), left(1), right(2)
        // 从 root(0) 到 left(1) 的路径应该是 "L"
        String p2 = solver.findPath(2, 0, 1);
        System.out.println("Preorder test2: " + p2);
        assert "L".equals(p2);
    }
}
```

---

## 2. Follow-up：In-order 编号版本

> 与 Basic 的区别：  
> 只改“index → 左 / 右 / 根”的划分逻辑，其余完全一样。

### 2.1 In-order 下的编号分布

对 `Fn(n)`，中序遍历顺序：`[left-inorder] [root] [right-inorder]`。

记：

- `leftSize = size[n-2]`  
- `rightSize = size[n-1]`  

则整棵树的 in-order 编号为：

```text
indices [0 .. leftSize-1]         -> left subtree
index  leftSize                   -> root
indices [leftSize+1 .. leftSize+rightSize] -> right subtree
```

因此递推关系：

```text
inPath(n, idx):
    if n <= 1:   // leaf tree
        assert idx == 0
        return ""

    left = size[n-2]

    if idx < left:
        return 'L' + inPath(n-2, idx)                       // left subtree
    else if idx == left:
        return ""                                           // root
    else:
        return 'R' + inPath(n-1, idx - left - 1)            // right subtree
```

---

### 2.2 Java 实现（In-order）

```java
public class FibonacciTreePathInorder {

    // Public API: path in an inorder-labeled Fibonacci tree
    public String findPath(int order, int source, int dest) {
        int[] size = computeSizes(order);

        String ps = buildInorderPath(order, source, size);
        String pd = buildInorderPath(order, dest, size);

        // Longest common prefix = path to LCA
        int i = 0;
        int minLen = Math.min(ps.length(), pd.length());
        while (i < minLen && ps.charAt(i) == pd.charAt(i)) {
            i++;
        }

        StringBuilder ans = new StringBuilder();
        for (int j = i; j < ps.length(); j++) {
            ans.append('U');
        }
        ans.append(pd.substring(i));
        return ans.toString();
    }

    private int[] computeSizes(int order) {
        int[] size = new int[order + 1];
        size[0] = 1;
        if (order >= 1) {
            size[1] = 1;
        }
        for (int i = 2; i <= order; i++) {
            size[i] = 1 + size[i - 2] + size[i - 1];
        }
        return size;
    }

    private String buildInorderPath(int order, int idx, int[] size) {
        StringBuilder sb = new StringBuilder();
        buildInorderPathHelper(order, idx, size, sb);
        return sb.toString();
    }

    // Recursive helper for inorder labeling
    private void buildInorderPathHelper(int order, int idx, int[] size, StringBuilder sb) {
        if (order <= 1) {
            // Single node tree: only index 0 is valid and it is the root
            if (idx != 0) {
                throw new IllegalArgumentException("Invalid index for leaf Fibonacci tree");
            }
            return;
        }

        int leftSize = size[order - 2]; // size of left subtree Fn(order-2)

        if (idx < leftSize) {
            // Node lies in the left subtree
            sb.append('L');
            buildInorderPathHelper(order - 2, idx, size, sb);
        } else if (idx == leftSize) {
            // This index corresponds to the root of this subtree
            return;
        } else {
            // Node lies in the right subtree
            sb.append('R');
            int childIdx = idx - leftSize - 1; // subtract left subtree + root
            buildInorderPathHelper(order - 1, childIdx, size, sb);
        }
    }

    // --- Simple unit tests ---
    public static void main(String[] args) {
        FibonacciTreePathInorder solver = new FibonacciTreePathInorder();

        // 对 order = 2，树结构同样是:
        //        root
        //       /    \
        //     L        R
        // Inorder 顺序为 [L, root, R]，
        // 所以 index: L=0, root=1, R=2

        // Test 1: root(1) -> L(0)  应该是 "L"
        String p1 = solver.findPath(2, 1, 0);
        System.out.println("Inorder test1: " + p1);
        assert "L".equals(p1);

        // Test 2: L(0) -> root(1) 应该是 "U"
        String p2 = solver.findPath(2, 0, 1);
        System.out.println("Inorder test2: " + p2);
        assert "U".equals(p2);
    }
}
```

---

## 3. Follow-up：Post-order 编号版本

同样，只是换成 post-order 的 index 划分。

### 3.1 Post-order 下的编号分布

对 `Fn(n)`，后序遍历顺序：`[left-postorder] [right-postorder] [root]`。

仍记：

- `leftSize  = size[n-2]`  
- `rightSize = size[n-1]`  

则 post-order 编号为：

```text
indices [0 .. leftSize-1]                 -> left subtree
indices [leftSize .. leftSize+rightSize-1] -> right subtree
index   leftSize + rightSize              -> root
```

所以递推：

```text
postPath(n, idx):
    if n <= 1:
        assert idx == 0
        return ""

    left  = size[n-2]
    right = size[n-1]

    if idx < left:
        return 'L' + postPath(n-2, idx)              // left subtree
    else if idx < left + right:
        return 'R' + postPath(n-1, idx - left)       // right subtree
    else:
        return ""                                    // root
```

---

### 3.2 Java 实现（Post-order）

```java
public class FibonacciTreePathPostorder {

    // Public API: path in a postorder-labeled Fibonacci tree
    public String findPath(int order, int source, int dest) {
        int[] size = computeSizes(order);

        String ps = buildPostorderPath(order, source, size);
        String pd = buildPostorderPath(order, dest, size);

        // Longest common prefix = path to LCA
        int i = 0;
        int minLen = Math.min(ps.length(), pd.length());
        while (i < minLen && ps.charAt(i) == pd.charAt(i)) {
            i++;
        }

        StringBuilder ans = new StringBuilder();
        for (int j = i; j < ps.length(); j++) {
            ans.append('U');
        }
        ans.append(pd.substring(i));
        return ans.toString();
    }

    private int[] computeSizes(int order) {
        int[] size = new int[order + 1];
        size[0] = 1;
        if (order >= 1) {
            size[1] = 1;
        }
        for (int i = 2; i <= order; i++) {
            size[i] = 1 + size[i - 2] + size[i - 1];
        }
        return size;
    }

    private String buildPostorderPath(int order, int idx, int[] size) {
        StringBuilder sb = new StringBuilder();
        buildPostorderPathHelper(order, idx, size, sb);
        return sb.toString();
    }

    // Recursive helper for postorder labeling
    private void buildPostorderPathHelper(int order, int idx, int[] size, StringBuilder sb) {
        if (order <= 1) {
            // Single node tree: only index 0 is valid and it is the root
            if (idx != 0) {
                throw new IllegalArgumentException("Invalid index for leaf Fibonacci tree");
            }
            return;
        }

        int leftSize = size[order - 2];
        int rightSize = size[order - 1];

        if (idx < leftSize) {
            // Node is in the left subtree
            sb.append('L');
            buildPostorderPathHelper(order - 2, idx, size, sb);
        } else if (idx < leftSize + rightSize) {
            // Node is in the right subtree
            sb.append('R');
            int childIdx = idx - leftSize;
            buildPostorderPathHelper(order - 1, childIdx, size, sb);
        } else {
            // idx == leftSize + rightSize: this is the root node
            return;
        }
    }

    // --- Simple unit tests ---
    public static void main(String[] args) {
        FibonacciTreePathPostorder solver = new FibonacciTreePathPostorder();

        // 对 order = 2，结构相同：
        //        root
        //       /    \
        //     L        R
        // Postorder 顺序为 [L, R, root]，
        // index: L=0, R=1, root=2

        // Test 1: root(2) -> L(0) 应该是 "L"
        String p1 = solver.findPath(2, 2, 0);
        System.out.println("Postorder test1: " + p1);
        assert "L".equals(p1);

        // Test 2: L(0) -> root(2) 应该是 "U"
        String p2 = solver.findPath(2, 0, 2);
        System.out.println("Postorder test2: " + p2);
        assert "U".equals(p2);
    }
}
```

---

## 4. 整体小结

1. **三种遍历的唯一差异**：  
   - 如何根据 `(order, idx)` 判断当前节点在「左子树 / 右子树 / 根」中的哪个位置，并转换为更小阶数子树里的局部 index。  

2. **共同框架**：  
   - `size[n]` 预处理  
   - `indexToPath(order, idx)` 递归构造 L/R 路径  
   - `source/dest` 路径求最长公共前缀 → LCA  
   - 上行 `'U'` + 下行 `'L'/'R'` 得到答案  

3. **复杂度统一**：  
   - 时间：`O(order)` ≈ `O(log N)`  
   - 空间：`O(order)`  

这份文件可以直接作为面试复习资料或放到仓库中使用。
