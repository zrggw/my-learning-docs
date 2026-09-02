# C 语言算法题实战指南（刷题速查）

> 用途：用 **C 语言**刷算法题的速查手册。面向**熟悉 C++ 刷题**的读者，重点讲"C 没有 STL 容器，怎么手写等价物 + 高频写法"。
> 使用方式：做题时随查随用；需要展开某节可让我补充。
> 定位：解决 **"C 怎么写出 C++ 里 vector / map / stack / queue / sort / priority_queue 的效果"** 。不重复讲算法思想，只讲 C 的落地写法。
> 与 `cpp-to-go-cheatsheet.md` 区别：那篇是 C++→Go，这篇是 C++→**C**；Go 靠语言自带容器，C 全靠手写。

---

## 1. 心智模型：C 刷题的"三无"

- [ ] C 在算法题里的"三无"：**无 STL 容器、无自动 GC、无 `std::` 命名空间**。所有容器要么手写要么用数组模拟。
- [ ] 但 C 也有优势：**零依赖、极快**（直接捏到内存、无动态指针追逐），常数性能好，更贴近底层。
- [ ] 常用做法：**用固定大小数组模拟一切容器**（牺牲一点灵活性，换来速度与简单）。比赛/刷题场景几乎不会开满内存，开 `int a[100000]` 这类大数组是常态。
- [ ] 数组大小建议开**额外裕量**：题目给 `n <= 1e5`，数组开 `1e5 + 5` 或 `2e5`，避免越界。

> 核心思想：**C 里"容器 = 数组 + 头尾指针/下标"**，动态扩容几乎不用管（刷题场景直接开大）。

---

## 2. 输入输出（第一步，必须快）

- [ ] **`scanf` / `printf`**——标准用法，够用：
  ```c
  scanf("%d", &n);
  scanf("%d%d", &a, &b);
  printf("%d\n", ans);
  ```
  - `scanf` 读 `int` 用 `%d`，`long long` 用 `%lld`，`double` 用 `%lf`，`char` 用 `%c`，字符串（单词）用 `%s`
  - `%s` 只读到空白（空格/换行/制表）为止；要读整行用 `fgets`
- [ ] **读入一整行（含空格）**：
  ```c
  char line[1000];
  fgets(line, sizeof(line), stdin);     // 读一行，含末尾换行
  // 去掉末尾换行：line[strcspn(line, "\n")] = 0;
  ```
- [ ] **快读快写（速度敏感 / 大量输入时）- 手写**：
  ```c
  // 快读整数（比赛最常用）
  #include <ctype.h>
  int readInt() {
      int x = 0, f = 1; char c;
      c = getchar();
      while (c != '-' && !isdigit(c)) c = getchar();   // 跳过空白
      if (c == '-') { f = -1; c = getchar(); }
      while (isdigit(c)) { x = x * 10 + (c - '0'); c = getchar(); }
      return x * f;
  }
  // 快写
  void writeInt(int x) { if (x < 0) { putchar('-'); x = -x; }
      if (x > 9) writeInt(x / 10); putchar(x % 10 + '0'); }
  ```
  - 用 `getchar()` / `putchar()` 比 `scanf`/`printf` 快；配合 `setvbuf` 或直接按行读 `fread` 更快
  - 一句口诀：**`getchar` + 手写解析 = C 世界最快的读法**。
- [ ] 与 C++ 对照：`scanf/printf` ≈ `cin/cout`；手写快读 ≈ 关闭同步的 `cin` / `getchar_unlocked`。C 无 `ios::sync_with_stdio(false)` 的等价物，习惯手写。

---

## 3. 容器速查：C 怎么手写 STL

> 以下"容器"全部用数组 + 下标实现，是 C 刷题的标配写法。

### 3.1 动态数组（≈ vector）
```c
int a[100005]; int len = 0;       // 固定大小，len 记录长度
a[len++] = x;                     // push_back
// 访问 a[i]、a[len-1]（尾元素）、len（size）
```
> 刷题直接开大数组即可，不需要真正"动态"。

### 3.2 栈（≈ stack）
```c
int st[100005]; int top = 0;      // top 指向栈顶的下一位置
st[top++] = x;                    // push
int t = st[--top];                // pop + 取栈顶
int t = st[top - 1];              // 只看栈顶（top()）
```
> 用 `top` 当栈针即可，比 C++ 的 `std::stack` 还直白。

