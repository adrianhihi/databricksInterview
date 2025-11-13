
# Find Optimal Commute — 解题思路与多版本解法总结 (Java)

> 说明：  
> - 讲解用中文，方便面试时复述。  
> - 代码用 Java，注释使用英文，便于现场和面试官讨论。  
> - 一共三种版本：  
>   - Part 1：Min-Heap / Dijkstra 一次性扫描所有模式（你最熟的那类写法）。  
>   - Part 2：对每个 mode 分别 BFS（k 次 BFS，逻辑最直观，也是我们之前写的版本）。  
>   - Part 3：单次 BFS，但在状态里带上 mode（给“只 BFS 一次”的面试官一个交代）。  

---

## 1. 题目重述与约束

> You are commuting across a simplified map of San Francisco, represented as a 2D grid. Each cell is one of:
> - `'S'`: start (home)
> - `'D'`: destination (office)
> - `'1'..'k'`: road reserved for exactly one transportation mode
> - `'X'`: impassable roadblock
>
> Arrays:
> - `modes[i]`: mode name (e.g., "Walk", "Bike", "Car"...)
> - `times[i]`: time per block for mode i
> - `costs[i]`: cost per block for mode i
>
> You can move up / down / left / right。  
> 一旦选择了某种交通方式（例如数字 `'2'` → 对应 `modes[1]`），从 `S` 到 `D` 的整条路径上：  
> - 只能走：`S`、`D` 和 digit 为 `'2'` 的格子；  
> - 不能走其它 digit，也不能走 `'X'`。  
>
> 对某个 mode i：  
> - 每经过 **一个 digit 为该 mode 的格子**，增加 `times[i]` 分钟和 `costs[i]` 美元。  
> - `S` 和 `D` 不计入 time / cost（只作为起点终点）。  
>
> 目标：  
> - 在所有可达的 mode 中，选出 **总时间最小** 的；  
> - 若有多个 time 相同，选 **总 cost 最小** 的；  
> - 返回对应 `modes[i]`，如果没有任何 mode 能从 `S` 到 `D`，返回 `""`。

约束：

```text
1 ≤ grid.length, grid[0].length ≤ 100
Exactly one 'S' and one 'D' exist.
modes.length ≤ 10
costs[i], times[i] are non-negative integers.
```

矩阵最大 100×100，mode 最多 10 个，规模非常友好。

---

## 2. 三种思路 & 时间复杂度对比

记：`R = rows`, `C = cols`, `k = modes.length (≤ 10)`。

### 思路 A：Dijkstra / Min-Heap（Part 1）

- 用最小堆按 `(time, cost)` 排序。
- 状态里带 `(r, c, modeIndex, time, cost)`。
- 从 `S` 的临近 digit cell 选择 mode，并沿着该 mode 的道路 + `D` 做 Dijkstra。
- **复杂度**：状态数 `O(k * R * C)`，每个状态最多入堆一次：  
  `O(k * R * C * log(k * R * C))`。  
- 在本题约束下完全没压力。

### 思路 B：对每个 mode 单独 BFS（Part 2）

- 对每个 mode i：
  - 只允许走当前 digit `'1' + i` + `S` + `D`；
  - BFS 求从 `S` 到 `D` 的最少“道路格子数”；
  - time = blocks × times[i]，cost = blocks × costs[i]；
- 最后选最小 time / cost 的 mode。

**复杂度**：  
- 一次 BFS：`O(R * C)`；  
- 共 k 次：`O(k * R * C)`；  
- 题目里 `k ≤ 10`，可以看作 `O(R * C)`。

> 这是最直观、最容易写对的版本。

### 思路 C：状态里带 mode 的单次 BFS（Part 3）

- 把 `(r, c, modeIndex)` 当作状态，一次 BFS，同时探索所有 mode；
- 时间复杂度仍然是 `O(k * R * C)`，只是**代码风格上只写了一次 BFS**。

> 这个版本主要是政治正确：当面试官坚持“最好只 BFS 一次”的时候，用来沟通。

---

