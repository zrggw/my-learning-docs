# 4 MiB 环形缓冲区字符设备驱动测试报告（WSL2 实测）

> 用途：给"环形缓冲字符设备驱动"这份实验一个可交付的结论——六个用例各测什么、实测数字是多少、数字背后的原因、以及从零复现与排错的完整路径。面向**已写出驱动、需要证明它对且量化它快不快**的读者。
> 使用方式：每学完一个小点，把 `[ ]` 改成 `[x]`；需要展开某节可随时让我补充。
> 定位：回答"驱动写完了怎么验证"这一层，从"能编译、能 insmod"到"有实测数据与运行时观测可以写进报告"。
> 背景贴合：异步读写在内核里怎么走（`read_iter`/`write_iter` 契约、`kiocb` 生命周期、`fasync`/`io_uring`）见 `char-driver-async-io.md`；那篇标注"内核模块代码未在本机实测"，本文是同一个驱动的真机实测结果，两篇互补。
> 前提：WSL2，运行内核 **6.18.40.1-microsoft-standard-WSL2**，自建同版本内核源码树（WSL 不提供匹配的 `linux-headers`）；按 §5.2 给源码树打 ABI 补丁后，模块的 vermagic 会带 `-dirty` 后缀，这是正常现象（原因见 §5.4）。文中命令均在 WSL 内执行，硬件为 18 逻辑核 / 15.4 GiB 内存。

---

## 核心速览（先看这一页，5 分钟覆盖 80% 内容）

**一句话结论**：六个用例全部通过——数据通路（含 4 MiB 回绕）、`poll`/`epoll` 就绪通知、`/proc/ringbuf` 状态与计数、吞吐基准、驱动内真异步都符合设计预期；**异步吞吐与同步同量级**（内存环形缓冲没有真实 I/O 延迟可隐藏），而**并发深度增大反而急剧变慢**（1 MiB 块下深度 1 → 32 掉到 1/27）。累计计数进一步给出两个反直觉但重要的事实：**异步提交 88.7% 走快路径根本没挂起**，**274,714 次通知只遇上了 6 人次等待者**。

### ① 关键数字（每组 64 MiB，五次重复的区间）

| 模式 | 峰值吞吐 | 峰值出现在 | 最低点 |
|---|---|---|---|
| 同步流水线（双进程） | 9.7 ~ 13.1 GB/s | 64 ~ 256 KiB | 4 KiB 仅 2.0 ~ 3.3 GB/s |
| 单进程乒乓 | 15.2 ~ 19.3 GB/s | 256 KiB | 4 KiB 约 3.2 ~ 5.3 GB/s |
| 异步读（原生 AIO） | 9.9 ~ 11.7 GB/s | 64 KiB，深度 1 | **1 MiB × 深度 32 仅 0.3 ~ 1.9 GB/s** |
| 异步写（原生 AIO） | 6.9 ~ 11.1 GB/s | 64 KiB ~ 1 MiB，深度 1 | 4 KiB 仅 1.7 ~ 2.9 GB/s |

### ② 运行时画像（累计计数，取自"含一轮完整 bench"的快照，1.9 GB 流量）

| 指标 | 值 | 含义 |
|---|---|---|
| 异步读**快路径命中率** | `direct_serve_read / (direct_serve_read + park_read)` = 46487 / 52420 = **88.7%** | 异步提交绝大多数根本不需要挂起 |
| 异步写快路径命中率 | 52378 / 52416 = **99.93%** | 几乎从不因"缓冲满"而挂起 |
| **部分完成率** | `partial_completions / completions` = 4 / 5971 = **0.07%** | best-effort 契约在实际负载下极少触发 |
| 异步承载占比 | `completion_bytes / bytes_written` = 273 MB / 1904 MB = 14.4% | 异步路径承担的数据量 |
| 通知频率 | 274,714 次 / 1904 MB ≈ **每 6.9 KB 一次** | 与块大小同量级 |
| 唤醒人次 | `woken_tasks` = **6**（该快照） | 与 274,714 次通知相比，**绝大多数通知发生在空队列上** |

### ③ 复现命令（三条）

```bash
sudo insmod ~/ringbuf-driver/ringbuf.ko && sudo chmod 666 /dev/ringbuf
cd ~/ringbuf-driver && make test
bash scripts/run-tests.sh 64      # 采集环境 + 每次重编测试程序 + 顺序跑六个用例，落盘到 test-results/
```

