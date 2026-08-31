# C 语言高级内容学习指南（C++ 背景版）

> 用途：C 语言系统进阶路线图，**面向熟悉 C++ 的读者**，聚焦"C 比 C++ 更原始、更贴近底层、更易踩坑"的部分。
> 使用方式：每学完一个小点，把 `[ ]` 改成 `[x]`；有疑问或要展开某节，随时让我补充内容。
> 定位：不讲入门语法（`printf`、`for`、数组这些默认你会），专讲**指针、内存、宏、链接、标准库底层、未定义行为**这类"高阶内容"。
> 配套：本文档解决"学什么、按什么顺序学、和 C++ 有啥区别"。

---

## 阶段 1：指针进阶（C 的"命门"）

> 覆盖：指针的本质、多级指针、const、void*、数组与指针、函数指针、算术与陷阱。这是 C 进阶必须彻底吃透的一节。

- [ ] **指针的本质**：一个"值为另变量地址 + 类型信息"的变量。类型决定解引用时的**步长与解释方式**，地址本身不带类型。
  ```c
  int  a = 10;        // 占 4 字节
  int *p = &a;        // p 存 a 的地址，类型 int*：*p 读出 4 字节当 int
  char *q = (char*)&a; // 同一地址，按 char 解读：*q 只读出第 1 字节
  ```
- [ ] **指针 vs 引用（C++ 对照）**：C 没有引用。引用是"隐式解引用的别名"，指针是"显式保存地址的值"。所以：
  - 引用非空、不可重新绑定；指针可为 `NULL`、可随意改指向
  - C 里要"改调用方变量"只能传指针：`void swap(int *a, int *b)`，调用 `swap(&x, &y)`
- [ ] **多级指针（指向指针的指针）**：需要修改"一个指针变量本身"时用 `T**`。
  ```c
  int x = 1, *p = &x, **pp = &p;
  printf("%d", **pp);   // 1：pp -> p -> x
  // 经典用途：函数里给调用方分配一个指针指向的缓冲区
  void alloc(int **out, size_t n) { *out = malloc(n * sizeof(int)); }
  ```
- [ ] **const 指针三态（经典考点）**——"const 修饰谁，谁不可变"：
  | 写法 | 含义 | 例子 |
  |---|---|---|
  | `const int *p` / `int const *p` | 指向**常量**的指针：`*p = 5` 非法，`p = ...` 合法 | `const int *p = &a;` |
  | `int *const p` | 指针本身是常量：`p = ...` 非法，`*p = 5` 合法 | `int *const p = &a;` |
  | `const int *const p` | 两者都不可改 | — |
  > 记忆：`const` 在 `*` 左边修饰**指向的值**，在 `*` 右边修饰**指针本身**。
- [ ] **void\*（泛型指针）**：无类型指针，可接收任意数据指针，但不能直接解引用/算术（编译器不知道步长），用前需强转。
  - `malloc` 返回 `void*`，C 里**不需要**强转（C++ 需要）；`malloc` 返回的内存**未初始化**，建议用 `calloc` 或手动清零
- [ ] **数组与指针的关系（数组衰减）**：
  ```c
  int a[5];
  int *p = a;        // 数组名在多数表达式里"衰减"为首元素地址
  // a 与 &a 类型不同：a 是 int*（指向元素），&a 是 int(*)[5]（指向整个数组）
  printf("%zu %zu", sizeof(a), sizeof(p));  // 20 vs 8（sizeof 数组返回整个大小，不衰减）
  ```
- [ ] **函数指针**：保存函数入口地址，用于回调/分发表（详见阶段 4）。
- [ ] **指针算术**：`p + n` 实际跨 `n * sizeof(*p)` 字节。`sizeof(char)==1`，所以 `char*` 算术最"直观"，`int*` 加 1 跳 4 字节。
- [ ] **野指针 / 悬垂指针 / 空指针**——三大雷区：
  | 类型 | 定义 | 后果 |
  |---|---|---|
  | 未初始化指针 | `int *p;`（未赋值） | 值是垃圾，解引用=UB |
  | 悬垂指针 | 指向已 `free`/已失效的内存 | use-after-free=UB |
  | 空指针 | `NULL`（`(void*)0`） | 解引用=UB；判空 `if (p != NULL)` |
  ```c
  int *p = malloc(sizeof(int));
  *p = 1; free(p);
  // p 现在是悬垂指针：p 仍保存旧地址，但内存已还给系统
  p = NULL;   // 好习惯：free 后置空，避免双释放/误用
  ```

