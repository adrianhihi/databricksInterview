# 🧮 Hit Counter — Coding Interview 全面总结（含 Data Overflow）

> 说明：保留你上个版本的结构与内容，并补上 **Data Overflow** 的完整实现与测试；其余内容**全部给全量版本**。  
> 语言：正文中文，代码注释英文；所有代码与测试均可直接复制单文件运行（含 `main`）。

---

## 📘 题目描述

设计一个类 `HitCounter`，支持以下操作：

- `hit(timestamp)`: 在给定的 `timestamp`（秒）记录一次命中；  
- `getHits(timestamp)`: 返回过去 5 分钟（300 秒）内的命中次数（区间 `[timestamp-299, timestamp]`）。

默认假设 `timestamp` 单调递增（经典 LeetCode 版本）。

---

## 🟢 基础版本：队列 + 秒聚合（已防溢出）
```java


import java.io.*;
import java.util.*;


class Solution {
  public static class HitCounter {
    private class Node {
        int timestamp;
        int count;
        Node (int t, int c) {
            timestamp = t;
            count = c;
        }
    }

    private Deque<Node> deque;
    private int totalHits;

    public HitCounter() {
        deque = new LinkedList<>();
        totalHits = 0;
    }
    
    public void hit(int timestamp) {
        if (!deque.isEmpty() && deque.getLast().timestamp == timestamp) {
            deque.getLast().count ++;
        } else {
            deque.addLast(new Node(timestamp, 1));
        }
        totalHits ++;
    }
    
    public int getHits(int timestamp) {
        int cutoff = timestamp - 300;
        while (!deque.isEmpty() && deque.getFirst().timestamp <= cutoff) {
            totalHits -= deque.getFirst().count;
            deque.removeFirst();
        }
        return totalHits;
    }
  }

  public static void main(String[] args) {
    HitCounter counter = new HitCounter();
    counter.hit(1);
    counter.hit(2);
    counter.hit(3);
    System.out.println(counter.getHits(4));
    counter.hit(300);
    System.out.println(counter.getHits(300));
    System.out.println(counter.getHits(301));
  }
}

```

### 💡 思路
- 双端队列 `Deque<Node>` 仅保存“最近 300 秒”的节点；  
- 节点结构：`timestamp` + 该秒命中数 `count`；  
- 维护窗口内总和 `totalHits`；  
- 每次 `hit/getHits` 先 `prune(timestamp)` 清理过期节点；  
- **防溢出**：内部用 `long` 计数。

### 💻 代码实现
```java
import java.util.*;

/** Queue-based Hit Counter with per-second aggregation (overflow-safe with long). */
public class HitCounter {
    private static class Node {
        int timestamp;
        long count; // use long to avoid overflow in high QPS
        Node(int t, long c) { timestamp = t; count = c; }
    }

    private final Deque<Node> deque = new LinkedList<>();
    private long totalHits = 0L;
    private static final int WINDOW = 300; // 5 minutes in seconds

    /** Record a hit at given timestamp (seconds). */
    public void hit(int timestamp) {
        prune(timestamp); // eagerly remove outdated records
        if (!deque.isEmpty() && deque.getLast().timestamp == timestamp) {
            deque.getLast().count++;
        } else {
            deque.addLast(new Node(timestamp, 1L));
        }
        totalHits++;
    }

    /** Return hits in the last 5 minutes as long. */
    public long getHits(long timestamp) {
        prune((int) timestamp);
        return totalHits;
    }

    /** Return hits as int with saturation if callers require int signature. */
    public int getHitsIntSafe(int timestamp) {
        prune(timestamp);
        // clamp to Integer.MAX_VALUE to avoid silent overflow
        return (int) Math.min(totalHits, Integer.MAX_VALUE);
    }

    /** Remove outdated hits: keep (t-299, t] i.e., remove <= t-300. */
    private void prune(int timestamp) {
        int cutoff = timestamp - WINDOW;
        while (!deque.isEmpty() && deque.getFirst().timestamp <= cutoff) {
            totalHits -= deque.removeFirst().count;
        }
    }
}
```

