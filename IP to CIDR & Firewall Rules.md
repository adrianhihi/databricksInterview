
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

---

# 代码区（每段均含 solution + System.out 打印用例，可单文件运行）

> 说明：出于“每一段 code 自成一体可直接 `javac`+`java` 运行”的要求，以下每个代码块都写成**一个 Java 文件**的形式：把代码复制到对应的 `*.java` 文件名中，再编译运行即可。代码注释为英文、讲解为中文。

---

## A) Basic：最少 CIDR 覆盖（`int n`）

- 文件名：`IpToCidrBasicDemo.java`  
- 运行：`javac IpToCidrBasicDemo.java && java IpToCidrBasicDemo`

```java
import java.util.*;

/** Basic greedy ip->CIDR cover with int n. */
public class IpToCidrBasicDemo {
    // ----- Solution -----
    static class Solver {
        public List<String> ipToCIDR(String ip, int n) {
            if (n <= 0) return Collections.emptyList();
            long start = ipToLong(ip);
            List<String> ans = new ArrayList<>();
            while (n > 0) {
                int alignZeros = (start == 0L) ? 32 : Math.min(32, Long.numberOfTrailingZeros(start));
                long alignBlock = 1L << alignZeros;
                long capByN = Integer.highestOneBit(n);
                long blockSize = Math.min(alignBlock, capByN);
                int prefixLen = 32 - Long.numberOfTrailingZeros(blockSize);
                ans.add(longToIp(start) + "/" + prefixLen);
                start += blockSize;
                n -= blockSize;
            }
            return ans;
        }

        // helpers (self-contained)
        private static long ipToLong(String ip) {
            String[] parts = ip.split("\\.");
            if (parts.length != 4) throw new IllegalArgumentException("Invalid IPv4: " + ip);
            long num = 0L;
            for (String p : parts) {
                int v = Integer.parseInt(p.trim());
                if (v < 0 || v > 255) throw new IllegalArgumentException("IPv4 segment out of range: " + p);
                num = (num << 8) + v;
            }
            return num & 0xFFFFFFFFL;
        }
        private static String longToIp(long x) {
            int b1 = (int)((x >>> 24) & 0xFF);
            int b2 = (int)((x >>> 16) & 0xFF);
            int b3 = (int)((x >>> 8)  & 0xFF);
            int b4 = (int)( x         & 0xFF);
            return b1 + "." + b2 + "." + b3 + "." + b4;
        }
    }

    // ----- System.out tests -----
    private static void expect(String name, List<String> actual, String... expected) {
        List<String> exp = Arrays.asList(expected);
        boolean ok = exp.equals(actual);
        System.out.println(name + " -> actual=" + actual + ", expected=" + exp + " [" + (ok ? "PASS" : "FAIL") + "]");
    }

    public static void main(String[] args) {
        Solver s = new Solver();
        expect("ex1", s.ipToCIDR("255.0.0.7", 10),
                "255.0.0.7/32","255.0.0.8/29","255.0.0.16/32");
        expect("ex2", s.ipToCIDR("0.0.0.0", 1),
                "0.0.0.0/32");
        expect("aligned 256", s.ipToCIDR("10.0.0.0", 256),
                "10.0.0.0/24");
        expect("sample", s.ipToCIDR("117.145.102.62", 8),
                "117.145.102.62/31","117.145.102.64/30","117.145.102.68/31");
    }
}
```

---

## B) Variation：大 n（`long n`）与 0 对齐安全处理

- 文件名：`IpToCidrSafeDemo.java`  
- 运行：`javac IpToCidrSafeDemo.java && java IpToCidrSafeDemo`

