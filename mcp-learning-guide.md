# MCP（Model Context Protocol）学习指南

> 用途：从零学习 MCP 的路线图 + 要点笔记。每学完一项勾选 `[ ]`，需要展开某节可随时让我补充。
> 背景贴合：你正在用 Go + DeepSeek Harness（DSH，本机已内置 MCP 客户端组件 dsh-mcp-client），文档会结合 Go SDK 示例。

---

## 1. MCP 是什么、解决什么问题

- [ ] 痛点：LLM 应用要接外部数据/工具/系统，传统做法是每个应用写一套私有集成（function calling、自定义 API），不可复用
- [ ] MCP = **Model Context Protocol**，Anthropic 于 2024-11 提出的**开放标准**，把"AI 应用 ↔ 外部能力"的连接方式标准化
- [ ] 两个类比：
  - MCP 之于 AI 应用 ≈ **USB-C** 之于外设：一个标准接口，插上即用
  - MCP 之于 LLM 生态 ≈ **LSP**（Language Server Protocol）之于 IDE 生态
- [ ] 对比 function calling：function calling 是单模型单应用的私有约定；MCP 是跨厂商、跨语言、可复用的通用协议（你的模型可以"发现并调用"任何 MCP 服务器提供的工具）

> 一句话：**MCP = AI 世界的标准化"插件接口"**，一端是模型/应用（Host），一端是能力提供方（Server）。

---

## 2. 核心架构

- [ ] 三个角色：
  - **Host**：LLM 应用/客户端（如 Claude Desktop、Cursor、**DSH**）——决定允许哪些 Server 接入
  - **Client**：Host 内为**每个** Server 创建的一个连接（1:1）
  - **Server**：暴露能力的服务（工具/资源/提示词），轻量进程，可本地可远程
- [ ] 连接拓扑：`Host → (Client₁ → Server₁, Client₂ → Server₂, ...)`，模型不直接连 Server
- [ ] 与 C++ 类比：Host ≈ 主程序，Client ≈ 连接句柄（fd），Server ≈ 动态库/子进程服务

---

## 3. 原语（Primitives）——MCP 的内容三件套

- [ ] **Tools（工具）**：模型主动调用的可执行函数（如"查询天气""执行 SQL"）
  - 特征：用户**显式授权**后才可调用；执行有副作用
  - 定义含 `name`、`description`、`inputSchema`（JSON Schema 描述入参）
- [ ] **Resources（资源）**：只读数据/内容，类似"暴露给模型的文件系统"
  - 特征：由应用控制（用户同意才读）；用 URI 标识（如 `file://`、`db://`），支持模板 `resourceTemplate`
- [ ] **Prompts（提示词）**：可复用的提示词模板（参数化），由**用户**触发，不是模型调用
  - 类似"预制 prompt 函数"
- [ ] （2025-11-25 起新增）**Tasks**：把多步工作流/代理任务抽象为可复用的任务单元
- [ ] 辅助能力（了解）：Sampling（Server 反向请求模型生成）、Roots（Server 请求访问文件路径）、Logging、Completion、Pagination

> 记忆口诀：**Tools = 手（能做事）、Resources = 眼睛（能看数据）、Prompts = 剧本（预先写好的指令）。**

---

## 4. 通信机制

- [ ] 底层协议：**JSON-RPC 2.0**（request / response / notification 三种消息）
- [ ] 传输方式（Transport）：
  - **stdio**：Server 作为子进程，通过标准输入/输出通信（本地开发最常用，DSH/Claude Desktop 默认方式）
  - **Streamable HTTP**（2025-06-18 起替代旧 HTTP+SSE）：远程 Server 用，支持长连接流式返回
- [ ] 生命周期（握手流程）：
  1. Client 发 `initialize`（带协议版本 + 能力声明）
  2. Server 回响应（带自己的版本 + 能力）
  3. Client 发 `notifications/initialized` 通知
  4. 进入正常操作（列工具/调工具/读资源…）
- [ ] **能力协商**：双方声明 capabilities，比如"我支持工具调用""我支持资源订阅"，不支持的不承诺
- [ ] 协议版本演进（学习时知道差异即可）：

| 版本 | 要点 |
|---|---|
| 2024-11-05 | 初版：stdio、Tools/Resources/Prompts |
| 2025-03-26 | 修订：流式、采样等细节 |
| 2025-06-18 | **Streamable HTTP** 取代 HTTP+SSE；远程 Server 引入 **OAuth 2.1** 授权 |
| 2025-11-25 | 新增 **Tasks** 抽象 |
| 2026-07-28 | 迄今最大更新：转向**无状态（stateless）核心**，简化会话语义；官方 Go SDK 跟进 |

---

## 5. 动手：写一个最小 MCP Server（Go）

> 前置：已装 Go（≥1.21）。社区最常用 Go SDK：`github.com/mark3labs/mcp-go`；官方 SDK `github.com/modelcontextprotocol/go-sdk`（新，随 2026-07-28 规范演进，API 仍在变动，学习期二选一即可）。

- [ ] 建项目并引入 SDK：
  ```bash
  mkdir mcp-demo && cd mcp-demo
  go mod init mcp-demo
  go get github.com/mark3labs/mcp-go
  ```