### ④ 五个易错点（前三个本次实测踩到，详见 §5）

| 现象（`insmod` 统一报 `Invalid module format`） | 根因 | 处理 |
|---|---|---|
| dmesg 显示 `version magic '...' should be '...'` | `scripts/setlocalversion` 只认名为 `v<KERNELVERSION>` 的附注 tag，仓库 tag 命名不同就追加 `+` | 补附注 tag，重新生成版本头文件（§5.1） |
| dmesg 显示 `.gnu.linkonce.this_module section size must match ...` | 关掉 `DEBUG_INFO_BTF` 后 `struct module` 少了受 `CONFIG_DEBUG_INFO_BTF_MODULES` 保护的字段 | 该 `#ifdef` 改为无条件编译（§5.2） |
| dmesg 显示 `disagrees about version of symbol module_layout` | `struct module` 改了但符号表未重算，`module_layout` 的 CRC 仍是旧值 | 重算符号表 CRC（§5.3） |
| `vermagic` 带 `-dirty`，看着像不匹配 | 打过 ABI 补丁的内核树有未提交改动 | **无需处理**：模块带 `__versions` 段时内核只比较第一个空格之后的部分（§5.4） |
| 自己写的计数器恒为 0，或断言按旧结果失败 | `wake_up_interruptible()` 的返回值只统计**排他**等待者；累计计数不能当"本次用例的值"；只检查二进制是否存在会导致跑旧程序 | 自己数等待队列；断言一律用增量；脚本每次重编（§5.5、§5.6） |

### ⑤ 阅读路线

只求跑通 → §6；要看数据 → §3；要理解数字为何如此 → §4；装不上/对不上账 → §5；想知道运行时能看到什么 → §7。

---

## 1. 被测对象与测试环境

> 覆盖：这份报告测的是哪个驱动、它提供哪些接口、在什么环境下测。不讲异步机制原理（见 `char-driver-async-io.md`）。

- [ ] **被测驱动是一份 4 MiB 环形缓冲区字符设备**：`vzalloc` 分配缓冲区，`alloc_chrdev_region` + `cdev_add` 注册，`device_create` 生成 `/dev/ringbuf`，`proc_create` 生成只读观测接口 `/proc/ringbuf`。

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

- [ ] **读写都是"尽力而为"（best-effort）语义**：返回值是**实际搬运的字节数**，可以小于 `count`；调用方必须循环直到搬完。异步请求同理，完成事件里的 `res` 可能小于申请长度。
- [ ] **环形状态只有三个量**：`read_pos`、`write_pos`、`data_len`；空 = `data_len == 0`，满 = `data_len == RINGBUF_BUFFER_SIZE`。位置在拷贝时回绕，不靠预留槽位区分空满。
- [ ] **并发保护用 `mutex` + 两个等待队列**（`read_wait`、`write_wait`），不用自旋锁——因为拷贝走 `copy_to_iter`/`copy_from_iter`，可能睡眠。
- [ ] **观测分两层**：L1 是锁内一起取的瞬时快照（同一份快照内部自洽），L2 是 per-CPU 累计计数（跨 CPU 求和，属读取瞬间的近似值）。两者不混用、不互推。

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

> 记法：装载要连过几道关——vermagic（带 CRC 时只比标志位，见 §5.4）、`struct module` 段尺寸、`module_layout` 符号 CRC；任何一道不过，`insmod` 都只说"Invalid module format"，必须看 `dmesg` 才知道是哪一道。

## 2. 测试用例设计

> 覆盖：六个测试程序各自验证什么、覆盖哪些 I/O 模型、哪些没覆盖。

