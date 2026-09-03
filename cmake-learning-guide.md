# CMake 学习指南（C/C++ 构建系统）

> 用途：从零到上手 CMake 的路线图 + 要点笔记。面向**熟悉 C/C++ 但没用过 CMake（或只抄过别人 CMakeLists）**的读者。
> 使用方式：每学完一项勾选 `[ ]`，需要展开某节可随时让我补充。
> 定位：解决 **"CMakeLists.txt 到底在干嘛、怎么写、怎么排错"** ，从"能编过一个项目"到"能写一个规范的现代 CMake 工程"。
> 背景贴合：你已在学 C 语言高级内容（`c-advanced-learning.md`），本文档补上"怎么用 CMake 组织 C/C++ 工程"这一课。

---

## 1. CMake 是什么、为什么需要它

- [ ] **CMake**：跨平台的 C/C++ 构建系统（"Make 的跨平台替代"），本质是**生成构建脚本的构建脚本**
  - 输入：`CMakeLists.txt`（一种声明式脚本语言）
  - 输出：各平台的原生构建文件——Linux/macOS 生成 `Makefile`，Windows 生成 `Visual Studio 工程` 或 `Ninja`/`MinGW` 的构建文件
- [ ] **为什么用它**：
  1. **跨平台**：同一份 `CMakeLists.txt` 能在 Windows/Linux/macOS 编译
  2. **处理依赖**：自动 vs. 手动找第三方库（`find_package`）
  3. **抽象了 "怎么编、编成什么"**：库/可执行文件/安装/测试都是声明式配置
  - 与 C++ 对照：我们平时用的 `gcc`/`g++` 是在**单文件命令行**层面，CMake 是**工程层面**的编排
- [ ] **基本组成**：每个项目一个或多个 `CMakeLists.txt`；顶层文件用 `add_subdirectory` 组织子模块。
- [ ] **两个关键概念**：
  - **`CMakeLists.txt`**：CMake 脚本，描述"目标（target）、依赖、编译选项"
  - **CMake 构建目录（build/）**: 放生成文件的目录，**建议和源码目录分离**（out-of-source build）
- [ ] 一句话：**CMake = 用声明式脚本描述"工程结构"，再自动生成各平台的编译配置。**

> 记法：CMake 是"中间层"——你写一份声明，它帮你生成 Makefile / VS 工程 / Ninja。

---

## 2. 最小可运行示例（先跑通）

- [ ] **目录结构**：
  ```
  hello/
  ├── CMakeLists.txt
  └── main.c
  ```
- [ ] **CMakeLists.txt**（最小版）：
  ```cmake
  cmake_minimum_required(VERSION 3.16)
  project(Hello C)

  add_executable(hello main.c)
  ```
  - `cmake_minimum_required`：声明最低 CMake 版本（**必须第一行**，很多报错源于版本太低）
  - `project(名字 语言)`：声明工程名与语言（`C` / `CXX` / 混合）
  - `add_executable(目标名 源文件...)`：生成可执行文件
- [ ] **编译流程**（两段式）：
  ```bash
  mkdir build && cd build
  cmake ..            # 1. 配置阶段：读取 CMakeLists.txt，生成构建文件
  cmake --build .     # 2. 构建阶段：调用底层编译器，产出可执行文件
  ```
  - 也常用一条命令：`cmake -S . -B build`（`-S` 源码目录，`-B` 构建目录），再 `cmake --build build`
- [ ] **配置阶段 vs 构建阶段**——理解这两段是排错的关键：
  - **配置（Configure）**：跑 `CMakeLists.txt`，做变量/依赖/生成器决策，产 `Makefile`/`CMakeCache.txt`
  - **构建（Build）**：调用真正的编译器（gcc/g++），把源文件编成目标文件，再链接成可执行/库
- [ ] **验证**：`./hello`（Linux/macOS）或 `hello.exe`（Windows）。

> 关键：**改了 `CMakeLists.txt` 要重新配置**（重新 `cmake -S . -B build`），只改源码则免配置直接 `cmake --build`。

---

## 3. 核心语法与常用命令

- [ ] **变量与变量引用**：
  ```cmake
  set(MY_SOURCES main.c util.c)   # 定义变量
  add_executable(app ${MY_SOURCES}) # 用 ${变量名} 引用
  set(CMAKE_C_STANDARD 11)        # C 标准
  ```
- [ ] **目标类型**：
  | 命令 | 作用 |
  |---|---|
  | `add_executable(name src...)` | 可执行文件 |
  | `add_library(name src...)` | 库：默认 `STATIC`，可 `SHARED`（动态库）、`INTERFACE`（纯头文件接口库） |
  | `add_library(name STATIC src...)` | 显式静态库 |
  | `add_library(name SHARED src...)` | 共享库（.so / .dll / .dylib） |
