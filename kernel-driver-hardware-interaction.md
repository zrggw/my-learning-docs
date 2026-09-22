# Linux 内核 / 驱动模块 / 硬件交互关系（从插上设备到用户态读写）

> 用途：从"会写单个模块"到"能说清三者谁在什么时候调用谁"的路线图 + 要点笔记。面向**写过 hello-world 模块、但说不清内核怎么找到驱动**的读者。
> 使用方式：每学完一个小点，把 `[ ]` 改成 `[x]`；需要展开某节可随时让我补充。
> 定位：讲清 **"一次硬件事件如何变成用户态的一次返回，一次用户请求又如何落到寄存器上"**，重点是**控制通路（谁调用谁）与数据通路（数据怎么流动）**。
> 背景贴合：你已写过字符设备驱动（`char-driver-async-io.md`）——本文补上它未覆盖的"驱动如何被内核发现、如何拿到硬件资源、硬件如何打断内核"这一层。
> 前提：源码引文锚定 **Linux v7.2**（stable 系列，当前 7.2.7；mainline 7.3-rc4）。若按 6.x 资料书写，至少 4 处会写错（见 1.5）。**本文为 Windows 环境撰写，内核代码未在本机编译或加载**，依据是一手源码（见附录 C）。

---

## 核心速览（先看这一页，5 分钟覆盖 80% 内容）

**一句话定位**：内核是**中介与裁判**，驱动是**翻译官**，硬件是**只认寄存器的执行者**。三者之间只有两条通路：**控制通路**（谁调用谁）与**数据通路**（数据怎么流动）。

**① 两条通路的全景**

```text
        控制通路（谁调用谁）                        数据通路（数据怎么走）
 ────────────────────────────────────────  ──────────────────────────────────
 用户态  read()/ioctl()/mmap()                用户缓冲区 ←→ 驱动缓冲 ←→ 设备
    │                                            ▲            ▲
 内核    VFS 系统调用层                          copy_*_user   dma_map / 寄存器
    │                                            │            │
 驱动    fops 回调 → readl/writel                kmalloc /
    │                dma_map_* + 触发 DMA        dma_alloc_coherent
 硬件    寄存器 / FIFO / DMA 引擎                设备读写内存
```

**③ 六个核心概念**
**② 四层结构与故障边界**

```text
 ④ 用户态        应用进程
      ▲           │  read / ioctl / mmap / epoll
      │           ▼
 ════╪═══════════╪════════════════════════════════════════
      │           │  边界①  用户态出错 → 进程收信号
      │           ▼
 ① 内核核心      ├─ 设备模型 / 总线      决定谁绑定谁
      ▲           ├─ 中断子系统          把硬件事件交给谁
      │           └─ 内存 / DMA 子系统    把地址翻译给设备
      │           │  （上行：注册回调）    （下行：调用回调）
 ════╪═══════════╪════════════════════════════════════════
      │           │  边界②  内核态出错 → oops / 整机挂死
      │           ▼
 ② 驱动模块      翻译官：读写寄存器、管缓冲、上报事件
      ▲           │  （中断线拉高）  （下行：寄存器写）
      │           ▼
 ③ 硬件          寄存器 / FIFO / DMA 引擎

  边界① 之上：权限、地址校验、返回值检查——写错只影响一个进程
  边界② 之上：一行错误的寄存器写就能挂死整机——驱动代码要按"能挂死内核"对待
```


| 概念 | 一句话 |
|---|---|
| **控制通路 vs 数据通路** | 前者是"函数调用 + 中断回调"，后者是"寄存器读写 + DMA"；混为一谈是理解困难的主因 |
| **设备模型三元组** | `bus_type` / `device` / `device_driver`，`bus->match()` 是唯一的配对裁判 |
| **probe 是唯一的绑定时刻** | MMIO、IRQ、时钟等资源都在 `probe()` 申请、`remove()` 回收；没 probe 就没有驱动 |
| **上下文决定能做什么** | 硬中断里不能睡眠；要睡眠就交给线程化 IRQ 或 workqueue |
| **三段地址** | 虚拟地址 ≠ 物理地址 ≠ 总线地址；设备只认总线地址，必须经 DMA API 转换 |
| **devm_ 管资源生命周期** | 资源挂在 `struct device` 上，probe 失败与设备移除走同一条回收路径 |

**④ 最常用的 5 条命令 / 3 个观察点**

| 命令 | 作用 |
|---|---|
| `lsmod` / `cat /proc/modules` | 模块是否加载、被谁引用（第三列 Used by） |
| `cat /proc/interrupts` | 每个 IRQ 的触发次数与注册名，判断中断是否真的到达驱动 |
| `ls -l /sys/bus/platform/drivers/<drv>/` | 哪些设备已绑定到该驱动（符号链接即绑定关系） |
| `dmesg -w` | 跟踪 probe/remove 与 `dev_err_probe()` 输出 |
| `cat /sys/kernel/debug/devices_deferred` | 卡在 `-EPROBE_DEFER` 的设备清单 |

| 观察点 | 期望 |
|---|---|
| `/proc/interrupts` 该行计数是否增长 | 不增长说明中断没到 CPU，而不是驱动写得不对 |
| `ls /sys/bus/<bus>/drivers/` 下有无对应目录 | 无目录说明驱动没注册成功（多半 init 返回了负值） |
| `ls /dev/` 有无设备节点 | 无节点说明驱动没调用 `device_create()`/`cdev_device_add()` |

**⑤ 三大高频现象（完整对照表见第 6 节）**

| 现象 | 第一反应 |
|---|---|
| `insmod` 成功但设备不工作 | 看 `probe()` 是否被调用过（`dmesg`），大概率是没匹配上 |
| `rmmod` 报 `ERROR: Module is in use` | 引用计数非零；先查是谁持有（`lsmod` 第三列） |
| 中断计数在涨但用户态收不到 | ISR 里漏了 `wake_up_*()`/`rtc_update_irq()` 之类的事件上报 |

**阅读路线**：只想搞清"谁调用谁" → 第 1 节；要写一个能跑的驱动 → 第 2、3 节；关心数据怎么进出 → 第 4 节；出问题 → 第 6 节；查 API → 附录 A；查源码出处 → 附录 C。

---

## 1. 三方分工与两条通路

> 覆盖：内核/驱动/硬件的职责边界、控制通路双向 + 数据通路的方向、一个完整实例串起来的调用次序、与 6.x 资料的版本差异。不讲具体 API 用法（见第 2、3 节）。

### 1.1 三方职责边界

- [ ] **先把三者拟人化，后面所有细节都挂在这张表上**：

  | | 内核 | 驱动模块 | 硬件 |
  |---|---|---|---|
  | 角色 | 中介 + 裁判 + 资源管理者 | 翻译官：内核语义 ↔ 寄存器操作 | 执行者：只认寄存器和中断线 |
  | 知道什么 | 有哪些总线、设备、驱动；谁在用谁 | 自己这块芯片的寄存器含义 | 什么都不知道 |
  | 被谁调用 | 系统调用、中断、内核线程 | 被内核调（probe/ISR/fops），也可主动调内核 API | 被驱动通过 MMIO/DMA 驱动 |
  | 关键约束 | 不能睡眠的上下文不许睡眠 | 不能在无硬件依据时臆造寄存器语义 | 时序不可协商 |

  - 驱动**既不轮询也不决定"谁先谁后"**：一切时序由内核（设备模型 + 中断子系统）编排。
  - 硬件事件永远**先到 CPU**，再由内核转交给驱动——驱动不可能"监听"硬件。

### 1.2 控制通路：双向调用

- [ ] **下行（用户态 → 硬件）**：系统调用 → VFS → 驱动回调 → 寄存器。

  ```text
  read()/ioctl()/mmap()
    → VFS（file->f_op 指向你的 fops）
      → 你的 read_iter / unlocked_ioctl
        → readl()/writel() + dma_map_*() + 写"启动位"
  ```

- [ ] **上行（硬件 → 用户态）**：中断 → ISR → 下半部 → 唤醒进程。

  ```text
   ① 硬件：状态变化 → 在中断线上置起电平/边沿
        │
   ② 中断控制器：记录、按优先级选一个、发给某个 CPU
        │
   ③ CPU：保存现场 → 进入架构异常入口（irq_enter()）
        │
   ④ 内核：找到控制器对应的 Linux IRQ 号
        │    generic_handle_irq(irq) / generic_handle_domain_irq(domain, hwirq)
        ▼
   ⑤ irq_desc[irq].handle_irq(desc)          ← flow handler
        │    先做芯片级动作：ack / mask（时序随控制器而异）
        ▼
   ⑥ handle_irq_event(desc)                  ← 锁外调用，避免 handler 里拿锁
        │
   ⑦ for_each_action_of_desc: action->handler(irq, action->dev_id)
        │    共享中断：链表上每个驱动的 handler 都会被调用
        │
        ├── 返回 IRQ_HANDLED ──────────→ 完成
        └── 返回 IRQ_WAKE_THREAD ──────→ __irq_wake_thread()
                                              │
                                              ▼
                                     唤醒内核线程 irq/<irq>-<name>
                                     执行 thread_fn（可睡眠）
        │
   ⑧ 中断退出（__irq_exit_rcu）：处理 softirq / 唤醒 ksoftirqd
    → 中断控制器 → CPU 异常入口 → generic_handle_irq()
      → irq_desc 的 flow handler → 遍历 action 链表
        → 你的 handler(irq, dev_id)
          → 清中断 + 记录数据 + 唤醒等待者（或返回 IRQ_WAKE_THREAD）
            → 用户态 read()/poll() 返回
  ```

- [ ] **两条方向共用"绑定关系"**：下行靠 `file->f_op`（打开设备时确定），上行靠 `irq_desc->action`（`request_irq()` 时建立）。**两者都在 `probe()` 里就位**——这就是为什么 probe 是理解一切的起点。

  ```text
  下行绑定    设备 → 驱动代码
              cdev_add() / device_create() → /dev/<name>
              open() 时 chrdev_open() 把 file->f_op 换成你的 fops
  ────────────────────────────────────────────────────────────
  上行绑定    中断线 → 驱动代码
              request_irq(irq, my_handler, ..., dev_id)
                作用：在 irq_desc[irq].action 链表上挂一个 irqaction
                内容：handler = my_handler, dev_id = 设备指针, flags
  ────────────────────────────────────────────────────────────
  共享绑定    两边靠同一份私有数据串起来
              platform_set_drvdata(pdev, info)   ← probe 里存
              filp->private_data = ...           ← open 里取（下行）
              dev_get_drvdata(dev_id)            ← ISR 里取（上行）
  ────────────────────────────────────────────────────────────
  下行（用户 read）:  应用 → file->f_op → 你的 fops → 硬件寄存器
  上行（硬件中断）:  硬件 → irq_desc->action → 你的 handler → wake_up
  ```


### 1.3 一个完整实例的调用次序

- [ ] **以 `drivers/rtc/rtc-sa1100.c`（v7.2，354 行）为主线**，这是本篇贯穿始终的真实样例（选用理由见 5.4）：

  | # | 时序 | 谁调用谁 | 关键代码 |
  |---|---|---|---|
  | 1 | 内核启动 | `do_initcalls()` → 模块 init | `init/main.c` |
  | 2 | 驱动注册 | `platform_driver_register()` → `driver_register()` | `module_platform_driver()` |
  | 3 | 匹配 | `bus->match()` = `platform_match()` 比 DT `compatible` | `drivers/base/platform.c` |
  | 4 | 绑定 | `really_probe()` → `call_driver_probe()` | `drivers/base/dd.c` |
  | 5 | 申请资源 | `devm_*()` 拿 MMIO/IRQ/时钟 | `sa1100_rtc_probe()` |
  | 6 | 暴露用户态 | `devm_rtc_register_device()` → `/dev/rtcN` | RTC 子系统 |
  | 7 | 硬件事件 | ISR 读状态、清中断、上报 | `sa1100_rtc_interrupt()` |
  | 8 | 通知用户态 | `rtc_update_irq()` → 唤醒 `read()`/`poll()` | RTC 子系统 |
  | 9 | 卸载 | `rmmod` → `mod->exit()` → `remove()` → devres 回收 | `delete_module(2)` |

  - 第 3–5 步的源码逐行分析见第 2 节；第 7–8 步见第 3 节；第 2、9 步的模块侧细节见 2.0。

