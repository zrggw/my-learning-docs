# 4 MiB 环形缓冲区字符设备驱动测试报告（WSL2 实测）

> 用途：给"环形缓冲字符设备驱动"这份实验一个可交付的结论——十二个用例各测什么、实测数字是多少、数字背后的原因、以及从零复现与排错的完整路径。面向**已写出驱动、需要证明它对且量化它快不快**的读者。
> 使用方式：每学完一个小点，把 `[ ]` 改成 `[x]`；需要展开某节可随时让我补充。
> 定位：回答"驱动写完了怎么验证"这一层，从"能编译、能 insmod"到"有实测数据与运行时观测可以写进报告"。
> 背景贴合：异步读写在内核里怎么走（`read_iter`/`write_iter` 契约、`kiocb` 生命周期、`fasync`/`io_uring`）见 `char-driver-async-io.md`；那篇标注"内核模块代码未在本机实测"，本文是同一个驱动的真机实测结果，两篇互补。
> 前提：WSL2，运行内核 **6.18.40.1-microsoft-standard-WSL2**，自建同版本内核源码树（WSL 不提供匹配的 `linux-headers`）；按 §5.2 给源码树打 ABI 补丁后，模块的 vermagic 会带 `-dirty` 后缀，这是正常现象（原因见 §5.4）。文中命令均在 WSL 内执行，硬件为 18 逻辑核 / 15.4 GiB 内存。

---

## 核心速览（先看这一页，5 分钟覆盖 80% 内容）

**一句话结论**：十二个用例全部通过——数据通路（含 4 MiB 回绕）、`poll`/`epoll` 就绪通知、`/proc/ringbuf` 状态与计数、吞吐基准、驱动内真异步、接口失败路径（`-EFAULT` 不改状态也不丢数据）、阻塞唤醒与信号打断、异步流程顺序（完成顺序 = 提交顺序）、`close()`/`release()` 时机、io_uring 实际路径、多消费者/多生产者（独立 `open`、不丢不重）都符合设计预期；**异步吞吐与同步同量级**（内存环形缓冲没有真实 I/O 延迟可隐藏）。运行时观测进一步给出三条反直觉但有证据的结论：**异步提交 88.7% 走快路径根本没挂起**、**274,714 次通知只遇上 6 人次等待者**、以及**并发深度买到的是"排队"而不是"并行"**（事件时间线显示挂起请求被对面逐个完成，每个 64 KiB 约 75 µs）。

### ① 关键数字（每组 64 MiB，五次重复的区间）

| 模式 | 峰值吞吐 | 峰值出现在 | 最低点 |
|---|---|---|---|
| 同步流水线（双进程） | 9.7 ~ 13.1 GB/s | 64 ~ 256 KiB | 4 KiB 仅 2.0 ~ 3.3 GB/s |
| 单进程乒乓 | 15.2 ~ 19.3 GB/s | 256 KiB | 4 KiB 约 3.2 ~ 5.3 GB/s |
| 异步读（原生 AIO） | 9.9 ~ 11.7 GB/s | 64 KiB，深度 1 | **1 MiB × 深度 32 仅 0.3 ~ 1.9 GB/s** |
| 异步写（原生 AIO） | 6.9 ~ 11.1 GB/s | 64 KiB ~ 1 MiB，深度 1 | 4 KiB 仅 1.7 ~ 2.9 GB/s |

### ② 运行时画像（累计计数 + 事件时间线）

| 指标 | 值 | 含义 |
|---|---|---|
| 异步读**快路径命中率** | 46487 / 52420 = **88.7%** | 异步提交绝大多数根本不需要挂起 |
| 异步写快路径命中率 | 52378 / 52416 = **99.93%** | 几乎从不因"缓冲满"而挂起 |
| **部分完成率** | 4 / 5971 = **0.07%** | best-effort 契约在实际负载下极少触发 |
| 通知频率 | 274,714 次 / 1904 MB ≈ 每 6.9 KB 一次 | 与块大小同量级 |
| 唤醒人次 | `woken_tasks` = 6（该快照） | 绝大多数通知发生在空队列上 |
| 挂起请求排队时长 | p50 **446 µs**、p90 **795 µs**、p99 **1.256 ms**、max **1.342 ms** | 来自 8788 条事件的时间线（§7.3） |
| 有等待者的通知占比 | 7 / 4293 = **0.16%** | 当前拓扑下几乎没有唤醒放大可省 |

### ③ 复现命令（四条）

```bash
sudo insmod ~/ringbuf-driver/ringbuf.ko && sudo chmod 666 /dev/ringbuf
cd ~/ringbuf-driver && make test
bash scripts/run-tests.sh 64                     # 采集环境 + 每次重编 + 顺序跑十二个用例
sudo bash scripts/trace-ringbuf.sh               # 采集事件时间线并汇总（需 root 写 tracefs）
```

### ④ 六个易错点（前三个本次实测踩到，详见 §5）

| 现象（`insmod` 统一报 `Invalid module format`） | 根因 | 处理 |
|---|---|---|
| dmesg 显示 `version magic '...' should be '...'` | `scripts/setlocalversion` 只认名为 `v<KERNELVERSION>` 的附注 tag，仓库 tag 命名不同就追加 `+` | 补附注 tag，重新生成版本头文件（§5.1） |
| dmesg 显示 `.gnu.linkonce.this_module section size must match ...` | 关掉 `DEBUG_INFO_BTF` 后 `struct module` 少了受 `CONFIG_DEBUG_INFO_BTF_MODULES` 保护的字段 | 该 `#ifdef` 改为无条件编译（§5.2） |
| dmesg 显示 `disagrees about version of symbol module_layout` | `struct module` 改了但符号表未重算，`module_layout` 的 CRC 仍是旧值 | 重算符号表 CRC（§5.3） |
| `vermagic` 带 `-dirty`，看着像不匹配 | 打过 ABI 补丁的内核树有未提交改动 | **无需处理**：模块带 `__versions` 段时内核只比较第一个空格之后的部分（§5.4） |
| 自己写的计数器恒为 0，或断言按旧结果失败 | `wake_up_interruptible()` 的返回值只统计**排他**等待者；累计计数不能当"本次用例的值"；只检查二进制是否存在会导致跑旧程序 | 自己数等待队列；断言一律用增量；脚本每次重编（§5.5、§5.6） |
| 加 tracepoint 后模块编译报 `redefinition of 'xxx'` | `include/trace/define_trace.h` 会带 `TRACE_HEADER_MULTI_READ` 再包含一次事件头 | 把 `static inline` 助手放到多读保护块**之外**，并给 Makefile 加 `ccflags-y += -I$(src)`（§5.7） |

### ⑤ 阅读路线

只求跑通 → §6；要看数据 → §3；要理解数字为何如此 → §4；装不上/对不上账 → §5；想知道运行时能看到什么 → §7。

---

## 1. 被测对象与测试环境

> 覆盖：这份报告测的是哪个驱动、它提供哪些接口、在什么环境下测。不讲异步机制原理（见 `char-driver-async-io.md`）。

- [ ] **被测驱动是一份 4 MiB 环形缓冲区字符设备**：`vzalloc` 分配缓冲区，`alloc_chrdev_region` + `cdev_add` 注册，`device_create` 生成 `/dev/ringbuf`，`proc_create` 生成只读观测接口 `/proc/ringbuf`，并注册 5 个 tracepoint 事件。

驱动注册的入口与语义：

