# AgentCore MCP Server 构建与部署 Skill

> 将任意第三方 API 封装为 MCP Server，部署到 AWS Bedrock AgentCore，通过 Quick Suite 使用。
> 基于领星 ERP MCP 项目的实战经验提炼，覆盖从 API 分析到生产部署的完整流程。

---

## 适用场景

用户希望将一个业务 API（如 ERP、CRM、电商平台等）封装为 MCP Server 并部署到 AWS Bedrock AgentCore，最终在 Amazon Q Business (Quick Suite) 中通过 Chat 或 Flows 调用。

---

## 整体架构

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

---

## 流程总览（6 个阶段）

| 阶段 | 输入 | 输出 | 关键动作 |
|------|------|------|----------|
| 1. API 分析 | API 文档 URL/文件 | 接口理解 + 需求确认 | 阅读文档，与用户确认封装范围 |
| 2. MCP 代码生成 | 接口文档 + 需求 | api_client.py + mcp_server.py + requirements.txt + Dockerfile | 生成代码 |
| 3. 配置生成 | AWS 账号信息 | .bedrock_agentcore.yaml + 策略文件 + example 模板 | 配置 AWS 资源 |
| 4. 部署 | 代码 + 配置 + 凭证 | 运行中的 AgentCore Runtime | agentcore deploy |
| 5. 测试调试 | MCP 端点 + Cognito 凭证 | 验证通过的工具列表 | 测试脚本 |
| 6. 输出文档 | 部署信息 | deployment-info.md | 用户可直接用于 Quick Suite 配置 |

---

## 阶段 1：API 分析与需求确认

### 触发 Prompt

用户会说类似：
- "我想接入 XXX 的 API，封装成 MCP Server 部署到 AgentCore"
- "帮我把这个 API 文档封装成 MCP 工具"

### 你需要做的

1. 向用户询问以下信息（如果没有提供）：
   - API 接口文档的 URL 或文件路径
   - 希望封装哪些接口 / 实现哪些功能
   - API 的认证方式（API Key、OAuth、签名等）
   - 是否需要生成报告文件（Excel/CSV）并上传 S3

2. 阅读并理解 API 文档，重点关注：
   - 认证流程（Token 获取、刷新、签名规则）
   - 核心业务接口（路径、参数、返回值）
   - 限流规则（令牌桶容量、频率限制）
   - 错误码定义
   - 分页机制

3. 输出接口分析摘要，与用户确认封装范围

---

## 阶段 2：MCP 代码生成

### 文件结构

在 `mcp-server/` 目录下生成以下文件：

```
mcp-server/
├── mcp_server.py              # MCP Server 主入口
├── {service}_client.py        # 业务 API 客户端
├── requirements.txt           # Python 依赖
├── Dockerfile                 # 容器镜像（备用）
├── .bedrock_agentcore.yaml    # AgentCore 部署配置（阶段 3 生成）
└── .bedrock_agentcore.example.yaml  # 配置模板（不含敏感信息）
```

### 2.1 API 客户端 (`{service}_client.py`)

必须包含的模块：

```python
class ServiceClient:
    def __init__(self, api_key="", api_secret=""):
        # 从环境变量读取凭证
        self.api_key = api_key or os.environ.get("SERVICE_API_KEY", "")
        self.api_secret = api_secret or os.environ.get("SERVICE_API_SECRET", "")
        self.access_token = None
        self.token_expires_at = 0
        self._http = httpx.Client(base_url=BASE_URL, timeout=30)

    def _ensure_token(self):
        """确保 token 有效，过期则刷新"""
        ...

    def _request(self, method, path, **kwargs):
        """统一请求方法，自动处理认证和错误"""
        self._ensure_token()
        # 添加公共参数（token、签名等）
        # 发送请求
        # 处理错误码
        ...

    # 业务方法
    def get_xxx(self, ...):
        return self._request("POST", "/api/xxx", json={...})
```

关键设计原则：
- 凭证从环境变量读取（部署时通过 `--env` 注入）
- Token 自动续约（处理过期和刷新）
- 统一错误处理（区分认证错误、权限错误、限流、参数错误）
- 签名逻辑封装在客户端内部

### 2.2 MCP Server (`mcp_server.py`)

必须遵循的模板结构：

