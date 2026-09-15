# CMake 学习指南（C/C++ 构建系统）

> 用途：从零到上手 CMake 的路线图 + 要点笔记。面向**熟悉 C/C++ 但没用过 CMake（或只抄过别人 CMakeLists）**的读者。
> 使用方式：每学完一个小点，把 `[ ]` 改成 `[x]`；需要展开某节可随时让我补充。
> 定位：解决 **"CMakeLists.txt 到底在干嘛、怎么写、怎么排错"**，从"能编过一个项目"到"能写一个规范的现代 CMake 工程"。
> 背景贴合：你已在学 C 语言高级内容（`c-advanced-learning.md`），本文档补上"怎么用 CMake 组织 C/C++ 工程"这一课。
> 前提：示例基于 **CMake 3.16+**（`CMakePresets.json` 需 3.19+）；代码示例统一用 `cmake -S . -B build` 风格。

## 章节速览

| 阶段 | 章节 | 一句话 |
|---|---|---|
| 入门 | 1. CMake 是什么 | 它是"生成构建脚本的构建脚本"，管工程层面 |
| 入门 | 2. 最小可运行示例 | 两段式：配置 → 构建，先跑通再说 |
| 核心 | 3. 核心语法 | 变量、目标、`target_*` 三件套 |
| 核心 | 4. 作用域 | `PRIVATE` / `PUBLIC` / `INTERFACE`（最容易搞错） |
| 核心 | 5. 工程组织 | `src`/`include` 规范目录 + 子目录工程 |
| 进阶 | 6. 依赖管理 | `find_package` 找已装、`FetchContent` 拉源码 |
| 进阶 | 7. 配置与工具链 | `-D` / `-G` / `CMAKE_BUILD_TYPE` / 编译器 |
| 进阶 | 8. 工具衔接 | IDE、`compile_commands.json`、presets、`install` |
| 必备 | 9. 排错 | 报错对照表 + 三步法 + 清缓存重来 |
| 附录 | A / B / C | 命令速查 / 进度表 / 后续扩展 |

> 站点右侧另有自动大纲可跳转；此表用于快速判断"该看哪一节"。

---

## 1. CMake 是什么、为什么需要它

> 覆盖：CMake 的定位与职责边界、与 `gcc/g++` 的分工、最小概念模型。

- [ ] **CMake**：跨平台的 C/C++ 构建系统（"Make 的跨平台替代"），本质是**生成构建脚本的构建脚本**
  - 输入：`CMakeLists.txt`（一种声明式脚本语言）
  - 输出：各平台的原生构建文件——Linux/macOS 生成 `Makefile`，Windows 生成 Visual Studio 工程，或 Ninja / MinGW 的构建文件
- [ ] **为什么用它**：
  1. **跨平台**：同一份 `CMakeLists.txt` 能在 Windows / Linux / macOS 上编译
  2. **处理依赖**：自动（而非手动）查找第三方库（`find_package`）
  3. **抽象了"怎么编、编成什么"**：库 / 可执行文件 / 安装 / 测试都是声明式配置
- [ ] **与 `gcc`/`g++` 的分工**：`gcc`/`g++` 解决的是**单文件命令行**层面的事，CMake 负责**工程层面**的编排。
- [ ] **基本组成**：每个项目有一个或多个 `CMakeLists.txt`；顶层文件用 `add_subdirectory` 组织子模块。
- [ ] **两个关键概念**：
  - **`CMakeLists.txt`**：CMake 脚本，描述"目标（target）、依赖、编译选项"
  - **构建目录（`build/`）**：存放生成文件的目录，**建议与源码目录分离**（out-of-source build）
- [ ] 一句话：**CMake = 用声明式脚本描述"工程结构"，再自动生成各平台的编译配置。**

> 记法：CMake 是"中间层"——你写一份声明，它帮你生成 Makefile / VS 工程 / Ninja。

---

