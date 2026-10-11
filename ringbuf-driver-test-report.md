# 4 MiB 环形缓冲区字符设备驱动测试报告（WSL2 实测）

> 用途：给"环形缓冲字符设备驱动"这份实验一个可交付的结论——五个用例各测什么、实测数字是多少、数字背后的原因、以及从零复现与排错的完整路径。面向**已写出驱动、需要证明它对且量化它快不快**的读者。
> 使用方式：每学完一个小点，把 `[ ]` 改成 `[x]`；需要展开某节可随时让我补充。
> 定位：回答"驱动写完了怎么验证"这一层，从"能编译、能 insmod"到"有实测数据可以写进报告"。
> 背景贴合：异步读写在内核里怎么走（`read_iter`/`write_iter` 契约、`kiocb` 生命周期、`fasync`/`io_uring`）见 `char-driver-async-io.md`；那篇标注"内核模块代码未在本机实测"，本文是同一个驱动的真机实测结果，两篇互补。
> 前提：WSL2，运行内核 **6.18.40.1-microsoft-standard-WSL2**，自建同版本内核源码树（WSL 不提供匹配的 `linux-headers`）；按 §5.2 给源码树打 ABI 补丁后，模块的 vermagic 会带 `-dirty` 后缀，这是正常现象（原因见 §5.4）。文中命令均在 WSL 内执行，硬件为 18 逻辑核 / 15.4 GiB 内存。

---

## 核心速览（先看这一页，5 分钟覆盖 80% 内容）

**一句话结论**：五个用例全部通过——数据通路（含 4 MiB 回绕）、`poll`/`epoll` 就绪通知、`/proc/ringbuf` 状态接口、吞吐基准、驱动内真异步都符合设计预期；**异步吞吐与同步同量级**（内存环形缓冲没有真实 I/O 延迟可隐藏），而**并发深度增大反而急剧变慢**（1 MiB 块下深度 1 → 32 掉到 1/27）。

### ① 关键数字（每组 64 MiB，三次重复的区间）

| 模式 | 峰值吞吐 | 峰值出现在 | 最低点 |
|---|---|---|---|
| 同步流水线（双进程） | 10.6 ~ 13.1 GB/s | 256 KiB | 4 KiB 仅 3.0 ~ 3.3 GB/s |
| 单进程乒乓 | 16.9 ~ 19.3 GB/s | 256 KiB | 4 KiB 约 4.5 ~ 5.3 GB/s |
| 异步读（原生 AIO） | 9.9 ~ 12.3 GB/s | 64 KiB，深度 1~8 | **1 MiB × 深度 32 仅 0.3 ~ 1.9 GB/s** |
| 异步写（原生 AIO） | 9.1 ~ 11.1 GB/s | 64 KiB ~ 1 MiB，深度 1 | 4 KiB 仅 2.2 ~ 2.9 GB/s |

### ② 复现命令（三条）

```bash
sudo insmod ~/ringbuf-driver/ringbuf.ko && sudo chmod 666 /dev/ringbuf
cd ~/ringbuf-driver && make test
bash scripts/run-tests.sh 64      # 采集环境 + 顺序跑五个用例，落盘到 test-results/
```

### ③ 四个易错点（前三个本次实测踩到，详见 §5）

| 现象（`insmod` 统一报 `Invalid module format`） | 根因 | 处理 |
|---|---|---|
| dmesg 显示 `version magic '...' should be '...'` | `scripts/setlocalversion` 只认名为 `v<KERNELVERSION>` 的附注 tag，仓库 tag 命名不同就追加 `+` | 补附注 tag，重新生成版本头文件（§5.1） |
| dmesg 显示 `.gnu.linkonce.this_module section size must match ...` | 关掉 `DEBUG_INFO_BTF` 后 `struct module` 少了受 `CONFIG_DEBUG_INFO_BTF_MODULES` 保护的字段 | 该 `#ifdef` 改为无条件编译（§5.2） |
| dmesg 显示 `disagrees about version of symbol module_layout` | `struct module` 改了但符号表未重算，`module_layout` 的 CRC 仍是旧值 | 重算符号表 CRC（§5.3） |
| `vermagic` 带 `-dirty` 后缀，看着像不匹配 | 打过 ABI 补丁的内核树有未提交改动，`setlocalversion` 就加 `-dirty` | **无需处理**：模块带 `__versions` 段时内核只比较第一个空格之后的部分（§5.4） |