```java
import java.util.*;

/** Safer greedy that accepts long n and handles start==0 alignment. */
public class IpToCidrSafeDemo {
    static class Solver {
        public List<String> ipToCIDR(String ip, long n) {
            if (n <= 0) return Collections.emptyList();
            long start = ipToLong(ip);
            List<String> ans = new ArrayList<>();
            while (n > 0) {
                int alignZeros = (start == 0L) ? 32 : Math.min(32, Long.numberOfTrailingZeros(start));
                long alignBlock = 1L << alignZeros;
                long capByN = Long.highestOneBit(n);
                long blockSize = Math.min(alignBlock, capByN);
                int prefixLen = 32 - Long.numberOfTrailingZeros(blockSize);
                ans.add(longToIp(start) + "/" + prefixLen);
                start += blockSize;
                n -= blockSize;
            }
            return ans;
        }
        // helpers (self-contained)
        private static long ipToLong(String ip) {
            String[] parts = ip.split("\\.");
            if (parts.length != 4) throw new IllegalArgumentException("Invalid IPv4: " + ip);
            long num = 0L;
            for (String p : parts) {
                int v = Integer.parseInt(p.trim());
                if (v < 0 || v > 255) throw new IllegalArgumentException("IPv4 segment out of range: " + p);
                num = (num << 8) + v;
            }
            return num & 0xFFFFFFFFL;
        }
        private static String longToIp(long x) {
            int b1 = (int)((x >>> 24) & 0xFF);
            int b2 = (int)((x >>> 16) & 0xFF);
            int b3 = (int)((x >>> 8)  & 0xFF);
            int b4 = (int)( x         & 0xFF);
            return b1 + "." + b2 + "." + b3 + "." + b4;
        }
    }

    private static void print(String name, List<String> val) {
        System.out.println(name + " -> " + val);
    }
    public static void main(String[] args) {
        Solver s = new Solver();
        print("n=1 from 0.0.0.0", s.ipToCIDR("0.0.0.0", 1L));         // [0.0.0.0/32]
        print("big n (300)", s.ipToCIDR("10.0.0.0", 300L));           // [10.0.0.0/24, 10.0.1.0/25, 10.0.1.128/26, ...]
    }
}
```

---

## C) Helpers：CIDR 解析 / 包含 / 重叠 判断

- 文件名：`CidrHelpersDemo.java`  
- 运行：`javac CidrHelpersDemo.java && java CidrHelpersDemo`

```java
/** CIDR helpers: maskOf, cidrRange, cidrContains, cidrOverlap. */
public class CidrHelpersDemo {
    // ----- helpers -----
    static long ipToLong(String ip) {
        String[] parts = ip.split("\\.");
        if (parts.length != 4) throw new IllegalArgumentException("Invalid IPv4: " + ip);
        long num = 0L;
        for (String p : parts) {
            int v = Integer.parseInt(p.trim());
            if (v < 0 || v > 255) throw new IllegalArgumentException("IPv4 segment out of range: " + p);
            num = (num << 8) + v;
        }
        return num & 0xFFFFFFFFL;
    }
    static String longToIp(long x) {
        int b1 = (int)((x >>> 24) & 0xFF);
        int b2 = (int)((x >>> 16) & 0xFF);
        int b3 = (int)((x >>> 8)  & 0xFF);
        int b4 = (int)( x         & 0xFF);
        return b1 + "." + b2 + "." + b3 + "." + b4;
    }
    static long maskOf(int k) {
        if (k <= 0) return 0L;
        if (k >= 32) return 0xFFFFFFFFL;
        long m = -1L << (32 - k);
        return m & 0xFFFFFFFFL;
    }
    static long[] cidrRange(String cidr) {
        String[] ps = cidr.split("/");
        long base = ipToLong(ps[0]) & 0xFFFFFFFFL;
        int k = Integer.parseInt(ps[1]);
        long size = (k == 0) ? (1L << 32) : (1L << (32 - k));
        long start = base & maskOf(k);
        long end = start + size - 1;
        return new long[]{ start, end };
    }
    static boolean cidrContains(String big, String small) {
        String[] pa = big.split("/");
        String[] pb = small.split("/");
        long A = ipToLong(pa[0]) & 0xFFFFFFFFL;
        long B = ipToLong(pb[0]) & 0xFFFFFFFFL;
        int kA = Integer.parseInt(pa[1]);
        int kB = Integer.parseInt(pb[1]);
        if (kA > kB) return false;
        long mA = maskOf(kA);
        return (A & mA) == (B & mA);
    }
    static boolean cidrOverlap(String a, String b) {
        long[] A = cidrRange(a), B = cidrRange(b);
        return Math.max(A[0], B[0]) <= Math.min(A[1], B[1]);
    }

    // ----- System.out tests -----
    static void expect(String name, Object actual, Object expected) {
        System.out.println(name + " -> actual=" + actual + ", expected=" + expected +
                " [" + (String.valueOf(actual).equals(String.valueOf(expected)) ? "PASS" : "FAIL") + "]");
    }
    public static void main(String[] args) {
        long[] r = cidrRange("123.45.67.89/20");
        expect("range.start", longToIp(r[0]), "123.45.64.0");
        expect("range.end",   longToIp(r[1]), "123.45.79.255");
        expect("contains", cidrContains("10.0.0.0/8", "10.1.2.0/24"), true);
        expect("overlap",  cidrOverlap("10.0.0.0/8", "11.0.0.0/8"), false);
    }
}
```