- [ ] 最小 server（一个 echo 工具）：
  ```go
  package main

  import (
      "context"
      "github.com/mark3labs/mcp-go/mcp"
      "github.com/mark3labs/mcp-go/server"
  )

  func main() {
      s := server.NewMCPServer("demo", "0.1.0") // 名称 + 版本

      // 注册工具：echo 一个字符串参数
      s.AddTool(mcp.NewTool("echo",
          mcp.WithDescription("原样返回你输入的内容"),
          mcp.WithString("message", mcp.Required()),
      ), handleEcho)

      // 通过 stdio 启动（供 Host 以子进程方式连接）
      if err := server.ServeStdio(s); err != nil {
          panic(err)
      }
  }

  func handleEcho(ctx context.Context, req mcp.CallToolRequest) (*mcp.CallToolResult, error) {
      msg, _ := req.Params.Arguments["message"].(string)
      return mcp.NewToolResultText("echo: " + msg), nil
  }
  ```
- [ ] 跑起来 & 测试：
  ```bash
  go run .
  ```
  启动后无输出是正常的（它在等 stdin 上的 JSON-RPC 消息）。
- [ ] 用 **MCP Inspector** 图形化调试（推荐）：
  ```bash
  npx @modelcontextprotocol/inspector go run .
  ```
  或本机 DSH：在设置里添加一个 stdio 类型的 MCP 服务器，命令填 `go run .`，即可在会话中调用 `echo` 工具。

> 理解要点：你写的其实是一个"JSON-RPC 服务"：客户端发 `tools/list` 得到工具清单，发 `tools/call`（带参数）得到执行结果。SDK 帮你封装了这些。

---

## 6. 原语深入与最佳实践

- [ ] Tool 定义要点：
  - `inputSchema` 用 JSON Schema（类型、必填、枚举、默认值）
  - 描述写清楚，模型靠 description 决定何时调用
  - 标注（annotations）：`readOnlyHint`（只读）、`destructiveHint`（危险，需二次确认）、`idempotentHint`（可重试）、`openWorldHint`
- [ ] Resource：注册 `resources/list` 与 `resources/read`；带参数的资源用 `resourceTemplate`（如 `file://{path}`）
- [ ] Prompt：`prompts/list` + `prompts/get`，参数化模板返回给客户端展示/填充
- [ ] 错误处理：返回结构化错误信息而不是崩溃；工具执行失败要返回可读的 error message 给模型
- [ ] 安全底线：
  - 危险操作（删库、写文件、执行命令）标注 `destructiveHint` 并让 Host 侧确认
  - 输入校验：MCP 不替你校验，参数必须自己检查
  - 最小权限：Server 只暴露必要能力；远程 Server 用 OAuth 2.1 授权，别用裸 API Key
  - 资源 URI 做白名单/路径规范化，防目录穿越

---

## 7. 客户端侧集成（进阶）

- [ ] 在应用里做 **Client**：SDK 提供 `NewClient`，可编程地连接某个 Server 并调用其工具（测试 Server 时常用）
- [ ] 传输差异：stdio 需要能启动子进程的 Host；浏览器端只能用 Streamable HTTP
- [ ] 你当前环境：DSH 已安装 MCP 客户端组件（`dsh-mcp-client`），可以直接把自建 Server 接入本机会话使用，是练手的好去处

---

## 8. 学习路线

| 阶段 | 内容 | 预计投入 | 状态 |
|---|---|---|---|
| 1 | 概念：读完本文 1-4 节，能讲清 Host/Client/Server 与三大原语 | 0.5 天 | ☐ |
| 2 | 动手：跑通 §5 的 Go 示例，用 Inspector 调通 | 0.5-1 天 | ☐ |
| 3 | 实践：给 Server 加一个 Resource 和一个带参数 Tool（如文件读取） | 1 天 | ☐ |
| 4 | 进阶：把某个现有 API/脚本包成 MCP Server（如天气、数据库查询） | 1-2 天 | ☐ |
| 5 | 深化：读官方规范对应章节（versioning、transports、authorization） | 按需 | ☐ |
| 6 | 拓展：看官方 servers 仓库优秀实现、学习 Tasks/OAuth 远程部署 | 按需 | ☐ |

- [ ] 阶段练习建议：做一个"根据城市名返回天气"的 Server（用免费天气 API 或本地 mock），并接进 DSH 用自然语言调用——这是最典型的 MCP 入门验收题

---

## 9. 参考资料

- 官方规范（含中文版）：[modelcontextprotocol.io](https://modelcontextprotocol.io) / [spec.modelcontextprotocol.io](https://spec.modelcontextprotocol.io)
- 版本历史：[versioning 文档](https://modelcontextprotocol.io/docs/learn/versioning.md)
- 2026-07-28 最新版说明：[官方博客](https://blog.modelcontextprotocol.io/posts/2026-07-28/) / [changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog)
- 官方仓库：[modelcontextprotocol 组织（spec、servers、SDK）](https://github.com/modelcontextprotocol)
- Go SDK：社区 [mark3labs/mcp-go](https://pkg.go.dev/github.com/mark3labs/mcp-go) / 官方 [go-sdk](https://github.com/modelcontextprotocol/go-sdk)
- 生态列表：Awesome MCP（GitHub 搜索 "awesome-mcp"）
- 调试工具：MCP Inspector（`npx @modelcontextprotocol/inspector`）

---

## 待填充内容（按需展开）

1. §5 扩写：完整可运行的多工具 Server 项目
2. §6 扩写：Tool/Resource/Prompt 的完整 JSON Schema 示例
3. 远程部署：Streamable HTTP + OAuth 2.1 实战
4. DSH 接入 MCP 的具体配置步骤（结合本机环境实测）

> 告诉我"展开第 X 节"或"帮我做一个 XX 的 MCP Server"，我按需补充。