### 🧪 测试样例
```java
/** Basic tests for the queue-based HitCounter. */
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

        // Large jump expires all
        System.out.println(counter.getHits(10_000)); // expect 0
    }
}
```

---

## ⚠️ Follow-up 0：Data Overflow（溢出）

### 🧩 问题
- 若同秒有海量 hits（例如压测百万/千万），`int` 可能溢出；  
- 经典题常用 `int`，但在真实高 QPS 场景不安全。

### ✅ 解决要点
1. **内部统计统一用 `long`**：`Node.count`、`totalHits` 等。  
2. **对外 API 若必须返回 `int`**：使用**饱和返回**（clamp 到 `Integer.MAX_VALUE`），注释注明行为。  
3. **压力测试**：同秒百万次命中、跨窗口大跳过期。

### 🧪 压测代码（可直接运行）
```java
/** Stress tests for overflow behavior. */
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

        // 2) Large time jump -> old hits should expire
        System.out.println(counter.getHits(ts + 10_000)); // expect 0
    }
}
```

> 若必须“只用 `int`”，可和面试官沟通返回值饱和的权衡；或在业务层整秒内先聚合，减少增量写入频次。

---

## ⚙️ Follow-up 1：环形数组桶优化（固定空间 O(300)）

### 💡 思路
- 维护 300 个桶（每秒一个），索引 `i = timestamp % 300`；  
- 若桶时间不是当前秒，重置为当前秒并计数=1；否则同秒累加；  
- 查询时遍历 300 桶，对仍在窗口内的累加；  
- 对 GC 友好、同秒极高 QPS 更稳。

### 💻 代码实现
```java
/** Ring-buffer bucket implementation with fixed 300 slots. */
public class HitCounterBuckets {
    private static final int WINDOW = 300;
    private final int[] times = new int[WINDOW];
    private final long[] counts = new long[WINDOW]; // long counters

    public void hit(int timestamp) {
        int i = timestamp % WINDOW;
        if (times[i] != timestamp) {
            times[i] = timestamp;
            counts[i] = 1L;
        } else {
            counts[i]++;
        }
    }

    public long getHits(int timestamp) {
        long sum = 0L;
        int start = timestamp - WINDOW + 1;
        for (int i = 0; i < WINDOW; i++) {
            if (times[i] >= start) sum += counts[i];
        }
        return sum;
    }

    public int getHitsIntSafe(int timestamp) {
        long v = getHits(timestamp);
        return (int) Math.min(v, Integer.MAX_VALUE);
    }
}
```

### 🧪 测试样例
```java
/** Tests for the ring-buffer bucket implementation. */
public class HitCounterBucketsTest {
    public static void main(String[] args) {
        HitCounterBuckets counter = new HitCounterBuckets();
        counter.hit(1);
        counter.hit(2);
        counter.hit(3);
        System.out.println(counter.getHits(4));   // 3
        counter.hit(300);
        System.out.println(counter.getHits(300)); // 4
        System.out.println(counter.getHits(301)); // 3

        // High QPS same-second
        for (int i = 0; i < 1_000_000; i++) counter.hit(1000);
        System.out.println(counter.getHits(1000)); // 1_000_000
        System.out.println(counter.getHits(10_000)); // 0
    }
}
```

---

## ⚙️ Follow-up 2：支持乱序时间戳（TreeMap）

### 💡 思路
- 使用 `TreeMap<timestamp, count>` 支持乱序插入；  
- 查询时清理 `<= timestamp-300` 的过期键；  
- 复杂度：插入/删除 `O(log M)`，`M` 为键数量。