### ④ 阅读路线

只求跑通 → §6；要看数据 → §3；要理解数字为何如此 → §4；装不上 → §5。

---

## 1. 被测对象与测试环境

> 覆盖：这份报告测的是哪个驱动、它提供哪些接口、在什么环境下测。不讲异步机制原理（见 `char-driver-async-io.md`）。

- [ ] **被测驱动是一份 4 MiB 环形缓冲区字符设备**：`vzalloc` 分配缓冲区，`alloc_chrdev_region` + `cdev_add` 注册，`device_create` 生成 `/dev/ringbuf`，`proc_create` 生成只读状态文件 `/proc/ringbuf`。

驱动注册的入口与语义：

| 入口 | 语义 |
|---|---|
| `.read` / `.write` | 同步读写，非阻塞模式下无数据/无空间返回 `-EAGAIN` |
| `.read_iter` / `.write_iter` | 内核 AIO 与 `io_uring` 的入口；缓冲就绪则在锁内直接拷贝 |
| `.read_iter` / `.write_iter`（请求不可立即满足时） | 把请求挂到 `async_readers` / `async_writers` 链表，返回 `-EIOCBQUEUED`，稍后由**反方向的操作**调用 `ki_complete()` 完成 |
| `.poll` | 就绪通知：有数据返回 `EPOLLIN\|EPOLLRDNORM`，有空间返回 `EPOLLOUT\|EPOLLWRNORM` |
| `.fasync` | `SIGIO` 通知，`kill_fasync` 在状态变化处调用 |
| `.open` | 设置 `FMODE_NOWAIT`，声明支持非阻塞完成 |
| `/proc/ringbuf` | 只读状态快照（0444）：`capacity`/`used`/`free`/`read_pos`/`write_pos`/`async_readers`/`async_writers`/`fasync` |

- [ ] **读写都是"尽力而为"（best-effort）语义**：返回值是**实际搬运的字节数**，可以小于 `count`；调用方必须循环直到搬完。异步请求同理，完成事件里的 `res` 可能小于申请长度。
- [ ] **环形状态只有三个量**：`read_pos`、`write_pos`、`data_len`；空 = `data_len == 0`，满 = `data_len == RINGBUF_BUFFER_SIZE`。位置在拷贝时回绕，不靠预留槽位区分空满。
- [ ] **并发保护用 `mutex` + 两个等待队列**（`read_wait`、`write_wait`），不用自旋锁——因为拷贝走 `copy_to_iter`/`copy_from_iter`，可能睡眠。
- [ ] **procfs 只读是刻意的**：状态接口只暴露快照，不承担控制职责（要控制就用 `ioctl`），避免把 procfs 当成万能配置通道。

实测环境（由 `scripts/run-tests.sh` 自动采集）：

| 项 | 值 |
|---|---|
| 运行内核 | `6.18.40.1-microsoft-standard-WSL2`，`#1 SMP PREEMPT_DYNAMIC`，构建于 2026-07-31（gcc 13.2.0） |
| CPU | 18 逻辑核，Intel Core Ultra 5 125H |
| 内存 | 15.4 GiB |
| 用户态编译器 | gcc 11.4.0（Ubuntu 22.04） |
| 关键内核配置 | `CONFIG_MODVERSIONS=y`、`CONFIG_MODULE_UNLOAD=y`、`CONFIG_AIO=y`、`CONFIG_IO_URING=y` |
| 模块 `vermagic` | `6.18.40.1-microsoft-standard-WSL2-dirty SMP preempt mod_unload modversions` |
| `sizeof(struct module)` | 运行内核 BTF 记录 **1280** 字节；`.ko` 的 `.gnu.linkonce.this_module` 段 **1280** 字节（一致） |
| 符号 CRC | 模块内 `module_layout` = `0x48a0def8` |
| 设备节点 | `crw-rw-rw- root root 239,0 /dev/ringbuf` |
| 状态接口 | `-r--r--r-- root root /proc/ringbuf` |