| 入口 | 语义 |
|---|---|
| `.read` / `.write` | 同步读写，非阻塞模式下无数据/无空间返回 `-EAGAIN` |
| `.read_iter` / `.write_iter` | 内核 AIO 与 `io_uring` 的入口；缓冲就绪则在锁内直接拷贝 |
| `.read_iter` / `.write_iter`（请求不可立即满足时） | 把请求挂到 `async_readers` / `async_writers` 链表，返回 `-EIOCBQUEUED`，稍后由**反方向的操作**调用 `ki_complete()` 完成 |
| `.poll` | 就绪通知：有数据返回 `EPOLLIN\|EPOLLRDNORM`，有空间返回 `EPOLLOUT\|EPOLLWRNORM` |
| `.fasync` | `SIGIO` 通知，`kill_fasync` 在状态变化处调用 |
| `.open` | 设置 `FMODE_NOWAIT`，声明支持非阻塞完成 |
| `/proc/ringbuf` | 只读观测接口（0444）：**8 项状态快照** + **24 项累计计数**，详见 §7 |
| tracepoint（`events/ringbuf/`） | 5 个事件：`serve`/`park`/`complete`/`wake`/`sync_wait`，详见 §7 |

- [ ] **读写都是"尽力而为"（best-effort）语义**：返回值是**实际搬运的字节数**，可以小于 `count`；调用方必须循环直到搬完。异步请求同理，完成事件里的 `res` 可能小于申请长度。
- [ ] **环形状态只有三个量**：`read_pos`、`write_pos`、`data_len`；空 = `data_len == 0`，满 = `data_len == RINGBUF_BUFFER_SIZE`。位置在拷贝时回绕，不靠预留槽位区分空满。
- [ ] **并发保护用 `mutex` + 两个等待队列**（`read_wait`、`write_wait`），不用自旋锁——因为拷贝走 `copy_to_iter`/`copy_from_iter`，可能睡眠。
- [ ] **观测分三层，各有分工**：L1 是锁内一起取的瞬时快照（同一份快照内部自洽）；L2 是 per-CPU 累计计数（跨 CPU 求和，属读取瞬间的近似值）；L3 是 tracepoint 事件时间线（带时间戳，与 `sched_*`/`mutex_*` 同一时间轴）。三者不混用、不互推：**计数回答"多少、多快"，时间线回答"谁先谁后、为什么"**。

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
| 观测接口 | `-r--r--r-- root root /proc/ringbuf` |
| tracefs | `/sys/kernel/tracing`（只有 root 可写） |

> 记法：装载要连过几道关——vermagic（带 CRC 时只比标志位，见 §5.4）、`struct module` 段尺寸、`module_layout` 符号 CRC；任何一道不过，`insmod` 都只说"Invalid module format"，必须看 `dmesg` 才知道是哪一道。

## 2. 测试用例设计

> 覆盖：十二个测试程序各自验证什么、覆盖哪些 I/O 模型、哪些没覆盖。

- [ ] **`tests/ringbuf_smoke_test.c` 测数据通路与回绕**（单进程，`O_NONBLOCK`）：空缓冲区读必须 `-EAGAIN`；写 `SIZE-1000` 把 `write_pos` 推到接近末端；读走 1000；再写 2000 使其**跨越 4 MiB 边界**；循环读回全部 4 MiB，按 `gen(i) = (i*31+7) & 0xff` 校验整条字节流。
- [ ] **`tests/ringbuf_poll_test.c` 测 poll/epoll 就绪通知**：空缓冲 `poll(POLLIN)` 必须超时、`poll(POLLOUT)` 立即就绪；写满后 `POLLOUT` 必须超时；`fork` 出子进程**阻塞在 `poll()` 上**，父进程写入后子进程必须被唤醒（并把等待时长回报父进程校验，以排除"靠超时返回"的假通过）；epoll **水平触发**在数据未读走时反复报告，**边沿触发**只投递一次、新数据到来才再报。
- [ ] **`tests/ringbuf_proc_test.c` 测状态快照字段**：读走/写入后 `used`、`free`、`read_pos`、`write_pos` 必须与操作量严格对应（位置按模 4 MiB 前移）；写满后 `free=0`；**提交一个会被挂起的 AIO 读后 `async_readers=1`，被写入完成后归零**；`fasync` 随 `O_ASYNC` 注册/注销变化；以 `O_WRONLY` 打开必须失败。
- [ ] **`tests/ringbuf_stats_test.c` 测累计计数对账**：不是"字段存在"，而是**增量等于已知操作量**——写 100000 字节 → `bytes_written +100000`、`write_ops +1`；读 40000 → `bytes_read +40000`、`read_ops +1`；挂起 64 字节的 AIO 读再用 16 字节写入完成它 → `park_read +1`、`completions_read +1`、`partial_completions +1`（申请 64 得 16）、`completion_bytes +16`；空缓冲非阻塞读 → `eagain +1`；`poll()` 一次 → `poll_calls +1`。另有两条跨字段不变量：无等待者时 `woken_tasks` 增量必须为 0；**缓冲区为空时 `bytes_read` 必须等于 `bytes_written`**。
- [ ] **`tests/ringbuf_aio_test.c` 测驱动内真异步**（单进程，原生 AIO）：缓冲区为空时 `io_submit` 一个 16 字节异步读，必须**立即返回**；带零超时的 `io_getevents` 必须返回 **0 个事件**（证明请求被挂起而非同步完成）；随后同步 `write` 16 字节反向完成它，`io_getevents` 拿到 `res=16`、`data=0x1234`。
- [ ] **`tests/ringbuf_bench.c` 测吞吐量**，共四组：同步流水线（`fork` 出生产者/消费者，双方阻塞同步）、单进程乒乓（写完一块立刻读回）、异步读、异步写（父进程 AIO 保持 `depth` 个在途请求，子进程做同步对端）。
- [ ] **`tests/ringbuf_api_test.c` 测接口契约与失败路径**（单进程）：`lseek()` 必须 `-ESPIPE`；长度为 0 的读写必须返回 0 且不改变状态（**这条当初逼出了一个真缺陷**：`.read`/`.write` 少了 `count == 0` 的立即返回，阻塞 fd 上会睡在 `read_wait` 上，非阻塞 fd 会返回 `-EAGAIN`）；请求 8 MiB（> 4 MiB 容量）按可读量返回；满缓冲非阻塞写 `-EAGAIN` 且 `used` 不变；`poll` 同时关心读写时只有对应方向就绪；**读入非法地址 / 从非法地址写出 → `-EFAULT`，且 `used` 不变、数据不丢、`fault +1`**。
- [ ] **`tests/ringbuf_sync_test.c` 测阻塞读写与信号打断**（`fork`，子进程各自 `open()` 阻塞 fd）：阻塞 `read` 被跨进程写入唤醒、阻塞 `write`（缓冲满）被读出唤醒，且子进程必须**睡够父进程的延时**（排除"靠超时返回"的假通过）；**两个读者都在等时一次写入唤醒全部（`woken_tasks +2`），但 32 字节只够一个消费**，另一个等下一次写入；**被信号打断返回 `-EINTR`**（驱动 `-ERESTARTSYS` 的用户态可见结果）、`eagain` 不变、数据未被消耗。
- [ ] **`tests/ringbuf_aio_flow_test.c` 测异步流程语义**（单进程即可，`io_submit` 不阻塞）：一次提交 8 个挂起读（proc 里 `async_readers = 8`）、分 8 次各写 64 字节 → **完成顺序必须等于提交顺序**（按 `aio_data` 判定）、每个 `res=64`、内容与对应输入逐一相符；部分完成（申请 64 只写 16 → `res=16`、`partial_completions +1`）；**`RWF_NOWAIT` 在空缓冲上立即以 `-EAGAIN` 完成且不挂起**（`nowait_attempt +1`、`park_read` 不变），去掉标志后同一请求就会挂起；**异步写方向**：写满后 4 个 16 KiB 写全部挂起，读出后按提交顺序完成、排空内容等于四个输入缓冲的级联。
- [ ] **`tests/ringbuf_release_test.c` 测 `close()`/`release()` 与挂起请求**：用 proc 的 `fasync` 当"release 是否执行"的探针（`release()` 第一件事就是摘掉本文件的注册）——**`close()` 之后请求仍挂着、`async_readers` 与 `fasync` 都不变**（证明 release 未执行、AIO 请求跨 `close` 有效）；请求被另一个 fd 完成后 release 才执行，此时链表已空 → **取消循环在 AIO 模型下不可达**。
- [ ] **`tests/ringbuf_uring_test.c` 测 io_uring 走哪条路径**：有数据时首次 NOWAIT 尝试即成功（`nowait_success +1`）；空缓冲时**既没立刻 `-EAGAIN` 也没挂起**——实测是 io_uring 把请求挂到驱动 `.poll` 上（`poll_calls +2`），数据到达后重试走同步核（`read_ops +1`）→ **io_uring 不使用 `-EIOCBQUEUED` 挂起路径**，驱动的真异步挂起目前只有原生 AIO 在用。
- [ ] **`tests/ringbuf_multi_test.c` 测多消费者/多生产者**（每个进程各自 `open()`）：定长 64 B 记录（`producer` + `seq` + 由二者确定的载荷），共享位图登记"是否收到过"（用 `__sync_fetch_and_or` 的**旧值**检测重复投递），收尾扫位图检测丢失，每个消费者要求同一生产者的 `seq` **严格递增**（FIFO 重排检测），并断言 `read 返回值 % 64 == 0`（记录被切开即拷贝语义出错）。三种拓扑：1×3（多消费者）、3×1（多生产者，压到 `write_wait` 争抢）、2×2。
- [ ] **覆盖矩阵（前六个用例）**：

