# 4 MiB 环形缓冲区字符设备驱动测试报告（WSL2 实测）

> 用途：给"环形缓冲字符设备驱动"这份实验一个可交付的结论——三个用例各测什么、实测数字是多少、数字背后的原因、以及从零复现与排错的完整路径。面向**已写出驱动、需要证明它对且量化它快不快**的读者。
> 使用方式：每学完一个小点，把 `[ ]` 改成 `[x]`；需要展开某节可随时让我补充。
> 定位：回答"驱动写完了怎么验证"这一层，从"能编译、能 insmod"到"有实测数据可以写进报告"。
> 背景贴合：异步读写在内核里怎么走（`read_iter`/`write_iter` 契约、`kiocb` 生命周期、`fasync`/`io_uring`）见 `char-driver-async-io.md`；那篇标注"内核模块代码未在本机实测"，本文是同一个驱动的真机实测结果，两篇互补。
> 前提：WSL2，运行内核 **6.18.40.1-microsoft-standard-WSL2**，自建同版本内核源码树（WSL 不提供匹配的 `linux-headers`）。文中命令均在 WSL 内执行，硬件为 18 逻辑核 / 15.4 GiB 内存。

---

## 核心速览（先看这一页，5 分钟覆盖 80% 内容）

**一句话结论**：三个用例全部通过——数据通路（含 4 MiB 回绕）、吞吐基准、驱动内真异步都符合设计预期；**异步吞吐与同步同量级**（内存环形缓冲没有真实 I/O 延迟可隐藏），且**并发深度增大反而变慢**。

### ① 关键数字（每组 64 MiB，本机两次重复）

| 模式 | 峰值吞吐 | 峰值出现在 | 最低点 |
|---|---|---|---|
| 同步流水线（双进程） | 10.6 ~ 13.1 GB/s | 256 KiB | 4 KiB 仅 3.2 GB/s |
| 单进程乒乓 | 18.0 ~ 19.3 GB/s | 256 KiB | 4 KiB 约 4.9 GB/s |
| 异步读（原生 AIO） | 11.4 ~ 12.3 GB/s | 64 KiB，深度 1~8 | 1 MiB × 深度 32 掉到 1.9 GB/s |
| 异步写（原生 AIO） | 10.8 ~ 11.1 GB/s | 64 KiB ~ 1 MiB，深度 1 | 4 KiB 仅 2.6 GB/s |

### ② 复现命令（三条）

```bash
sudo insmod ~/ringbuf-driver/ringbuf.ko && sudo chmod 666 /dev/ringbuf
cd ~/ringbuf-driver && make test
bash scripts/run-tests.sh 64      # 采集环境 + 顺序跑三个用例，落盘到 test-results/
```

### ③ 三个易错点（本次全部实测踩到，详见 §5）

| 现象（`insmod` 统一报 `Invalid module format`） | 根因 | 处理 |
|---|---|---|
| dmesg 显示 `version magic '...' should be '...'` | `scripts/setlocalversion` 只认名为 `v<KERNELVERSION>` 的附注 tag，仓库 tag 命名不同就追加 `+` | 补附注 tag，重新生成版本头文件（§5.1） |
| dmesg 显示 `.gnu.linkonce.this_module section size must match ...` | 关掉 `DEBUG_INFO_BTF` 后 `struct module` 少了受 `CONFIG_DEBUG_INFO_BTF_MODULES` 保护的字段 | 该 `#ifdef` 改为无条件编译（§5.2） |
| dmesg 显示 `disagrees about version of symbol module_layout` | `struct module` 改了但符号表未重算，`module_layout` 的 CRC 仍是旧值 | 重算符号表 CRC（§5.3） |

### ④ 阅读路线

只求跑通 → §6；要看数据 → §3；要理解数字为何如此 → §4；装不上 → §5。

---

## 1. 被测对象与测试环境

> 覆盖：这份报告测的是哪个驱动、它提供哪些接口、在什么环境下测。不讲异步机制原理（见 `char-driver-async-io.md`）。

- [ ] **被测驱动是一份 4 MiB 环形缓冲区字符设备**：`vzalloc` 分配缓冲区，`alloc_chrdev_region` + `cdev_add` 注册，`device_create` 生成 `/dev/ringbuf`。

