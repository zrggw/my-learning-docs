# uthash / utarray 速查手册（C 哈希表与动态数组）

> 用途：`uthash.h`、`utarray.h` 的增删改查速查 + 上手示例。
> 定位：查宏名、参数顺序、键失效条件、内存归属。不覆盖 `utlist` / `utstring` / `utringbuffer`。
> 前提：两个头文件均为 **2.4.0**（单文件、无外部依赖、BSD 修订版许可）；C99+。
> 实测环境：gcc 14.2.0（MSYS2，Win64）；文中数值均为实测结果。

---

## 增删改查速查（先看这里）

### uthash（哈希表：结构体内放 `UT_hash_handle`，表头初始化为 `NULL`）

| 操作 | 宏 | 参数与要点 |
|---|---|---|
| **增** | `HASH_ADD_INT(head, intfield, item)` | 第 2 参是**键字段名**；字符串键 `HASH_ADD_STR(head, strfield, item)`；指针键 `HASH_ADD_PTR` |
| **删** | `HASH_DEL(head, item)` | 只从表中摘除，**不释放内存** |
| **改** | `HASH_REPLACE_INT(head, intfield, newitem, olditem)` | 旧元素经 `olditem` 返回，**不释放**，由调用者处理 |
| **查** | `HASH_FIND_INT(head, &key, out)` | int 键**传地址**；字符串键 `HASH_FIND_STR(head, key, out)` **不传地址**；`out` 未命中为 `NULL` |
| 判存在 | `HASH_FIND*` 后判断 `out != NULL` | `head == NULL` 时安全，直接得 `NULL` |
| 遍历 | `HASH_ITER(hh, head, el, tmp)` | 遍历中删除的唯一安全写法 |
| 计数 | `HASH_COUNT(head)` | O(1) |
| 排序 | `HASH_SORT(head, cmp)` | `cmp` 收**元素指针**：`int f(T *a, T *b)` |
| 清表 | `HASH_CLEAR(hh, head)` | 只释放桶与表，**元素仍归调用者** |
| 自定义句柄名 | 泛型宏：`HASH_ADD(hh, head, field, keylen, item)`、`HASH_FIND`、`HASH_DELETE`、`HASH_ITER`、`HASH_CNT`、`HASH_SRT`、`HASH_CLEAR` | 便捷宏与 `HASH_DEL`/`HASH_COUNT`/`HASH_SORT` **硬编码 `hh`** |

### utarray（动态数组：先选 `UT_icd` 描述符）

| 操作 | 宏 | 参数与要点 |
|---|---|---|
| **增** | `utarray_push_back(a, &elem)` | 传**元素地址**；字符串数组用 `ut_str_icd` 时传 `char**` |
| **删** | `utarray_erase(a, pos, len)` / `utarray_pop_back(a)` | 空数组 `pop_back` 会**长度下溢**为 `4294967295` |
| **改** | `utarray_eltptr(a, i)` 取地址直接改 / `utarray_replace(a, &v, i)` | `eltptr` 越界返回 `NULL` |
| **查** | `utarray_find(a, &key, cmp)` | 即 `bsearch`，**数组必须先用同一 `cmp` 排序** |
| 取元素 | `utarray_eltptr(a, i)` / `utarray_front(a)` / `utarray_back(a)` | 返回 `void*`，需强转 |
| 遍历 | `for (p = utarray_front(a); p; p = utarray_next(a, p))` | `next/prev` 以 `NULL` 表示起头 / 起尾 |
| 长度 | `utarray_len(a)` | 元素个数 |
| 排序 | `utarray_sort(a, cmp)` | 即 `qsort`，`cmp` 收**元素地址**（与 `HASH_SORT` 不同） |
| 清空 | `utarray_clear(a)` | 有 `dtor` 则逐个析构；**保留数据缓冲区与容量** |

### 最小骨架

```c
/* uthash：定义 → 初始化表头 → 增查删 */
typedef struct { int id; const char *name; UT_hash_handle hh; } u_t;

u_t *h = NULL, *x, *tmp;                       /* 表头必须初始化为 NULL */
x = (u_t *)calloc(1, sizeof(u_t)); x->id = 1; x->name = "u";
HASH_ADD_INT(h, id, x);                        /* 增 */

int k = 1; HASH_FIND_INT(h, &k, x);            /* 查 */
HASH_ITER(hh, h, x, tmp) { HASH_DEL(h, x); free(x); }   /* 遍历删除 */
```

```c
/* utarray：选描述符 → 操作 → 释放 */
UT_array *a;
utarray_new(a, &ut_int_icd);                   /* int 元素 */

int v = 42; utarray_push_back(a, &v);          /* 增 */
int *p = (int *)utarray_front(a);              /* 取首元素 */
utarray_free(a);
```