- [ ] **给库/可执行文件加属性（target 指向）**：
  ```cmake
  add_library(mylib STATIC impl.c)
  target_include_directories(mylib PUBLIC include)   # 头文件路径
  target_compile_features(mylib PUBLIC c_std_11)     # 要求 C11
  target_link_libraries(app PRIVATE mylib)           # 链接依赖
  ```
- [ ] **`target_` 系列（现代 CMake 的核心，务必掌握核心三个）**：
  | 命令 | 用途 |
  |---|---|
  | `target_include_directories` | 头文件搜索路径 |
  | `target_compile_features` / `target_compile_options` | 编译标准 / 编译选项 |
  | `target_link_libraries` | 链接库 |
- [ ] **`PUBLIC` / `PRIVATE` / `INTERFACE`（作用域传播，容易混淆，重点）**：
  - `PRIVATE`：只对本目标生效（依赖不传播给下游）
  - `INTERFACE`：只传给下游（本目标自己不用，如头文件目录才能编下游）
  - `PUBLIC`：本目标 + 下游都用
  - 例：`target_link_libraries(app PRIVATE libA)` 表示 `libA` 只给 `app` 用，`app` 的调用方不需要知道 `libA`。
- [ ] **子目录（模块化）**：
  ```cmake
  add_subdirectory(src)      # 进入 src/ 下的 CMakeLists.txt
  target_link_libraries(app PRIVATE sublib)  # 引用子目录里定义的库
  ```

> 记法：现代 CMake 的三大件是 `add_executable/add_library` + `target_*` 加属性。**作用于 scope（PRIVATE/PUBLIC/INTERFACE）比"全局 set 变量"更干净**，这正是"现代 CMake"与老写法的区别。

---

## 4. 结构良好的工程模板（可直接套用）