## 2. 最小可运行示例（先跑通）

> 覆盖：目录结构、最小 `CMakeLists.txt` 逐行解释、配置/构建两阶段、运行验证。

- [ ] **目录结构**：

  ```text
  hello/
  ├── CMakeLists.txt
  └── main.c
  ```

- [ ] **`CMakeLists.txt`（最小版）**：

  ```cmake
  cmake_minimum_required(VERSION 3.16)
  project(Hello C)

  add_executable(hello main.c)
  ```

  - `cmake_minimum_required`：声明最低 CMake 版本（**必须第一行**，很多报错源于版本太低）
  - `project(名字 语言)`：声明工程名与语言（`C` / `CXX` / 混合）
  - `add_executable(目标名 源文件...)`：生成可执行文件

- [ ] **两段式流程（配置 → 构建）**：

  ```bash
  cmake -S . -B build     # 1. 配置阶段：读取 CMakeLists.txt，生成构建文件到 build/
  cmake --build build     # 2. 构建阶段：调用底层编译器，产出可执行文件
  ```

  - 老写法等价于：`mkdir build && cd build && cmake .. && cmake --build .`（**推荐用 `-S`/`-B`**，不用切目录）
  - `-S` 指定源码目录，`-B` 指定构建目录

- [ ] **配置阶段 vs 构建阶段**——理解这两段是排错的关键，全文只在这里详述：
  - **配置（Configure）**：跑 `CMakeLists.txt`，做变量 / 依赖 / 生成器决策，产出 `Makefile`、`CMakeCache.txt`
  - **构建（Build）**：调用真正的编译器（`gcc`/`g++`），把源文件编成目标文件，再链接成可执行文件或库
- [ ] **验证运行**：`./hello`（Linux/macOS）或 `hello.exe`（Windows）。

> 记法：**改了 `CMakeLists.txt` 必须重新配置**（重跑 `cmake -S . -B build`）；只改源码则免配置，直接 `cmake --build build`。

---

## 3. 核心语法：变量、目标与 `target_*` 属性

> 覆盖：变量与引用、目标类型、`target_*` 三件套、编译标准与编译选项（集中在本文）。

- [ ] **变量与变量引用**：

  ```cmake
  set(MY_SOURCES main.c util.c)      # 定义变量
  add_executable(app ${MY_SOURCES})  # 用 ${变量名} 引用
  set(CMAKE_C_STANDARD 11)           # 全局设 C 标准（目标级写法见下）
  ```

- [ ] **目标类型**：

  | 命令 | 作用 |
  |---|---|
  | `add_executable(name src...)` | 可执行文件 |
  | `add_library(name src...)` | 库：默认 `STATIC`，可指定 `SHARED`（动态库）、`INTERFACE`（纯头文件接口库） |
  | `add_library(name STATIC src...)` | 显式静态库 |
  | `add_library(name SHARED src...)` | 共享库（`.so` / `.dll` / `.dylib`） |

- [ ] **给目标加属性（`target_*` 指向）**：

  ```cmake
  add_library(mylib STATIC impl.c)
  target_include_directories(mylib PUBLIC include)   # 头文件路径
  target_compile_features(mylib PUBLIC c_std_11)     # 要求 C11
  target_link_libraries(app PRIVATE mylib)           # 链接依赖
  ```

- [ ] **`target_*` 三件套（现代 CMake 的核心，务必掌握）**：

  | 命令 | 用途 |
  |---|---|
  | `target_include_directories` | 头文件搜索路径 |
  | `target_compile_features` / `target_compile_options` | 编译标准 / 编译选项 |
  | `target_link_libraries` | 链接库 |

- [ ] **编译标准写法二选一**：
  - 全局：`set(CMAKE_C_STANDARD 11)`（所有目标生效，老写法）
  - 目标级（**推荐**）：`target_compile_features(app PRIVATE c_std_11)`（只作用于该目标）
  - 一般配套 `set(CMAKE_C_STANDARD_REQUIRED ON)`，避免"标准不满足就降级"
