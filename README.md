# Kiro AgentCore MCP Builder Skill

一个 [Kiro](https://kiro.dev) Skill，帮你一键将任意第三方 API 封装为 MCP Server 并部署到 [AWS Bedrock AgentCore](https://aws.amazon.com/bedrock/agentcore/)，最终在 Amazon Q Business (Quick Suite) 中通过 Chat 或 Flows 调用。

> 基于 [领星 ERP MCP Server](https://github.com/quanquan1996/lingxing-erp-mcp-agentcore) 项目的实战经验提炼。

---

## 这个 Skill 能做什么

当你在 Kiro 中激活这个 Skill 后，它会引导你完成从 API 接入到生产部署的完整流程：

```
API 文档 → MCP 代码生成 → AWS 资源配置 → 部署 → 测试 → 输出部署文档
```

具体来说，6 个阶段：

| 阶段 | 做什么 | 产出 |
|------|--------|------|
| 1. API 分析 | 阅读 API 文档，确认封装范围 | 接口分析摘要 |
| 2. 代码生成 | 生成 API 客户端 + MCP Server | `api_client.py` `mcp_server.py` `requirements.txt` `Dockerfile` |
| 3. AWS 配置 | 创建 Cognito、S3、IAM 策略 | `.bedrock_agentcore.yaml` + 策略文件 |
| 4. 部署 | 执行 agentcore deploy | 运行中的 MCP 端点 |
| 5. 测试 | 本地 API 测试 + 端点测试 | 验证通过的工具列表 |
| 6. 文档输出 | 生成部署信息文档 | `deployment-info.md`（可直接用于 Quick Suite 配置） |

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

## 安装

### 方式一：复制到你的项目（推荐）

将 skill 文件复制到你项目的 `.kiro/skills/` 目录下：

```bash
# 在你的项目根目录执行
mkdir -p .kiro/skills
curl -o .kiro/skills/agentcore-mcp-builder.md \
  https://raw.githubusercontent.com/quanquan1996/kiro-agentcore-mcp-skill/main/.kiro/skills/agentcore-mcp-builder.md
```

### 方式二：全局安装

将 skill 文件复制到 Kiro 的全局 skills 目录，所有项目都能使用：

```bash
# Windows
mkdir %USERPROFILE%\.kiro\skills 2>nul
curl -o %USERPROFILE%\.kiro\skills\agentcore-mcp-builder.md ^
  https://raw.githubusercontent.com/quanquan1996/kiro-agentcore-mcp-skill/main/.kiro/skills/agentcore-mcp-builder.md

# macOS / Linux
mkdir -p ~/.kiro/skills
curl -o ~/.kiro/skills/agentcore-mcp-builder.md \
  https://raw.githubusercontent.com/quanquan1996/kiro-agentcore-mcp-skill/main/.kiro/skills/agentcore-mcp-builder.md
```

## 使用

在 Kiro 聊天中输入 `#agentcore-mcp-builder` 引用这个 Skill，然后告诉 Kiro 你想接入什么 API：

```
#agentcore-mcp-builder

我想把 XXX 的 API 封装成 MCP Server 部署到 AgentCore。
API 文档在这里：[文档链接或文件路径]
我需要封装这些功能：查询订单、生成报表...
```

Kiro 会按照 Skill 中定义的 6 个阶段，逐步引导你完成整个流程。

## 前置条件

- [Kiro](https://kiro.dev) IDE
- AWS 账号（开通 Bedrock AgentCore、Cognito、S3）
- Python 3.10+
- `agentcore` CLI：`pip install bedrock-agentcore-starter-toolkit`
- Windows 额外需要：`choco install zip -y`

## 内置的踩坑经验

这个 Skill 内置了实战中遇到的所有坑和解决方案：

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
.kiro/
└── skills/
    └── agentcore-mcp-builder.md    # Skill 主文件（530 行）
```

## 相关项目

- [领星 ERP MCP Server](https://github.com/quanquan1996/lingxing-erp-mcp-agentcore) — 使用此 Skill 构建的实际项目
- [Kiro](https://kiro.dev) — AI IDE
- [AWS Bedrock AgentCore](https://aws.amazon.com/bedrock/agentcore/) — MCP Server 托管平台
- [Amazon Q Business](https://aws.amazon.com/q/business/) — Quick Suite 所在平台

## License

MIT
