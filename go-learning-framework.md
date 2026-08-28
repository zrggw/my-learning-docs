# Go 语言学习框架（C++ 背景版）

> 用途：Go 语言系统学习路线图。当前只含**框架与要点提纲**，每节内容待逐步填充。
> 使用方式：每学完一个小点，把 `[ ]` 改成 `[x]`；有疑问或要展开某节，随时让我补充内容。
> 配套文档：`cpp-to-go-cheatsheet.md`（刷题速查，解决"怎么写"）；本文档解决"学什么、按什么顺序学"。

---

## 阶段 1：基础语法

- [ ] 包（package）与 import，`main` 包与 `main` 函数
- [ ] 变量：`var` / `:=` 短声明、类型推断、**零值**（无需手动初始化）
- [ ] 常量 `const`（编译期求值）+ **iota**（常量计数器，用法见下方「阶段 1A」）
- [ ] 基本类型：int / float64 / string / bool / byte / rune
- [ ] **类型转换**：无隐式转换，必须显式 `int(x)`
- [ ] 流程控制：
  - `if`（条件不加括号；可带初始化语句 `if x := f(); x > 0`）
  - `for`（唯一循环：三表达式 / 条件 / range；无 while）
  - `switch`（自动 break；`fallthrough`；可替代 if-else 链）
- [ ] 函数：多返回值、命名返回值、可变参数 `...T`
- [ ] 函数是一等公民：函数值、闭包（lambda 的 Go 版）
- [ ] `defer`（延迟执行，资源释放；类似 RAII 但显式）

> 与 C++ 对照：无重载、无默认参数、无三元运算符；`++` 是语句不是表达式。

### 阶段 1A：常量与 iota

**iota：常量计数器**

- 是什么：Go 内置的常量计数器，在 `const` 块内按"声明行"自动递增，从 0 开始
- 基础用法（定义枚举）：
  ```go
  type Weekday int

  const (
      Sunday Weekday = iota // 0
      Monday                // 1
      Tuesday               // 2
      Wednesday             // 3
  )
  ```
- 跳过某个值用 `_` 占位：
  ```go
  const (
      A = iota // 0
      _        // 1 被跳过
      B        // 2
  )
  ```
- 表达式用法（经典：字节单位）：
  ```go
  const (
      _  = iota             // 0，忽略
      KB = 1 << (10 * iota) // 1 << 10 = 1024
      MB                    // 1 << 20
      GB                    // 1 << 30
  )
  ```
- 同一行多条声明：`a, b = iota, iota` 两个值相同（按行计数，不按表达式数）
- 注意事项：
  - iota 只在 `const` 块内有效；**每次出现 `const` 关键字都会重置为 0**
  - 空行与注释不占计数；`_` 占位跳过但计数照常递增
  - iota 是整数常量、**不是类型**：赋给自定义类型要显式标注，如 `Sunday Weekday = iota`
- 与 C++ 对照：类似 `enum`（同样自动递增），区别是 iota 不自动绑定类型，且可出现在任意常量表达式中（移位、运算），比 C++ 枚举更灵活

---

## 阶段 2：复合类型与容器（核心）

- [ ] 数组 `[N]T`（值语义）vs 切片 `[]T`（引用头语义）
- [ ] 切片：`make` / `append` / `copy` / 切片表达式 `s[a:b]`（左闭右开）
  - **append 必须接收返回值**、底层数组共享陷阱
- [ ] map：增删查、`v, ok := m[k]` 双值查找、遍历顺序随机
- [ ] string：不可变、`s[i]` 是 byte、`[]rune` 处理 Unicode、strings 包
- [ ] struct：定义、字段、字面量、嵌套、匿名结构体
- [ ] 指针：`&` / `*`，**无指针运算**（比 C++ 安全简单）
- [ ] 内建函数速查：len / cap / append / copy / delete / make / new

> 与 C++ 对照：vector→切片、unordered_map→map、set/stack/queue/堆→模拟（见 cheatsheet §2-4）。

---

## 阶段 3：方法与接口（Go 的"面向对象"）

- [ ] 方法：接收者 `func (s *T) M()`，**值接收者 vs 指针接收者**的区别
- [ ] 接口：**隐式实现**（无需声明 implements）、空接口 `any`
- [ ] 类型断言 `x.(T)` 与 type switch
- [ ] 组合优于继承：结构体嵌入（embedding）
- [ ] 泛型（Go 1.18+）：类型参数 `[T any]`、约束（constraints）、泛型函数/类型

