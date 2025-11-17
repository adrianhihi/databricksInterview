
1. Basic: https://app.coderpad.io/ZH936P9A
2. Search Single IP: https://app.coderpad.io/6ENWJ3CH
3. Ranged CIDR: https://app.coderpad.io/P42DEWXG




# IP to CIDR：题意 + 快速规则 + 示例详解（为何 /28 不行、/29 和 /32 可以；/31 /30 为什么不合适）

> 说明用中文，便于面试时快速复述。

---

## 1) 题目在说什么？

- **IPv4** 是一个 32 位无符号整数，按 8 位一组打印成十进制，并用点号分隔。  
  例如：`00001111 10001000 11111111 01101011` → `15.136.255.107`。

- **CIDR 块** 的写法是：`<baseIP>/<k>`。它代表“所有**前 k 位**与 `baseIP` 完全相同”的地址集合。  
  例如：`123.45.67.89/20` 覆盖所有二进制形如 `01111011 00101101 0100xxxx xxxxxxxx` 的地址（`x` 为任意位）。

- 给你一个起始 IP `ip` 和数量 `n`，要用**尽量少**的 CIDR 块**刚好**覆盖闭区间 `[ip, ip+n-1]`，**不能**覆盖区间外的地址。

**核心约束两条：**
1. **块数最少**（每步贪心选尽量大的块）；  
2. **不能多盖**（每个选出来的 CIDR 必须完全落在目标区间里）。

---

## 2) 快速规则（很重要）

- 前缀长度 `/k` 的**块大小** = `2^(32 - k)` 个地址。  
- CIDR **必须按块大小对齐**：baseIP（看作无符号 32 位整数）必须是该块大小的倍数。

直观例子：
- `/32` 覆盖 1 个地址，随便从哪儿都能起；
- `/29` 覆盖 8 个地址，base 的最后一段要是 **8 的倍数**（…0、8、16、24）；
- `/28` 覆盖 16 个地址，base 的最后一段要是 **16 的倍数**（…0、16、32…）。

---

## 3) 示例 1：为什么 `/28` 不行、`/29` 和 `/32` 可以？

**输入：** `ip = "255.0.0.7"`, `n = 10` → 目标区间：`.7 ~ .16`（共 10 个）。

### 看 `/28`
- `/28` 的块大小是 **16**，必须从 **16 的倍数**开始：只能 `...0` 或 `...16`。
  - `255.0.0.0/28` → 覆盖 `.0 ~ .15`：**多盖** `.0~.6` 且 **漏** `.16`；  
  - `255.0.0.16/28` → 覆盖 `.16 ~ .31`：**多盖** `.17~.31`；  
  ⇒ **/28 不行**。

### 看 `/29`
- `/29` 的块大小是 **8**，必须从 **8 的倍数**开始：`255.0.0.8/29` → 覆盖 `.8 ~ .15`，**刚好**在目标区间内。

### 看 `/32`
- `/32` 的块大小是 **1**，无对齐限制：  
  - `255.0.0.7/32` 覆盖 `.7`；  
  - `255.0.0.16/32` 覆盖 `.16`。

**最优三块解：**
```text
["255.0.0.7/32","255.0.0.8/29","255.0.0.16/32"]
```
块数最少（3）且不越界覆盖。

> 对比更大块的尝试：
> - `/31`（2 个地址）从 `.16` 起会覆盖 `.16~.17`，把 `.17` 也带上了；  
> - `/30`（4 个地址）想覆盖 `.8~.15` 需要两块（`.8~.11` 和 `.12~.15`），块数反而更多。

---

## 4) 小结：拿到题就这么做（贪心步骤）

1. 把起点 `ip` 转成无符号 32 位整数 `start`；  
2. 循环直到覆盖完 `n`：
   - 计算**对齐能力**：`block ≤ 2^{trailingZeros(start)}`（`start==0` 视为 32）；  
   - 计算**剩余能力**：`block ≤ largestPowerOfTwo ≤ n`；  
   - 取两者**最小**作为 `block`，输出 `start/mask`，其中 `mask = 32 - log2(block)`；  
   - `start += block`，`n -= block`。

示例 1 会得到：`/32`（先吃 `.7`）→ `/29`（吃 `.8~.15`）→ `/32`（吃 `.16`）。

---

## 5) 为什么 `/31`、`/30` 不可以（或不优）？

