# uthash / utarray 速查手册（C 哈希表与动态数组）

> 用途：`uthash.h` 与 `utarray.h` 的上手示例 + 速查表。面向**熟悉 C 指针与结构体、需要给 C 工程加"哈希表 / 动态数组"**的读者。
> 使用方式：每学完一个小点，把 `[ ]` 改成 `[x]`；需要展开某节可随时让我补充。
> 定位：解决 **"该用哪个宏、参数顺序是什么、键什么时候会失效、谁负责 free"**，从"能查表"到"能用对不泄漏"。
> 背景贴合：你已在学 C 语言高级内容（`c-advanced-learning.md`），本文补上"工程里最常用的两个容器头文件"。
> 前提：`uthash.h` / `utarray.h` **2.4.0**（单文件、无外部依赖、BSD 修订版许可）；C99 及以上；文中所有输出均为 **gcc 14.2.0（MSYS2，Win64）实测**。

---

## 核心速览（先看这一页，5 分钟覆盖 80% 内容）

**一句话定位**：两个头文件都是"宏实现的数据结构"，`uthash.h` 提供哈希表（键值查找），`utarray.h` 提供动态数组（可增长的连续数组）。用它们**不需要链接任何库**，`#include` 后直接调用宏。

**① 最小可用示例**

uthash：结构体里放一个 `UT_hash_handle`，表头指针初始化为 `NULL`。

```c
#include <stdio.h>
#include <stdlib.h>
#include "uthash.h"

typedef struct { int id; const char *name; UT_hash_handle hh; } u_t;

int main(void)
{
    u_t *h = NULL, *x, *tmp;
    int i;

    for (i = 1; i <= 3; i++) {                  /* 增 */
        x = (u_t *)calloc(1, sizeof(u_t));
        x->id = i; x->name = "u";
        HASH_ADD_INT(h, id, x);
    }

    i = 2;                                      /* 查：int 键要传 &key */
    HASH_FIND_INT(h, &i, x);
    printf("count=%u find2_id=%d name=%s\n", HASH_COUNT(h),
           x ? x->id : -1, x ? x->name : "-");

    HASH_ITER(hh, h, x, tmp) { HASH_DEL(h, x); free(x); }   /* 遍历删除 */
    printf("after_free_all count=%u head_is_null=%d\n",
           (unsigned)HASH_COUNT(h), h == NULL);
    return 0;
}
```

```text
/* 实测输出 */
count=3 find2_id=2 name=u
after_free_all count=0 head_is_null=1
```

utarray：先选一个"元素描述符"（`UT_icd`），再操作数组。

```c
#include <stdio.h>
#include "utarray.h"

int main(void)
{
    UT_array *a;
    int i, *p;

    utarray_new(a, &ut_int_icd);                 /* int 元素 */
    for (i = 1; i <= 3; i++) utarray_push_back(a, &i);

    printf("len=%u front=%d back=%d\n", utarray_len(a),
           *(int *)utarray_front(a), *(int *)utarray_back(a));

    for (p = (int *)utarray_front(a); p != NULL; p = (int *)utarray_next(a, p))
        printf(" %d", *p);
    printf("\n");

    utarray_free(a);
    return 0;
}
```

```text
/* 实测输出 */
len=3 front=1 back=3
 1 2 3
```

**② 五个核心概念**

| 概念 | 说明 |
|---|---|
| **元素描述符 `UT_icd`** | utarray 靠它知道元素大小与拷贝/析构方式；`{sizeof(T),NULL,NULL,NULL}` 即"按字节复制、不释放" |
| **键不复制** | uthash 只保存键的**指针**；字符串键必须指向存活且内容不变的缓冲区 |
| **句柄名 `hh`** | 结构体里的 `UT_hash_handle` 成员；便捷宏（`HASH_ADD_INT` 等）**硬编码 `hh`**，改名必须换泛型宏 |
| **两个 sort 的签名不同** | `HASH_SORT` 的比较函数收**元素指针** `int f(T*,T*)`；`utarray_sort` 是 qsort 语义，收**元素地址** |
| **所有权在调用者** | uthash 只管表结构，`HASH_DEL`/`HASH_REPLACE`/`HASH_CLEAR` **都不释放元素**；utarray 只在 icd 提供 dtor 时释放 |

**③ 高频操作对照**