> 记法：装载要连过几道关——vermagic（带 CRC 时只比标志位，见 §5.4）、`struct module` 段尺寸、`module_layout` 符号 CRC；任何一道不过，`insmod` 都只说"Invalid module format"，必须看 `dmesg` 才知道是哪一道。

## 2. 测试用例设计

> 覆盖：五个测试程序各自验证什么、覆盖哪些 I/O 模型、哪些没覆盖。

- [ ] **`tests/ringbuf_smoke_test.c` 测数据通路与回绕**（单进程，`O_NONBLOCK`）：空缓冲区读必须 `-EAGAIN`；写 `SIZE-1000` 把 `write_pos` 推到接近末端；读走 1000；再写 2000 使其**跨越 4 MiB 边界**；循环读回全部 4 MiB，按 `gen(i) = (i*31+7) & 0xff` 校验整条字节流。
- [ ] **`tests/ringbuf_poll_test.c` 测 poll/epoll 就绪通知**：空缓冲 `poll(POLLIN)` 必须超时、`poll(POLLOUT)` 立即就绪；写满后 `POLLOUT` 必须超时；`fork` 出子进程**阻塞在 `poll()` 上**，父进程写入后子进程必须被唤醒（并把等待时长回报父进程校验，以排除"靠超时返回"的假通过）；epoll **水平触发**在数据未读走时反复报告，**边沿触发**只投递一次、新数据到来才再报。
- [ ] **`tests/ringbuf_proc_test.c` 测 `/proc/ringbuf` 字段**：读走/写入后 `used`、`free`、`read_pos`、`write_pos` 必须与操作量严格对应（位置按模 4 MiB 前移）；写满后 `free=0`；**提交一个会被挂起的 AIO 读后 `async_readers=1`，被写入完成后归零**；`fasync` 随 `O_ASYNC` 注册/注销变化；以 `O_WRONLY` 打开必须失败。
- [ ] **`tests/ringbuf_aio_test.c` 测驱动内真异步**（单进程，原生 AIO）：缓冲区为空时 `io_submit` 一个 16 字节异步读，必须**立即返回**；带零超时的 `io_getevents` 必须返回 **0 个事件**（证明请求被挂起而非同步完成）；随后同步 `write` 16 字节反向完成它，`io_getevents` 拿到 `res=16`、`data=0x1234`。
- [ ] **`tests/ringbuf_bench.c` 测吞吐量**，共四组：同步流水线（`fork` 出生产者/消费者，双方阻塞同步）、单进程乒乓（写完一块立刻读回）、异步读、异步写（父进程 AIO 保持 `depth` 个在途请求，子进程做同步对端）。
- [ ] **覆盖矩阵**（本次实测的能力边界）：

| 维度 | smoke | poll | proc | aio | bench 同步两组 | bench 异步两组 |
|---|---|---|---|---|---|---|
| 同步 `read`/`write` | 是 | 是 | 是 | 是（写那一步） | 是 | 子进程是 |
| 原生异步 I/O | 否 | 否 | 是（观察挂起） | 是 | 否 | 是 |
| 阻塞语义 | 否（用 `O_NONBLOCK`） | 子进程是 | 否 | 否 | 是 | 子进程是 |
| 非阻塞 `-EAGAIN` | 是 | 是 | 是 | 否 | 否 | 否 |
| `poll`/`epoll`（LT / ET） | 否 | **是** | 否 | 否 | 否 | 否 |
| procfs 状态 | 否 | 否 | **是** | 否 | 否 | 否 |
| 多进程（`fork`） | 否 | 是 | 否 | 否 | 是 | 是 |
| **多线程（pthread）** | 否 | 否 | 否 | 否 | 否 | 否 |
| `fasync` 收信号、`io_uring` | 否 | 否 | 只验状态 | 否 | 否 | 否 |