### 三个最容易错的地方

| 点 | 事实 |
|---|---|
| 键不复制 | uthash 只存键指针；字符串键缓冲区必须存活且内容不变，否则元素失联 |
| 两个 sort 的 cmp 不同 | `HASH_SORT` 收元素指针 `int f(T*,T*)`；`utarray_sort` 收元素地址（qsort 语义） |
| 谁分配谁释放 | `HASH_DEL` / `HASH_REPLACE` / `HASH_CLEAR` 都不释放元素；utarray 仅在 `icd.dtor` 存在时释放 |

**分类速查**：注册与句柄 → 第 2 节；键与内存 → 第 3 节；遍历排序 → 第 4 节；utarray 语义 → 第 5 节；报错 → 第 6 节；完整宏表 → 附录 A。

---

## 1. 头文件与编译

> 覆盖：获取方式、编译命令、版本宏限制。

- [ ] 从 <https://github.com/troydhanson/uthash> 的 `src/` 取 `uthash.h`、`utarray.h`，放入工程 `include/`；**不需要链接任何库**。
- [ ] 编译：`gcc -std=c11 -Wall -Wextra -Iinclude main.c -o app`
- [ ] **版本宏不能当整数用**：

  ```c
  #define STR_(x) #x
  #define STR(x)  STR_(x)
  printf("%s\n", STR(UTHASH_VERSION));   /* 输出：2.4.0 */
  ```

  | 写法 | 结果 |
  |---|---|
  | `(unsigned)UTHASH_VERSION` | 编译失败：`too many decimal points in number` |
  | `#if UTHASH_VERSION > 0` | 编译失败（同上） |
  | `STR(UTHASH_VERSION)` | 正确，输出 `2.4.0` |

> 记法：版本宏只能字符串化。

---

## 2. uthash：注册与句柄规则

> 覆盖：注册三步、句柄命名规则、多表、重复键。

- [ ] **注册三步**：

  ```c
  typedef struct { int id; char name[32]; UT_hash_handle hh; } user_t;  /* 1. 加句柄成员 */
  user_t *users = NULL;                                                 /* 2. 表头初始化 NULL */
  HASH_ADD_INT(users, id, u);                                           /* 3. 用宏维护 */
  ```

- [ ] **句柄命名规则**（决定能否改名）：

  | 类别 | 宏 |
  |---|---|
  | 硬编码 `hh` | `HASH_ADD_INT` / `_STR` / `_PTR`、`HASH_FIND_INT` / `_STR` / `_PTR`、`HASH_REPLACE_*`、`HASH_DEL`、`HASH_COUNT`、`HASH_SORT` |
  | 支持自定义名 | `HASH_ADD`、`HASH_ADD_KEYPTR`、`HASH_FIND`、`HASH_REPLACE`、`HASH_DELETE`、`HASH_DELETE_HH`、`HASH_ITER`、`HASH_CNT`、`HASH_SRT`、`HASH_CLEAR`、`HASH_OVERHEAD` |

- [ ] **一个结构体可带多个句柄，同时属于多张表**（实测可行）：

  ```c
  typedef struct { int id; char name[16]; UT_hash_handle hh_id; UT_hash_handle hh_name; } p_t;

  HASH_ADD(hh_id,   by_id,   id,   sizeof(int),              e);
  HASH_ADD(hh_name, by_name, name, (unsigned)strlen(e->name), e);
  ```

- [ ] **`HASH_ADD*` 不去重**：同键添加两次 → `HASH_COUNT` 为 2，`HASH_FIND` 返回**后加入**的那个。需要唯一性时先查后插，或用 `HASH_REPLACE*`。
- [ ] **泛型宏的 `keylen` 必须与查找时一致**：`keylen=5` 插入、`keylen=4` 查找会被视为不同键（实测）。

> 记法：增删查记三件套 `HASH_ADD_INT` / `HASH_FIND_INT` / `HASH_DEL`；改句柄名就换泛型宏。

---

## 3. 键与内存所有权（重点）

> 覆盖：键是否复制、字符串键要求、各操作的释放责任。

- [ ] **键不被复制**：`HASH_ADD*` 只保存键指针（实测 `hh.key` 与调用者缓冲区同址；`"alpha"` 的 `keylen` 为 5，不含结尾 NUL）。
- [ ] **字符串键两条要求**：缓冲区**生命周期覆盖元素在表内的整段时间**、**内容不可变**。
- [ ] **原地改键会让元素失联**（实测）：

  ```text
  find_before_mutate=1
  find_alpha_after_mutate=0     /* 用旧内容查不到 */
  find_ALPHA_after_mutate=0     /* 用新内容也查不到（哈希值按插入时计算） */
  ```

  - 正确做法：先 `HASH_DEL` → 改键 → 重新 `HASH_ADD`；或键单独 `strdup` 并自行释放。