> 与 C++ 对照：没有 class/继承/虚函数；接口类似"鸭子类型"；泛型比模板简单但有约束限制。

---

## 阶段 4：错误处理与测试

- [ ] error 接口：`errors.New` / `fmt.Errorf`（`%w` 包装错误）
- [ ] 错误处理风格：显式 if err != nil（不抛异常）
- [ ] panic / recover：什么情况用、什么情况**不要**用
- [ ] 单元测试：`_test.go` 文件、`go test`、表驱动测试（table-driven）（测试进阶/mock 见下方「阶段 4A」）
- [ ] **测试进阶**：断言库、Mock、httptest、覆盖率/模糊测试——内容见下方「阶段 4A」
- [ ] **测试替身家族**：Dummy / Stub / Fake / Spy / Mock 与相关方法——内容见下方「阶段 4B」
- [ ] 基准测试与 `go test -bench`（性能对比 C++ 习惯）

> 与 C++ 对照：没有 try/catch；错误用返回值显式传递。

### 阶段 4A：测试进阶（断言 / Mock / HTTP 测试）

**断言库（推荐 testify）**

- 标准库 `testing` 只有 `t.Errorf` / `t.Fatal`；断言库写起来更简洁、报错更友好
- `github.com/stretchr/testify`：`assert`（失败继续跑）与 `require`（失败立即终止）
  ```go
  func TestAdd(t *testing.T) {
      assert.Equal(t, 3, add(1, 2))      // 不相等会打印 want/got
      require.NoError(t, err)             // 出错立即终止，防止后续 nil 解引用
  }
  ```

**什么是 Mock**

- Mock = 用假实现替换真实依赖（数据库、外部 API、时钟、网络），目的：
  1. 测试**快且稳定**（不联网、不碰真实数据）
  2. 方便控制返回值与**错误路径**
  3. 验证"是否按预期被调用"（次数、参数）
- Go 里能 mock 的前提是**面向接口编程**：依赖写成接口，测试时注入假实现
- 与 C++ 对照：≈ 虚函数 + 依赖注入；思路与 GoogleTest Mock 一致，但 Go 靠接口（更轻量），不需要继承体系

**三种写 mock 的方式**

1. **手写假实现**（最简单，先掌握这个）：
   ```go
   type UserStore interface { Get(id int) (*User, error) }

   type fakeStore struct{ users map[int]*User }
   func (f *fakeStore) Get(id int) (*User, error) {
       if u, ok := f.users[id]; ok { return u, nil }
       return nil, errors.New("not found")
   }
   // 测试里用 fakeStore 替代真实实现注入
   ```
2. **代码生成**（项目大、接口多时）：
   - `go.uber.org/mock/mockgen`（原 gomock）或 `github.com/vektra/mockery/v2`
   - 用法：`ctrl := gomock.NewController(t)`；`m := NewMockUserStore(ctrl)`；`m.EXPECT().Get(1).Return(&User{ID: 1}, nil)`；用 `m.EXPECT().Get(2).Return(nil, errNotFound)` 覆盖错误路径
3. **专项 mock 库**：
   - 数据库：`github.com/DATA-DOG/go-sqlmock`（mock `database/sql` 连接，不用真库）
   - HTTP 客户端：`httptest.NewServer(...)` 起假服务，测真实请求逻辑

**httptest：测 HTTP handler**

```go
req := httptest.NewRequest("GET", "/user/1", nil)
w := httptest.NewRecorder()
myHandler(w, req)                    // handler 直接用，不用真端口
assert.Equal(t, 200, w.Code)
assert.Contains(t, w.Body.String(), "alice")
```

**常用测试技巧**

- 子测试：`t.Run("case1", func(t *testing.T){...})`（表驱动测试的标配）
- `t.Helper()`：标记辅助函数，报错定位到调用处而非辅助函数内部
- `t.Cleanup(func())`：注册收尾逻辑；`t.TempDir()`：自动建临时目录并清理
- `t.Setenv("KEY", "val")`：临时改环境变量（测试结束自动还原）
- 并行与跳过：`t.Parallel()`；平台相关用例 `t.Skip()`
- 覆盖率：`go test -cover`；生成报告 `go test -coverprofile=out.txt -covermode=atomic`
- 竞态：`go test -race`（并发代码必跑）
- **模糊测试 fuzz**（Go 1.18+）：`f.Fuzz(func(t *testing.T, ...){...})`，`go test -fuzz=FuzzX` 自动找崩溃/panic 输入
- 分层原则：**单测用 mock**（快、稳定），**集成测试用真实依赖**（可信）