- [ ] **测试顺序不能调换**：`ringbuf_aio_test` 要求缓冲区初始为空，因此固定为 smoke → poll → proc → bench → aio；前三者结束时都会把缓冲区清空。
- [ ] **关于"多进程而非多线程"**：`fork` 之后父子**共享同一个 `struct file`**（子进程继承的是已打开的 fd，不是各自 `open`）。好处是父进程挂起的异步请求能被子进程的同步操作完成、`poll` 的唤醒路径也跨了任务边界；副作用是 `release()` 里的清理路径（摘 `fasync`、取消挂起请求）要等父子都关闭 fd 才会执行。

> 记法：断言"真异步"的最小证据组合 = `io_submit` 立即返回 + 零超时查询到 0 个完成事件 + 反方向操作后拿到完成事件；断言"边沿触发"的最小证据 = 不读走数据时第二次 `epoll_wait` 返回 0。

## 3. 实测结果

> 覆盖：五个用例的原始输出与吞吐表，不解释原因（见 §4）。数据来自 `test-results/raw-20261011-153329.txt`（当前代码，含 procfs）。

- [ ] **五个用例全部通过**，退出码均为 0：

| 用例 | 结果 | 墙钟 |
|---|---|---|
| ① `ringbuf_smoke_test` | PASS | 0.033 s |
| ② `ringbuf_poll_test` | PASS | 1.438 s（含 200 ms 唤醒等待与多次 200 ms 超时判定） |
| ③ `ringbuf_proc_test` | PASS | 0.076 s |
| ④ `ringbuf_bench 64` | 退出码 0 | 1.489 s |
| ⑤ `ringbuf_aio_test` | PASS | 0.085 s |

- [ ] **用例 ①：数据通路与回绕**

```text
[1] /dev/ringbuf 打开成功
[2] 空缓冲区 read() = -EAGAIN（非阻塞语义正确）
[3] 写入 4193304 字节，write_pos 已接近末端
[4] 读回 1000 字节，内容校验通过
[5] 再写入 2000 字节 —— 已跨越 4 MiB 边界回绕
[6] 读回 4194304 字节，跨回绕的完整序列校验通过
PASS: 缓冲区读写（含回绕）工作正常
```

- [ ] **用例 ②：poll 与 epoll 的四种行为**

```text
[1] 空缓冲区 poll(POLLIN, 200ms) 超时返回 0（未假报可读）
[1] 空缓冲区 poll(POLLOUT) 立即就绪（有空闲可写）
[2] 写满 4 MiB 后：poll(POLLOUT) 超时、poll(POLLIN) 就绪
[3] 子进程阻塞在 poll(POLLIN)，父进程写入 200ms 后被唤醒（revents=0x1）
[4] epoll 水平触发：数据未读走时反复报告可读
[4] epoll 边沿触发：无新边沿时不重复报告（与水平触发区分开）
[4] epoll 边沿触发：新数据产生新边沿（此时可读 7 字节）
PASS: poll/epoll 就绪通知工作正常
```

第 [3] 行的 200 ms 正是父进程 `fork` 后的固定延时，说明子进程确实是**被驱动的 `wake_up` 唤醒**，而不是靠超时返回。

- [ ] **用例 ③：`/proc/ringbuf` 与真实操作逐项对齐**（`used`/`free` 单位字节）

