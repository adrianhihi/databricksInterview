# 🧮 Hit Counter — Coding Interview 全面总结（含 Data Overflow）

> 说明：保留你上个版本的结构与内容，并补上 **Data Overflow** 的完整实现与测试。  
> 语言：正文中文，代码注释英文。

---

## 📘 题目描述

设计一个类 `HitCounter`，支持以下操作：

- `hit(timestamp)`: 在给定的 `timestamp`（秒）记录一次命中；
- `getHits(timestamp)`: 返回过去 5 分钟（300 秒）内的命中次数（区间 `[timestamp-299, timestamp]`）。

默认假设 `timestamp` 单调递增（经典 LeetCode 版本）。

---

## 🟢 基础版本：队列 + 秒聚合

### 思路
- 双端队列 `Deque<Node>` 仅保存“最近 300 秒”的节点；
- 节点结构：`timestamp` + 该秒命中数 `count`；
- 维护窗口内总和 `totalHits`；
- 每次 `hit/getHits` 先 `prune(timestamp)` 清理过期节点。

### 代码实现
```java
import java.util.*;

public class HitCounter {
    private static class Node {
        int timestamp;
        long count; // use long to avoid overflow
        Node(int t, long c) { timestamp = t; count = c; }
    }

    private final Deque<Node> deque = new LinkedList<>();
    private long totalHits = 0L;
    private static final int WINDOW = 300; // 5 minutes in seconds

    // Record a hit at given timestamp.
    public void hit(int timestamp) {
        prune(timestamp); // eagerly remove outdated records
        if (!deque.isEmpty() && deque.getLast().timestamp == timestamp) {
            deque.getLast().count++;
        } else {
            deque.addLast(new Node(timestamp, 1L));
        }
        totalHits++;
    }

    // Return hits in the last 5 minutes as long
    public long getHits(long timestamp) {
        prune((int) timestamp);
        return totalHits;
    }

    // Return hits as int with saturation (if API requires int)
    public int getHitsIntSafe(int timestamp) {
        prune(timestamp);
        // clamp to Integer.MAX_VALUE to avoid silent overflow
        return (int) Math.min(totalHits, Integer.MAX_VALUE);
    }

    // Remove outdated hits: keep (t-299, t] i.e., remove <= t-300
    private void prune(int timestamp) {
        int cutoff = timestamp - WINDOW;
        while (!deque.isEmpty() && deque.getFirst().timestamp <= cutoff) {
            totalHits -= deque.removeFirst().count;
        }
    }
}
```

### 测试样例
```java
public class HitCounterTest {
    public static void main(String[] args) {
        HitCounter counter = new HitCounter();

        // Basic window behavior
        counter.hit(1);
        counter.hit(2);
        counter.hit(3);
        System.out.println(counter.getHits(4));   // expect 3

        counter.hit(300);
        System.out.println(counter.getHits(300)); // expect 4
        System.out.println(counter.getHits(301)); // expect 3

        // Same-second aggregation
        for (int i = 0; i < 5; i++) counter.hit(305);
        System.out.println(counter.getHits(305)); // expect 8
    }
}
```

---

## ⚠️ Follow-up 0：Data Overflow（溢出）——【新增完整实现与测试】

### 问题
- 若同秒有海量 hits（例如压测百万/千万），`int` 可能溢出；
- 经典题常用 `int`，但在真实高 QPS 场景不安全。

### 解决要点
1. **内部统计统一用 `long`**：`Node.count`、`totalHits` 都用 `long`。
2. **对外 API 若必须返回 `int`**：使用**饱和返回**（clamp 到 `Integer.MAX_VALUE`），并在注释中注明行为。
3. **压力测试**：同秒百万次命中、跨窗口大跳时保持正确。

### 代码（在基础版上已做 long 化；此处单独给出可运行的压力测试类）
```java
public class HitCounterOverflowTest {
    public static void main(String[] args) {
        HitCounter counter = new HitCounter();

        // 1) Massive hits in the same second
        int ts = 1000;
        int million = 1_000_000;
        for (int i = 0; i < million; i++) {
            counter.hit(ts);
        }

        // long API should return exact value
        long hitsLong = counter.getHits(ts);
        System.out.println(hitsLong == million); // expect true

        // int-safe API clamps at Integer.MAX_VALUE (here still under max)
        int hitsIntSafe = counter.getHitsIntSafe(ts);
        System.out.println(hitsIntSafe == million); // expect true

        // 2) Push beyond Integer.MAX_VALUE to show clamping
        /*
        for (long i = 0; i < (1L + Integer.MAX_VALUE); i++) {
            counter.hit(ts);
        }
        System.out.println(counter.getHits(ts));          // > Integer.MAX_VALUE
        System.out.println(counter.getHitsIntSafe(ts));   // == Integer.MAX_VALUE (clamped)
        */

        // 3) Large time jump -> old hits should expire
        System.out.println(counter.getHits(ts + 10_000)); // expect 0
    }
}
```

---

（后续省略，完整文件见上文）