| 要做什么 | uthash | utarray |
|---|---|---|
| 新增 | `HASH_ADD_INT(head, intfield, add)` | `utarray_push_back(a, &elem)` |
| 查找 | `HASH_FIND_INT(head, &key, out)` | `utarray_find(a, &key, cmp)`（**要求已排序**） |
| 按键删除 | `HASH_DEL(head, delptr)` | `utarray_erase(a, pos, len)` |
| 判存在 / 原子查 | `HASH_FIND*` 后判断 `out != NULL` | `utarray_find` 后判断返回值 |
| 遍历 | `HASH_ITER(hh, head, el, tmp)` | `for (p = utarray_front(a); p; p = utarray_next(a, p))` |
| 计数 / 长度 | `HASH_COUNT(head)` | `utarray_len(a)` |
| 排序 | `HASH_SORT(head, cmp)` | `utarray_sort(a, cmp)` |
| 清空 | `HASH_CLEAR(hh, head)` 后自行释放元素 | `utarray_clear(a)`（有 dtor 则逐个析构） |
| 取第 i 个元素 | 按值查找，无下标 | `utarray_eltptr(a, i)`（越界返回 `NULL`） |

**④ 三大高频错误**（完整对照表见第 6 节）

| 现象 | 原因 |
|---|---|
| 运行即崩 / 野指针 | 表头指针未初始化为 `NULL`，或元素已 `free` 但仍在表内 |
| 查不到刚插入的元素 | 字符串键缓冲区被修改或已释放（哈希值按插入时计算） |
| `NOT FOUND` 但元素确实存在 | `utarray_find` 用 bsearch，**数组未排序**；或泛型宏的 `keylen` 与查找时不一致 |

**阅读路线**：只想跑通 → 第 1 节；要写业务代码 → 第 2–3 节（键与所有权）；用动态数组 → 第 5 节；出问题 → 第 6 节；查签名 → 附录 A。

---

## 1. 快速上手

> 覆盖：获取头文件、编译方式、两个最小示例、版本宏的用法限制。

### 1.1 获取与编译

- [ ] **两个头文件都是"单文件库"**：从 <https://github.com/troydhanson/uthash> 的 `src/` 目录取 `uthash.h`、`utarray.h`，放进工程的 `include/` 即可，**不需要链接任何库**。
- [ ] **编译命令**（以本机为例）：

  ```bash
  gcc -std=c11 -Wall -Wextra -Iinclude main.c -o app
  ```

- [ ] **同一套宏还能解决其他容器需求**：`utlist.h`（双向链表）、`utstring.h`（动态字符串）、`utringbuffer.h`（环形缓冲），用法风格一致。

### 1.2 验证版本宏（易错点）

- [ ] **`UTHASH_VERSION` / `UTARRAY_VERSION` 是预处理数字 `2.4.0`，不是整型常量**，不能参与比较或强制转换：

  ```c
  #define STR_(x) #x
  #define STR(x)  STR_(x)
  printf("uthash=%s utarray=%s\n", STR(UTHASH_VERSION), STR(UTARRAY_VERSION));
  ```

  ```text
  /* 实测输出 */
  uthash=2.4.0 utarray=2.4.0
  ```

  | 写法 | 结果 |
  |---|---|
  | `(unsigned)UTHASH_VERSION` | 编译失败：`too many decimal points in number` |
  | `#if UTHASH_VERSION > 0` | 编译失败（同上） |
  | `STR(UTHASH_VERSION)` | 输出 `2.4.0`（正确做法） |

> 记法：**版本宏只能字符串化**；要判断版本请在构建系统里检查头文件内容或文档标注。

---

## 2. uthash：结构体注册与宏家族

> 覆盖：注册结构体的三步、便捷宏与泛型宏的对应关系、句柄命名规则、多表与重复键行为。

### 2.1 注册三步

- [ ] **第一步：在结构体里加一个句柄成员**

  ```c
  typedef struct { int id; char name[32]; UT_hash_handle hh; } user_t;
  ```

- [ ] **第二步：表头指针初始化为 `NULL`**（漏掉这步是崩溃的第一大原因）

  ```c
  user_t *users = NULL;      /* 局部变量务必显式初始化 */
  ```

- [ ] **第三步：用宏维护表**

  ```c
  user_t *u = (user_t *)calloc(1, sizeof(user_t));
  u->id = 1;
  HASH_ADD_INT(users, id, u);              /* 第 2 参是"键字段名"，不是键值 */
  ```

### 2.2 宏家族：便捷宏与泛型宏

