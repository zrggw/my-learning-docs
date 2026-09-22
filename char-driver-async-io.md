# 字符设备驱动异步读写实现原理（AIO / io_uring / fasync）

> 用途：从"只会写阻塞 read/write"到"能说清异步读写在内核里怎么走、能自己实现并排错"的路线图 + 要点笔记。面向**写过基础字符设备驱动、想搞清异步机制**的读者。
> 使用方式：每学完一个小点，把 `[ ]` 改成 `[x]`；需要展开某节可随时让我补充。
> 定位：讲清 **"用户态的一次异步提交，究竟在内核里经过了谁、由谁在什么时候报告结果"**，重点是**`read_iter`/`write_iter` 契约**与**`kiocb` 生命周期**。
> 背景贴合：你已有 C 语言与内核模块基础（`c-advanced-learning.md`）；本文补上 VFS 与驱动之间的异步接口这一层。驱动如何被内核发现、如何拿到 IRQ/MMIO 资源，见 `kernel-driver-hardware-interaction.md`。
> 前提：源码引文锚定 **Linux 7.2.6（stable）/ 7.3-rc4（mainline）**；模块编译需与运行内核匹配的头文件；用户态示例需 `libaio` / `liburing`。与老资料差异较大的几处已逐一标注版本（见 1.6）。**本文为 Windows 环境撰写，内核模块代码未在本机实测**，依据是一手源码（见附录 C）。

---

## 核心速览（先看这一页，5 分钟覆盖 80% 内容）

**一句话定位**：字符设备的"异步"由两条独立的路组成——**就绪/通知**（`poll` + `fasync`，驱动只报告状态，不搬运数据）与**完成**（`read_iter`/`write_iter` 返回 `-EIOCBQUEUED`，驱动稍后调 `ki_complete` 报告结果）。绝大多数真实字符设备走前者。

**① 三种异步模型的驱动侧最小写法**

```c
#include <linux/fs.h>
#include <linux/poll.h>
#include <linux/wait.h>
#include <linux/sched/signal.h>
#include <linux/cleanup.h>          /* guard()；无此头文件的旧内核改为 spin_lock_irqsave */

struct my_dev {
	wait_queue_head_t inq;              /* 就绪等待队列 */
	struct fasync_struct *async_queue;  /* SIGIO 通知队列 */
	spinlock_t lock;
	bool data_ready;
};

/* 1) 就绪轮询：先 poll_wait（必须无条件调用），再取快照判状态 */
static __poll_t my_poll(struct file *filp, poll_table *wait)
{
	struct my_dev *dev = filp->private_data;
	__poll_t mask = 0;

	poll_wait(filp, &dev->inq, wait);
	guard(spinlock_irqsave)(&dev->lock);
	if (dev->data_ready)
		mask |= EPOLLIN | EPOLLRDNORM;
	return mask;
}

/* 2) 异步通知：一行转发，锁由 fasync_helper 自己管 */
static int my_fasync(int fd, struct file *filp, int on)
{
	struct my_dev *dev = filp->private_data;

	return fasync_helper(fd, filp, on, &dev->async_queue);
}

/* 3) 声明"本文件支持非阻塞完成"——VFS 不会代你设置 */
static int my_open(struct inode *inode, struct file *filp)
{
	filp->private_data = container_of(inode->i_cdev, struct my_dev, cdev);
	filp->f_mode |= FMODE_NOWAIT;
	return 0;
}
```

数据到达处（中断或工作队列中）统一"叫醒"两侧。**先改状态、再唤醒**，两处判据必须一致：

```c
spin_lock_irqsave(&dev->lock, flags);
dev->data_ready = true;
spin_unlock_irqrestore(&dev->lock, flags);

wake_up_interruptible(&dev->inq);                  /* 唤醒 poll/epoll */
kill_fasync(&dev->async_queue, SIGIO, POLL_IN);    /* 投递 SIGIO；POLL_IN 即 EPOLLIN|EPOLLRDNORM */
```

> 注意：驱动**不要**在 `->release` 里手工清理 fasync 队列——`__fput()` 会替你调用 `->fasync(-1, file, 0)`（见 3.4）。

**② 五个核心概念**

| 概念 | 一句话 |
|---|---|
| **异步 = 发起者与完成者解耦** | 不是"并行"，而是提交之后的完成由另一个上下文（ISR / 工作队列 / 内核任务）报告 |
| **`read_iter`/`write_iter`** | v4.1 起 AIO 与 io_uring 唯一到达驱动的读写入口；老的 `aio_read`/`aio_write` 已删除 |
| **`-EIOCBQUEUED`** | 驱动从 `read_iter` 返回它 = "我已排队，稍后自己回调"；内核从此不再过问这次 I/O |
| **`ki_complete`** | 驱动稍后调用 `iocb->ki_complete(iocb, ret)` 报告最终结果（`ret` 为字节数或负 errno） |
| **`fasync` + `kill_fasync`** | 与数据搬运无关的**单向通知**通道：把"就绪"事件变成 `SIGIO`，可在中断中调用 |

**③ 三种异步模型对照（决定你用哪条路）**

| | 阻塞 + `O_NONBLOCK` | `poll`/epoll + `fasync`/SIGIO | AIO / io_uring |
|---|---|---|---|
| 驱动要实现 | `read`/`write`（自己睡眠） | `poll`、`fasync` | **`read_iter`/`write_iter`** |
| 数据搬运者 | 驱动内联拷贝 | 用户态稍后自己 `read` | 驱动内联拷贝或 DMA |
| 完成报告者 | — | 不报告，只报告"可读/可写" | 驱动调 `ki_complete` |
| 用户态阻塞点 | `read()` 内部 | `poll()`/信号处理 | 无（结果在 ring 里） |
| 真实字符设备 | 常见 | **最主流**（input/tty/FUSE/RDMA/ALSA） | 少见（块设备/文件系统为主，见 1.5） |

**④ 最常用的 5 条命令 / 3 个检查点**

| 命令 | 作用 |
|---|---|
| `fio --ioengine=libaio --rw=read --bs=4k` | 用 AIO 引擎压测，验证 `-EIOCBQUEUED` 路径 |
| `cat /sys/kernel/tracing/trace_marker` 写入 | `tracefs` 打点，配合 `trace_printk` 看驱动时序 |
| `echo 1 > /sys/kernel/tracing/events/io_uring/enable` | 开启 io_uring 事件跟踪 |
| `strace -e trace=io_submit,io_getevents,io_uring_enter` | 从用户态确认提交是否真的异步 |
| `echo 1 > /proc/sys/fs/aio-nr` | 查看 AIO 上下文占用（满则 `io_setup` 返 `EAGAIN`） |

| 检查点 | 期望 |
|---|---|
| `->open` 是否设 `FMODE_NOWAIT` | 未设则 io_uring 只能靠 `->poll` 兜底或退化到 io-wq 阻塞线程 |
| `poll_wait()` 是否无条件调用 | 有提前 `return` 就再也注册不上等待者 |
| 同步调用时是否可能返回 `-EIOCBQUEUED` | 会触发 `BUG_ON(ret == -EIOCBQUEUED)` |

**⑤ 三大高频异常（完整对照表见第 6 节）**

| 现象 | 第一反应 |
|---|---|
| 异步请求永远不完成 | 返回了 `-EIOCBQUEUED` 却没人调 `ki_complete`；或同步路径误返回 |
| 完成时数据错乱 / 内核 oops | `kiocb` 或用户缓冲区在完成前已失效（生命周期，见第 5 节） |
| `io_submit` 报 `EINVAL` | 驱动没实现 `read_iter`（`aio_read()` 直接 `return -EINVAL`） |