- [ ] **`tests/ringbuf_smoke_test.c` 测数据通路与回绕**（单进程，`O_NONBLOCK`）：空缓冲区读必须 `-EAGAIN`；写 `SIZE-1000` 把 `write_pos` 推到接近末端；读走 1000；再写 2000 使其**跨越 4 MiB 边界**；循环读回全部 4 MiB，按 `gen(i) = (i*31+7) & 0xff` 校验整条字节流。
- [ ] **`tests/ringbuf_poll_test.c` 测 poll/epoll 就绪通知**：空缓冲 `poll(POLLIN)` 必须超时、`poll(POLLOUT)` 立即就绪；写满后 `POLLOUT` 必须超时；`fork` 出子进程**阻塞在 `poll()` 上**，父进程写入后子进程必须被唤醒（并把等待时长回报父进程校验，以排除"靠超时返回"的假通过）；epoll **水平触发**在数据未读走时反复报告，**边沿触发**只投递一次、新数据到来才再报。
- [ ] **`tests/ringbuf_proc_test.c` 测状态快照字段**：读走/写入后 `used`、`free`、`read_pos`、`write_pos` 必须与操作量严格对应（位置按模 4 MiB 前移）；写满后 `free=0`；**提交一个会被挂起的 AIO 读后 `async_readers=1`，被写入完成后归零**；`fasync` 随 `O_ASYNC` 注册/注销变化；以 `O_WRONLY` 打开必须失败。
- [ ] **`tests/ringbuf_stats_test.c` 测累计计数对账**：不是"字段存在"，而是**增量等于已知操作量**——写 100000 字节 → `bytes_written +100000`、`write_ops +1`；读 40000 → `bytes_read +40000`、`read_ops +1`；挂起 64 字节的 AIO 读再用 16 字节写入完成它 → `park_read +1`、`completions_read +1`、`partial_completions +1`（申请 64 得 16）、`completion_bytes +16`；空缓冲非阻塞读 → `eagain +1`；`poll()` 一次 → `poll_calls +1`。另有两条跨字段不变量：无等待者时 `woken_tasks` 增量必须为 0；**缓冲区为空时 `bytes_read` 必须等于 `bytes_written`**。
- [ ] **`tests/ringbuf_aio_test.c` 测驱动内真异步**（单进程，原生 AIO）：缓冲区为空时 `io_submit` 一个 16 字节异步读，必须**立即返回**；带零超时的 `io_getevents` 必须返回 **0 个事件**（证明请求被挂起而非同步完成）；随后同步 `write` 16 字节反向完成它，`io_getevents` 拿到 `res=16`、`data=0x1234`。
- [ ] **`tests/ringbuf_bench.c` 测吞吐量**，共四组：同步流水线（`fork` 出生产者/消费者，双方阻塞同步）、单进程乒乓（写完一块立刻读回）、异步读、异步写（父进程 AIO 保持 `depth` 个在途请求，子进程做同步对端）。
- [ ] **覆盖矩阵**（本次实测的能力边界）：

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

- [ ] **测试顺序不能调换**：`ringbuf_aio_test` 要求缓冲区初始为空，因此固定为 smoke → poll → proc → stats → bench → aio；前四者结束时都会把缓冲区清空。
- [ ] **关于"多进程而非多线程"**：`fork` 之后父子**共享同一个 `struct file`**（子进程继承的是已打开的 fd，不是各自 `open`）。好处是父进程挂起的异步请求能被子进程的同步操作完成、`poll` 的唤醒路径也跨了任务边界；副作用是 `release()` 里的清理路径（摘 `fasync`、取消挂起请求）要等父子都关闭 fd 才会执行。

> 记法：断言"真异步"的最小证据组合 = `io_submit` 立即返回 + 零超时查询到 0 个完成事件 + 反方向操作后拿到完成事件；断言"边沿触发"的最小证据 = 不读走数据时第二次 `epoll_wait` 返回 0。

## 3. 实测结果

> 覆盖：六个用例的原始输出与吞吐表，不解释原因（见 §4）。吞吐数据来自 `test-results/raw-20261011-185132.txt`（当前代码，含计数）；计数画像取自含完整 bench 的快照。

- [ ] **六个用例全部通过**，退出码均为 0：

| 用例 | 结果 | 墙钟 |
|---|---|---|
| ① `ringbuf_smoke_test` | PASS | 0.023 s |
| ② `ringbuf_poll_test` | PASS | 1.419 s（含 200 ms 唤醒等待与多次 200 ms 超时判定） |
| ③ `ringbuf_proc_test` | PASS | 0.085 s |
| ④ `ringbuf_stats_test` | PASS | 0.066 s |
| ⑤ `ringbuf_bench 64` | 退出码 0 | 1.485 s |
| ⑥ `ringbuf_aio_test` | PASS | 0.048 s |

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