- [ ] **编译选项**：`target_compile_options(app PRIVATE -Wall -Wextra -Wpedantic)`；MSVC 下是 `/W4`，可用 `if(MSVC)` 分支处理。

> 记法：现代 CMake 的三大件是 `add_executable` / `add_library` + `target_*` 加属性。**作用于目标（target）比"全局 set 变量"更干净**，这正是"现代 CMake"与老写法的区别。

---

## 4. 作用域：`PRIVATE` / `PUBLIC` / `INTERFACE`（重点）

> 覆盖：三种作用域的语义与传播规则、判断口诀、常见误用。这是全篇最容易混淆、也最能体现"现代 CMake"的一节。

- [ ] **三种作用域**：
  - `PRIVATE`：只对本目标生效，依赖**不传播**给下游
  - `INTERFACE`：只传给下游（本目标自己不用；如"下游编我的头文件时才需要的 include 目录"）
  - `PUBLIC`：本目标 + 下游都用
- [ ] **怎么判断**（口诀）：
  - 只在**自己**编译时用 → `PRIVATE`
  - 只在**别人**编译时用 → `INTERFACE`
  - **两边都要** → `PUBLIC`
- [ ] **例子**：`target_link_libraries(app PRIVATE libA)` 表示 `libA` 只给 `app` 用；`app` 的调用方不需要知道 `libA` 的存在。
- [ ] **常见误用**：
  - 用 `include_directories()`（全局、旧式）代替 `target_include_directories()`（目标级、推荐）——前者会把路径泄漏给所有目标
  - 库的头文件目录写成 `PRIVATE`，导致下游 `#include` 不到

> 记法：**`PRIVATE` 自私、`INTERFACE` 利他、`PUBLIC` 兼济**；拿不准时先写 `PRIVATE`，编不过再放宽。

---

## 5. 工程组织：规范目录与多目录工程

> 覆盖：推荐目录结构、可直接套用的顶层 `CMakeLists.txt`、`add_subdirectory` 子模块、子目录 target 的串联方式。

- [ ] **推荐目录结构**：

  ```text
  myapp/
  ├── CMakeLists.txt          # 顶层
  ├── include/                # 公共头文件
  │   └── myapp.h
  ├── src/                    # 源码
  │   ├── main.c
  │   ├── CMakeLists.txt      # 子目录（可选）
  │   └── util.c
  └── tests/                  # 测试（可选）
  ```

- [ ] **顶层 `CMakeLists.txt`（示范）**：

  ```cmake
  cmake_minimum_required(VERSION 3.16)
  project(MyApp VERSION 1.0.0 LANGUAGES C)

  set(CMAKE_C_STANDARD 11)
  set(CMAKE_C_STANDARD_REQUIRED ON)

  add_executable(myapp src/main.c src/util.c)
  target_include_directories(myapp PRIVATE include)
  target_compile_options(myapp PRIVATE -Wall -Wextra)   # 严格告警（gcc/clang）
  ```

- [ ] **`project` 的 `VERSION`**：`project(MyApp VERSION 1.0.0)` 定义工程版本，可用于 `install` 规则与 `configure_file` 生成版本头。
- [ ] **子目录（模块化）**：

  ```cmake
  add_subdirectory(src)                       # 进入 src/ 下的 CMakeLists.txt
  target_link_libraries(app PRIVATE sublib)   # 引用子目录里定义的目标
  ```

- [ ] **组织要点**：源文件进 `src/`、头文件进 `include/`、顶层管整体、子目录管模块，目标之间用 `target_link_libraries` 串联。

> 记法：**顶层只做"项目级声明 + 组装"，具体目标的属性写在定义它的那个 `CMakeLists.txt` 里**（就近原则，避免全局变量满天飞）。

---

## 6. 依赖管理：`find_package` 与 `FetchContent`