**阅读路线**：先分清三条路 → 按需读 2 / 3 / 4 节；要写驱动实现 → 第 4 节 + 第 5 节；出问题 → 第 6 节；查 API → 附录 A；查源码出处 → 附录 C。

---

## 1. 分层定位：一次异步提交经过了谁（重点）

> 覆盖：字符设备特有的 fops 替换时机、异步三层的分工、`-EIOCBQUEUED` 的权威语义、与老资料的版本差异。不讲具体驱动实现（见第 4 节）。

### 1.1 字符设备的 `f_op` 何时确定

- [ ] **字符设备的 `f_op` 在 `open` 时才被替换**：注册的 `cdev` 只让系统认得 inode，`chrdev_open()` 通过 `replace_fops(filp, fops)` 把 `file->f_op` 换成你的驱动表，随后才调用你的 `->open`。所以**任何异步路径最终都落在你自己的 fops 上**。

  ```c
  /* fs/char_dev.c（v7.2） */
  const struct file_operations def_chr_fops = {
  	.open = chrdev_open,
  	.llseek = noop_llseek,
  };
  ```

### 1.2 三层分工

- [ ] **三层分工**：用户态 API 负责"提交并等待完成事件"，VFS 负责"把请求变成 `kiocb` 并保管生命周期"，驱动负责"要么就地完成，要么排队并稍后回调"。

  ```text
  用户态              内核 VFS                     驱动
  ────────────────────────────────────────────────────────────────
  io_submit()   →  fs/aio.c: aio_read()      →  f_op->read_iter(kiocb, iter)
  io_uring_enter() → io_uring/rw.c: io_read() →  f_op->read_iter(kiocb, iter)
  read()        →  fs/read_write.c: new_sync_read()（同步，ki_complete == NULL）
                                  │
                    返回 -EIOCBQUEUED？ ──是──→ VFS 什么都不做，等驱动回调
                                  └──否──→ VFS 立即自己调用 ki_complete(req, ret)
                                                │
  完成事件            aio ring / eventfd  ←─────┘  iocb->ki_complete(iocb, ret)
  完成事件            io_uring CQ ring   ←────────  （在 ISR / 工作队列中调用）
  ```

### 1.3 `-EIOCBQUEUED` 的准确语义

- [ ] **`-EIOCBQUEUED` 的准确含义**（`fs/aio.c` `aio_rw_done()`，v7.2）：

  ```c
  static inline void aio_rw_done(struct kiocb *req, ssize_t ret)
  {
  	switch (ret) {
  	case -EIOCBQUEUED:
  		break;                       /* 内核不再做任何事，等驱动回调 */
  	case -ERESTARTSYS:
  	case -ERESTARTNOINTR:
  	case -ERESTARTNOHAND:
  	case -ERESTART_RESTARTBLOCK:
  		ret = -EINTR;
  		fallthrough;
  	default:
  		req->ki_complete(req, ret);  /* 其它任何返回值，内核立刻自己回调 */
  	}
  }
  ```

  - 它的定义不在 `fs.h`，而在 `include/linux/errno.h`：`#define EIOCBQUEUED 529 /* iocb queued, will get completion event */`（v7.2）。
  - **返回 `-EIOCBQUEUED` 是"我接管了完成"的承诺**；返回其它值（含错误码）都是"这次 I/O 已经结束了"。
  - 术语陷阱：`-EIOCBQUEUED` 是 **`read_iter` 的返回值**；`ki_complete` 返回 `void`，不可能"返回"控制码。

### 1.4 同步调用者与异步调用者的分流

- [ ] **同步调用者绝不接受 `-EIOCBQUEUED`**：`fs/read_write.c` 的同步读路径有硬断言。

  ```c
  ret = filp->f_op->read_iter(&kiocb, &iter);
  BUG_ON(ret == -EIOCBQUEUED);
  ```

  - 判据是 `is_sync_kiocb(kiocb)`，即 `kiocb->ki_complete == NULL`（`init_sync_kiocb()` 不设置它）。
  - 因此一个 `read_iter` 实现必须**同时**能应付异步与同步两种调用者（见 4.2）。

### 1.5 一个反直觉的事实：纯字符设备很少用 `-EIOCBQUEUED`

- [ ] **【纠偏】真实内核里，纯字符设备几乎不用 `-EIOCBQUEUED`**。对约 30 个高概率文件的排查结论：返回 `-EIOCBQUEUED` 的路径集中在**块设备**（`block/fops.c`）、**文件系统 Direct-IO**（`fs/iomap/direct-io.c`、`fs/fuse/file.c`）与 **`io_uring_cmd`**（`drivers/nvme/host/ioctl.c`）。字符设备的主流通行方案是 **`poll` + `fasync`**。
  - 这不矛盾：接口对所有文件类型开放，`-EIOCBQUEUED` 是对字符设备**合法且受支持**的契约；只是"用 DMA 异步搬运数据"的字符设备（如数据采集卡）才会真正用到它。
  - 唯一豁免：io_uring 存在 `->uring_cmd` 通道（`IORING_OP_URING_CMD`），它绕开 `read_iter`，由驱动在命令完成时调 `io_uring_cmd_done()`。本文不展开。

> 记法：**接口是通用的，用法是有偏好的：字符设备先想 `poll`+`fasync`，真要异步搬运数据时才动 `read_iter`+`ki_complete`。**

### 1.6 与老资料差异速查（写代码前先看这张表）

> 记法：**"返回 `-EIOCBQUEUED` = 我把完成权拿走了；返回别的 = 事情已经办完。"**

| 老写法 / 常见说法 | 现状（7.2 / 7.3-rc4） | 依据 |
|---|---|---|
| `->aio_read` / `->aio_write` | **v4.1 起已删除**，改用 `read_iter`/`write_iter` | commit `8436318205b9`（Al Viro, 2015-04-11） |
| `ki_complete(iocb, ret, ret2)` 三参数 | **v5.16 起为两参数** `(iocb, ret)` | commit `6b19b766e8f0`（Jens Axboe, 2021-10-25） |
| `kiocb` 里的 `ki_ctx`/`ki_users`/`ki_retry`/`ki_user_iocb` | 均已不存在；上下文改由 `struct aio_kiocb` + `container_of` 承载 | `include/linux/fs.h`（7.2） |
| `kiocb_set_rw_flags(ki, flags)` | **v6.11 起三参数** `(ki, flags, rw_type)` | commit `c34fc6f26ab8`（2024-06-20） |
| `IOCB_DIO_CALLER_COMP` | v6.6 加入 → v6.7 禁用 → **v6.19 删除**，7.2 中不存在 | commit `f9f85149994d`（2025-11-25） |
| `kill_fasync(..., SIGIO, POLL_IN)` | `POLL_IN` 宏在 7.3-rc4 树内**已不存在**；语义等价于 `EPOLLIN \| EPOLLRDNORM` | `fs/fcntl.c` 的 `band_table[]` 注释 |
| `poll` 返回 `unsigned int` + `POLLIN` | 内核侧类型为 `__poll_t`，写法用 `EPOLLIN` | `include/linux/fs.h` |
| `ioctl(struct inode *, struct file *, ...)` | 早已改为 `unlocked_ioctl(struct file *, unsigned int, unsigned long)` | `include/linux/fs.h` |
| `down()/up(&dev->sem)` | 用 `mutex_lock()/mutex_unlock()` | LDD3 之后再无此惯用法 |

---

## 2. 就绪模型：`poll` / epoll 与等待队列

