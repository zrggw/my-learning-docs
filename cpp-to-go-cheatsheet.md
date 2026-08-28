# C++ → Go 算法题快速转换指南

> 面向熟悉 C++ 刷题、切换到 Go 的选手。只讲刷题需要的差异，不讲语言细节。

## 1. 心智模型转变

| C++ 习惯 | Go 的对应 |
|---|---|
| 类 / 继承 / 多态 | 没有类。用 `struct` + 方法（`func (s *T) M()`） |
| 模板 template | 泛型 `[T any]`（Go 1.18+），但刷题基本用不上 |
| 异常 try/catch | 没有异常。用多返回值 + error |
| 函数重载 / 默认参数 | 都不支持。不同签名就起不同名字 |
| 引用传参 `&` | 值传递；需要改调用方就传指针 `*T`；slice/map 天然"引用头" |
| while / do-while | 只有 `for`：`for cond {}` 就是 while |
| 三元运算符 `a ? b : c` | 不支持，写 if/else |
| `std::pair` / 出参 | 多返回值 `a, b := f()` 直接替代 |
| UB（越界、未初始化） | 越界会 panic，但别依赖；零值初始化安全 |

**核心思想：Go 里"一切皆值"，slice/map/string 是引用语义的"头"，底层数据共享。**

## 2. 容器对照表（最重要）

| 场景 | C++ | Go |
|---|---|---|
| 动态数组 | `std::vector<T>` | `[]T` + `append` |
| 定长数组 | `std::array<T,N>` | `[N]T` |
| 哈希表 | `std::unordered_map<K,V>` | `map[K]V` |
| 有序映射 | `std::map<K,V>`（红黑树） | 没有内置，需要时手动 `sort` 键 |
| 集合 | `std::unordered_set<T>` | `map[T]struct{}`（用 map 模拟） |
| 栈 | `std::stack<T>` | `[]T`（append / 截断） |
| 队列 | `std::queue<T>` | `[]T` + 头指针（避免 `q = q[1:]` 拷贝） |
| 双端队列 | `std::deque<T>` | `container/list`（刷题极少用） |
| 优先队列 | `std::priority_queue<T>` | `container/heap`（实现接口） |
| 字符串 | `std::string` | `string`（不可变）+ `strings.Builder` |
| 位集 | `std::bitset<N>` | `uint64` 位运算手写 |
| 链表 | `std::list<T>` | `container/list`（刷题几乎不用，一般用数组模拟） |

**注意**：Go 标准库没有现成的 set / stack / queue / priority_queue 容器，栈和队列用切片模拟，集合用 map 模拟，堆要写一次 `heap.Interface`（模板见 §4）。

## 3. 常用容器速查

```go
// 切片 = vector
s := make([]int, n)        // 长度 n，零值
s := make([]int, 0, cap)   // 空，预分配容量
s = append(s, x)           // 尾插（必须接收返回值！）
s = s[:len(s)-1]           // 尾删（栈 pop）
s = append(s[:i], s[i+1:]...) // 删中间（O(n)，少用）
copy(dst, src)             // 深拷贝
// 切片 s[a:b]：左闭右开，共享底层数组

// map = unordered_map
m := make(map[K]V)
v, ok := m[k]              // 判断存在（C++ 的 count/find）
delete(m, k)
// 遍历顺序随机！不要依赖顺序

// 集合 = map[T]struct{}
set := map[int]struct{}{}
set[x] = struct{}{}
_, ok := set[x]

// 栈：切片即可
st = append(st, x)         // push
x := st[len(st)-1]; st = st[:len(st)-1]  // top + pop

// 队列：切片 + 头指针（BFS 性能关键）
q := make([]int, 0)
q = append(q, x)           // push
x := q[head]; head++       // pop（head 只增不减，避免反复拷贝）
```

## 4. 高频模板（可直接背）

```go
// 二分查找：在 [lo, hi) 中找第一个满足 ok 的位置
lo, hi := 0, n
for lo < hi {
    mid := lo + (hi-lo)/2
    if ok(mid) { hi = mid } else { lo = mid + 1 }
}
// 等价内置：sort.Search(n, ok)

// 最小堆（container/heap 只需实现一次）
type MinHeap []int
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] }  // 改 > 变最大堆
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x any)        { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() any {
    old := *h; n := len(old); x := old[n-1]; *h = old[:n-1]; return x
}
// 用法：h := &MinHeap{}; heap.Init(h); heap.Push(h, 3); x := heap.Pop(h).(int)

// 并查集
type DSU struct{ p []int }
func NewDSU(n int) *DSU {
    p := make([]int, n)
    for i := range p { p[i] = i }
    return &DSU{p}
}
func (d *DSU) Find(x int) int {
    if d.p[x] != x { d.p[x] = d.Find(d.p[x]) }  // 路径压缩
    return d.p[x]
}
func (d *DSU) Union(a, b int) { d.p[d.Find(a)] = d.Find(b) }

// BFS（网格题，队列用头指针）
dirs := [][2]int{{1,0},{-1,0},{0,1},{0,-1}}
q := [][2]int{{0,0}}   // 或拆成两个 []int 队列更省内存
for head := 0; head < len(q); head++ {
    cur := q[head]
    for _, d := range dirs {
        nx, ny := cur[0]+d[0], cur[1]+d[1]
        // 越界检查、入队 q = append(q, [2]int{nx, ny})
    }
}

// 快速幂
func pow(a, b, mod int64) int64 {
    res := int64(1)
    for b > 0 {
        if b&1 == 1 { res = res * a % mod }
        a = a * a % mod
        b >>= 1
    }
    return res
}
```