| 用途 | 便捷宏（硬编码句柄 `hh`） | 泛型宏（句柄名自定） |
|---|---|---|
| 添加 | `HASH_ADD_INT(head,intfield,add)` / `HASH_ADD_STR(head,strfield,add)` / `HASH_ADD_PTR(head,ptrfield,add)` | `HASH_ADD(hh,head,fieldname,keylen,add)` |
| 键在结构体外 | — | `HASH_ADD_KEYPTR(hh,head,keyptr,keylen,add)` |
| 查找 | `HASH_FIND_INT(head,&key,out)` / `HASH_FIND_STR(head,key,out)` / `HASH_FIND_PTR(head,&key,out)` | `HASH_FIND(hh,head,keyptr,keylen,out)` |
| 替换 | `HASH_REPLACE_INT(head,intfield,add,replaced)` / `_STR` / `_PTR` | `HASH_REPLACE(hh,head,fieldname,keylen,add,replaced)` |
| 删除 | `HASH_DEL(head,delptr)` | `HASH_DELETE(hh,head,delptr)`、`HASH_DELETE_HH(hh,head,&el->hh)` |
| 计数 | `HASH_COUNT(head)` | `HASH_CNT(hh,head)` |
| 遍历 | — | `HASH_ITER(hh,head,el,tmp)` |
| 排序 | `HASH_SORT(head,cmp)` | `HASH_SRT(hh,head,cmp)` |
| 只清表结构 | — | `HASH_CLEAR(hh,head)` |
| 表结构开销 | — | `HASH_OVERHEAD(hh,head)` |
| 复用已算哈希 | `HASH_VALUE(keyptr,keylen,hashv)` + `HASH_FIND_BYHASHVALUE(hh,head,keyptr,keylen,hashv,out)` | 同左 |
| 一致性自检 | — | `HASH_FSCK(hh,head,where)`（需定义 `HASH_DEBUG` 才生效） |

- [ ] **参数顺序容易记错**：便捷宏是 `(head, 键字段, 元素指针)`；泛型宏是 `(句柄, head, 键字段或键指针, 键长, 元素指针)`。
- [ ] **查找的两处不对称**（实测确认）：`HASH_FIND_INT` 第 2 参是**键的地址**（`&key`）；`HASH_FIND_STR` 第 2 参是**键字符串本身**（`char*`，不加 `&`）。
- [ ] **head 为 `NULL` 时 `HASH_FIND*` 安全**：直接把 `out` 置为 `NULL`，不需要先判空。

> 记法：**"增删查"记三件套**——`HASH_ADD_INT` / `HASH_FIND_INT` / `HASH_DEL`；字符串键把 `_INT` 换成 `_STR`，且查找不用取地址。

### 2.3 句柄命名规则

- [ ] **硬编码 `hh` 的宏**：所有 `_INT` / `_STR` / `_PTR` 便捷宏、`HASH_DEL`、`HASH_COUNT`、`HASH_SORT`。
- [ ] **支持自定义句柄名的宏**：`HASH_ADD` / `HASH_ADD_KEYPTR` / `HASH_FIND` / `HASH_REPLACE` / `HASH_DELETE` / `HASH_DELETE_HH` / `HASH_ITER` / `HASH_CNT` / `HASH_SRT` / `HASH_CLEAR` / `HASH_OVERHEAD`。

  ```c
  typedef struct { int id; char name[16]; UT_hash_handle hh_id; UT_hash_handle hh_name; } p_t;

  HASH_ADD(hh_id,   by_id,   id,   sizeof(int),          e);
  HASH_ADD(hh_name, by_name, name, (unsigned)strlen(e->name), e);
  HASH_FIND(hh_id, by_id, &k, sizeof(int), e);
  ```

  ```text
  /* 实测输出：同一元素同时存在于两张表 */
  B by_id[2].name=adam
  B by_name[mike].id=1 count_id=3 count_name=3
  B after_del by_id_null=1 by_name_null=1
  ```

- [ ] **一个结构体可以有多个句柄**，从而同时属于多张哈希表（实测可行，两张表各自维护链表与桶）。

### 2.4 重复键

- [ ] **`HASH_ADD*` 不去重**：同一个键添加两次，`HASH_COUNT` 变为 2，`HASH_ITER` 会遍历出两个元素，`HASH_FIND` 返回**后加入**的那个。
- [ ] **需要"同键唯一"时**：先 `HASH_FIND*` 判断，或用 `HASH_REPLACE*` 覆盖。

> 记法：**uthash 不做唯一性约束**；唯一性由你的业务逻辑保证。

---

## 3. 键与内存所有权（重点）

> 覆盖：键是否复制、字符串键的存活与不可变要求、keylen 一致性、各操作的内存所有权归属。

### 3.1 键不被复制

- [ ] **`HASH_ADD*` 保存的是键指针**，不是键副本（实测 `hh.key` 与调用者缓冲区地址相同）：

  ```text
  C stored_key_same_ptr=1 keylen=5      /* "alpha" 的 keylen 是 5，不含结尾 NUL */
  ```