> 覆盖：`__poll_t` 返回值语义、`poll_wait` 的绑定方式、ERR/HUP 的归属、`O_NONBLOCK` 与 `FMODE_NOWAIT` 的区别。不讲 fasync（见第 3 节）。

### 2.1 `poll` 的职责边界

- [ ] **`poll` 只报告状态，不做任何 I/O**：用户态得到"可读"后仍要自己 `read()`。这也是 epoll 能同时管理上千个 fd 的原因。

### 2.2 `poll_wait` 的约束

- [ ] **`poll_wait()` 必须无条件调用**：它不睡眠、不加锁，只把调用方的 `poll_table_entry` 挂到你的 `wait_queue_head_t` 上；VFS 在"已发现就绪"后会把 `_qproc` 置 NULL，此后再调用就是空操作。

  ```c
  /* include/linux/poll.h（v7.2）：p->_qproc 为空时什么都不做 */
  static inline void poll_wait(struct file *filp, wait_queue_head_t *wait_address,
                               poll_table *p)
  {
  	if (p && p->_qproc) {
  		p->_qproc(filp, wait_address, p);
  		smp_mb();
  	}
  }
  ```

  - 正确顺序：**先 `poll_wait` 注册，再取状态快照**。反过来写会丢事件。

### 2.3 返回值语义

- [ ] **返回值语义与 ERR/HUP 的归属**：

  | 位 | 含义 | 谁负责置位 |
  |---|---|---|
  | `EPOLLIN` \| `EPOLLRDNORM` | 可读 | 驱动（`POLLRDNORM` 在 Linux 上等价 `POLLIN`） |
  | `EPOLLOUT` \| `EPOLLWRNORM` | 可写 | 驱动 |
  | `EPOLLERR` | 设备错误 | 驱动（真的出错时才报） |
  | `EPOLLHUP` | 挂断（对端关闭语义） | 驱动；字符设备通常不需要 |
  | `EPOLLNVAL` | fd 无效 | **VFS**，与驱动无关 |

  - **驱动不必主动返回 `EPOLLERR\|EPOLLHUP`**：`poll(2)` 路径的 `do_pollfd()` 会把它们无条件并入 `_key`（`filter = demangle_poll(pollfd->events) | EPOLLERR | EPOLLHUP;`）。
  - 未实现 `->poll` 的文件被当作**永远就绪**：`vfs_poll()` 返回 `DEFAULT_POLLMASK`（`EPOLLIN|EPOLLOUT|EPOLLRDNORM|EPOLLWRNORM`）。

### 2.4 `O_NONBLOCK` 与 `FMODE_NOWAIT`

- [ ] **`O_NONBLOCK` 与 `FMODE_NOWAIT` 是两件不同的事**：

  | | `O_NONBLOCK`（`filp->f_flags`） | `FMODE_NOWAIT`（`filp->f_mode`） |
  |---|---|---|
  | 谁设置 | 用户态 `open()`/`fcntl()` | **驱动自己在 `->open` 里设** |
  | 含义 | `read`/`write` 不许睡眠 | "本文件在 `IOCB_NOWAIT` 下会返 `-EAGAIN` 而不是阻塞" |
  | VFS 是否代设 | 用户态决定 | **从不代设**（`do_dentry_open()` 不设这一位） |

  - io_uring 的判据是二者取或：`(file->f_flags & O_NONBLOCK) || (file->f_mode & FMODE_NOWAIT)` 即视为支持 NOWAIT。
  - 没设 `FMODE_NOWAIT` 的后果：io_uring 退化为"先 `vfs_poll()` 试一次，不行就丢给 io-wq 阻塞线程"（见 4.3）。

### 2.5 一个常见误解

- [ ] **就绪不等于数据还在**：`poll` 返回可读与 `read()` 真正取到数据之间可能被别的读者抢先。`read` 仍须在无数据时正确返回 `-EAGAIN`（非阻塞）或睡眠（阻塞），不能假定 `poll` 的结果是一次保证。

> 记法：**`poll` 报"门开着"，`read` 才是"进屋拿东西"；`poll_wait` 只登记门铃，不按门铃。**

---

## 3. 通知模型：`fasync` 与 SIGIO

> 覆盖：`fasync_helper`/`kill_fasync` 的用法与约束、`FASYNC` 位归谁维护、文件关闭时的自动拆除、真实的延迟通知改进（ALSA）。不讲 `poll`（见第 2 节）。

### 3.1 与 `poll` 的分工

- [ ] **`fasync` 是单向通知，与数据无关**：驱动只在"事件发生"时喊一声，用户态收到信号后仍需自己 `read()`。它**不替代** `poll`，二者常同时提供。

### 3.2 驱动侧只写一行

- [ ] **驱动侧只需一行转发**：`fasync_helper()` 内部处理链表插入/删除、加锁、并维护 `FASYNC` 位。

  ```c
  static int my_fasync(int fd, struct file *filp, int on)
  {
  	struct my_dev *dev = filp->private_data;

  	/* fasync_helper() 自带锁，驱动不需要额外加锁 */
  	return fasync_helper(fd, filp, on, &dev->async_queue);
  }
  ```

  - `on` 为真 → 建条目；为假 → 摘条目。返回值为"是否发生了增删"（正数会被 VFS 折算为 0）。
  - **`FASYNC` 位由 `->fasync` 维护**，`setfl()` 的注释写得很直白：`->fasync() is responsible for setting the FASYNC bit.`

### 3.3 `kill_fasync` 的调用时机与 band 取值

- [ ] **`kill_fasync()` 可在中断上下文调用**（`include/linux/fs.h` 注释：`/* can be called from interrupts */`），第三个参数是 poll band：

  | 传入 band | 用户态 `si_band` 等价位（`fs/fcntl.c` `band_table[]`） |
  |---|---|
  | `POLL_IN` | `EPOLLIN \| EPOLLRDNORM` |
  | `POLL_OUT` | `EPOLLOUT \| EPOLLWRNORM \| EPOLLWRBAND` |
  | `POLL_MSG` | `EPOLLIN \| EPOLLRDNORM \| EPOLLMSG` |
  | `POLL_ERR` | `EPOLLERR` |
  | `POLL_PRI` | `EPOLLPRI \| EPOLLRDBAND` |
  | `POLL_HUP` | `EPOLLHUP \| EPOLLERR` |

  - **`POLL_IN` 这个宏名在 7.3-rc4 树内已经不存在**，只有 `band_table[]` 的注释保留了对应关系。新代码建议写 `EPOLLIN | EPOLLRDNORM`，语义完全一致。
  - 信号默认是 `SIGIO`；`fcntl(F_SETSIG)` 可换其他信号。`kill_fasync` 只负责发信号，**不保证不丢**——信号会合并。

### 3.4 文件关闭时的自动拆除

- [ ] **文件关闭时队列会被自动拆除**：`__fput()` 在调用 `->release` **之前**检查 `FASYNC` 位并调用 `->fasync(-1, file, 0)`。

  ```c
  /* fs/file_table.c __fput()（v7.2） */
  if (unlikely(file->f_flags & FASYNC)) {
  	if (file->f_op->fasync)
  		file->f_op->fasync(-1, file, 0);
  }
  if (file->f_op->release)
  	file->f_op->release(inode, file);
  ```

  - 该机制由 commit `233e70f4228e`（Al Viro, 2008-11-01）引入，动机是多数驱动忘了清理。
  - 结论：**驱动不必、也不应再在 `->release` 里手工调用 `my_fasync(-1, filp, 0)`**；写了也合法但冗余。