> 覆盖：四组表的形状说明什么、为什么异步不比同步快、为什么深度增大反而变慢、计数画像告诉我们什么，以及本次测量的局限。原因分析中标注"推断"者为待验证假设。

- [ ] **同步流水线：4 KiB 到 256 KiB 上升，4 MiB 回落。** 4 KiB 只有约 2.0 ~ 3.3 GB/s，瓶颈是系统调用与拷贝次数（每 MiB 需要 256 次读写）；块增大到 64 ~ 256 KiB 后接近内存带宽上限（约 10 ~ 13 GB/s）；4 MiB 反而降到约 3 ~ 6 GB/s，因为单次拷贝 4 MiB 已超出各级 cache 的工作集，且两个进程几乎无法重叠搬运。
- [ ] **单进程乒乓在中等块下最高（约 15 ~ 19 GB/s），属于预期结果。** 没有第二个进程的调度与同步开销，写完立刻读回时数据仍在 L2/L3 里，所以它测的是"缓存内往返上限"，不代表真实并发场景的吞吐。它在 4 MiB 处常反超流水线（例如 9246.7 对 3087.9），正说明流水线的瓶颈是"两个进程争用拷贝带宽"而不是设备本身。
- [ ] **异步吞吐与同步同量级，符合设计预期。** 缓冲区在内存里，`ki_complete` 只是在另一个上下文里做一次 `memcpy`，没有真实 I/O 延迟可以隐藏，异步只额外付出 `io_submit` + `io_getevents` 两次系统调用的成本。因此"异步没比同步快"不是缺陷，而是这个场景的必然结果。
- [ ] **计数画像纠正了一个直觉：异步请求绝大多数根本没挂起。** 异步读快路径命中率 88.7%、异步写 99.93%——因为父进程提交请求时缓冲区通常已经就绪，"挂起"只是少数派路径。所以评估异步实现时，**快路径的实现质量与挂起路径同等重要**；只测"会不会挂起"会漏掉 88% 的实际流量。
- [ ] **best-effort 语义在实际负载下几乎不触发**：部分完成率 0.07%（4 / 5971）。但它是 API 契约的一部分，必须实现并测试——这次是靠"申请 64 字节、只写入 16 字节"这个刻意构造的用例才把它逼出来。
- [ ] **通知绝大多数发生在空队列上**：274,714 次通知只遇到 6 人次等待者。这说明当前负载（单生产者单消费者、块 ≥ 4 KiB）下**几乎没有唤醒放大可省**——水位线（通知合并）的收益要等到"多个消费者同时阻塞 + 小块写"的拓扑才会显现，这也决定了下一步该补什么实验，而不是先写优化。
- [ ] **深度增大反而变慢，且块越大越严重**：三种块下的异步读趋势一致（64 KiB 在深度 8 见顶后回落，1 MiB 从深度 1 起一路暴跌）。1 MiB × 32 这一格五次测量横跨 299.6 ~ 1898.4 MB/s，**极差 6.3 倍**。可能的原因（**推断，尚未逐条隔离验证**）：
  - 环形缓冲只有 4 MiB，而 1 MiB × 32 的在途请求量达 32 MiB，绝大多数请求只能挂在链表上；完成时又在**持有 `mutex`** 的状态下逐个拷贝，锁的持有时间被拉长，与对方进程的读写直接争用；
  - 每个在途请求必须独占用户缓冲区（`depth × block`），深度 32 时工作集 32 MiB 超出 cache，拷贝退化为内存带宽受限；
  - 基准程序的补齐逻辑在"部分完成"时会按剩余量重新提交，深度大时碎片请求更多，进一步放大每次调用的固定开销；
  - 深度大时"提交—挂起—完成—收割"的调度往返次数也随之增加，可能触发调度抖动的正反馈（极端值更像这类抖动，而非稳定态的带宽下降）。
  - 实践结论：**并发深度要与缓冲容量和块大小匹配**，本机 4 MiB 缓冲下 64 KiB 块是异步路径的甜点；把深度当"免费加速按钮"会在 1 MiB 块上付出一个数量级的代价。
- [ ] **本次测量的局限（必须与数字一起引用）**：每组仅 64 MiB，单行耗时落在 3 ms ~ 50 ms，运行间离散度最高达 78%。因此这些数字**只用于判断趋势与量级，不宜作为精确指标**；要得到稳定数据，请把参数放大并重复取中位数：