| 维度 | smoke | poll | proc | stats | aio | bench 同步两组 | bench 异步两组 |
|---|---|---|---|---|---|---|---|
| 同步 `read`/`write` | 是 | 是 | 是 | 是 | 是（写那一步） | 是 | 子进程是 |
| 原生异步 I/O | 否 | 否 | 是（观察挂起） | 是（观察挂起与完成） | 是 | 否 | 是 |
| 阻塞语义 | 否（用 `O_NONBLOCK`） | 子进程是 | 否 | 否 | 否 | 是 | 子进程是 |
| 非阻塞 `-EAGAIN` | 是 | 是 | 是 | 是 | 否 | 否 | 否 |
| `poll`/`epoll`（LT / ET） | 否 | **是** | 否 | 否 | 否 | 否 | 否 |
| procfs 状态与计数 | 否 | 否 | **是**（状态） | **是**（计数） | 否 | 否 | 否 |
| 多进程（`fork`） | 否 | 是 | 否 | 否 | 否 | 是 | 是 |
| **多线程（pthread）** | 否 | 否 | 否 | 否 | 否 | 否 | 否 |
| `fasync` 收信号、`io_uring` | 否 | 否 | 只验状态 | 否 | 否 | 否 | 否 |

- [ ] **后六个用例补上的维度**（前六个没覆盖到的部分）：

| 维度 | api | sync | aio_flow | release | uring | multi |
|---|---|---|---|---|---|---|
| `-EFAULT` 不改状态、不丢数据 | **是** | 否 | 否 | 否 | 否 | 否 |
| `-ESPIPE` / `count == 0` / 超容量请求 | **是** | 否 | 否 | 否 | 否 | 否 |
| 阻塞唤醒（含"唤醒全部但数据只够一个"） | 否 | **是** | 否 | 否 | 否 | 否 |
| 信号打断 → `-EINTR`、数据未被消耗 | 否 | **是** | 否 | 否 | 否 | 否 |
| 多请求完成顺序 / 部分完成 / `RWF_NOWAIT` | 否 | 否 | **是** | 否 | 否 | 否 |
| 异步写方向与级联内容 | 否 | 否 | **是** | 否 | 否 | 否 |
| `close()` 与挂起请求 / `release()` 执行时机 | 否 | 否 | 否 | **是** | 否 | 否 |
| io_uring 实际走哪条路径 | 否 | 否 | 否 | 否 | **是** | 否 |
| 多消费者/多生产者（独立 `open`、不丢不重） | 否 | 部分（2 读者，仅唤醒语义） | 否 | 否 | 否 | **是** |
| **多线程（pthread）** | 否 | 否 | 否 | 否 | 否 | 否 |

- [ ] **测试顺序**：`ringbuf_aio_test` 要求缓冲区初始为空，因此它排在能造数据的用例之后；其余用例都会在开头先 `rb_drain()`，所以顺序本身不敏感。实际顺序：smoke → poll → proc → stats → bench → aio → api → sync → aio_flow → release → uring → multi。
- [ ] **关于"多进程而非多线程"**：`fork` 之后父子**共享同一个 `struct file`**（子进程继承的是已打开的 fd，不是各自 `open`）。好处是父进程挂起的异步请求能被子进程的同步操作完成、`poll` 的唤醒路径也跨了任务边界；副作用是 `release()` 里的清理路径（摘 `fasync`、取消挂起请求）要等父子都关闭 fd 才会执行。

> 记法：断言"真异步"的最小证据组合 = `io_submit` 立即返回 + 零超时查询到 0 个完成事件 + 反方向操作后拿到完成事件；断言"边沿触发"的最小证据 = 不读走数据时第二次 `epoll_wait` 返回 0。

## 3. 实测结果

> 覆盖：十二个用例的原始输出与吞吐表，不解释原因（见 §4）。用例退出码与墙钟来自 `test-results/raw-20261011-201642.txt`（一次完整运行，12 个用例）；吞吐表来自 `test-results/raw-20261011-185132.txt`；观测类结果（计数画像与事件时间线）见 §7。

- [ ] **十二个用例全部通过**，退出码均为 0：

| 用例 | 结果 | 墙钟 |
|---|---|---|
| ① `ringbuf_smoke_test` | PASS | 0.010 s |
| ② `ringbuf_poll_test` | PASS | 1.410 s（含 200 ms 唤醒等待与多次 200 ms 超时判定） |
| ③ `ringbuf_proc_test` | PASS | 0.090 s |
| ④ `ringbuf_stats_test` | PASS | 0.070 s |
| ⑤ `ringbuf_bench 64` | 退出码 0 | 1.600 s |
| ⑥ `ringbuf_aio_test` | PASS | 0.060 s |
| ⑦ `ringbuf_api_test` | PASS | 0.010 s |
| ⑧ `ringbuf_sync_test` | PASS | 1.110 s（两次 200 ms 唤醒延时 + 300 ms 竞抢 + 200 ms 信号等待） |
| ⑨ `ringbuf_aio_flow_test` | PASS | 0.080 s |
| ⑩ `ringbuf_release_test` | PASS | 0.090 s |
| ⑪ `ringbuf_uring_test` | PASS | 0.250 s（含 200 ms 非阻塞探测） |
| ⑫ `ringbuf_multi_test` | PASS | 0.650 s（三种拓扑，含子进程收敛） |