### 1.4 与相邻文档的分工

| 主题 | 在哪篇 |
|---|---|
| 异步读写（AIO/io_uring/fasync）、`kiocb` 生命周期 | `char-driver-async-io.md` |
| `poll`/`fasync`/`kill_fasync` 的语义细节 | `char-driver-async-io.md` 第 2、3 节 |
| 内核怎么找到驱动、probe 拿到什么、中断怎么进来 | **本文** |
| MMIO/DMA 的具体 API 与地址空间 | **本文**第 4 节 |

### 1.5 与 6.x 资料的版本差异（写代码前先看）

| 常见写法 / 说法 | v7.2 现状 | 依据 |
|---|---|---|
| `request_irq()` 等价于 `request_threaded_irq(..., NULL, flags, ...)` | **多了隐式 `IRQF_COND_ONESHOT`** | `include/linux/interrupt.h` |
| `schedule_work()` 投到 `system_wq` | 投到 **`system_percpu_wq`**；`system_wq` 标注 `/* use system_percpu_wq, this will be removed */` | `include/linux/workqueue.h` |
| tasklet 已被删除 | **只是 deprecated，仍在**；官方建议改用线程化 IRQ | `include/linux/interrupt.h` 的 Tasklets 注释块 |
| `struct bus_type` / `struct device` 都在 `include/linux/device.h` | 已拆分：`device/bus.h`、`device/driver.h`、`device.h` | v7.2 头文件布局 |
| `of_device_id` / `acpi_device_id` 在 `mod_devicetable.h` | 已挪到 `include/linux/device-id/{of,acpi}.h`；旧头只做转发 | v7.2 头文件布局 |
| `rmmod` 因引用计数失败返回 `EBUSY` | 多数情况是 **`-EWOULDBLOCK`**（`EBUSY` 另有条件，见第 6 节） | `kernel/module/main.c` |

> 记法：**下行靠 `f_op`，上行靠 `irq_desc->action`，两者都在 probe 里就位。**

---

## 2. 控制通路（一）：模块如何被装载、驱动如何被匹配

> 覆盖：`module_init` 的两种展开与 initcall 机制、insmod/modprobe 的差别、`rmmod` 的引用计数、设备模型三元组与 match→probe 全链路、device tree 匹配、`-EPROBE_DEFER`。不讲中断（见第 3 节）。

### 2.0 模块生命周期：从 `insmod` 到 `rmmod`

- [ ] **`module_init()` 是编译期二态宏**，这正是"同一个驱动可以内建也可以编成模块"的原因：内建时它变成 initcall 段里的一项，模块时它变成 `init_module` 的别名。

  ```c
  /* include/linux/module.h（v7.2）：内建（!MODULE） */
  #define module_init(x)	__initcall(x);
  /* 可加载模块（MODULE）：生成 init_module 别名 */
  #define module_init(initfn)					\
  	static inline initcall_t __maybe_unused __inittest(void)		\
  	{ return initfn; }					\
  	int init_module(void) __copy(initfn)			\
  		__attribute__((alias(#initfn)));		\
  	___ADDRESSABLE(init_module, __initdata);
  ```

  - `__initcall` → `device_initcall` → `__define_initcall(fn, 6)`，即在 `.initcall6.init` 段里放一个指向你的 init 函数的指针。
  - 内建时它由 `init/main.c` 的 `do_one_initcall()` 在启动阶段调用；模块时由 `kernel/module/main.c` 的 `do_init_module()` 调用同一函数。

- [ ] **`insmod` 走 `init_module(2)`，`modprobe` 走 `finit_module(2)`**，两者最终都进 `load_module()`：

  | 命令 | 系统调用 | 差别 |
  |---|---|---|
  | `insmod foo.ko` | `init_module(2)`（读裸 ELF 镜像） | 不解析依赖，参数必须给全 |
  | `modprobe foo` | `finit_module(2)`（按 fd） | 读 `modules.dep` 自动先加载依赖 |
  | `rmmod foo` | `delete_module(2)` | 需引用计数归零 |

  - 权限门槛是 `may_init_module()`：`if (!capable(CAP_SYS_MODULE) || modules_disabled) return -EPERM;`
  - **自动加载**：内核发现需要某驱动时调 `request_module()` → 执行 `modprobe_path`（默认 `/sbin/modprobe`）。`MODULE_DEVICE_TABLE` 的作用就是把设备标识写进模块的 `modules.alias`，让 modprobe 能按 ID 找到模块。

- [ ] **`rmmod` 失败的真正原因与返回码**（`kernel/module/main.c`，v7.2）：

  ```c
  if (!list_empty(&mod->source_list)) {
  	/* Other modules depend on us: get rid of them first. */
  	ret = -EWOULDBLOCK;
  	goto out;
  }
  /* Doing init or already dying? */
  if (mod->state != MODULE_STATE_LIVE) {
  	ret = -EBUSY;
  	goto out;
  }
  ```

  - 引用计数来自 `try_stop_module()` → `try_release_module_ref()`：`atomic_sub_return(MODULE_REF_BASE, &mod->refcnt)` 非零即失败。
  - **驱动应当为自己的活动对象持有模块引用**：打开设备/在飞 I/O 期间 `try_module_get(THIS_MODULE)`，结束时 `module_put()`。这样 `rmmod` 在设备被占用时会失败而不是把代码卸掉。

- [ ] **`__init` 函数可能失效**：内建模块的 `.init.text` 在启动后期被 `free_initmem()` 释放，模块的 init 段由 `do_init_module()` 里的任务释放。所以**不要保存 `__init` 函数的地址、不要期望它被调用第二次**。

- [ ] **模块状态机**：`rmmod` 的三种失败原因都能在这张图上找到位置。

  ```text
   insmod / modprobe
        │
        ▼
   ┌──────────┐  load_module()：校验 ELF、版本、签名、符号
   │ 加载中    │  · 版本不符      → 失败（Invalid module format）
   │ LOADING  │  · 校验/签名失败 → 失败（Key was rejected）
   └────┬─────┘
        │ do_init_module() → 调用你的 module_init()
        ▼
   ┌──────────┐  init 返回 0   → 转入 LIVE
   │ 初始化中  │  init 返回负值 → 失败并回收
   │ COMING   │  init 返回正值 → 仅警告，仍算成功
   └────┬─────┘
        │
        ▼
   ┌──────────┐  ◀── 正常状态：probe 完成，/dev 与 /sys 就绪
   │ 运行中    │      rmmod 前会检查引用计数：
   │  LIVE    │      · 有模块依赖我   → -EWOULDBLOCK
   └────┬─────┘      · 状态非 LIVE    → -EBUSY
        │            · 有 init 无 exit → -EBUSY
        │ rmmod：引用计数必须归零
        ▼
   ┌──────────┐  调用你的 module_exit() → 注销驱动
   │ 卸载中    │  随后 async_synchronize_full() → free_module()
   │ GOING   │
   └────┬─────┘
        │
        ▼
   ┌──────────┐
   │ 已卸载    │  代码段与数据段释放，/sys/module/<name> 消失
   └──────────┘
  ```


### 2.1 设备模型三元组

- [ ] **三个结构各管什么**（v7.2 路径已拆分，注意别按老资料找）：

  | 结构 | 定义位置（v7.2） | 职责 |
  |---|---|---|
  | `struct bus_type` | `include/linux/device/bus.h` | 定义**匹配规则**与总线级操作 |
  | `struct device` | `include/linux/device.h` | 描述**一个硬件实例** |
  | `struct device_driver` | `include/linux/device/driver.h` | 描述**一份驱动代码** |

- [ ] **`bus_type.match()` 是唯一的配对裁判**（v7.2 kerneldoc 原文）：

  ```c
  /**
   * struct bus_type - The bus type of the device
   * @match:	Called, perhaps multiple times, whenever a new device or driver
   *		is added for this bus. It should return a positive value if the
   *		given device can be handled by the given driver and zero
   *		otherwise. It may also return error code if determining that
   *		the driver supports the device is not possible. In case of
   *		-EPROBE_DEFER it will queue the device for deferred probing.
   *		Note: This callback may be invoked with or without the device
   *		lock held.
   * @probe:	Called when a new device or driver add to this bus, and callback
   *		the specific driver's probe to initial the matched device.
   */
  struct bus_type {
  	const char		*name;
  	...
  	int (*match)(struct device *dev, const struct device_driver *drv);
  	int (*probe)(struct device *dev);
  	void (*remove)(struct device *dev);
  	...
  };
  ```

  - 返回值语义：`>0` 匹配成功、`0` 不匹配、`<0` 出错（`-EPROBE_DEFER` 走延迟探测）。
  - **"platform 总线"是虚拟总线**：即使设备挂不上任何真实总线，也一定挂在 platform（或其它虚拟）总线上——这是设备模型"万物皆有总线"的约定。

- [ ] **两个方向都会触发扫描**：

  | 触发点 | 路径 |
  |---|---|
  | 新设备加入 | `device_add()` → `bus_probe_device()` → `device_initial_probe()` → `__device_attach()` |
  | 新驱动注册 | `driver_register()` → `bus_add_driver()` → `driver_attach()` |

  - 两方向都收敛到 `driver_match_device()`：`return drv->bus->match ? drv->bus->match(dev, drv) : 1;`（`drivers/base/base.h`）。

### 2.2 匹配成功之后：match → probe 全链路

- [ ] **调用链（v7.2 逐层）**：

  ```text
  __device_attach()
    → bus_for_each_drv(.., __device_attach_driver)
        → driver_match_device(drv, dev)      ← 调用 bus->match()
        → driver_probe_device(drv, dev)
            → __driver_probe_device(drv, dev)  ← 校验设备状态/链路依赖
                → really_probe(dev, drv)
                    → device_set_driver(dev, drv)
                    → pinctrl_bind_pins(dev)
                    → call_driver_probe(dev, drv)
                        → dev->bus->probe(dev)   ← platform_probe()
                            → drv->probe(dev)    ← 你的 probe()
  ```

- [ ] **`__device_attach_driver()` 里的匹配与延迟探测判据**（`drivers/base/dd.c`，v7.2）：

  ```c
  ret = driver_match_device(drv, dev);
  if (ret == 0) {
  	/* no match */
  	return 0;
  } else if (ret == -EPROBE_DEFER) {
  	dev_dbg(dev, "Device match requests probe deferral\n");
  	dev_set_can_match(dev);
  	driver_deferred_probe_add(dev);
  	/*
  	 * Device can't match with a driver right now, so don't attempt
  	 * to match or bind with other drivers on the bus.
  	 */
  	return ret;
  } else if (ret < 0) {
  	dev_dbg(dev, "Bus failed to match device: %d\n", ret);
  	return ret;
  } /* ret > 0 means positive match */
  ```

- [ ] **probe 的前置检查**（v7.2 `__driver_probe_device()`，理解"为什么 probe 没被调用"的关键）：

  ```c
  if (dev->p->dead || !device_is_registered(dev))
  	return -ENODEV;
  if (dev->driver)
  	return -EBUSY;
  ...
  if (!dev_ready_to_probe(dev))
  	return dev_err_probe(dev, -EPROBE_DEFER, "Device not ready to probe\n");
  ```

  - `really_probe()` 还会拒绝"带着已申请资源进入 probe"的设备：`if (!list_empty(&dev->devres_head)) { dev_crit(dev, "Resources present before probing\n"); ... }`。