- [ ] 推荐目录结构：
  ```
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
- [ ] **`project` 的 `VERSION`**：`project(MyApp VERSION 1.0.0)` 定义工程版本，可用于 `install` 与 `configure_file` 生成版本头。
- [ ] **`CMAKE_C_STANDARD` vs `target_compile_features`**：
  - 全局设：`set(CMAKE_C_STANDARD 11)`（所有目标生效）
  - 目标级（推荐）：`target_compile_features(app PRIVATE c_std_11)`（只作用于该目标）
- [ ] **告警选项**：`-Wall -Wextra -Wpedantic`；MSVC 下是 `/W4`。用 `if(MSVC)` 分支处理不同编译器。

> 工程组织要点：**源文件进 `src/`、头文件进 `include/`、顶层管整体、子目录管模块**，目标用 `target_link_libraries` 串联。

---

## 5. 查找第三方库（`find_package`）

- [ ] **`find_package`（找已安装的库）**：
  ```cmake
  find_package(ZLIB REQUIRED)             # 找 zlib（qmake/pkg-config 安装的）
  target_link_libraries(app PRIVATE ZLIB::ZLIB)
  ```
  - `REQUIRED`：找不到就**报错**（可选）；`QUIET`：找不到也不打扰
  - 找到的库通常以 `名字::名字` 的"目标"形式使用（`ZLIB::ZLIB`、`OpenCV::opencv`）
- [ ] **两种查找方式（理解即可）**：
  - **Module 模式**：CMake 自带 `FindXXX.cmake`，只管常见库（`ZLIB`、`CURL` 等）
  - **Config 模式**：库自带 `XXXConfig.cmake`，更通用、更新（`OpenCV`、`Qt`、`gtest` 等多用这种）
- [ ] **`find_package` 找不到的替代**：手动指定查找路径：
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
- [ ] **`FetchContent`（把第三方源码直接下载进工程，现代常用）**：
  ```cmake
  include(FetchContent)
  FetchContent_Declare(googletest
      GIT_REPOSITORY https://github.com/google/googletest.git
      GIT_TAG v1.14.0)
  FetchContent_MakeAvailable(googletest)
  ```

> 记法：`find_package` 找"系统已装"的库；`FetchContent` 拉"源码/远程"的库；两者都通过**目标**链接。

---

## 6. 常用选项与工具链

- [ ] **`option`（开关，可让用户 `-D` 打开/关闭）**：
  ```cmake
  option(BUILD_TESTING "构建测试" ON)   # 默认开
  if(BUILD_TESTING)
      enable_testing()
  endif()
  ```
  - 命令行：`cmake -DBUILD_TESTING=ON ..`（传变量值）
- [ ] **`CMAKE_BUILD_TYPE`（编译配置）**：
  - `Debug` / `Release` / `RelWithDebInfo` / `MinSizeRel`
  - 命令行：`cmake -DCMAKE_BUILD_TYPE=Release ..`
  - 一般 `Release` 开 `-O2/-O3`，`Debug` 加符号 `-g` 且不优化（利于 gdb）
- [ ] **生成器（generator，底层用啥构建）**：
  - 默认：Unix 用 `Unix Makefiles`；Windows 用 `Visual Studio xx`（或 `Ninja` 更快）
  - 指定：`cmake -G "Ninja" ..`、`cmake -G "MinGW Makefiles" ..`
- [ ] **`CMAKE_C_COMPILER` / `CMAKE_CXX_COMPILER`（指定编译器）**：
  ```bash
  cmake -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++ ..
  ```
- [ ] **`include` / `install` 路径**：
  - `include_directories()`（全局旧式）vs `target_include_directories()`（推荐）
  - `install(TARGETS ... DESTINATION ...)`：安装规则（把库/头/可执行安到系统目录）

> 记法：`-D` 传变量、`-G` 选生成器、`-DCMAKE_BUILD_TYPE` 选优化级别，这三个命令行参数覆盖绝大多数需求。

---

## 7. 常见报错与排错（重点）

| 报错 | 原因 | 解决 |
|---|---|---|
| `CMake Error: Could not find ...` | `find_package` 找不到库 | 检查是否安装、设 `CMAKE_PREFIX_PATH`/`XXX_ROOT` |
| `add_library cannot create target ... because another target with the same name already exists` | 目标名重复 | 改目标名，或用不同 `add_library` 位置 |
| `The C compiler ... is not able to compile a simple test program` | 编译器没装/路径错 | 检查 `gcc`、设 `CMAKE_C_COMPILER` |
| `CMakeLists.txt: xx: error: ...` | 语法/版本问题 | 看行号；常见是 `cmake_minimum_required` 版本太高或命令拼错 |
| `undefined reference to ...` | 链接时找不到符号 | 没 `target_link_libraries` 到对应库，或库没被包含 |
| `No rule to make target ...` | 构建清单滞后 | 重新 `cmake -S . -B build`（`CMakeLists.txt` 变了要重配） |
| 找不到头文件 `fatal error: xxx.h: No such file` | 头文件路径没配 | `target_include_directories` 加路径 |
- [ ] **排错三步法**：
  1. **先看配置报错还是构建报错**（阶段不同，处理不同）
  2. **配置报错**：通常 `CMakeLists.txt` 语法/依赖/版本 → 重跑 `cmake -S . -B build` 看完整信息
  3. **构建报错**：编译器/链接器错误 → 看具体 `.c` 文件与链接库
- [ ] **清掉缓存重来（万能重定位）**：
  ```bash
  rm -rf build && mkdir build && cd build && cmake .. && cmake --build .
  ```
  （Windows 用 `Remove-Item -Recurse build`）。很多"灵异问题"清空 `build/` 后消失。

> 记法：**配置错 = 脚本/依赖问题，构建错 = 代码/链接问题**。分不清就先 `rm -rf build` 重来。

---

## 8. 与 C++ / 实际工具的衔接

- [ ] **写 C++ 项目**：`project(... LANGUAGES C CXX)`，用 `target_compile_features(app PRIVATE cxx_std_17)` 指定标准。
- [ ] **CMake + Makefile 关系**：CMake 生成 Makefile（或 Ninja/VS 工程），实际编译仍由这些底层构建工具跑。
- [ ] **VS Code / CLion 集成**：这些 IDE 直接读 `CMakeLists.txt` 自动配置智能提示与编译。
- [ ] **`compile_commands.json`（给 IDE/静态分析用）**：
  ```bash
  cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..
  ```
  产出后供 clangd / clang-tidy 等做精确跳转与检查。
- [ ] **`CMakePresets.json`（现代工程规范）**：把常用配置/构建命令存成 preset，一键复用（CMake 3.19+）。

> 记法：CMake 只是"构建编排"，真正编译交给底层工具；IDE 通过读 `CMakeLists.txt` 或 `compile_commands.json` 干活。

---

## 学习路径建议

| 阶段 | 内容 | 目标 | 状态 |
|---|---|---|---|
| 1 | 概念与最小示例 | 跑通一个可执行文件 | ☐ |
| 2 | 核心命令 | 会 `add_*` / `set` / `${}` | ☐ |
| 3 | target 作用域 | 会用 PRIVATE/PUBLIC/INTERFACE | ☐ |
| 4 | 工程模板 | 搭一个 src/include 规范工程 | ☐ |
| 5 | 找库 | 会用 `find_package`/`FetchContent` | ☐ |
| 6 | 选项与工具链 | `-D` / `-G` / 生成器 / 告警 | ☐ |
| 7 | 排错 | 区分配置错/构建错 | ☐ |

---

## 待填充内容（后续逐节展开）

1. 完整可运行的示例工程（顶层 + 子目录 + 源文件 + 测试）
2. `find_package` 与 `FetchContent` 的完整对比示例
3. 静态库 vs 动态库 vs 接口库的深入讲解
4. `install` / 打包（CPack）与 `configure_file` 生成配置头
5. 常见排错反例与调试技巧

> 告诉我"从第 X 节开始填充"或"填充某节"，我会按框架逐节写内容。