> 覆盖：`find_package` 的用法与两种模式、找不到库时的兜底、`FetchContent` 拉源码、两者的选择标准。

- [ ] **`find_package`（找系统已安装的库）**：

  ```cmake
  find_package(ZLIB REQUIRED)
  target_link_libraries(app PRIVATE ZLIB::ZLIB)
  ```

  - `REQUIRED`：找不到就**报错**（不加则视为可选）；`QUIET`：找不到也不打扰
  - 找到的库通常以 `名字::名字` 的"导入目标"形式使用（`ZLIB::ZLIB`、`OpenCV::opencv`）
- [ ] **两种查找模式（理解即可）**：
  - **Module 模式**：CMake 自带 `FindXXX.cmake`，只覆盖常见库（`ZLIB`、`CURL` 等）
  - **Config 模式**：库自带 `XXXConfig.cmake`，更通用、更新（`OpenCV`、`Qt`、`gtest` 多用这种）
- [ ] **找不到库时的兜底**：手动指定查找路径：

  ```cmake
  set(CMAKE_PREFIX_PATH /path/to/install)   # 前缀搜索路径
  set(ZLIB_ROOT /path/to/zlib)              # 某些库用 XXX_ROOT
  ```

- [ ] **常见第三方库示例**：

  ```cmake
  find_package(Threads REQUIRED)
  target_link_libraries(app PRIVATE Threads::Threads)

  find_package(OpenCV REQUIRED)
  target_link_libraries(app PRIVATE ${OpenCV_LIBS})   # 老写法；新的用 OpenCV::opencv
  ```

- [ ] **`FetchContent`（把第三方源码直接拉进工程，现代常用）**：

  ```cmake
  include(FetchContent)
  FetchContent_Declare(googletest
      GIT_REPOSITORY https://github.com/google/googletest.git
      GIT_TAG v1.14.0)
  FetchContent_MakeAvailable(googletest)
  ```

- [ ] **怎么选**：系统/包管理器里已经有 → `find_package`；希望锁定版本、随工程一起编 → `FetchContent`。

> 记法：`find_package` 找"系统已装"，`FetchContent` 拉"远程源码"，两者最终都通过**目标**链接（`X::X` 或 target 名）。

---

## 7. 配置与工具链：选项、构建类型、生成器、编译器

> 覆盖：`option` 开关、`CMAKE_BUILD_TYPE`、生成器（`-G`）、指定编译器、命令行传参（`-D`）。

- [ ] **`option`（开关，可让用户用 `-D` 打开/关闭）**：

  ```cmake
  option(BUILD_TESTING "构建测试" ON)   # 默认开
  if(BUILD_TESTING)
      enable_testing()
  endif()
  ```

- [ ] **`CMAKE_BUILD_TYPE`（编译配置）**：`Debug` / `Release` / `RelWithDebInfo` / `MinSizeRel`
  - 一般 `Release` 开 `-O2/-O3`，`Debug` 加符号 `-g` 且不优化（利于 gdb 调试）
- [ ] **生成器（generator：底层用哪个构建工具）**：
  - 默认：Unix 用 `Unix Makefiles`；Windows 用 Visual Studio 工程
  - 换更快的：`-G Ninja`（跨平台，推荐）；Windows 上用 MinGW 工具链时选 `-G "MinGW Makefiles"`
- [ ] **指定编译器**：`CMAKE_C_COMPILER` / `CMAKE_CXX_COMPILER`
- [ ] **常用命令行参数（覆盖绝大多数需求）**：

  ```bash
  cmake -S . -B build -G Ninja                        # 选生成器
  cmake -S . -B build -DCMAKE_BUILD_TYPE=Release      # 选优化级别
  cmake -S . -B build -DBUILD_TESTING=ON              # 传 option / 变量
  cmake -S . -B build -DCMAKE_C_COMPILER=gcc \
                      -DCMAKE_CXX_COMPILER=g++         # 指定编译器
  ```

  > Windows（PowerShell）同一条命令同样可用；只是生成器默认变成 Visual Studio 工程。