- [ ] **后六个用例的关键断言与实测**（⑦~⑫，括号里是对应的计数）：

| 用例 | 关键断言 | 实测 |
|---|---|---|
| ⑦ `ringbuf_api_test` | `-ESPIPE`、`count=0` 返回 0、超容量按可读量、满缓冲 `-EAGAIN`、`-EFAULT` 不改状态 | 全通过；读入非法地址与写出非法地址各让 `fault +1`，`used` 在每一步都保持不变，且 EFAULT 之后那 100 字节仍可读（数据未丢） |
| ⑧ `ringbuf_sync_test` | 阻塞唤醒必须睡够延时、`woken_tasks` 增量、信号打断 `-EINTR` | 200 ms / 201 ms 唤醒；`woken_tasks +2` 而 32 字节只够一个读者；打断时 `eagain +0`、`used` 仍为 0 |
| ⑨ `ringbuf_aio_flow_test` | 完成顺序 = 提交顺序、部分完成、`RWF_NOWAIT` 不挂起、写方向级联 | `park_read +8` → 8 个事件按 `aio_data` 递增、每个 `res=64`；部分完成 `res=16`；NOWAIT 时 `park_read +0`；写方向 `park_write +4` 且排空末尾等于四个输入级联 |
| ⑩ `ringbuf_release_test` | `close()` 不取消挂起请求、release 执行时机 | close 后 `async_readers` 仍为 1、`fasync` 仍为 1（release 未执行）；请求完成后 0 ms 内 release 执行、`cancel_on_release +0` |
| ⑪ `ringbuf_uring_test` | 有数据走 NOWAIT 快路径；空缓冲走哪条路径 | `nowait_success +1`；空缓冲 `poll_calls +2`、`read_ops +1`、`park_read +0`（**不使用挂起路径**） |
| ⑫ `ringbuf_multi_test` | 不丢不重、同生产者内顺序严格递增、消费者分布 | 1×3：分布 640/666/740（≈1.16:1）；3×1：6144 条全收；2×2：1156/2940（≈2.54:1，跨轮次在 1.6~2.5 之间波动，但无饿死） |

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

- [ ] **用例 ③：状态快照与真实操作逐项对齐**（`used`/`free` 单位字节）

```text
      [1] 排空后          used=0 free=4194304 read_pos=41017 write_pos=41017 async_r=0 async_w=0 fasync=0
      [2] 写入 100000 后  used=100000 free=4094304 read_pos=41017 write_pos=101017 async_r=0 async_w=0 fasync=0
      [3] 读走 40000 后   used=60000 free=4134304 read_pos=41017 write_pos=101017 async_r=0 async_w=0 fasync=0
      [4] 写满后          used=4194304 free=0 async_r=0 async_w=0 fasync=0
      [5] 排空后          used=0 free=4194304 async_r=0 async_w=0 fasync=0
      [6] 提交挂起读之后  used=0 free=4194304 async_r=1 async_w=0 fasync=0
      [6] 被写入完成之后  used=0 free=4194304 async_r=0 async_w=0 fasync=0
[6] 挂起请求可观测：提交后 async_readers=1，完成后归零且 res=16
[7] fasync 字段随 SIGIO 注册/注销变化
[8] /proc/ringbuf 只读：O_WRONLY 打开被拒绝（Permission denied）
PASS: /proc/ringbuf 状态接口与真实状态一致
```

第 [6] 行是唯一能**直接看见内核态挂起队列长度**的地方：AIO 请求提交后 `async_readers` 从 0 变 1，写入把它完成之后又归零，两次快照之间 `used` 始终为 0（那 16 字节被挂起的读直接取走了）。

- [ ] **用例 ④：累计计数对账**（完整断言链，节选）

```text
[1] 已取计数基线
      eagain               +1
[2] 空缓冲非阻塞读：eagain 恰好 +1
      bytes_written        +100000
      write_ops            +1
[3] 写入 100000 字节：bytes_written/write_ops 与操作量一致
      bytes_read           +40000
      read_ops             +1
[4] 读走 40000 字节：bytes_read/read_ops 与操作量一致
      park_read            +1
      completions_read     +1
      partial_completions  +1
      completion_bytes     +16
[5] 挂起 64 字节、被 16 字节写入完成：park/completions/partial 全部对账一致
      read_wakeups         +1
      write_wakeups        +0
      woken_tasks          +0
[6] 通知频率与唤醒人次被分别统计
      sync_sleep_read      +0
      sync_sleep_write     +0
[7] 本用例全程非阻塞：sync_sleep_* 增量为 0
      poll_calls           +1
[8] poll() 调用一次：poll_calls 恰好 +1
      累计读 = 累计写 = 12723961（缓冲区为空，写入的数据全部被读走）
[9] 跨字段不变量：空缓冲时 bytes_read == bytes_written
PASS: 累计计数与已知操作逐项对账一致
```

- [ ] **用例 ⑤：吞吐量（每组 64 MiB，单位 MB/s）**

| 块大小 | 同步流水线 | 单进程乒乓 |
|---|---|---|
| 4 KiB | 1973.9 | 3209.3 |
| 64 KiB | 6450.1 | 12218.3 |
| 256 KiB | 9680.4 | 15208.1 |
| 1 MiB | 9591.4 | 7959.9 |
| 4 MiB | 3087.9 | 9246.7 |

| 块大小 × 深度 | 异步读 | 异步写 |
|---|---|---|
| 4 KiB × 1 | 2398.5 | 1743.0 |
| 4 KiB × 8 | 2595.7 | 2311.8 |
| 4 KiB × 32 | 3003.2 | 2231.0 |
| 64 KiB × 1 | 11032.7 | 8391.4 |
| 64 KiB × 8 | 7901.4 | 8704.3 |
| 64 KiB × 32 | 5669.4 | 6973.2 |
| 1 MiB × 1 | 6378.1 | 8149.3 |
| 1 MiB × 8 | 3247.7 | 5584.6 |
| 1 MiB × 32 | **1468.5** | 4001.7 |

- [ ] **重复性：同一配置多次运行的离散度**（同机、同参数，仅时间不同）

| 观测点 | 第 1 次 | 第 2 次 | 第 3 次 | 第 4 次 | 第 5 次 | 极差 |
|---|---|---|---|---|---|---|
| 同步流水线 64 KiB | 7914.9 | 11483.4 | 11095.4 | 10409.0 | 6450.1 | 78% |
| 同步流水线 256 KiB | 10603.8 | 13075.3 | 11840.2 | 9160.9 | 9680.4 | 43% |
| 乒乓 256 KiB | 19272.1 | 18012.4 | 16940.7 | 17000.6 | 15208.1 | 27% |
| 异步读 1 MiB × 1 | 8964.4 | 9745.0 | 8271.0 | 8355.8 | 6378.1 | 53% |
| 异步读 1 MiB × 32 | 1727.8 | 1898.4 | 299.6 | 1388.0 | 1468.5 | **6.3 倍** |

- [ ] **内核侧无异常**：重载模块后 `dmesg` 只有两行，测试期间没有新增告警；卸载也是干净的（先 `unloaded` 再重新 `registered`）。

```text
[239626.746819] ringbuf: unloaded
[239626.818431] ringbuf: registered /dev/ringbuf (4194304 bytes) and /proc/ringbuf
```