### 3.3 队列（≈ queue，BFS 关键）
```c
int q[100005]; int head = 0, tail = 0;
q[tail++] = x;                    // push（tail 只用增）
int cur = q[head++];              // pop（head 只用增）
int len = tail - head;            // 当前队列长度
```
> **关键**：头尾指针**只增不减**（环形队列退化为线性），避免反复 `memmove`。这就是 C 世界 BFS 的标准写法。

### 3.4 哈希表（≈ unordered_map / set）
- **刷题优先用"值域数组"**（若 key 范围小且可做下标）：
  ```c
  int cnt[100005] = {0};          // key 直接当下标
  cnt[x]++;                       // 插入/计数
  int ok = cnt[x] > 0;            // 查找
  ```
- **key 很大 / string 时**再手写**哈希**（数组拉链法）或排序后二分。两个实用实现：
  - **整数哈希（拉链法）**：
    ```c
    #define HS 200003             // 找质数，约为数据量 2 倍
    int head[HS], nxt[100005], key[100005], val[100005], tot = 0;
    int h(int x) { return (x % HS + HS) % HS; }
    void insert(int k, int v) {
        int id = h(k);
        key[tot] = k; val[tot] = v; nxt[tot] = head[id]; head[id] = tot++;
    }
    int* find(int k) {
        for (int i = head[h(k)]; i != -1; i = nxt[i])
            if (key[i] == k) return &val[i];
        return NULL;
    }
    ```
  - **更省事：排序 + 二分**。把要查的东西收集进数组 `sort` 后 `bsearch`，是 C 里最"无脑"且高效的替代。

### 3.5 优先队列 / 堆（≈ priority_queue）
- 手写**二叉堆**（数组 + 上浮/下沉），是 C 刷题的标配：
  ```c
  int heap[200005]; int hsz = 0;
  void push(int x) {               // 上浮
      int i = ++hsz; heap[i] = x;
      while (i > 1 && heap[i] < heap[i >> 1]) {
          int t = heap[i]; heap[i] = heap[i >> 1]; heap[i >> 1] = t; i >>= 1;
      }
  }
  int pop() {                      // 下沉
      int res = heap[1], t = heap[hsz--], i = 1;
      while ((i << 1) <= hsz) {
          int c = i << 1;
          if (c + 1 <= hsz && heap[c + 1] < heap[c]) c++;
          if (heap[c] >= t) break;
          heap[i] = heap[c]; i = c;
      }
      heap[i] = t; return res;
  }
  // 最小值在 heap[1]（小顶堆）；改比较符号变最大堆
  ```
> 与 C++ 对照：`std::priority_queue` 写好了堆，C 要自己维护。这个模板背下来，TopK / Dijkstra / 合并 k 个链表都靠它。

---

## 4. 高频模板（可直接用）

```c
// 二分查找：在 [lo, hi) 中找第一个满足 ok 的位置
int lo = 0, hi = n;
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;
    if (ok(mid)) hi = mid; else lo = mid + 1;
}

// 快速幂（取模）
long long pow_mod(long long a, long long b, long long mod) {
    long long res = 1;
    while (b > 0) {
        if (b & 1) res = res * a % mod;
        a = a * a % mod; b >>= 1;
    }
    return res;
}

// 并查集（数组版，路径压缩）
int fa[100005];
int find(int x) { return fa[x] == x ? x : fa[x] = find(fa[x]); }
void union_(int a, int b) { fa[find(a)] = find(b); }
// 初始化：for (int i=0;i<n;i++) fa[i]=i;

// BFS（网格，队列用线性指针）
int dirs[4][2] = {{1,0},{-1,0},{0,1},{0,-1}};
int qx[100005], qy[100005], headq = 0, tailq = 0;
qx[tailq]=sx; qy[tailq++]=sy;
while (headq < tailq) {
    int x = qx[headq], y = qy[headq++];
    for (int k = 0; k < 4; k++) {
        int nx = x + dirs[k][0], ny = y + dirs[k][1];
        // 越界/访问判断，然后入队 qx[tailq]=nx; qy[tailq++]=ny;
    }
}

// 快速排序（直接用库函数）
#include <stdlib.h>
int cmp(const void *a, const void *b) {
    int x = *(const int*)a, y = *(const int*)b;
    return (x > y) - (x < y);        // <0 / 0 / >0，升序
}
qsort(arr, n, sizeof(int), cmp);
```