- [ ] **后果**：键缓冲区必须满足两个条件——**生命周期覆盖元素在表内的整段时间**、**内容不可变**。
- [ ] **原地修改键会让元素失联**（实测：新旧值都查不到）：

  ```text
  C find_before_mutate=1
  C find_alpha_after_mutate=0    /* 用旧内容查不到 */
  C find_ALPHA_after_mutate=0    /* 用新内容也查不到（哈希值按插入时计算） */
  ```

  - 正确做法：`HASH_DEL` 删除元素 → 修改键 → 重新 `HASH_ADD`；或让键指向独立的 `strdup` 缓冲区，并自行管理其释放。

- [ ] **`HASH_ADD_STR` 的 keylen 是 `strlen`（不含 NUL）**，因此 `"abc"` 与 `"abc\0"` 视为同一键。
- [ ] **泛型宏的 `keylen` 必须与查找时完全一致**：用 `keylen=5` 插入、用 `keylen=4` 查找，会被当成不同的键（实测确认）。

### 3.2 内存所有权

| 操作 | 元素本体 | 表结构 |
|---|---|---|
| `HASH_ADD*` | 调用者分配（`malloc`/`calloc`），uthash 只记录指针 | 首次插入时自动分配 |
| `HASH_DEL` | **不释放**，调用者负责 `free` | 元素从表内摘除 |
| `HASH_REPLACE*` | 被替换的旧元素通过 `replaced` 参数返回，**不释放**（实测其字段仍可读） | 表内换成新元素 |
| `HASH_CLEAR` | **不释放**，元素成为"孤儿"，调用者需另行保存指针逐个释放 | 释放桶与表，`head` 置 `NULL` |

- [ ] **实测证据（`HASH_REPLACE_INT`）**：

  ```text
  D replaced_id=7 replaced_v=100 count=1 new_v=999
  ```

- [ ] **实测证据（`HASH_CLEAR`）**：`head` 变 `NULL`、计数归零，但元素字段仍可读，必须由调用者释放。

  ```text
  E after_clear head_null=1 count=0 element_still_readable=10
  ```

- [ ] **`HASH_DEL` 不清空被删元素的句柄字段**：`hh.next` / `hh.prev` / `hh.tbl` 保持原值，元素被 `free` 后这些字段即悬垂指针，不可再用。

> 记法：**uthash 只管"表"，不管"元素"**；`free` 的时机永远是"先 `HASH_DEL`，再 `free`"，且只由调用者执行一次。

---

## 4. 遍历、排序与诊断

> 覆盖：`HASH_ITER` 的正确用法与反例、`HASH_SORT` 的比较函数签名、计数与开销、预计算哈希、一致性自检。

### 4.1 遍历并删除：只用 `HASH_ITER`

- [ ] **正确写法**（先把后继存进 `tmp`，再删当前元素）：

  ```c
  user_t *u, *tmp;
  HASH_ITER(hh, users, u, tmp) {
      HASH_DEL(users, u);
      free(u);
  }
  ```

- [ ] **错误写法**：`for (u = users; u; u = u->hh.next) { HASH_DEL(users, u); free(u); }`——`free` 之后仍要读 `u->hh.next`，是典型的释放后使用；实测在开启"释放即覆写"的分配器下**确定性崩溃**（`0xC0000005`）。
- [ ] `tmp` 必须与 `el` 同类型；`HASH_ITER` 内部总是先取后继再执行循环体，因此"边遍历边删"是安全的。

### 4.2 排序：`HASH_SORT` 的比较函数收"元素指针"

- [ ] 正确签名（参数是元素本身）：

  ```c
  static int by_v(r_t *a, r_t *b) { return a->v - b->v; }
  HASH_SORT(head, by_v);
  ```

  ```text
  /* 实测输出 */
  F sorted_v: 10 20 30 40
  ```

- [ ] **写成 qsort 风格（`const void *` 再解引用为 `int*`）会编译通过但结果错误**：实际收到的是元素指针，`*(int*)a` 读到的是结构体的第一个 `int` 字段，于是"按错误的字段排序"，且不报任何警告。
- [ ] **`utarray_sort` 才是 qsort 语义**（收元素地址），两者不能互换：

  ```c
  static int int_cmp(const void *a, const void *b) {
      int x = *(const int *)a, y = *(const int *)b;
      return (x > y) - (x < y);
  }
  utarray_sort(a, int_cmp);
  ```

> 记法：**`HASH_SORT` 传 `T*` 比较函数，`utarray_sort` 传 qsort 比较函数**——写错不会编译失败，只会结果错。

### 4.3 计数、开销与哈希复用