### 阶段 4B：测试替身家族（Dummy / Stub / Fake / Spy / Mock）与相关方法

> 为什么要补：mock 只是五种**测试替身（Test Double）**之一。日常说的"mock"常常其实是 stub 或 fake，分清五兄弟，测试才写得准、读得懂。

**五兄弟对照表**

| 替身 | 作用 | 典型场景 | Go 写法示意 |
|---|---|---|---|
| Dummy（哑元） | 只占参数位，永远不会被调用 | 必须传参但用不到 | 空结构体，方法直接 `panic("not used")` |
| Stub（桩） | 返回**固定值**，控制被测代码走某分支 | 接口需要固定结果/错误 | 空结构体 + `Get()` 返回写死的值 |
| Fake（假件） | 有**真实行为**的轻量实现 | 内存 map 当数据库 | `fakeStore`（map 版，逻辑真实） |
| Spy（间谍） | **记录**调用信息，事后断言 | 验证"调没调、调几次、传了啥" | 手写字段 `called int; lastMsg string` |
| Mock（模拟） | 预设**期望** + 自动验证调用 | 交互复杂、需严格验证 | gomock：`EXPECT().Get(1).Return(...).Times(1)` |

**选择建议**
- 只需要"返回值/错误" → **Stub**（最常用）
- 需要真实逻辑的简化版 → **Fake**（如内存数据库）
- 只需验证是否被调用 → **Spy**（手写几个字段即可）
- 需要严格期望 + 次数/顺序验证 → **Mock**（上 mockgen/gomock）
- Dummy 顺手写掉即可，别过度设计

**手写示例（对比三种）**

```go
// Stub：固定返回
type stubClock struct{}
func (stubClock) Now() time.Time { return time.Date(2026, 1, 1, 0, 0, 0, 0, time.UTC) }

// Spy：记录调用
type spySender struct{ called int; lastMsg string }
func (s *spySender) Send(msg string) error { s.called++; s.lastMsg = msg; return nil }

// Mock：预置期望（由 mockgen 生成，这里示意）
// m := NewMockSender(ctrl)
// m.EXPECT().Send("hello").Times(1).Return(nil)
```

**与 mock 相异（但互补）的测试方法**

| 方法 | 思路 | 何时用 |
|---|---|---|
| 集成测试 | 用**真实依赖**（真库、真 HTTP） | 验证组件间协作，单测覆盖不到的部分 |
| golden file 测试 | 输出与"黄金文件"**逐字比对** | 序列化、代码生成、CLI 输出 |
| property-based 测试 | 随机大量输入断言**性质/不变量** | 排序、解析器、数学函数（`testing/quick`、gopter） |
| contract testing | 服务间按**契约**联测（如 Pact） | 微服务 API 兼容性 |
| fuzz（见 4A） | 随机输入找崩溃 | 输入解析类代码 |

**一句话总结**：stub 控制输入、fake 提供真实简化行为、spy 记录调用、mock 严格验证交互——这四者解决"**如何替换依赖**"；golden / property / contract 解决"替换掉依赖之后**如何断言**"。

---

## 阶段 5：并发编程（Go 招牌，刷题少用但必学）

- [ ] goroutine：`go f()` 启动轻量线程
- [ ] channel：无缓冲/有缓冲、发送接收、`close`、`range` 遍历、`select` 多路复用
- [ ] sync：WaitGroup / Mutex / RWMutex / Once / 原子操作（Once 用法见下方「阶段 5A」）
- [ ] **context**：取消 / 超时 / 传值——内容见下方「阶段 5A：context 与 sync.Once」
- [ ] 经典模式：worker pool、生产者-消费者、超时与取消
- [ ] 竞态检测：`go test -race`，养成习惯

> 与 C++ 对照：替代 std::thread + mutex 的组合，语法简洁得多；刷题阶段可先跳过。

### 阶段 5A：context 与 sync.Once 使用详解

**context：跨 goroutine 的取消 / 超时 / 传值**

