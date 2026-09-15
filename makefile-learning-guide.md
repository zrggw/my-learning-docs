# Makefile 学习指南（C/C++ 手工构建）

> 用途：从"照抄他人 Makefile"到"自己能写、能改、能排错"的路线图 + 要点笔记。面向**熟悉 C/C++ 与 gcc 命令行，但没系统写过 Makefile**的读者。
> 使用方式：每学完一个小点，把 `[ ]` 改成 `[x]`；需要展开某节可随时让我补充。
> 定位：讲清 **make 依据什么判定需要重新编译，以及 Makefile 的正确写法**，重点是**增量构建**与**依赖关系**。
> 背景贴合：你已在学 C 语言高级内容（`c-advanced-learning.md`）与 CMake（`cmake-learning-guide.md`）——本文补上"手写构建脚本"这一层，理解 CMake 生成的 Makefile 的实际内容。
> 前提：以 **GNU make 4.x** 与 gcc/clang 为例；Windows 上用 MinGW/MSYS2 的 `make`（原生 `nmake` 语法不同，见 6.3）。

---

## 核心速览（先看这一页，5 分钟覆盖 80% 内容）

**一句话定位**：make 是**时间戳驱动的增量构建工具**，只重新生成"比依赖更旧"的目标；Makefile 描述的是"产物、输入与生成方式"。

**① 最小可用 Makefile**（可直接复用于多文件工程）

```make
CC      := gcc
CFLAGS  := -Wall -Wextra -g
SRCS    := main.c util.c
OBJS    := $(SRCS:.c=.o)
TARGET  := app

$(TARGET): $(OBJS)
	$(CC) $(CFLAGS) $^ -o $@

%.o: %.c
	$(CC) $(CFLAGS) -c $< -o $@

clean:
	rm -f $(OBJS) $(TARGET)

.PHONY: clean
```

```bash
make            # 构建（只编译改过的文件）
make -j8        # 并行构建（8 路）
make clean      # 清理
```

> 注意：命令行的缩进**必须是 TAB**，不能是空格——这是 Makefile 最常见的错误（报 `missing separator`）。

**② 五个核心概念**（Makefile 的心智模型）

| 概念 | 一句话 |
|---|---|
| **规则 = 目标 : 依赖 + 命令** | 目标是产物，依赖是输入，命令是"怎么造"；三者缺一不可 |
| **TAB 缩进** | 每条命令行以一个 TAB 开头（不是空格），这是语法而非风格 |
| **自动变量** | `$@` 目标、`$<` 第一个依赖、`$^` 全部依赖、`$?` 更新的依赖——写通用规则全靠它们 |
| **时间戳驱动** | 依赖比目标新 → 重跑命令；否则跳过。make 的"智能"只有这一条规则 |
| **变量与展开时机** | `:=` 立即展开、`=` 用到才展开（新手优先使用 `:=`） |

**③ 最常用的 5 条命令 + 3 个变量**

| make 命令 | 作用 |
|---|---|
| `make` / `make 目标名` | 构建默认目标 / 指定目标 |
| `make -j8` | 并行构建（依赖写对才安全） |
| `make -n` | 只打印要执行的命令，不真跑（干跑） |
| `make -B` | 强制全部重建（忽略时间戳） |
| `make clean` | 清理产物（用 `.PHONY` 声明） |

| 变量 | 约定含义 |
|---|---|
| `CC` / `CXX` | 编译器（`gcc` / `g++`） |
| `CFLAGS` / `CXXFLAGS` | 编译选项（`-Wall -O2 -g -Iinclude`） |
| `LDFLAGS` / `LDLIBS` | 链接选项 / 要链接的库（`-Llib -lm`） |

**④ 三大高频报错**（完整对照表见第 6 节）

| 报错 | 第一反应 |
|---|---|
| `missing separator` | 命令行用了空格缩进 → 改成 TAB |
| `No rule to make target 'xxx.o'` | 缺规则或依赖名写错 → 检查模式规则 `%.o: %.c` 与文件名 |
| `undefined reference to ...` | 链接时少库/少对象文件 → 补 `LDLIBS`、补进 `OBJS` |