```text
      [1] 排空后          used=0 free=4194304 read_pos=1017 write_pos=1017 async_r=0 async_w=0 fasync=0
[1] 空缓冲区：capacity/used/free 自洽，挂起计数为 0
      [2] 写入 100000 后  used=100000 free=4094304 read_pos=1017 write_pos=101017 async_r=0 async_w=0 fasync=0
[2] 写入后 used/free/write_pos 都与操作量一致
      [3] 读走 40000 后   used=60000 free=4134304 read_pos=41017 write_pos=101017 async_r=0 async_w=0 fasync=0
[3] 读走后 used/read_pos 都与操作量一致
      续写 4134304 字节把缓冲区填满
      [4] 写满后          used=4194304 free=0 read_pos=41017 write_pos=41017 async_r=0 async_w=0 fasync=0
[4] 写满 4 MiB 后 free 为 0
      排空 4194304 字节
      [5] 排空后          used=0 free=4194304 read_pos=41017 write_pos=41017 async_r=0 async_w=0 fasync=0
[5] 排空后 used 归零
      [6] 提交挂起读之后 used=0 free=4194304 read_pos=41017 write_pos=41017 async_r=1 async_w=0 fasync=0
      [6] 被写入完成之后 used=0 free=4194304 read_pos=41033 write_pos=41033 async_r=0 async_w=0 fasync=0
[6] 挂起请求可观测：提交后 async_readers=1，完成后归零且 res=16
      注册 SIGIO（F_SETOWN + O_ASYNC）后 fasync=1
[7] fasync 字段随 SIGIO 注册/注销变化
[8] /proc/ringbuf 只读：O_WRONLY 打开被拒绝（Permission denied）
PASS: /proc/ringbuf 状态接口与真实状态一致
```

第 [6] 行是这份报告里唯一能**直接看见内核态挂起队列长度**的地方：AIO 请求提交后 `async_readers` 从 0 变 1，写入把它完成之后又归零，两次快照之间 `used` 始终为 0（那 16 字节被挂起的读直接取走了）。

- [ ] **用例 ④：吞吐量（每组 64 MiB，单位 MB/s）**

| 块大小 | 同步流水线 | 单进程乒乓 |
|---|---|---|
| 4 KiB | 2983.3 | 4691.7 |
| 64 KiB | 11095.4 | 13705.6 |
| 256 KiB | 11840.2 | 16940.7 |
| 1 MiB | 8290.0 | 8925.3 |
| 4 MiB | 3942.9 | 11104.1 |

| 块大小 × 深度 | 异步读 | 异步写 |
|---|---|---|
| 4 KiB × 1 | 3329.7 | 2205.2 |
| 4 KiB × 8 | 3230.9 | 2364.9 |
| 4 KiB × 32 | 3150.0 | 2455.8 |
| 64 KiB × 1 | 9853.2 | 9069.3 |
| 64 KiB × 8 | 10565.7 | 9312.5 |
| 64 KiB × 32 | 7296.3 | 7654.3 |
| 1 MiB × 1 | 8271.0 | 9268.9 |
| 1 MiB × 8 | 3229.1 | 7841.8 |
| 1 MiB × 32 | **299.6** | 3840.4 |

- [ ] **重复性：同一配置三次运行的离散度**（同机、同参数，仅时间不同）

| 观测点 | 第 1 次 | 第 2 次 | 第 3 次（本次） | 极差 |
|---|---|---|---|---|
| 同步流水线 64 KiB | 7914.9 | 11483.4 | 11095.4 | 45% |
| 同步流水线 256 KiB | 10603.8 | 13075.3 | 11840.2 | 23% |
| 乒乓 256 KiB | 19272.1 | 18012.4 | 16940.7 | 14% |
| 异步读 1 MiB × 1 | 8964.4 | 9745.0 | 8271.0 | 18% |
| 异步读 1 MiB × 32 | 1727.8 | 1898.4 | **299.6** | **6 倍** |

- [ ] **内核侧无异常**：重载模块后 `dmesg` 只有两行，测试期间没有新增告警；卸载也是干净的（先 `unloaded` 再重新 `registered`）。

```text
[227743.815732] ringbuf: unloaded
[227743.917991] ringbuf: registered /dev/ringbuf (4194304 bytes) and /proc/ringbuf
```