## 3. Part 1 — Min-Heap / Dijkstra 解法（一次性所有模式）

### 思路

1. 先从 grid 中找到 `S` 和 `D` 的坐标。  
2. 从 `S` 出发，看四个方向的邻居：
   - 如果是 digit `ch` 在 `'1'..'9'` 且对应一个合法 mode：  
     - 这一步相当于“选择 mode 并踏上第一块道路”：
     - time = `times[modeIndex]`，cost = `costs[modeIndex]`，入堆。  
   - 如果是 `D`：说明 `S` 和 `D` 相邻且中间没有道路格子，time / cost = 0。  
3. 堆中的每个状态为：`(r, c, modeIndex, time, cost)`：
   - 每次从堆取出 `(time 最小，若相同 cost 最小)` 的状态。
   - 若当前 cell 是 `D`：
     - 按堆的性质，这一定是全局最小 `(time, cost)`；
     - 直接返回 `modes[modeIndex]`。
   - 否则扩展四个邻居：
     - 如果邻居是 `D`：time / cost 不变；
     - 如果邻居是 **同一个 mode 的 digit**：
       - time += `times[modeIndex]`，cost += `costs[modeIndex]`。

4. 用 `visited[mode][r][c]` 防止重复处理同一个状态。

### 代码实现（Java）