**阅读路线**：只想编译多文件工程 → 第 1 节；要写自己的规则 → 第 2 节；工程变大 → 第 3–4 节；出问题 → 第 6 节；查语法 → 附录 A。

---

## 1. 快速上手：make 是什么 + 最小工程

> 覆盖：make 的定位与时间戳驱动机制、与 CMake/gcc 的分工、最小多文件 Makefile 逐行解释。

### 1.1 make 是什么、和 CMake 什么关系

- [ ] **make**：一个读 `Makefile`、按**规则**和**文件时间戳**决定"哪些文件需要重新生成"的工具。本身不认识 C 语言，只负责**按你的命令去调编译器**。
- [ ] **make 的核心只有一条判断**：目标文件不存在，或**任一依赖比目标新** → 执行命令；否则什么都不做。这就是**增量构建**。
- [ ] **与 gcc 的分工**：`gcc main.c util.c -o app` 是"一句话全量编译"；make 解决的是"**改了一个文件，只重编它、再重新链接**"。
- [ ] **与 CMake 的分工**（对照 `cmake-learning-guide.md`）：

  | | Makefile | CMake |
  |---|---|---|
  | 层次 | 直接描述"怎么编译" | 描述"工程结构"，**生成** Makefile / Ninja / VS 工程 |
  | 跨平台 | 自己写 shell 命令，一般不跨平台 | 一份 `CMakeLists.txt` 到处编 |
  | 依赖查找 | 自己写 `-I` / `-L`，或用 `pkg-config` | `find_package` 自动找 |
  | 适用 | 小型工程、嵌入式、想完全掌控构建细节 | 中大型工程、多平台 |

  - **为什么要学 Makefile**：① 大量开源项目仍是 Makefile；② CMake 最终生成的还是 Makefile/Ninja 文件，看得懂才能排错；③ 交叉编译、内核模块等场景必须手写。

> 记法：**gcc 管"怎么编一个文件"，make 管"该编哪些文件"，CMake 管"整个工程怎么组织"。**

### 1.2 最小可运行的 Makefile

- [ ] **目录结构**：

  ```text
  hello/
  ├── Makefile
  ├── main.c
  └── util.c
  ```

- [ ] **Makefile（多文件版，可直接用）**：

  ```make
  CC      := gcc
  CFLAGS  := -Wall -Wextra -g
  SRCS    := main.c util.c
  OBJS    := $(SRCS:.c=.o)
  TARGET  := app

  $(TARGET): $(OBJS)
  	$(CC) $(CFLAGS) $^ -o $@

  %.o: %.c
  	$(CC) $(CFLAGS) -c $< -o $@

  clean:
  	rm -f $(OBJS) $(TARGET)

  .PHONY: clean
  ```

  - **第一行之前的变量区**：`CC`/`CFLAGS`/`SRCS` 都是普通变量，用 `$(名字)` 引用
  - `OBJS := $(SRCS:.c=.o)`：**后缀替换引用**——把 `main.c util.c` 变成 `main.o util.o`
  - `$(TARGET): $(OBJS)`：**链接规则**——目标 `app` 依赖两个 `.o`；`$^` 展开成全部依赖，`$@` 是目标名
  - `%.o: %.c`：**模式规则**——"任何 `.o` 都由同名 `.c` 生成"，`$<` 是那个 `.c`
  - `clean:` 与 `.PHONY`：伪目标（不是文件名），避免目录里真有 `clean` 文件时命令不执行

- [ ] **运行与验证**：

  ```bash
  make            # 第一次：编译两个 .c 并链接成 app
  ./app
  touch main.c    # 假装改了 main.c
  make            # 只重编 main.o 并重新链接（注意观察输出）
  make clean      # 删除 app 与两个 .o
  ```

  - 第二次 `make` 只编译了 `main.o`，这就是**增量构建**的直接证据
  - Windows/MinGW 下产物是 `app.exe`，规则里的目标名最好也带上 `.exe`，否则每次都会重新链接（见 6.3）

> 记法：**"目标 : 依赖" 下一行用 TAB 写命令**；变量区放最上面，规则区依次写「链接规则 → 模式规则 → 清理」。90% 的小工程用这个骨架就够了。