### 3.5 用户态启用异步通知的标准四步

- [ ] **用户态启用异步通知的标准四步**（`fcntl` 三件套 + 信号处理）：

  ```c
  #include <fcntl.h>
  #include <signal.h>
  #include <unistd.h>

  static volatile sig_atomic_t got_io;

  static void on_sigio(int sig) { got_io = 1; }

  int enable_async(int fd)
  {
  	struct sigaction sa = { .sa_handler = on_sigio };
  	sigemptyset(&sa.sa_mask);
  	if (sigaction(SIGIO, &sa, NULL) < 0)
  		return -1;
  	if (fcntl(fd, F_SETOWN, getpid()) < 0)   /* 谁收信号 */
  		return -1;
  	int flags = fcntl(fd, F_GETFL);
  	if (flags < 0)
  		return -1;
  	return fcntl(fd, F_SETFL, flags | FASYNC);  /* 打开异步通知 */
  }
  ```

  - `F_SETOWN` 正数为进程、负数为进程组；`F_SETSIG` 可把默认的 `SIGIO` 换成实时信号（可排队、不合并）。
  - `FASYNC` 与 `O_ASYNC` 数值相同（`(1 << 13)`），`F_SETFL` 传的就是它。

### 3.6 【进阶】在原子上下文中延迟通知

- [ ] **【进阶】在原子上下文里做通知可能死锁，ALSA 的做法值得抄**：`kill_fasync()` 会经 `send_sigio()` 触达 `tasklist_lock`。ALSA 因此把真正的 `kill_fasync()` 推进工作队列，中断侧只做记录 + `schedule_work()`。

  ```c
  /* sound/core/misc.c（v7.3-rc4）注释逐字：
   * Deferred async signal helpers
   * ... The main purpose is to avoid the messy deadlock
   * around tasklist_lock and co at the kill_fasync() invocation.
   */
  void snd_kill_fasync(struct snd_fasync *fasync, int signal, int poll)
  {
  	if (!fasync)
  		return;
  	guard(spinlock_irqsave)(&snd_fasync_lock);
  	if (!fasync->on)
  		return;
  	fasync->signal = signal;
  	fasync->poll = poll;
  	list_move(&fasync->list, &snd_fasync_list);
  	schedule_work(&snd_fasync_work);   /* 真正的 kill_fasync 在工作队列里跑 */
  }
  ```

  - 如果你的 `kill_fasync()` 调用点在持自旋锁或关中断的路径上，且实测出现"软锁死/tasklist_lock 相关 hung task"，优先考虑这一模式。

### 3.7 小结

> 记法：**`fasync` 只负责"喊一嗓子"，喊完用户态还得自己 `read`；喊话用的宏名现在是 `EPOLLIN|EPOLLRDNORM`，不是 `POLL_IN`。**

---

## 4. 完成模型：AIO 与 io_uring（重点）

> 覆盖：`kiocb` 契约、AIO 的完整调用链、io_uring 如何复用同一接口、`IOCB_NOWAIT` 与 `-EAGAIN` 规则、可抄的驱动实现骨架。不讲生命周期细则（见第 5 节）。

### 4.1 `kiocb` 与驱动的接口契约

- [ ] **`struct kiocb` 的全部字段**（`include/linux/fs.h`，v7.2）：

  ```c
  struct kiocb {
  	struct file		*ki_filp;
  	loff_t			ki_pos;
  	void (*ki_complete)(struct kiocb *iocb, long ret);
  	void			*private;
  	int			ki_flags;
  	u16			ki_ioprio; /* See linux/ioprio.h */
  	u8			ki_write_stream;

  	/*
  	 * Only used for async buffered reads, where it denotes the page
  	 * waitqueue associated with completing the read.
  	 * Valid IFF IOCB_WAITQ is set.
  	 */
  	struct wait_page_queue	*ki_waitq;
  };

  static inline bool is_sync_kiocb(struct kiocb *kiocb)
  {
  	return kiocb->ki_complete == NULL;
  }
  ```

  | 字段 | 驱动要关心的点 |
  |---|---|
  | `ki_filp` | 本次 I/O 的 `struct file`；异步排队期间必须持有引用（见 5.3） |
  | `ki_pos` | 已由内核置为 `iocb->aio_offset`；驱动自己决定是否推进 |
  | `ki_complete` | **非 NULL = 异步调用方**；为 NULL 时必须就地完成 |
  | `private` | 给驱动自由使用的指针（放自己的请求上下文） |
  | `ki_flags` | 关注 `IOCB_WRITE`、`IOCB_NOWAIT`、`IOCB_HIPRI`、`IOCB_AIO_RW` |

- [ ] **常用的 `ki_flags` 位**（v7.2，`RWF_*` 位直接复用）：

  | 位 | 值 | 用途 |
  |---|---|---|
  | `IOCB_HIPRI` | `RWF_HIPRI` | 高优先级 / 可轮询 |
  | `IOCB_NOWAIT` | `RWF_NOWAIT`（user 侧 v4.14+） | **不许阻塞**，做不到就返 `-EAGAIN` |
  | `IOCB_WRITE` | `(1 << 18)` | 这是写操作 |
  | `IOCB_DIRECT` | `(1 << 17)` | 绕过页缓存 |
  | `IOCB_AIO_RW` | 位号见 `fs.h`（曾为 `1<<23`，现为 `1<<22`） | 本次 I/O 来自 AIO 的读写，可用于 `kiocb_set_cancel_fn()` 判据 |

  - **不要硬编码 `IOCB_AIO_RW` 的位号**：`IOCB_DIO_CALLER_COMP` 删除后位号发生过下移。

- [ ] **驱动侧四条硬规则**：

  1. 实现 `ssize_t (*read_iter)(struct kiocb *, struct iov_iter *)`（写为 `write_iter`），二进制接口用 `copy_to_iter`/`copy_from_iter`。
  2. **同步调用（`is_sync_kiocb()`）必须就地完成**并返回字节数或负 errno，绝不能返回 `-EIOCBQUEUED`。
  3. 决定异步时：保存 `kiocb`，**立即返回 `-EIOCBQUEUED`**，之后在 ISR / 工作队列里调 `ki_complete(iocb, ret)`。
  4. **恰好回调一次**。返回 `-EIOCBQUEUED` 之后，AIO 侧 `aio_kiocb` 的引用由内核持有，驱动无需操心内存释放。

### 4.2 AIO 的完整调用链

- [ ] **`io_submit(2)` 到驱动的路径**（`fs/aio.c`，v7.2）：

  ```c
  /* 1) 装配 kiocb：挂上完成回调，记录偏移与标志 */
  static int aio_prep_rw(struct kiocb *req, const struct iocb *iocb, int rw_type)
  {
  	req->ki_complete = aio_complete_rw;
  	req->private = NULL;
  	req->ki_pos = iocb->aio_offset;
  	req->ki_flags = req->ki_filp->f_iocb_flags | IOCB_AIO_RW;
  	if (iocb->aio_flags & IOCB_FLAG_RESFD)
  		req->ki_flags |= IOCB_EVENTFD;
  	...
  }

  /* 2) 检查能力并调用驱动；没有 read_iter 直接失败 */
  static int aio_read(struct kiocb *req, const struct iocb *iocb,
                      bool vectored, bool compat)
  {
  	...
  	if (unlikely(!file->f_op->read_iter))
  		return -EINVAL;
  	...
  	if (!ret)
  		aio_rw_done(req, file->f_op->read_iter(req, &iter));
  	return ret;
  }
  ```

  - **这就是"驱动没实现 `read_iter` 时 `io_submit` 返 `EINVAL`"的源码依据**。
  - `aio_write()` 同构，另有一条 `if (S_ISREG(...)) kiocb_start_write(req);`——**字符设备不走该分支**，即写操作不伴随文件系统冻结保护。