```java
import java.util.*;

public class SolutionPart1_Dijkstra {
    // State for Dijkstra
    static class State {
        int r, c;
        int mode;      // index in modes[]
        long time;     // accumulated time
        long cost;     // accumulated cost

        State(int r, int c, int mode, long time, long cost) {
            this.r = r;
            this.c = c;
            this.mode = mode;
            this.time = time;
            this.cost = cost;
        }
    }

    public String findOptimalCommute(char[][] grid, String[] modes, int[] costs, int[] times) {
        int rows = grid.length;
        int cols = grid[0].length;

        int sr = -1, sc = -1;
        int dr = -1, dc = -1;

        // 1) Locate S and D
        for (int r = 0; r < rows; r++) {
            for (int c = 0; c < cols; c++) {
                if (grid[r][c] == 'S') {
                    sr = r; sc = c;
                } else if (grid[r][c] == 'D') {
                    dr = r; dc = c;
                }
            }
        }

        if (sr == -1 || dr == -1) return "";

        // Edge case: S == D (very unlikely by problem statement)
        if (sr == dr && sc == dc) {
            // No blocks traversed; choose arbitrary mode, e.g., modes[0]
            return modes.length > 0 ? modes[0] : "";
        }

        // Min-heap ordered by (time, cost)
        PriorityQueue<State> pq = new PriorityQueue<>((a, b) -> {
            if (a.time != b.time) return Long.compare(a.time, b.time);
            return Long.compare(a.cost, b.cost);
        });

        boolean[][][] visited = new boolean[modes.length][rows][cols];
        int[] dR = {1, -1, 0, 0};
        int[] dC = {0, 0, 1, -1};

        // 2) Initialize from neighbors of S
        //    This is where we "choose" a mode by stepping onto its digit cell.
        long bestTime = Long.MAX_VALUE;
        long bestCost = Long.MAX_VALUE;
        int bestMode = -1;

        for (int k = 0; k < 4; k++) {
            int nr = sr + dR[k];
            int nc = sc + dC[k];
            if (nr < 0 || nr >= rows || nc < 0 || nc >= cols) continue;

            char ch = grid[nr][nc];
            if (ch == 'X') continue;

            if (ch >= '1' && ch <= '9') {
                int modeIndex = ch - '1';
                if (modeIndex < 0 || modeIndex >= modes.length) continue;

                // Step onto first block of this mode
                long time = times[modeIndex];
                long cost = costs[modeIndex];
                pq.offer(new State(nr, nc, modeIndex, time, cost));
            } else if (ch == 'D') {
                // S adjacent to D: no mode blocks traversed
                long time = 0L;
                long cost = 0L;
                if (time < bestTime || (time == bestTime && cost < bestCost)) {
                    bestTime = time;
                    bestCost = cost;
                    bestMode = 0; // any mode is fine; choose modes[0]
                }
            }
        }

        // 3) Dijkstra
        while (!pq.isEmpty()) {
            State cur = pq.poll();
            int r = cur.r, c = cur.c, m = cur.mode;

            if (visited[m][r][c]) continue;
            visited[m][r][c] = true;

            // If we reached D, this is the globally optimal (time, cost)
            if (r == dr && c == dc) {
                return modes[m];
            }

            // Expand neighbors
            for (int i = 0; i < 4; i++) {
                int nr = r + dR[i];
                int nc = c + dC[i];
                if (nr < 0 || nr >= rows || nc < 0 || nc >= cols) continue;

                char ch = grid[nr][nc];
                if (ch == 'X') continue;

                if (ch == 'D') {
                    // Reaching D does not add extra time/cost
                    if (!visited[m][nr][nc]) {
                        pq.offer(new State(nr, nc, m, cur.time, cur.cost));
                    }
                } else if (ch == (char)('1' + m)) {
                    // Same mode road segment
                    if (!visited[m][nr][nc]) {
                        long newTime = cur.time + times[m];
                        long newCost = cur.cost + costs[m];
                        pq.offer(new State(nr, nc, m, newTime, newCost));
                    }
                }
                // Other digits are not allowed because we cannot switch mode
            }
        }

        // If we found direct S-D (no blocks) earlier
        if (bestMode != -1) {
            return modes[bestMode];
        }

        // No valid path for any mode
        return "";
    }

    // Simple test harness
    public static void main(String[] args) {
        SolutionPart1_Dijkstra solver = new SolutionPart1_Dijkstra();

        // Example 1
        char[][] grid1 = {
            {'3', '3', 'S', '2', 'X', 'X'},
            {'3', '1', '1', '2', 'X', '2'},
            {'3', '1', '1', '2', '2', '2'},
            {'3', '1', '1', '1', 'D', '3'},
            {'3', '3', '3', '3', '3', '4'},
            {'4', '4', '4', '4', '4', '4'}
        };
        String[] modes = {"Walk", "Bike", "Car", "Train"};
        int[] costs = {0, 1, 3, 2};
        int[] times = {3, 2, 1, 1};
        System.out.println("Part1 Example1: " + solver.findOptimalCommute(grid1, modes, costs, times)); // Expected: Bike

        // Example 2
        char[][] grid2 = {
            {'S', '1', '1'},
            {'2', '2', '2'},
            {'D', '1', '1'}
        };
        System.out.println("Part1 Example2: " + solver.findOptimalCommute(grid2, modes, costs, times)); // Expected: Bike

        // Example 3
        char[][] grid3 = {
            {'S', '1', 'X'},
            {'X', '3', '3'},
            {'2', '2', 'D'}
        };
        String[] modes3 = {"Walk", "Bike", "Car"};
        int[] costs3 = {0, 1, 3};
        int[] times3 = {3, 2, 1};
        System.out.println("Part1 Example3: " + solver.findOptimalCommute(grid3, modes3, costs3, times3)); // Expected: ""
    }
}
```

---

## 4. Part 2 — 对每个 mode 分别 BFS（k 次 BFS）

这是你原先写的版本，我们只做一个关键修正：**计数的时候不要 Off-by-one**。

### 核心修正点

- 我们让 `d` 表示“已经走过的该 mode 的 **道路格子数量**（digit block 数）”。
- 从 `S` 出发时，`d = 0`。  
- 当走进一个 digit 为 `modeChar` 的格子时，`d + 1`。  
- 当走进 `D` 时，不再加 1，直接使用当前 `d`。

也就是把原来：

```java
if (grid[nr][nc] == 'D') {
    foundDist = d + 1;
    queue.clear();
    break;
}
```

改成：

```java
if (grid[nr][nc] == 'D') {
    // FIX: we do NOT add 1 here. 'd' already counts how many mode blocks we traversed.
    foundBlocks = d;
    queue.clear();
    break;
}
```