```bash
bash scripts/run-tests.sh 1024      # 每组 1024 MiB，单行耗时进入百毫秒量级
```

> 记法：内存环缓冲的吞吐看"块大小"（拷贝效率）而非"并发深度"（那是在隐藏真实 I/O 延迟时才起作用）；深度是**受缓冲容量约束**的参数，不是越大越好。

## 5. 构建、装载与观测的易错点（本次实测踩到）

> 覆盖：`insmod: Invalid module format` 的三层根因与修法、`-dirty` vermagic 为何无害、内核升级后要重建，以及两个"自己给自己挖的坑"（唤醒人次计数、测试与断言的卫生）。这是本次实验最花时间的部分。

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

- [ ] **内核升级后必须按新版本重建**：本次实验期间 WSL 内核由 `6.18.33.2` 更新为 `6.18.40.1`。由于 §5.4，模块**可能**带着旧版本串也能装载，但 `Module.symvers` 与内核头文件仍应按新树重建才可靠；`ls -d ~/kernel-*` 可以确认当前存在哪些版本的源码树。

> 记法：`Invalid module format` 先看 `dmesg`——`version magic` 找版本、`this_module section size` 找结构体 ABI、`disagrees about version of symbol` 找符号 CRC；看到 `-dirty` 不必惊慌（§5.4）。计数对不上账时先怀疑三件事：用错了 API 返回值（§5.5）、把累计值当增量、跑的是旧二进制（§5.6）。

## 6. 一键复现

> 覆盖：如何一条命令跑完全部测试并落下可交付的原始日志。

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

产物字段：内核完整版本与启动时间、内核源码树列表、模块内嵌 `vermagic`、`this_module` 段尺寸、`module_layout` 的 CRC、运行内核 BTF 的 `sizeof(struct module)` 对比、`/dev/ringbuf` 与 `/proc/ringbuf` 权限、**当前 `/proc/ringbuf` 完整快照**、以及六个用例的原始输出与墙钟耗时。

- [ ] **退出条件**：六个用例的退出码均为 0，且 smoke / poll / proc / stats / aio 各自打印 `PASS`；否则按 §5 的顺序查 `dmesg` 与计数。

```bash
sudo dmesg | tail -5
cat /proc/ringbuf | tail -25    # 计数段
sudo rmmod ringbuf              # 退出时会检查是否仍有未完成的异步请求
```

> 记法：把"跑测试"变成"一条命令 + 一个日志文件"，报告与复查都只需引用那个文件。

## 7. 运行时观测：状态快照与累计计数

> 覆盖：`/proc/ringbuf` 提供什么、24 项计数怎么设计、怎么证明计数可信、以及这些数字能回答什么问题。不讲事件时间线（尚未实现，见附录 C）。

- [ ] **接口形态**：`/proc/ringbuf` 是**只读**（0444）观测接口，两段——8 项状态快照与 24 项累计计数。用 `seq_file` + `struct proc_ops` 实现（内核 5.6 起 procfs 用 `proc_ops` 取代 `file_operations`），状态字段在同一把锁下一起取，所以同一份快照内部自洽。
- [ ] **为什么观测要分两层**：状态快照回答"现在是什么状态"，累计计数回答"发生了多少次、比值是多少"。**比值才是信息**——例如"唤醒 274714 次"没有意义，"每次唤醒遇上 0.00002 个等待者"才说明当前负载下没有唤醒放大可省。
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

> 记法：先问"看到这个数我会怎么做"，答不上来的计数就别加；每个计数都要配一个分母（比值），并有一条对账断言。

---

## 附录 A 速查表

