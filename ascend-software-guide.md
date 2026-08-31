# 昇腾软件（Ascend Software）学习指南

> 用途：从零认识华为昇腾 AI 计算平台，重点讲**软件栈**（CANN、框架适配、工具链），配合硬件体系速览。
> 使用方式：每学完一项勾选 `[ ]`，需要展开某节可随时让我补充。
> 背景贴合：你已熟悉 C/C++/Go 与深度学习基础，文档会结合"从模型到 NPU 部署"这条主线讲。
> 定位：先建立"昇腾到底是什么、软件分几层、模型怎么跑上去"的**全景心智模型**，细节按需深入。

---

## 1. 昇腾是什么、解决什么问题

- [ ] **昇腾（Ascend）**：华为自研的 AI 计算平台，包含**芯片（Ascend 系列 AI 处理器）+ 全栈软件**，覆盖训练、推理、边缘、云端等 AI 场景
- [ ] 解决什么问题：把深度学习模型（PyTorch/TensorFlow 训练出来的 *"模型文件"*）高效地跑到专门的 AI 芯片上。
  - 相比 GPU：昇腾用**专用 AI Core + 达芬奇（Da Vinci）架构**，在特定算子（卷积、矩阵乘、Transformer 类）上能效更高
  - 但换来的是：**软件栈更复杂**、生态相对年轻，需要理解其分层模型
- [ ] 三条主线（贯穿全文）：
  1. **硬件**：Ascend 芯片 → Atlas 板卡/服务器/集群
  2. **软件**：CANN（核心软件栈）→ 框架适配 → 工具链
  3. **流程**：训练 → 模型转换 → 部署推理
- [ ] 一句话：**昇腾 = 华为的"芯片 + 软件"AI 算力底座**，好比"英伟达 GPU + CUDA"之于 AI 训练，但架构和软件栈不同。

> 背景记法：芯片层对标英伟达的 GPU；软件栈对标 CUDA + cuDNN + 框架适配这一整套。

---

## 2. 硬件体系速览（让软件有落脚点）

- [ ] **AI 处理器（AI Processor）**：
  - **Ascend 310**：推理场景（低功耗，边缘/端侧）
  - **Ascend 910**：训练场景（高性能）；后续演进有 910B / 910C 等（面向集群/大模型）
- [ ] **达芬奇（Da Vinci）架构**：昇腾处理器的核心设计。
  - 关键单元：**AI Core**（算力核心，含向量/标量/矩阵）→ 内部有 **AI Core 计算单元**（Vector 单元、Cube 单元、Scalar 单元）
  - 围绕 AI Core 还有缓存、存储管理器、指令控制等；**理解"矩阵乘/卷积由专用单元加速"即可**
- [ ] **Atlas 系列（整机/板卡产品）**：把芯片装成可用的算力形态：
  - Atlas 200（端侧推理卡）、Atlas 300（推理卡）、Atlas 500（边缘小站）、Atlas 800（推理/训练服务器）、Atlas 900（训练集群）
  - 还有 Atlas A2 系列（更新一代，配套 CANN 新版）
- [ ] **同构 vs 异构**：昇腾常用于**异构计算**——CPU 做调度/通用逻辑，NPU 做 AI 计算，软硬件协同。

> 记法：310 推理、910 训练；Atlas 是"板卡/服务器/集群"的包装。软件要学的都是**跑在这些硬件上**的东西。

---

## 3. CANN：昇腾的核心软件栈（重点）

- [ ] **CANN** = **C**ompute **A**rchitecture for **N**eural **N**etworks，华为面向 AI 场景的**异构计算架构**
- [ ] 定位（承上启下）：
  - **对上**：支持多种 AI 框架（PyTorch / TensorFlow / MindSpore / PaddlePaddle / ONNX 模型）
  - **对下**：面向昇腾处理器与硬件，提供算子、运行时、图引擎、编译器
- [ ] CANN 的**分层结构**（学习主线，从上到下）：
  | 层 | 作用 | 关键组件 |
  |---|---|---|
  | **应用层** | 用户写的业务/AI 应用 | 框架适配层、AscendCL 应用 |
  | **框架层** | 让主流框架跑在昇腾上 | PyTorch（`torch_npu`）、TensorFlow、MindSpore |
  | **计算/图引擎层** | 图优化、算子调度、资源编排 | **GE（Graph Engine）**、算子编译器 |
  | **算子层** | 提供/开发算子 | 内置算子库、**TBE**（Tensor Boost Engine，算子开发 DSL） |
  | **运行时/驱动层** | 设备管理、内存、任务下发 | **AscendCL**、Driver、Runtime |
- [ ] **关键概念**：
  - **AscendCL（Ascend Computing Language）**：C 语言 API，是应用直接操控算力的入口，含设备管理、内存、模型加载、算子调用、图执行
  - **GE（Graph Engine）**：图引擎，把模型图做**图编译/图优化**（算子融合、内存复用、调度）
  - **ATC（Ascend Tensor Compiler）**：把模型文件转成昇腾能直接跑的 `.om` 模型
  - **HCCL（Huawei Collective Communication Library）**：集合通信库，跨卡/跨机之间的通信（类似 NCCL），多卡训练/大模型必用