> 与 C++ 对照：指针/引用/智能指针三件套 → C 只有裸指针 + 手动生命周期管理。**RAII 在 C 里不存在，一切手动**。

---

## 阶段 2：内存管理（手动生命周期的真功夫）

- [ ] **内存布局（进程地址空间，从低到高）**：
  | 区域 | 内容 | 特点 |
  |---|---|---|
  | 代码段 (.text) | 编译后的机器码 | 只读 |
  | 数据段 (.data) | 已初始化全局/静态变量 | 读写 |
  | BSS (.bss) | 未初始化全局/静态变量 | 程序加载时自动置零 |
  | 堆 (heap) | `malloc` 动态分配 | 从低地址向高地址增长，手动管理 |
  | 栈 (stack) | 局部变量、函数调用帧 | 从高地址向低地址增长，自动管理 |
- [ ] **malloc / calloc / realloc / free**：
  ```c
  int *p = malloc(10 * sizeof(int));      // 未初始化
  int *q = calloc(10, sizeof(int));       // 自动清零（calloc 更适合初始化为 0）
  q = realloc(q, 20 * sizeof(int));       // 扩容（可能整块搬移，返回值必须接收！）
  free(p); free(q);
  ```
  - **`realloc` 必须用返回值**：扩容失败返回 `NULL` 且**原内存仍在**，直接覆盖原指针会丢；用临时变量接：
    ```c
    int *tmp = realloc(p, newsize);
    if (tmp) { p = tmp; } else { /* 处理失败，p 仍有效 */ }
    ```
- [ ] **三大内存错误（必考）**：
  | 错误 | 表现 | 检测工具 |
  |---|---|---|
  | 内存泄漏 | 分配了没 free，进程常驻时越堆越大、OOM | valgrind / ASan / LeakSanitizer |
  | 双重释放 | 对同一块 `free` 两次 → 堆损坏/崩溃 | ASan / glibc 检测 |
  | use-after-free | 释放后仍读写 → 未定义行为 | ASan |
  ```c
  // 泄漏：每次循环泄漏一块
  for (int i = 0; i < 100; i++) { char *s = malloc(64); /* 没 free(s) */ }
  ```
- [ ] **柔性数组**（struct 尾部不完整数组，用于"一条 malloc 装可变长数据"的高频技巧）：
  ```c
  typedef struct { int len; char data[]; } Buffer;   // data 是柔性数组（C99）
  Buffer *b = malloc(sizeof(Buffer) + 100);
  b->len = 100;
  free(b);  // 一次 free 即可（data 不单独分配/释放）
  ```
- [ ] **对齐（alignment）**：内存按类型对齐（`int` 通常 4 字节对齐）利于 CPU 高效访问。
  - `_Alignof(T)` 查询对齐（C11）、`_Alignas(n)` 指定对齐、`offsetof(struct, member)` 查字段偏移
  - 跨结构体/网络序列化时，填充字节（padding）会导致"sizeof ≠ 字段和"，是序列化常见坑

> 与 C++ 对照：C 的 `malloc/new`、`free/delete` 完全分离，无构造/析构概念。C++ 的 `new/delete` 会调构造/析构；C 的 `malloc` 不初始化对象，`free` 不清理。

---

## 阶段 3：结构体、联合体与位域

- [ ] **struct 内存对齐与填充**：编译器在字段间插填充字节以满足对齐，所以 `sizeof(struct)` 往往大于字段大小之和。
  ```c
  struct S { char c; int i; };   // c 后填充 3 字节对齐 i -> sizeof = 8
  ```
  - 调字段顺序省内存：把成员按对齐大小降序排。取消填充：`#pragma pack(1)` / `__attribute__((packed))`（用时要接受未对齐访问性能代价）
- [ ] **struct 指针 vs 值传递**：大结构体用 `struct S *` 传参避免拷贝；`p->m` 是 `(*p).m` 的语法糖。
- [ ] **位域（bit-field）**——按位取内存：
  ```c
  struct Flags { unsigned int a:1, b:1, c:1, pad:29; };
  ```
  适合寄存器/协议头/节省空间；但顺序、宽度因编译器而异，**跨平台不可靠**。