```python
"""
{Service Name} MCP Server
部署到 Amazon Bedrock AgentCore，通过 Quick Suite 调用
"""
import os, sys, json, logging
from datetime import datetime, timedelta

# ============================================================
# STEP 1: Patch logging（AgentCore /var/ 只读，必须在所有 import 之前）
# ============================================================
_original_fh_init = logging.FileHandler.__init__

def _patched_fh_init(self, filename, mode="a", encoding=None, delay=False, errors=None):
    if filename.startswith("/var/"):
        filename = "/tmp/" + os.path.basename(filename)
    _original_fh_init(self, filename, mode, encoding, delay, errors)

logging.FileHandler.__init__ = _patched_fh_init

# ============================================================
# STEP 2: 环境变量
# ============================================================
os.environ.setdefault("EXCEL_FILES_PATH", "/tmp/excel_files")
os.makedirs(os.environ["EXCEL_FILES_PATH"], exist_ok=True)

S3_BUCKET = os.environ.get("SERVICE_S3_BUCKET", "")
S3_PREFIX = os.environ.get("SERVICE_S3_PREFIX", "reports/")

# ============================================================
# STEP 3: 创建 MCP Server（必须 0.0.0.0:8000 + stateless_http）
# ============================================================
from mcp.server.fastmcp import FastMCP

mcp = FastMCP(
    name="{service}-mcp",
    host="0.0.0.0",
    port=8000,
    stateless_http=True,
)

_client = None
def _get_client():
    global _client
    if _client is None:
        from {service}_client import ServiceClient
        _client = ServiceClient()
    return _client

# ============================================================
# STEP 4: 工具定义（@mcp.tool() 装饰器）
# ============================================================
@mcp.tool()
def get_xxx(...) -> str:
    """工具描述（Quick Suite 会展示给用户）"""
    try:
        result = _get_client().get_xxx(...)
        return json.dumps(result, ensure_ascii=False, indent=2)
    except Exception as e:
        return json.dumps({"success": False, "error": str(e)}, ensure_ascii=False)

# ============================================================
# STEP 5: Excel 报告生成 + S3 上传（如果需要）
# ============================================================
@mcp.tool()
def generate_xxx_report_excel(...) -> str:
    """生成 Excel 报告并上传 S3，返回预签名下载链接"""
    try:
        from openpyxl import Workbook
        import boto3
        from urllib.parse import quote
        # 1. 获取数据
        # 2. 创建 Excel（带格式）
        # 3. 保存到 /tmp/
        # 4. 上传 S3 + 生成预签名 URL
        # 5. 返回下载链接
        ...
    except Exception as e:
        return json.dumps({"success": False, "error": str(e)}, ensure_ascii=False)

# ============================================================
# STEP 6: 启动
# ============================================================
if __name__ == "__main__":
    mcp.run(transport="streamable-http")
```

### 2.3 requirements.txt

```
mcp[cli]>=1.0.0
fastmcp>=0.1.0
httpx>=0.27.0
boto3>=1.34.0
openpyxl>=3.1.0       # 如果需要 Excel 报告
pydantic>=2.0.0
# 根据 API 认证需要添加，如：
# pycryptodome>=3.20.0  # AES 加密签名
```

### 2.4 Dockerfile

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY {service}_client.py .
COPY mcp_server.py .
EXPOSE 8000
CMD ["python", "mcp_server.py"]
```


---

## 阶段 3：AWS 资源配置

### 3.1 获取账号信息

```bash
aws sts get-caller-identity
aws configure get region
```

### 3.2 查找 AgentCore Runtime Role

```bash
aws iam list-roles --query "Roles[?contains(RoleName, 'AgentCore')].{Name:RoleName,Arn:Arn}"
```

> 如果没有 Role，需要先在 AgentCore 控制台创建一个 Agent 让它自动生成 Role。

### 3.3 创建 Cognito 资源

创建 User Pool（如果没有）：
```bash
aws cognito-idp create-user-pool --pool-name {service}-mcp-pool --region <REGION>
```

配置 Domain（Client Credentials 必需）：
```bash
aws cognito-idp create-user-pool-domain --user-pool-id <POOL_ID> --domain "<UNIQUE_DOMAIN>" --region <REGION>
```

创建 Resource Server：
```bash
aws cognito-idp create-resource-server \
  --user-pool-id <POOL_ID> \
  --identifier "{service}-mcp" \
  --name "{Service} MCP Server" \
  --scopes "ScopeName=invoke,ScopeDescription=Invoke MCP tools" \
  --region <REGION>
```

创建 App Client：
```bash
aws cognito-idp create-user-pool-client \
  --user-pool-id <POOL_ID> \
  --client-name "{service}-mcp-agentcore" \
  --generate-secret \
  --allowed-o-auth-flows "client_credentials" \
  --allowed-o-auth-scopes "{service}-mcp/invoke" \
  --allowed-o-auth-flows-user-pool-client \
  --region <REGION>
