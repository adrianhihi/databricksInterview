
# IP to CIDR & Firewall Rules（All‑Of 语义）——**完整 Solution + Test Cases（一起放在本页）**

> 语言：中文讲解；**代码注释均为英文**。  
> 覆盖三类场景：
> 1) 原题速解要点（最少 CIDR 覆盖，简述）；  
> 2) **Ban 列表**：给一堆 CIDR 与一个 IP，判断是否会被 ban（并集 + `contains`）；  
> 3) **Firewall（ALLOW / DENY）All‑Of 语义**：**必须满足每一条 rule**；考虑 overlap，先 merge。  
>    - `matchIp(ip)`：该 IP 是否 **allow**；  
>    - `matchCidr(cidr)`：该 CIDR **整段**是否能被 allow。

---

## A. 思路与数据结构（精简）

- **CIDR → 区间**：`"a.b.c.d/k"` → `[start, end]`（无符号 32 位）。
- **Ban 列表**：用 `RangeSet(TreeMap)` 维护**并集**（插入即 merge）；查询用 `contains(ip)`。  
- **Firewall（All‑Of）**：
  - **DENY**：并集（`RangeSet`），命中即 false；
  - **ALLOW**：**交集**（running intersection），必须完全落入公共交集才 true；交集为空则全拒绝。
- **复杂度**：插入/查询 `O(log M)`；ALLOW 交集更新 `O(1)`。

> 原题（最少 CIDR 覆盖）速记：每步块大小 = `min(2^{tz(start)}, highestPow2(n))`；输出 `/mask`。

---

## B. **完整代码（Solution）**

> 复制即用。为便于粘贴演示，以下类**不加 package**；若放入项目，可自行加包名。

### B.1 `IpUtils.java` —— IPv4/CIDR 工具

```java
import java.util.*;

/** IPv4/CIDR helpers. */
public final class IpUtils {
    private IpUtils() {}

    /** Convert dotted IPv4 to unsigned 32-bit long. */
    public static long ipToLong(String ip) {
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

    /** Convert unsigned 32-bit number back to dotted IPv4. */
    public static String longToIp(long x) {
        int b1 = (int)((x >>> 24) & 0xFF);
        int b2 = (int)((x >>> 16) & 0xFF);
        int b3 = (int)((x >>> 8)  & 0xFF);
        int b4 = (int)( x         & 0xFF);
        return b1 + "." + b2 + "." + b3 + "." + b4;
    }

    /** Top k bits 1, others 0, as unsigned 32-bit in long. */
    public static long maskOf(int k) {
        if (k <= 0) return 0L;
        if (k >= 32) return 0xFFFFFFFFL;
        long m = -1L << (32 - k);
        return m & 0xFFFFFFFFL;
    }

    /** "a.b.c.d/k" -> [start,end] (unsigned). */
    public static long[] cidrRange(String cidr) {
        String[] parts = cidr.split("/");
        if (parts.length != 2) throw new IllegalArgumentException("Invalid CIDR: " + cidr);
        long base = ipToLong(parts[0]);
        int k = Integer.parseInt(parts[1]);
        long size = (k == 0) ? (1L << 32) : (1L << (32 - k));
        long start = base & maskOf(k);
        long end = start + size - 1;
        return new long[]{ start, end };
    }
}
```

### B.2 `RangeSet.java` —— 区间并集（插入即合并）

```java
import java.util.*;

/** Union of disjoint, merged closed intervals [start,end]. */
public class RangeSet {
    private final TreeMap<Long, Long> m = new TreeMap<>(); // start -> end

    /** Add [s,e] and merge overlaps/touches. */
    public void add(long s, long e) {
        if (s > e) return;

        Map.Entry<Long, Long> prev = m.floorEntry(s);
        if (prev != null && prev.getValue() + 1 >= s) {
            s = Math.min(s, prev.getKey());
            e = Math.max(e, prev.getValue());
            m.remove(prev.getKey());
        }

        Map.Entry<Long, Long> cur = m.ceilingEntry(s);
        while (cur != null && cur.getKey() <= e + 1) {
            e = Math.max(e, cur.getValue());
            m.remove(cur.getKey());
            cur = m.ceilingEntry(s);
        }
        m.put(s, e);
    }

    /** Whether a point x is covered. */
    public boolean contains(long x) {
        Map.Entry<Long, Long> prev = m.floorEntry(x);
        return prev != null && prev.getValue() >= x;
    }

    /** Whether [s,e] is fully covered by the union. */
    public boolean covers(long s, long e) {
        Map.Entry<Long, Long> prev = m.floorEntry(s);
        return prev != null && prev.getValue() >= e;
    }

    /** Whether [s,e] intersects the union. */
    public boolean overlaps(long s, long e) {
        Map.Entry<Long, Long> prev = m.floorEntry(e);
        if (prev != null && prev.getValue() >= s) return true;
        Map.Entry<Long, Long> next = m.ceilingEntry(s);
        return next != null && next.getKey() <= e;
    }
}
```