---

## D) Follow-up b1：Ban List（并集 + contains）

- 文件名：`BanListDemo.java`  
- 运行：`javac BanListDemo.java && java BanListDemo`

```java
/** Ban list using RangeSet union; check if a single IP is banned. */
public class BanListDemo {
    // ----- RangeSet (union of closed intervals) -----
    static class RangeSet {
        private final java.util.TreeMap<Long, Long> m = new java.util.TreeMap<>();
        void add(long s, long e) {
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
        boolean contains(long x) {
            var prev = m.floorEntry(x);
            return prev != null && prev.getValue() >= x;
        }
    }
    // ----- BanList -----
    static class BanList {
        private final RangeSet banned = new RangeSet();
        void addBan(String cidr) {
            long[] rg = cidrRange(cidr);
            banned.add(rg[0], rg[1]);
        }
        boolean isBanned(String ip) {
            long x = ipToLong(ip);
            return banned.contains(x);
        }
    }
    // ----- helpers (self-contained) -----
    static long ipToLong(String ip) {
        String[] parts = ip.split("\\.");
        long num = 0L;
        for (String p : parts) num = (num << 8) + Integer.parseInt(p.trim());
        return num & 0xFFFFFFFFL;
    }
    static long maskOf(int k) {
        if (k <= 0) return 0L;
        if (k >= 32) return 0xFFFFFFFFL;
        long m = -1L << (32 - k);
        return m & 0xFFFFFFFFL;
    }
    static long[] cidrRange(String cidr) {
        String[] ps = cidr.split("/");
        long base = ipToLong(ps[0]) & 0xFFFFFFFFL;
        int k = Integer.parseInt(ps[1]);
        long size = (k == 0) ? (1L << 32) : (1L << (32 - k));
        long start = base & maskOf(k);
        long end = start + size - 1;
        return new long[]{ start, end };
    }
    // ----- System.out tests -----
    static void casePrint(String name, boolean actual, boolean expected) {
        System.out.println(name + " -> actual=" + actual + ", expected=" + expected +
                " [" + (actual == expected ? "PASS" : "FAIL") + "]");
    }
    public static void main(String[] args) {
        System.out.println("=== BanList Demo ===");
        BanList bl = new BanList();
        bl.addBan("10.0.0.0/8");
        bl.addBan("192.168.1.0/24");
        casePrint("10.2.3.4", bl.isBanned("10.2.3.4"), true);
        casePrint("192.168.1.200", bl.isBanned("192.168.1.200"), true);
        casePrint("11.0.0.1", bl.isBanned("11.0.0.1"), false);
        casePrint("192.168.2.1", bl.isBanned("192.168.2.1"), false);

        bl = new BanList(); // overlap merge
        bl.addBan("10.0.0.0/8");
        bl.addBan("10.1.0.0/16");
        casePrint("overlap:10.2.3.4", bl.isBanned("10.2.3.4"), true);
        casePrint("overlap:10.1.2.3", bl.isBanned("10.1.2.3"), true);
    }
}
```

---

## E) Follow-up b2：Firewall（ALLOW/DENY，**All‑Of** 必须满足每条规则）

- 文件名：`FirewallAllOfDemo.java`  
- 运行：`javac FirewallAllOfDemo.java && java FirewallAllOfDemo`