- [ ] **释放责任**：

  | 操作 | 元素本体 | 表结构 |
  |---|---|---|
  | `HASH_ADD*` | 调用者分配 | 首次插入自动分配 |
  | `HASH_DEL` | **不释放** | 元素摘除 |
  | `HASH_REPLACE*` | 旧元素经 `replaced` 返回，**不释放** | 换成新元素 |
  | `HASH_CLEAR` | **不释放**（元素变孤儿，调用者需另存指针逐个释放） | 释放桶与表，`head` 置 `NULL` |

- [ ] **`HASH_DEL` 不清空被删元素的句柄字段**：`hh.next` / `hh.prev` / `hh.tbl` 保留原值，`free` 后即为悬垂指针。

> 记法：uthash 只管表，不管元素；顺序永远是"先 `HASH_DEL`，再 `free`"，且只 free 一次。

---

## 4. 遍历、排序与诊断

> 覆盖：`HASH_ITER`、`HASH_SORT` 比较函数、计数与开销、一致性自检。

- [ ] **遍历中删除只能用 `HASH_ITER`**（先把后继存入 `tmp`）：

  ```c
  user_t *u, *tmp;
  HASH_ITER(hh, users, u, tmp) { HASH_DEL(users, u); free(u); }
  ```

  - 反例：`for (u = users; u; u = u->hh.next) { HASH_DEL(users, u); free(u); }` —— `free` 后仍读 `u->hh.next`，实测在覆写式分配器下**确定性崩溃**（`0xC0000005`）。

- [ ] **`HASH_SORT` 的比较函数收元素指针**：

  ```c
  static int by_v(r_t *a, r_t *b) { return a->v - b->v; }
  HASH_SORT(head, by_v);                    /* 实测结果：10 20 30 40 */
  ```

  - 写成 qsort 风格（`const void *` 再解引用为 `int *`）可编译，但实际读到的是结构体首个 `int` 字段 → **静默按错字段排序**。

- [ ] `HASH_COUNT(head)`：O(1)。`HASH_OVERHEAD(hh, head)`：表结构占用，实测 4 个元素共 800 字节（200 字节/元素，`sizeof(UT_hash_handle) = 56`）。
- [ ] `HASH_FSCK(hh, head, "where")` 默认是空宏，需在包含头文件前定义 `HASH_DEBUG` 才生效。
- [ ] `HASH_VALUE(keyptr, keylen, hashv)` 与 `HASH_FIND_BYHASHVALUE(hh, head, keyptr, keylen, hashv, out)` 用于复用预计算哈希。

> 记法：遍历删除用 `HASH_ITER`；排序比较函数收元素指针。

---

## 5. utarray：UT_icd 与语义

> 覆盖：描述符结构、三个预定义描述符、深拷贝行为、扩容与清空语义。

- [ ] **`UT_icd`**（实测 `sizeof = 32`）决定元素如何复制与释放：

  | 字段 | 作用 |
  |---|---|
  | `sz` | 元素字节数（必填） |
  | `init` | 扩展元素时的初始化函数，缺省清零 |
  | `copy` | 复制函数，缺省 `memcpy` |
  | `dtor` | 析构函数，`pop_back` / `clear` / `free` / `resize` 缩小时调用 |

- [ ] **三个预定义描述符**（实测）：

  | 描述符 | `sz` | `copy` | `dtor` | 语义 |
  |---|---|---|---|---|
  | `ut_int_icd` | 4 | 无 | 无 | 浅拷贝 |
  | `ut_ptr_icd` | 8 | 无 | 无 | 只复制指针 |
  | `ut_str_icd` | 8 | 有 | 有 | **深拷贝字符串，析构时 `free`** |

- [ ] **自定义结构体**：`static const UT_icd lot_icd = { sizeof(lot_t), NULL, NULL, NULL };`
- [ ] **`ut_str_icd` 的 `push_back` 传 `char**`**；数组内是独立副本（实测改动调用者缓冲区不影响数组内容），副本由 `dtor` 释放，调用者自己那份仍需自行 `free`。
- [ ] **容量按 0 → 8 → 翻倍增长**（实测 4 个元素时 `n = 8`）；`utarray_clear` 长度归零但**保留缓冲区与容量**。
- [ ] **`utarray_find` 必须已排序**：未排序数组上查找会漏（实测同一数组 `find(30)` 命中、`find(20)` 未命中）。