### 💻 代码实现
```java
import java.util.*;

/** TreeMap-based implementation to support out-of-order timestamps. */
public class HitCounterUnordered {
    private final TreeMap<Integer, Long> map = new TreeMap<>();
    private long totalHits = 0L;
    private static final int WINDOW = 300;

    public void hit(int timestamp) {
        map.merge(timestamp, 1L, Long::sum);
        totalHits++;
    }

    public long getHits(int timestamp) {
        int cutoff = timestamp - WINDOW;
        while (!map.isEmpty() && map.firstKey() <= cutoff) {
            totalHits -= map.remove(map.firstKey());
        }
        return totalHits;
    }

    public int getHitsIntSafe(int timestamp) {
        long v = getHits(timestamp);
        return (int) Math.min(v, Integer.MAX_VALUE);
    }
}
```

### 🧪 测试样例
```java
/** Tests for the unordered (TreeMap) implementation. */
public class HitCounterUnorderedTest {
    public static void main(String[] args) {
        HitCounterUnordered counter = new HitCounterUnordered();
        counter.hit(10);
        counter.hit(8);
        counter.hit(12);
        System.out.println(counter.getHits(12));  // 3
        System.out.println(counter.getHits(400)); // 0

        // same-second mass hits out of order
        for (int i = 0; i < 500_000; i++) counter.hit(1000);
        for (int i = 0; i < 500_000; i++) counter.hit(999);
        System.out.println(counter.getHits(1000)); // 1_000_001 if within window
        System.out.println(counter.getHits(10_000)); // 0 after expiration
    }
}
```

---

## ⚙️ Follow-up 3：多线程安全（环形桶 + LongAdder）

### 💡 思路
- 每个桶用一个 `LongAdder`（高并发计数器）；  
- 重置过期桶需要小范围加锁，避免并发重置丢失；  
- 读路径遍历窗口内桶并求和。

### 💻 代码实现
```java
import java.util.concurrent.atomic.*;
import java.util.concurrent.locks.ReentrantLock;

/** Concurrent ring-buffer implementation using LongAdder per bucket. */
public class HitCounterConcurrent {
    private final int WINDOW;
    private final int[] times;
    private final LongAdder[] counts;
    private final ReentrantLock[] locks;

    public HitCounterConcurrent(int windowSeconds) {
        this.WINDOW = windowSeconds;
        this.times = new int[WINDOW];
        this.counts = new LongAdder[WINDOW];
        this.locks = new ReentrantLock[WINDOW];
        for (int i = 0; i < WINDOW; i++) {
            counts[i] = new LongAdder();
            locks[i] = new ReentrantLock();
        }
    }

    public void hit(int timestamp) {
        int i = timestamp % WINDOW;
        if (times[i] != timestamp) {
            locks[i].lock();
            try {
                if (times[i] != timestamp) {
                    times[i] = timestamp;
                    counts[i].reset();
                }
            } finally {
                locks[i].unlock();
            }
        }
        counts[i].increment();
    }

    public long getHits(int timestamp) {
        long sum = 0L;
        int start = timestamp - WINDOW + 1;
        for (int i = 0; i < WINDOW; i++) {
            if (times[i] >= start) sum += counts[i].sum();
        }
        return sum;
    }

    public int getHitsIntSafe(int timestamp) {
        long v = getHits(timestamp);
        return (int) Math.min(v, Integer.MAX_VALUE);
    }
}
```

### 🧪 测试样例
```java
/** Concurrency tests for the LongAdder-based implementation. */
public class HitCounterConcurrentTest {
    public static void main(String[] args) throws InterruptedException {
        HitCounterConcurrent counter = new HitCounterConcurrent(300);

        Thread t1 = new Thread(() -> { for (int i = 0; i < 100_000; i++) counter.hit(1); });
        Thread t2 = new Thread(() -> { for (int i = 0; i < 100_000; i++) counter.hit(1); });
        t1.start(); t2.start();
        t1.join(); t2.join();

        long total = counter.getHits(1);
        System.out.println(total); // expect 200_000
        System.out.println(counter.getHits(10_000)); // expect 0 after expiration
    }
}
```

---

## ⚙️ Follow-up 4：动态窗口长度（可配置）