> 记法：判断"数据对不对"看用例 ①②③⑤，判断"快不快"看用例 ④，二者不可互相替代；引用任何一个吞吐数字都要同时引用它的离散度。

## 4. 结果解读

> 覆盖：四组表的形状说明什么、为什么异步不比同步快、为什么深度增大反而变慢，以及本次测量的局限。原因分析中标注"推断"者为待验证假设。

- [ ] **同步流水线：4 KiB 到 256 KiB 上升，4 MiB 回落。** 4 KiB 只有约 3.0 GB/s，瓶颈是系统调用与拷贝次数（每 MiB 需要 256 次读写）；块增大到 256 KiB 后接近内存带宽上限（约 12 ~ 13 GB/s）；4 MiB 反而降到约 3.9 GB/s，因为单次拷贝 4 MiB 已超出各级 cache 的工作集，且两个进程几乎无法重叠搬运。
- [ ] **单进程乒乓在中等块下最高（约 17 ~ 19 GB/s），属于预期结果。** 没有第二个进程的调度与同步开销，写完立刻读回时数据仍在 L2/L3 里，所以它测的是"缓存内往返上限"，不代表真实并发场景的吞吐。它在 4 MiB 处（11104）反超流水线（3942.9），正说明流水线的瓶颈是"两个进程争用拷贝带宽"而不是设备本身。
- [ ] **异步吞吐与同步同量级，符合设计预期。** 缓冲区在内存里，`ki_complete` 只是在另一个上下文里做一次 `memcpy`，没有真实 I/O 延迟可以隐藏，异步只额外付出 `io_submit` + `io_getevents` 两次系统调用的成本。因此"异步没比同步快"不是缺陷，而是这个场景的必然结果。
- [ ] **深度增大反而变慢，且块越大越严重**：三种块下的异步读趋势完全一致（64 KiB 在深度 8 见顶后回落，1 MiB 从深度 1 起一路暴跌）。1 MiB × 32 这一格三次测量分别是 1727.8 / 1898.4 / 299.6 MB/s——**前两次还在 GB 量级，第三次直接掉到 0.3 GB/s（同格耗时 0.214 s 完成 64 MiB）**。可能的原因（**推断，尚未逐条隔离验证**）：
  - 环形缓冲只有 4 MiB，而 1 MiB × 32 的在途请求量达 32 MiB，绝大多数请求只能挂在链表上；完成时又在**持有 `mutex`** 的状态下逐个拷贝，锁的持有时间被拉长，与对方进程的读写直接争用；
  - 每个在途请求必须独占用户缓冲区（`depth × block`），深度 32 时工作集 32 MiB 超出 cache，拷贝退化为内存带宽受限；
  - 基准程序的补齐逻辑在"部分完成"时会按剩余量重新提交，深度大时碎片请求更多，进一步放大每次调用的固定开销；
  - 深度大时"提交—挂起—完成—收割"的调度往返次数也随之增加，可能触发调度抖动的正反馈（第三次的极端值更像这类抖动，而非稳定态的带宽下降）。
  - 实践结论：**并发深度要与缓冲容量和块大小匹配**，本机 4 MiB 缓冲下 64 KiB 块是异步路径的甜点；把深度当"免费加速按钮"会在 1 MiB 块上付出 27 倍代价。
- [ ] **本次测量的局限（必须与数字一起引用）**：每组仅 64 MiB，单行耗时落在 3 ms ~ 214 ms，抖动可观——同步流水线 64 KiB 的三次极差达 45%。因此这些数字**只用于判断趋势与量级，不宜作为精确指标**；要得到稳定数据，请把参数放大并重复取中位数：

```bash
bash scripts/run-tests.sh 1024      # 每组 1024 MiB，单行耗时进入百毫秒量级
```

> 记法：内存环缓冲的吞吐看"块大小"（拷贝效率）而非"并发深度"（那是在隐藏真实 I/O 延迟时才起作用）；深度是**受缓冲容量约束**的参数，不是越大越好。

## 5. 构建与装载的易错点（本次实测踩到）