- [ ] **完成回调做什么**（驱动不需要知道，但决定"在中断里回调是否安全"）：

  ```c
  static void aio_complete_rw(struct kiocb *kiocb, long res)
  {
  	struct aio_kiocb *iocb = container_of(kiocb, struct aio_kiocb, rw);
  	...
  	iocb->ki_res.res = res;
  	iocb->ki_res.res2 = 0;
  	iocb_put(iocb);
  }
  ```

  - `aio_complete()` 用 `spin_lock_irqsave(&ctx->completion_lock, ...)` 保护 ring，注释明确写了"might be called from irq context"；`IOCB_FLAG_RESFD` 时走 `eventfd_signal()`，同样声明可在 IRQ 上下文调用。
  - **结论：字符驱动可以在中断处理函数里直接调 `ki_complete()`。** 但要注意此时**用户缓冲区必须已由你自己保证仍有效**（见 5.2）。

- [ ] **用户态 AIO 写法**（需 `libaio`；本文未实测）：

  ```c
  #include <libaio.h>
  #include <linux/aio_abi.h>
  #include <fcntl.h>
  #include <stdio.h>
  #include <string.h>

  int main(void)
  {
  	io_context_t ctx = 0;
  	if (io_setup(8, &ctx) < 0)
  		return 1;

  	int fd = open("/dev/mydev", O_RDWR);
  	char buf[4096];
  	struct iocb cb = {0};
  	struct iocb *cbs[1] = { &cb };
  	struct io_event ev;

  	cb.aio_lio_opcode = IOCB_CMD_PREAD;   /* 或 IOCB_CMD_PWRITE */
  	cb.aio_fildes = fd;
  	cb.aio_buf = (unsigned long)buf;
  	cb.aio_nbytes = sizeof(buf);
  	cb.aio_offset = 0;
  	cb.data = 0x1234;                     /* 完成时原样回到 ev.data */

  	if (io_submit(ctx, 1, cbs) != 1)      /* 返 EINVAL：驱动没有 read_iter */
  		return 1;
  	if (io_getevents(ctx, 1, 1, &ev, NULL) != 1)
  		return 1;

  	printf("res=%ld (负值为 -errno), data=%#llx\n",
  	       (long)ev.res, (unsigned long long)ev.data);
  	io_destroy(ctx);
  	return 0;
  }
  ```

  - 观察点：`ev.res` 是字节数或负 errno；`ev.data` 是提交时填的 `aio_data`。
  - 检查是否真的异步：用 `strace -e trace=io_submit,io_getevents` 观察 `io_submit` 是否立即返回。

### 4.3 io_uring 复用同一接口

- [ ] **io_uring 也走 `read_iter`/`write_iter`**（`io_uring/rw.c`，v7.2）：

  ```c
  static inline int io_iter_do_read(struct io_rw *rw, struct iov_iter *iter)
  {
  	struct file *file = rw->kiocb.ki_filp;

  	if (likely(file->f_op->read_iter))
  		return file->f_op->read_iter(&rw->kiocb, iter);
  	else if (file->f_op->read)
  		return loop_rw_iter(READ, rw, iter);   /* 降级：循环调 ->read */
  	else
  		return -EINVAL;
  }
  ```

  - `ki_complete` 由 io_uring 自己装配（普通模式挂 `io_complete_rw`，`IORING_SETUP_IOPOLL` 挂 `io_complete_rw_iopoll`）。
  - **对驱动而言契约完全相同**：返回 `-EIOCBQUEUED` + 稍后 `ki_complete()`。
  - 只实现 `->read`/`->write` 的驱动走 `loop_rw_iter()`：非阻塞模式下它必然返回 `-EAGAIN`，于是每次请求都被丢给 io-wq 阻塞线程执行——**能用，但等于没有异步**。

- [ ] **`IOCB_NOWAIT` 的规则：做不到就返回 `-EAGAIN`**。

  ```c
  /* io_uring/rw.c：第一次尝试用非阻塞语义进入驱动 */
  if (force_nonblock) {
  	if (unlikely(!io_file_supports_nowait(req, EPOLLIN)))
  		return -EAGAIN;
  	kiocb->ki_flags |= IOCB_NOWAIT;
  } else {
  	kiocb->ki_flags &= ~IOCB_NOWAIT;   /* io-wq 重进时会清掉此位 */
  }
  ```

  | 驱动返回 | io_uring 的动作 |
  |---|---|
  | 字节数 | 完成，写 CQE |
  | `-EIOCBQUEUED` | 等驱动 `ki_complete()`，不重试 |
  | `-EAGAIN` | 先尝试 poll 武装（`io_arm_poll_handler`），不可 poll 则交给 io-wq 以阻塞语义重进 |
  | `-EOPNOTSUPP` | 内核会**代你纠正为 `-EAGAIN`**（源码注释：files "should be returning -EAGAIN"） |

  - **重要推论**：`-EAGAIN` 会导致同一个 `read_iter` 被 io-wq 线程以"允许阻塞"的方式再次调用。所以实现必须**同时**支持非阻塞与阻塞两种模式，不能假定 `IOCB_NOWAIT` 永远在。
  - `REQ_F_NOWAIT` 一旦置位（用户显式给了 `RWF_NOWAIT`），`-EAGAIN` 就是终局，不再重试。

- [ ] **`FMODE_NOWAIT` 决定"是否值得走快速路径"**（`io_uring/io_uring.c`）：

  ```c
  io_req_flags_t io_file_get_flags(struct file *file)
  {
  	io_req_flags_t res = 0;
  	...
  	if ((file->f_flags & O_NONBLOCK) || (file->f_mode & FMODE_NOWAIT))
  		res |= REQ_F_SUPPORT_NOWAIT;
  	return res;
  }
  ```

  - 未设时的兜底（`io_file_supports_nowait()`）：实现 `->poll` 也还能被接受——io_uring 会 `vfs_poll()` 探一次就绪位；否则只能落 io-wq。
  - **因此"实现 `->poll`"除了服务 epoll，还能救 io_uring 的非阻塞路径**，一举两得。

- [ ] **用户态 io_uring 写法**（需 `liburing`；本文未实测）：核心是"填 SQE → 提交 → 取 CQE"，CQE 的 `res` 与 AIO 的 `ev.res` 同义。

  ```c
  #include <liburing.h>
  #include <fcntl.h>
  #include <stdio.h>

  int main(void)
  {
  	struct io_uring ring;
  	if (io_uring_queue_init(8, &ring, 0) < 0)
  		return 1;

  	int fd = open("/dev/mydev", O_RDWR);
  	char buf[4096];
  	struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
  	io_uring_prep_read(sqe, fd, buf, sizeof(buf), 0);
  	io_uring_submit(&ring);

  	struct io_uring_cqe *cqe;
  	if (io_uring_wait_cqe(&ring, &cqe) < 0)
  		return 1;
  	printf("res=%d\n", cqe->res);        /* 字节数或 -errno */
  	io_uring_cqe_seen(&ring, cqe);
  	io_uring_queue_exit(&ring);
  	return 0;
  }
  ```

### 4.4 可抄的驱动实现骨架（同步 + 异步双模式）