> 记法：判断"数据对不对"看用例 ①②③⑥，判断"快不快"看用例 ⑤，判断"计数是否可信"看用例 ④；引用任何一个吞吐数字都要同时引用它的离散度。

## 4. 结果解读

> 覆盖：四组表的形状说明什么、为什么异步不比同步快、为什么深度增大反而变慢、计数与时间线各告诉我们什么，以及本次测量的局限。原因分析中标注"推断"者为待验证假设。

- [ ] **同步流水线：4 KiB 到 256 KiB 上升，4 MiB 回落。** 4 KiB 只有约 2.0 ~ 3.3 GB/s，瓶颈是系统调用与拷贝次数（每 MiB 需要 256 次读写）；块增大到 64 ~ 256 KiB 后接近内存带宽上限（约 10 ~ 13 GB/s）；4 MiB 反而降到约 3 ~ 6 GB/s，因为单次拷贝 4 MiB 已超出各级 cache 的工作集，且两个进程几乎无法重叠搬运。
- [ ] **单进程乒乓在中等块下最高（约 15 ~ 19 GB/s），属于预期结果。** 没有第二个进程的调度与同步开销，写完立刻读回时数据仍在 L2/L3 里，所以它测的是"缓存内往返上限"，不代表真实并发场景的吞吐。它在 4 MiB 处常反超流水线（例如 9246.7 对 3087.9），正说明流水线的瓶颈是"两个进程争用拷贝带宽"而不是设备本身。
- [ ] **异步吞吐与同步同量级，符合设计预期。** 缓冲区在内存里，`ki_complete` 只是在另一个上下文里做一次 `memcpy`，没有真实 I/O 延迟可以隐藏，异步只额外付出 `io_submit` + `io_getevents` 两次系统调用的成本。因此"异步没比同步快"不是缺陷，而是这个场景的必然结果。
- [ ] **计数纠正了一个直觉：异步请求绝大多数根本没挂起。** 异步读快路径命中率 88.7%、异步写 99.93%——因为父进程提交请求时缓冲区通常已经就绪，"挂起"只是少数派路径。所以评估异步实现时**快路径的实现质量与挂起路径同等重要**；只测"会不会挂起"会漏掉 88% 的实际流量。
- [ ] **best-effort 语义在实际负载下几乎不触发**：部分完成率 0.07%（4 / 5971）。但它是 API 契约的一部分，必须实现并测试——这次是靠"申请 64 字节、只写入 16 字节"这个刻意构造的用例才把它逼出来。
- [ ] **通知绝大多数发生在空队列上**：274,714 次通知只遇到 6 人次等待者；在 8788 条事件的时间线窗口里，有等待者的通知只占 0.16%（7 / 4293）。因此当前负载（单生产者单消费者、块 ≥ 4 KiB）下**几乎没有唤醒放大可省**——通知合并的收益要等到"多个消费者同时阻塞 + 小块写"的拓扑才会显现。
- [ ] **深度增大反而变慢：时间线把它从"四个推断"收敛成一个机制。** 三种块下的趋势一致（64 KiB 在深度 8 见顶后回落，1 MiB 从深度 1 起一路暴跌），且 1 MiB × 32 那一格五次测量横跨 299.6 ~ 1898.4 MB/s（极差 6.3 倍）。§7.3 的时间线给出了直接证据：
  - 16 个 64 KiB 请求在 **8 µs 内全部挂起**（`parked` 从 1 涨到 16），随后对面（写者）**一次只完成一个**，每个约 **75 µs**；
  - 于是排队时长随队列位置**线性增长**：第 1 个 128 µs，第 16 个 1342 µs；
  - 结论：**并发深度买到的是"排队"，不是"并行"**。吞吐被对面单线程的服务速率锁死（约 1 个请求 / 75 µs），而块越大单次服务越久（1 MiB 请求的服务时间约 16 倍于 64 KiB），所以"深度 32 + 1 MiB"这一格的时间被彻底拉爆——这也解释了为什么深度 1 时 1 MiB 反而不慢。
  - 剩下的次要因素（工作集超 cache、部分完成的碎片请求、调度抖动）仍未逐条隔离，建议的隔离实验见附录 C。
  - 实践结论：**并发深度要与缓冲容量、块大小、以及对面单线程的服务速率匹配**；把深度当"免费加速按钮"会在 1 MiB 块上付出一个数量级的代价。
- [ ] **本次测量的局限（必须与数字一起引用）**：每组仅 64 MiB，单行耗时落在 3 ms ~ 50 ms，运行间离散度最高达 78%。因此这些数字**只用于判断趋势与量级，不宜作为精确指标**；要得到稳定数据，请把参数放大并重复取中位数：

```bash
bash scripts/run-tests.sh 1024      # 每组 1024 MiB，单行耗时进入百毫秒量级
```

> 记法：内存环缓冲的吞吐看"块大小"（拷贝效率）而非"并发深度"（那是在隐藏真实 I/O 延迟时才起作用）；深度是**受对面服务速率约束**的参数，不是越大越好。

## 5. 构建、装载与观测的易错点（本次实测踩到）

> 覆盖：`insmod: Invalid module format` 的三层根因与修法、`-dirty` vermagic 为何无害、内核升级后要重建，以及三个"自己给自己挖的坑"（唤醒人次计数、测试与断言的卫生、tracepoint 头的多读保护）。这是本次实验最花时间的部分。

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

- [ ] **§5.5 `wake_up_interruptible()` 的返回值不是唤醒人次**：我最初用它累计 `woken_tasks`，结果**始终为 0**，而同一份快照里 `sync_sleep_read = 704` 说明确实唤醒过阻塞读者。原因是该返回值只统计被唤醒的**排他**等待者（递减的是 `nr_exclusive` 记账），而 `wait_event_interruptible()` 加入的是**非排他**等待者——`wake_up_interruptible()` 会把它们全部唤醒，却在返回值里一个都不算。正确做法是在通知点自己数等待队列长度：

```c
static unsigned int ringbuf_waiters(wait_queue_head_t *wq)
{
	unsigned long flags;
	struct list_head *pos;
	unsigned int n = 0;

	spin_lock_irqsave(&wq->lock, flags);
	list_for_each(pos, &wq->head)
		n++;
	spin_unlock_irqrestore(&wq->lock, flags);
	return n;
}
```

数队列长度还有个额外好处：它同时覆盖 `poll`/`epoll` 注册的等待者，正是这次 `wake_up` 会唤醒的全部对象（修复后快照里 `woken_tasks` 从 0 变成 6，来源正是 poll 用例中阻塞在 `poll()` 上的子进程）。

- [ ] **§5.6 两个"测试卫生"坑**：①`run-tests.sh` 原先只在"二进制不存在"时才编译，源码改了却跑的是旧二进制——一条已经修好的断言又按旧版跑了一遍；改为**每次都重编**，编译失败即终止。②把累计计数当"本次用例的值"断言：`/proc/ringbuf` 的计数是**自模块装载以来**的累计值，含之前所有用例甚至上一轮 bench 的 704 次阻塞睡眠，所以断言一律先取基线、再比**增量**。

- [ ] **§5.7 tracepoint 头的多读保护**：加事件时间线时，模块编译报 `redefinition of 'ringbuf_next_req_id'`。原因是 `include/trace/define_trace.h` 会带着 `TRACE_HEADER_MULTI_READ` **再包含一次事件头**（为了让 `TRACE_EVENT` 能展开成两套代码），而我的 `static inline` 助手写在那个多读保护块里面，于是被定义两次。正确做法是把辅助声明放在多读保护块之外，并单独加一个普通 include guard：

