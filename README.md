# Agent Harness —— 纯手写 Python AI Agent

 https://www.bilibili.com/video/BV1nXHs68Egp
````
## 这套东西在讲什么

LLM 只会"文本进、文本出"，它自己不能调工具、不能上网、不能读写文件。**Agent harness 就是套在 LLM 外面的那层壳**（大脑之外的"手和腿"），负责：

1. 把可用工具以 JSON schema 告诉模型；
2. 模型说"我要调 xx 工具"时，由我们的 Python 代码真正去执行；
3. 把执行结果塞回对话历史，让模型继续决策；
4. 循环直到模型给出最终文本答案。

## 目录

| 文件 | 说明 | 
| --- | --- |  
| `v0.py` | 最小可运行：一次 chat completion，打通 LLM |  
| `v1.py` | 加工具调用（`list_files` / `read_file`）＋ 手写 tool schema，但只允许一轮工具调用 | 
| `v2.py` | 加 `write_file` ＋ **agent loop**（`while True`，直到模型不再请求工具） |  
| `v3.py` | 接入 MCP server（time / fetch），异步化，本地工具与 MCP 工具统一分发 |  
| `v4.py` | 持久记忆：`memory.md` 注入 system prompt ＋ `save_memory` 工具 |  
| `harness.py` | 完整版：多模型路由、斜杠命令、MCP 从 json 加载、记忆、agent loop 全部合一 | 
| `mcp_servers.json` | MCP 服务器配置（运行时加载，和 Claude Code 一样的格式） |  
| `memory.md` | 长期记忆文件（一行一条事实） | 
| `workspace/` | Agent 的工作目录，文件工具只能在这里读写 |  

## 快速开始

```bash
# 1）依赖（视频里用的是 uv，pip 等价）
uv init && uv add mcp openai pydantic
# 或者
pip install -r requirements.txt

# 2）本地模型（有独显 / Apple Silicon 才建议跑本地）
ollama pull qwen3.5:4b        # 机器弱一点就 qwen3.5:2b，并改 MODEL
ollama serve

# 3）跑
python v0.py
python v1.py
python v2.py
python v3.py       # 需要 uv 已安装，uvx 会自动拉起 mcp-server-time / mcp-server-fetch
python v4.py
python harness.py  # 完整版
```

### 不想跑本地模型，直接用 OpenAI

把每个文件顶部的 client 改成：

```python
client = OpenAI(api_key="sk-...")          # 去掉 base_url
MODEL = "gpt-4o-mini"                      # 换成你要用的模型名
```

`harness.py` 里则不用改代码，只要设置环境变量 `OPENAI_API_KEY`，然后 `/model gpt` 切换过去；Claude 同理（`ANTHROPIC_API_KEY` + `/model claude`）。

## harness.py 里的斜杠命令

| 命令 | 作用 |
| --- | --- |
| `/models` | 列出注册表里的所有模型（`*` 是当前生效的） |
| `/model <name>` | 切换模型，例如 `/model local-small` |
| `/tools` | 列出全部工具，标注 `[local]` 还是 `[mcp]` |
| `/memory` | 打印 `memory.md` 的内容 |
| `/quit` | 退出 |

启动输出形如：

```
starting harness...
[mcp] connected 'time': ['get_current_time', 'convert_time']
[mcp] connected 'fetch': ['fetch']
harness ready - model: local (qwen3.5:4b), 7 tools, memory loaded
type /models, /model <name>, /tools, /memory, or just talk
```

## 自己加工具要改哪三处

1. 写一个普通 Python 函数（返回值统一用字符串）；
2. 加进 `LOCAL_TOOLS` 字典；
3. 在 `TOOL_SCHEMAS` 里补一段 `{"type": "function", "function": {...}}` 的 JSON schema —— 这一步就是"告诉模型它有权调用什么"。

## 说明

- 模型名 `qwen3.5:4b` 是原样保留的；如果你的 Ollama 里拉不到同名模型，改成 `ollama list` 里实际存在的名字即可。
- `v3.py` / `harness.py` 依赖 `uvx` 拉起 MCP server（首次运行需要联网下载），没装 uv 时对应功能会连接失败，但本地工具部分不受影响。
