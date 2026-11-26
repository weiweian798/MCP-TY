# date-tool-mcp：日期工具 MCP 服务

一个基于 Python 官方 MCP SDK 和 FastMCP 编写的最小 MCP 服务示例。服务提供 `get_today` 工具，供支持 MCP 的客户端或 LangChain Agent 获取服务器当前日期。

本项目同时演示了三种 MCP 使用方式：

- **stdio**：客户端启动本地 Python 子进程，通过标准输入输出通信。
- **SSE**：服务常驻 HTTP 端口，客户端连接 `/sse`。
- **Streamable HTTP**：服务常驻 HTTP 端口，客户端连接 `/mcp`。

## 1. 目录结构

```text
08_MCP协议、Skills框架/
├─ MCP/
│  ├─ server.py              # 本目录核心：stdio 日期工具服务
│  ├─ chatmcp.yaml           # 服务元数据、启动命令和客户端配置示例
│  ├─ requirements.txt       # MCP 服务依赖
│  └─ README.md
├─ mcp_server_stdio.py       # stdio 服务示例
├─ mcp_server_sse.py         # SSE 服务示例，端口 8000
├─ mcp_server_streamable.py  # Streamable HTTP 服务示例，端口 8001
├─ mcp_client.py             # LangChain Agent 调用本地 MCP 工具
├─ 01_MCP服务在项目中使用 .py # 调用 uvx 提供的远程时间工具
└─ 02_必应.py                # 同时接入时间工具和远程必应搜索工具
```

## 2. 核心工具

`server.py` 中只注册了一个工具：

| 工具名 | 参数 | 返回值 | 说明 |
|---|---|---|---|
| `get_today` | 无 | `YYYY-MM-DD` 字符串 | 返回运行 MCP 服务的机器当前日期 |

核心代码如下：

```python
from datetime import datetime
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("date-tool-mcp")

@mcp.tool()
def get_today() -> str:
    """返回当前的年月日，格式 YYYY-MM-DD。"""
    return datetime.now().strftime("%Y-%m-%d")

if __name__ == "__main__":
    mcp.run()
```

工具名、参数类型和 docstring 会被 FastMCP 自动转换为 MCP 工具描述，大模型据此决定何时调用工具。因此，**docstring 应明确描述工具用途，不能只写方法名**。

## 3. 环境准备

建议使用 Python 3.10 或更高版本。

进入本目录并安装 MCP 服务依赖：

```powershell
cd "d:\class\PythonProject1\core4\08_MCP协议、Skills框架\MCP"
python -m pip install -r requirements.txt
```

如果还要运行上级目录的 LangChain Agent 示例，需要额外安装：

```powershell
python -m pip install langchain langchain-openai langchain-mcp-adapters python-dotenv uv
```

其中：

- `langchain-mcp-adapters`：把 MCP 工具转换为 LangChain Tool。
- `uv`：提供 `uvx` 命令，用于运行 `mcp-server-time`。
- `python-dotenv`：从 `.env` 加载大模型 API Key。

## 4. 配置大模型 API Key

仅在 MCP 客户端中调用 `get_today` 不需要大模型 Key；但运行 LangChain Agent 示例时，需要在项目根目录的 `.env` 中配置：

```ini
DASHSCOPE_API_KEY=你的阿里云百炼APIKey
```

代码通过阿里百炼的 OpenAI 兼容接口调用 `qwen-plus`：

```python
model = init_chat_model(
    "qwen-plus",
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
    model_provider="openai",
)
```

注意：`load_dotenv()` 必须放在读取环境变量之前。不要把真实 `.env` 或 API Key 提交到公开仓库。

## 5. 在 MCP 客户端中接入本服务

`server.py` 使用默认的 **stdio** 传输，不监听 HTTP 端口。将下面的配置加入 MCP 客户端，并确保路径指向本机实际文件：

```json
{
  "mcpServers": {
    "date-tool-mcp": {
      "command": "python",
      "args": [
        "d:/class/PythonProject1/core4/08_MCP协议、Skills框架/MCP/server.py"
      ]
    }
  }
}
```

客户端启动时会自动执行 `python server.py`，不需要提前手动启动服务。

直接在终端执行下面的命令也可以启动进程，但它会等待客户端通过 stdin 发送协议消息，不适合当作普通 HTTP 服务测试：

```powershell
python server.py
```

### stdio 的重要约束

stdio 模式下，stdout 只能传输 JSON-RPC 协议消息：

- 不要在 MCP 服务中使用 `print()` 输出调试信息。
- 调试日志应写入 stderr 或日志文件。
- 命令和脚本路径必须真实存在。
- Windows 建议使用 `python`、`sys.executable` 或 Python 的绝对路径，不要照搬 macOS/Linux 的 `python3` 和 `/Users/...` 路径。

否则可能出现 `Invalid JSON`、`EOF while parsing a value` 或子进程启动失败。

## 6. 在 LangChain Agent 中调用

上级目录的 `mcp_client.py` 展示了完整流程：