## 5. 字符串处理

- `string` 不可变；`s[i]` 是 **byte**；要按字符遍历用 `for i, r := range s`（自动解码 UTF-8）
- 中文按字处理：`rs := []rune(s)`，处理完 `string(rs)` 转回
- 拼接：`strings.Builder`（循环里别用 `+=`，O(n²)）
- 常用：`strings.Split(s, sep)`、`strings.Join(a, sep)`、`strings.Contains`、`strings.HasPrefix`、`strings.TrimSpace`、`strings.ToLower`
- 转换：`strconv.Atoi(s)` / `strconv.Itoa(n)` / `strconv.ParseInt(s, 10, 64)`
- 格式化：`fmt.Sprintf` 相当于 `snprintf`

## 6. 排序与数学

```go
sort.Ints(s)                       // 升序
sort.Slice(s, func(i, j int) bool { return s[i] > s[j] })  // 自定义（降序）
sort.Search(n, func(i int) bool { ... })  // 二分
// Go 1.21+ 还有 slices 包：slices.Sort / slices.SortFunc / slices.BinarySearch

min(a, b) / max(a, b)              // 内置函数（Go 1.21+，LeetCode 可用）
math.MaxInt64, math.MinInt64       // 极值（int 范围）
// int 绝对值：a < 0 手写取反（math.Abs 只接受 float）
```

## 7. 输入输出

```go
// 简单场景（数据少）
fmt.Scan(&n)                       // 相当于 cin >> n

// 快读快写（数据量大时用，比赛必备）
sc := bufio.NewScanner(os.Stdin)
sc.Buffer(make([]byte, 1<<20), 1<<20)  // 行长 >64KB 时扩容
for sc.Scan() {
    line := sc.Text()              // 拿一行，再 strconv 拆数字
}
out := bufio.NewWriter(os.Stdout)
defer out.Flush()
fmt.Fprintln(out, ans)
```

**LeetCode 上**：不用写 `main` 和 IO，按给的函数签名实现即可，返回切片/值就行。

## 8. 易错点清单

1. `append` 必须接收返回值：`s = append(s, x)`，漏了 = 是经典 bug
2. 切片共享底层数组：`a := s[:2]` 后改 `a` 会影响 `s`；要独立就 `copy` 或 `append([]T{}, s...)`
3. map 遍历顺序**随机**，需要有序就收集键后 `sort`
4. 字符串不可变：`s[i] = 'x'` 编译错误，先转 `[]byte`
5. `for i, v := range s` 中 `v` 是**拷贝**，改 `v` 不影响原切片；要改用 `s[i]`
6. 整数溢出静默回绕（不会像 C++ 那样是 UB，但结果同样错），注意 `int` 在 64 位机上是 64 位
7. 负数取模向零截断，和 C++ 一致
8. `map` 读不存在的键返回零值（`int` 是 0），用 `v, ok := m[k]` 区分
9. `switch` 默认自动 break；要穿透写 `fallthrough`
10. `++` 是语句不是表达式：`i++` 可以，`a = i++` 不行

## 9. C++ → Go 对照示例（两数之和）

```cpp
// C++
vector<int> twoSum(vector<int>& nums, int target) {
    unordered_map<int,int> mp;
    for (int i = 0; i < nums.size(); ++i) {
        if (mp.count(target - nums[i]))
            return {mp[target-nums[i]], i};
        mp[nums[i]] = i;
    }
    return {};
}
```

```go
// Go —— 多返回值 + map 双值查找，几乎一一对应
func twoSum(nums []int, target int) []int {
    mp := map[int]int{}
    for i, x := range nums {
        if j, ok := mp[target-x]; ok {
            return []int{j, i}
        }
        mp[x] = i
    }
    return nil
}
```

## 10. 刷题速查：C++ 语句 → Go 语句

| C++ | Go |
|---|---|
| `int a[100]` | `a := make([]int, 100)` |
| `for (int i=0; i<n; ++i)` | `for i := 0; i < n; i++` |
| `for (auto x : v)` | `for _, x := range v` |
| `while (cond)` | `for cond {}` |
| `sort(v.begin(), v.end())` | `sort.Ints(v)` |
| `v.push_back(x)` | `v = append(v, x)` |
| `mp.find(k) != mp.end()` | `_, ok := mp[k]` |
| `return {a, b}` | `return []int{a, b}` 或 `return a, b` |
| `st.top(); st.pop()` | `x := st[len(st)-1]; st = st[:len(st)-1]` |
| `cout << x << endl` | `fmt.Println(x)` |