- [ ] `HASH_COUNT(head)` / `HASH_CNT(hh, head)`：元素个数（O(1)，读 `tbl->num_items`）。
- [ ] `HASH_OVERHEAD(hh, head)`：表结构（桶数组 + 表头 + 每个元素的句柄）占用字节数。实测：4 个元素共 800 字节，即 **200 字节/元素**，其中 `sizeof(UT_hash_handle) = 56`。
- [ ] `HASH_VALUE(keyptr,keylen,hashv)`：手工计算哈希；`HASH_FIND_BYHASHVALUE(hh,head,keyptr,keylen,hashv,out)`：复用该哈希查找（实测与元素内的 `hh.hashv` 一致）。
- [ ] `HASH_FSCK(hh,head,where)`：一致性自检，**默认是空宏**，需在包含头文件前定义 `HASH_DEBUG` 才生效。
- [ ] 默认哈希函数是 `HASH_JEN`（Jenkins）；`HASH_BER` / `HASH_SAX` / `HASH_FNV` / `HASH_OAT` / `HASH_SFH` 也可用，通过定义 `HASH_FUNCTION` 替换。

> 记法：**先量再调**——`HASH_OVERHEAD` 看内存、`HASH_COUNT` 看规模，哈希函数与 Bloom 过滤器属于压测后的优化项。

---

## 5. utarray：描述符与常用操作

> 覆盖：`UT_array` / `UT_icd` 结构、三个预定义描述符、深拷贝行为、常用操作与语义、扩容策略、排序查找。

### 5.1 结构与描述符

- [ ] **`UT_array`**（实测 `sizeof = 48`）内部只有三样东西：`unsigned i`（元素个数）、`unsigned n`（已分配槽位数）、`char *d`（数据块）。
- [ ] **`UT_icd`**（实测 `sizeof = 32`）决定"元素怎么复制、怎么释放"：

  | 字段 | 作用 |
  |---|---|
  | `sz` | 单个元素字节数（必填） |
  | `init` | 扩展元素时的初始化函数，缺省用 `memset(0)` |
  | `copy` | 复制元素的函数，缺省用 `memcpy` |
  | `dtor` | 析构函数，`pop_back` / `clear` / `free` / `resize` 缩小时调用 |

  ```c
  typedef struct { int x; double y; char name[8]; } lot_t;
  static const UT_icd lot_icd = { sizeof(lot_t), NULL, NULL, NULL };   /* 纯字节复制 */
  ```

- [ ] **三个预定义描述符**（实测）：

  | 描述符 | `sz` | `copy` | `dtor` | 语义 |
  |---|---|---|---|---|
  | `ut_int_icd` | 4 | 无 | 无 | 浅拷贝，`push_back(a, &intvar)` |
  | `ut_ptr_icd` | 8 | 无 | 无 | 存指针，只复制指针本身 |
  | `ut_str_icd` | 8 | 有 | 有 | **深拷贝字符串，并在析构时 `free`** |

### 5.2 字符串数组的深拷贝（易错点）

- [ ] **`ut_str_icd` 的 `push_back` 要传 `char**`**（元素本身就是 `char*`）：

  ```c
  char *orig = (char *)malloc(6); strcpy(orig, "hello");
  utarray_push_back(a, &orig);            /* 传的是 char** */
  ```

  ```text
  /* 实测输出 */
  H str_icd deep_copy=1 value=hello
  H after_mutate_caller_buf=hello      /* 改调用者缓冲区，数组内不受影响 */
  ```

- [ ] **数组内的字符串由 icd 的 `dtor` 释放**：`utarray_free` / `utarray_clear` / `utarray_pop_back` 都会调用它；调用者自己那份字符串仍需自行 `free`。
- [ ] 浅拷贝描述符（`ut_int_icd` / `ut_ptr_icd` / 自定义无 dtor 的 icd）**不会释放元素内容**，指针型元素的释放由调用者负责。

### 5.3 常用操作