> 记法：`-D` 传变量、`-G` 选生成器、`-DCMAKE_BUILD_TYPE` 选优化级别，这三个参数覆盖绝大多数场景。

---

## 8. 与 IDE / 实际工具的衔接（含安装打包，进阶）

> 覆盖：写 C++ 工程、CMake 与 Makefile 的关系、IDE 集成、`compile_commands.json`、`CMakePresets.json`、`install` 安装规则。

- [ ] **写 C++ 项目**：`project(... LANGUAGES C CXX)`，用 `target_compile_features(app PRIVATE cxx_std_17)` 指定标准。
- [ ] **CMake 与 Makefile 的关系**：CMake 生成 Makefile（或 Ninja / VS 工程），实际编译仍由这些底层构建工具执行。
- [ ] **VS Code / CLion 集成**：IDE 直接读 `CMakeLists.txt` 自动完成配置、智能提示与编译。
- [ ] **`compile_commands.json`（给 IDE / 静态分析用）**：

  ```bash
  cmake -S . -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
  ```

  产出后供 clangd / clang-tidy 等做精确跳转与检查。
- [ ] 【进阶】**`CMakePresets.json`（现代工程规范）**：把常用配置 / 构建命令存成 preset，一键复用（CMake 3.19+）。
- [ ] 【进阶】**安装规则 `install`**：`install(TARGETS ... DESTINATION ...)` 把库 / 头文件 / 可执行文件装到系统或指定目录；配合前面的 `project(... VERSION ...)` 可实现版本化安装（打包 CPack 见附录 C）。

> 记法：CMake 只做"构建编排"，真正编译交给底层工具；IDE 通过读 `CMakeLists.txt` 或 `compile_commands.json` 干活。

---

## 9. 常见报错与排错（重点）

> 覆盖：高频报错对照表、排错三步法、清缓存重来。排查时先回到第 2 节的"配置 / 构建两阶段"判断。

- [ ] **高频报错对照表**：

  | 报错 | 原因 | 解决 |
  |---|---|---|
  | `CMake Error: Could not find ...` | `find_package` 找不到库 | 检查是否安装、设 `CMAKE_PREFIX_PATH` / `XXX_ROOT` |
  | `add_library cannot create target ... because another target with the same name already exists` | 目标名重复 | 改目标名，或调整 `add_library` 位置 |
  | `The C compiler ... is not able to compile a simple test program` | 编译器没装 / 路径错 | 检查 `gcc`，设 `CMAKE_C_COMPILER` |
  | `CMakeLists.txt: xx: error: ...` | 语法 / 版本问题 | 看行号；常见是 `cmake_minimum_required` 版本太高或命令拼错 |
  | `undefined reference to ...` | 链接时找不到符号 | 没 `target_link_libraries` 到对应库，或库没被包含 |
  | `No rule to make target ...` | 构建清单滞后 | 重新配置：`cmake -S . -B build` |
  | `fatal error: xxx.h: No such file` | 头文件路径没配 | `target_include_directories` 加路径 |

- [ ] **排错三步法**：
  1. **先判断是配置报错还是构建报错**（阶段不同，处理方向完全不同）
  2. **配置报错**：通常是 `CMakeLists.txt` 的语法 / 依赖 / 版本问题 → 重跑 `cmake -S . -B build` 看完整信息
  3. **构建报错**：编译器 / 链接器错误 → 看具体 `.c` 文件与链接库
- [ ] **清掉缓存重来（万能重定位）**：

  ```bash
  rm -rf build && cmake -S . -B build && cmake --build build
  ```

  ```powershell
  Remove-Item -Recurse -Force build; cmake -S . -B build; cmake --build build
  ```

  很多"灵异问题"清空 `build/` 后消失。