驱动注册的文件操作与语义：

| 入口 | 语义 |
|---|---|
| `.read` / `.write` | 同步读写，非阻塞模式下无数据/无空间返回 `-EAGAIN` |
| `.read_iter` / `.write_iter` | 内核 AIO 与 `io_uring` 的入口；缓冲就绪则在锁内直接拷贝 |
| `.read_iter` / `.write_iter`（请求不可立即满足时） | 把请求挂到 `async_readers` / `async_writers` 链表，返回 `-EIOCBQUEUED`，稍后由**反方向的操作**调用 `ki_complete()` 完成 |
| `.poll` | 就绪通知（`EPOLLIN` / `EPOLLOUT`） |
| `.fasync` | `SIGIO` 通知，`kill_fasync` 在状态变化处调用 |
| `.open` | 设置 `FMODE_NOWAIT`，声明支持非阻塞完成 |

- [ ] **读写都是"尽力而为"（best-effort）语义**：返回值是**实际搬运的字节数**，可以小于 `count`；调用方必须循环直到搬完。异步请求同理，完成事件里的 `res` 可能小于申请长度。
- [ ] **环形状态只有三个量**：`read_pos`、`write_pos`、`data_len`；空 = `data_len == 0`，满 = `data_len == RINGBUF_BUFFER_SIZE`。位置在拷贝时回绕，不靠预留槽位区分空满。
- [ ] **并发保护用 `mutex` + 两个等待队列**（`read_wait`、`write_wait`），不用自旋锁——因为拷贝走 `copy_to_iter`/`copy_from_iter`，可能睡眠。

实测环境（由 `scripts/run-tests.sh` 自动采集）：

| 项 | 值 |
|---|---|
| 运行内核 | `6.18.40.1-microsoft-standard-WSL2`，`#1 SMP PREEMPT_DYNAMIC`，构建于 2026-07-31（gcc 13.2.0） |
| CPU | 18 逻辑核，Intel Core Ultra 5 125H |
| 内存 | 15.4 GiB |
| 用户态编译器 | gcc 11.4.0（Ubuntu 22.04） |
| 关键内核配置 | `CONFIG_MODVERSIONS=y`、`CONFIG_MODULE_UNLOAD=y`、`CONFIG_AIO=y`、`CONFIG_IO_URING=y` |
| 运行内核的 `sizeof(struct module)` | **1280** 字节（由 `/sys/kernel/btf/vmlinux` 的 BTF 记录读出） |
| 设备节点 | `crw-rw-rw- root root 239,0 /dev/ringbuf` |

> 记法：模块能装载成功本身就是环境匹配的证据——内核会先校验 vermagic（版本串 + SMP/PREEMPT/`mod_unload`/`modversions` 标志），再校验 `struct module` 尺寸与 `module_layout` 的 CRC。

## 2. 测试用例设计

> 覆盖：三个测试程序各自验证什么、覆盖哪些 I/O 模型、哪些没覆盖。

- [ ] **`tests/ringbuf_smoke_test.c` 测数据通路与回绕**（单进程，`O_NONBLOCK`）：空缓冲区读必须 `-EAGAIN`；写 `SIZE-1000` 把 `write_pos` 推到接近末端；读走 1000；再写 2000 使其**跨越 4 MiB 边界**；循环读回全部 4 MiB，按 `gen(i) = (i*31+7) & 0xff` 校验整条字节流。
- [ ] **`tests/ringbuf_aio_test.c` 测驱动内真异步**（单进程，原生 AIO）：缓冲区为空时 `io_submit` 一个 16 字节异步读，必须**立即返回**；带零超时的 `io_getevents` 必须返回 **0 个事件**（证明请求被挂起而非同步完成）；随后同步 `write` 16 字节反向完成它，`io_getevents` 拿到 `res=16`、`data=0x1234`。
- [ ] **`tests/ringbuf_bench.c` 测吞吐量**，共四组：同步流水线（`fork` 出生产者/消费者，双方阻塞同步）、单进程乒乓（写完一块立刻读回）、异步读、异步写（父进程 AIO 保持 `depth` 个在途请求，子进程做同步对端）。
- [ ] **覆盖矩阵**（本次实测的能力边界）：