### 代码实现（Java）

```java
import java.util.*;

public class SolutionPart2_BFSPerMode {

    public String findOptimalCommute(char[][] grid, String[] modes, int[] costs, int[] times) {
        int rows = grid.length;
        int cols = grid[0].length;

        int sr = -1, sc = -1;
        int dr = -1, dc = -1;

        // Locate S and D
        for (int r = 0; r < rows; r++) {
            for (int c = 0; c < cols; c++) {
                if (grid[r][c] == 'S') {
                    sr = r; sc = c;
                } else if (grid[r][c] == 'D') {
                    dr = r; dc = c;
                }
            }
        }

        if (sr == -1 || dr == -1) return "";

        long bestTime = Long.MAX_VALUE;
        long bestCost = Long.MAX_VALUE;
        int bestIdx = -1;

        int[] dR = {1, -1, 0, 0};
        int[] dC = {0, 0, 1, -1};

        // For each mode independently
        for (int i = 0; i < modes.length; i++) {
            char modeChar = (char)('1' + i);

            boolean[][] visited = new boolean[rows][cols];
            Queue<int[]> queue = new ArrayDeque<>();
            // state: [r, c, blocks]
            queue.offer(new int[]{sr, sc, 0});
            visited[sr][sc] = true;

            int foundBlocks = -1;

            while (!queue.isEmpty()) {
                int[] cur = queue.poll();
                int r = cur[0], c = cur[1], blocks = cur[2];

                for (int k = 0; k < 4; k++) {
                    int nr = r + dR[k];
                    int nc = c + dC[k];

                    if (nr < 0 || nr >= rows || nc < 0 || nc >= cols) continue;
                    if (visited[nr][nc]) continue;

                    char ch = grid[nr][nc];

                    if (ch == 'D') {
                        // FIX: number of mode blocks is exactly `blocks`
                        foundBlocks = blocks;
                        queue.clear(); // we already found the shortest path
                        break;
                    }

                    if (ch == modeChar) {
                        visited[nr][nc] = true;
                        queue.offer(new int[]{nr, nc, blocks + 1});
                    }
                }

                if (foundBlocks != -1) break;
            }

            if (foundBlocks != -1) {
                long time = (long) foundBlocks * times[i];
                long cost = (long) foundBlocks * costs[i];

                if (time < bestTime || (time == bestTime && cost < bestCost)) {
                    bestTime = time;
                    bestCost = cost;
                    bestIdx = i;
                }
            }
        }

        return bestIdx == -1 ? "" : modes[bestIdx];
    }

    // Simple test harness
    public static void main(String[] args) {
        SolutionPart2_BFSPerMode solver = new SolutionPart2_BFSPerMode();

        // Example 1
        char[][] grid1 = {
            {'3', '3', 'S', '2', 'X', 'X'},
            {'3', '1', '1', '2', 'X', '2'},
            {'3', '1', '1', '2', '2', '2'},
            {'3', '1', '1', '1', 'D', '3'},
            {'3', '3', '3', '3', '3', '4'},
            {'4', '4', '4', '4', '4', '4'}
        };
        String[] modes = {"Walk", "Bike", "Car", "Train"};
        int[] costs = {0, 1, 3, 2};
        int[] times = {3, 2, 1, 1};
        System.out.println("Part2 Example1: " + solver.findOptimalCommute(grid1, modes, costs, times)); // Expected: Bike

        // Example 2
        char[][] grid2 = {
            {'S', '1', '1'},
            {'2', '2', '2'},
            {'D', '1', '1'}
        };
        System.out.println("Part2 Example2: " + solver.findOptimalCommute(grid2, modes, costs, times)); // Expected: Bike

        // Example 3
        char[][] grid3 = {
            {'S', '1', 'X'},
            {'X', '3', '3'},
            {'2', '2', 'D'}
        };
        String[] modes3 = {"Walk", "Bike", "Car"};
        int[] costs3 = {0, 1, 3};
        int[] times3 = {3, 2, 1};
        System.out.println("Part2 Example3: " + solver.findOptimalCommute(grid3, modes3, costs3, times3)); // Expected: ""
    }
}
```

