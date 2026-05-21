# Module 29 — Competitive Programming in Java

> Goal: write fast, correct Java for contests (Codeforces, AtCoder, LeetCode). The standard library is huge; the slow defaults are deadly. After this module you have a tight personal toolkit and don't get TLE'd by trivial things like `Scanner`.

---

## 1. The contest template

Save this as `Main.java`. It's the starting point for every problem.

```java
import java.io.*;
import java.util.*;

public class Main {

    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        StringBuilder out = new StringBuilder();
        StreamTokenizer in = new StreamTokenizer(br);

        in.nextToken(); int t = (int) in.nval;          // number of test cases
        while (t-- > 0) {
            in.nextToken(); int n = (int) in.nval;
            int[] a = new int[n];
            for (int i = 0; i < n; i++) { in.nextToken(); a[i] = (int) in.nval; }
            solve(n, a, out);
        }

        System.out.print(out);
    }

    static void solve(int n, int[] a, StringBuilder out) {
        long sum = 0;
        for (int x : a) sum += x;
        out.append(sum).append('\n');
    }
}
```

Reads input with `StreamTokenizer` (fastest), accumulates output in a `StringBuilder`, prints once. Calling `System.out.println` per line is **catastrophically slow** under contest constraints.

---

## 2. Fast input

Three options, in increasing speed:

### 2.1 `Scanner` — DON'T

```java
Scanner sc = new Scanner(System.in);
int n = sc.nextInt();
```

Convenient. ~5–20× slower than the alternatives. Avoid in contests.

### 2.2 `BufferedReader` + manual parsing

```java
BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
StringTokenizer st = new StringTokenizer(br.readLine());
int n = Integer.parseInt(st.nextToken());
int m = Integer.parseInt(st.nextToken());

int[] a = new int[n];
st = new StringTokenizer(br.readLine());
for (int i = 0; i < n; i++) a[i] = Integer.parseInt(st.nextToken());
```

Fast and explicit. The "battle-tested" choice.

### 2.3 `StreamTokenizer` — fastest text parsing

```java
StreamTokenizer in = new StreamTokenizer(new BufferedReader(new InputStreamReader(System.in)));
in.nextToken(); int n = (int) in.nval;
```

Handles whitespace and integer/float parsing internally. `nval` is always a `double`; cast to `int`/`long` as needed.

**For really tight time limits**, write a `FastReader` that reads bytes from `System.in` directly:

```java
static final InputStream IN = System.in;
static final byte[] BUF = new byte[1 << 16];
static int BUF_POS = 0, BUF_LEN = 0;

static int readByte() throws IOException {
    if (BUF_POS == BUF_LEN) {
        BUF_LEN = IN.read(BUF);
        BUF_POS = 0;
        if (BUF_LEN <= 0) return -1;
    }
    return BUF[BUF_POS++];
}

static int nextInt() throws IOException {
    int c, n = 0; boolean neg = false;
    do { c = readByte(); } while (c <= ' ' && c != -1);
    if (c == '-') { neg = true; c = readByte(); }
    while (c > ' ') { n = n * 10 + c - '0'; c = readByte(); }
    return neg ? -n : n;
}
```

This is ~10× faster than `Integer.parseInt` and matches C++'s `scanf`.

---

## 3. Fast output

Always:
```java
StringBuilder out = new StringBuilder();
// ... build all output ...
System.out.print(out);
```

Or `PrintWriter` with `BufferedWriter`:
```java
PrintWriter pw = new PrintWriter(new BufferedWriter(new OutputStreamWriter(System.out)));
pw.println(answer);
pw.flush();          // remember to flush at the end
```

Calling `System.out.println` 10⁵ times → time-limit exceeded on most contest judges.

---

## 4. Primitive arrays beat collections

In contest code, prefer:
- `int[]`, `long[]`, `char[]` over `List<Integer>`, `List<Long>`.
- 1D arrays over 2D where possible (cache-friendlier, fewer allocations).
- `int[][] graph` adjacency-array over `Map<Integer, List<Integer>>`.

Why: autoboxing is slow, lists allocate more, hash maps have constant-factor overhead. A `HashMap<Integer, Integer>` lookup can be 10× slower than an `int[]` array access.

For graphs, use *adjacency lists* via flat arrays:
```java
int[] head = new int[n];      // index of first edge from v
int[] next = new int[2 * m];   // next edge with same head
int[] to   = new int[2 * m];   // destination of edge
int edges = 0;

Arrays.fill(head, -1);

void addEdge(int u, int v) {
    to[edges] = v; next[edges] = head[u]; head[u] = edges++;
}

// iterate edges from u
for (int e = head[u]; e != -1; e = next[e]) {
    int v = to[e];
    ...
}
```

Same big-O as `List<List<Integer>>`, ~3× faster in practice.

---

## 5. Common data structures