**目标区间：** `255.0.0.7 ~ 255.0.0.16`（10 个）

- 规则回顾：
  - `/31` 大小 2，要从**偶数**开始：…0、2、4、6、8、10、12、14、16…  
  - `/30` 大小 4，要从**4 的倍数**开始：…0、4、8、12、16…  
  - `/29` 大小 8，要从**8 的倍数**开始：…0、8、16…

**覆盖 `.7`：**
- `/31` 若从 `.6` 起→ `.6~.7`，**多盖** `.6`；从 `.7` 起不对齐（非法）；  
- `/30` 只能从 `.4` 或 `.8` 起：`.4~.7` 会**多盖** `.4~.6`；`.8~.11` 又**不含** `.7`；  
⇒ `.7` 只能用 `/32`。

**覆盖 `.16`：**
- `/31` 从 `.16` 起→ `.16~.17`，**多盖** `.17`；  
- `/30` 从 `.16` 起→ `.16~.19`，**多盖** `.17~.19`；  
⇒ `.16` 也只能用 `/32`。

**覆盖中段 `.8~.15`：**
- `/29` 一块正好 `.8~.15`，且对齐；  
- `/30` 需要两块：`.8~.11` + `.12~.15`；  
- `/31` 需要四块：`.8~.9`、`.10~.11`、`.12~.13`、`.14~.15`；  
⇒ 块数比 `/29` 多。

**综合：**
- 用 `/31` 或 `/30` 覆盖边界会**溢出**（越界）；
- 拆中段会**增加块数**；
- 因此最优解是：`/32（.7） + /29（.8~.15） + /32（.16）`，共 **3 块**。

---

## 6) 一句话口诀

> **“对齐 + 不越界，取两者最小；边界不整齐，用 `/32` 点名；中段尽量大块（如 `/29`）。”**




## 3) 解题思路（贪心 + 位运算）与正确性

每步在当前 `start` 处选择**最大**的 2 的幂大小的块 `block`，同时满足：
1) **对齐限制**：`block ≤ 2^{tz(start)}`（`tz` 为尾随 0 个数；`start==0` 视作 32）；  
2) **剩余限制**：`block ≤ highestOneBit(n)`；  
取两者最小，输出 `/mask`，其中 `mask = 32 - log2(block)`（用 trailingZeros 取代浮点）。  
**正确性**：二进制分块的标准贪心，保证块数最少。


## 4) Java 实现：`IpToCidrBasic`（`int n`）

```java
import java.util.*;

public class IpToCidrBasic {
    public List<String> ipToCIDR(String ip, int n) {
        if (n <= 0) return Collections.emptyList();

        long start = ipToLong(ip);
        List<String> res = new ArrayList<>();

        while (n > 0) {
            // alignment capacity by trailing zeros (start==0 -> fully aligned)
            int alignZeros = (start == 0L) ? 32 : Math.min(32, Long.numberOfTrailingZeros(start));
            long alignBlock = 1L << alignZeros;      // up to 2^32
            long capByN = Long.highestOneBit(n);     // largest power-of-two ≤ n
            long blockSize = Math.min(alignBlock, capByN);

            int prefixLen = 32 - Long.numberOfTrailingZeros(blockSize);
            res.add(longToIp(start) + "/" + prefixLen);

            start += blockSize;
            n -= blockSize; // safe: blockSize ≤ n
        }
        return res;
    }

    // Convert dotted IPv4 string to unsigned 32-bit number in a long
    public static long ipToLong(String ip) {
        String[] parts = ip.split("\\.");
        if (parts.length != 4) {
            throw new IllegalArgumentException("Invalid IPv4: " + ip);
        }
        long num = 0L;
        for (String p : parts) {
            int v = Integer.parseInt(p.trim());
            if (v < 0 || v > 255) throw new IllegalArgumentException("IPv4 segment out of range: " + p);
            num = (num << 8) + v;
        }
        return num; // 0..2^32-1
    }

    // Convert unsigned 32-bit number back to dotted IPv4
    public static String longToIp(long x) {
        int b1 = (int)((x >>> 24) & 0xFF);
        int b2 = (int)((x >>> 16) & 0xFF);
        int b3 = (int)((x >>> 8)  & 0xFF);
        int b4 = (int)( x         & 0xFF);
        return b1 + "." + b2 + "." + b3 + "." + b4;
    }
}
```