- [ ] **`call_driver_probe()` 决定调谁**：总线有 `probe` 就用总线的，否则用驱动的。

  ```c
  if (dev->bus->probe)
  	ret = dev->bus->probe(dev);
  else if (drv->probe)
  	ret = drv->probe(dev);
  ```

  - platform 总线的 `probe` 是 `platform_probe()`，它做完时钟/电源域等通用准备后才调 `drv->probe(dev)`。

### 2.3 platform 总线与设备树匹配

- [ ] **`module_platform_driver()` 展开成什么**（两层宏，v7.2）：

  ```c
  /* include/linux/platform_device.h */
  #define module_platform_driver(__platform_driver) \
  	module_driver(__platform_driver, platform_driver_register, \
  			platform_driver_unregister)
  /* include/linux/device/driver.h */
  #define module_driver(__driver, __register, __unregister, ...) \
  static int __init __driver##_init(void) \
  { \
  	return __register(&(__driver) , ##__VA_ARGS__); \
  } \
  module_init(__driver##_init); \
  static void __exit __driver##_exit(void) \
  { \
  	__unregister(&(__driver) , ##__VA_ARGS__); \
  } \
  module_exit(__driver##_exit);
  ```

  - 于是 `module_platform_driver(sa1100_rtc_driver);` 等价于"写了 `module_init`/`module_exit` 并在里面注册/注销 platform 驱动"。
  - 对应地，`__platform_driver_register()` 干的事就是**把 platform 总线填进 `device_driver`**：`drv->driver.bus = &platform_bus_type;` 然后 `driver_register(&drv->driver)`。

- [ ] **`platform_match()` 的判定顺序**（`drivers/base/platform.c`，v7.2 原文）——这是"内核如何选驱动"最具体的答案：

  ```c
  static int platform_match(struct device *dev, const struct device_driver *drv)
  {
  	struct platform_device *pdev = to_platform_device(dev);
  	struct platform_driver *pdrv = to_platform_driver(drv);
  	int ret;

  	/* When driver_override is set, only bind to the matching driver */
  	ret = device_match_driver_override(dev, drv);
  	if (ret >= 0)
  		return ret;

  	/* Attempt an OF style match first */
  	if (of_driver_match_device(dev, drv))
  		return 1;

  	/* Then try ACPI style match */
  	if (acpi_driver_match_device(dev, drv))
  		return 1;

  	/* Then try to match against the id table */
  	if (pdrv->id_table)
  		return platform_match_id(pdrv->id_table, pdev) != NULL;

  	/* fall-back to driver name match */
  	return (strcmp(pdev->name, drv->name) == 0);
  }
  ```

  | 顺序 | 依据 | 数据来源 |
  |---|---|---|
  | 1 | `driver_override` | 用户态写 `/sys/bus/.../driver_override` |
  | 2 | OF（设备树） | `of_match_table[].compatible` ↔ DT 节点 `compatible` |
  | 3 | ACPI | `acpi_match_table[].id` ↔ `_HID` |
  | 4 | id_table | 平台自定义 ID |
  | 5 | 名字 | `pdev->name` == `drv->name`（老式写法） |

  ```text
  一次 platform_match() 的判定路径（自上而下，命中即返回）

    device 侧（硬件描述）        driver 侧（驱动声明）            结果
    ───────────────────────     ────────────────────────────    ────────
    ① DT 节点 compatible   ◄──►  of_match_table[].compatible    命中 → 1
    ② ACPI _HID / _CID     ◄──►  acpi_match_table[].id          命中 → 1
    ③ platform_device      ◄──►  id_table[]                     命中 → 1
         .name
    ④ platform_device      ◄──►  driver.name                    命中 → 1
         .name
    ───────────────────────     ────────────────────────────    ────────
    都没命中 → 返回 0（不匹配）；判定过程出错 → 返回负值

    driver_override 被用户态写过时，先于以上所有规则生效
    注：x86 常见路径是 ②，ARM / RISC-V 常见路径是 ①
  ```


- [ ] **设备树怎么描述硬件、驱动怎么取数据**：

  ```dts
  rtc@40900000 {
  	compatible = "mrvl,sa1100-rtc";
  	reg = <0x40900000 0x100>;      /* → devm_platform_ioremap_resource() */
  	interrupts = <...>;            /* → platform_get_irq_byname()        */
  	interrupt-names = "rtc 1Hz", "rtc alarm";
  };
  ```

  - 匹配靠 `compatible` 字符串（`Documentation/devicetree/usage-model.rst`：内核从 DT 根开始找带 `compatible` 的节点，为每个节点分配并注册 `platform_device`，后者再绑定到 `platform_driver`）。
  - 驱动侧取私有数据的两种方式：

  | 方式 | 用途 |
  |---|---|
  | `of_match_device(...)->data` / `of_device_get_match_data(dev)` | 取 `.data` 指向的常量配置结构 |
  | `of_device_is_compatible(dev->of_node, "...")` | 同一驱动支持多版本芯片时按字符串分支 |

  - `module_platform_driver` 与 `MODULE_DEVICE_TABLE(of, ...)` 配合，才能让设备热插拔时 modprobe 自动加载。

### 2.4 `-EPROBE_DEFER`：为什么 probe 会被叫第二次

- [ ] **用途**：驱动依赖的资源（GPIO 控制器、时钟、电源域）还没就绪时，返回 `-EPROBE_DEFER` 让内核稍后重试，而不是失败退出（`drivers/base/dd.c` 文件头注释原文）：

  ```text
   * Sometimes driver probe order matters, but the kernel doesn't always have
   * dependency information which means some drivers will get probed before a
   * resource it depends on is available. ...
   * If a required resource is not available yet, a driver can
   * request probing to be deferred by returning -EPROBE_DEFER from its probe hook
  ```

- [ ] **机制**：设备被挂到 pending 链表；**任一驱动 probe 成功**都会触发把 pending 全部转入 active 并 `queue_work(system_dfl_wq, &deferred_probe_work)` 重试。

  - 观测点：`/sys/kernel/debug/devices_deferred`（由 `deferred_probe_initcall()` 里的 `debugfs_create_file("devices_deferred", ...)` 创建）。
  - 延迟探测在 `late_initcall` 阶段才启用，以避开启动期大量驱动探测的干扰。
  - 放弃条件由 `driver_deferred_probe_check_state()` 决定：无模块支持且 initcalls 结束 → `-ENODEV`；超时且模块可用 → `-ETIMEDOUT`；否则继续 `-EPROBE_DEFER`。

- [ ] **官方警告（`Documentation/driver-api/driver-model/driver.rst`，v7.2）**：

  ```text
  .. warning::
        -EPROBE_DEFER must not be returned if probe() has already created
        child devices, even if those child devices are removed again
        in a cleanup path. If -EPROBE_DEFER is returned after a child
        device has been registered, it may result in an infinite loop of
        .probe() calls to the same driver.
  ```

- [ ] **探测顺序没有全局保证**：`enum probe_type` 的 kerneldoc 明确说 `PROBE_PREFER_ASYNCHRONOUS` 适用于"探测顺序对启动不关键"的设备，且"最终目标是默认异步探测"。顺序只能靠 initcall 级别、注册先后、`-EPROBE_DEFER`、device links 间接约束。

  - 需要看"谁先谁后"时用 `driver_async_probe=` 启动参数或逐驱动的 `probe_type` 控制。
  - 若某驱动的 probe 依赖另一设备已就绪，**唯一可靠做法是让它返回 `-EPROBE_DEFER`**，而不是指望顺序。

- [ ] **该不该返回 `-EPROBE_DEFER`：三种情况的判断**：

  ```text
   probe() 里拿不到某个资源
        │
        ├─ 这个资源可能"稍后才出现"？
        │     （时钟、GPIO 控制器、电源域、由另一驱动注册的子系统设备）
        │     └── 是 ──→ return -EPROBE_DEFER;
        │                内核把你挂到 pending 链表，等任一驱动 probe 成功后重试
        │                注意：必须在 probe 早期返回，别先建了子设备再 defer
        │
        ├─ 这个资源在这台机器上"根本不存在"？
        │     └── 是 ──→ return -ENODEV;
        │                内核不再重试
        │
        └─ 拿到了但初始化硬件失败？
              └── 是 ──→ return <具体负 errno>;（-EIO / -ETIMEDOUT ...）
                         内核不再重试，用户态看到明确原因

   观测：/sys/kernel/debug/devices_deferred 列出所有卡在 pending 的设备
   兜底：driver_deferred_probe_check_state() 会在超时后改判 -ETIMEDOUT
  ```


### 2.5 devm_：资源生命周期交给设备

- [ ] **核心语义**（`Documentation/driver-api/driver-model/devres.rst`，v7.2 原文）：

  ```text
  devres is basically linked list of arbitrarily sized memory areas
  associated with a struct device.  Each devres entry is associated with
  a release function.  A devres can be released in several ways.  No
  matter what, all devres entries are released on driver detach.  On
  release, the associated release function is invoked and then the
  devres entry is freed.
  ```

  - **probe 中途失败与设备移除共用同一条回收路径**——这就是驱动 `remove()` 往往只有几行的原因。
  - 释放顺序是**后进先出**（`release_nodes()` 用 `list_for_each_entry_safe_reverse`），因此 IRQ、映射、内存的回收次序与申请相反。

- [ ] **时序上谁先谁后（这是"混用 devm_ 与手动清理为什么危险"的结构性答案）**：

  ```text
  __device_release_driver()
    → device_remove(dev)
        → dev->bus->remove(dev) → drv->remove(dev)   ← 你的 remove() 先跑
    → device_unbind_cleanup(dev)
        → devres_release_all(dev)                    ← devm 资源随后统一释放
  ```

  - 结论：`remove()` 里**能看到** devm 资源仍然有效（可以放心关时钟、清寄存器）；但**不要再手动释放**那些用 `devm_*` 申请的东西。
  - 官方也提醒 devm 只管释放、不管检查：`Managed resources pertains to the freeing of these resources *only* - all other checks needed are still on you.`

- [ ] **devres 链表与释放顺序**（"后申请的先释放"为什么重要）：

  ```text
   probe() 执行顺序                    devres 链表（挂在 dev->devres_head）
   ─────────────────                   ──────────────────────────────────
   devm_kzalloc()      ─────────────►  [内存]
   devm_rtc_allocate_device() ──────►  [内存] → [rtc_device]
   devm_request_irq(irq_1hz) ───────►  [内存] → [rtc_device] → [irq_1hz]
   devm_platform_ioremap_resource() ►  追加 [iomap] 到表头
                                        ▲
                        新条目插在表头 ─┘

   释放时（device_unbind_cleanup → devres_release_all → release_nodes）
   用 list_for_each_entry_safe_reverse 从表尾开始：
        [iomap] 先释放 → [irq_1hz] → [rtc_device] → [内存]
        即"申请的逆序"，与手写 remove() 的惯例一致

   危险写法：devm_request_irq() 之后又手工 free_irq()
     → 释放时 devres 再释放一次 → 双重释放
   危险写法：probe 失败分支里手工 kfree(devm_kzalloc 的内存)
     → 同上，devres 会再 free 一遍
   正确：devm 申请的，就交给 devres 释放，一行都别多写
  ```


> 记法：**"设备注册找驱动、驱动注册找设备，两边都由 `bus->match()` 裁决；probe 里用 devm 申请，remove 里别重复释放。"**

---

## 3. 控制通路（二）：中断如何打通硬件到用户态

> 覆盖：中断子系统三层抽象、中断到达驱动的完整链路、顶半部/线程化 IRQ 的划分与睡眠规则、下半部选型、唤醒用户态。不讲 probe 与资源申请（见第 2 节）。