---

## 2. 核心语法：规则、变量、自动变量

> 覆盖：显式规则与伪目标、变量四种赋值方式、预定义变量与隐式规则、自动变量与模式规则、多目标与静态模式规则。

### 2.1 规则三要素与伪目标

- [ ] **一条规则 = 目标 + 依赖 + 命令**：

  ```make
  app: main.o util.o
  	$(CC) $^ -o $@          # ← 必须以 TAB 开头
  ```

  - 目标可以多依赖、多命令；**命令行的 TAB 不可省**
  - 依赖也可以是目标（递归推导）：`app` 依赖 `main.o`，`main.o` 又由 `%.o: %.c` 规则生成
- [ ] **默认目标**：Makefile 里**第一条规则的目标**就是 `make` 不带参数时的目标；习惯上写 `all`：

  ```make
  all: $(TARGET)
  ```
- [ ] **伪目标（`.PHONY`）**：目标名不是文件（如 `clean`、`all`、`test`、`install`）时一定要声明，否则目录里出现同名文件就会"命令不执行"：

  ```make
  .PHONY: all clean test
  ```
- [ ] **同一目标的多条命令互不影响**：每条命令行是独立的 shell，`cd` 不跨行生效：

  ```make
  bad:
  	cd build
  	pwd            # ← 还在原目录！
  good:
  	cd build && pwd
  ```

> 记法：**"目标 : 依赖" + TAB 命令 = 一条规则**；非文件名的目标一律进 `.PHONY`。

### 2.2 变量：四种赋值与预定义变量

- [ ] **四种赋值**：

  | 写法 | 时机 | 用途 |
  |---|---|---|
  | `VAR := value` | **立即**展开 | 推荐默认用这个，结果可预期 |
  | `VAR = value` | **延迟**展开（用到时才算） | 需要引用后面才定义的变量时用 |
  | `VAR ?= value` | 未定义才赋值 | 给用户留默认值（可被 `make VAR=x` 或环境变量覆盖） |
  | `VAR += value` | 追加 | 累加选项，如 `CFLAGS += -O2` |

  ```make
  CFLAGS := -Wall
  CFLAGS += -O2            # 追加 → -Wall -O2
  PREFIX ?= /usr/local     # 用户可用 make PREFIX=... 覆盖
  ```
- [ ] **引用与函数调用**：`$(VAR)`（推荐，带括号）；`$(VAR:.c=.o)` 后缀替换；`$(wildcard src/*.c)` 取文件列表。
- [ ] **预定义/约定变量**（make 自带默认值，改它们即可影响内置规则）：

  | 变量 | 默认 | 含义 |
  |---|---|---|
  | `CC` / `CXX` | `cc` / `g++` | C / C++ 编译器 |
  | `CFLAGS` / `CXXFLAGS` | 空 | 编译选项 |
  | `CPPFLAGS` | 空 | 预处理器选项（`-I`、`-D` 都放这里） |
  | `LDFLAGS` / `LDLIBS` | 空 | 链接选项（`-L`）/ 库（`-l`） |
  | `RM` | `rm -f` | 删除命令 |

- [ ] **命令行覆盖 & 环境变量**：`make CFLAGS="-O3 -g"` 可临时改（命令行 > Makefile 内 `=`；`override` 可强制）；环境变量默认会被 Makefile 里的赋值覆盖。
- [ ] **查看所有内置规则与变量**：`make -p`（输出很长，配合 `grep` 用，如 `make -p | grep -A2 '^%.o'`）。

> 记法：**新手先用 `:=`，需要"用户可覆盖的默认值"用 `?=`，累加选项用 `+=`**；`-I` 放 `CPPFLAGS`、`-O2/-g/-Wall` 放 `CFLAGS`、`-L` 放 `LDFLAGS`、`-lxxx` 放 `LDLIBS`——这是社区的通用约定，便于他人接手。

### 2.3 自动变量与模式规则