| 维度 | smoke | aio | bench 同步两组 | bench 异步两组 |
|---|---|---|---|---|
| 同步 `read`/`write` | 是 | 是（写那一步） | 是 | 子进程是 |
| 原生异步 I/O | 否 | 是 | 否 | 是 |
| 阻塞语义 | 否（用 `O_NONBLOCK`） | 否 | 是 | 子进程是 |
| 非阻塞 `-EAGAIN` | 是 | 否 | 否 | 否 |
| 多进程（`fork`） | 否 | 否 | 是 | 是 |
| **多线程（pthread）** | 否 | 否 | 否 | 否 |
| `poll`/`epoll`、`fasync`/`SIGIO`、`io_uring` | 否 | 否 | 否 | 否 |

- [ ] **测试顺序不能调换**：`ringbuf_aio_test` 要求缓冲区初始为空，而 `smoke_test` 与 `bench` 结束时都会把缓冲区清空，因此固定为 smoke → bench → aio。
- [ ] **关于"多进程而非多线程"**：`fork` 之后父子**共享同一个 `struct file`**（子进程继承的是已打开的 fd，不是各自 `open`）。好处是父进程挂起的异步请求能被子进程的同步操作完成；副作用是 `release()` 里的清理路径（摘 `fasync`、取消挂起请求）要等父子都关闭 fd 才会执行。

> 记法：断言"真异步"的最小证据组合 = `io_submit` 立即返回 + 零超时查询到 0 个完成事件 + 反方向操作后拿到完成事件。

## 3. 实测结果

> 覆盖：三个用例的原始输出与四张吞吐表（两次重复），不解释原因（见 §4）。

- [ ] **用例 ① 通过**（两次运行均为 `退出码=0`，墙钟 0.011 s / 0.021 s）：

```text
[1] /dev/ringbuf 打开成功
[2] 空缓冲区 read() = -EAGAIN（非阻塞语义正确）
[3] 写入 4193304 字节，write_pos 已接近末端
[4] 读回 1000 字节，内容校验通过
[5] 再写入 2000 字节 —— 已跨越 4 MiB 边界回绕
[6] 读回 4194304 字节，跨回绕的完整序列校验通过
PASS: 缓冲区读写（含回绕）工作正常
```

- [ ] **用例 ② 通过**（`退出码=0`，墙钟 1.351 s / 1.592 s），四张表如下；单位 MB/s，`×N` 表示并发深度。

| 块大小 | 同步流水线 第1次 / 第2次 | 单进程乒乓 第1次 / 第2次 |
|---|---|---|
| 4 KiB | 3257.1 / 3202.5 | 4490.0 / 5251.9 |
| 64 KiB | 7914.9 / 11483.4 | 16149.0 / 16345.8 |
| 256 KiB | 10603.8 / 13075.3 | 19272.1 / 18012.4 |
| 1 MiB | 7594.8 / 10998.9 | 14361.2 / 15039.3 |
| 4 MiB | 4971.3 / 5829.9 | 11139.7 / 11063.5 |

| 块大小 × 深度 | 异步读 第1次 / 第2次 | 异步写 第1次 / 第2次 |
|---|---|---|
| 4 KiB × 1 | 2827.7 / 3259.9 | 2552.3 / 2777.8 |
| 4 KiB × 8 | 3395.1 / 3325.7 | 2475.6 / 2940.0 |
| 4 KiB × 32 | 3168.1 / 3317.9 | 2633.8 / 2919.8 |
| 64 KiB × 1 | 11711.7 / 11355.0 | 10074.4 / 10823.2 |
| 64 KiB × 8 | 12312.6 / 11950.6 | 9579.3 / 9627.3 |
| 64 KiB × 32 | 8408.0 / 7676.8 | 8094.7 / 9759.6 |
| 1 MiB × 1 | 8964.4 / 9745.0 | 10421.4 / 11136.8 |
| 1 MiB × 8 | 5215.6 / 4466.2 | 7030.2 / 7556.2 |
| 1 MiB × 32 | 1727.8 / 1898.4 | 4302.6 / 5970.6 |

- [ ] **用例 ③ 通过**（`退出码=0`，墙钟 0.063 s / 0.063 s，两次完全一致）：