- [ ] 版本演进（知道有社区版/商业版即可，细节不必背）：CANN 社区版如 `8.0.RC2`、`7.x` 等，出厂伴随 Atlas 产品的 "软件套装版本"，通常配套特定驱动。

> 记法：CANN 下管**芯片**、上接**框架**，中间是**图引擎 + 算子 + 运行时**。学软件栈就是按这四层往下钻。

---

## 4. AscendCL：应用层直接操控算力的 C 接口

> 用途：写底层应用/算子/服务时用。如果只做"框架模型推理"，走第 5 节更省事；但理解 AscendCL 是理解整个栈的钥匙。

- [ ] 核心 API 分组（`acl` 前缀，C 接口）：
  | 类别 | 作用 | 代表性接口 |
  |---|---|---|
  | **设备管理** | 初始化/切换设备 | `aclInit` / `aclFinalize` / `aclrtSetDevice` |
  | **上下文/流** | 排队执行、同步 | `aclrtCreateStream` / `aclrtSynchronizeStream` |
  | **内存管理** | 分配/释放设备内存 | `aclrtMalloc` / `aclrtFree` / `aclrtMemcpy` |
  | **数据搬运** | 宿主↔设备拷贝 | `aclrtMemcpy`（`ACL_MEMCPY_HOST_TO_DEVICE` 等） |
  | **模型管理** | 加载/执行模型 | `aclmdlLoadFromFile` / `aclmdlExecute` |
  | **算子/张量** | 数据描述与算子 | `aclCreateTensorDesc` / `aclCreateDataBuffer` |
- [ ] 经典流程（加载单个 `.om` 模型推理）：
  1. `aclInit` 初始化 → `aclrtSetDevice` 选设备
  2. `aclrtCreateStream` 创建流（异步执行队列）
  3. `aclmdlLoadFromFile("model.om")` 加载模型
  4. 准备输入/输出：`aclCreateDataBuffer`（绑定设备内存）→ `aclmdlExecute`
  5. `aclrtSynchronizeStream` 同步等待完成 → 拿输出
  6. 释放资源：`aclmdlUnload` → `aclrtFree` → `aclrtDestroyStream` → `aclFinalize`
- [ ] 与 CUDA 对照：
  | CUDA | AscendCL | 作用 |
  |---|---|---|
  | `cudaSetDevice` | `aclrtSetDevice` | 选设备 |
  | `cudaMalloc`/`cudaMemcpy` | `aclrtMalloc`/`aclrtMemcpy` | 设备内存 |
  | `cudaStreamCreate` | `aclrtCreateStream` | 执行流 |
  | `cudaLaunchKernel`/库 | `aclmdlExecute`/算子 | 执行计算 |
- [ ] 学习建议：先会"用 AscendCL 加载一个现成 `.om` 模型跑一轮推理"，再深入算子与图。

---

## 5. 框架适配：把模型跑上昇腾的最省事路径

- [ ] **PyTorch + `torch_npu`**（当前主流，最常用）：
  - `torch_npu` 让 PyTorch 的 `device` 认识昇腾：`device = torch.device('npu:0')`
  - 使用方式几乎零改动：模型、数据、运算挪到 `npu` 设备即可
  ```python
  import torch
  import torch_npu   # 引入后 npu 设备可用
  device = torch.device("npu:0")
  model = model.to(device)
  x = x.to(device)
  y = model(x)       # 算子后台由 CANN/算子库执行
  ```
  - 训练时：数据加载、优化器、`loss.backward()` 都与 CUDA 相似，主要差异在**算子底层执行路径**与**通信（HCCL）**
- [ ] **MindSpore**（华为自研框架）：
  - 原生支持昇腾，`context.set_context(device_target="Ascend")`
  - 图模式（Graph）与动态图（PyNative）两种；和昇腾的图引擎配合更"原生"
- [ ] **TensorFlow / PaddlePaddle / ONNX**：各有适配插件；ONNX 模型可先用 ATC 转成 `.om` 再上昇腾。
- [ ] **迁移成本通常很低**：PyTorch 项目往往只改"设备"与个别算子；但**算子是否被昇腾支持/替代**是主要风险点，需查算子支持清单。

> 记法：`.to("npu:0")` 是 PyTorch 上昇腾的**标志动作**。你的模型层代码基本不用动。

---

## 6. 模型转换：从框架模型到 `.om`

- [ ] **ATC（Ascend Tensor Compiler）**：把 ONNX / MindSpore / Caffe 等模型转成昇腾可执行的 **`.om`（Offline Model）**
- [ ] **为什么需要转换**：`.om` 是**离线编译好的、已做算子融合/图优化的**可执行模型，运行时直接"核上跑"，省去每步解释。
- [ ] 典型命令（示意）：
  ```bash
  atc --model=model.onnx --framework=5 --output=model_om \
      --soc_version=Ascend310 --input_shape="input:1,3,224,224" --output_type=FP32
  ```
  - `--soc_version`：指定芯片型号（如 `Ascend310` / `Ascend910` / `Ascend910B1`）
  - `--framework`：源框架编号（如 ONNX=5）；`--input_shape`：固定输入尺寸