```c
#include <linux/atomic.h>
#include <linux/tracepoint.h>

/* 助手放在多读块之外，否则会被 define_trace.h 的二次包含重定义 */
#ifndef RINGBUF_TRACE_HELPERS
#define RINGBUF_TRACE_HELPERS
extern atomic64_t ringbuf_req_seq;
static inline u64 ringbuf_next_req_id(void) { ... }
#endif

#if !defined(_TRACE_RINGBUF_H) || defined(TRACE_HEADER_MULTI_READ)
#define _TRACE_RINGBUF_H
TRACE_EVENT(...)
#endif

#undef TRACE_INCLUDE_PATH
#define TRACE_INCLUDE_PATH .
#undef TRACE_INCLUDE_FILE
#define TRACE_INCLUDE_FILE ringbuf_trace
#include <trace/define_trace.h>
```

配套还要给 Makefile 加 `ccflags-y += -I$(src)`：`define_trace.h` 最后会以 `<TRACE_INCLUDE_PATH/TRACE_INCLUDE_FILE>`（尖括号形式，只搜 `-I` 路径）再包含一次事件头，不加这一行在外部模块里会找不到文件。

- [ ] **内核升级后必须按新版本重建**：本次实验期间 WSL 内核由 `6.18.33.2` 更新为 `6.18.40.1`。由于 §5.4，模块**可能**带着旧版本串也能装载，但 `Module.symvers` 与内核头文件仍应按新树重建才可靠；`ls -d ~/kernel-*` 可以确认当前存在哪些版本的源码树。

> 记法：`Invalid module format` 先看 `dmesg`——`version magic` 找版本、`this_module section size` 找结构体 ABI、`disagrees about version of symbol` 找符号 CRC；看到 `-dirty` 不必惊慌（§5.4）。计数对不上账时先怀疑三件事：用错了 API 返回值（§5.5）、把累计值当增量、跑的是旧二进制（§5.6）。

## 6. 一键复现

> 覆盖：如何一条命令跑完全部测试与观测并落下可交付的原始日志。

- [ ] **前置**：驱动已加载、设备可写、观测接口存在（重载模块后才会出现）。

```bash
cd ~/ringbuf-driver
sudo rmmod ringbuf 2>/dev/null; sudo insmod ringbuf.ko   # 换过驱动代码后必须重载
sudo chmod 666 /dev/ringbuf     # device_create 默认 0600 root:root，非 root 会 EACCES
cat /proc/ringbuf               # 状态快照 + 累计计数
```

- [ ] **一条命令跑完并落盘**：`scripts/run-tests.sh` 会先采集环境与 ABI 信息，**每次重编**测试程序，再按 smoke → poll → proc → stats → bench → aio 的顺序执行，全部输出同时打印到终端并写入 `test-results/raw-<时间戳>.txt`。

```bash
bash scripts/run-tests.sh 64     # 参数 = 每组 MiB 数，默认 64；求稳用 1024
```

产物字段：内核完整版本与启动时间、内核源码树列表、模块内嵌 `vermagic`、`this_module` 段尺寸、`module_layout` 的 CRC、运行内核 BTF 的 `sizeof(struct module)` 对比、`/dev/ringbuf` 与 `/proc/ringbuf` 权限、**当前 `/proc/ringbuf` 完整快照**、以及十二个用例的原始输出与墙钟耗时。

- [ ] **采集事件时间线**（tracefs 只有 root 能写）：脚本会打开 `events/ringbuf/enable`、制造一段流量（aio 用例 + 1 MiB/组的小流量基准）、把原始 trace 落盘，再调用汇总器输出可读结论。

```bash
sudo bash scripts/trace-ringbuf.sh               # 只采 ringbuf 事件（推荐先跑）
sudo bash scripts/trace-ringbuf.sh --with-sched  # 额外采 sched 事件，用于判断"等锁还是等调度"
```

- [ ] **退出条件**：十二个用例的退出码均为 0，且除 `ringbuf_bench` 外的十一个各自打印 `PASS`；否则按 §5 的顺序查 `dmesg` 与计数。

```bash
sudo dmesg | tail -5
cat /proc/ringbuf | tail -25    # 计数段
sudo rmmod ringbuf              # 退出时会检查是否仍有未完成的异步请求
```

> 记法：把"跑测试"变成"一条命令 + 一个日志文件"，把"查性能"变成"一条命令 + 一份事件时间线"，报告与复查都只需引用那两个文件。

## 7. 运行时观测：状态快照、累计计数与事件时间线

> 覆盖：`/proc/ringbuf` 与 tracepoint 各提供什么、怎么设计、怎么证明可信、以及这些数据能回答什么问题。

- [ ] **三层观测各有分工**：L1 状态快照（"现在是什么状态"）、L2 累计计数（"发生了多少次、比值多少"）、L3 事件时间线（"谁先谁后、为什么"）。**比值才是信息**——例如"唤醒 274,714 次"没有意义，"每次通知遇上 0.00002 个等待者"才说明当前负载下没有唤醒放大可省。

### 7.1 状态快照与累计计数（`/proc/ringbuf`）

- [ ] **接口形态**：只读（0444），用 `seq_file` + `struct proc_ops` 实现（内核 5.6 起 procfs 用 `proc_ops` 取代 `file_operations`）。状态字段在同一把锁下一起取，所以同一份快照内部自洽；计数是各 CPU 之和，属"读取瞬间的近似值"。
- [ ] **8 项状态**：`capacity`/`used`/`free`、`read_pos`/`write_pos`、`async_readers`/`async_writers`（**内核态挂起队列长度**）、`fasync`。
- [ ] **24 项计数按"谁服务的"三分区，互不重叠**：

| 分区 | 字段 | 含义 |
|---|---|---|
| 同步路径 | `read_ops`、`write_ops` | `.read`/`.write` 成功返回的次数 |
| 异步快路径 | `direct_serve_read`、`direct_serve_write` | 提交时缓冲已就绪，锁内直接拷贝 |
| 挂起后完成 | `park_read`、`park_write`、`completions_read`、`completions_write` | 挂起与随后被反方向操作完成 |

另有：吞吐（`bytes_read`/`bytes_written`）、完成质量（`partial_completions`/`completion_bytes`）、通知（`read_wakeups`/`write_wakeups`/`woken_tasks`）、同步等待（`sync_sleep_read`/`write`）、失败与边界（`eagain`/`nowait_attempt`/`nowait_success`/`poll_calls`/`fault`/`cancel_on_release`/`park_alloc_fail`）。

- [ ] **热路径用 per-CPU 计数**：`atomic64_t` 会让 18 个核争用同一缓存行，统计本身就成为新瓶颈。`this_cpu_inc/add` 是单指令；即使执行期间被抢占、结果计入了另一个 CPU 的副本也无害——读取侧只关心所有 CPU 之和。汇总放在 `ringbuf_stats_read()`，按 u64 数组逐项累加（结构体只由 u64 组成且有 `BUILD_BUG_ON` 保证无填充），以后增删字段不必改汇总代码。
- [ ] **计数必须能对账，否则不可信**：`tests/ringbuf_stats_test.c` 用**增量断言**逐项核对"已知操作 → 计数恰好增加已知量"，并包含两条跨字段不变量：无等待者时 `woken_tasks` 增量为 0；**缓冲区为空时 `bytes_read` 必须等于 `bytes_written`**（数据只会被读走，不会被丢弃）。后者把"状态快照"和"累计计数"绑在一起检查，单看任何一侧都发现不了问题。
- [ ] **累计计数的三条使用纪律**：①它们是**自装载以来**的值，断言与判断都要用增量；②跨 CPU 求和是**近似**，不要用它做精确的因果判断；③它**不能替代时间线**——计数回答"多少、多快"，回答不了"谁先谁后、为什么"。