- [ ] **常用自动变量**（写通用规则的核心）：

  | 变量 | 含义 |
  |---|---|
  | `$@` | 规则的目标名（如 `main.o`） |
  | `$<` | **第一个**依赖（如 `main.c`） |
  | `$^` | **全部**依赖（去重，如 `main.o util.o`） |
  | `$?` | 比目标**新**的依赖（增量场景有用） |
  | `$*` | 模式规则的"词干"（`%.o` 匹配 `main.o` 时 `$*` 为 `main`） |
  | `$\|` | order-only 依赖（见 4.2，用于"目录必须先存在"） |

- [ ] **模式规则**：一次写好"所有 `.o` 从 `.c` 来"，不必逐个文件写规则：

  ```make
  %.o: %.c
  	$(CC) $(CPPFLAGS) $(CFLAGS) -c $< -o $@
  ```
  - make 自带类似的隐式规则；**显式写出来更好**——能加 `-MMD` 等选项、行为更可控
- [ ] **多目标 / 静态模式规则**（源文件分散在不同目录时更清晰）：

  ```make
  # 静态模式规则：对 $(OBJS) 里的每个元素应用 %.o: %.c
  $(OBJS): %.o: %.c
  	$(CC) $(CFLAGS) -c $< -o $@
  ```
- [ ] **生成多个产物**：`$@` 只指第一个目标，多产物要写 `$@` 的兄弟变量或用 `&:` 分组目标（GNU make 4.3+）。简单起见，拆成多条规则更稳。

> 记法：**记牢三个就够日常用：`$@` 目标、`$<` 第一个依赖、`$^` 全部依赖**；配合 `%.o: %.c` 一条模式规则，就能覆盖绝大多数 C/C++ 工程。

---

## 3. 工程组织：多目录与模块化

> 覆盖：`src`/`include`/`build` 目录布局、`VPATH` 与 `vpath`、把对象文件放进 `build/`、`include` 拆分多个 `.mk`、库的生成规则。

### 3.1 推荐目录布局

- [ ] **小工程**（源码在当前目录）：直接用第 1 节的骨架。
- [ ] **中等工程**（源码在 `src/`，产物进 `build/`）：

  ```text
  myapp/
  ├── Makefile
  ├── src/
  │   ├── main.c
  │   └── util.c
  ├── include/
  │   └── util.h
  └── build/                 # 产物与 .o 都放这里（保持源码干净）
  ```

  ```make
  CC      := gcc
  CFLAGS  := -Wall -Wextra -g -Iinclude
  BUILD   := build
  SRCS    := $(wildcard src/*.c)
  OBJS    := $(patsubst src/%.c,$(BUILD)/%.o,$(SRCS))
  DEPS    := $(OBJS:.o=.d)
  TARGET  := $(BUILD)/app

  $(TARGET): $(OBJS)
  	$(CC) $^ -o $@

  $(BUILD)/%.o: src/%.c | $(BUILD)
  	$(CC) $(CFLAGS) -MMD -MP -c $< -o $@

  $(BUILD):
  	mkdir -p $@

  -include $(DEPS)

  clean:
  	rm -rf $(BUILD)

  .PHONY: clean
  ```

  - `$(wildcard src/*.c)`：自动收集源文件，**新增文件不用改 Makefile**
  - `$(patsubst src/%.c,build/%.o,$(SRCS))`：把源文件路径映射成对象文件路径
  - `| $(BUILD)`：**order-only 依赖**——只保证"目录先建好"，目录时间戳变化不会触发重编（见 4.2）
  - `-MMD -MP` 与 `-include $(DEPS)`：自动处理头文件依赖（见第 4 节，**这一条是中等工程的必备**）
- [ ] **`VPATH` / `vpath`**：让 make 去别的目录找依赖（源码与 Makefile 分离时用）：

  ```make
  VPATH = src:include          # 全局搜索路径
  vpath %.c src                # 只对 .c 文件生效（更精确，推荐）
  ```
  - 注意：`VPATH` 只影响**查找依赖**，产物仍生成在当前目录；要控制产物位置还是得用 `build/%.o` 这类显式路径

### 3.2 模块化：拆分与复用

- [ ] **用 `include` 拆文件**（大工程按模块拆，主 Makefile 只做汇总）：

  ```make
  include config.mk        # 变量与开关
  include rules.mk         # 通用规则（如 %.o: %.c）
  include src/module.mk    # 各模块的目标
  ```