| 宏 | 签名 | 语义与注意点 |
|---|---|---|
| `utarray_new` / `utarray_free` | `(a,_icd)` / `(a)` | `new` = 分配 `UT_array` + `init`；`free` = `done` + 释放本体 |
| `utarray_init` / `utarray_done` | `(a,_icd)` / `(a)` | 用于栈上 `UT_array`；`init` 会把 `icd` 整体复制进数组 |
| `utarray_push_back` | `(a,p)` | `p` 是**元素地址**；按 `icd.copy` 或 `memcpy` 写入 |
| `utarray_pop_back` | `(a)` | 有 `dtor` 先析构；**空数组调用会使长度下溢**（见第 6 节） |
| `utarray_extend_back` | `(a)` | 追加一个"空元素"（`init` 或清零），返回前需自行取地址填值 |
| `utarray_len` | `(a)` | 元素个数 |
| `utarray_eltptr` | `(a,j)` | 第 j 个元素地址；**越界返回 `NULL`**；宏参数 `j` 会被求值两次 |
| `utarray_front` / `utarray_back` | `(a)` | 首 / 尾元素地址，空数组返回 `NULL` |
| `utarray_next` / `utarray_prev` | `(a,e)` | 以 `e = NULL` 表示"从首 / 尾开始" |
| `utarray_eltidx` | `(a,e)` | 由元素地址反推下标 |
| `utarray_insert` | `(a,p,j)` | 在下标 `j` 处插入；`j` 超过长度时先补空位 |
| `utarray_replace` | `(a,p,j)` | 覆盖第 `j` 个元素（先析构旧值） |
| `utarray_inserta` / `utarray_concat` | `(a,w,j)` / `(dst,src)` | 把整段数组插入到位置 `j` / 追加到尾部 |
| `utarray_erase` | `(a,pos,len)` | 析构被删元素并搬移后续元素 |
| `utarray_resize` | `(dst,num)` | 缩小则析构多余元素，扩大则按 `init` / 清零初始化 |
| `utarray_clear` | `(a)` | 析构全部元素，长度归零，**保留数据缓冲区与容量** |
| `utarray_renew` | `(a,u)` | 非空则 `clear`，为空则 `new` |
| `utarray_sort` | `(a,cmp)` | 即 `qsort`，比较函数收元素地址 |
| `utarray_find` | `(a,v,cmp)` | 即 `bsearch`，**要求数组已按同一比较函数排序** |
| `utarray_done` 之后 | — | 数组不可再使用，除非重新 `init` |

- [ ] **扩容策略**：容量从 0 起步，首次分配 8 个槽位，之后每次翻倍（实测 4 个元素时 `n = 8`）。

  ```text
  G len=4 n_slots=8 front=30 back=20
  G eltptr_oob_is_null=1
  ```

- [ ] **`utarray_find` 必须已排序**（实测：未排序时只有部分元素能被命中）：

  ```text
  G find_unsorted_20=NOT FOUND
  G find_sorted_20=found sorted: 10 20 30 40
  G after_erase_len=2 back=40
  G after_insert_concat_len=5: 99 10 40 7 8
  ```

- [ ] **遍历时不要把 `utarray_next` 的结果强转错**：宏返回 `void*`，需转成 `T*`（元素是指针类型时则为 `T**`）。
- [ ] **`utarray_eltptr` 允许通过返回地址直接修改元素**（实测把 `elt0.x` 改成 99 生效）。

> 记法：**先选 `UT_icd` 再谈操作**——`ut_int_icd` 浅拷贝、`ut_str_icd` 深拷贝、自定义结构体用 `{sizeof(T),NULL,NULL,NULL}`；`sort` 与 `find` 共用同一个比较函数。

---

## 6. 常见错误与排错（重点）

> 覆盖：编译期、运行期、逻辑错误三类清单，排错步骤，以及编译器告警的处理。

### 6.1 编译期错误

| 报错 | 原因 | 解决 |
|---|---|---|
| `too many decimal points in number` | 把 `UTHASH_VERSION` / `UTARRAY_VERSION` 当整数用 | 只能字符串化（见 1.2） |
| `invalid type argument of unary '*'` / 类型不匹配 | `HASH_FIND_INT` 第 2 参传了值而不是地址 | 改为 `HASH_FIND_INT(head, &key, out)` |
| `request for member 'hh' in something not a structure` | 句柄成员名不是 `hh`，却用了硬编码 `hh` 的宏 | 换泛型宏并显式传句柄名（见 2.3） |
| `'hh_id' undeclared` 一类错误 | 泛型宏参数顺序写错（把元素指针写到了键字段位置） | 核对 `(hh, head, 键字段, keylen, 元素指针)` |

### 6.2 运行期错误

| 现象 | 原因 | 解决 |
|---|---|---|
| 首次插入即崩溃 | 表头指针未初始化为 `NULL` | 声明处写 `T *head = NULL;` |
| 崩溃 / 野指针访问 | `free` 了仍在表内的元素，或 `HASH_DEL` 后重复 `free` | 严格执行"先 `HASH_DEL` 再 `free`"，且只 free 一次 |
| 遍历删除时随机崩溃 | 用 `for (x = h; x; x = x->hh.next)` 且释放了当前元素 | 改用 `HASH_ITER` |
| 内存持续增长 | 只 `HASH_CLEAR` / `HASH_REPLACE` 而未释放元素 | 这两者都不释放元素，需调用者处理 |
| 长度变成 `4294967295` | 对空数组调用 `utarray_pop_back`（无符号下溢） | 先判断 `utarray_len(a) > 0` |
| `printf` 输出异常 / 告警 | `utarray_eltidx` 返回类型在 64 位下不是 `unsigned` | 用 `(long long)` 或 `%lld` 打印 |
| `-Wtype-limits` 告警：`comparison of unsigned expression in '< 0' is always false` | 给 `utarray_insert` / `utarray_resize` 传了字面量 `0` | 用变量传下标，或对 utarray 头文件关闭该告警 |