```java
/** Firewall with All-Of semantics: ALLOW intersection + DENY union. */
public class FirewallAllOfDemo {
    enum Action { ALLOW, DENY, NONE }

    interface RuleEngine {
        void addRule(String cidr, Action action);
        Action matchIp(String ip);
        Action matchCidr(String cidr);
    }

    // ----- RangeSet for DENY union -----
    static class RangeSet {
        private final java.util.TreeMap<Long, Long> m = new java.util.TreeMap<>();
        void add(long s, long e) {
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
        boolean contains(long x) {
            var prev = m.floorEntry(x);
            return prev != null && prev.getValue() >= x;
        }
        boolean overlaps(long s, long e) {
            var prev = m.floorEntry(e);
            if (prev != null && prev.getValue() >= s) return true;
            var next = m.ceilingEntry(s);
            return next != null && next.getKey() <= e;
        }
    }

    // ----- All-Of Engine -----
    static class RuleEngineAllOf implements RuleEngine {
        private static final long MAX = 0xFFFFFFFFL;
        private boolean hasAnyRule = false;
        private boolean hasAllow = false;
        private long mustStart = 0L, mustEnd = MAX; // running intersection of ALLOW
        private final RangeSet deny = new RangeSet();       // union of DENY

        @Override public void addRule(String cidr, Action action) {
            hasAnyRule = true;
            long[] rg = cidrRange(cidr);
            if (action == Action.ALLOW) {
                hasAllow = true;
                if (rg[0] > mustStart) mustStart = rg[0];
                if (rg[1] < mustEnd)   mustEnd   = rg[1];
            } else if (action == Action.DENY) {
                deny.add(rg[0], rg[1]);
            }
        }
        @Override public Action matchIp(String ip) {
            long x = ipToLong(ip);
            if (deny.contains(x)) return Action.DENY;
            if (hasAllow) {
                if (!(mustStart <= x && x <= mustEnd)) return Action.DENY;
                return Action.ALLOW;
            }
            if (!hasAnyRule) return Action.NONE;
            return Action.ALLOW;
        }
        @Override public Action matchCidr(String cidr) {
            long[] rg = cidrRange(cidr);
            if (deny.overlaps(rg[0], rg[1])) return Action.DENY;
            if (hasAllow) {
                if (!(rg[0] >= mustStart && rg[1] <= mustEnd)) return Action.DENY;
                return Action.ALLOW;
            }
            if (!hasAnyRule) return Action.NONE;
            return Action.ALLOW;
        }
    }

    // ----- helpers -----
    static long ipToLong(String ip) {
        String[] parts = ip.split("\\.");
        long num = 0L;
        for (String p : parts) num = (num << 8) + Integer.parseInt(p.trim());
        return num & 0xFFFFFFFFL;
    }
    static long maskOf(int k) {
        if (k <= 0) return 0L;
        if (k >= 32) return 0xFFFFFFFFL;
        long m = -1L << (32 - k);
        return m & 0xFFFFFFFFL;
    }
    static long[] cidrRange(String cidr) {
        String[] ps = cidr.split("/");
        long base = ipToLong(ps[0]) & 0xFFFFFFFFL;
        int k = Integer.parseInt(ps[1]);
        long size = (k == 0) ? (1L << 32) : (1L << (32 - k));
        long start = base & maskOf(k);
        long end = start + size - 1;
        return new long[]{ start, end };
    }

    // ----- System.out tests -----
    static void casePrint(String name, Object actual, Object expected) {
        System.out.println(name + " -> actual=" + actual + ", expected=" + expected +
                " [" + (actual.equals(expected) ? "PASS" : "FAIL") + "]");
    }
    public static void main(String[] args) {
        System.out.println("=== Firewall All-Of Demo ===");
        RuleEngine eng;

        eng = new RuleEngineAllOf();
        eng.addRule("10.0.0.0/8", Action.ALLOW);
        eng.addRule("10.1.0.0/16", Action.ALLOW);
        casePrint("IP in intersection", eng.matchIp("10.1.2.3"), Action.ALLOW);
        casePrint("IP outside intersection", eng.matchIp("10.2.3.4"), Action.DENY);
        casePrint("CIDR fully inside", eng.matchCidr("10.1.128.0/17"), Action.ALLOW);
        casePrint("CIDR exceeds intersection", eng.matchCidr("10.0.0.0/8"), Action.DENY);

        eng = new RuleEngineAllOf();
        eng.addRule("10.0.0.0/8", Action.ALLOW);
        eng.addRule("10.1.0.0/16", Action.ALLOW);
        eng.addRule("10.1.2.0/24", Action.DENY);
        casePrint("IP allowed region", eng.matchIp("10.1.3.4"), Action.ALLOW);
        casePrint("IP denied region", eng.matchIp("10.1.2.5"), Action.DENY);
        casePrint("CIDR overlaps deny", eng.matchCidr("10.1.2.0/24"), Action.DENY);
        casePrint("CIDR inside allow", eng.matchCidr("10.1.1.0/24"), Action.ALLOW);

        eng = new RuleEngineAllOf();
        eng.addRule("10.0.0.0/8", Action.ALLOW);
        eng.addRule("11.0.0.0/8", Action.ALLOW); // empty intersection
        casePrint("IP 10.x deny", eng.matchIp("10.1.2.3"), Action.DENY);
        casePrint("IP 11.x deny", eng.matchIp("11.1.2.3"), Action.DENY);
        casePrint("CIDR 10/8 deny", eng.matchCidr("10.0.0.0/8"), Action.DENY);

        eng = new RuleEngineAllOf();
        eng.addRule("10.1.0.0/16", Action.DENY);
        casePrint("IP in deny", eng.matchIp("10.1.2.3"), Action.DENY);
        casePrint("IP not denied", eng.matchIp("10.2.3.4"), Action.ALLOW);
        casePrint("CIDR denied", eng.matchCidr("10.1.0.0/16"), Action.DENY);
        casePrint("CIDR not denied", eng.matchCidr("10.2.0.0/16"), Action.ALLOW);

        eng = new RuleEngineAllOf();
        casePrint("no rules ip -> NONE", eng.matchIp("1.2.3.4"), Action.NONE);
        casePrint("no rules cidr -> NONE", eng.matchCidr("1.2.3.0/24"), Action.NONE);
    }
}
```