- [ ] **union（联合体）与类型双关**：所有成员共享同一起始地址，大小取最大成员。常用于"内存复用"或序列化：
  ```c
  union U { int i; float f; unsigned char bytes[4]; };
  union U u; u.i = 0x3f800000;
  printf("%f", u.f);   // 用同一块内存的不同解读（类型双关）
  ```
  > 注意：读取非活跃成员在严格别名下是 UB（`-fstrict-aliasing` 默认开）；跨字节序时**大小端**会反过来，别拿 union 在异机间传数据。
- [ ] **typedef vs #define**：`typedef` 定义类型别名（编译期、有作用域、不破坏词法）；`#define` 是纯文本替换（无作用域、易出坑）。给指针起别名时 `typedef int* P;` 的 `P a, b;` 只有 `a` 是指针——见阶段 5 的宏坑。
- [ ] **匿名 struct/union（C11）**：嵌套时省去标签名：
  ```c
  struct Outer { int x; union { int i; float f; }; };  // 匿名联合体
  ```

> 与 C++ 对照：C++ 的 struct 默认成员是 public 且有方法/继承；C 的 struct 纯数据。C 没有 class，只有 struct + 函数指针模拟"对象"。

---

## 阶段 4：函数指针与回调（C 的"多态"）

- [ ] **函数指针语法与"右左法则"**：声明从变量名出发，先右后左读。
  ```c
  int (*fp)(int, int);    // fp 是指向"返回 int、收两个 int"函数的指针
  fp = add;               // 赋值（函数名 add 即地址）
  int r = fp(1, 2);       // 或 (*fp)(1, 2)
  ```
- [ ] **解读复杂声明**：`int *(*fp[10])(double)` —— `fp` 是 10 个元素的数组，元素是指向"返回 int*、收 double"函数的指针。用 `cdecl` 工具可反向解析。
- [ ] **回调函数（callback）**：把函数指针作为参数传入，被调方在合适时机调用。
  ```c
  int cmp_int(const void *a, const void *b) {
      int x = *(const int*)a, y = *(const int*)b;
      return (x > y) - (x < y);   // 返回 <0 / 0 / >0
  }
  int arr[] = {3,1,2};
  qsort(arr, 3, sizeof(int), cmp_int);   // 标准库 qsort 收回调
  ```
- [ ] **函数指针数组（命令/分发表）**：
  ```c
  int (*op[])(int,int) = { add, sub, mul, div };
  int r = op[2](4, 2);   // 按索引分配函数
  ```
- [ ] **返回函数指针的函数**、**把函数指针封进 struct 模拟"成员函数/对象"**：
  ```c
  typedef struct { int (*open)(const char*); void (*close)(void); } Device;
  ```
- [ ] **回调与函数对象的对比（C++ 对照）**：C 的回调 = 裸函数指针 + `void*` 传上下文；C++ 的 `std::function`/`lambda`/functior 更灵活（可捕获状态、有类型安全），但 C 的 `void*` 传上下文是唯一选择：
  ```c
  typedef void (*cb)(void *ctx, int val);
  void run(cb f, void *ctx) { for (...) f(ctx, i); }
  ```

> 与 C++ 对照：这就是 C 版的"多态/策略模式"——没有虚函数表，用函数指针或函数指针数组手动实现（很多 C 库/内核就这么干）。

---

## 阶段 5：宏与预处理（强大又危险）

- [ ] **`#define` 函数式宏**：文本替换，**注意括号**，否则优先序出坑：
  ```c
  #define SQUARE(x) ((x)*(x))     // 外层再套一层括号最稳
  #define MAX(a,b) ((a)>(b)?(a):(b))
  ```
  ```c
  SQUARE(x)      // 若写 #define SQ(x) x*x，则 SQ(1+2) -> 1+2*1+2=5，错！必须 ((x)*(x))
  ```
- [ ] **字符串化 `#` 与 连接 `##`**：
  ```c
  #define STR(x) #x        // # 把参数变字符串字面量：STR(hello) -> "hello"
  #define CAT(a,b) a##b    // ## 把两记号粘成一个：CAT(foo,bar) -> foobar
  #define XSTR(x) #x       // 两层包装可先展开再字符串化（否则参数不会展开）
  ```
- [ ] **可变参数宏**：
  ```c
  #define LOG(fmt, ...) printf(fmt, __VA_ARGS__)
  #define ERR(fmt, ...) fprintf(stderr, fmt, ##__VA_ARGS__)  // ## 处理零参数（GNU 扩展）
  // C23 起支持 __VA_OPT__(...)，零参数更干净： fprintf(stderr, fmt __VA_OPT__(,) __VA_ARGS__)
  ```