## 5) Java 实现：`IpToCidrSafe`（`long n`，更稳健）

```java
import java.util.*;

public class IpToCidrSafe {
    public List<String> ipToCIDR(String ip, long n) {
        if (n <= 0) return Collections.emptyList();

        long start = IpToCidrBasic.ipToLong(ip);
        List<String> res = new ArrayList<>();

        while (n > 0) {
            int alignZeros = (start == 0L) ? 32 : Math.min(32, Long.numberOfTrailingZeros(start));
            long alignBlock = 1L << alignZeros;       // up to 2^32
            long capByN = Long.highestOneBit(n);      // largest power-of-two ≤ n
            long blockSize = Math.min(alignBlock, capByN);

            int prefixLen = 32 - Long.numberOfTrailingZeros(blockSize);
            res.add(IpToCidrBasic.longToIp(start) + "/" + prefixLen);

            start += blockSize;
            n -= blockSize;
        }
        return res;
    }
}
```


## 6) 常见坑与边界

- **`start==0` 死循环/TLE**：不能直接 `lowbit = x & -x`，`x=0` 得到 0。做法：把 `start==0` 视作完全对齐（32）。
- **避免浮点 `Math.log`**：使用 `Long.numberOfTrailingZeros(blockSize)` 取 `log2`。
- **位运算优先级**：统一加括号，例如 `(ip & mask) == net`、`a & (b - 1)`。
- **输入校验**：段数必须 4、每段 `0..255`。


## 7) CIDR 判定工具

```java
// Return mask whose top k bits are 1, as an unsigned 32-bit long
public static long maskOf(int k) {
    if (k <= 0) return 0L;
    if (k >= 32) return 0xFFFFFFFFL;
    long m = -1L << (32 - k);          // high k bits are 1
    return m & 0xFFFFFFFFL;            // keep low 32 bits
}

// Convert "a.b.c.d/k" into [start,end] range (unsigned)
public static long[] cidrRange(String cidr) {
    String[] parts = cidr.split("/");
    long base = IpToCidrBasic.ipToLong(parts[0]) & 0xFFFFFFFFL;
    int k = Integer.parseInt(parts[1]);
    long size = (k == 0) ? (1L << 32) : (1L << (32 - k));
    long start = base & maskOf(k);
    long end = start + size - 1;
    return new long[]{ start, end };
}

// Whether CIDR A contains CIDR B
public static boolean cidrContains(String a, String b) {
    String[] pa = a.split("/");
    String[] pb = b.split("/");
    long baseA = IpToCidrBasic.ipToLong(pa[0]) & 0xFFFFFFFFL;
    long baseB = IpToCidrBasic.ipToLong(pb[0]) & 0xFFFFFFFFL;
    int kA = Integer.parseInt(pa[1]);
    int kB = Integer.parseInt(pb[1]);
    if (kA > kB) return false;
    long mA = maskOf(kA);
    return (baseA & mA) == (baseB & mA);
}

// Whether two CIDRs overlap
public static boolean cidrOverlap(String a, String b) {
    long[] A = cidrRange(a);
    long[] B = cidrRange(b);
    return Math.max(A[0], B[0]) <= Math.min(A[1], B[1]);
}
```


## 8) RuleEngine 接口与两套实现

### 接口定义（统一 API）

```java
public interface RuleEngine {
    // Add a rule in CIDR form, e.g., "10.0.0.0/8" with ALLOW or DENY
    void addRule(String cidr, Action action);

    // Match a single IP and return the effective action (ALLOW/DENY/NONE)
    Action matchIp(String ip);

    // Match a CIDR: ALLOW if fully allowed, DENY if fully denied, or NONE if mixed/partial/undefined.
    Action matchCidr(String cidr);
}
```

### 8.1 Trie（LPM，最后写入生效）

- 适用：按**最长前缀匹配**（路由/ACL常用）。
- `matchIp`：沿 32 位向下，记录“路径上最近设置的动作”。
- `matchCidr`：检查该前缀子树是否**动作一致**（用继承语义 DFS）。