> 记法：先选 `UT_icd` 再谈操作；`sort` 与 `find` 共用同一个比较函数。

---

## 6. 常见错误与排错

> 覆盖：编译期、运行期、逻辑错误三类清单与排错步骤。

### 6.1 编译期

| 报错 | 原因 | 解决 |
|---|---|---|
| `too many decimal points in number` | 把版本宏当整数 | 只做字符串化 |
| `invalid type argument of unary '*'` | `HASH_FIND_INT` 第 2 参传了值 | 改为 `&key` |
| 找不到成员 `hh` | 句柄改名后仍用硬编码 `hh` 的宏 | 换泛型宏并显式传句柄名 |
| 泛型宏报未声明标识符 | 参数顺序错（元素指针写到了键字段位置） | 核对 `(hh, head, 键字段, keylen, 元素指针)` |

### 6.2 运行期

| 现象 | 原因 | 解决 |
|---|---|---|
| 首次插入即崩溃 | 表头未初始化为 `NULL` | 声明处写 `T *head = NULL;` |
| 崩溃 / 野指针 | 元素已 `free` 仍在表内，或重复 `free` | 先 `HASH_DEL` 再 `free`，只 free 一次 |
| 遍历删除随机崩溃 | 用 `for` + `hh.next` 且释放了当前元素 | 改用 `HASH_ITER` |
| 内存持续增长 | 只 `HASH_CLEAR` / `HASH_REPLACE` 未释放元素 | 这两者不释放元素 |
| 长度变成 `4294967295` | 空数组调用 `utarray_pop_back` | 先判断 `utarray_len(a) > 0` |
| `-Wtype-limits` 告警 | 给 `utarray_insert` / `utarray_resize` 传了字面量 `0` | 用变量传下标 |

### 6.3 逻辑错误（不报错但结果错）

| 现象 | 原因 | 解决 |
|---|---|---|
| 查找突然失败 | 字符串键缓冲区被改或提前释放 | 键单独 `strdup`，或改键前先 `HASH_DEL` |
| 用不同 `keylen` 查找失败 | 插入与查找的 `keylen` 不一致 | 两边用同一表达式 |
| 计数比预期多 | `HASH_ADD*` 不去重 | 先查后插，或用 `HASH_REPLACE*` |
| `utarray_find` 找不到 | 数组未排序 | 先 `utarray_sort`，复用同一 `cmp` |
| 排序结果不对 | `HASH_SORT` 用了 qsort 风格比较函数 | 改为 `int cmp(T *a, T *b)` |
| `utarray_eltptr(a, i++)` 异常 | 宏参数被求值两次 | 下标先算进变量 |

### 6.4 排错步骤

- [ ] 确认键存活且未改动、元素只被释放一次。
- [ ] 缩小到 2–3 个元素的最小复现，打印 `HASH_COUNT` / `utarray_len` 与关键字段。
- [ ] 打开诊断：uthash 加 `-DHASH_DEBUG`；编译加 `-Wall -Wextra -fsanitize=address`。

> 记法：崩溃先查"表头初始化 + 重复释放"；查不到先查"键被改 + 数组未排序"。

---

## 附录 A 完整宏表

**uthash**

| 宏 | 签名 | 要点 |
|---|---|---|
| `HASH_ADD_INT` / `_STR` / `_PTR` | `(head, keyfield, add)` | 键分别为 int / 字符串 / 指针 |
| `HASH_FIND_INT` / `_STR` / `_PTR` | `(head, key, out)` | int 与指针键**传地址**，字符串键不传 |
| `HASH_REPLACE_INT` / `_STR` / `_PTR` | `(head, keyfield, add, replaced)` | 旧元素不释放 |
| `HASH_DEL` | `(head, delptr)` | 不释放元素 |
| `HASH_COUNT` | `(head)` | O(1) |
| `HASH_ITER` | `(hh, head, el, tmp)` | 遍历中删除安全 |
| `HASH_SORT` | `(head, cmpfcn)` | cmp 收元素指针 |
| `HASH_ADD` / `HASH_ADD_BYHASHVALUE` | `(hh, head, fieldname, keylen, add)` | 泛型 |
| `HASH_ADD_KEYPTR` | `(hh, head, keyptr, keylen, add)` | 键在结构体外 |
| `HASH_FIND` / `HASH_FIND_BYHASHVALUE` | `(hh, head, keyptr, keylen[, hashval], out)` | 泛型查找 |
| `HASH_REPLACE` | `(hh, head, fieldname, keylen, add, replaced)` | 泛型替换 |
| `HASH_DELETE` / `HASH_DELETE_HH` | `(hh, head, delptr)` / `(hh, head, &el->hh)` | 泛型删除 |
| `HASH_CNT` / `HASH_SRT` | `(hh, head)` / `(hh, head, cmpfcn)` | 泛型计数 / 排序 |
| `HASH_CLEAR` / `HASH_OVERHEAD` | `(hh, head)` | 只回收表结构 / 查询开销 |
| `HASH_VALUE` | `(keyptr, keylen, hashv)` | 预计算哈希 |
| `HASH_SELECT` | `(hh_dst, dst, hh_src, src, cond)` | 按条件复制到另一张表 |
| `HASH_FSCK` | `(hh, head, where)` | 需 `HASH_DEBUG` |
| 哈希函数 | `HASH_JEN`（默认）/ `HASH_BER` / `HASH_SAX` / `HASH_FNV` / `HASH_OAT` / `HASH_SFH` | 定义 `HASH_FUNCTION` 替换 |