- 作用：把"取消信号、超时、截止时间、请求级小数据"沿着调用链传递，是所有并发代码的**标准第一参数**
- 四种来源：
  - `context.Background()`：根 context（main 函数 / 顶层用）
  - `context.TODO()`：暂时不知道用什么时占位（别传 nil）
  - `context.WithCancel(parent)` → `(ctx, cancel)`：手动取消
  - `context.WithTimeout(parent, d)` / `context.WithDeadline(parent, t)`：到点自动取消
- **铁律**：`cancel` 必须被调用，否则资源泄漏；通常 `defer cancel()` 立即挂上

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel() // 必须调用！

go func() {
    select {
    case <-time.After(3 * time.Second):
        fmt.Println("任务完成")
    case <-ctx.Done(): // 取消/超时时关闭
        fmt.Println("任务被取消:", ctx.Err()) // context.DeadlineExceeded 或 Canceled
        return
    }
}()
```

- 监听取消：`<-ctx.Done()`（channel 被关闭）；`ctx.Err()` 返回 `context.Canceled` / `context.DeadlineExceeded`
- 传值（少用，只放请求级元数据，如 requestId）：`context.WithValue(ctx, "reqId", "abc-123")`，取用 `ctx.Value("reqId")`
- 与标准库结合（高频）：`http.NewRequestWithContext(ctx, ...)`、`db.QueryContext(ctx, ...)`、`exec.CommandContext(ctx, ...)`——超时自动中断
- 五条规范：① 作为第一个参数传；② 不要存进 struct 字段；③ 不传 nil（用 TODO()）；④ cancel 必须执行；⑤ Value 只放不可变小数据

**sync.Once：保证代码只执行一次**

- 作用：**并发安全**地让某段初始化代码只执行一次，典型场景：懒加载单例、进程内只初始化一次的全局资源（配置、连接池）
- API：`o.Do(func())`——多个 goroutine 同时调用也只会执行一次，其余调用阻塞等它完成
- 若 `Do` 里的函数 panic：Once 视为**未完成**，下次调用会重新执行

```go
var (
    once   sync.Once
    config *Config
)

func GetConfig() *Config {
    once.Do(func() {
        config = loadConfig() // 全局只执行一次（并发安全）
    })
    return config
}
```

- 两个坑：
  - Once **不可复制**（拷贝后行为未定义），务必用指针/包级变量
  - `Do` 内部不要再调用同一个 Once 的 `Do`（死锁）
- 与 C++ 对照：等价于 `std::call_once` + `std::once_flag`
- 职责区分：`context` 管**请求级**生命周期（取消/超时）；`sync.Once` 管**进程级**一次性初始化，两者不冲突

---

## 阶段 6：标准库速览（按需深入）

- [ ] fmt（Print/Scan/Sprintf）与 strconv（字符串↔数字）
- [ ] strings / bytes（字符串与字节处理）
- [ ] sort / slices（Go 1.21+）/ container/heap / container/list
- [ ] bufio / os / io（文件与快速 IO）
- [ ] math / math/big（大数，替代手写高精度）
- [ ] time（计时、格式化）
- [ ] encoding/json（序列化，Web/工具类常用）
- [ ] net/http + 简单 Web 服务（入门）

> 刷题阶段重点：fmt、strconv、strings、sort、slices、container/heap、bufio、math。

---

## 阶段 7：进阶主题（可选）

- [ ] 内存模型：值语义 vs 引用语义、逃逸分析、GC（了解即可）
- [ ] 反射 reflect（少用，了解原理）
- [ ] 泛型进阶：类型集、自定义约束
- [ ] 常用模式：单例、选项模式、依赖注入的 Go 写法
- [ ] 性能工具：pprof 性能剖析、benchmark 对比
- [ ] 内建函数完整清单过一遍
- [ ] **cgo**（Go 与 C/C++ 互操作）——内容见下方「阶段 7A：cgo 专题」
- [ ] **GODEBUG**（运行时调试开关）——内容见下方「阶段 7B：GODEBUG」

### 阶段 7A：cgo 专题（Go 与 C/C++ 互操作）

**cgo 是什么**
- cgo 是 Go 官方的 C 互操作机制：Go 代码直接调用 C 函数、使用 C 类型
- 用法：在 Go 源码中写 `import "C"`，其**正上方紧邻**（不得有空行）的注释块里写 C 代码或 `#include`
- 注意：cgo 只认 **C ABI**，不直接支持 C++；调用 C++ 必须用 `extern "C"` 包一层 C 桥接
- 对 C++ 背景的你：语法不陌生，重点在于理解"桥接层"和内存/指针规则