> 记法：**配置错 = 脚本 / 依赖问题，构建错 = 代码 / 链接问题**。分不清就先删掉 `build/` 重来。

---

## 附录 A 常用命令速查

| 命令 / 变量 | 用途 | 见 |
|---|---|---|
| `cmake_minimum_required(VERSION x.y)` | 声明最低版本（必须第一行） | §2 |
| `project(名 LANGUAGES C CXX VERSION x)` | 声明工程、语言、版本 | §2 / §5 |
| `add_executable` / `add_library` | 定义可执行文件 / 库目标 | §3 |
| `target_include_directories` | 目标级头文件路径 | §3 |
| `target_compile_features` / `target_compile_options` | 编译标准 / 编译选项 | §3 |
| `target_link_libraries` | 目标级链接依赖（配 `PRIVATE/PUBLIC/INTERFACE`） | §3 / §4 |
| `set(变量 值)` / `${变量}` | 定义 / 引用变量 | §3 |
| `add_subdirectory(dir)` | 纳入子目录工程 | §5 |
| `find_package(X REQUIRED)` | 查找系统已安装的库 | §6 |
| `FetchContent_Declare` / `MakeAvailable` | 拉取远程源码依赖 | §6 |
| `option(名 "说明" ON)` | 定义开关 | §7 |
| `install(TARGETS ... DESTINATION ...)` | 安装规则 | §8 |
| `cmake -S . -B build` | 配置（生成构建文件） | §2 |
| `cmake --build build` | 构建 | §2 |
| `-D<变量>=<值>` / `-G <生成器>` | 传变量 / 选生成器 | §7 |
| `-DCMAKE_BUILD_TYPE=Release` | 选优化级别 | §7 |
| `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON` | 生成 `compile_commands.json` | §8 |

---

## 附录 B 学习路径与进度表

| 阶段 | 对应章节 | 内容 | 目标 | 状态 |
|---|---|---|---|---|
| 1 | §1 | 概念与最小示例 | 说得清 CMake 在构建链里的位置 | ☐ |
| 2 | §2 | 跑通第一个工程 | 会用 `-S`/`-B` 完成配置 + 构建 | ☐ |
| 3 | §3 | 核心命令 | 会用 `add_*` / `set` / `${}` / `target_*` | ☐ |
| 4 | §4 | 作用域 | 会判断该写 `PRIVATE` / `PUBLIC` / `INTERFACE` | ☐ |
| 5 | §5 | 工程组织 | 搭一个规范的 `src`/`include` 多目录工程 | ☐ |
| 6 | §6 | 依赖管理 | 会用 `find_package` 与 `FetchContent` | ☐ |
| 7 | §7 | 配置与工具链 | 会用 `-D` / `-G` / `CMAKE_BUILD_TYPE` | ☐ |
| 8 | §8 | 工具衔接（进阶） | 会出 `compile_commands.json`、了解 presets | ☐ |
| 9 | §9 | 排错 | 能区分配置错与构建错，会清缓存重来 | ☐ |

---

## 附录 C 后续扩展（当前缺口）

正文 9 节已覆盖"能写规范工程 + 能排错"的主干，以下为**尚未展开**的部分（斜体为正文已有雏形、待补完整示例）：

1. 完整可运行示例工程（顶层 + 子目录 + 源文件 + 测试，可 `git clone` 直接跑）
2. `find_package` 与 `FetchContent` 的完整对比示例（含离线/内网镜像场景）
3. 静态库 vs 动态库 vs 接口库（`INTERFACE`）的深入讲解与符号可见性
4. *`install` 与打包（CPack）*、`configure_file` 生成版本配置头（正文 §8 只给了 `install` 一行）
5. *排错反例集*：把 §9 的对照表补成"可复现的最小错误工程 + 修法"

> 告诉我"从第 X 节开始填充"或"填充某节"，我会按框架逐节展开。