```text
[1] 异步读已提交（未阻塞）
[2] 提交后立即查询完成事件数 = 0（期望 0，请求被挂起）
[3] 已同步写入 15 字节，写动作应反向完成挂起的读
[4] 异步读完成：res=15, data=0x1234
    读到的内容 = "hello async io!"
PASS: 驱动内真异步工作正常
```

- [ ] **内核侧无异常**：两次运行后 `dmesg` 只有加载期的两行记录，测试期间没有新增告警。

```text
[ 2566.783327] ringbuf: loading out-of-tree module taints kernel.
[ 2566.784930] ringbuf: registered /dev/ringbuf (4194304 bytes)
```

> 记法：判断"数据对不对"看用例 ①③ 的逐字节校验，判断"快不快"看用例 ② 的四张表，二者不可互相替代。

## 4. 结果解读

> 覆盖：四张表的形状说明什么、为什么异步不比同步快、为什么深度增大反而变慢，以及本次测量的局限。原因分析中标注"推断"者为待验证假设。

- [ ] **同步流水线：4 KiB 到 256 KiB 上升，4 MiB 回落。** 4 KiB 只有 3.2 GB/s，瓶颈是系统调用与拷贝次数（每 MiB 需要 256 次读写）；块增大到 256 KiB 后接近内存带宽上限；到 4 MiB 反而降到 5.8 GB/s，因为单次拷贝 4 MiB 已超出各级 cache 的工作集，且两个进程几乎无法重叠搬运。
- [ ] **单进程乒乓是全场最高（18 ~ 19 GB/s），属于预期结果。** 没有第二个进程的调度与同步开销，写完立刻读回时数据仍在 L2/L3 里，所以它测的更像"缓存内往返上限"，不代表真实并发场景的吞吐。
- [ ] **异步吞吐与同步同量级，符合设计预期。** 缓冲区在内存里，`ki_complete` 只是在另一个上下文里做一次 `memcpy`，没有真实 I/O 延迟可以隐藏，异步只额外付出 `io_submit` + `io_getevents` 两次系统调用的成本。因此"异步没比同步快"不是缺陷，而是这个场景的必然结果。
- [ ] **深度增大反而变慢，是本次最值得记录的现象**：1 MiB 块下异步读从深度 1 的 9.7 GB/s 掉到深度 32 的 1.9 GB/s，且两次重复一致（说明不是偶然抖动）。可能的原因（**推断，尚未单独验证**）：
  - 环形缓冲只有 4 MiB，而 1 MiB × 32 的在途请求量达 32 MiB，绝大多数请求只能挂在链表上；完成时又在**持有 `mutex`** 的状态下逐个拷贝，锁的持有时间被拉长，与对方进程的读写直接争用；
  - 每个在途请求必须独占用户缓冲区（`depth × block`），深度 32 时工作集 32 MiB 超出 cache，拷贝退化为内存带宽受限；
  - 基准程序的补齐逻辑在"部分完成"时会按剩余量重新提交，深度大时碎片请求更多，进一步放大每次调用的固定开销。
  - 实践结论：**并发深度要与缓冲容量和块大小匹配**，本机 4 MiB 缓冲下 64 KiB 块是异步路径的甜点；盲目加大深度会同时恶化吞吐与延迟。
- [ ] **本次测量的局限（必须与数字一起引用）**：每组仅 64 MiB，单行耗时落在 3 ~ 37 ms，导致重复之间的抖动可观——例如同步流水线 64 KiB 的两次结果相差 45%（7914.9 与 11483.4 MB/s）。因此本报告的数字**只用于判断趋势与量级，不宜作为精确指标**。要得到稳定数据，请把参数放大并重复取中位数：

```bash
bash scripts/run-tests.sh 1024      # 每组 1024 MiB，单行耗时进入百毫秒量级
```

> 记法：内存环缓冲的吞吐看"块大小"（拷贝效率）而非"并发深度"（那是在隐藏真实 I/O 延迟时才起作用）。

## 5. 构建与装载的三个易错点（本次实测踩到）

> 覆盖：`insmod: Invalid module format` 的三层根因与修法，以及内核升级后必须重建这件事。这是本次实验最花时间的部分，值得单独记住。