```

> 记录返回的 ClientId 和 ClientSecret。

查看 Client Secret（如果忘记）：
```bash
aws cognito-idp describe-user-pool-client \
  --user-pool-id <POOL_ID> \
  --client-id <CLIENT_ID> \
  --query "UserPoolClient.ClientSecret" \
  --output text --region <REGION>
```

查看 Cognito Domain：
```bash
aws cognito-idp describe-user-pool --user-pool-id <POOL_ID> --query "UserPool.Domain" --output text
```

### 3.4 创建 S3 桶

报告桶（如果需要 Excel/文件生成）：
```bash
aws s3 mb s3://{service}-reports-<ACCOUNT_ID> --region <REGION>
```

CodeBuild 桶（agentcore deploy 需要）：
```bash
aws s3 mb s3://bedrock-agentcore-codebuild-<ACCOUNT_ID> --region <REGION>
```

### 3.5 给 AgentCore Role 添加权限

S3 权限策略 (`s3-role-policy.json`)：
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "S3ReportAccess",
    "Effect": "Allow",
    "Action": ["s3:PutObject", "s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::{service}-reports-<ACCOUNT_ID>",
      "arn:aws:s3:::{service}-reports-<ACCOUNT_ID>/*"
    ]
  }]
}
```

```bash
aws iam put-role-policy \
  --role-name <AGENTCORE_ROLE_NAME> \
  --policy-name "S3ReportAccess" \
  --policy-document file://s3-role-policy.json
```

VPC 权限（如果使用 VPC 模式）：
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "VPCNetworkAccess",
    "Effect": "Allow",
    "Action": [
      "ec2:CreateNetworkInterface", "ec2:DescribeNetworkInterfaces",
      "ec2:DeleteNetworkInterface", "ec2:AssignPrivateIpAddresses",
      "ec2:UnassignPrivateIpAddresses", "ec2:DescribeSubnets",
      "ec2:DescribeVpcs", "ec2:DescribeSecurityGroups"
    ],
    "Resource": "*"
  }]
}
```

### 3.6 生成 .bedrock_agentcore.yaml

实际配置文件（含真实值，gitignore）：

```yaml
default_agent: {service}_mcp

agents:
  {service}_mcp:
    name: {service}_mcp
    language: python
    entrypoint: mcp_server.py
    deployment_type: direct_code_deploy
    runtime_type: PYTHON_3_11
    platform: linux/arm64
    source_path: .
    aws:
      execution_role: arn:aws:iam::<ACCOUNT_ID>:role/service-role/<AGENTCORE_ROLE>
      execution_role_auto_create: false
      account: "<ACCOUNT_ID>"
      region: <REGION>
      s3_path: s3://bedrock-agentcore-codebuild-<ACCOUNT_ID>
      s3_auto_create: false
      network_configuration:
        network_mode: PUBLIC    # 或 VPC
        # VPC 模式需要：
        # network_mode: VPC
        # network_mode_config:
        #   security_groups: [sg-xxx]
        #   subnets: [subnet-xxx]
      protocol_configuration:
        server_protocol: MCP
      observability:
        enabled: true
    memory:
      mode: NO_MEMORY
    authorizer_configuration:
      customJWTAuthorizer:
        discoveryUrl: https://cognito-idp.<REGION>.amazonaws.com/<POOL_ID>/.well-known/openid-configuration
        allowedClients:
          - <COGNITO_CLIENT_ID>
```

Example 模板（不含敏感信息，提交到 Git）：

```yaml
# 同上结构，但所有值用 <PLACEHOLDER> 替代
```

### 3.7 .gitignore 规则

确保以下文件不提交到 Git：
```
mcp-server/.bedrock_agentcore.yaml
mcp-server/s3-policy.json
mcp-server/s3-role-policy.json
mcp-server/test_invoke.py
doc/deployment-info.md
```

---

## 阶段 4：部署

### 4.1 前置安装

```bash
# AgentCore CLI
pip install bedrock-agentcore-starter-toolkit