**最小示例**

```go
package main

/*
#include <stdlib.h>

int add(int a, int b) { return a + b; }
*/
import "C"
import (
	"fmt"
	"unsafe"
)

func main() {
	fmt.Println(C.add(3, 4)) // 7

	cs := C.CString("hello")            // Go 字符串 -> C 字符串（C 堆分配！）
	defer C.free(unsafe.Pointer(cs))    // 必须手动释放
	fmt.Println(C.GoString(cs))         // C 字符串 -> Go 字符串（拷贝）
}
```

**C++ 桥接套路**

```cpp
// bridge.h —— 用 extern "C" 把 C++ 函数暴露成 C 符号
#ifdef __cplusplus
extern "C" {
#endif
int cpp_max_plus(int a, int b);
#ifdef __cplusplus
}
#endif
```

```cpp
// bridge.cpp —— C++ 实现随便用 STL / 类
#include "bridge.h"
#include <algorithm>
int cpp_max_plus(int a, int b) { return std::max(a, b) + b; }
```

```go
// main.go
/*
#cgo LDFLAGS: -lstdc++   // 链接 C++ 运行时
#include "bridge.h"
*/
import "C"

func main() { println(C.cpp_max_plus(1, 2)) }
```

要点：Go 侧永远只面对 C 函数声明；C++ 的一切（类、模板、STL）都封装在桥接层里。

**常用类型与字符串转换**

| 场景 | 写法 |
|---|---|
| C 基本类型 | `C.int` / `C.double` / `C.size_t` / `C.long` … |
| C 指针 / 数组 | `*C.char`、`unsafe.Pointer` |
| Go → C 字符串 | `cs := C.CString(s)`（用完 `C.free`） |
| C → Go 字符串 | `C.GoString(cs)` / `C.GoStringN(cs, n)` |
| C → Go 字节 | `C.GoBytes(ptr, n)` |
| C 结构体 | 注释块里定义，Go 侧用 `C.struct_xxx` |

**#cgo 指令与构建**

- `#cgo CFLAGS: -I...`（头文件路径）、`#cgo LDFLAGS: -lxxx`（链接库）、`#cgo pkg-config: xxx`（pkg-config）
- `go build` 会自动调用 C 编译器（gcc/clang），本机需安装
- `CGO_ENABLED=0`：禁用 cgo（要求代码无 C 依赖），产物为纯静态、便于交叉编译
- 带 cgo 的交叉编译需要目标平台的 C 工具链（很麻烦；能用 `CGO_ENABLED=0` 就尽量用）

**指针与内存规则（最容易踩坑）**

- `C.CString` 等 C 侧分配的内存**必须**手动 `C.free`，Go 的 GC 管不到
- Go 指针可以传给 C 函数，但 **C 代码不能在调用返回后继续持有**该指针（运行时检查，违规直接 panic）
- 含 Go 指针的 Go 内存（如 `[]*T`）**不能**直接传给 C；需要时先拷贝进 `C.malloc` 的内存
- C 代码里的段错误/崩溃无法被 Go 的 `recover` 捕获，**整个进程直接挂掉**

**性能与使用时机**

- 每次 cgo 调用有固定开销（goroutine 绑定线程、栈切换），热路径要**批量/合并**跨界调用，不要逐元素调
- 适合用 cgo：复用成熟 C/C++ 库（数据库驱动、图像/加密库）、需要 C ABI 兼容
- 不适合：能用纯 Go 实现就别用（维护、构建、交叉编译成本高）；简单系统调用用 `syscall` / `os` 包即可
- 进阶：`//export` 让 C 回调 Go 函数、SWIG 自动生成绑定（按需再学）

### 阶段 7B：GODEBUG（运行时调试开关）

**是什么**
- GODEBUG 是 Go 运行时的**调试/调优环境变量**，格式为逗号分隔的 `key=value` 列表
- 不改代码就能观察 GC、调度器、内存分配等运行时内部行为；很多开关还是"兼容性开关"（控制行为向后兼容）
- 查看完整清单：`go doc runtime`（GODEBUG 小节）；各版本新增开关见该版本 release notes