- [ ] **条件编译**：`#if`/`#ifdef`/`#ifndef`/`#else`/`#elif`/`#endif`；头文件防重复包含用 `#pragma once`（简单）或 include guard（`#ifndef X_H` ... `#endif`，可移植）。`#if` 里只能判断宏/**常量**表达式，不能 `sizeof`（`#if` 阶段还不认识类型）。
- [ ] **常见宏坑**：
  | 坑 | 例子 | 建议 |
  |---|---|---|
  | 未加括号 | `#define M(a,b) a*b` → `M(1+2,3)` 变 `1+2*3` | 参数和整体都加括号 |
  | 参数多重求值 | `MAX(i++, j)` 中 `i` 被求值两次 | 改用内联函数/`typeof`（GNU） |
  | 无作用域/类型 | 宏没有类型检查 | 优先用 `enum`/`const`/`static inline` 代替 |
  | 字符串化参数不展开 | `#define S(x) #x; S(__LINE__)` | 用 `#define XS(x) #x` + `XS(__LINE__)` |

> **现代 C 建议**：能用 `const`/`enum`/`static inline` 就别用宏。C++ 还有 `constexpr`/`template`/`inline`，比宏安全得多。

---

## 阶段 6：可变参数（`<stdarg.h>`）

- [ ] **va_list / va_start / va_arg / va_end**——实现类似 `printf` 的可变参数函数：
  ```c
  #include <stdarg.h>
  int sum(int count, ...) {
      va_list ap; va_start(ap, count);
      int s = 0;
      for (int i = 0; i < count; i++) s += va_arg(ap, int);
      va_end(ap); return s;
  }
  ```
  - **必须**：`va_start` 依赖最后一个**固定**参数（`count`）定位可变区开头；用完必调 `va_end`
  - `va_arg` 不检查类型，传错类型（如 `double` 读成 `int`）= UB；**没有内建机制告知参数个数/类型**，通常靠固定参数（如 `count`）或格式串（如 `%d`）
- [ ] **转发到其他可变函数**：直接转必须二次封装：
  ```c
  void my_log(const char *fmt, ...) { va_list ap; va_start(ap, fmt);
      vfprintf(stderr, fmt, ap); va_end(ap); }   // 用 v 版（vprintf/vfprintf/vsnprintf）转发
  ```
- [ ] **`vsnprintf`**：安全格式化（指定缓冲区大小），是许多日志库的底层。

> 与 C++ 对照：C++ 的模板/`std::variant`/`std::initializer_list` 基本取代了 C 的可变参数，但学习它有助于理解 `printf` 族。

---

## 阶段 7：位运算技巧

- [ ] 基础：`&`（与）、`|`（或）、`^`（异或）、`~`（取反）、`<<`/`>>`（移位）。适合标志位集合、权限位掩码、哈希、高效运算。
- [ ] **通用掩码操作**：
  ```c
  unsigned x = 0;
  x |=  1 << 3;              // 置位第 3 位
  x &= ~(1 << 3);            // 清零第 3 位（先取反再与）
  x ^=  1 << 3;              // 翻转第 3 位
  int k = (x >> 3) & 1;      // 读取第 3 位
  ```
- [ ] **高频位技巧（背下来）**：
  ```c
  x & (x-1)        // 清零最低的 1 位（用于判 2 的幂：x>0 时 x&(x-1)==0）
  x & -x           // 取最低位的 1（lowbit）
  x & 1            // 判断奇偶
  x ^ y            // 无临时变量交换（了解即可，现代编译优化下无实际优势）
  ~x + 1           // 取相反数（补码）
  (x >> n) & 1     // 取第 n 位
  ```
- [ ] **位计数（popcount）**：`__builtin_popcount(x)`（GCC/Clang）、`__builtin_ctz`（末尾连续 0 个数）。C23 有 `stdc_count_ones`。
- [ ] **位运算陷阱**：
  - 移位位数 >= 类型位宽是 UB：`x << 32`（int）非法，先转 `unsigned long long` 或用 64 位
  - 有符号负数右移是**实现定义**（大多算术右移），补码环境下别依赖
  - 负数 << 是 UB；`~0` 是 -1（全 1），`~0U` 才是全 1 无符号

> 与 C++ 对照：位运算规则几乎一致；但 C++ 有 `std::bitset`/`std::bit_cast`（C++20），C 主要靠裸位运算。

---

## 阶段 8：C11 / C17 / C23 新特性（现代 C）

- [ ] **`_Static_assert`（编译期断言）**：
  ```c
  _Static_assert(sizeof(int) == 4, "int 必须是 4 字节");
  ```
- [ ] **`_Generic`（泛型选择，C11）**——C 版"模板/重载"，按类型选择表达式：
  ```c
  #define type_name(x) _Generic((x), \
      int: "int", float: "float", default: "other")
  ```
- [ ] **`_Noreturn` / `[[noreturn]]`**：声明函数不返回（如 `exit`、`panic`），利于编译器优化与警告。
- [ ] **`restrict`**：告诉编译器该指针是访问某内存的唯一途径，便于优化；**滥用导致 UB**。
- [ ] **复合字面量 & 指定初始化器（C99）**：
  ```c
  struct Point p = { .x = 1, .y = 2 };        // 指定初始化器，省去顺序记忆
  int arr[] = { [0]=10, [2]=30 };             // 用索引指定元素
  printf("%d", ((struct Point){.x=3,.y=4}).x); // 复合字面量，临时对象
  ```
- [ ] **`_Alignof` / `_Alignas`**：类型对齐查询/指定（见阶段 2）。
- [ ] **C11 线程与原子**：
  - `<threads.h>`：`thrd_create`/`thrd_join`/`mtx_t` 互斥锁/`thrd_sleep` 等
  - `_Atomic`：原子类型与原子操作（比信号量/自旋更底层）
  - 注意：C 的线程库较原始，**实际项目大多用 pthread（POSIX）或编译器内置原子**；`<stdatomic.h>` 提供 `atomic_load`/`atomic_store`/`atomic_fetch_add` 等
- [ ] **C23 概览（了解）**：
  - `nullptr` 关键字、`typeof`/`typeof_unqual`、`constexpr`
  - `[[noreturn]]`/`[[deprecated]]` 标准化属性
  - `#embed`（嵌资源）、`__VA_OPT__`
  - 十六进制浮点、`stdc_` 位函数族
  - `bool`/`true`/`false` 成为关键字（`<stdbool.h>` 不再必需）

> 与 C++ 对照：C 的新特性是"补 C++ 早有的能力"，但 C 没有类/继承/模板/异常/RAII/智能指针/STL。**学会识界**：C 更贴近硬件与 ABI，C++ 更侧重抽象与安全。

---

## 阶段 9：编译、链接与库

- [ ] **编译单元 / 翻译单元**：一个 `.c` → `.o` 是独立翻译单元，各自拥有类型检查，链接时合并。头文件只是"复制粘贴进源文件"的文本。
- [ ] **外部链接 vs 内部链接**：
  - `static` 修饰全局变量/函数 → 内部链接（本文件可见，避免命名冲突）
  - 缺省全局符号 → 外部链接（跨文件可见，可被 `extern` 引用）
  - `const` 全局变量默认**内部链接**（C 与 C++ 行为不同，C++ 的 const 默认外部链接）——经典坑
- [ ] **`static inline` 函数**：放头文件里，每个引用处内联展开且不产生重复符号（解决"头文件函数多定义"问题）。
- [ ] **静态库 `.a` / 动态库 `.so` / `.dll`**：
  - 静态库：编译期把 `.o` 打包，链接时拷入可执行文件，产物大、独立运行
  - 动态库：运行时 `dlopen`/`LoadLibrary` 加载，产物小、可共享、热更新；链接期加 `-lxxx -L/path`，运行期需 `LD_LIBRARY_PATH`/`PATH`
- [ ] **`extern` 声明**：`extern int counter;` 声明变量存在但**不分配存储**，配合某 `.c` 里的定义使用。
- [ ] **include guard / `#pragma once`**：防止头文件被重复包含（见阶段 5）。
- [ ] **关键编译步骤**（了解即可）：预处理（宏展开）→ 编译（生成汇编/目标码）→ 汇编 → 链接（合并符号、解析地址、生成可执行/库）。

> 与 C++ 对照：链接语义大体相同；但 C++ 有**名字修饰（name mangling）**、`extern "C"`（禁修饰，供 C 调用）、模板的隐式实例化等，比 C 复杂。

---

## 阶段 10：未定义行为（UB）与安全

- [ ] **UB（Undefined Behavior）**：C 标准未规定结果的行为，编译器可任意处理，**可能正常、可能崩溃、可能优化成错**。
- [ ] **常见 UB 清单（务必牢记）**：
  | 操作 | 是否 UB | 说明 |
  |---|---|---|
  | 解引用空指针/野指针 | UB | 段错误或更糟 |
  | 越界访问数组 | UB | 无边界检查 |
  | 有符号整数溢出 | UB | `INT_MAX + 1` |
  | 除零 | UB | 整数除零 |
  | 修改 `const`/只读对象 | UB | |
  | 未初始化变量读取 | UB | |
  | 同一表达式内多次修改同一变量 | UB | 序列点问题，`i = i++` |
  | 用 `va_arg` 读错类型 | UB | |
  | 位运算：有符号负数移位/溢出 | UB 或实现定义 | |
- [ ] **序列点与求值顺序**：`f() + g()` 的求值顺序是**未规定**；`i = i++ + i` 这类多修改同一对象是 UB。别依赖求值顺序，拆成单独语句。
- [ ] **严格别名规则**：通过不兼容类型的指针访问对象大多 UB（`-fstrict-aliasing` 默认开）；`char*` 例外（可访问任何对象）。跨类型解读尽量用 union/memcpy。
- [ ] **内存安全三件套**：越界读/写、use-after-free、内存泄漏——**没有 GC，全靠纪律 + 工具**。

> 与 C++ 对照：UB 概念相同，但 C++ 还有**对象生命周期**、虚表、异常安全等额外 UB 维度。总体理解：**C 让你自己负责一切，C++ 给你更多工具但也要更多规则。**

---

## 阶段 11：工具链与调试

- [ ] **常用编译选项**：
  ```bash
  gcc -std=c11 -Wall -Wextra -Wpedantic -O2 -g -c foo.c   # 严格告警 + 优化 + 调试符号
  gcc -fsanitize=address -g foo.c -o foo                  # AddressSanitizer 内存错误检测
  gcc -fsanitize=undefined -g foo.c -o foo                # UndefinedBehaviorSanitizer
  clang --analyze foo.c                                   # 静态分析
  ```
- [ ] **`-Wall -Wextra -Werror`**：把常见问题变成警告甚至错误，**开发期就该开满**。
- [ ] **gdb 基本命令**：`gdb ./a.out` → `break main` → `run` → `print x` / `bt`（调用栈）/ `next` / `step` / `watch var` / `list`。
  - `-g` 参数**必须**才能看到源码/变量名。
- [ ] **Sanitizers（最实用）**：`-fsanitize=address`（泄漏/越界/use-after-free）、`-fsanitize=undefined`（溢出/除零/null）、`-fsanitize=thread`（数据竞争）。
- [ ] **Valgrind**：`valgrind --leak-check=full ./a.out`，Chrome/内核之外的跨平台内存检测主力（比 ASan 慢但更通用）。
- [ ] **性能分析**：`perf`（Linux）、`gprof`；`-O2`/`-O3`/`-march=native` 等优化影响很大。

> 与 C++ 对照：工具链基本共享（都是 gcc/clang/gdb）；C 没有 RAII 和异常，所以资源和错误处理全部自己管，调试时尤其依赖 sanitizer 抓泄漏和 UB。

---

## 学习进度总表

| 阶段 | 内容 | 预计投入 | 状态 |
|---|---|---|---|
| 1 | 指针进阶 | 2-3 天 | ☐ |
| 2 | 内存管理 | 2 天 | ☐ |
| 3 | 结构体、联合体与位域 | 1-2 天 | ☐ |
| 4 | 函数指针与回调 | 1-2 天 | ☐ |
| 5 | 宏与预处理 | 1-2 天 | ☐ |
| 6 | 可变参数 | 0.5 天 | ☐ |
| 7 | 位运算技巧 | 1 天 | ☐ |
| 8 | C11/C17/C23 新特性 | 1-2 天 | ☐ |
| 9 | 编译、链接与库 | 1-2 天 | ☐ |
| 10 | 未定义行为与安全 | 1 天 | ☐ |
| 11 | 工具链与调试 | 持续 | ☐ |

---

## 待填充内容（后续逐节展开）

1. 每节的详细讲解 + 代码示例（按当前框架顺序推进）
2. 每节配套练习题目
3. 与 C++ 逐主题深入对照笔记
4. 常见坑与面试高频考点标注（尤其指针、内存、UB 章节）

> 告诉我"从第 X 节开始填充"或"填充某节"，我会按框架逐节写内容。