### 6.3 逻辑错误（不报错但结果错）

| 现象 | 原因 | 解决 |
|---|---|---|
| 查找突然失败 | 字符串键缓冲区被修改或提前 `free` | 键单独 `strdup` 保存，或在修改前先 `HASH_DEL` |
| 用不同 `keylen` 查找失败 | 插入与查找的 `keylen` 不一致 | 两边用同一表达式，例如都是 `strlen(key)` |
| 计数比预期多 | `HASH_ADD*` 不去重 | 先查后插，或用 `HASH_REPLACE*` |
| `utarray_find` 找不到已存在的元素 | 数组未排序（bsearch 前提） | 先 `utarray_sort`，并用同一个比较函数 |
| `HASH_SORT` 结果看似"按别的字段排" | 比较函数写成了 qsort 风格 | 改为 `int cmp(T *a, T *b)` |
| `utarray_eltptr(a, i++)` 行为异常 | 宏参数被求值两次，`i` 多自增一次 | 先算下标到变量，再传给宏 |

### 6.4 排错步骤

- [ ] **第一步：确认键与所有权**——键缓冲区是否存活且未改动？元素是否只被释放一次？
- [ ] **第二步：缩小到最小复现**——把表/数组规模降到 2–3 个元素，逐个打印 `HASH_COUNT` / `utarray_len` 与关键字段。
- [ ] **第三步：打开诊断开关**——uthash 加 `-DHASH_DEBUG` 使 `HASH_FSCK` 生效；编译器加 `-Wall -Wextra -fsanitize=address` 能直接指出越界与重复释放。

> 记法：**崩溃先看"表头是否初始化 + 元素是否被重复释放"；查不到先看"键是否被改 + 数组是否已排序"。**

---

## 附录 A 速查表

**uthash 常用宏**

| 宏 | 说明 |
|---|---|
| `HASH_ADD_INT(head,intfield,add)` | 以 `int` 字段为键插入 |
| `HASH_ADD_STR(head,strfield,add)` | 以字符串字段为键插入（键不复制，`keylen = strlen`） |
| `HASH_ADD_PTR(head,ptrfield,add)` | 以指针字段为键插入 |
| `HASH_FIND_INT(head,&key,out)` | 按 `int` 键查找（**传地址**） |
| `HASH_FIND_STR(head,key,out)` | 按字符串键查找（**传字符串，不加 `&`**） |
| `HASH_FIND_PTR(head,&key,out)` | 按指针键查找（传地址） |
| `HASH_REPLACE_INT(head,intfield,add,replaced)` | 同键替换，旧元素由 `replaced` 返回且不释放 |
| `HASH_DEL(head,delptr)` | 从表中摘除元素（不释放内存） |
| `HASH_COUNT(head)` | 元素个数 |
| `HASH_ITER(hh,head,el,tmp)` | 安全遍历（可取后继后再删） |
| `HASH_SORT(head,cmp)` | 排序，`cmp` 收**元素指针** |
| `HASH_ADD(hh,head,fieldname,keylen,add)` | 泛型插入，支持自定义句柄名 |
| `HASH_ADD_KEYPTR(hh,head,keyptr,keylen,add)` | 键位于结构体之外 |
| `HASH_FIND(hh,head,keyptr,keylen,out)` | 泛型查找 |
| `HASH_DELETE(hh,head,delptr)` / `HASH_DELETE_HH(hh,head,&el->hh)` | 泛型删除 |
| `HASH_CNT(hh,head)` / `HASH_SRT(hh,head,cmp)` | 泛型计数 / 排序 |
| `HASH_CLEAR(hh,head)` | 只释放桶与表，不释放元素 |
| `HASH_OVERHEAD(hh,head)` | 表结构占用字节数 |
| `HASH_VALUE(keyptr,keylen,hashv)` / `HASH_FIND_BYHASHVALUE(hh,head,keyptr,keylen,hashv,out)` | 预计算并复用哈希 |
| `HASH_FSCK(hh,head,"where")` | 一致性自检（需 `HASH_DEBUG`） |
| `HASH_SELECT(hh_dst,dst,hh_src,src,cond)` | 按条件把元素复制到另一张表 |

**utarray 常用宏**