- [ ] **`config.mk` 常见内容**：交叉编译器前缀、`PREFIX`、调试开关、条件分支：

  ```make
  ifeq ($(DEBUG),1)
    CFLAGS += -O0 -g
  else
    CFLAGS += -O2
  endif
  ```
  - 用法：`make DEBUG=1`
- [ ] **子目录的两种风格**：① **递归 make**（顶层 `for d in ...; do $(MAKE) -C $$d; done`）；② **非递归 make**（一个 Makefile 管全工程，用 3.1 的 `wildcard` + `patsubst`）。**推荐非递归**——依赖关系完整、`-j` 并行安全、不用处理子 make 的变量传递。

  > 递归时子目录务必用 `$(MAKE)` 而不是 `make`（`$(MAKE)` 会传递 `-j` 等参数并标记递归）。

### 3.3 生成静态库与动态库

- [ ] **静态库**（`.a`，把对象文件打包）：

  ```make
  libutil.a: util.o
  	ar rcs $@ $^
  ```
- [ ] **动态库**（`.so`，源码需 `-fPIC`）：

  ```make
  libutil.so: util.o
  	$(CC) -shared -o $@ $^
  util.o: util.c
  	$(CC) $(CFLAGS) -fPIC -c $< -o $@
  ```
- [ ] **使用外部库**：`-I` 进 `CPPFLAGS`、`-L` 进 `LDFLAGS`、`-lxxx` 进 `LDLIBS`：

  ```make
  CPPFLAGS += -I/opt/mymath/include
  LDFLAGS  += -L/opt/mymath/lib
  LDLIBS   += -lmymath -lm
  ```
  - 用 `pkg-config` 省掉手写路径：`CFLAGS += $(shell pkg-config --cflags glib-2.0)`、`LDLIBS += $(shell pkg-config --libs glib-2.0)`
  - **静态库链接顺序敏感**：被依赖的库放后面（GNU ld 单遍扫描）

> 记法：**源码在 `src/`、头文件在 `include/`、产物在 `build/`**；用 `wildcard + patsubst` 自动收集，用 `vpath` 找源码，用 `include` 拆模块；**除非确有必要，否则不使用递归 make**。

---

## 4. 依赖关系与增量构建（重点）

> 覆盖：make 的判断逻辑、头文件依赖这一经典问题及其自动解法（`-MMD -MP`）、order-only 依赖、并行构建的安全前提。

### 4.1 修改头文件后未重新编译（经典问题）

- [ ] **现象**：修改 `util.h` 后，`make` 报 "Nothing to be done"，或只编译了未修改的文件 → 因为 Makefile 里**没有声明"`.o` 依赖 `.h`"**，make 只看到 `.o` 依赖 `.c`。
- [ ] **手工解法**（小工程可行，但要人工维护，容易漏）：

  ```make
  main.o: main.c util.h
  util.o: util.c util.h
  ```
- [ ] **自动解法（推荐）**：让编译器同时生成依赖文件，再由 make 读入：

  ```make
  CFLAGS += -MMD -MP          # -MMD 生成 .d；-MP 为每个头文件加空目标，防止头文件被删后报错
  DEPS   := $(OBJS:.o=.d)
  -include $(DEPS)            # 前置减号：.d 还不存在时不要报错
  ```

  - 首次构建：`.d` 还不存在，`-include` 静默跳过；编译后生成 `build/main.d`，内容形如 `build/main.o: src/main.c include/util.h`
  - 之后每次 `make` 都会读到这些依赖 → **改任何头文件，相关 `.o` 自动重编**

> 记法：**"改了头文件不重编"永远是同一类原因：依赖没写全。** 手写靠不住，直接上 `-MMD -MP` + `-include $(DEPS)`。

### 4.2 order-only 依赖：目录要先存在，但不该触发重编

- [ ] **问题**：把 `build/` 当普通依赖写（`$(OBJS): | ...` 之外的写法），目录时间戳一变，所有 `.o` 就被判为过期 → 全量重编。
- [ ] **解法**：竖线 `|` 右边是 **order-only 依赖**，只保证顺序，不参与时间戳比较：

  ```make
  $(BUILD)/%.o: src/%.c | $(BUILD)
  	$(CC) $(CFLAGS) -c $< -o $@
  ```