| 目的 | 命令 | 详见 |
|---|---|---|
| 编译驱动与测试程序 | `make && make test` | §6 |
| 加载并放开权限 | `sudo insmod ringbuf.ko && sudo chmod 666 /dev/ringbuf` | §6 |
| 看状态快照 + 计数 | `cat /proc/ringbuf` | §7 |
| 跑全部测试并落盘 | `bash scripts/run-tests.sh 64` | §6 |
| 单跑数据通路 / 回绕 | `./tests/ringbuf_smoke_test` | §2 |
| 单跑 poll / epoll | `./tests/ringbuf_poll_test` | §2 |
| 单跑状态字段校验 | `./tests/ringbuf_proc_test` | §2 |
| 单跑计数对账 | `./tests/ringbuf_stats_test` | §7 |
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
| 2 | 2 | 测试用例设计 | 知道六个用例各自断言什么、哪些 I/O 模型没覆盖 | ☐ |
| 3 | 3 | 实测结果 | 拿到可引用进报告的原始输出与吞吐表（含离散度） | ☐ |
| 4 | 4 | 结果解读 | 能解释四条曲线的形状、深度退化与计数画像 | ☐ |
| 5 | 5 | 构建、装载与观测的易错点 | 遇 `Invalid module format` 能用 dmesg 定位；计数对不上能自查 | ☐ |
| 6 | 6 | 一键复现 | 一条命令跑完六个用例并产出日志文件 | ☐ |
| 7 | 7 | 运行时观测 | 说清状态快照与累计计数的分工、对账方法与使用纪律 | ☐ |

> 时间紧的读法：**核心速览 → 第 3 节 → 第 4 节 → 附录 A**。

## 附录 C 当前缺口

- **事件时间线**：目前只有计数，没有顺序与因果。计划给 `serve`/`park`/`complete`/`wake`/`sync_wait` 加 tracepoint（`park` 与 `complete` 带请求 `id` 以便配对），与内核已有的 `sched_switch`、`mutex_lock`、`syscalls:*` 放在同一时间轴上；并用一个采集脚本把原始 trace 整理成"等待最久的 N 个请求"。它能证明两条可断言性质：**AIO 完成顺序 = 提交顺序**、**异步完成总是排在同步等待者被调度之前**。
- **水位线（通知合并）与老化**：计数已经说明当前负载下没有唤醒放大可省（274,714 次通知仅遇 6 人次等待者），因此收益必须用**多消费者 + 小块写**的新实验来验证；届时还要处理"被抑制的通知不能无限期丢失"——即尾块老化（`delayed_work`，默认 20 ms，开启水位线后亚水位线数据的唤醒延迟上限等于它）。另需评估是否采用"按可服务批量数唤醒"（`wake_up_interruptible_nr`，需配排他等待者，并验证不跳过队尾的 poll 订阅者）。
- **测量精度**：本报告为每组 64 MiB 的多次重复，运行间离散度最高 78%，异步读 1 MiB × 32 甚至横跨 6.3 倍；建议以 `bash scripts/run-tests.sh 1024` 复测并取中位数。深度退化的四条推断尚未逐条隔离验证——建议的隔离实验是：固定 1 MiB 块，单独扫深度 1/2/4/8/16/32，并记录每个请求从提交到完成的延迟分布。
- **计数自身的开销未分离**：`this_cpu_inc` 是单指令，理论上开销在纳秒级，但本次无法从 10 ~ 78% 的运行间离散度中把它分离出来。要回答"计数是否真的无影响"，需要一个 `static_branch` 统计开关做 A/B（可与水位线的开关复用）。
- **未覆盖的驱动能力**（驱动侧已实现，尚无用例）：缓冲区满时的非阻塞写 `-EAGAIN`；阻塞 `read`/`write` 的跨进程唤醒断言；`fasync` **实际收到 `SIGIO` 信号**（当前只验证 proc 里的注册状态与内核队列变化）；`close()` 之后不再收信号；`io_uring` 走同一条 `.read_iter`/`.write_iter` 路径。
- **未覆盖的并发拓扑**：目前所有并发用例都靠 `fork` 且共享同一个 `struct file`，因此 `release()` 的清理路径（摘 `fasync`、取消该文件挂起的请求）未被有效触发——计数给出了证据：`cancel_on_release`、`fault`、`park_alloc_fail`、`nowait_attempt` 全为 0。也没有多线程（pthread）用例。
- **procfs 只做只读**：没有写侧控制（这是刻意设计，见 §1）；如果后续要暴露"重置统计""强制唤醒"之类的操作，应先考虑 `ioctl` 而不是给 procfs 加写接口。
- **异步机制原理**：`kiocb` 生命周期、`-EIOCBQUEUED` 契约、`fasync` 与 `io_uring` 的关系，见 `char-driver-async-io.md`（该篇为源码级整理，驱动的实测数据以本文为准）。