- [ ] **一个把"请求排队 + 中断完成"写全的数据采集类字符设备**（骨架，示意用）：

  ```c
  #include <linux/cdev.h>
  #include <linux/fs.h>
  #include <linux/poll.h>
  #include <linux/uio.h>
  #include <linux/wait.h>
  #include <linux/workqueue.h>

  struct my_xfer {
  	struct kiocb *iocb;          /* 异步请求；NULL 表示同步等待 */
  	struct completion done;      /* 同步请求用 */
  	void *buf;
  	size_t len;
  	int result;
  };

  struct my_dev {
  	struct cdev cdev;
  	spinlock_t lock;
  	wait_queue_head_t inq;
  	struct fasync_struct *async_queue;
  	struct my_xfer *xfer;        /* 同一时刻只允许一次在飞 */
  };

  /* 中断上下文：DMA 完成，交付结果 */
  static irqreturn_t my_irq(int irq, void *data)
  {
  	struct my_dev *dev = data;
  	struct my_xfer *x = NULL;
  	unsigned long flags;

  	spin_lock_irqsave(&dev->lock, flags);
  	x = dev->xfer;
  	dev->xfer = NULL;
  	if (x)
  		x->result = (int)x->len;
  	spin_unlock_irqrestore(&dev->lock, flags);

  	if (x) {
  		if (x->iocb) {
  			/* 异步：把结果交回 VFS。此后不得再碰 x->iocb */
  			x->iocb->ki_complete(x->iocb, x->result);
  			kfree(x);
  		} else {
  			complete(&x->done);      /* 同步：唤醒调用者 */
  		}
  	}
  	wake_up_interruptible(&dev->inq);
  	kill_fasync(&dev->async_queue, SIGIO, EPOLLIN | EPOLLRDNORM);
  	return IRQ_HANDLED;
  }

  static ssize_t my_read_iter(struct kiocb *iocb, struct iov_iter *to)
  {
  	struct file *filp = iocb->ki_filp;
  	struct my_dev *dev = filp->private_data;
  	struct my_xfer *x;
  	unsigned long flags;
  	ssize_t ret;

  	if (is_sync_kiocb(iocb)) {
  		/* 同步调用者：必须就地完成，绝不返回 -EIOCBQUEUED */
  		x = kzalloc(sizeof(*x), GFP_KERNEL);
  		if (!x)
  			return -ENOMEM;
  		x->iocb = NULL;
  		init_completion(&x->done);
  	} else {
  		x = kzalloc(sizeof(*x), GFP_KERNEL);
  		if (!x)
  			return -ENOMEM;
  		x->iocb = iocb;
  	}

  	/* NOWAIT 下无法立刻启动 DMA，就交回内核重试/落 io-wq */
  	if (iocb->ki_flags & IOCB_NOWAIT) {
  		kfree(x);
  		return -EAGAIN;
  	}

  	spin_lock_irqsave(&dev->lock, flags);
  	if (dev->xfer) {                     /* 忙 */
  		spin_unlock_irqrestore(&dev->lock, flags);
  		kfree(x);
  		return -EBUSY;
  	}
  	dev->xfer = x;
  	spin_unlock_irqrestore(&dev->lock, flags);

  	my_hw_start_dma(...);                /* 启动硬件，稍后中断 */

  	if (is_sync_kiocb(iocb)) {
  		if (wait_for_completion_interruptible(&x->done)) {
  			/* 被信号打断：此时硬件可能仍在跑，必须取消并等它收尾，
  			 * 让中断侧照常 complete(&x->done)，再自行 kfree(x)。
  			 * 这里为简洁直接返回——真实驱动不能这样漏掉 x。 */
  			return -ERESTARTSYS;
  		}
  		ret = x->result;
  		copy_to_iter(x->buf, ret, to);   /* 同步路径：就地拷回用户缓冲 */
  		kfree(x);
  		return ret;
  	}

  	/* 异步：先返回 -EIOCBQUEUED，结果由 my_irq() 里 ki_complete() 交付 */
  	return -EIOCBQUEUED;
  }
  ```

  - 写方向把 `copy_from_iter()` 放在"启动 DMA 之前"（同步语义），不要在中断里碰 `iov_iter`。
  - 想在 `x->iocb` 上注册取消回调，用 `kiocb_set_cancel_fn()`；它内部靠 `IOCB_AIO_RW` 判断是否来自 AIO，**对同步 `kiocb` 调用是安全的空操作**。
  - `iov_iter` 本身不适合长期持有：真正实现时建议在提交时把用户数据落到自己的 DMA 缓冲（或 `iov_iter_extract_pages()` 取页并固定），再启动传输。
  - 骨架刻意省略的部分：错误路径的 `x` 释放、请求排队（现在忙时返 `-EBUSY`）、`->release` 时的收尾（见 5.6）。

> 记法：**同步分支就地干活就地返回；异步分支"先记录、后返回 `-EIOCBQUEUED`、中断里 `ki_complete`"。**

---

## 5. 工程细节：kiocb 生命周期、引用与取消（重点）

> 覆盖：`kiocb` 存活期规则、用户缓冲区归属、`struct file` 与设备的引用、取消路径、并发与关闭时的清理。不讲 API 用法（见第 4 节）。

### 5.1 `kiocb` 由谁负责释放

- [ ] **规则一：异步保存的 `kiocb` 只需"恰好回调一次"，内存由内核管**。AIO 侧 `aio_kiocb` 带 `refcount_t ki_refcnt`，`io_submit_one()` 末尾会放掉"提交路径的同步引用"，并把要求写在注释里：

  ```c
  	/* Done with the synchronous reference */
  	iocb_put(req);
  	/*
  	 * If err is 0, we'd either done aio_complete() ourselves or have
  	 * arranged for that to be done asynchronously.  Anything non-zero
  	 * means that we need to destroy req ourselves.
  	 */
  ```

  - 但**同步调用者的 `kiocb` 在栈上**（`init_sync_kiocb()` 就地初始化）。所以：**任何情况下都不要把同步路径的 `kiocb` 指针存起来**。

### 5.2 用户缓冲区的归属

- [ ] **规则二：用户缓冲区的生命周期是你自己的责任**。VFS 只保证 `kiocb` 对象存活，**不保证 `iov_iter` 指向的用户内存还在**——进程可以 `munmap`、线程可以退出。参考 `io_uring/rw.c` 的警告：

  ```text
   * This is really a bug in the core code that does this, any issue
   * path should assume that a successful (or -EIOCBQUEUED) return can
   * mean that the underlying data can be gone at any time.
  ```

  - 推荐做法：**提交时就把用户数据拷进驱动的 DMA 缓冲**（读方向相反，完成前拷回），异步期间完全不碰用户地址。
  - 若必须零拷贝：在提交时取页并固定（`iov_iter_extract_pages()` / pin），完成后释放；不要直接保存 `iov_iter`。

### 5.3 为在飞请求持有引用

- [ ] **规则三：为"在飞请求"持有 `struct file` 与设备引用**。设备可以在 I/O 未完成时被 `close` 甚至 `rmmod`。

  | 对象 | 拿引用 | 放引用 |
  |---|---|---|
  | `struct file` | `get_file(filp)` | `fput(filp)` |
  | 模块自身 | `try_module_get(THIS_MODULE)` | `module_put()` |
  | 设备实例 | 你自己的 `kref` / `refcount_t` | 完成后减计数 |

  - 在 `ki_complete()` 之后、`fput()` 之前不要再访问用户缓冲。

### 5.4 取消路径