- [ ] **`Invalid module format` 是个"通用外壳"**：vermagic 不匹配、`struct module` 尺寸不匹配、符号 CRC 不匹配、ELF 重定位不支持，对外都是这一句话。**唯一的定位手段是看 `dmesg`**，它会打出具体是哪一道检查失败。

```bash
sudo insmod ringbuf.ko
sudo dmesg | tail -3        # 这一行决定后面怎么修
```

- [ ] **§5.1 vermagic 多一个 `+`**：`scripts/setlocalversion` 只把名为 `v<KERNELVERSION>` 的**附注 tag** 视为"正在 tag 上"，而 WSL 源码仓库的 tag 叫 `linux-msft-wsl-<ver>`，名字对不上就追加 `+`，vermagic 变成 `6.18.33.2-microsoft-standard-WSL2+`。修法是补一个同名附注 tag 并重新生成版本头文件。

- [ ] **§5.2 `struct module` 尺寸不匹配**：`struct module` 里有若干字段受 `CONFIG_DEBUG_INFO_BTF_MODULES` 保护（`btf_data_size`、`btf_base_data_size`、`btf_data`）。构建时为节省时间关掉了 `DEBUG_INFO_BTF`，模块里的 `sizeof(struct module)` 就比运行内核小，`vermagic` 检查照样通过，但装载时报：

```text
module ringbuf: .gnu.linkonce.this_module section size must match the kernel's built struct module size at run time
```

修法与树莓派/Debian 内核长期携带的 `950-0449-module-Avoid-ABI-changes-when-debug-info-is-disabled.patch` 完全一致——把 `include/linux/module.h` 里那处 `#ifdef` 改成无条件编译，使结构体在关掉 debug info 时保持同一 ABI：

```c
-#ifdef CONFIG_DEBUG_INFO_BTF_MODULES
+#if 1
 	unsigned int btf_data_size;
 	unsigned int btf_base_data_size;
 	void *btf_data;
```

修完可自检：`.ko` 里 `.gnu.linkonce.this_module` 段的 `sh_size` 应与运行内核 BTF 记录的 `sizeof(struct module)` 相等（本机为 **1280** 字节）。

- [ ] **§5.3 `module_layout` 的符号 CRC**：装载还有第二道检查。`module_layout()` 是个空函数，参数全是 `struct module`、`struct kernel_param` 等结构体指针；genksyms 会把结构体定义**递归展开**进 CRC，所以它就是这些结构体布局的指纹。一旦按 §5.2 改了结构体，CRC 必然变化，而模块里的 CRC 来自内核树的 `Module.symvers`（外部模块编译时 modpost 通过 `-i <内核树>/Module.symvers` 读取）——那份符号表是改结构体**之前**生成的，于是报：

```text
ringbuf: disagrees about version of symbol module_layout
```

修法有快慢两条：快路只重编 `kernel/module/version.o`，从 kbuild 追加到 `kernel/module/.version.o.cmd` 的 `#SYMVER module_layout 0x...` 行取出新 CRC，写回 `Module.symvers` 后重编模块；稳路是 `make -j$(nproc) vmlinux && make -j$(nproc) modules` 整体重算（约 10~30 分钟）。

- [ ] **内核升级后必须按新版本重建**：本次实验期间 WSL 内核由 `6.18.33.2` 更新为 `6.18.40.1`。`vermagic` 里含版本串，旧模块在新内核上无法装载，因此每棵内核树、每个版本都要各自完整走一遍"建树 → 编译 → 装载"。`ls -d ~/kernel-*` 可以确认当前存在哪些版本的源码树。

> 记法：`Invalid module format` 先看 `dmesg`——`version magic` 找版本、`this_module section size` 找结构体 ABI、`disagrees about version of symbol` 找符号 CRC。

## 6. 一键复现

> 覆盖：如何一条命令跑完全部测试并落下可交付的原始日志。

- [ ] **前置**：驱动已加载、设备可写。

```bash
cd ~/ringbuf-driver
sudo insmod ringbuf.ko          # 已加载则跳过
sudo chmod 666 /dev/ringbuf     # device_create 默认 0600 root:root，非 root 会 EACCES
```

- [ ] **一条命令跑完并落盘**：`scripts/run-tests.sh` 会先采集环境与 ABI 信息，再按 smoke → bench → aio 的顺序执行，全部输出同时打印到终端并写入 `test-results/raw-<时间戳>.txt`。