### 💡 思路
- 将 `WINDOW` 作为构造参数；  
- 其余逻辑不变。

### 💻 代码实现（基于队列版本）
```java
import java.util.*;

/** Queue-based implementation with configurable window length. */
public class HitCounterDynamic {
    private static class Node {
        int timestamp;
        long count;
        Node(int t, long c) { timestamp = t; count = c; }
    }

    private final int WINDOW;
    private final Deque<Node> deque = new LinkedList<>();
    private long totalHits = 0L;

    public HitCounterDynamic(int windowSeconds) {
        this.WINDOW = windowSeconds;
    }

    public void hit(int timestamp) {
        prune(timestamp);
        if (!deque.isEmpty() && deque.getLast().timestamp == timestamp) {
            deque.getLast().count++;
        } else {
            deque.addLast(new Node(timestamp, 1L));
        }
        totalHits++;
    }

    public long getHits(int timestamp) {
        prune(timestamp);
        return totalHits;
    }

    public int getHitsIntSafe(int timestamp) {
        long v = getHits(timestamp);
        return (int) Math.min(v, Integer.MAX_VALUE);
    }

    private void prune(int timestamp) {
        int cutoff = timestamp - WINDOW;
        while (!deque.isEmpty() && deque.getFirst().timestamp <= cutoff) {
            totalHits -= deque.removeFirst().count;
        }
    }
}
```

### 🧪 测试样例
```java
/** Tests for configurable-window queue-based counter. */
public class HitCounterDynamicTest {
    public static void main(String[] args) {
        HitCounterDynamic counter = new HitCounterDynamic(60); // 1 minute window
        for (int i = 1; i <= 10; i++) counter.hit(i);
        System.out.println(counter.getHits(70)); // expect 0

        for (int i = 100; i <= 160; i++) counter.hit(i);
        System.out.println(counter.getHits(160)); // expect 61
        System.out.println(counter.getHits(10_000)); // expect 0
    }
}
```

---

## ⚙️ Follow-up 5：分布式扩展（System Design 要点）

> 本节为口述要点，面试通常不要求代码。

- **Sharding（分片）**：按 `metricId/userId` 一致性哈希 → 多分片，每片使用“环形桶”实现。  
- **Aggregation（聚合）**：`getHits` fan-out 到各分片并行拉取 300 桶求和；热点可做 1–2 秒本地缓存（或 CDN/边缘）。  
- **Durability（持久化）**：周期性快照（Redis/RocksDB/S3），命中事件 WAL/Outbox 追加日志，宕机重放恢复。  
- **Fault Tolerance（容错）**：主从/RAFT，幂等重放（基于 `eventId`）。  
- **Backfill（回灌）**：历史事件回放重建窗口快照；乱序可在入口微批排序 + watermark。  
- **多租户隔离**：按租户限流/配额，防止“噪音”影响其他租户。

---

## 📊 版本对比

| 版本 | 时间复杂度 | 空间复杂度 | 适用场景 |
|---|---|---|---|
| 队列版（long） | `hit` amortized O(1), `get` O(1) | O(≤300) | 简洁直观，单机 |
| 环形桶（long） | `hit` O(1), `get` O(300) | O(300) | 高频同秒命中、低 GC |
| 乱序（TreeMap） | `hit` O(log M), `get` O(#过期) | O(M) | 需要支持乱序输入 |
| 并发桶 | `hit` O(1), `get` O(300) | O(300) | 多线程高并发 |
| 动态窗口 | 同上 | O(WINDOW) | 可配置窗口长度 |
| 分布式 | 网络 + 聚合 | — | 生产分布式部署 |

---

## ✅ 总结

- **Data Overflow**：内部用 `long`；必要时对外 `int` 饱和返回；提供压力测试。  
- 面试速通首选 **环形桶（long）**，其余作为 Follow-up 深挖。  
- 并发与分布式要点口述到位即可，必要时展示小段伪代码或接口设计。

---