### 7.2 事件时间线（tracepoint）

- [ ] **5 个事件**（字段统一成 `key=value`，既好读又好解析）：

| 事件 | 字段 | 用途 |
|---|---|---|
| `ringbuf_serve` | `rw`、`bytes` | 快路径（同步或提交时缓冲已就绪）的一次成功搬运 |
| `ringbuf_park` | `rw`、`want`、`parked`、`id` | 请求被挂起；`parked` 是挂起队列长度 |
| `ringbuf_complete` | `rw`、`res`、`waited_ns`、`id` | 挂起请求被完成；`id` 与 park 配对，`waited_ns` 是排队时长 |
| `ringbuf_wake` | `waitq`、`waiters`、`readable`、`writable` | 一次通知及当时的等待者数 |
| `ringbuf_sync_wait` | `rw`、`waited_ns` | 同步路径真正睡下去 |

- [ ] **为什么不用 `printk` 做时间线**：本驱动在持锁路径上，`printk` 到慢速终端会把锁持有时间放大几个数量级，正好污染被观测对象；tracepoint 未启用时开销接近零（static branch），启用后只写 per-CPU 的 trace 环形缓冲。
- [ ] **采集与汇总**：

```bash
sudo bash scripts/trace-ringbuf.sh               # 只采 ringbuf 事件
sudo bash scripts/trace-ringbuf.sh --with-sched  # 额外采调度事件
```

脚本把原始 trace 存到 `test-results/trace-<时间戳>.txt`（约 1 MB，**不入版本库**，可一条命令重采），再调用 `scripts/trace-summary.py` 输出六段：事件计数、park/complete 配对、排队时长分位与最慢 5 个请求、**有等待者的通知**、同步睡眠分布、最慢请求附近 ±2 ms 的时间线片段。
- [ ] **汇总器可离线验证**：`scripts/trace-summary.py` 只依赖标准库、与数据来源无关，因此用一份合成 trace（`test-results/trace-sample.txt`）就能在任意机器上验证解析逻辑——本次就是在采集之前先把它跑通的。

### 7.3 时间线实测：深度买到的是排队，不是并行

一次真实采集（`ringbuf_bench 1`，小流量窗口）共 **8788 条事件**：

| 观测 | 值 |
|---|---|
| 事件计数 | `serve` 4293、`wake` 4293、`park` 97、`complete` 97、`sync_wait` 8 |
| park/complete 配对 | **97 / 97 全部按 `id` 配上**（说明 trace 缓冲没有被冲掉） |
| 排队时长 | p50 446 µs、p90 795 µs、p99 1.256 ms、max 1.342 ms |
| 有等待者的通知 | 7 / 4293 = **0.16%** |

最慢请求附近的 ±2 ms 时间线片段（节选）：

```text
240272.595513 CPU16  ringbuf_park:     rw=read id=79 want=65536 parked=1
   ...（8 µs 内连续挂起 16 个 64 KiB 请求，parked 从 1 涨到 16）...
240272.595521 CPU16  ringbuf_park:     rw=read id=94 want=65536 parked=16
240272.595555 CPU0   ringbuf_serve:    rw=write bytes=65536
240272.595633 CPU0   ringbuf_complete: rw=read id=79 res=65536 waited_ns=127789
240272.595657 CPU0   ringbuf_serve:    rw=write bytes=65536
240272.595732 CPU0   ringbuf_complete: rw=read id=80 res=65536 waited_ns=229278
   ...（每个请求约 75 µs 完成一个，waited_ns 单调增长）...
240272.596797 CPU0   ringbuf_complete: rw=read id=94 res=65536 waited_ns=1341952
```

读法：16 个请求在 **8 µs 内全部挂起**，但对面（写者）**一次只完成一个**，每个约 **75 µs**；排队时长随队列位置线性增长（第 1 个 128 µs → 第 16 个 1342 µs）。这条时间线直接解释了 §4 里"深度越大越慢"：**深度买到的是排队而不是并行**，吞吐由对面单线程的服务速率决定；块越大单次服务越久，于是"深度 32 + 1 MiB"那一格被彻底拉爆。

- [ ] **时间线与计数的分工**：计数已经能告诉我们"挂起 97 次、完成 97 次、快路径命中 88.7%"；只有时间线能告诉我们"这 97 次是怎么排队的、每个花了多久、由谁在哪个 CPU 上完成"。**先加计数、再按需加时间线**是成本合理的顺序。
- [ ] **时间线的边界**：trace 环形缓冲会被写满覆盖（默认每 CPU 约 1.4 MB），因此它只适合**短窗口、高信息密度**的观察，长期统计仍归计数；要判断"等锁还是等调度"需额外采 `sched_switch`/`sched_wakeup`（`--with-sched`），代价是高频 sched 事件会更快冲掉 ringbuf 事件。

> 记法：先问"看到这个数我会怎么做"，答不上来的计数就别加；每个计数都要配一个分母（比值），并有一条对账断言。计数与时间线不要互相替代——**计数看频率与比值，时间线看顺序与因果**。

---

## 附录 A 速查表

| 目的 | 命令 | 详见 |
|---|---|---|
| 编译驱动与测试程序 | `make && make test` | §6 |
| 加载并放开权限 | `sudo insmod ringbuf.ko && sudo chmod 666 /dev/ringbuf` | §6 |
| 看状态快照 + 计数 | `cat /proc/ringbuf` | §7.1 |
| 跑全部测试并落盘 | `bash scripts/run-tests.sh 64` | §6 |
| 采事件时间线 | `sudo bash scripts/trace-ringbuf.sh` | §7.2 |
| 汇总已有 trace | `python3 scripts/trace-summary.py <trace 文件>` | §7.2 |
| 单跑数据通路 / 回绕 | `./tests/ringbuf_smoke_test` | §2 |
| 单跑 poll / epoll | `./tests/ringbuf_poll_test` | §2 |
| 单跑状态字段校验 | `./tests/ringbuf_proc_test` | §2 |
| 单跑计数对账 | `./tests/ringbuf_stats_test` | §7.1 |
| 单跑真异步验证 | `./tests/ringbuf_aio_test` | §2 |
| 单跑接口契约 / `-EFAULT` | `./tests/ringbuf_api_test` | §2 |
| 单跑阻塞唤醒与信号打断 | `./tests/ringbuf_sync_test` | §2 |
| 单跑异步流程（顺序 / 部分完成 / NOWAIT） | `./tests/ringbuf_aio_flow_test` | §2 |
| 单跑 `close()`/`release()` 关系 | `./tests/ringbuf_release_test` | §2 |
| 单跑 io_uring 路径 | `./tests/ringbuf_uring_test` | §2 |
| 单跑多消费者 / 多生产者 | `./tests/ringbuf_multi_test` | §2 |
| 单跑吞吐基准 | `./tests/ringbuf_bench 64` | §3 |
| 查装载失败原因 | `sudo dmesg \| tail -3` | §5 |
| 量 `sizeof(struct module)` | `python3 scripts/kernel_struct_size.py module` | §5.2 |
| 编译期模块元信息 | `strings ringbuf.ko \| grep -E '^(vermagic\|name)='` | §1 |
| 卸载 | `sudo rmmod ringbuf` | §6 |