### B.3 `BanList.java` —— 仅判断 IP 是否被 Ban（并集 + contains）

```java
/** A simple ban list using a union of CIDR ranges. */
public class BanList {
    private final RangeSet banned = new RangeSet();

    /** Add a CIDR to ban set. */
    public void addBan(String cidr) {
        long[] rg = IpUtils.cidrRange(cidr);
        banned.add(rg[0], rg[1]);
    }

    /** Is a single IP banned? */
    public boolean isBanned(String ip) {
        long x = IpUtils.ipToLong(ip);
        return banned.contains(x);
    }
}
```

### B.4 `RuleEngineAllOf.java` —— Firewall（All‑Of，必须满足每条规则）

```java
enum Action { ALLOW, DENY, NONE }

interface RuleEngine {
    void addRule(String cidr, Action action);
    Action matchIp(String ip);    // ALLOW / DENY / NONE
    Action matchCidr(String cidr);
}

/**
 * All-of semantics: a candidate must satisfy EVERY rule.
 * - ALLOW rules: running intersection across all ALLOW CIDRs
 * - DENY rules: union (RangeSet); any overlap/containment causes failure
 *
 * Policy:
 *   - No rules at all -> NONE   (switch to ALLOW if product needs "default allow")
 *   - Only DENY rules  -> not denied -> ALLOW
 *   - With ALLOW rules -> candidate must be inside the intersection
 */
public class RuleEngineAllOf implements RuleEngine {
    private static final long MAX = 0xFFFFFFFFL;

    private boolean hasAnyRule = false;
    private boolean hasAllow = false;

    // Running intersection of ALLOW rules
    private long mustStart = 0L;
    private long mustEnd   = MAX;

    // Union of DENY ranges
    private final RangeSet deny = new RangeSet();

    @Override
    public void addRule(String cidr, Action action) {
        hasAnyRule = true;
        long[] rg = IpUtils.cidrRange(cidr);
        if (action == Action.ALLOW) {
            hasAllow = true;
            if (rg[0] > mustStart) mustStart = rg[0];
            if (rg[1] < mustEnd)   mustEnd   = rg[1];
        } else if (action == Action.DENY) {
            deny.add(rg[0], rg[1]);
        }
    }

    @Override
    public Action matchIp(String ip) {
        long x = IpUtils.ipToLong(ip);
        if (deny.contains(x))  return Action.DENY;

        if (hasAllow) {
            if (!(mustStart <= x && x <= mustEnd)) return Action.DENY;
            return Action.ALLOW;
        }
        if (!hasAnyRule) return Action.NONE;  // or ALLOW
        return Action.ALLOW;
    }

    @Override
    public Action matchCidr(String cidr) {
        long[] rg = IpUtils.cidrRange(cidr);
        if (deny.overlaps(rg[0], rg[1])) return Action.DENY;

        if (hasAllow) {
            if (!(rg[0] >= mustStart && rg[1] <= mustEnd)) return Action.DENY;
            return Action.ALLOW;
        }
        if (!hasAnyRule) return Action.NONE;  // or ALLOW
        return Action.ALLOW;
    }

    /** For debugging/inspection. */
    public long[] currentAllowIntersection() { return new long[]{ mustStart, mustEnd }; }
}
```

---

## C. **测试用例（Test Cases）** —— 与 Solution 放在一起

> 直接复制为 `*Test.java` 使用 JUnit5；或把断言替换为 `System.out.println` 快速观察。

### C.1 `BanListTest.java`

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

public class BanListTest {
    @Test
    void banAndQuery() {
        BanList bl = new BanList();
        bl.addBan("10.0.0.0/8");       // ban 10.0.0.0 ~ 10.255.255.255
        bl.addBan("192.168.1.0/24");   // ban 192.168.1.0 ~ 192.168.1.255

        assertTrue(bl.isBanned("10.2.3.4"));
        assertTrue(bl.isBanned("192.168.1.200"));
        assertFalse(bl.isBanned("11.0.0.1"));
        assertFalse(bl.isBanned("192.168.2.1"));
    }