- [ ] 同理适用于"必须先生成的代码/配置头文件"：`main.o: main.c | generated.h`。

### 4.3 并行构建 `-j` 的安全前提

- [ ] `make -j8` 会按依赖图并行跑互不依赖的命令；**依赖写漏了就会偶发失败**（尤其是生成中间文件的规则）。
- [ ] 让 `-j` 安全的做法：把所有产物都写进规则的依赖里（包括 `.d`、生成的配置头、目录）；**不要**用 `cd` + 相对路径的"隐式"顺序假设。
- [ ] 排错手段：`make -j8` 偶发失败 → 先用 `make -j1` 复现；`make -n -j8` 看计划执行的顺序。

> 记法：**增量构建的正确性 100% 取决于依赖是否写全**；`-j` 只是把"依赖写漏"的后果从"偶尔重编"放大成"偶尔编译失败"。

---

## 5. 常用命令与调试

> 覆盖：常用 make 命令行选项、变量覆盖、打印变量与规则、干跑与调试输出。

- [ ] **常用命令行选项**：

  | 选项 | 作用 |
  |---|---|
  | `make` / `make 目标` | 构建默认目标 / 指定目标 |
  | `make -j N` | 并行 N 路（不加 N 则不限） |
  | `make -n` | 干跑：只打印命令不执行（配合 `-j` 看并行计划） |
  | `make -B` | 无视时间戳，全部重建 |
  | `make -k` | 遇到错误继续构建其他目标（一次性看全所有错误） |
  | `make -s` | 静默：不打印命令本身 |
  | `make -C dir` | 切到 dir 再执行 make（等价 `cd dir && make`） |
  | `make --debug=b` | 打印"谁被重建、为什么"（调依赖问题的利器） |
  | `make -p` | 打印内置规则与全部变量（配 `grep` 查隐式规则） |

- [ ] **变量覆盖**：`make CFLAGS="-O3 -g"`、`make DEBUG=1`、`make PREFIX=/opt`（在 Makefile 里用 `?=` 才能被覆盖）。
- [ ] **在 Makefile 里打印调试信息**：

  ```make
  $(info SRCS = $(SRCS))            # 纯打印（不报错）
  $(warning BUILD 未设置，使用默认值)  # 警告，继续执行
  $(error 必须先设置 CROSS_COMPILE)   # 报错并停止
  ```
- [ ] **查看某条规则的实际定义**：`make -p | grep -A3 '^build/main.o'`；或 `make --debug=b 目标` 看 make 的推导链。
- [ ] **清理与重建**：`make clean && make`；顽固问题 `make -B`。

> 记法：**调依赖用 `--debug=b`，查变量用 `$(info)`，验证命令用 `-n`，一次性看全部错误用 `-k`。**

---

## 6. 常见报错与排错（重点）

> 覆盖：高频报错对照表、排错三步法、清理重来，以及 Windows 上 make 的差异。

### 6.1 高频报错对照表

| 报错 | 原因 | 解决 |
|---|---|---|
| `missing separator. Stop.` | 命令行**用了空格缩进** | 改成 TAB（编辑器里设置"Tab 不转空格"或用 `cat -A` 检查 `^I`） |
| `No rule to make target 'xxx.o'` | 没有对应规则，或源文件路径不对 | 检查 `%.o: %.c` 规则、`vpath`/`VPATH`、文件名拼写 |
| `No rule to make target 'xxx.h'` | 头文件被依赖但不存在（常见于 `.d` 残留） | `-MP` 选项可缓解；或 `make clean` 后重来 |
| `undefined reference to 'foo'` | 链接时符号缺失 | 对象文件没进 `OBJS`；或库没写进 `LDLIBS`；静态库顺序问题 |
| `recipe for target 'xxx' failed` | 上面的命令本身失败（信息在前面） | 往上翻第一条真正的编译/链接错误 |
| `make: 'xxx' is up to date.` | 目标比依赖新（确实无变更时属正常） | 需要强制重建用 `make -B`；改了头文件没生效看 4.1 |
| `Nothing to be done for 'all'.` | `all` 没有依赖（只写了 `.PHONY`） | 给 `all` 加上真正的目标依赖 |
| `warning: overriding recipe for target` | 同一目标被定义两次 | 删掉重复规则，或改用不同目标名 |
| `*** missing separator` 出现在变量行 | 变量赋值行前有 TAB | 变量行顶格写，不要缩进 |
| `Permission denied`（执行 `./app`） | 输出文件不可执行 / 被占用 | 检查规则是否产出到预期路径；Windows 上被占用时关掉进程 |