### 3.1 中断子系统的三层抽象

- [ ] **官方定义（`Documentation/core-api/genericirq.rst`，v7.2 原文）**：

  ```text
  There are three main levels of abstraction in the interrupt code:

  1. High-level driver API
  2. High-level IRQ flow handlers
  3. Chip-level hardware encapsulation
  ```

- [ ] **三层各自的数据结构与职责**：

  ```text
  ① 高层驱动 API（你的代码唯一直接接触的一层）
       request_irq() / request_threaded_irq() / free_irq()
       │
       ▼  注册到
  ② flow handler（内核，负责 ack / mask / eoi 与事件分发）
       handle_level_irq() / handle_edge_irq() / handle_fasteoi_irq()
       │
       ▼  调用
  ③ chip 封装（中断控制器驱动，直接操作控制器寄存器）
       struct irq_chip: irq_mask / irq_ack / irq_unmask / irq_eoi

  你的 handler 由 ② 遍历 desc->action 链表时调用（见 3.2）
  ② 之所以存在：不同控制器的时序不同，内核把它抽象成可替换的 flow handler
  ```


  | 层 | 结构 / 函数 | 职责 | 谁提供 |
  |---|---|---|---|
  | 驱动 API | `request_irq()` / `request_threaded_irq()` / `free_irq()` | 注册与注销 handler | 你（驱动） |
  | flow handler | `handle_level_irq()` / `handle_edge_irq()` / `handle_fasteoi_irq()` | 编排 ack/mask/eoi 与事件分发 | 内核 |
  | chip 封装 | `struct irq_chip`（`irq_mask`/`irq_ack`/`irq_eoi`…） | 直接操作中断控制器寄存器 | 中断控制器驱动 |

- [ ] **每个中断号一个描述符**（`include/linux/irqdesc.h`，v7.2 节选）：

  ```c
  struct irq_desc {
  	struct irq_common_data	irq_common_data;
  	struct irq_data		irq_data;
  	struct irqstat __percpu	*kstat_irqs;
  	irq_flow_handler_t	handle_irq;
  	struct irqaction	*action;	/* IRQ action list */
  	unsigned int		depth;		/* nested irq disables */
  	...
  	raw_spinlock_t		lock;
  	...
  };
  ```

  - `handle_irq` 是 flow handler；`action` 是**所有共享该 IRQ 的驱动 handler 构成的链表**——这就是 `IRQF_SHARED` 的实现基础。
  - `dev_id` 必须是全局唯一的 cookie（通常是设备结构体地址）：共享中断靠它区分"这个中断是不是发给我的"。

- [ ] **硬件号到 Linux 中断号的映射由 `irq_domain` 负责**（`Documentation/core-api/irq/irq-domain.rst` 原文）：

  ```text
  The irq_domain library adds a mapping between hwirq and IRQ numbers on
  top of the irq_alloc_desc*() API. An irq_domain to manage the mapping
  is preferred over interrupt controller drivers open coding their own
  reverse mapping scheme.
  ```

  - 层级 domain 串联多级控制器，例如 x86 上：`Device --> IOAPIC -> Interrupt remapping Controller -> Local APIC -> CPU`。
  - 三级控制器场景下由芯片驱动在解复用后调 `generic_handle_domain_irq(domain, hwirq)` 继续向内分发。

### 3.2 从硬件到 handler 的完整链路

- [ ] **调用链（v7.2）**：

  ```text
  硬件置起中断线
    → 中断控制器
      → CPU 异常入口（架构代码）
        → generic_handle_irq(irq)  或 generic_handle_domain_irq(domain, hwirq)
          → handle_irq_desc(desc)
            → desc->handle_irq(desc)          ← flow handler
              → 先做芯片级动作（mask_ack / ack / eoi）
                → handle_irq_event(desc)
                  → __handle_irq_event_percpu(desc)
                    → for_each_action_of_desc: action->handler(irq, action->dev_id)
  ```

- [ ] **核心入口源码**（`kernel/irq/irqdesc.c`，v7.2）：

  ```c
  int generic_handle_irq(unsigned int irq)
  {
  	return handle_irq_desc(irq_to_desc(irq));
  }
  EXPORT_SYMBOL_GPL(generic_handle_irq);
  ```

  - `irq_desc[]` 数组与稀疏中断树：默认 `CONFIG_SPARSE_IRQ` 下用 maple tree（`sparse_irqs`）按需分配，`irq_to_desc()` 即 `mtree_load(&sparse_irqs, irq)`。

- [ ] **事件分发的核心**（`kernel/irq/handle.c`，v7.2 原文节选）：

  ```c
  		res = action->handler(irq, action->dev_id);
  		trace_irq_handler_exit(irq, action, res);

  		if (WARN_ONCE(!irqs_disabled(),"irq %u handler %pS enabled interrupts\n",
  			      irq, action->handler))
  			local_irq_disable();

  		switch (res) {
  		case IRQ_WAKE_THREAD:
  			/*
  			 * Catch drivers which return WAKE_THREAD but
  			 * did not set up a thread function
  			 */
  			if (unlikely(!action->thread_fn)) {
  				warn_no_thread(irq, action);
  				break;
  			}

  			__irq_wake_thread(desc, action);
  			break;
  ```

  - 关键点：**handler 返回 `IRQ_WAKE_THREAD` 时，内核唤醒 `irq/<irq>-<name>` 内核线程去执行 `thread_fn`**。
  - 内核在此处强制校验"handler 不许开中断"：一旦发现 `irqs_disabled()` 为假就 WARN 并强制关掉——这是"ISR 里不要做重活"的机制性提醒。

### 3.3 顶半部与线程化 IRQ

- [ ] **`irqreturn_t` 三个取值**（`include/linux/irqreturn.h`，v7.2 原文）：

  ```c
  enum irqreturn {
  	IRQ_NONE		= (0 << 0),
  	IRQ_HANDLED		= (1 << 0),
  	IRQ_WAKE_THREAD		= (1 << 1),
  };

  typedef enum irqreturn irqreturn_t;
  #define IRQ_RETVAL(x)	((x) ? IRQ_HANDLED : IRQ_NONE)
  ```

  | 返回值 | 含义 |
  |---|---|
  | `IRQ_NONE` | 这个中断不是我的（共享线上必须老实返回它，否则内核误判中断风暴） |
  | `IRQ_HANDLED` | 我处理了 |
  | `IRQ_WAKE_THREAD` | 请唤醒我的 `thread_fn` 继续处理 |

- [ ] **线程化 IRQ 的分裂点在哪**：

  ```text
   request_threaded_irq(irq, handler, thread_fn, flags, name, dev_id)
                            │        │
   ┌────────────────────────┘        └──────────────────────────┐
   ▼                                                            ▼
   顶半部 handler：硬中断上下文                  thread_fn：内核线程上下文
   · 关抢占、关本 CPU 中断                       · 进程上下文，可以睡眠
   · 读状态寄存器、判"是不是我"                   · 可以拿 mutex、GFP_KERNEL 分配
   · 清中断、把数据搬进自己的缓冲                  · 可以 msleep、可与用户态交互
   · 不能睡眠 / 不能 msleep / 不能拿 mutex        · 仍应尽快返回（占用内核线程）
   · 不能 GFP_KERNEL 分配                       · 同一 IRQ 的 thread_fn 天然串行
   返回：HANDLED / NONE / WAKE_THREAD             返回：HANDLED / NONE
                    │
      返回 IRQ_WAKE_THREAD 时 ──→ 内核唤醒 irq/<irq>-<name> 线程跑 thread_fn
      handler == NULL 时 ──────→ 内核装默认顶半部（仅 return IRQ_WAKE_THREAD）
                                 此时必须带 IRQF_ONESHOT，否则 -EINVAL
  ```

  - 判断标准：**这段代码会不会睡眠？** 会 → 放 `thread_fn`；不会且很短 → 放顶半部。

- [ ] **`request_irq()` 在 v7.2 多带了一个标志**（`include/linux/interrupt.h` 逐字）：

  ```c
  static inline int __must_check
  request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags,
  	    const char *name, void *dev)
  {
  	return request_threaded_irq(irq, handler, NULL, flags | IRQF_COND_ONESHOT, name, dev);
  }
  ```

  - 即 `request_irq()` = "只要不冲突就允许 ONESHOT"的线程化请求。按 6.x 资料写"等价于传 NULL thread_fn"在 v7.2 就不准确了。
  - 线程化 IRQ 的完整签名：`request_threaded_irq(irq, handler, thread_fn, irqflags, devname, dev_id)`。

- [ ] **顶半部/底半部的分工与硬性约束**（`kernel/irq/manage.c` kerneldoc 与官方文档）：

  | | 顶半部 `handler` | 线程化 `thread_fn` |
  |---|---|---|
  | 上下文 | 硬中断（关抢占、关本 CPU 中断） | 内核线程（进程上下文，**可睡眠**） |
  | 能做什么 | 读状态寄存器、清中断、取数据到缓冲、`wake_up_*()` | 拿 mutex、`kmalloc(GFP_KERNEL)`、与用户态交互 |
  | 不能做什么 | 睡眠、`msleep`、拿 mutex、`GFP_KERNEL` 分配、长忙等 | 长时间独占 CPU（会影响系统实时性） |
  | 上报方式 | 返回 `IRQ_HANDLED` 或 `IRQ_WAKE_THREAD` | 返回 `IRQ_HANDLED` |

  - 官方对顶半部的正面禁止（`Documentation/core-api/real-time/differences.rst`，v7.2 逐字）：primary handler "must not acquire any sleeping locks … must avoid introducing delays, such as busy-waiting on hardware registers."
  - `might_sleep()` 的 kerneldoc 也把 irq-handler 直接列为原子上下文（`include/linux/kernel.h`）："this macro will print a stack trace if it is executed in an atomic context (spinlock, irq-handler, ...)"。
  - **`handler == NULL` 时**内核装默认顶半部 `irq_default_primary_handler()`（只有 `return IRQ_WAKE_THREAD;`），此时**必须**设 `IRQF_ONESHOT`，否则 `__setup_irq()` 直接 `-EINVAL` 并在日志里报 "Threaded irq requested with handler=NULL and !ONESHOT"。

- [ ] **`IRQF_ONESHOT` 的语义**：中断线**保持屏蔽直到 `thread_fn` 跑完**，由 `irq_finalize_oneshot()` 在合适时机解除屏蔽。

  - 对电平触发中断尤为重要：不屏蔽的话顶半部唤醒线程后线仍被拉低，会立刻再次触发（"rinse and repeat"）。
  - 共享中断上 `IRQF_ONESHOT` 的协商由 `__setup_irq()` 处理：若已有 action 设了 ONESHOT，新来的带 `IRQF_COND_ONESHOT` 者会被追加上 ONESHOT；不一致则判为 mismatch。

- [ ] **实测入口：`/proc/interrupts`** 直接反映内核视角的中断到达与归属。

  ```bash
  cat /proc/interrupts            # 看每个 IRQ 的每 CPU 计数与注册名
  watch -n1 'grep rtc /proc/interrupts'   # 实时观察某个中断是否在增长
  ```

  - 计数不涨 → 问题在硬件/中断控制器/DT 配置，不在你的 ISR。
  - 计数飞涨但系统卡 → 典型中断风暴：多半是没清中断源，或电平中断没屏蔽。内核有专门的检测：若 100000 次中 99900 次没被处理，会打印 "nobody cared" 并禁用该 IRQ。

### 3.4 下半部选型：线程化 IRQ、workqueue、tasklet

- [ ] **四者的现状与选型**（v7.2）：