    @Test
    void overlapMerge() {
        BanList bl = new BanList();
        bl.addBan("10.0.0.0/8");       // [10.0.0.0 .. 10.255.255.255]
        bl.addBan("10.1.0.0/16");      // overlap; union still covers 10.2.3.4

        assertTrue(bl.isBanned("10.2.3.4"));
        assertTrue(bl.isBanned("10.1.2.3"));
    }
}
```

### C.2 `RuleEngineAllOfTest.java`

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

public class RuleEngineAllOfTest {
    @Test
    void allowIntersection_basic() {
        RuleEngineAllOf eng = new RuleEngineAllOf();
        eng.addRule("10.0.0.0/8", Action.ALLOW);    // ALLOW universe inside 10/8
        eng.addRule("10.1.0.0/16", Action.ALLOW);   // intersection -> 10.1/16

        assertEquals(Action.ALLOW, eng.matchIp("10.1.2.3"));
        assertEquals(Action.DENY,  eng.matchIp("10.2.3.4")); // outside intersection

        // CIDR fully inside intersection
        assertEquals(Action.ALLOW, eng.matchCidr("10.1.128.0/17"));
        // CIDR exceeds intersection
        assertEquals(Action.DENY,  eng.matchCidr("10.0.0.0/8"));
    }

    @Test
    void withDeny_union() {
        RuleEngineAllOf eng = new RuleEngineAllOf();
        eng.addRule("10.0.0.0/8", Action.ALLOW);
        eng.addRule("10.1.0.0/16", Action.ALLOW);
        eng.addRule("10.1.2.0/24", Action.DENY);    // deny union

        assertEquals(Action.ALLOW, eng.matchIp("10.1.3.4"));
        assertEquals(Action.DENY,  eng.matchIp("10.1.2.5")); // denied

        // CIDR overlaps deny -> DENY
        assertEquals(Action.DENY,  eng.matchCidr("10.1.2.0/24"));
        // CIDR fully inside intersection and no deny overlap -> ALLOW
        assertEquals(Action.ALLOW, eng.matchCidr("10.1.1.0/24"));
    }

    @Test
    void emptyIntersection_allDenied() {
        RuleEngineAllOf eng = new RuleEngineAllOf();
        eng.addRule("10.0.0.0/8", Action.ALLOW);
        eng.addRule("11.0.0.0/8", Action.ALLOW); // no overlap -> empty intersection

        assertEquals(Action.DENY, eng.matchIp("10.1.2.3"));
        assertEquals(Action.DENY, eng.matchIp("11.1.2.3"));
        assertEquals(Action.DENY, eng.matchCidr("10.0.0.0/8"));
    }

    @Test
    void onlyDeny_thenNotDeniedMeansAllow() {
        RuleEngineAllOf eng = new RuleEngineAllOf();
        eng.addRule("10.1.0.0/16", Action.DENY);

        assertEquals(Action.DENY,  eng.matchIp("10.1.2.3"));
        assertEquals(Action.ALLOW, eng.matchIp("10.2.3.4"));

        assertEquals(Action.DENY,  eng.matchCidr("10.1.0.0/16"));
        assertEquals(Action.ALLOW, eng.matchCidr("10.2.0.0/16"));
    }

    @Test
    void noRules_returnsNone() {
        RuleEngineAllOf eng = new RuleEngineAllOf();
        assertEquals(Action.NONE, eng.matchIp("1.2.3.4"));
        assertEquals(Action.NONE, eng.matchCidr("1.2.3.0/24"));
    }
}
```

---

## D. 运行说明（可选）

- **直接复制进项目**：用 JUnit5 运行以上 `*Test.java`。  
- **无测试框架也可**：把断言替换为 `System.out.println` 做 smoke test。  
- **原题最少 CIDR 覆盖**：若需要完整实现与单测，可把我之前的 `IpToCIDR` 代码一并粘进来（此页重心在 Firewall）。

---

## E. 面试复盘要点（30 秒）

- Ban：CIDR→区间；RangeSet 并集；`contains(ip)`。  
- Firewall（All‑Of）：
  - 任何 DENY 命中/重叠 ⇒ **false**；
  - ALLOW 取**交集**，IP 要在交集内，CIDR 要被交集**完全覆盖**；
  - 只有 DENY：未命中即 true；无规则：`NONE`（或改策略为默认 allow）。