- [ ] **规则四：处理"已提交但被取消"**。AIO 在上下文销毁时会遍历在飞请求并调用驱动注册的取消函数：

  ```c
  /* fs/aio.c：上下文销毁时 */
  while (!list_empty(&ctx->active_reqs)) {
  	req = list_first_entry(&ctx->active_reqs, struct aio_kiocb, ki_list);
  	req->ki_cancel(&req->rw);
  	list_del_init(&req->ki_list);
  }
  ```

  - 注册方式 `kiocb_set_cancel_fn(iocb, fn)`；`fn` 里通常做"标记取消 + 中止 DMA"。
  - **无论是否成功取消，最终仍必须回调 `ki_complete()` 恰好一次**——取消不等于免除回调义务。
  - 判据提醒：`kiocb_set_cancel_fn()` 只对 `IOCB_AIO_RW` 的 iocb 生效，对同步 `kiocb` 与 io_uring 请求都是空操作。

### 5.5 并发来源

- [ ] **规则五：识别并发的两个来源**。同一个 `file` 可被多线程共享，io_uring 也可能从 io-wq 线程重进：

  | 并发来源 | 表现 | 对策 |
  |---|---|---|
  | 多线程共用 fd | 两个线程同时 `read_iter` | 用 `dev->lock` 保护"在飞请求"槽位，忙则返 `-EBUSY` 或排队 |
  | io-wq 重试 | `-EAGAIN` 后由另一个线程以阻塞语义重进 | 实现必须允许"阻塞模式"路径存在，且状态检查在同一把锁下完成 |
  | 同步与异步混用 | 同一个设备同时有同步 `read()` 与 AIO 在飞 | 把两者归入同一请求队列统一调度 |

### 5.6 `->release` 的收尾顺序

- [ ] **规则六：`->release` 与在飞 I/O 的收尾顺序**。`release` 返回后 fops 可能被替换/卸载，因此：

  1. 在 `release` 中先**停止接受新请求**（置 `closed` 标志），让后续 `read_iter` 直接返 `-EIO`；
  2. **中止或等待**在飞 DMA，并对每个在飞请求调用 `ki_complete(iocb, -EIO)`；
  3. 最后再释放自己的缓冲与设备状态。
  - fasync 队列**不需要**在这里清理（见 3.4）。
  - 顺序写反（先释放缓冲、后完成请求）就是"完成时数据错乱 / oops"的头号成因。

### 5.7 小结

> 记法：**"谁提交、谁排队、谁回调、回调一次"——`kiocb` 不由你释放，用户缓冲不由内核替你保。**

---

## 6. 调试、验证与高频异常

> 覆盖：用户态与内核态的观察手段、高频异常对照表、排错顺序、平台差异。

- [ ] **先确认驱动能力是否被内核看到**：

  | 现象 | 判定 |
  |---|---|
  | `io_submit` 返 `EINVAL` | 驱动没实现 `read_iter`（`fs/aio.c` 直接 `return -EINVAL`） |
  | io_uring 每次请求都落 io-wq | 没设 `FMODE_NOWAIT` 且 `->poll` 也没探到就绪 |
  | `IORING_SETUP_IOPOLL` 直接失败 | 只提供了 `->read`/`->write`，`loop_rw_iter()` 对 `IOCB_HIPRI` 返 `-EOPNOTSUPP` |
  | `io_setup` 返 `EAGAIN` | AIO 上下文耗尽，看 `/proc/sys/fs/aio-max-nr` |

- [ ] **观察手段（按需组合）**：

  | 目的 | 手段 |
  |---|---|
  | 确认用户态提交真的异步 | `strace -e trace=io_submit,io_getevents,io_uring_enter` |
  | 看 io_uring 内核事件 | `echo 1 > /sys/kernel/tracing/events/io_uring/enable` |
  | 看驱动时序 | `trace_printk()` 打点，读 `/sys/kernel/tracing/trace` |
  | 看进程因何睡眠 | `cat /proc/<pid>/stack`、`/proc/<pid>/wchan` |
  | 看谁在发信号 | `kill -0` 排除法 + `cat /proc/<pid>/status` 里的 `SigQ`/`SigPnd` |
  | 压力验证 | `fio --ioengine=libaio --rw=read --bs=4k --direct=1 --filename=/dev/mydev` |

- [ ] **高频异常对照表**：

  | 现象 | 根因 | 解决 |
  |---|---|---|
  | 异步 I/O 永不完成，进程卡在 `io_getevents` | 从 `read_iter` 返了 `-EIOCBQUEUED`，但没人（或漏了某分支）调 `ki_complete()` | 排查所有提前返回分支；确认 ISR 一定送达；"恰好一次"用计数器自检 |
  | `BUG_ON` / 内核 oops 指向 `read_iter` | 同步调用时返回了 `-EIOCBQUEUED` | 用 `is_sync_kiocb(iocb)` 分流，同步分支就地完成 |
  | 完成时 oops 或数据错乱 | 用户缓冲已 `munmap`，或 `struct file` 已释放 | 提交时拷贝到自有缓冲（5.2）；为在飞请求持引用（5.3） |
  | `poll` 卡住不唤醒 | 只在内核内部更新了状态却没调 `wake_up_interruptible()` | 状态变更处统一"改状态 + `wake_up`"，放在同一把锁外调用 |
  | epoll 一直报可读但 `read` 返 `-EAGAIN` | `poll` 与 `read` 的状态判据不一致（两处各写了一遍） | 抽出一个 `bool has_data(dev)` 供两处共用 |
  | 收不到 SIGIO | 用户态漏了 `F_SETOWN`/`FASYNC`；或驱动只调了 `wake_up` 没调 `kill_fasync` | 按 3.5 四步设置；注意二者**不自动联动** |
  | `kill_fasync` 相关的 hung task | 在持自旋锁/关中断路径里直接发信号，撞上 `tasklist_lock` | 学 ALSA：`schedule_work()` 延迟到工作队列（3.6） |
  | 设备 unload 后崩溃 | 在飞 I/O 未收尾就释放了驱动状态 | 按 5.6 顺序收尾：拒新 → 中止/等待 → 完成回调 → 释放 |

- [ ] **排错三步法**：

  1. **先分路**：用户态到底走的是阻塞 `read`、epoll，还是 AIO/io_uring？用 `strace` 确认，别猜。
  2. **再定位契约**：卡在"提交阶段"还是"完成阶段"？提交阶段看返回值（`EINVAL` = 没实现 `read_iter`；`EAGAIN` = NOWAIT 拒绝）；完成阶段看有没有人调 `ki_complete`。
  3. **最后查生命周期**：涉及并发、`close`、`rmmod` 的崩溃，一律先怀疑在飞请求的引用与缓冲归属（第 5 节）。

- [ ] **平台与版本差异**：

  | 事项 | 说明 |
  |---|---|
  | 内核版本 | 本文契约适用于 v5.16+（单参数 `ki_complete`）；v4.1+ 才有 `read_iter`/`write_iter`；`IOCB_DIO_CALLER_COMP` 在 v6.19 已删除 |
  | 用户态库 | `libaio` 走 `io_submit`；`liburing` 走 `io_uring_enter`；二者不是同一后端，`io_uring` 不需要 `libaio` |
  | WSL2 | 支持加载自编译模块需自行编译匹配内核；许多发行版默认内核不带 `CONFIG_IO_URING` 之外的必要调试开关，先用 `modinfo`/`zcat /proc/config.gz` 确认 |
  | 与 POSIX AIO 的区别 | glibc 的 `aio_read(3)` 在多数实现里是**用户态线程池**模拟，与本篇的内核 AIO 不是一回事；要测内核路径必须用 `io_submit(2)` 或 io_uring |