- [ ] **选型流程图**（从"中断里干不完"出发）：

  ```text
   硬中断里发现"活干不完"（要睡眠 / 要分配 / 要拿锁 / 耗时）
        │
        ├─ 这段工作属于这个中断本身？
        │     （读 FIFO、处理协议、应答设备）
        │        └── 是 ──→ 线程化 IRQ：thread_fn
        │                     · 优点：同一 IRQ 的 thread_fn 天然串行
        │                     · 需要掩码到处理完 → IRQF_ONESHOT
        │
        └─ 只是"稍后要做"的通用后台任务？
              （超时检查、缓存回写、设备复位）
                 └── 是 ──→ workqueue：schedule_work() / queue_work()
                              · queue_work 的返回值提供内存序保证
                              · 需要自己的执行上下文 → alloc_workqueue()
                              · 想随设备自动销毁   → devm_alloc_workqueue()

   不要选：tasklet（已 deprecated）
   注意：BH workqueue 跑在 softirq 上下文，同样不能睡眠
  ```


  | 机制 | 上下文 | 能否睡眠 | 状态与建议 |
  |---|---|---|---|
  | 线程化 IRQ（`thread_fn`） | 专用内核线程 | ✅ | **首选**：与中断语义绑定最紧，天然串行化同一 IRQ |
  | workqueue | 内核工作线程 | ✅ | 通用延迟工作；适合"与具体中断无关"的后台任务 |
  | BH workqueue（`WQ_BH`） | softirq | ❌ | tasklet 的官方替代方向 |
  | tasklet | softirq | ❌ | **已 deprecated，但未删除**；新代码不要用 |

- [ ] **tasklet 的弃用原文与替代方向**（`include/linux/interrupt.h`，v7.2）：

  ```text
  /* Tasklets --- multithreaded analogue of BHs.

     This API is deprecated. Please consider using threaded IRQs instead:
     https://lore.kernel.org/lkml/20200716081538.2sivhkj4hcyrusem@linutronix.de
  ```

  - 注意弃用声明写在**注释块**里，而不是 `tasklet_setup()` 的定义处——查 API 文档看不到，只能读源码。
  - v7.2 中 tasklet 与 BH workqueue **共用同一个 softirq 上下文**：`kernel/softirq.c` 的 `tasklet_action()` 第一句就是 `workqueue_softirq_action(false);`。

- [ ] **workqueue 的入队与并发语义**（`include/linux/workqueue.h`，v7.2 原文）：

  ```c
  static inline bool schedule_work(struct work_struct *work)
  {
  	return queue_work(system_percpu_wq, work);
  }
  ```

  - **`schedule_work()` 现在投到 `system_percpu_wq`**；`system_wq` 仍存在但已标注 `/* use system_percpu_wq, this will be removed */`。
  - `queue_work()` 有明确的内存序保证（kerneldoc 原文）：若返回 `true`，则调用前所有写操作对执行该 work 的 CPU 可见——**这正是"ISR 填数据 + `schedule_work`"模式能安全工作的依据**。
  - 需要自己的执行上下文时用 `alloc_workqueue(fmt, flags, max_active, ...)`；`devm_alloc_workqueue()` 可随设备自动销毁。
  - `WQ_MEM_RECLAIM` 用于内存回收路径（回收过程中也要能跑），普通驱动一般不需要。

- [ ] **为什么"中断里做不完就丢给下半部"是官方立场**（`Documentation/RCU/checklist.rst` 第 5 条，v7.2 逐字）：

  ```text
  5.	If any of call_rcu(), call_srcu(), call_rcu_tasks(), or
  	call_rcu_tasks_trace() is used, the callback function may be
  	invoked from softirq context, and in any case with bottom halves
  	disabled.  In particular, this callback function cannot block.
  	If you need the callback to block, run that code in a workqueue
  	handler scheduled from the callback.  The queue_rcu_work()
  	function does this for you in the case of call_rcu().
  ```

  - 同一原则适用于所有原子上下文：**需要睡眠的工作，就得搬到进程上下文（线程化 IRQ / workqueue）**。

### 3.5 唤醒用户态

- [ ] **中断不直接"返回数据给用户"**，它只做两件事：改状态 + 唤醒等待者。真正把数据交给用户态的仍是 `read()`。

  ```c
  /* ISR 或 thread_fn 里 */
  spin_lock(&dev->lock);
  dev->data_ready = true;
  spin_unlock(&dev->lock);

  wake_up_interruptible(&dev->wait);   /* 唤醒 poll()/read() 的等待者 */
  ```

- [ ] **与用户态的接口只有三种**：

  | 用户态动作 | 内核侧被调用的驱动回调 | 由谁唤醒 |
  |---|---|---|
  | 阻塞 `read()` | `read`/`read_iter`（内部 `wait_event_*`） | `wake_up_interruptible()` |
  | `poll()`/`epoll_wait()` | `->poll` | `wake_up_interruptible()` |
  | `fcntl(F_SETFL, O_ASYNC)` 后等信号 | 无（ISR 直接发信号） | `kill_fasync()` |

  - 这三条路径的细节、常见错误（`poll_wait` 顺序、`EPOLLERR` 归属、fasync 队列拆除）见 `char-driver-async-io.md` 第 2、3 节。

- [ ] **"先改状态、再唤醒"的顺序不能反**：反了就会出现"唤醒后检查状态发现没数据，于是又睡下去"，用户态表现为莫名其妙的卡住。

- [ ] **唤醒与返回：以 `poll` 为例的时序**（谁在哪个上下文跑，一眼看清）。

  ```text
   用户进程               驱动            内核中断路径            硬件
      │                    │                    │                 │
      │ open("/dev/x")     │                    │                 │
      ├───────────────────►│ open(): 绑定 fops  │                 │
      │                    │                    │                 │
      │ epoll_ctl(ADD)     │                    │                 │
      ├───────────────────►│ ->poll() 第一次调用 │                 │
      │                    │  poll_wait() 登记   │                 │
      │                    │  返回 0（无数据）    │                 │
      │                    │                    │                 │
      │ epoll_wait()       │                    │                 │
      ├───────────────────────────────────────►│ 进程睡眠         │
      │ （睡眠中）          │                    │                 │
      │                    │                    │   数据到达        │
      │                    │                    │◄────────────────┤
      │                    │    调用 ISR         │                 │
      │                    ├◄───────────────────┤                 │
      │                    │ 读状态 / 清中断      │                 │
      │                    │ 改状态：data_ready  │                 │
      │                    │ wake_up_interruptible()               │
      │                    ├───────────────────►│ 唤醒等待队列      │
      │◄───────────────────────────────────────┤                 │
      │ epoll_wait() 返回 1（可读）              │                 │
      │                    │                    │                 │
      │ read()             │                    │                 │
      ├───────────────────►│ ->read_iter()       │                 │
      │                    │ copy_to_user()      │                 │
      │◄───────────────────┤ 返回字节数           │                 │
  ```

  - 两次"进入驱动"分别走不同上下文：`->poll` 在**进程上下文**，ISR 在**硬中断上下文**——两者之间只能靠 `wake_up` 与共享状态沟通。

> 记法：**顶半部只做"认领 + 清中断 + 上报"，要睡眠就返回 `IRQ_WAKE_THREAD`；唤醒只改状态加 `wake_up`，数据仍由 `read()` 取。**

---

## 4. 数据通路：寄存器、内存与 DMA

> 覆盖：MMIO 映射与访问器语义、`volatile` 与屏障、三段地址空间、DMA 的 coherent/streaming 两类 API 与所有权移交、缓冲区生命周期。不讲中断与 probe（见第 2、3 节）。

### 4.1 MMIO：把寄存器映射进内核地址空间

- [ ] **三步走**：查资源 → 映射 → 用访问器读写。

  | 步骤 | 传统写法 | v7.2 推荐写法 |
  |---|---|---|
  | 查资源 | `platform_get_resource(pdev, IORESOURCE_MEM, 0)` | 合并进映射函数 |
  | 映射 | `ioremap()` + 手工 `iounmap()` | `devm_platform_ioremap_resource(pdev, 0)` |
  | 访问 | `readl()`/`writel()` | 同左 |

  - `devm_platform_ioremap_resource()` 内部就是 `platform_get_resource()` + `devm_ioremap_resource()`，并自动在 probe 失败/设备解绑时 unmap。

- [ ] **为什么推荐 devm 版本（官方理由，`Documentation/driver-api/device-io.rst` 逐字）**：

  ```text
  Instead of using the above raw ioremap() modes, drivers are encouraged to
  use higher-level APIs ...
  ```

  - 同文档对 `devm_ioremap_resource()` 的说明："Can automatically select ioremap_np() over ioremap() according to platform requirements … Uses devres to automatically unmap the resource when the driver probe() function fails or a device in unbound from its driver."
  - 警告："Not using these wrappers may make drivers unusable on certain platforms with stricter rules for mapping I/O memory."

- [ ] **`devm_ioremap_resource()` 实际做的三件事**（`lib/devres.c` 内部，v7.2）：校验资源类型 → `devm_request_mem_region()` 独占登记 → `__devm_ioremap()`。

  ```c
  if (!res || resource_type(res) != IORESOURCE_MEM) {
  	ret = dev_err_probe(dev, -EINVAL, "invalid resource %pR\n", res);
  	return IOMEM_ERR_PTR(ret);
  }
  ...
  if (!devm_request_mem_region(dev, res->start, size, pretty_name)) {
  	ret = dev_err_probe(dev, -EBUSY, "can't request region for resource %pR\n", res);
  	return IOMEM_ERR_PTR(ret);
  }
  ```

  - 返回 `-EBUSY` 的常见原因是**另一个驱动已经登记了同一段物理区间**——这是排查"两个驱动抢同一块寄存器"的直接线索。

### 4.2 访问器：`readl` / `writel` 与 `_relaxed` 变体

- [ ] **绝不能直接解引用 `__iomem` 指针**：访问器负责"编译器屏障 + CPU 屏障 + 字节序转换"三件事。

  ```c
  /* include/asm-generic/io.h（v7.2）：readl 的实现骨架 */
  static inline u32 readl(const volatile void __iomem *addr)
  {
  	u32 val;
  	...
  	__io_br();
  	val = __le32_to_cpu((__le32 __force)__raw_readl(addr));
  	__io_ar(val);
  	...
  	return val;
  }
  ```

  - `__io_br()`/`__io_ar()` 是架构可覆盖的屏障钩子；`__raw_readl()` 才是"裸"读（无屏障无字节序转换）。

- [ ] **三档访问器的差别（`Documentation/driver-api/device-io.rst` 表格，v7.2 原文）**：

  | 访问器 | 屏障与有序性 | 使用场合 |
  |---|---|---|
  | `readl()` / `writel()` | 与其它 MMIO、DMA、spinlock 有序 | **默认选择**，可移植代码一律用它 |
  | `readl_relaxed()` / `writel_relaxed()` | **只在彼此之间**有序，屏障更廉价 | 性能敏感快路径，且必须写注释说明为何安全 |
  | `__raw_readl()` / `__raw_writel()` | 无屏障、无字节序转换 | 仅用于设备总线内存，**不要**用于 MMIO 寄存器 |

- [ ] **PCI posted write 与"dummy read"**：总线写是异步的，写完不一定已到达设备。

  ```text
  While the basic functions are defined to be synchronous with respect to
  each other and ordered with respect to each other the buses the devices
  sit on may themselves have asynchronicity. In particular many authors
  are burned by the fact that PCI bus writes are posted asynchronously. A
  driver author must issue a read from the same device to ensure that
  writes have occurred in the specific cases the author cares.
  ```

  - 典型场景：写完配置寄存器后立刻触发操作位，若不"读回一次"再触发，可能顺序错乱。

