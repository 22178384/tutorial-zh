# 01 · MCP 是什么、怎么用

MCP（Model Context Protocol）是一套开放协议，让 AI 应用统一地接入外部工具与数据。

## 一、为什么需要它
没有 MCP 时，每个 AI 应用都要为每一个外部系统单独写适配代码，重复且脆弱。
MCP 把「AI 应用」和「工具提供方」解耦：应用只认协议，工具方按协议暴露能力。

## 二、三个角色
1. **Host**：用户直接使用的 AI 客户端（如 WorkBuddy）。
2. **Client**：Host 内部与某个 Server 通信的桥接模块。
3. **Server**：真正提供能力的服务（搜索、数据库、文件系统…）。

## 三、一次典型调用
1. Host 启动 Client，连接到 Server。
2. `initialize` 握手，约定协议版本。
3. `tools/list` 拿到可用工具清单。
4. 模型决定调用某个工具，`tools/call` 带上参数。
5. Server 返回结果，模型据此继续回答。

## 四、动手试
见 [@c991china/anysearch-mcp-guide](https://github.com/c991china/anysearch-mcp-guide)——
它用 AnySearch 做完整示例，附可运行代码。