---

## 5. Part 3 — 单次 BFS，状态中带上 mode

### 设计动机

当面试官 insist：“最好只 BFS 一次，而不是跑 k 遍”的时候，你可以：

1. 先强调：**时间复杂度本质上仍然是 `O(kRC)`**，因为状态空间多了一个 mode 维度；
2. 再展示这个“单 BFS，多模式”的实现：
   - 状态为 `(r, c, modeIndex, blocks)`；
   - BFS 队列里同时包含多种 mode 的状态；
   - 访问标记 `visited[modeIndex][r][c]`。

### 核心规则

1. 从 `S` 出发，看四个方向邻居：
   - 如果是 digit `ch`，对应 mode m：放入 `(nr, nc, m, 1)`。  
   - 如果是 `D`：说明可以 0 成本到达 `D`，视为候选答案。  
2. BFS 时：
   - 如果当前 `(r, c)` 就是 `D`，用 `blocks * times[mode]` / `blocks * costs[mode]` 更新答案。  
   - 扩展 4 邻居：
     - 若邻居是 `D`：只更新答案，`blocks` 不增加。  
     - 若邻居是相同的 digit：入队 `(nr, nc, mode, blocks + 1)`。  
     - 其它情况不走。

### 代码实现（Java）

```java
import java.util.*;

public class SolutionPart3_SingleBFSWithMode {

    static class State {
        int r, c;
        int mode;    // index in modes[]
        int blocks;  // how many blocks of this mode we have traversed

        State(int r, int c, int mode, int blocks) {
            this.r = r;
            this.c = c;
            this.mode = mode;
            this.blocks = blocks;
        }
    }

    public String findOptimalCommute(char[][] grid, String[] modes, int[] costs, int[] times) {
        int rows = grid.length;
        int cols = grid[0].length;

        int sr = -1, sc = -1;
        int dr = -1, dc = -1;

        // Locate S and D
        for (int r = 0; r < rows; r++) {
            for (int c = 0; c < cols; c++) {
                if (grid[r][c] == 'S') {
                    sr = r; sc = c;
                } else if (grid[r][c] == 'D') {
                    dr = r; dc = c;
                }
            }
        }

        if (sr == -1 || dr == -1) return "";

        long bestTime = Long.MAX_VALUE;
        long bestCost = Long.MAX_VALUE;
        int bestMode = -1;

        int[] dR = {1, -1, 0, 0};
        int[] dC = {0, 0, 1, -1};

        // visited[mode][r][c]
        boolean[][][] visited = new boolean[modes.length][rows][cols];
        Queue<State> q = new ArrayDeque<>();

        // 1) From S, select initial mode by stepping onto a digit cell
        for (int k = 0; k < 4; k++) {
            int nr = sr + dR[k];
            int nc = sc + dC[k];
            if (nr < 0 || nr >= rows || nc < 0 || nc >= cols) continue;

            char ch = grid[nr][nc];
            if (ch == 'X') continue;

            if (ch >= '1' && ch <= '9') {
                int modeIndex = ch - '1';
                if (modeIndex < 0 || modeIndex >= modes.length) continue;

                if (!visited[modeIndex][nr][nc]) {
                    visited[modeIndex][nr][nc] = true;
                    q.offer(new State(nr, nc, modeIndex, 1)); // first block
                }
            } else if (ch == 'D') {
                // S adjacent to D: 0 blocks traversed
                long time = 0L, cost = 0L;
                if (time < bestTime || (time == bestTime && cost < bestCost)) {
                    bestTime = time;
                    bestCost = cost;
                    bestMode = 0; // arbitrary
                }
            }
        }

        // 2) BFS on (r, c, mode)
        while (!q.isEmpty()) {
            State cur = q.poll();
            int r = cur.r, c = cur.c, m = cur.mode, blocks = cur.blocks;

            // If we reached D, update answer
            if (r == dr && c == dc) {
                long time = (long) blocks * times[m];
                long cost = (long) blocks * costs[m];
                if (time < bestTime || (time == bestTime && cost < bestCost)) {
                    bestTime = time;
                    bestCost = cost;
                    bestMode = m;
                }
                // Do not expand from D
                continue;
            }

            for (int i = 0; i < 4; i++) {
                int nr = r + dR[i];
                int nc = c + dC[i];
                if (nr < 0 || nr >= rows || nc < 0 || nc >= cols) continue;

                char ch = grid[nr][nc];
                if (ch == 'X') continue;

                if (ch == 'D') {
                    // Reaching D, blocks do not increase
                    long time = (long) blocks * times[m];
                    long cost = (long) blocks * costs[m];
                    if (time < bestTime || (time == bestTime && cost < bestCost)) {
                        bestTime = time;
                        bestCost = cost;
                        bestMode = m;
                    }
                    // No need to enqueue D
                } else if (ch == (char)('1' + m)) {
                    // Same mode segment
                    if (!visited[m][nr][nc]) {
                        visited[m][nr][nc] = true;
                        q.offer(new State(nr, nc, m, blocks + 1));
                    }
                }
                // Different digits not allowed; cannot switch mode
            }
        }

        return bestMode == -1 ? "" : modes[bestMode];
    }

    // Simple test harness
    public static void main(String[] args) {
        SolutionPart3_SingleBFSWithMode solver = new SolutionPart3_SingleBFSWithMode();

        // Example 1
        char[][] grid1 = {
            {'3', '3', 'S', '2', 'X', 'X'},
            {'3', '1', '1', '2', 'X', '2'},
            {'3', '1', '1', '2', '2', '2'},
            {'3', '1', '1', '1', 'D', '3'},
            {'3', '3', '3', '3', '3', '4'},
            {'4', '4', '4', '4', '4', '4'}
        };
        String[] modes = {"Walk", "Bike", "Car", "Train"};
        int[] costs = {0, 1, 3, 2};
        int[] times = {3, 2, 1, 1};
        System.out.println("Part3 Example1: " + solver.findOptimalCommute(grid1, modes, costs, times)); // Expected: Bike

        // Example 2
        char[][] grid2 = {
            {'S', '1', '1'},
            {'2', '2', '2'},
            {'D', '1', '1'}
        };
        System.out.println("Part3 Example2: " + solver.findOptimalCommute(grid2, modes, costs, times)); // Expected: Bike

        // Example 3
        char[][] grid3 = {
            {'S', '1', 'X'},
            {'X', '3', '3'},
            {'2', '2', 'D'}
        };
        String[] modes3 = {"Walk", "Bike", "Car"};
        int[] costs3 = {0, 1, 3};
        int[] times3 = {3, 2, 1};
        System.out.println("Part3 Example3: " + solver.findOptimalCommute(grid3, modes3, costs3, times3)); // Expected: ""
    }
}
```

---

## 6. 面试时的讲解顺序建议

1. **先说清业务 &建模**：
   - grid 是有障碍的地图；
   - mode 决定可以走哪一条 digit 路；
   - time / cost 只跟“经过多少块该 mode 的道路格子”有关。

2. **优先写你最熟的 Dijkstra（Part 1）**：
   - 正确性、细节都能掌控；
   - 写完跑一下例题。

3. **主动提 Part 2 的优化**：
   - “因为在固定 mode 下每一步权重相同，我可以对每个 mode 做单独 BFS，复杂度 O(kRC)，k ≤ 10。”

4. **如果面试官继续 push “能不能只 BFS 一次”**：
   - 拿出 Part 3 的想法：状态里带上 mode，单队列 BFS；
   - 顺带强调：复杂度本质还是 O(kRC)，只是实现形式不同。

> 这样你既展示出：  
> - 能写出一个 **完整可跑** 的解法（Part 1 / Part 2）；  
> - 又能和面试官聊复杂度优化 & 状态建模（Part 3）；  
> - 同时逻辑清晰、可讲性强。  
> 这套 MD 可以直接贴进你面试准备 repo 里复习使用。