# Windows 必需
choco install zip -y
```

> 如果 agentcore 不在 PATH，手动添加：
> ```powershell
> $env:PATH = "C:\Users\<USER>\AppData\Local\Programs\Python\Python312\Scripts;" + $env:PATH
> ```

### 4.2 执行部署

在 `mcp-server/` 目录下执行：

```bash
agentcore deploy -a {service}_mcp \
  --auto-update-on-conflict \
  --env "SERVICE_API_KEY=xxx" \
  --env "SERVICE_API_SECRET=yyy" \
  --env "SERVICE_S3_BUCKET=bucket-name" \
  --env "SERVICE_S3_PREFIX=reports/"
```

### 4.3 验证部署状态

```bash
agentcore status
```

---

## 阶段 5：测试调试

### 5.1 本地 API 测试 (`test_local_all_apis.py`)

直接调用业务 API，验证认证和接口可用性：

```python
"""本地测试所有 API 接口"""
import os, sys, json
sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from {service}_client import ServiceClient

client = ServiceClient()

# 测试每个接口，分类错误类型
tests = [
    ("接口名称", lambda: client.get_xxx(...)),
    ...
]

for name, fn in tests:
    try:
        result = fn()
        code = result.get("code", "N/A")
        print(f"  {name}: code={code}")
    except Exception as e:
        print(f"  {name}: ERROR - {e}")
```

### 5.2 AgentCore 端点测试 (`test_invoke.py`)

通过 Cognito Token 调用已部署的 MCP 端点：

```python
"""Test invoking the deployed MCP server via AgentCore endpoint"""
import asyncio, boto3, base64, requests

REGION = "<REGION>"
USER_POOL_ID = "<USER_POOL_ID>"
CLIENT_ID = "<COGNITO_CLIENT_ID>"
AGENT_ARN = "arn:aws:bedrock-agentcore:<REGION>:<ACCOUNT_ID>:runtime/<AGENT_ID>"
COGNITO_DOMAIN = "<COGNITO_DOMAIN>"

# 1. 获取 Cognito Token
idp = boto3.client("cognito-idp", region_name=REGION)
client_info = idp.describe_user_pool_client(UserPoolId=USER_POOL_ID, ClientId=CLIENT_ID)
client_secret = client_info["UserPoolClient"]["ClientSecret"]

token_url = f"https://{COGNITO_DOMAIN}.auth.{REGION}.amazoncognito.com/oauth2/token"
auth_b64 = base64.b64encode(f"{CLIENT_ID}:{client_secret}".encode()).decode()
resp = requests.post(token_url,
    headers={"Content-Type": "application/x-www-form-urlencoded", "Authorization": f"Basic {auth_b64}"},
    data={"grant_type": "client_credentials", "scope": "{service}-mcp/invoke"})
access_token = resp.json()["access_token"]

# 2. 构建 MCP URL
encoded_arn = AGENT_ARN.replace(":", "%3A").replace("/", "%2F")
mcp_url = f"https://bedrock-agentcore.{REGION}.amazonaws.com/runtimes/{encoded_arn}/invocations?qualifier=DEFAULT"

# 3. 用 MCP 客户端连接
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client

async def main():
    headers = {"authorization": f"Bearer {access_token}", "Content-Type": "application/json"}
    async with streamablehttp_client(mcp_url, headers, timeout=120, terminate_on_close=False) as (r, w, _):
        async with ClientSession(r, w) as session:
            await session.initialize()
            tools = await session.list_tools()
            print(f"Available tools ({len(tools.tools)}):")
            for t in tools.tools:
                print(f"  - {t.name}: {t.description[:60]}")

asyncio.run(main())
```

### 5.3 Raw HTTP 测试 (`test_raw.py`)

不依赖 MCP SDK，直接用 JSON-RPC over HTTP 测试：

```python
# Initialize
requests.post(url, headers=hdrs, json={
    "jsonrpc": "2.0", "id": 1, "method": "initialize",
    "params": {"protocolVersion": "2024-11-05", "capabilities": {},
               "clientInfo": {"name": "test", "version": "1"}}
}, timeout=60)

# Initialized notification
requests.post(url, headers=hdrs, json={
    "jsonrpc": "2.0", "method": "notifications/initialized"
}, timeout=30)

# List tools
requests.post(url, headers=hdrs, json={
    "jsonrpc": "2.0", "id": 2, "method": "tools/list", "params": {}
}, timeout=60)