**设置方式**

```bash
# Linux/macOS
GODEBUG=gctrace=1 go run main.go
export GODEBUG=gctrace=1,schedtrace=1000

# Windows（PowerShell）
$env:GODEBUG = "gctrace=1,schedtrace=1000"
go run main.go
```

**常用开关表**

| 开关 | 作用 |
|---|---|
| `gctrace=1` | 每次 GC 打印一行统计（堆大小变化、STW 暂停时长等） |
| `schedtrace=1000` | 每 1000ms 打印调度器概览（M/P/G 数量、运行队列） |
| `scheddetail=1` | 配合 schedtrace 输出每个 M/P/G 的详细信息 |
| `memprofilerate=1` | 内存分配采样改为全量（默认约每 512KB 采样一次；影响 pprof 精度） |
| `allocfreetrace=1` | 每次内存分配/释放都打印（输出量巨大，慎用） |
| `inittrace=1` | 打印每个包的 init 函数耗时（排查启动慢） |
| `cgocheck=0/1/2` | cgo 指针检查级别（默认 2；调 0 可提速但牺牲安全检查） |
| `netdns=go/cgo` | 选择 DNS 解析器实现（go 纯 Go / cgo 走系统） |
| `panicnil=1` | 允许 `panic(nil)`（行为兼容开关） |
| `http2client=0` 等 | 各类兼容性行为开关，具体按 Go 版本查阅 |

**看输出（两个最常用）**

- `gctrace=1` 示例行：
  ```
  gc 14 @2.123s 4%: 0.5+1.0+0.4 ms clock, ... 8->8->4 MB, 5 MB goal, 8 P
  ```
  含义：第 14 次 GC、发生在 2.123s、STW 停顿各阶段耗时、堆 `GC前->GC中->GC后` 大小、目标堆大小、P 数量。**重点看 STW 时长和 heap 是否持续增长**。
- `schedtrace=1000` 示例行：
  ```
  SCHED 2019ms: gomaxprocs=8 idleprocs=5 threads=5 runqueue=0 [0 0 0 0 0 0 0 0]
  ```
  含义：P 上限、空闲 P 数、线程数、全局运行队列长度、各 P 本地队列长度。**队列长期非空说明任务饱和/调度压力大**。

**使用场景与注意事项**

- 工作流：先用 GODEBUG 观察现象（GC 频繁？STW 长？goroutine 堆积？），再用 `go tool pprof` 深挖热点
- 与 `go test` 结合：测试前同样设置 GODEBUG 即生效，如 `$env:GODEBUG="gctrace=1"; go test -run TestX`
- 调试 cgo 相关问题时可临时 `cgocheck=0`，但**生产环境不要关**
- 注意：GODEBUG 输出会显著拖慢程序，适合排障期使用，生产按需开

---

## 阶段 8：实战巩固

- [ ] 用 Go 重刷 LeetCode 热题 50 道（对照 `cpp-to-go-cheatsheet.md`）
- [ ] 小项目 1：命令行工具（如 JSON 格式化器、日志统计）
- [ ] 小项目 2：HTTP 服务（如短链接服务、待办 API）
- [ ] 小项目 3：并发任务（如并发下载器 / 爬虫）
- [ ] 阅读：《The Go Programming Language》、官方 Tour（tour.golang.org）、Effective Go

---

## 学习进度总表

| 阶段 | 内容 | 预计投入 | 状态 |
|---|---|---|---|
| 1 | 基础语法 | 1-2 天 | ☐ |
| 2 | 复合类型与容器 | 2-3 天 | ☐ |
| 3 | 方法与接口 | 2 天 | ☐ |
| 4 | 错误处理与测试 | 1 天 | ☐ |
| 5 | 并发编程 | 3-4 天 | ☐ |
| 6 | 标准库速览 | 2 天 | ☐ |
| 7 | 进阶主题 | 按需 | ☐ |
| 8 | 实战巩固 | 持续 | ☐ |

---

## 待填充内容（后续逐节展开）

1. 每节的详细讲解 + 代码示例（按当前框架顺序推进）
2. 每节配套练习题目
3. C++ 与 Go 逐主题对照笔记
4. 常见坑与面试高频考点标注

> 告诉我"从第 X 节开始填充"或"填充某节"，我会按框架逐节写内容。