> 覆盖：`insmod: Invalid module format` 的三层根因与修法、`-dirty` vermagic 为何无害，以及内核升级后必须重建这件事。这是本次实验最花时间的部分，值得单独记住。

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

修完可自检：`.ko` 里 `.gnu.linkonce.this_module` 段的 `sh_size` 应与运行内核 BTF 记录的 `sizeof(struct module)` 相等（本机两侧都是 **1280** 字节）。

- [ ] **§5.3 `module_layout` 的符号 CRC**：装载还有第二道检查。`module_layout()` 是个空函数，参数全是 `struct module`、`struct kernel_param` 等结构体指针；genksyms 会把结构体定义**递归展开**进 CRC，所以它就是这些结构体布局的指纹。一旦按 §5.2 改了结构体，CRC 必然变化，而模块里的 CRC 来自内核树的 `Module.symvers`（外部模块编译时 modpost 通过 `-i <内核树>/Module.symvers` 读取）——那份符号表是改结构体**之前**生成的，于是报：

```text
ringbuf: disagrees about version of symbol module_layout
```

修法有快慢两条：快路只重编 `kernel/module/version.o`，从 kbuild 追加到 `kernel/module/.version.o.cmd` 的 `#SYMVER module_layout 0x...` 行取出新 CRC，写回 `Module.symvers` 后重编模块；稳路是 `make -j$(nproc) vmlinux && make -j$(nproc) modules` 整体重算（约 10~30 分钟）。**补丁生效后这个值确实变了**：修复前 `.ko` 里是 `0x0db00678`，修复后是 `0x48a0def8`——这就是"结构体布局进 CRC"的直接证据。

- [ ] **§5.4 vermagic 里的 `-dirty` 无害**：给源码树打了 §5.2 的补丁后，`scripts/setlocalversion` 检测到树有未提交改动，就把模块的 vermagic 从 `...-WSL2` 变成 `...-WSL2-dirty`。它看着像版本不匹配，但模块照样装载成功，原因是内核比较 vermagic 时**在模块带 CRC 的情况下会跳过第一个空格之前的部分**：

```c
/* kernel/module/version.c，注释就是原文 */
/* First part is kernel version, which we ignore if module has crcs. */
int same_magic(const char *amagic, const char *bmagic, bool has_crcs)
{
	if (has_crcs) {
		amagic += strcspn(amagic, " ");
		bmagic += strcspn(bmagic, " ");
	}
	return strcmp(amagic, bmagic) == 0;
}
```

`has_crcs` 传的是 `info->index.vers`（模块是否有 `__versions` 段），本驱动在 `CONFIG_MODVERSIONS=y` 下必然有。所以真正把关的是**段尺寸检查与符号 CRC**，版本串只用来比对 `SMP preempt mod_unload modversions` 这些标志位。理解了这一点，就不会在看到 `-dirty` 时误判方向。

- [ ] **内核升级后必须按新版本重建**：本次实验期间 WSL 内核由 `6.18.33.2` 更新为 `6.18.40.1`。由于 §5.4，模块**可能**带着旧版本串也能装载，但 `Module.symvers` 与内核头文件仍应按新树重建才可靠；`ls -d ~/kernel-*` 可以确认当前存在哪些版本的源码树。

> 记法：`Invalid module format` 先看 `dmesg`——`version magic` 找版本、`this_module section size` 找结构体 ABI、`disagrees about version of symbol` 找符号 CRC；看到 `-dirty` 不必惊慌（§5.4）。

## 6. 一键复现

> 覆盖：如何一条命令跑完全部测试并落下可交付的原始日志。

- [ ] **前置**：驱动已加载、设备可写、procfs 接口存在（重载模块后才会出现）。

```bash
cd ~/ringbuf-driver
sudo rmmod ringbuf 2>/dev/null; sudo insmod ringbuf.ko   # 换过驱动代码后必须重载
sudo chmod 666 /dev/ringbuf     # device_create 默认 0600 root:root，非 root 会 EACCES
cat /proc/ringbuf               # 状态接口：0444，任何用户可读
```