`PriorityQueue<Integer>` works but boxes. For pure performance:

```java
// Min-heap on long
long[] heap = new long[n];
int sz = 0;
void push(long x) {
    heap[sz] = x;
    int i = sz++;
    while (i > 0 && heap[(i-1)/2] > heap[i]) {
        long t = heap[i]; heap[i] = heap[(i-1)/2]; heap[(i-1)/2] = t;
        i = (i-1)/2;
    }
}
long pop() {
    long r = heap[0];
    heap[0] = heap[--sz];
    int i = 0;
    while (true) {
        int l = 2*i+1, r2 = 2*i+2, m = i;
        if (l < sz && heap[l] < heap[m]) m = l;
        if (r2 < sz && heap[r2] < heap[m]) m = r2;
        if (m == i) break;
        long t = heap[i]; heap[i] = heap[m]; heap[m] = t;
        i = m;
    }
    return r;
}
```

That's a full int-heap implementation worth memorizing. The standard `PriorityQueue<Long>` is slower because every pop/push involves boxing.

For a `TreeSet`-equivalent of `int`s with O(log n) operations, you usually need a Fenwick tree (BIT), segment tree, or a hand-rolled balanced BST. Java's `TreeSet<Integer>` is fine for most problems; reach for the specialized one only when TLE'd.

### 5.1 Hash-map alternatives

Java's `HashMap<Integer, Integer>` is acceptable for most problems. For the absolute fastest int-keyed maps, libraries like Eclipse Collections or HPPC provide `IntIntHashMap`. In contests where you can paste code, hand-roll an open-addressing primitive hash map:

```java
static int[] keys;
static int[] vals;
static boolean[] used;
static int mask;
static void initMap(int cap) {
    int p = 1; while (p < cap * 2) p <<= 1;
    keys = new int[p]; vals = new int[p]; used = new boolean[p]; mask = p - 1;
}
static int hash(int x) { x = (x ^ (x>>>16)) * 0x85ebca6b; x = (x ^ (x>>>13)) * 0xc2b2ae35; return x ^ (x>>>16); }
static void put(int k, int v) {
    int i = hash(k) & mask;
    while (used[i] && keys[i] != k) i = (i + 1) & mask;
    used[i] = true; keys[i] = k; vals[i] = v;
}
static int getOrDefault(int k, int d) {
    int i = hash(k) & mask;
    while (used[i]) {
        if (keys[i] == k) return vals[i];
        i = (i + 1) & mask;
    }
    return d;
}
```

---

## 6. Modular arithmetic

A standard tool in competitive programming. Java has `BigInteger.modPow`, but for `long`-fitting values, hand-roll:

```java
static final int MOD = 1_000_000_007;

static long add(long a, long b) { long s = a + b; return s >= MOD ? s - MOD : s; }
static long sub(long a, long b) { long s = a - b; return s < 0 ? s + MOD : s; }
static long mul(long a, long b) { return a * b % MOD; }
static long pow(long a, long e) {
    long r = 1 % MOD;
    a %= MOD;
    while (e > 0) {
        if ((e & 1) == 1) r = r * a % MOD;
        a = a * a % MOD;
        e >>= 1;
    }
    return r;
}
static long inv(long a) { return pow(a, MOD - 2); }   // Fermat, MOD prime
```

Watch for overflow: `a * b` where both are near `MOD` ≈ 10⁹ produces a product ≈ 10¹⁸, which fits in a `long` (max ~9.2 × 10¹⁸). Adding two such products before modding overflows — always mod after each multiply.

For non-prime moduli or 64-bit moduli, use `BigInteger` or the Math.floorMod / Math.multiplyHigh tricks.

---

## 7. Big numbers

`BigInteger` and `BigDecimal` are *slow* but unavoidable for arbitrary precision:

```java
BigInteger x = BigInteger.valueOf(123456789L);
BigInteger y = new BigInteger("12345678901234567890");
BigInteger z = x.multiply(y).mod(BigInteger.valueOf(MOD));
```

Avoid in tight loops. If a problem fits `long`, use `long`. `BigInteger` operations are 100× slower than primitive ones.

---

## 8. Algorithmic templates worth memorizing

### 8.1 Binary search on integers

```java
int lo = 0, hi = n;
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;
    if (predicate(mid)) hi = mid; else lo = mid + 1;
}
// lo == hi == smallest index where predicate is true
```

The `lo + (hi - lo) / 2` form avoids overflow vs `(lo + hi) / 2`.

### 8.2 DFS iterative (to avoid stack overflow)

JVM stack ~512 KB ⇒ ~10⁴ recursion depth. For deeper, iterate:

```java
int[] stack = new int[n];
int[] iter  = new int[n];
int top = 0;
stack[top] = root; iter[top] = head[root];

while (top >= 0) {
    int u = stack[top];
    int e = iter[top];
    if (e == -1) { top--; continue; }
    iter[top] = next[e];
    int v = to[e];
    if (!visited[v]) {
        visited[v] = true;
        top++;
        stack[top] = v;
        iter[top] = head[v];
    }
}
```

### 8.3 Fenwick tree (BIT)

```java
int[] bit = new int[n + 1];
void update(int i, int delta) { for (; i <= n; i += i & -i) bit[i] += delta; }
int query(int i)              { int s = 0; for (; i > 0; i -= i & -i) s += bit[i]; return s; }
```

Range-sum queries in O(log n). Trivial to write, hard to live without.

### 8.4 Sort with `Arrays.sort` — primitive caveat

`Arrays.sort(int[])` uses dual-pivot quicksort (fast, but O(n²) worst-case on adversarial inputs). Codeforces "anti-quicksort" tests can TLE.

The fix:
```java
Long[] a = ...;                       // Long[], not long[]
Arrays.sort(a);                       // uses TimSort — guaranteed O(n log n)
```

Or shuffle first:
```java
Random rng = new Random();
for (int i = n - 1; i > 0; i--) {
    int j = rng.nextInt(i + 1);
    int t = a[i]; a[i] = a[j]; a[j] = t;
}
Arrays.sort(a);
```

---

## 9. Common gotchas

- **Integer overflow.** `int * int` overflows past 2 × 10⁹. Use `long` or `(long) a * b`.
- **Recursion depth.** Default JVM stack ≈ 512 KB. Deep recursion (n > 10⁴) hits `StackOverflowError`. Either iterate or set `-Xss64m` (most judges don't let you, so iterate).
- **`BigDecimal` precision.** Default `BigDecimal.divide(...)` throws on non-terminating decimals. Pass a `MathContext` or scale.
- **`long` literal**. `1 << 40` is `int` overflow → `0`. Write `1L << 40`.
- **`String + char` is fine; `char + char` is `int`.** `'a' + 'b'` is `195`.
- **`Arrays.asList(int[])` is a list of *one* element**, the array. To wrap, autobox: `Arrays.stream(a).boxed().toList()`.
- **`HashSet<int[]>`/`HashMap<int[], ?>`.** Arrays use identity equals, so different arrays with same content aren't equal. Use `List.of(...)` or wrap.
- **Reading input fully.** Don't forget `\n` lines between blocks. `nextInt()` skips whitespace; `readLine()` doesn't.
- **`Math.floorMod` vs `%`.** `%` returns negative for negative dividends. Use `Math.floorMod(a, m)` for nonnegative remainder.

---

## 10. Microbenchmarking note

`System.nanoTime()` is the right clock for measuring durations. *Run multiple iterations*, JIT-warm up first. For serious benchmarking outside contests, use **JMH**.

```java
long t0 = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) ...;
System.out.println((System.nanoTime() - t0) / 1_000_000.0 + " ms");
```

---

## 11. Sample contest problem

> "Given an array of n integers, find the number of pairs (i, j) with i < j and a[i] + a[j] = k."

```java
import java.io.*;
import java.util.*;

public class Main {
    public static void main(String[] args) throws IOException {
        StreamTokenizer in = new StreamTokenizer(new BufferedReader(new InputStreamReader(System.in)));
        StringBuilder out = new StringBuilder();

        in.nextToken(); int n = (int) in.nval;
        in.nextToken(); long k = (long) in.nval;

        int[] a = new int[n];
        for (int i = 0; i < n; i++) { in.nextToken(); a[i] = (int) in.nval; }

        HashMap<Long, Integer> seen = new HashMap<>();
        long count = 0;
        for (int x : a) {
            count += seen.getOrDefault(k - x, 0);
            seen.merge((long) x, 1, Integer::sum);
        }
        out.append(count).append('\n');
        System.out.print(out);
    }
}
```

`HashMap<Long, Integer>` here is convenient. For tighter constraints, switch to a primitive open-addressing map.

---

## 12. Try this

1. Solve a Codeforces problem with `Scanner` and submit. Then resubmit with `BufferedReader`. Compare runtimes.
2. Implement Dijkstra with a primitive heap on long-encoded (distance, vertex) pairs vs `PriorityQueue<long[]>` vs `PriorityQueue<long[]>` boxed. Measure.
3. Reproduce the "anti-quicksort" Codeforces failure: sort a 10⁵-element `int[]` constructed adversarially. Note the TLE. Apply the shuffle-then-sort trick; it now finishes.
4. Solve a tree-DP problem of depth 10⁵. First with recursion (`StackOverflowError`); then iterate.

---

This is the last module. You've now covered Java from the language and JVM through enterprise frameworks to monorepo navigation and contest tricks.

The course is yours to revisit. Good luck, and keep typing.

— **Back to [README](./README.md)**