> 记法：**"提交看返回值，完成看回调，崩溃看生命周期。"**

---

## 附录 A 速查表

| API / 宏 | 用途 | 见 |
|---|---|---|
| `ssize_t (*read_iter)(struct kiocb *, struct iov_iter *)` | AIO 与 io_uring 的读入口 | 4.1 |
| `ssize_t (*write_iter)(struct kiocb *, struct iov_iter *)` | 写入口 | 4.1 |
| `is_sync_kiocb(iocb)` | 判断是否同步调用者（`ki_complete == NULL`） | 4.1 |
| `iocb->ki_complete(iocb, ret)` | 异步完成后报告结果（`ret` 为字节数或负 errno） | 4.4 |
| `-EIOCBQUEUED`（529） | `read_iter` 返回值："已排队，稍后由我回调" | 1.3 |
| `IOCB_NOWAIT` | 不许阻塞；做不到返回 `-EAGAIN` | 4.3 |
| `IOCB_WRITE` / `IOCB_HIPRI` / `IOCB_AIO_RW` | 写标志 / 可轮询 / 来自 AIO | 4.1 |
| `FMODE_NOWAIT` | 驱动在 `->open` 里声明支持非阻塞完成 | 2.4 |
| `kiocb_set_cancel_fn()` | 注册取消回调（仅对 `IOCB_AIO_RW` 生效） | 5.4 |
| `__poll_t (*poll)(struct file *, poll_table *)` | 就绪查询 | 2.1 |
| `poll_wait()` | 把调用方登记到等待队列（必须无条件调用） | 2.2 |
| `EPOLLIN \| EPOLLRDNORM` | 可读位（替代已消失的 `POLL_IN`） | 3.3 |
| `wake_up_interruptible()` | 唤醒 poll/epoll 等待者 | 2.5 |
| `int (*fasync)(int, struct file *, int)` | 异步通知注册入口 | 3.2 |
| `fasync_helper()` | 维护 fasync 链表与 `FASYNC` 位 | 3.2 |
| `kill_fasync()` | 投递信号（可中断上下文调用） | 3.3 |
| `fcntl(F_SETOWN)` + `F_SETFL, FASYNC` | 用户态开启异步通知 | 3.5 |
| `io_setup` / `io_submit` / `io_getevents` | 内核 AIO 的用户态接口 | 4.2 |
| `io_uring_queue_init` / `io_uring_submit` / `io_uring_wait_cqe` | io_uring 用户态接口 | 4.3 |

---

## 附录 B 学习路径与进度表

| 阶段 | 对应章节 | 内容 | 目标 | 状态 |
|---|---|---|---|---|
| 1 | 1 | 分层定位、`-EIOCBQUEUED` 语义、版本差异 | 能说出一次异步提交经过了谁、三种模型如何取舍 | ☐ |
| 2 | 2 | `poll` 与等待队列、`O_NONBLOCK` 与 `FMODE_NOWAIT` | 能给设备加上正确的就绪查询 | ☐ |
| 3 | 3 | `fasync` 与 SIGIO、关闭时自动拆除、延迟通知 | 能让设备在事件发生时通知用户态 | ☐ |
| 4 | 4 | `kiocb` 契约、AIO 链路、io_uring、驱动骨架 | 能写出一条可工作的 `read_iter` 异步路径 | ☐ |
| 5 | 5 | 生命周期、引用、取消、收尾顺序 | 能让异步路径在并发与卸载下不崩 | ☐ |
| 6 | 6 | 观察手段、异常对照表、排错三步法 | 能独立定位"不完成/错乱/收不到信号" | ☐ |

> 时间紧的读法：**核心速览 → 第 1 节 → 第 4 节 → 第 6 节 → 附录 A**。
> 正文各节首个 `- [ ]` 即对应上表该阶段的总目标，子条目可逐条勾选。

---

## 附录 C 源码出处与后续扩展

**本文关键结论的一手出处**（版本：stable 7.2.6；括号内为 mainline 7.3-rc4 复核结果）

| 结论 | 出处 |
|---|---|
| `read_iter`/`write_iter` 是唯一迭代接口，`aio_read`/`aio_write` 已删 | `include/linux/fs.h`、commit `8436318205b9`（v4.1） |
| `-EIOCBQUEUED` 的定义与判定语义 | `include/linux/errno.h`；`fs/aio.c` `aio_rw_done()` |
| 同步路径禁止异步返回 | `fs/read_write.c` 的 `BUG_ON(ret == -EIOCBQUEUED)` |
| AIO 装配 `ki_complete`、缺少 `read_iter` 返 `EINVAL` | `fs/aio.c` `aio_prep_rw()` / `aio_read()` |
| io_uring 复用 `read_iter`/`write_iter`，`-EAGAIN` 后 poll 武装或落 io-wq | `io_uring/rw.c`、`io_uring/io_uring.c` |
| `FMODE_NOWAIT` 需驱动自设 | `include/linux/fs.h`；`fs/open.c` `do_dentry_open()` |
| `fasync_helper`/`kill_fasync` 语义与"可在中断调用" | `fs/fcntl.c`；`include/linux/fs.h` |
| 关闭时自动调 `->fasync(-1, file, 0)` | `fs/file_table.c` `__fput()`；commit `233e70f4228e`（2008-11-01） |
| `band` → `si_band` 映射，`POLL_IN` 已非树内宏 | `fs/fcntl.c` 的 `band_table[]` |
| `poll_wait` 语义、`DEFAULT_POLLMASK`、ERR/HUP 自动并入 | `include/linux/poll.h`、`fs/select.c` |
| 延迟通知（避免 `tasklist_lock` 死锁） | `sound/core/misc.c`；`sound/core/pcm_lib.c` |
| 真实字符设备的 `read + poll + fasync` 三件套 | `drivers/infiniband/core/uverbs_main.c`、`fs/pipe.c`、`fs/fuse/dev.c`、`drivers/char/random.c` |
| 官方文档现状：无驱动侧异步 I/O 专页 | `Documentation/filesystems/vfs.rst` 仅有 `read_iter`/`write_iter`/`iopoll`/`poll`/`fasync` 的一行式描述；`driver-api/` 与 `core-api/` 无对应页面 |

**后续扩展（当前缺口）**

正文 6 节已覆盖"就绪 / 通知 / 完成三条路 + 工程细节 + 排错"的主干；以下为**尚未展开**的部分：

1. **`->uring_cmd`（`IORING_OP_URING_CMD`）通道**：正文 1.5 只标注了它的存在与 `io_uring_cmd_done()` 这一回调名，未给出完整实现示例。
2. **零拷贝与页固定的完整写法**：5.2 给了原则（提交时拷贝 / `iov_iter_extract_pages()`）与取舍，未给可编译的 pin/unpin 示例。
3. **`iopoll`（`IORING_SETUP_IOPOLL`）**：正文第 6 节只提到"`loop_rw_iter()` 对 `IOCB_HIPRI` 返 `-EOPNOTSUPP`"，未展开驱动侧 `->iopoll` 的实现。
4. **本文示例的编译与实测**：所有内核模块代码为依据一手源码编写的骨架，**未在本机编译或加载**（作者环境为 Windows，无 Linux 内核树）；用户态示例需 `libaio`/`liburing`，同样未实测。
5. *字符设备 `-EIOCBQUEUED` 实例的穷尽检索*：1.5 的"少见"结论来自约 30 个高概率文件的排查，未做全树 `grep` 证明（可在有内核源码的机器上跑 `rg -l EIOCBQUEUED | rg -v '^(fs|block|mm|io_uring)/'` 复核）。