---

## F) Variation（可选）：Firewall（Trie/LPM，最长前缀匹配）

- 文件名：`FirewallTrieDemo.java`  
- 运行：`javac FirewallTrieDemo.java && java FirewallTrieDemo`

```java
/** Firewall using Trie (Longest Prefix Match). Last-write-wins per prefix. */
public class FirewallTrieDemo {
    enum Action { ALLOW, DENY, NONE }

    interface RuleEngine {
        void addRule(String cidr, Action action);
        Action matchIp(String ip);
        Action matchCidr(String cidr);
    }

    static class RuleEngineTrie implements RuleEngine {
        static class Node {
            Node zero, one;
            Action action = Action.NONE;
            int order = -1; // larger order means later insert
        }
        private final Node root = new Node();
        private int counter = 0;

        @Override public void addRule(String cidr, Action action) {
            String[] ps = cidr.split("/");
            long base = ipToLong(ps[0]) & 0xFFFFFFFFL;
            int k = Integer.parseInt(ps[1]);
            Node cur = root;
            for (int i = 31; i >= 32 - k; i--) {
                int bit = (int)((base >>> i) & 1);
                cur = (bit == 0) ? (cur.zero = (cur.zero == null ? new Node() : cur.zero))
                                 : (cur.one  = (cur.one  == null ? new Node() : cur.one));
            }
            if (cur.order <= counter) {
                cur.action = action;
                cur.order = ++counter; // last write wins
            }
        }

        @Override public Action matchIp(String ip) {
            long x = ipToLong(ip) & 0xFFFFFFFFL;
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

        @Override public Action matchCidr(String cidr) {
            String[] ps = cidr.split("/");
            long base = ipToLong(ps[0]) & 0xFFFFFFFFL;
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

    // ----- helpers -----
    static long ipToLong(String ip) {
        String[] parts = ip.split("\\.");
        long num = 0L;
        for (String p : parts) num = (num << 8) + Integer.parseInt(p.trim());
        return num & 0xFFFFFFFFL;
    }

    // ----- System.out tests -----
    static void casePrint(String name, Object actual, Object expected) {
        System.out.println(name + " -> actual=" + actual + ", expected=" + expected +
                " [" + (actual.equals(expected) ? "PASS" : "FAIL") + "]");
    }
    public static void main(String[] args) {
        RuleEngineTrie eng = new RuleEngineTrie();
        eng.addRule("10.0.0.0/8", Action.ALLOW);
        eng.addRule("10.1.0.0/16", Action.DENY);
        eng.addRule("10.1.2.0/24", Action.ALLOW);

        casePrint("10.2.3.4", eng.matchIp("10.2.3.4"), Action.ALLOW);
        casePrint("10.1.3.4", eng.matchIp("10.1.3.4"), Action.DENY);
        casePrint("10.1.2.5", eng.matchIp("10.1.2.5"), Action.ALLOW);

        casePrint("CIDR 10.2/16", eng.matchCidr("10.2.0.0/16"), Action.ALLOW);
        casePrint("CIDR 10.1/16", eng.matchCidr("10.1.0.0/16"), Action.DENY);
    }
}
```

---

## G) 编译与运行总览

```bash
# 按需选择要演示的块，每个块都是可独立运行的单文件：

# A: Basic
javac IpToCidrBasicDemo.java && java IpToCidrBasicDemo

# B: Variation(long n)
javac IpToCidrSafeDemo.java && java IpToCidrSafeDemo

# C: Helpers
javac CidrHelpersDemo.java && java CidrHelpersDemo

# D: Follow-up b1
javac BanListDemo.java && java BanListDemo

# E: Follow-up b2 (All-Of)
javac FirewallAllOfDemo.java && java FirewallAllOfDemo

# F: Variation (Trie/LPM)
javac FirewallTrieDemo.java && java FirewallTrieDemo
```