### 6.2 排错三步法

- [ ] **第一步：确认"该不该重编"**——`make -n` 看它打算做什么；`make --debug=b` 看它为什么认为某个目标过期。分不清是"依赖写漏"还是"命令写错"。
- [ ] **第二步：看第一条错误**——make 会打印它执行的命令，编译错误的真正原因在**最上面那条**（`-k` 可一次看全）。
- [ ] **第三步：清干净重来**——`.d` 文件、半成品 `.o` 最容易导致异常现象：

  ```bash
  make clean && make
  ```
  ```powershell
  # Windows（PowerShell）
  Remove-Item -Recurse -Force build -ErrorAction SilentlyContinue; make
  ```

> 记法：**`missing separator` = 缩进问题；`No rule` = 依赖/规则缺失；`undefined reference` = 链接缺东西**。这三类覆盖了新手 80% 的报错。

### 6.3 Windows 平台差异

- [ ] 用 **MinGW / MSYS2** 的 `make`（`mingw32-make` 或 `make`）；路径分隔符建议用 `/`；注意 `\` 会被当作转义。
- [ ] **TAB 问题在 Windows 编辑器里更常见**（VS Code 右下角可切 Tab/空格；`.editorconfig` 里对 `Makefile` 设 `indent_style = tab`）。
- [ ] **原生 `nmake`（MSVC 自带）语法不同**：本文的 `:=`、`$(wildcard)`、`%.o: %.c` 大多不支持，不要混用。
- [ ] shell 差异：make 默认用 `sh`；Windows 上建议在 MSYS2/Git Bash 环境里跑，或显式 `SHELL := /bin/sh`。
- [ ] **产物名会多个 `.exe`**（实测发现的易错点，容易被误判为"增量构建失效"）：MinGW 的 gcc 收到 `-o app` 实际产出 `app.exe`，而规则里的目标名是 `app` → make 找不到该文件，**每次 make 都会重新链接**。解法：目标名带上后缀，或用变量区分平台：

  ```make
  ifeq ($(OS),Windows_NT)
    EXE := .exe
  else
    EXE :=
  endif
  TARGET := app$(EXE)          # Windows → app.exe；Linux/macOS → app
  ```
- [ ] **`rm` / `mkdir -p` 依赖 Unix shell**（实测发现的易错点）：若 make 使用 `cmd.exe` 作为 shell，`$(RM)`（默认 `rm -f`）会报"不是内部或外部命令"，而 `mkdir -p build` 会**多建一个名为 `-p` 的目录**。解法二选一：① 在 MSYS2 / Git Bash 里跑 make（推荐）；② 把这些命令写成平台安全的变量（Windows 下 `RM = del /q`、`MKDIR = mkdir`）。

---

## 附录 A 常用语法速查

**规则与目标**

| 语法 | 说明 |
|---|---|
| `目标: 依赖` + TAB 命令 | 显式规则（三要素） |
| `目标: 依赖 \| order-only` | 竖线右边只保证顺序、不比较时间戳 |
| `%.o: %.c` | 模式规则（一次定义所有同类文件） |
| `$(OBJS): %.o: %.c` | 静态模式规则（只对列表内元素生效） |
| `.PHONY: all clean` | 声明伪目标（不是文件） |
| `all: $(TARGET)` | 惯用默认目标（第一条规则才是真正的默认） |

**变量**

| 语法 | 说明 |
|---|---|
| `VAR := v` / `VAR = v` | 立即展开 / 延迟展开 |
| `VAR ?= v` / `VAR += v` | 未定义才赋值 / 追加 |
| `$(VAR)` | 引用（带括号更安全） |
| `$(VAR:.c=.o)` | 后缀替换引用 |
| `override VAR += v` | 防止被命令行覆盖（慎用） |

**自动变量**

| 变量 | 含义 |
|---|---|
| `$@` | 目标名 |
| `$<` | 第一个依赖 |
| `$^` | 全部依赖（去重） |
| `$?` | 比目标新的依赖 |
| `$*` | 模式规则的词干 |
| `$|` | order-only 依赖列表 |

**常用函数与预定义变量**

| 写法 | 用途 |
|---|---|
| `$(wildcard src/*.c)` | 按通配符取文件列表 |
| `$(patsubst src/%.c,build/%.o,$(SRCS))` | 批量替换路径/后缀 |
| `$(notdir ...)` / `$(dir ...)` | 取文件名 / 目录部分 |
| `$(shell pkg-config --cflags glib-2.0)` | 执行命令并取输出 |
| `$(info ...)` / `$(warning ...)` / `$(error ...)` | 打印 / 警告 / 报错停止 |
| `ifeq/ifneq/ifdef` | 条件分支 |
| `CC` `CXX` `CFLAGS` `CXXFLAGS` `CPPFLAGS` `LDFLAGS` `LDLIBS` `RM` | 约定变量（见 2.2） |

**命令行**

| 选项 | 作用 |
|---|---|
| `make` / `make 目标` | 构建默认 / 指定目标 |
| `-j N` | 并行构建 |
| `-n` / `-B` / `-k` / `-s` | 干跑 / 强制重建 / 出错继续 / 静默 |
| `-C dir` | 切目录执行 |
| `--debug=b` | 解释"为什么重建" |
| `-p` | 打印全部规则与变量 |
| `make VAR=value` | 覆盖变量（Makefile 用 `?=` 才允许） |

> 完整 C/C++ 工程骨架见第 1 节与 3.1；增量构建与头文件依赖见第 4 节。

---

## 附录 B 学习路径与进度表

| 阶段 | 对应章节 | 内容 | 目标 | 状态 |
|---|---|---|---|---|
| 1 | 1 | 概念 + 最小工程 | 说得清 make 靠时间戳做什么，会跑通多文件工程 | ☐ |
| 2 | 2 | 核心语法 | 会写规则、变量、`$@`/`$<`/`$^` 与模式规则 | ☐ |
| 3 | 3 | 工程组织 | 会搭 `src`/`include`/`build` 布局，会 `include` 拆模块 | ☐ |
| 4 | 4 | 依赖与增量 | 会用 `-MMD -MP`、order-only 依赖，`-j` 不出偶发失败 | ☐ |
| 5 | 5 | 命令与调试 | 会用 `-n` / `-B` / `-k` / `--debug=b` / `$(info)` | ☐ |
| 6 | 6 | 排错 | 能区分三类报错（缩进 / 缺规则 / 链接），会清理后重建 | ☐ |

> 时间紧的读法：**核心速览 → 第 1 节 → 第 4 节 → 第 6 节 → 附录 A**。第 4 节决定 Makefile 是否规范。

---

## 附录 C 后续扩展（当前缺口）

正文 6 节已覆盖"能写规范 Makefile + 能排错"的主干，以下为**尚未展开**的部分（斜体为正文已有雏形、待补完整示例）：

1. 完整可运行示例工程（`src/` + `include/` + 静态库 + 动态库 + 头文件依赖自动处理，可 `git clone` 直接跑）
2. make 函数进阶（`foreach` / `filter` / `call` / `eval` 与自动生成规则）
3. *递归 make* 与子目录工程的标准写法（含 `$(MAKE) -C` 的参数传递与并行）
4. *交叉编译*：`CROSS_COMPILE` 前缀、工具链变量组织（嵌入式 / ARM）
5. 与 CMake 的取舍与互操作：`cmake -G "Unix Makefiles"` 生成的 Makefile 怎么读、手动工程怎么平滑迁移到 CMake
6. `nmake`（MSVC）与 Windows 原生工具链的写法差异

> 告诉我"从第 X 节开始填充"或"填充某节"，我会按框架逐节展开。