- [ ] **不要自己写 `volatile`**（`Documentation/process/volatile-considered-harmful.rst` 逐字）：

  ```text
  But, within the kernel, I/O memory
  accesses are always done through accessor functions; accessing I/O memory
  directly through pointers is frowned upon and does not work on all
  architectures.  Those accessors are written to prevent unwanted
  optimization, so, once again, volatile is unnecessary.
  ```

  - 等待硬件状态时用 `cpu_relax()` 而不是空循环：`while (my_variable != what_i_want) cpu_relax();`

### 4.3 三段地址空间

- [ ] **虚拟地址、物理地址、总线地址是三套东西**（`Documentation/core-api/dma-api-howto.rst`，v7.2 原文）：
- [ ] **三段地址空间的转换链**（这是 DMA API 存在的根本原因）：

  ```text
  驱动代码看到的是            MMU 翻译后              设备实际使用
  ──────────────────────    ──────────────────      ──────────────────
  【虚拟地址】               【物理地址】             【总线地址】
  void *                    phys_addr_t             dma_addr_t

  kmalloc() 返回值           virt_to_phys()          dma_map_*() 返回
  ioremap() 返回值           resource->start         dma_alloc_coherent()
  vmalloc() 返回值                                    返回
  ──────────────────────    ──────────────────      ──────────────────
        │                          │                        ▲
        │ CPU 用它执行代码          │ MMU 用页表翻译           │ IOMMU / 主桥
        └─────────────────────────►┘                        │ 做第二层翻译
                                                           │
  设备侧只能发出总线地址 ─────────────────────────────────────┘

  踩坑点：

    ✗ 把 kmalloc 得到的虚拟地址直接写进设备寄存器
      → 设备会把"虚拟地址"当物理地址用，读写位置完全错误
    ✗ 把 vmalloc() 的内存拿去做 DMA
      → 官方明文禁止（虚拟连续 ≠ 物理连续）
    ✗ 把内核镜像 / 模块镜像 / 栈地址拿去做 DMA
      → 同样官方禁止
    ✗ 把用户态指针直接交给设备
      → 页可能被换出；应先 pin 住，再按页 dma_map_page() / dma_map_sg()
  ```


  ```text
  The kernel normally uses virtual addresses.  Any address returned by
  kmalloc(), vmalloc(), and similar interfaces is a virtual address and can
  be stored in a void *.
  ...
  The physical address is not directly useful to a driver; it must use ioremap() to map
  the space and produce a virtual address.

  I/O devices use a third kind of address: a "bus address".  If a device has
  registers at an MMIO address, or if it performs DMA to read or write system
  memory, the addresses used by the device are bus addresses.  In some
  systems, bus addresses are identical to CPU physical addresses, but in
  general they are not.  IOMMUs and host bridges can produce arbitrary
  mappings between physical and bus addresses.
  ```

  | 地址类型 | 谁用 | 典型来源 |
  |---|---|---|
  | 虚拟地址 | CPU 执行代码 | `kmalloc()`/`ioremap()` 的返回值 |
  | 物理地址 | MMU / 页表 | `virt_to_phys()`、`resource` 的 `start` |
  | 总线地址（`dma_addr_t`） | 设备 | DMA API 的返回值 |

  - **设备拿不到虚拟地址**：`DMA doesn't go through the CPU virtual memory system.` 所以驱动必须用 DMA API 把内存"翻译"成设备能用的地址。
  - **不能做 DMA 的内存**（同文档）：`vmalloc()` 分配的内存、内核镜像地址（data/text/bss）、模块镜像地址、栈地址。

### 4.4 DMA：两类映射与所有权移交

- [ ] **两类 API 的选择依据**：

  | | coherent（一致性） | streaming（流式） |
  |---|---|---|
  | 申请 | `dma_alloc_coherent()` / `dma_free_coherent()` | `dma_map_single()`/`dma_map_page()`/`dma_map_sg()` + unmap |
  | 典型用途 | 描述符环、mailbox、固件共享区 | 一次传输的收发缓冲 |
  | CPU 与设备 | 可**同时**访问，无需显式 flush | 同一时刻只归一方所有，需 sync 移交 |
  | 代价 | 某些平台上昂贵，最小分配可能是一页 | 每次传输都要 map/unmap |

  - coherent 的定义（`Documentation/core-api/dma-api.rst` 原文）："memory for which a write by either the device or the processor can immediately be read by the processor or device without having to worry about caching effects."
  - **coherent 也不免除屏障**（同文档 important 块原文）：写描述符时要在"设备可见的字段"之间插 `wmb()`，例如先写地址、`wmb()`、再写 `DESC_VALID`。

- [ ] **所有权移交：流式映射的三阶段**

  ```text
   阶段① 申请        阶段② 映射              阶段③ 传输后 sync
   ─────────────    ──────────────────      ─────────────────────
   kmalloc()        dma_map_single(...,     dma_sync_single_for_cpu()
   或 dma_alloc_      DMA_FROM_DEVICE)         （读方向：取回设备写的数据）
     coherent()     └─ 返回 dma_addr_t        dma_sync_single_for_device()
         │               │                       （写方向：交给设备前刷新）
         │               │                            │
         ▼               ▼                            ▼
   CPU 拥有          所有权移交设备            所有权回到 CPU
   （随便读写）       CPU 不要再碰这块内存       （继续随便读写）
                     ← 这段窗口内 CPU 访问
                        可能拿到旧数据/脏数据
   ──────────────────────────────────────────────────────────────
   常见错误：映射之后 CPU 又去写这块缓冲，然后直接启动 DMA
           → 写入还在缓存里没刷下去，设备读到旧内容
           → 写方向必须在启动 DMA 之前 sync_for_device
  ```

- [ ] **streaming 的方向与所有权规则**（同文档逐字）：

  | 方向 | 规则 |
  |---|---|
  | `DMA_TO_DEVICE` | 软件最后一次修改之后、交给设备之前同步；此后内存对设备只读 |
  | `DMA_FROM_DEVICE` | 驱动读取设备可能改过的数据之前同步；驱动应视其为只读 |
  | `DMA_BIDIRECTIONAL` | **必须同步两次**（交给设备前一次、取回后一次） |

  ```text
  .. note::

     You must do this:

     - Before reading values that have been written by DMA from the device
       (use the DMA_FROM_DEVICE direction)
     - After writing values that will be written to the device using DMA
       (use the DMA_TO_DEVICE) direction
     - before *and* after handing memory to the device if the memory is
       DMA_BIDIRECTIONAL
  ```

- [ ] **映射要短命**：官方原则是"只在真正使用时建立映射，传输结束立刻解除"。

  ```text
  So that Linux can use the dynamic DMA mapping, it needs some help from the
  drivers, namely it has to take into account that DMA addresses should be
  mapped only for the time they are actually used and unmapped after the DMA
  transfer.
  ```

- [ ] **用户缓冲做 DMA 的正确姿势**：先 pin 再按页映射，不要直接把用户指针交给设备。

  - 用户态指针的页可能被换出、可能不在设备可寻址范围内；正确做法是 pin 住页面后用 `dma_map_page()`/`dma_map_sg()` 建立映射。
  - `dma_map_single()` 对"映射失败/不可寻址"是允许失败的：IOMMU 或 bounce buffer 会兜底，可能带来拷贝开销。
  - 传输方向弄反的后果不只是数据错，官方在调试章节警告：违规"can result in data corruption up to destroyed filesystems"。

- [ ] **`dma_need_sync()` 判断是否真需要 sync**：返回 `false` 时可跳过 `dma_sync_*`（架构一致性内存无需 flush 时）。

> 记法：**寄存器用 `readl/writel`（要屏障），内存给设备用 DMA API（要方向正确），三段地址之间没有免费转换。**

---

## 5. 用户态接口与完整实例

> 覆盖：五种用户态接口的定位与选择、一个真实驱动的逐段拆解（probe/IRQ/remove）、用它串起全链路。不讲 API 细节（见第 2、3、4 节）。

### 5.1 五种接口的定位

| 机制 | 定位（v7.2 官方文档） | 适合什么 |
|---|---|---|
| **字符设备 `cdev`** | `ioctl.rst`：`ioctl() is the most common way for applications to interface with device drivers.` | 有状态会话、需要 `read`/`write`/`mmap`/`poll` |
| **sysfs 属性** | `Documentation/filesystems/sysfs.rst`：导出"内核数据结构、属性及其关联关系" | 单值状态与简单旋钮 |
| **ioctl** | `ioctl.rst`：`It is flexible and easily extended by adding new commands` | 设备专属、结构化、需原子语义的命令 |
| **debugfs** | `Documentation/filesystems/debugfs.rst`：`debugfs has no rules at all.`，且 `intended to not serve as a stable ABI` | 调试、寄存器转储 |
| **configfs** | `Documentation/filesystems/configfs.rst`：`configfs is a filesystem-based manager of kernel objects` | 用户态驱动对象生命周期（`mkdir` 建对象） |

- [ ] **sysfs 的硬约束**（sysfs.rst 原文）：`Mixing types, expressing multiple lines of data, and doing fancy formatting of data is heavily frowned upon.` —— 一文件一值。

- [ ] **它们各自对应"哪条通路"**（选错接口是最常见的架构错误）：

  ```text
  控制通路（低频、结构化、能阻塞）        数据通路（高频、流式）
  ────────────────────────────────      ──────────────────────────
  ioctl      设备专属命令                字符设备 read/write
  sysfs      单值状态与旋钮              字符设备 mmap（零拷贝）
  configfs   对象生命周期
  debugfs    调试寄存器

  写错代价：命令语义混乱 / 状态失配       写错代价：吞吐与延迟

  典型：配置采样率、启停设备             典型：传感器数据流、采集卡
  ────────────────────────────────      ──────────────────────────
  反例：把 4KB 采样数据用 sysfs 传 → 违反"一文件一值"，且承载不了速率
  反例：把"启停设备"放到与设备生命周期脱节的接口 → 状态难以维护
  ```

- [ ] **属性必须在设备注册前建好**（`Documentation/driver-api/driver-model/device.rst` 逐字警告）：设备注册时会产生 uevent 通知 udev；注册后再加属性，用户态不会被通知。正确做法是用 `dev_groups`，由 `device_add()` 阶段统一创建。
- [ ] **设备节点从哪里来**：`device_create()`（或 `cdev_device_add()`）→ devtmpfs 自动建 `/dev/<name>`；驱动无需手写 `mknod`。

  ```c
  /* include/linux/device.h（v7.2）：让用户态能按主次设备号自动加载模块 */
  #define MODULE_ALIAS_CHARDEV(major,minor) \
  	MODULE_ALIAS("char-major-" __stringify(major) "-" __stringify(minor))
  ```

### 5.2 实例：`drivers/rtc/rtc-sa1100.c` 逐段拆解

- [ ] **私有数据结构与寄存器偏移**（`probe()` 里填充，处处用 `container_of`/`drvdata` 取回）：

  ```c
  struct sa1100_rtc {
  	spinlock_t		lock;
  	void __iomem		*rcnr;   /* 计数器 */
  	void __iomem		*rtar;   /* 报警 */
  	void __iomem		*rtsr;   /* 状态/使能 */
  	void __iomem		*rttr;   /* 分频 */
  	int			irq_1hz;
  	int			irq_alarm;
  	struct rtc_device	*rtc;
  	struct clk		*clk;
  };
  ```