- [ ] **一条命令跑完并落盘**：`scripts/run-tests.sh` 会先采集环境与 ABI 信息，再按 smoke → poll → proc → bench → aio 的顺序执行，全部输出同时打印到终端并写入 `test-results/raw-<时间戳>.txt`。

```bash
bash scripts/run-tests.sh 64     # 参数 = 每组 MiB 数，默认 64；求稳用 1024
```

产物字段：内核完整版本与启动时间、内核源码树列表、模块内嵌 `vermagic`、`this_module` 段尺寸、`module_layout` 的 CRC、运行内核 BTF 的 `sizeof(struct module)` 对比、`/dev/ringbuf` 与 `/proc/ringbuf` 权限、以及五个用例的原始输出与墙钟耗时。

- [ ] **退出条件**：五个用例的退出码均为 0，且 smoke / poll / proc / aio 各自打印 `PASS`；否则按 §5 的顺序查 `dmesg`。

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
| 看状态快照 | `cat /proc/ringbuf` | §1 |
| 跑全部测试并落盘 | `bash scripts/run-tests.sh 64` | §6 |
| 单跑数据通路 / 回绕 | `./tests/ringbuf_smoke_test` | §2 |
| 单跑 poll / epoll | `./tests/ringbuf_poll_test` | §2 |
| 单跑 procfs 状态校验 | `./tests/ringbuf_proc_test` | §2 |
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
| 2 | 2 | 测试用例设计 | 知道五个用例各自断言什么、哪些 I/O 模型没覆盖 | ☐ |
| 3 | 3 | 实测结果 | 拿到可引用进报告的原始输出与吞吐表（含离散度） | ☐ |
| 4 | 4 | 结果解读 | 能解释四条曲线的形状与深度退化现象 | ☐ |
| 5 | 5 | 构建与装载的易错点 | 遇 `Invalid module format` 能用 dmesg 定位到具体检查 | ☐ |
| 6 | 6 | 一键复现 | 一条命令跑完五个用例并产出日志文件 | ☐ |

> 时间紧的读法：**核心速览 → 第 3 节 → 第 4 节 → 附录 A**。

## 附录 C 当前缺口

- **测量精度**：本报告为每组 64 MiB 的三次重复，同步流水线 64 KiB 的极差达 45%，异步读 1 MiB × 32 甚至出现 6 倍差异；建议以 `bash scripts/run-tests.sh 1024` 复测并取中位数。深度退化的四条推断（锁竞争、工作集超 cache、碎片请求、调度抖动）尚未逐条隔离验证——建议的隔离实验是：固定 1 MiB 块，单独扫深度 1/2/4/8/16/32，并记录每个请求从提交到完成的延迟分布。
- **未覆盖的驱动能力**（驱动侧已实现，尚无用例）：缓冲区满时的非阻塞写 `-EAGAIN`；阻塞 `read`/`write` 的跨进程唤醒断言；`fasync` **实际收到 `SIGIO` 信号**（当前只验证了 proc 里的注册状态与内核队列变化）；`close()` 之后不再收信号；`io_uring` 走同一条 `.read_iter`/`.write_iter` 路径。
- **未覆盖的并发拓扑**：目前所有并发用例都靠 `fork` 且共享同一个 `struct file`，因此 `release()` 的清理路径（摘 `fasync`、取消该文件挂起的请求）未被有效触发；也没有多线程（pthread）用例。
- **procfs 只做只读**：没有写侧控制（这是刻意设计，见 §1）；如果后续要暴露"重置统计""强制唤醒"之类的操作，应先考虑 `ioctl` 而不是给 procfs 加写接口。
- **异步机制原理**：`kiocb` 生命周期、`-EIOCBQUEUED` 契约、`fasync` 与 `io_uring` 的关系，见 `char-driver-async-io.md`（该篇为源码级整理，驱动的实测数据以本文为准）。