**utarray**

| 宏 | 签名 | 要点 |
|---|---|---|
| `utarray_new` / `free` | `(a, &icd)` / `(a)` | 堆上创建 / 销毁 |
| `utarray_init` / `done` | `(a, &icd)` / `(a)` | 栈上数组 |
| `utarray_push_back` | `(a, &elem)` | 传元素地址 |
| `utarray_pop_back` | `(a)` | 空数组下溢 |
| `utarray_extend_back` | `(a)` | 追加未初始化元素 |
| `utarray_len` | `(a)` | 元素个数 |
| `utarray_eltptr` | `(a, i)` | 越界返回 `NULL`；参数求值两次 |
| `utarray_front` / `back` | `(a)` | 首 / 尾地址 |
| `utarray_next` / `prev` | `(a, e)` | `e = NULL` 起头 / 起尾 |
| `utarray_eltidx` | `(a, e)` | 地址 → 下标 |
| `utarray_insert` / `replace` | `(a, &v, i)` | 插入 / 覆盖 |
| `utarray_erase` | `(a, pos, len)` | 删除区间 |
| `utarray_inserta` / `concat` | `(a, w, i)` / `(dst, src)` | 整段插入 / 追加 |
| `utarray_resize` | `(a, n)` | 缩小析构、扩大初始化 |
| `utarray_clear` / `renew` | `(a)` / `(a, &icd)` | 清空元素 / 按需创建或清空 |
| `utarray_sort` / `find` | `(a, cmp)` / `(a, &v, cmp)` | qsort / bsearch（查找前必须排序） |

**编译与调试开关**

| 开关 | 作用 |
|---|---|
| `-DHASH_DEBUG` | 启用 `HASH_FSCK` |
| 定义 `HASH_FUNCTION` | 替换哈希函数（默认 `HASH_JEN`） |
| 定义 `HASH_BLOOM`（位数） | 启用 Bloom 过滤器 |
| 定义 `uthash_malloc` / `uthash_free` / `uthash_fatal` | 替换内存分配与 OOM 行为（默认 `exit(-1)`） |
| 定义 `utarray_oom()` | 替换 utarray 分配失败行为（默认 `exit(-1)`） |
| `-fsanitize=address` | 排查越界、重复释放、释放后使用 |

---

## 附录 B 学习路径与进度表

| 阶段 | 对应章节 | 内容 | 目标 | 状态 |
|---|---|---|---|---|
| 1 | 速查表 | 增删改查 | 能查到宏名与参数顺序 | ☐ |
| 2 | 1 | 头文件与编译 | 跑通最小示例 | ☐ |
| 3 | 2 | 注册与句柄 | 会注册结构体、会换用泛型宏 | ☐ |
| 4 | 3 | 键与所有权 | 说清键何时失效、谁负责 `free` | ☐ |
| 5 | 4 | 遍历与排序 | 会用 `HASH_ITER`、`HASH_SORT` | ☐ |
| 6 | 5 | utarray | 会选 `UT_icd`，会用 `sort` / `find` | ☐ |
| 7 | 6 | 排错 | 能区分三类错误 | ☐ |

---

## 附录 C 后续扩展（当前缺口）

1. `utlist.h` / `utstring.h` / `utringbuffer.h` 对照速查
2. `HASH_SELECT` 与 `HASH_BLOOM` 的完整示例
3. 线程安全封装（两个库都不带锁）
4. 自定义 `HASH_FUNCTION` 与内存钩子的性能对比

> 告诉我"从第 X 节开始填充"或"填充某节"，我会按框架逐节展开。