- [ ] 转换过程产物：`.om` 主模型 + 可选权重；AI Core 算子被编译进模型，CPU 算子（如部分数据预处理）在 Host 侧执行
- [ ] 精度与性能权衡：可指定 O1/O2 等**精度模式**（混合精度），或算子融合开关；必要时做**精度对齐**（与 GPU/参考输出对比）

> 记法：**训练出来的模型 → ATC → `.om` → AscendCL/框架执行**，这就是昇腾上"模型怎么跑"的链条。

---

## 7. 部署形态与通信（多卡/集群）

- [ ] **单卡推理**：一张 Atlas 卡加载 `.om` 或直接框架推理，服务化用 **MindX** 或自建服务
- [ ] **多卡训练/大模型**：需要 **HCCL** 做集合通信（AllReduce / AllGather / Broadcast 等），对标 NVIDIA 的 **NCCL**
- [ ] **集群**：Atlas 900 训练集群，配套节点间高速互联（如 HCCS / RoCE），支撑分布式训练、大模型并行
- [ ] **推理服务化**：
  - **MindX（昇腾应用软件开发套件）**：提供 `mxVision`（视觉/推理）、`mxBase`（基础能力）等，加速落地
  - 常见做法：ONNX/PyTorch 模型 → 封装成服务 → gRPC / HTTP 对外提供推理接口

> 记法：单卡看算子，多卡看 HCCL，集群看互联，服务看 MindX/自建。这是从"跑通"到"生产"的升维路径。

---

## 8. 开发与调优工具链

- [ ] **MindStudio**：昇腾官方集成开发环境（IDE），覆盖算子开发、模型转换、调试、性能分析、部署
  - 功能：工程管理、模型转换向导、算子开发、**profiling 性能分析**、精度比对、可视化部署
- [ ] **算子开发**：模型缺算子或要自研时用。
  - **TBE（Tensor Boost Engine）**：基于 TVM 的算子开发 DSL，用 Python 写算子（类 TVM），生成 AI Core 可执行指令
  - 也可用 **AscendC**（C++ 编程接昇腾，贴近硬件）写高性能算子
- [ ] **性能分析**：
  - **Profiling 工具**：看算子耗时、Host/Device 侧耗时、数据搬运开销
  - 常用优化点：算子融合、减少 Host↔Device 拷贝、充分利用**异构流水**（计算与搬运并行）
- [ ] **精度比对**：把昇腾输出与 GPU/CPU 参考输出逐层对比，定位精度损失（混合精度/F16 常见）

> 记法：MindStudio 是"大脑"，TBE/AscendC 是"写算子"，Profiling 是"找瓶颈"，精度比对是"保正确"。

---

## 9. 典型落地流程（把整条链串起来）

- [ ] **1. 准备模型**：PyTorch/TF 训练好模型，导出 ONNX（或直接用框架 + 适配插件）
- [ ] **2. 迁移到昇腾**：
  - 推理：ONNX → ATC → `.om`
  - 训练：`torch_npu` 把 `.to("npu")`，或用 MindSpore Ascend 后端
- [ ] **3. 跑通**：AscendCL 加载 `.om`（底层）或框架直接推理（高层）；检查精度
- [ ] **4. 优化**：算子融合、混合精度、减少搬运；Profiling 找热点
- [ ] **5. 部署**：单卡/多卡/集群；服务化（MindX / 自建接口）；监控与重试
- [ ] **6. 生产**：模型版本管理、推理服务高可用、性能基准测试

> 一句话：**训练模型 → 转换/迁移到昇腾 → 跑通 + 调优 → 服务化**。学昇腾就是沿这条链把每层摸一遍。

---

## 学习路径建议

| 阶段 | 内容 | 目标 | 状态 |
|---|---|---|---|
| 1 | 硬件与架构 | 搞懂 310/910、达芬奇、Atlas | ☐ |
| 2 | CANN 分层 | 建立软件栈全景心智模型 | ☐ |
| 3 | AscendCL | 会用 C 接口加载 `.om` 推理 | ☐ |
| 4 | 框架适配 | 会用 `torch_npu` / MindSpore 跑模型 | ☐ |
| 5 | 模型转换 | 会用 ATC 转 `.om` | ☐ |
| 6 | 多卡/集群 | 理解 HCCL 与分布式 | ☐ |
| 7 | 工具链 | MindStudio、TBE/AscendC、Profiling | ☐ |

---

## 待填充内容（后续逐节展开）

1. 每节的详细讲解 + 代码示例（按当前框架顺序推进）
2. AscendCL 最小可运行示例（`aclInit` → 加载 `.om` → 推理完整流程）
3. `torch_npu` 从 CUDA 迁移的具体差异与算子适配清单
4. ATC 转换参数详解与常见报错
5. HCCL/分布式训练入门（多卡 AllReduce 示例）

> 告诉我"从第 X 节开始填充"或"填充某节"，我会按框架逐节写内容。