## 附录 B 学习路径与进度表

| 阶段 | 对应章节 | 内容 | 目标 | 状态 |
|---|---|---|---|---|
| 1 | 1 | 被测对象与测试环境 | 说清驱动接口清单、best-effort 语义与实测环境 | ☐ |
| 2 | 2 | 测试用例设计 | 知道十二个用例各自断言什么、哪些 I/O 模型没覆盖 | ☐ |
| 3 | 3 | 实测结果 | 拿到可引用进报告的原始输出与吞吐表（含离散度） | ☐ |
| 4 | 4 | 结果解读 | 能解释四条曲线、深度退化的机制与计数画像 | ☐ |
| 5 | 5 | 构建、装载与观测的易错点 | 遇 `Invalid module format` 能用 dmesg 定位；计数/tracepoint 踩坑能自查 | ☐ |
| 6 | 6 | 一键复现 | 一条命令跑完十二个用例与一次时间线采集 | ☐ |
| 7 | 7 | 运行时观测 | 说清三层观测的分工、对账方法、使用纪律与时间线读法 | ☐ |

> 时间紧的读法：**核心速览 → 第 3 节 → 第 4 节 → 附录 A**。

## 附录 C 当前缺口

- **`SIGIO` 实际收信未测（唯一剩下的用户态覆盖点）**：现在只验证 proc 里的注册状态与内核队列变化，没有断言"真的收到信号"。设计要点：`F_SETOWN` + `O_ASYNC` + `sigaction`，写入后用 `sigtimedwait` 断言收到 `SIGIO` 且 `si_fd` 正确；`close()` 之后必须收不到；顺带实测"普通 `SIGIO` 不携带读/写方向信息"。**注意**：不忽略 `SIGIO` 会让测试进程被默认动作杀掉（写 release 用例时当场踩到）。
- **挂起请求缺少取消钩子（真实风险，未修）**：驱动 park 时只保存 `kiocb` 裸指针，没有调用 `kiocb_set_cancel_fn()`。若请求仍挂起时 AIO 上下文被销毁（`io_destroy` 或**进程直接退出**），内核 `free_ioctx_users()` 会调用 `req->ki_cancel()`——对普通读写请求该指针**默认为 NULL**（`fs/aio.c` 里只有 `IOCB_CMD_POLL` 有默认取消函数），于是自旋锁 + 关中断上下文里发生空指针调用 → 内核 oops；即使不崩，链表里也留着已释放的 `kiocb`，之后任何一次完成都会拿悬空指针做函数指针调用。验证方式（**必须修复后**才能跑）：子进程 park 一个请求后直接退出，父进程观察新增的 teardown 计数增长且内核不崩。修法约束与设计要点见驱动仓库 README 的待补清单。
- **公平性没有机制、只有观察**：驱动是"唤醒全部 + 先进先得"，没有排他等待者、没有轮转，公平性完全交给调度器。`ringbuf_multi_test` 的分布：1×3 时 640/666/740（≈1.16:1），2×2 时跨轮次在 ≈1.6:1 ~ ≈2.5:1 之间波动（无饿死）。若要提供公平性承诺，需要引入 `wake_up_interruptible_nr` 批量唤醒 + 排他等待者，并验证不跳过队尾的 `poll` 订阅者。
- **通知合并（水位线）与老化**：计数已经说明当前负载下几乎没有唤醒放大可省（274,714 次通知仅遇 6 人次等待者；时间线窗口里只有 0.16% 的通知遇上等待者），因此收益必须用"多消费者 + 小块写"验证——`ringbuf_multi_test` 已经提供该拓扑与无水位线基线（分布、总通知次数可从计数取）；届时还要处理"被抑制的通知不能无限期丢失"，即尾块老化（`delayed_work`，开启水位线后亚水位线数据的唤醒延迟上限等于它）。
- **多线程（pthread）拓扑未测**：目前所有并发用例要么 `fork` + 各自 `open()`（`ringbuf_sync_test`、`ringbuf_multi_test`），要么 `fork` 继承同一个 `struct file`（`bench`）。同进程多线程共享同一 `struct file` 的拓扑没测过，`fasync` 的 per-file 语义在多线程下也值得一看。
- **时间线的调度归因未做**：只采了 ringbuf 事件，"排队是被调度还是被锁拖住"还没答案。用 `sudo bash scripts/trace-ringbuf.sh --with-sched` 复采，把 `park→complete` 与 `sched_switch`/`sched_wakeup` 对齐；必要时再加 `mutex_lock`/`contention_*` 事件看锁等待。
- **深度退化的隔离实验**：主机制已由时间线给出（对面单线程逐个服务、每请求约 75 µs），但"工作集超 cache""部分完成的碎片请求""调度抖动"三个次要因素仍未逐条隔离。建议固定 1 MiB 块，单独扫深度 1/2/4/8/16/32，把服务时长（相邻 `serve`/`complete` 的时间差）与排队时长分开看。
- **计数自身的开销未分离**：`this_cpu_inc` 是单指令，理论上开销在纳秒级，但无法从 10 ~ 78% 的运行间离散度中分离出来。要回答"计数是否真的无影响"，需要一个 `static_branch` 统计开关做 A/B。
- **procfs 只做只读**（刻意设计）：如果后续要暴露"重置统计""强制唤醒"之类的操作，应先考虑 `ioctl`，而不是给 procfs 加写接口。
- **异步机制原理**：`kiocb` 生命周期、`-EIOCBQUEUED` 契约、取消钩子、`fasync` 与 `io_uring` 的关系，见 `char-driver-async-io.md`（源码级整理，实测数据以本文为准）。

### 已补齐：当初列在缺口里、现在有测试与结论的项

| 原缺口 | 现在由谁覆盖 | 结论 |
|---|---|---|
| 缓冲区满时的非阻塞写 `-EAGAIN` | `ringbuf_api_test` [4] | 返回 `-EAGAIN` 且 `used` 不变 |
| 阻塞 `read`/`write` 的跨进程唤醒断言 | `ringbuf_sync_test` [1][2] | 200 / 201 ms 唤醒，且必须睡够父进程延时 |
| 异步写方向无断言 | `ringbuf_aio_flow_test` [6] | `park_write +4`、按提交顺序完成、级联内容一致 |
| 多请求完成顺序无断言 | `ringbuf_aio_flow_test` [1][2] | 完成顺序 = 提交顺序（按 `aio_data` 判定） |
| `io_uring` 是否走同一条异步路径 | `ringbuf_uring_test` | **不走**：首次 NOWAIT 尝试失败后挂到 `.poll` 上、就绪后重试走同步核 |
| `release()` 取消路径是否可达 | `ringbuf_release_test` | **不可达**（`fasync` 探针证明 release 未执行 + 完成时链表已空） |
| 多进程独立 `open` 的共享流拓扑 | `ringbuf_multi_test` | 三种拓扑不丢不重、同生产者内顺序严格递增；分布见上 |
| `count == 0` / 超容量 / `-ESPIPE` / `-EFAULT` | `ringbuf_api_test` | 全部符合契约；其中 `count == 0` 曾暴露真缺陷并已修 |