---

## 5. 排序与 C 标准库算法

- [ ] **`qsort`**：C 标准排序，传**比较函数指针**：
  ```c
  int cmp_asc(const void *a, const void *b) {
      int x = *(const int*)a, y = *(const int*)b;
      return x - y;                  // 注意：x-y 可能溢出大数，用 (x>y)-(x<y) 更稳
  }
  qsort(a, n, sizeof(int), cmp_asc);               // int
  ```
  - 结构体排序：`cmp` 里比较 `((const Node*)a)->key, ((const Node*)b)->key`
- [ ] **`bsearch`**：对已排序数组二分查找（返回 `void*` 或 `NULL`）。
- [ ] 稳定排序：`qsort` **不稳定**。需要稳定就改用计数排序/手写归并，或给元素附加值当第二关键字。
- [ ] 排序比较函数**不能只写 `return a - b`**（有符号溢出风险），用 `(a>b)-(a<b)` 形式最安全。

> 与 C++ 对照：`std::sort`（内省排序，快且稳）+ lambda → C 用 `qsort`（快排）+ 函数指针。性能相似，写法更原始；C 无 lambda，比较函数要单独定义。

---

## 6. 字符串处理

- [ ] **`<string.h>`**：C 的字符串是 `char[]` + 以 `\0` 结尾。
  - `strlen(s)`（长度）、`strcpy`/`strncpy`（拷贝）、`strcat`（连接）、`strcmp`（比较，返回 <0/=0/>0）
  - 比较相等：**`strcmp(s1, s2) == 0`**（不能直接 `==` 比较数组内容，那是在比地址）
- [ ] **`strtok`（按分隔符切分）**：
  ```c
  char s[] = "a,b,c";
  char *p = strtok(s, ",");
  while (p) { printf("%s\n", p); p = strtok(NULL, ","); }
  ```
- [ ] **查找**：`strchr`（找字符）、`strstr`（找子串，返回指针或 NULL）。
- [ ] **`sprintf` / `snprintf`**（格式化到字符串）与 **`sscanf`**（从字符串解析）：
  ```c
  snprintf(buf, sizeof(buf), "%d-%s", n, name);
  sscanf(str, "%d %s", &n, name);
  ```
- [ ] **数字 ↔ 字符串**：`atoi` / `atol`（字符串转数字，不检错）、`strtol`（可 `base` 进制）；转字符串用 `sprintf`。
- [ ] **字符判断**：`<ctype.h>` 的 `isalpha` / `isdigit` / `isspace` / `toupper` / `tolower`。

> 与 C++ 对照：`std::string` 的 `+`/`find`/`substr` 在 C 都用 `snprintf`/`strstr`/`strncpy` 手动做。C 字符串**常见坑**：忘加 `\0`、越界、`strcmp` 用 `==`。

---

## 7. 常用数学与技巧

- [ ] **最大公约数 gcd**：
  ```c
  int gcd(int a, int b) { return b ? gcd(b, a % b) : a; }
  ```
- [ ] **快速幂**（见 §4）。
- [ ] **位运算**（压状态 / 判奇偶 / 集合）：
  ```c
  x & (x-1)      // 清零最低的 1 位；x>0 时 x&(x-1)==0 即 x 是 2 的幂
  x & 1          // 判断奇偶
  (x >> n) & 1   // 取第 n 位
  __builtin_popcount(x)  // 统计 1 的个数（GCC/Clang）
  ```
- [ ] **二维数组**：`int a[N][M]`，遍历用 `a[i][j]`。注意 `sizeof` 用法：`memset(a, 0, sizeof(a))` 一整块清零（比循环快）。
- [ ] **`memset` / `memcpy`**：批量初始化/拷贝，速度极优：
  ```c
  memset(a, 0, sizeof(a));            // 清零（只对 0 和 -1 有效，因为按字节填）
  memcpy(dst, src, cnt * sizeof(int));
  ```
  - **坑**：`memset(a, 1, sizeof(a))` 会把每个字节填成 `1`，`int` 变成 `0x01010101`（不是 1）。要初始化成别的值用循环。
