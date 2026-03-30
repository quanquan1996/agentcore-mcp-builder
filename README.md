# AgentCore MCP Builder

将任意第三方 API 封装为 MCP Server，部署到 [AWS Bedrock AgentCore](https://aws.amazon.com/bedrock/agentcore/)，通过 Amazon Q Business (Quick Suite) 的 Chat / Flows 调用。

这是一份结构化的构建与部署指南，可以作为：

- **AI 编程助手的 Prompt / Rule / Skill** — 粘贴给 Kiro、Cursor、Windsurf、Claude、ChatGPT 等任何 AI 工具
- **人工操作手册** — 按步骤手动执行
- **团队知识库** — 标准化 MCP Server 的开发部署流程

> 基于 [领星 ERP MCP Server](https://github.com/quanquan1996/lingxing-erp-mcp-agentcore) 项目的实战经验提炼。

---

## 架构

```
用户 → Quick Suite (Chat/Flows)
        ↓ OAuth Token
   Cognito (Client Credentials Grant)
        ↓ Bearer Token
   AgentCore Runtime (MCP Server)
        ↓ API 调用
   业务 API (第三方 OpenAPI)
        ↓ 文件报告（可选）
   S3 (预签名 URL 下载)
```

## 覆盖的完整流程

| 阶段 | 做什么 | 产出 |
|------|--------|------|
| 1. API 分析 | 阅读 API 文档，确认封装范围 | 接口分析摘要 |
| 2. 代码生成 | 生成 API 客户端 + MCP Server | `api_client.py` `mcp_server.py` `requirements.txt` `Dockerfile` |
| 3. AWS 配置 | 创建 Cognito、S3、IAM 策略 | `.bedrock_agentcore.yaml` + 策略文件 |
| 4. 部署 | 执行 agentcore deploy | 运行中的 MCP 端点 |
| 5. 测试 | 本地 API 测试 + 端点测试 | 验证通过的工具列表 |
| 6. 文档输出 | 生成部署信息文档 | `deployment-info.md`（可直接用于 Quick Suite 配置） |

详细内容见 [agentcore-mcp-builder.md](./agentcore-mcp-builder.md)。

## 使用方式

### 作为 AI 助手的 Prompt

将 `agentcore-mcp-builder.md` 的内容粘贴到你使用的 AI 工具中：

| 工具 | 用法 |
|------|------|
| **Kiro** | 复制到 `.kiro/skills/` 目录，或粘贴到 Steering 文件 |
| **Cursor** | 复制到 `.cursor/rules/` 目录，或粘贴到 Project Rules |
| **Windsurf** | 复制到 `.windsurfrules` 或粘贴到 Rules |
| **Claude** | 粘贴到 Project Knowledge 或直接作为对话上下文 |
| **ChatGPT** | 粘贴到 Custom Instructions 或对话开头 |
| **其他** | 作为 System Prompt 或上下文文档使用 |

然后告诉 AI：

```
我想把 XXX 的 API 封装成 MCP Server 部署到 AgentCore。
API 文档在这里：[文档链接或文件路径]
我需要封装这些功能：查询订单、生成报表...
```

### 作为人工操作手册

直接阅读 [agentcore-mcp-builder.md](./agentcore-mcp-builder.md)，按 6 个阶段的 Checklist 逐步执行。

## 前置条件

- AWS 账号（开通 Bedrock AgentCore、Cognito、S3）
- Python 3.10+
- `agentcore` CLI：`pip install bedrock-agentcore-starter-toolkit`
- Windows 额外需要：`choco install zip -y`

## 内置的踩坑经验

这份指南内置了实战中遇到的所有坑和解决方案：

| 问题 | 解决 |
|------|------|
| Windows YAML 编码错误 | YAML 中不写中文，全用 ASCII |
| AgentCore `/var/` 只读 | 在 import 前 patch `logging.FileHandler` |
| Cognito JWT 验证失败 | 用 `allowedClients` 而不是 `allowedAudience` |
| PowerShell JSON 转义 | JSON 写文件，用 `file://` 引用 |
| AgentCore 无状态 | 文件生成和上传在同一 tool 调用中完成 |
| Windows 缺少 zip | `choco install zip -y` |

## 文件结构

```
├── agentcore-mcp-builder.md   # 核心指南（530 行）
├── README.md                  # 本文件
├── LICENSE                    # MIT
└── .gitignore
```

## 相关项目

- [领星 ERP MCP Server](https://github.com/quanquan1996/lingxing-erp-mcp-agentcore) — 使用此指南构建的实际项目
- [AWS Bedrock AgentCore](https://aws.amazon.com/bedrock/agentcore/) — MCP Server 托管平台
- [Amazon Q Business](https://aws.amazon.com/q/business/) — Quick Suite 所在平台

## License

MIT