| 宏 | 说明 |
|---|---|
| `utarray_new(a,&icd)` / `utarray_free(a)` | 创建 / 销毁（含元素析构） |
| `utarray_init(a,&icd)` / `utarray_done(a)` | 栈上数组的初始化 / 清理 |
| `utarray_push_back(a,&v)` | 尾部追加（传元素地址） |
| `utarray_pop_back(a)` | 尾部删除（有 `dtor` 则先析构；空数组会下溢） |
| `utarray_extend_back(a)` | 追加一个未初始化元素 |
| `utarray_len(a)` | 元素个数 |
| `utarray_eltptr(a,i)` | 第 `i` 个元素地址（越界 `NULL`） |
| `utarray_front(a)` / `utarray_back(a)` | 首 / 尾元素地址 |
| `utarray_next(a,e)` / `utarray_prev(a,e)` | 迭代（`e = NULL` 起头 / 起尾） |
| `utarray_eltidx(a,e)` | 元素地址 → 下标 |
| `utarray_insert(a,&v,i)` / `utarray_replace(a,&v,i)` | 插入 / 覆盖 |
| `utarray_erase(a,pos,len)` | 删除区间 |
| `utarray_inserta(a,w,i)` / `utarray_concat(dst,src)` | 整段插入 / 追加 |
| `utarray_resize(a,n)` | 改长度（缩小析构，扩大初始化） |
| `utarray_clear(a)` / `utarray_renew(a,&icd)` | 清空元素 / 按需创建或清空 |
| `utarray_sort(a,cmp)` / `utarray_find(a,&v,cmp)` | qsort / bsearch（查找前必须排序） |

**预定义描述符**

| 描述符 | 元素 | 复制 | 析构 |
|---|---|---|---|
| `ut_int_icd` | `int` | 无（`memcpy`） | 无 |
| `ut_ptr_icd` | `void *` | 无（仅复制指针） | 无 |
| `ut_str_icd` | `char *` | 有（`malloc` + `strcpy`） | 有（`free`） |

**编译与调试开关**

| 开关 | 作用 |
|---|---|
| `-DHASH_DEBUG` | 启用 `HASH_FSCK` 一致性自检 |
| 定义 `HASH_FUNCTION` | 替换默认哈希函数（默认 `HASH_JEN`） |
| 定义 `HASH_BLOOM`（位数） | 启用 Bloom 过滤器，加速"键不存在"的判断 |
| 定义 `uthash_malloc` / `uthash_free` | 替换 uthash 的内存分配函数 |
| 定义 `uthash_fatal(msg)` | 替换 OOM 时的默认行为（默认 `exit(-1)`） |
| 定义 `utarray_oom()` | 替换 utarray 越界分配失败时的默认行为（默认 `exit(-1)`） |
| `-fsanitize=address` | 排查越界、重复释放、释放后使用 |

---

## 附录 B 学习路径与进度表

| 阶段 | 对应章节 | 内容 | 目标 | 状态 |
|---|---|---|---|---|
| 1 | 1 | 头文件与最小示例 | 能编译并通过两个最小示例 | ☐ |
| 2 | 2 | 结构体注册与宏家族 | 记住"增删查"三件套与句柄命名规则 | ☐ |
| 3 | 3 | 键与所有权 | 能说清键何时失效、谁负责 `free` | ☐ |
| 4 | 4 | 遍历排序与诊断 | 会用 `HASH_ITER`、`HASH_SORT`、`HASH_OVERHEAD` | ☐ |
| 5 | 5 | utarray | 会选 `UT_icd`，会用 `sort` / `find` / `erase` | ☐ |
| 6 | 6 | 排错 | 能区分编译期、运行期、逻辑错误三类问题 | ☐ |

---

## 附录 C 后续扩展（当前缺口）

正文 6 节覆盖了"日常读写 + 排错"的主干，以下为**尚未展开**的部分（斜体为正文已有雏形、待补完整示例）：

1. `utlist.h`（双向链表）、`utstring.h`（动态字符串）、`utringbuffer.h`（环形缓冲）的对照速查
2. *`HASH_SELECT` / `HASH_BLOOM`* 的完整示例与适用场景（正文附录 A 只列了签名）
3. 自定义 `HASH_FUNCTION` 与内存分配钩子的性能对比
4. 线程安全用法：uthash / utarray 均**不带锁**，需与 `pthread_mutex` 组合的最小封装示例
5. 与 C++ 混用时的注意事项（结构体 POD 要求、`void*` 转换、异常安全）
6. 完整可运行示例工程（哈希表 + 动态数组 + 单元测试，可 `git clone` 直接跑）

> 告诉我"从第 X 节开始填充"或"填充某节"，我会按框架逐节展开。