```text
用户问题
   ↓
LangChain Agent / qwen-plus
   ↓ 判断需要日期工具
MultiServerMCPClient
   ↓ MCP 协议
get_today
   ↓ 工具结果返回大模型
自然语言最终回答
```

stdio 配置应启用 `mcp_client.py` 中对应的配置块：

```python
"date-tool-stdio": {
    "transport": "stdio",
    "command": sys.executable,
    "args": [str(Path(__file__).resolve().parent / "mcp_server_stdio.py")],
}
```

然后在 `08_MCP协议、Skills框架` 目录执行：

```powershell
python mcp_client.py
```

预期能看到 Agent 调用 `get_today`，工具返回类似 `2026-11-26` 的日期，最后生成“今天是……”的自然语言回答。

## 7. SSE 与 Streamable HTTP 示例

HTTP 传输需要**先启动服务，再启动客户端**。三个演示服务的端点如下：

| 服务文件 | 传输方式 | 地址 | 客户端 transport |
|---|---|---|---|
| `mcp_server_stdio.py` | stdio | 无 HTTP 地址 | `stdio` |
| `mcp_server_sse.py` | SSE | `http://127.0.0.1:8000/sse` | `sse` |
| `mcp_server_streamable.py` | Streamable HTTP | `http://127.0.0.1:8001/mcp` | `streamable_http` |

### 启动 SSE 服务

```powershell
python mcp_server_sse.py
```

客户端配置：

```python
"date-tool-sse": {
    "transport": "sse",
    "url": "http://127.0.0.1:8000/sse",
}
```

### 启动 Streamable HTTP 服务

另开一个终端：

```powershell
python mcp_server_streamable.py
```

客户端配置：

```python
"date-tool-streamable": {
    "transport": "streamable_http",
    "url": "http://127.0.0.1:8001/mcp",
}
```

SSE 和 Streamable HTTP 使用不同端点，且这里分别使用 8000、8001 端口，避免端口冲突。修改端口或传输方式时，客户端 URL 必须同步修改。

## 8. 添加一个新工具

在 `server.py` 中继续使用 `@mcp.tool()` 注册函数即可。例如：

```python
@mcp.tool()
def get_year() -> int:
    """返回服务器当前年份。"""
    return datetime.now().year
```

新增工具后重启 MCP 服务或让客户端重新连接，再通过 `client.get_tools()` 获取最新工具列表。

## 9. chatmcp.yaml 说明

`chatmcp.yaml` 提供服务发布或分发时使用的元数据：

- `name`：服务名称，当前为 `date-tool-mcp`。
- `description`：服务简介。
- `local`：本地启动命令。
- `config`：客户端接入配置模板。
- `params`：当前服务不需要额外参数，因此为空对象。

该文件不会代替 `server.py` 执行，实际工具逻辑仍以 Python 代码为准。

## 10. 常见问题

| 报错或现象 | 原因 | 处理方法 |
|---|---|---|
| `No module named 'langchain_mcp_adapters'` | 未安装适配器 | 执行 `python -m pip install langchain-mcp-adapters` |
| `openai.OpenAIError: Missing credentials` | 没配置 Key，或没有提前调用 `load_dotenv()` | 检查 `.env`，并把 `load_dotenv()` 放在读取环境变量之前 |
| `WinError 2`，找不到 `uvx` | 未安装 `uv` | 执行 `python -m pip install uv`，并确认其 Scripts 目录在 PATH 中 |
| `Invalid JSON: EOF while parsing...` | stdio 的 stdout 被额外输出污染，或服务进程启动异常 | 删除服务中的 `print()`，检查命令、路径和服务端依赖 |
| `Errno 10048`，端口只能使用一次 | 8000/8001 已被其他进程占用 | 关闭旧进程或修改端口，同时更新客户端 URL |
| 请求 `/sse` 返回 404 | 把 SSE 地址用于 Streamable HTTP 服务，或服务未启动 | SSE 使用 `/sse`，Streamable HTTP 使用 `/mcp` |
| 远程 MCP 返回 `record not found` | URL 缺少服务 ID，或服务已失效 | 从服务平台重新复制完整连接地址，不要只填写平台根地址 |
| 手动执行 `python server.py` 后没有交互结果 | stdio 服务正在等待协议输入 | 通过 MCP 客户端连接，不要在 stdin 中随意输入普通文本 |

Windows 下查看并关闭 8000 端口占用进程：

```powershell
netstat -ano | Select-String ":8000" | Select-String "LISTENING"
Stop-Process -Id <上一步查到的PID> -Force
```

关闭进程前先确认 PID 和进程名，避免误停其他项目。

## 11. 当前项目边界

- `get_today` 使用服务器本地时区，不提供城市时区转换。
- 服务没有数据库、鉴权和远程部署逻辑，适合学习 MCP 工具注册与客户端接入。
- 上级目录的 `02_必应.py` 还演示了远程 MCP 服务接入；远程地址和可用性由对应服务平台决定。