- [ ] **`probe()`：内核视角的"资源交付清单"**（v7.2 节选，省略处标 `...`，注释为本文所加）：

  ```c
  static int sa1100_rtc_probe(struct platform_device *pdev)
  {
  	int ret;
  	...
  	irq_1hz = platform_get_irq_byname(pdev, "rtc 1Hz");     /* ← 来自 DT interrupts */
  	irq_alarm = platform_get_irq_byname(pdev, "rtc alarm");
  	if (irq_1hz < 0 || irq_alarm < 0)
  		return -ENODEV;

  	info = devm_kzalloc(&pdev->dev, sizeof(struct sa1100_rtc), GFP_KERNEL);
  	...
  	info->rtc = devm_rtc_allocate_device(&pdev->dev);
  	...
  	ret = devm_request_irq(&pdev->dev, irq_1hz, sa1100_rtc_interrupt, 0,
  			       "rtc 1Hz", &pdev->dev);          /* ← 建立"中断→我"的绑定 */
  	...
  	base = devm_platform_ioremap_resource(pdev, 0);          /* ← 来自 DT reg */
  	...
  	info->rcnr = base + 0x04;
  	info->rtsr = base + 0x10;
  	...
  	platform_set_drvdata(pdev, info);                        /* ← 供 ISR/dev_get_drvdata 取回 */
  	device_init_wakeup(&pdev->dev, true);

  	return sa1100_rtc_init(pdev, info);
  }
  ```

  | probe 里发生的事 | 对应第 2/4 节的哪条规则 |
  |---|---|
  | `platform_get_irq_byname()` 取中断号 | DT `interrupts` + `interrupt-names` |
  | `devm_request_irq()` 注册 handler | 控制通路的上行起点（3.2） |
  | `devm_platform_ioremap_resource()` | MMIO 映射（4.1） |
  | `platform_set_drvdata()` | 让 ISR 能通过 `dev_id` 找回私有数据 |

- [ ] **ISR：顶半部的标准动作**（v7.2 原文节选）：

  ```c
  static irqreturn_t sa1100_rtc_interrupt(int irq, void *dev_id)
  {
  	struct sa1100_rtc *info = dev_get_drvdata(dev_id);
  	...
  	spin_lock(&info->lock);

  	rtsr = readl_relaxed(info->rtsr);
  	/* clear interrupt sources */
  	writel_relaxed(0, info->rtsr);
  	...
  	if (rtsr & RTSR_AL)
  		events |= RTC_AF | RTC_IRQF;
  	if (rtsr & RTSR_HZ)
  		events |= RTC_UF | RTC_IRQF;

  	rtc_update_irq(rtc, 1, events);     /* ← 上报子系统 → 唤醒 read()/poll() */

  	spin_unlock(&info->lock);

  	return IRQ_HANDLED;
  }
  ```

  - 标准三段式：**读状态 → 清中断 → 上报事件**。先清后判可避免重复触发。
  - 全部动作都在 `spin_lock` 内、无睡眠、无分配——符合 3.3 的硬中断约束。
  - `dev_id` 传的是 `&pdev->dev`，ISR 里用 `dev_get_drvdata()` 取回 `info`；这条"注册时给什么、回调时拿什么"的对应关系是中断调试的常见混淆点。

- [ ] **`remove()`：devm 带来的极简收尾**（v7.2 原文）：

  ```c
  static void sa1100_rtc_remove(struct platform_device *pdev)
  {
  	struct sa1100_rtc *info = platform_get_drvdata(pdev);

  	if (info) {
  		spin_lock_irq(&info->lock);
  		writel_relaxed(0, info->rtsr);   /* 关掉硬件中断源 */
  		spin_unlock_irq(&info->lock);
  		clk_disable_unprepare(info->clk);
  	}
  }
  ```

  - IRQ、iomap、`rtc_device`、私有内存都由 devm 自动回收（2.5），`remove()` 只需处理"devm 管不到的那件事"：**关掉硬件自身的中断使能**。
  - 这与 2.5 的时序一致：`remove()` 先跑（此时 devm 资源仍有效），随后 `devres_release_all()` 统一回收。

- [ ] **模块尾部：匹配表 + 注册宏**（v7.2 原文）：

  ```c
  static const struct of_device_id sa1100_rtc_dt_ids[] = {
  	{ .compatible = "mrvl,sa1100-rtc", },
  	{ .compatible = "mrvl,mmp-rtc", },
  	{}
  };
  MODULE_DEVICE_TABLE(of, sa1100_rtc_dt_ids);

  static struct platform_driver sa1100_rtc_driver = {
  	.probe		= sa1100_rtc_probe,
  	.remove		= sa1100_rtc_remove,
  	.driver		= {
  		.name	= "sa1100-rtc",
  		.pm	= &sa1100_rtc_pm_ops,
  		.of_match_table = of_match_ptr(sa1100_rtc_dt_ids),
  	},
  };

  module_platform_driver(sa1100_rtc_driver);
  MODULE_LICENSE("GPL");
  ```

### 5.3 用实例串起全链路
- [ ] **实例的数据结构与资源归属**（谁持有谁，一眼看清生命周期）：

  ```text
  ① probe 拿到的东西，各自挂在谁身上？

     struct platform_device                 struct rtc_device
     ┌────────────────────┐                ┌────────────────────┐
     │ resource[] ────────┼──┐             │ ops = &sa1100_rtc_ │
     │ id_entry           │  │             │         ops        │
     └─────────┬──────────┘  │             └─────────▲──────────┘
               │ devm_*      │                       │
               ▼             │       devm_rtc_register_device()
     ┌────────────────────┐  │                       │
     │ struct sa1100_rtc  │  │                       │
     │  lock              │  │             ┌─────────┴──────────┐
     │  rcnr/rtar ────────┼──┘             │ devm_request_irq() │
     │  rtsr/rttr         │                │  irq     = irq_1hz │
     │  irq_1hz / alarm   │                │  handler = my_isr  │
     │  rtc ──────────────┼───────────────►│  dev_id  = &pdev-> │
     │  clk               │                │              dev   │
     └────────────────────┘                └────────────────────┘
               ▲
               └── ISR 里 dev_get_drvdata(dev_id) 取回 sa1100_rtc

  ② 谁负责释放？（devres 链表，挂在内核的 struct device 上）

     申请顺序  kzalloc → rtc_allocate → request_irq → ioremap
     释放顺序  ioremap  → request_irq  → rtc_device  → kzalloc
               （后申请的先前放，与手写 remove() 的惯例一致）

  ③ remove() 里只需处理 devm 管不到的：关掉硬件自身的中断使能
  ```


- [ ] **从加载到用户态读到的完整调用序列**：

  ```text
  ① insmod / 启动           module_init → platform_driver_register()
  ② 驱动注册                driver_register() → bus_add_driver() → driver_attach()
  ③ 匹配                    bus->match() = platform_match() → of_driver_match_device()
  ④ 绑定                    really_probe() → call_driver_probe() → platform_probe()
  ⑤ 你的 probe              sa1100_rtc_probe()：devm_* 拿 MMIO/IRQ/时钟
  ⑥ 暴露用户态              devm_rtc_register_device() → /dev/rtcN + sysfs
  ───────────────────────── 此后硬件事件 ─────────────────────────
  ⑦ 硬件触发中断            RTSR 的 AL/HZ 位置起
  ⑧ 内核分发                generic_handle_irq() → handle_irq_event()
  ⑨ 你的 ISR                sa1100_rtc_interrupt()：读状态、清中断、rtc_update_irq()
  ⑩ 唤醒用户态              RTC 子系统唤醒阻塞在 read()/poll() 的进程
  ───────────────────────── 卸载 ─────────────────────────
  ⑪ rmmod                   delete_module() → mod->exit() → platform_driver_unregister()
  ⑫ 解绑                    device_remove() → drv->remove() → device_unbind_cleanup()
  ⑬ 资源回收                devres_release_all() 逆序释放 IRQ/iomap/rtc_device/内存
  ```

- [ ] **每一步的"谁调用谁"都能落到具体函数**，这正是"三者交互关系"的完整答案。

### 5.4 实例的选择理由与局限（如实说明）

| 维度 | 情况 |
|---|---|
| 体量 | 354 行 `.c` + 23 行 `.h`，无 DMA、无工作队列、无 regmap |
| 覆盖 | 同时用 `devm_*`、`devm_request_irq`、`devm_platform_ioremap_resource`、`readl_relaxed`/`writel_relaxed`、`of_match_table` |
| `remove()` | 极短，正好示范"devm 帮你做完大部分事" |
| 局限 1 | 依赖 `ARCH_SA1100 \|\| ARCH_PXA \|\| ARCH_MMP`，**QEMU `virt` 不模拟，无法直接实跑** |
| 局限 2 | 用户态接口由 RTC 子系统提供，**看不到 `struct file_operations`**；要学 fops 请配 `char-driver-async-io.md` 或自写 misc 设备 |
| 局限 3 | 用 `readl_relaxed`（非 `readl`），因为同文件内访问彼此有序即可满足需求 |
| 局限 4 | 无 ACPI 匹配表，只示范 OF 匹配 |

- [ ] **想真跑起来**：写一个 `misc_register()` 的小设备（无硬件依赖），或使用 QEMU 能模拟的 RTC（如 PL031）；前者适合练 fops，后者适合练 probe 与 sysfs。**本文不提供未经验证的"可跑示例"**（见附录 C）。

> 记法：**"匹配靠 compatible，资源靠 devm，上报靠 rtc_update_irq，回收靠 devres"——一个驱动的一生就这四件事。**

---

## 6. 调试、验证与高频现象

> 覆盖：从下到上的排查顺序、高频现象对照表、平台与版本差异。

- [ ] **按"分层"排查，不要一上来就读驱动代码**：

  | 层 | 检查 | 命令 / 文件 |
  |---|---|---|
  | ① 模块加载 | 模块在不在、init 有没有报错 | `lsmod`、`dmesg \| tail` |
  | ② 匹配绑定 | 驱动目录下有没有设备符号链接 | `ls -l /sys/bus/platform/drivers/<drv>/` |
  | ③ 资源申请 | probe 是否走到、MMIO/IRQ 是否拿到 | `dmesg`、`cat /proc/iomem`、`cat /proc/interrupts` |
  | ④ 中断到达 | 计数是否增长 | `cat /proc/interrupts` |
  | ⑤ 用户态 | 节点是否存在、权限是否正确 | `ls -l /dev/<name>`、`ls /sys/class/<cls>/` |
  | ⑥ 数据通路 | DMA 方向、缓冲区生命周期 | `Documentation/core-api/dma-api.rst` 的调试章节 |

- [ ] **排查决策树**（"设备没反应"往哪查）：

  ```text
   设备没反应
       │
       ▼
   ① lsmod 里有你的模块吗？
       ├── 没有 ──→ init 失败或根本没 insmod
       │             查 dmesg：Invalid module format / Unknown symbol
       └── 有
            │
            ▼
   ② /sys/bus/<bus>/drivers/<drv>/ 下有设备符号链接吗？
       ├── 没有 ──→ 从未匹配上：probe 根本没被调用
       │             查 compatible / ACPI _HID / id_table 是否与硬件描述一致
       │             查 /sys/kernel/debug/devices_deferred 是否卡在 -EPROBE_DEFER
       └── 有
            │
            ▼
   ③ dmesg 里 probe 报错了吗？
       ├── -EBUSY ─→ 资源被别的驱动占了（cat /proc/iomem 找占用者）
       ├── -ENODEV → 依赖资源没拿到（该返回 -EPROBE_DEFER 吗？）
       └── 无报错
            │
            ▼
   ④ 硬件真的在产生事件吗？→ cat /proc/interrupts
       ├── 计数不涨 ──→ 问题在硬件/DT/中断控制器，不在驱动
       │                 查 DT interrupts、线路、设备是否真的被使能
       └── 计数在涨
            │
            ▼
   ⑤ 用户态拿不到数据？
       ├── /dev 节点不存在 ──→ 驱动没调用 device_create()/cdev_device_add()
       ├── 节点在但 read 卡住 ─→ ISR 里漏了 wake_up_*()
       └── 数据内容不对 ────→ 回头看第 4 节：DMA 方向 / sync 时机 / 缓冲生命周期
  ```

  - 这棵树的价值在于**先证伪"驱动代码错了"**：第 ①–④ 步都能在不改动一行驱动代码的前提下完成。