```java
enum Action { ALLOW, DENY, NONE }

class RuleEngineTrie implements RuleEngine {
    static class Node {
        Node zero, one;
        Action action = Action.NONE;  // action set exactly at this node
        int order = -1;               // last-write-wins counter
    }
    private final Node root = new Node();
    private int counter = 0;

    @Override
    public void addRule(String cidr, Action action) {
        String[] ps = cidr.split("/");
        long base = IpToCidrBasic.ipToLong(ps[0]) & 0xFFFFFFFFL;
        int k = Integer.parseInt(ps[1]);
        Node cur = root;
        for (int i = 31; i >= 32 - k; i--) {
            int bit = (int)((base >>> i) & 1);
            cur = (bit == 0) ? (cur.zero = (cur.zero == null ? new Node() : cur.zero))
                             : (cur.one  = (cur.one  == null ? new Node() : cur.one));
        }
        if (cur.order <= counter) { // last write wins for the same prefix
            cur.action = action;
            cur.order = ++counter;
        }
    }

    @Override
    public Action matchIp(String ip) {
        long x = IpToCidrBasic.ipToLong(ip) & 0xFFFFFFFFL;
        Node cur = root;
        Action best = Action.NONE;
        int bestOrder = -1;
        for (int i = 31; i >= 0; i--) {
            if (cur == null) break;
            if (cur.action != Action.NONE && cur.order >= bestOrder) {
                best = cur.action; bestOrder = cur.order;
            }
            int bit = (int)((x >>> i) & 1);
            cur = (bit == 0) ? cur.zero : cur.one;
        }
        if (cur != null && cur.action != Action.NONE && cur.order >= bestOrder) {
            best = cur.action;
        }
        return best;
    }

    @Override
    public Action matchCidr(String cidr) {
        String[] ps = cidr.split("/");
        long base = IpToCidrBasic.ipToLong(ps[0]) & 0xFFFFFFFFL;
        int k = Integer.parseInt(ps[1]);
        Node cur = root;
        for (int i = 31; i >= 32 - k; i--) {
            if (cur == null) break;
            int bit = (int)((base >>> i) & 1);
            cur = (bit == 0) ? cur.zero : cur.one;
        }
        java.util.Set<Action> seen = new java.util.HashSet<>();
        dfsUniform(cur, inheritedAction(base, k), seen);
        if (seen.size() == 1) return seen.iterator().next();
        return Action.NONE;
    }

    private Action inheritedAction(long base, int k) {
        Node cur = root;
        Action best = Action.NONE;
        int bestOrder = -1;
        for (int i = 31; i >= 32 - k; i--) {
            if (cur == null) break;
            if (cur.action != Action.NONE && cur.order >= bestOrder) {
                best = cur.action; bestOrder = cur.order;
            }
            int bit = (int)((base >>> i) & 1);
            cur = (bit == 0) ? cur.zero : cur.one;
        }
        return best;
    }

    private void dfsUniform(Node node, Action inherit, java.util.Set<Action> seen) {
        if (node == null) { seen.add(inherit); return; }
        Action here = (node.action != Action.NONE) ? node.action : inherit;
        if (node.zero == null && node.one == null) { seen.add(here); return; }
        dfsUniform(node.zero, here, seen);
        if (seen.size() > 1) return;
        dfsUniform(node.one, here, seen);
    }
}
```

### 8.2 RangeSet（并集，deny 优先）

- 适用：**整段覆盖**/批量授权判断；`O(log M)` 插入/查询。