# Call tool
requests.post(url, headers=hdrs, json={
    "jsonrpc": "2.0", "id": 3, "method": "tools/call",
    "params": {"name": "get_xxx", "arguments": {...}}
}, timeout=120)
```

### 5.4 常见问题排查

| 问题 | 原因 | 解决 |
|------|------|------|
| YAML UnicodeDecodeError | Windows cp1252 编码 | YAML 中不写中文，全用 ASCII |
| zip utility not found | Windows 缺少 zip | `choco install zip -y` |
| /var/ write failure | AgentCore 只读 | Patch logging.FileHandler（见代码模板） |
| JWT validation failed | 用了 allowedAudience | 改用 allowedClients |
| PowerShell JSON 转义 | 特殊字符问题 | JSON 写文件，用 `file://` 引用 |
| agentcore not found | 不在 PATH | 手动添加 Python Scripts 到 PATH |
| Token 过期 | access_token 2h 有效 | 客户端自动续约逻辑 |
| 文件找不到 | AgentCore 无状态 | 生成+上传在同一 tool 调用中完成 |

---

## 阶段 6：输出部署信息文档

### deployment-info.md 模板

部署完成后，生成 `doc/deployment-info.md` 供用户配置 Quick Suite：

```markdown
# Deployment Info

## AgentCore MCP Server

- Agent Name: {service}_mcp
- Agent ARN: arn:aws:bedrock-agentcore:<REGION>:<ACCOUNT_ID>:runtime/<AGENT_ID>
- Endpoint: DEFAULT (READY)
- Region: <REGION>
- Protocol: MCP (Streamable HTTP)

## MCP Endpoint URL

https://bedrock-agentcore.<REGION>.amazonaws.com/runtimes/<ENCODED_AGENT_ARN>/invocations?qualifier=DEFAULT

## Cognito OAuth

- User Pool ID: <POOL_ID>
- Client ID: <CLIENT_ID>
- Client Secret: <CLIENT_SECRET>
- Token URL: https://<COGNITO_DOMAIN>.auth.<REGION>.amazoncognito.com/oauth2/token
- Scope: {service}-mcp/invoke
- Grant Type: client_credentials

## S3

- Bucket: {service}-reports-<ACCOUNT_ID>
- Prefix: reports/

## Available MCP Tools

1. tool_name_1 - 描述
2. tool_name_2 - 描述
...

## Quick Suite Configuration

1. Quick Suite → Integrations → Actions → 添加 MCP Server
2. 填入上方 MCP Endpoint URL
3. 认证方式：服务身份验证 → 基于用户的自定义 OAuth
4. 填写 Client ID、Client Secret、Token URL、Scope
5. 创建 Flows 编排业务流程
```

同时生成 `doc/deployment-info.example.md`（用占位符替代敏感值，可提交 Git）。

---

## 关键设计原则

1. **凭证不硬编码**：所有 API 凭证通过环境变量注入，部署时用 `--env` 传入
2. **敏感文件不入库**：`.bedrock_agentcore.yaml`、`test_invoke.py`、`deployment-info.md` 等含真实值的文件加入 `.gitignore`，只提交 `.example` 模板
3. **AgentCore 约束**：
   - `/var/` 只读 → 日志重定向到 `/tmp/`
   - 无状态 → 文件生成和上传在同一请求内完成
   - 必须 `0.0.0.0:8000` + `stateless_http=True`
4. **Cognito 认证**：使用 Client Credentials Grant + `allowedClients`（不是 `allowedAudience`）
5. **错误处理**：每个 tool 用 try/except 包裹，返回结构化 JSON 错误信息
6. **S3 报告**：使用预签名 URL，支持中文文件名（`Content-Disposition` + URL encode）

---

## 快速复制 Checklist

新项目接入时，按以下顺序执行：

- [ ] 阅读 API 文档，确认封装范围
- [ ] 生成 `{service}_client.py`（认证 + 业务方法）
- [ ] 生成 `mcp_server.py`（工具定义 + S3 集成）
- [ ] 生成 `requirements.txt` 和 `Dockerfile`
- [ ] 确认 AWS 账号有 AgentCore Runtime Role
- [ ] 创建/复用 Cognito User Pool + Resource Server + App Client
- [ ] 创建 S3 报告桶 + CodeBuild 桶
- [ ] 给 AgentCore Role 添加 S3 权限（+ VPC 权限如需要）
- [ ] 生成 `.bedrock_agentcore.yaml` 和 `.example` 模板
- [ ] 安装 agentcore CLI + zip（Windows）
- [ ] 执行 `agentcore deploy` 并传入环境变量
- [ ] 用测试脚本验证 MCP 端点（list tools + call tool）
- [ ] 生成 `deployment-info.md` 供用户配置 Quick Suite
- [ ] 生成 `deployment-info.example.md` 模板（可入库）
- [ ] 更新 `.gitignore`