- [ ] **高频现象对照表**：

  | 现象 | 根因 | 解决 |
  |---|---|---|
  | `insmod` 成功但设备不工作 | 没有匹配上，`probe()` 从未被调用 | 查 `of_match_table`/`compatible` 与 DT 是否一致；`driver_override` 是否被设 |
  | `probe` 返回 `-ENODEV` 且反复重试 | 依赖资源未就绪 | 检查是否应返回 `-EPROBE_DEFER`；看 `/sys/kernel/debug/devices_deferred` |
  | `rmmod` 报 `Module is in use` | 引用计数非零 | `lsmod` 看 Used by；查是否有打开的文件/在飞 I/O 未 `module_put()` |
  | `devm_request_mem_region` 失败（`-EBUSY`） | 同一物理区间已被别的驱动登记 | `cat /proc/iomem` 找占用者 |
  | `/proc/interrupts` 里该行不涨 | 中断没到 CPU | 查 DT `interrupts`、中断控制器驱动、硬件是否真的置起了线 |
  | `/proc/interrupts` 飞涨且系统卡 | 中断未清或电平中断未屏蔽 | ISR 里确认"读状态 + 写回清位"；电平触发加 `IRQF_ONESHOT` |
  | 报 "nobody cared (try booting with the irqpoll option)" | 共享线上没有 handler 认领 | 每个共享 handler 必须诚实返回 `IRQ_HANDLED`/`IRQ_NONE` |
  | 日志出现 "Threaded irq requested with handler=NULL and !ONESHOT" | `handler == NULL` 却没设 `IRQF_ONESHOT` | 补 `IRQF_ONESHOT`（或提供顶半部） |
  | `BUG: sleeping function called from invalid context` | 在 ISR/spinlock 内睡眠 | 搬到 `thread_fn` 或 workqueue |
  | 数据偶发错乱 | DMA 方向错误或缓冲区被提前释放 | 核对 `DMA_TO_DEVICE`/`FROM_DEVICE`；确认 sync 时机 |
  | `probe` 里报 "Resources present before probing" | 进 probe 前设备已带 devm 资源 | 检查是否有前一次 probe 失败未清理干净 |

- [ ] **观测手段（按需组合）**：

  | 目的 | 手段 |
  |---|---|
  | 看 probe/remove 时序 | `dmesg -w`，配合 `dev_info()`/`dev_err_probe()` |
  | 看 initcall 耗时与顺序 | 启动参数 `initcall_debug` |
  | 看中断归属与计数 | `/proc/interrupts`、`/proc/irq/<n>/` |
  | 看设备-驱动绑定关系 | `/sys/bus/<bus>/devices/` 与 `drivers/` 下的符号链接 |
  | 看延迟探测队列 | `/sys/kernel/debug/devices_deferred` |
  | 看 DMA 使用是否合规 | 内核配置 `CONFIG_DMA_API_DEBUG` |
  | 看驱动用了哪些模块符号 | `cat /proc/kallsyms`、`modinfo <mod>` |

- [ ] **平台与版本差异**：

  | 事项 | 说明 |
  |---|---|---|
  | 内核版本 | 本文引文锚定 v7.2；`request_irq` 的隐式标志、`system_percpu_wq`、`device-id/` 头文件拆分都是较新的变化（见 1.5） |
  | 设备描述来源 | x86 多为 ACPI，ARM/RISC-V 多为设备树；两者匹配顺序见 2.3 |
  | 无法在本机实测 | 内核模块需与运行内核匹配的头文件与可加载模块的环境；容器内通常不可加载 |

> 记法：**"先看 lsmod 与 /sys 绑定关系，再看 /proc/interrupts，最后才读代码。"**

---

## 附录 A 速查表

| API / 机制 | 用途 | 见 |
|---|---|---|
| `module_init()` / `module_exit()` | 模块入口/出口（二态宏） | 2.0 |
| `module_platform_driver()` | 一行注册 platform 驱动 | 2.3 |
| `MODULE_DEVICE_TABLE()` | 生成 `modules.alias`，支持自动加载 | 2.0 |
| `try_module_get()` / `module_put()` | 使用期间防止模块卸载 | 2.0 |
| `bus_type.match()` | 设备与驱动的唯一配对裁判 | 2.1 |
| `really_probe()` / `call_driver_probe()` | 绑定与调用 probe 的位置 | 2.2 |
| `platform_match()` | OF/ACPI/id_table/名字四级匹配 | 2.3 |
| `of_device_get_match_data()` | 取匹配项的 `.data` 私有配置 | 2.3 |
| `-EPROBE_DEFER` | 依赖未就绪时请求稍后重试 | 2.4 |
| `devm_kzalloc()` / `devm_ioremap_resource()` / `devm_request_irq()` | 资源随设备自动回收 | 2.5 |
| `devres_release_all()` | 解绑时统一回收（LIFO） | 2.5 |
| `request_irq()` / `request_threaded_irq()` | 注册中断 handler | 3.3 |
| `IRQ_HANDLED` / `IRQ_NONE` / `IRQ_WAKE_THREAD` | handler 返回值语义 | 3.3 |
| `IRQF_ONESHOT` / `IRQF_SHARED` | 屏蔽到线程结束 / 共享中断 | 3.3 |
| `generic_handle_irq()` / `generic_handle_domain_irq()` | 中断分发入口 | 3.2 |
| `struct irq_desc.action` | 共享中断的 handler 链表 | 3.1 |
| `schedule_work()` / `queue_work()` | 延迟到进程上下文 | 3.4 |
| `wake_up_interruptible()` | 唤醒阻塞的 read/poll | 3.5 |
| `devm_platform_ioremap_resource()` | 一站式 MMIO 映射 | 4.1 |
| `readl()` / `writel()` | 带屏障的 MMIO 访问 | 4.2 |
| `readl_relaxed()` / `writel_relaxed()` | 仅彼此有序的廉价访问 | 4.2 |
| `dma_alloc_coherent()` | 一致性内存（描述符环） | 4.4 |
| `dma_map_single()` / `dma_unmap_single()` | 流式映射（一次性传输） | 4.4 |
| `dma_sync_single_for_cpu/device()` | 移交缓冲区所有权 | 4.4 |
| `cdev_device_add()` / `device_create()` | 暴露 `/dev` 节点 | 5.1 |
| `dev_groups` / `DEVICE_ATTR_*` | 暴露 sysfs 属性 | 5.1 |
| `/proc/interrupts` | 中断计数与归属 | 3.3 / 6 |
| `/sys/kernel/debug/devices_deferred` | 延迟探测队列 | 2.4 / 6 |

---

## 附录 B 学习路径与进度表

| 阶段 | 对应章节 | 内容 | 目标 | 状态 |
|---|---|---|---|---|
| 1 | 1 | 三方分工、两条通路、调用次序、版本差异 | 能画出两条通路并说出每步谁调用谁 | ☐ |
| 2 | 2 | 模块生命周期、设备模型、match→probe、DT、defer、devm | 能写一个能被正确匹配并申请资源的驱动 | ☐ |
| 3 | 3 | 中断三层抽象、分发链路、顶/底半部、下半部选型、唤醒 | 能让硬件事件可靠地变成用户态返回 | ☐ |
| 4 | 4 | MMIO 映射与访问器、三段地址、DMA 两类映射与所有权 | 能让数据正确进出设备而不踩缓存/地址坑 | ☐ |
| 5 | 5 | 五种用户态接口、真实驱动逐段拆解 | 能独立读懂一个 in-tree 驱动并改它 | ☐ |
| 6 | 6 | 分层排查顺序、现象对照表、平台差异 | 能独立定位"匹配不上/中断不来/数据错乱" | ☐ |

> 时间紧的读法：**核心速览 → 第 1 节 → 第 3 节 → 第 5 节 → 第 6 节 → 附录 A**。
> 正文各节首个 `- [ ]` 即对应上表该阶段的总目标，子条目可逐条勾选。

---

## 附录 C 源码出处与后续扩展

**本文关键结论的一手出处**（版本 v7.2；stable 现为 7.2.7，mainline 7.3-rc4）

| 结论 | 出处 |
|---|---|
| `module_init` 二态展开、initcall 段 | `include/linux/module.h`、`include/linux/init.h`、`init/main.c` |
| `insmod`/`modprobe`/`rmmod` 的系统调用与返回码 | `kernel/module/main.c`（`-EWOULDBLOCK` 来自 `try_stop_module()`） |
| `schedule_work()` 投递目标 | `include/linux/workqueue.h` |
| 设备模型三元组与 `match` 契约 | `include/linux/device/bus.h`、`device.h`、`device/driver.h` |
| match → probe 全链路 | `drivers/base/base.h`、`dd.c`、`bus.c`、`driver.c` |
| `platform_match()` 四级判定 | `drivers/base/platform.c` |
| `-EPROBE_DEFER` 机制与观测点 | `drivers/base/dd.c`、`/sys/kernel/debug/devices_deferred` |
| devm 语义与 LIFO 释放 | `Documentation/driver-api/driver-model/devres.rst`、`drivers/base/devres.c` |
| 中断三层抽象 | `Documentation/core-api/genericirq.rst` |
| 中断分发链路与 `IRQ_WAKE_THREAD` | `kernel/irq/irqdesc.c`、`handle.c`、`chip.c` |
| `request_irq()` 的隐式 `IRQF_COND_ONESHOT` | `include/linux/interrupt.h` |
| 顶半部不得睡眠 | `Documentation/core-api/real-time/differences.rst`、`include/linux/kernel.h` |
| tasklet 弃用现状 | `include/linux/interrupt.h` 的 Tasklets 注释块 |
| MMIO 访问器三档差别、posted write | `Documentation/driver-api/device-io.rst`、`include/asm-generic/io.h` |
| 三段地址与 DMA 规则 | `Documentation/core-api/dma-api.rst`、`dma-api-howto.rst` |
| 实例驱动全部代码 | `drivers/rtc/rtc-sa1100.c`、`drivers/rtc/rtc-sa1100.h` |
| 官方文档现状 | `Documentation/driver-api/driver-model/` 下 `binding/bus/device/devres/driver/overview/platform.rst` 均存在（v7.2） |

**后续扩展（当前缺口）**

正文 6 节已覆盖"分工与两条通路 + 加载匹配 + 中断链路 + 数据通路 + 用户态接口 + 排错"的主干；以下为**尚未展开**的部分：

1. **可实跑的完整示例**：本文实例依赖 SA-1100/PXA 古董 SoC，QEMU `virt` 无法复现；正文第 5 节给了替代方向（misc 设备 / QEMU 可模拟的 RTC），未给出代码。
2. **ACPI 匹配路径**：2.3 给出了 `platform_match()` 的判定顺序与 `acpi_match_table` 的位置，未展开 `_HID`/`_CID` 与 `device_get_match_data()` 的 ACPI 实现（该函数实现未取到原文）。
3. **`device_add()` 内部细节**：v7.2 的 `drivers/base/core.c` 单文件超出一手抓取工具的上限，正文只引用了 v7.2 可确证的部分（`__driver_probe_device()` 中关于 `ready_to_probe` 的注释），未逐行引用 `device_add()` 本体。
4. **`kmap` 局部的 DMA 与 cacheline 共享问题**：4.4 只提到"用户缓冲需先 pin"，未展开 `__dma_from_device_group_*` 对齐宏与 cacheline 伪共享的真实案例。
5. **电源管理（runtime PM / system suspend）与设备模型的交互**：`dev_pm_ops`、`pm_runtime_get_sync()` 在 probe 顺序中的位置未展开。
6. **正文示例的编译与实测**：全部内核代码为依据一手源码摘录或改写的骨架，**未在本机编译或加载**（作者环境为 Windows，无 Linux 内核树）。