```java
class RangeSet {
    private final java.util.TreeMap<Long, Long> m = new java.util.TreeMap<>(); // start -> end
    public void add(long s, long e) {
        if (s > e) return;
        var prev = m.floorEntry(s);
        if (prev != null && prev.getValue() + 1 >= s) {
            s = Math.min(s, prev.getKey());
            e = Math.max(e, prev.getValue());
            m.remove(prev.getKey());
        }
        var cur = m.ceilingEntry(s);
        while (cur != null && cur.getKey() <= e + 1) {
            e = Math.max(e, cur.getValue());
            m.remove(cur.getKey());
            cur = m.ceilingEntry(s);
        }
        m.put(s, e);
    }
    public boolean contains(long x) {
        var prev = m.floorEntry(x);
        return prev != null && prev.getValue() >= x;
    }
    public boolean covers(long s, long e) {
        var prev = m.floorEntry(s);
        return prev != null && prev.getValue() >= e;
    }
    public boolean overlaps(long s, long e) {
        var prev = m.floorEntry(e);
        if (prev != null && prev.getValue() >= s) return true;
        var next = m.ceilingEntry(s);
        return next != null && next.getKey() <= e;
    }
}

class RuleEngineRangeSet implements RuleEngine {
    private final RangeSet allow = new RangeSet();
    private final RangeSet deny  = new RangeSet();

    @Override
    public void addRule(String cidr, Action action) {
        long[] rg = cidrRange(cidr);
        if (action == Action.ALLOW) allow.add(rg[0], rg[1]);
        else if (action == Action.DENY) deny.add(rg[0], rg[1]);
    }

    @Override
    public Action matchIp(String ip) {
        long x = IpToCidrBasic.ipToLong(ip) & 0xFFFFFFFFL;
        if (deny.contains(x))  return Action.DENY;
        if (allow.contains(x)) return Action.ALLOW;
        return Action.NONE;
    }

    @Override
    public Action matchCidr(String cidr) {
        long[] rg = cidrRange(cidr);
        if (deny.overlaps(rg[0], rg[1])) return Action.DENY;
        if (allow.covers(rg[0], rg[1]))  return Action.ALLOW;
        return Action.NONE;
    }
}
```


## 9) 单测与快速验证（可直接运行示例）

```java
// --- Ip to CIDR tests ---
System.out.println(new IpToCidrBasic().ipToCIDR("255.0.0.7", 10));
System.out.println(new IpToCidrBasic().ipToCIDR("0.0.0.0", 1));
System.out.println(new IpToCidrBasic().ipToCIDR("10.0.0.0", 256));
System.out.println(new IpToCidrBasic().ipToCIDR("117.145.102.62", 8));

// --- CIDR helpers ---
long[] rg = cidrRange("123.45.67.89/20");
System.out.printf("[start=%s, end=%s]%n", IpToCidrBasic.longToIp(rg[0]), IpToCidrBasic.longToIp(rg[1]));
System.out.println("contains? " + cidrContains("10.0.0.0/8", "10.1.2.0/24"));
System.out.println("overlap?  " + cidrOverlap("10.0.0.0/8", "11.0.0.0/8"));

// --- RuleEngine (Trie) ---
RuleEngine trie = new RuleEngineTrie();
trie.addRule("10.0.0.0/8", Action.ALLOW);
trie.addRule("10.1.0.0/16", Action.DENY);
trie.addRule("10.1.2.0/24", Action.ALLOW);
System.out.println("Trie 10.2.3.4 -> " + trie.matchIp("10.2.3.4"));      // ALLOW
System.out.println("Trie 10.1.3.4 -> " + trie.matchIp("10.1.3.4"));      // DENY
System.out.println("Trie 10.1.2.5 -> " + trie.matchIp("10.1.2.5"));      // ALLOW
System.out.println("Trie CIDR 10.2.0.0/16 -> " + trie.matchCidr("10.2.0.0/16"));

// --- RuleEngine (RangeSet) ---
RuleEngine rs = new RuleEngineRangeSet();
rs.addRule("10.0.0.0/8", Action.ALLOW);
rs.addRule("10.1.0.0/16", Action.DENY);
System.out.println("RS 10.2.3.4 -> " + rs.matchIp("10.2.3.4"));          // ALLOW
System.out.println("RS 10.1.3.4 -> " + rs.matchIp("10.1.3.4"));          // DENY
System.out.println("RS 10.1.2.0/24 -> " + rs.matchCidr("10.1.2.0/24"));  // DENY
System.out.println("RS 10.2.0.0/16 -> " + rs.matchCidr("10.2.0.0/16"));  // ALLOW
```

> 如需 Maven/JUnit5 工程与完整单测，可下载我提供的压缩包。

---

## 10) 面试速记（电梯版）
- **贪心**：每步块大小取 `min(2^{tz(start)}, highestPow2(n))`，O(k)。  
- **示例 1**：`/28` 要 16 对齐，越界；`/29` 吃 `.8~.15`，两端用 `/32`。  
- **CIDR 判定**：`cidrContains`（O(1) 位运算）、`cidrOverlap`（区间法）。  
- **RuleEngine**：
  - **Trie**（LPM）：路由/ACL常用；支持“最后写入生效”。
  - **RangeSet**（并集）：整段覆盖判断快；“deny 优先”。
- **位运算优先级坑**：一律加括号（`(ip & mask) == net`，`a & (b-1)`）。