- [ ] **前缀和 / 差分**（数组版）：
  ```c
  int pre[N]; pre[0]=0;
  for (int i=1;i<=n;i++) pre[i]=pre[i-1]+a[i];   // a[1..n]
  int range = pre[r]-pre[l-1];                     // [l,r] 区间和
  ```
- [ ] **`long long`**：大数/和可能溢出时用 `long long`（`%lld`），中间乘先 `(long long)a * b`。

---

## 8. 常见易错点（before commit 必看）

1. **数组越界**：C **不报错**，越界写入是未定义行为（可能崩溃/改坏别的数据）。开大数组 + 检查边界。
2. **`memset` 只对 0/`-1` 靠谱**：填别的值按字节填，结果不是你以为的数。
3. **`strcmp == 0`**：字符串相等判断必须用 `strcmp`，不是 `==`（那是在比较地址）。
4. **`sizeof` vs 数组名**：在 `main` 里 `sizeof(a)` 是整个数组；但**形参传进来后 `sizeof(a)` 是指针大小**（8 字节），要用 `sizeof` 算就得把长度也传进来。
5. **`scanf` 读字符**：`%c` 会**读取换行/空格**，注意 `scanf(" %c", &c)`（前加空格跳过空白）。
6. **负数取模**：C 的 `%` 向零截断（同 C++），`-7 % 3 == -1`。需要正结果时 `((x % mod) + mod) % mod`。
7. **比较函数溢出**：`qsort` 比较函数别写 `a - b`，用 `(a>b)-(a<b)`。
8. **`long long` 用 `%lld`**：`%d` 读 `long long` 会错。
9. **递归深度**：DFS 递归太深会**栈溢出**；大输入用 `BFS`/迭代，或看平台是否放宽栈。
10. **动态内存 `malloc`**：C 刷题**几乎不用** `malloc`/`free`（直接用数组），用到了记得 `free`，否则内存泄漏（OJ 通常不看重，但好习惯要养）。
11. **全局变量默认清零**：全局数组自动初始化为 0，**比局部省一次 memset**。刷题把大数组放全局。
12. **`scanf` 返回值**：要判断是否读到——`while (scanf("%d", &x) != EOF)` 处理不定行输入。

## 9. C++ → C 对照速查

| C++ | C |
|---|---|
| `vector<int> v` | `int v[N]; int len` |
| `v.push_back(x)` | `v[len++] = x` |
| `stack<int> s` | `int st[N]; int top` |
| `queue<int> q` | `int q[N]; int head, tail` |
| `unordered_map<k,v>` | 值域数组 / 手写哈希 / 排序+二分 |
| `priority_queue<int>` | 手写二叉堆 |
| `sort(b,e)` | `qsort(a, n, sizeof(int), cmp)` |
| `cin >> n` | `scanf("%d", &n)` |
| `cout << x` | `printf("%d\n", x)` |
| `string s` | `char s[N]`（+ `<string.h>`） |
| `std::gcd` | `gcd()` 手写 |
| `n % mod` | `((n % mod) + mod) % mod`（负数） |
| `memset(a,0,sizeof)` | C 里也有 `memset`（同） |

---

## 学习路径建议

| 阶段 | 内容 | 目标 | 状态 |
|---|---|---|---|
| 1 | 输入输出 | 会 `scanf/printf` + 手写快读 | ☐ |
| 2 | 容器手写 | 数组模拟 vector/stack/queue/hash/堆 | ☐ |
| 3 | 标准库算法 | `qsort` / `bsearch` / `memset` | ☐ |
| 4 | 高频模板 | 二分/快速幂/并查集/BFS/前缀和 | ☐ |
| 5 | 字符串 | `<string.h>` / `strtok` / `snprintf` | ☐ |
| 6 | 实战 | 用 C 刷 LeetCode 热题 50 道 | ☐ |

---

## 待填充内容（后续逐节展开）

1. 每节详细讲解 + 完整可运行示例
2. 手写哈希 / 手写堆的完整带注释版本
3. 常见题型的 C 模板（链表反转、二叉树遍历、字符串处理、DP 一维/二维）
4. C 版 Dijkstra / 拓扑排序 / 滑动窗口模板
5. 易错点对应的反例与调试技巧

> 告诉我"从第 X 节开始填充"或"填充某节"，我会按框架逐节写内容。