```bash
bash scripts/run-tests.sh 64     # 参数 = 每组 MiB 数，默认 64；求稳用 1024
```

产物字段：内核完整版本与启动时间、内核源码树列表、模块内嵌 `vermagic`、`this_module` 段尺寸、`module_layout` 的 CRC、运行内核 BTF 的 `sizeof(struct module)` 对比、以及三个用例的原始输出与墙钟耗时。

- [ ] **退出条件**：三个用例的退出码均为 0，且 `smoke`/`aio` 各自打印 `PASS`；否则按 §5 的顺序查 `dmesg`。

```bash
sudo dmesg | tail -5
sudo rmmod ringbuf              # 退出时会检查是否仍有未完成的异步请求
```

> 记法：把"跑测试"变成"一条命令 + 一个日志文件"，报告与复查都只需引用那个文件。

---

## 附录 A 速查表

| 目的 | 命令 | 详见 |
|---|---|---|
| 编译驱动与测试程序 | `make && make test` | §6 |
| 加载并放开权限 | `sudo insmod ringbuf.ko && sudo chmod 666 /dev/ringbuf` | §6 |
| 跑全部测试并落盘 | `bash scripts/run-tests.sh 64` | §6 |
| 单跑数据通路 / 回绕 | `./tests/ringbuf_smoke_test` | §2 |
| 单跑真异步验证 | `./tests/ringbuf_aio_test` | §2 |
| 单跑吞吐基准 | `./tests/ringbuf_bench 64` | §3 |
| 查装载失败原因 | `sudo dmesg \| tail -3` | §5 |
| 量 `sizeof(struct module)` | `python3 scripts/kernel_struct_size.py module` | §5.2 |
| 编译期模块元信息 | `strings ringbuf.ko \| grep -E '^(vermagic\|name)='` | §1 |
| 卸载 | `sudo rmmod ringbuf` | §6 |

## 附录 B 学习路径与进度表

| 阶段 | 对应章节 | 内容 | 目标 | 状态 |
|---|---|---|---|---|
| 1 | 1 | 被测对象与测试环境 | 说清驱动接口清单、best-effort 语义与实测环境 | ☐ |
| 2 | 2 | 测试用例设计 | 知道三个用例各自断言什么、哪些 I/O 模型没覆盖 | ☐ |
| 3 | 3 | 实测结果 | 拿到可引用进报告的原始输出与四张吞吐表 | ☐ |
| 4 | 4 | 结果解读 | 能解释四条曲线的形状与深度退化现象 | ☐ |
| 5 | 5 | 构建与装载的三个易错点 | 遇 `Invalid module format` 能用 dmesg 定位到具体检查 | ☐ |
| 6 | 6 | 一键复现 | 一条命令跑完全部用例并产出日志文件 | ☐ |

> 时间紧的读法：**核心速览 → 第 3 节 → 第 4 节 → 附录 A**。

## 附录 C 当前缺口

- **测量精度**：本报告为每组 64 MiB 的两次重复，抖动最大到 45%；建议以 `bash scripts/run-tests.sh 1024` 复测并取中位数。深度退化的三条推断假设（锁竞争、工作集超 cache、碎片请求）尚未逐条隔离验证。
- **未覆盖的驱动能力**（驱动侧已实现，尚无用例）：缓冲区满时的非阻塞写 `-EAGAIN`；阻塞 `read`/`write` 的跨进程唤醒断言；`poll`/`epoll`（含 `EPOLLET` 边沿触发）；`fasync`/`SIGIO` 的注册、收信号与 `close` 后不再通知；`io_uring` 走同一条 `.read_iter`/`.write_iter` 路径。
- **未覆盖的并发拓扑**：目前所有并发用例都靠 `fork` 且共享同一个 `struct file`，因此 `release()` 的清理路径（摘 `fasync`、取消该文件挂起的请求）未被有效触发；也没有多线程（pthread）用例。
- **异步机制原理**：`kiocb` 生命周期、`-EIOCBQUEUED` 契约、`fasync` 与 `io_uring` 的关系，见 `char-driver-async-io.md`（该篇为源码级整理，驱动的实测数据以本文为准）。
