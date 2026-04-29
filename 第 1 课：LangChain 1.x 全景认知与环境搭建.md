# 第 1 课：LangChain 1.x 全景认知与环境搭建

> 本课关键词：**LangChain 1.x、LangGraph、LangSmith、Deep Agents、环境搭建、Chat Model、invoke、Messages**

---

## 一、本课目标

学完本课，你应该能做到：

1. 理解 LangChain 1.x 生态的整体定位。
2. 分清 LangChain、LangGraph、LangSmith、Deep Agents 分别解决什么问题。
3. 搭建一个可运行的 Python + LangChain 1.x 开发环境。
4. 跑通第一个 LangChain 聊天模型调用程序。

本课不急着写复杂 Agent。第一课最重要的是建立正确地图：你要知道自己正在学的不是单个库，而是一套 LLM 应用开发体系。

---

## 二、为什么要学 LangChain 1.x

大模型本身只负责“根据输入生成输出”。真实应用还需要处理很多工程问题：

- 如何统一调用不同模型供应商。
- 如何管理 Prompt 和消息历史。
- 如何让模型输出结构化数据。
- 如何连接知识库，让模型基于外部资料回答。
- 如何让模型调用工具。
- 如何组织多步骤任务流程。
- 如何追踪、调试和评估模型调用过程。

LangChain 的价值就在于：它把这些常见能力抽象成可组合的组件，让你更快构建 LLM 应用。

LangChain 1.x 相比早期版本，更强调：

- 以 Agent 应用为核心。
- 使用更简洁的 `create_agent` 创建智能体。
- 通过统一消息、工具、结构化输出等接口减少供应商绑定。
- Agent 底层构建在 LangGraph 之上，天然支持更可靠的执行能力。
- 旧版部分能力迁移到 `langchain-classic`，新项目应优先学习 1.x 的推荐写法。

本教材的代码原则：

- 以 LangChain 1.x 官方文档和当前 SDK 为准。
- 新代码优先使用 1.x 推荐的公开入口，例如 `langchain.messages`。
- 模型名不在代码中写死，统一通过 `.env` 中的 `OPENAI_MODEL` 配置。
- 如果官方 API 后续变化，应优先修订教材代码，而不是沿用旧示例。

---

## 三、LangChain 生态全景

你可以把整个生态理解为四层：

| 组件 | 主要作用 | 初学阶段怎么理解 |
| --- | --- | --- |
| LangChain | 快速构建 LLM 应用和 Agent | 先学它，负责模型、Prompt、工具、结构化输出等高层能力 |
| LangGraph | 构建可控、可恢复的状态工作流 | 后面学，适合复杂 Agent 和多步骤流程 |
| LangSmith | 调试、追踪、评估 LLM 应用 | 项目阶段学，用来观察每一步发生了什么 |
| Deep Agents | 更高级的长任务智能体能力 | 最后拓展，关注规划、记忆、上下文管理和子任务隔离 |

### 1. LangChain：应用开发入口

LangChain 是最常用的入口。它帮你解决：

- 调模型。
- 写 Prompt。
- 管理消息。
- 调工具。
- 结构化输出。
- 快速创建 Agent。

初学者第一阶段要重点掌握 LangChain。

### 2. LangGraph：复杂流程编排

LangGraph 适合处理更复杂、更可控的流程，例如：

- 工单先分类，再查知识库，再生成建议。
- 某一步失败后重试。
- 需要人工审批后继续执行。
- 长任务需要保存中间状态。

你可以先把 LangGraph 理解成“状态机 + 工作流 + Agent 运行时”。

### 3. LangSmith：可观察与评估

当你的应用变复杂后，一个问题可能出在很多地方：

- Prompt 没写清楚。
- 检索召回错了。
- 工具参数错了。
- 模型输出不稳定。
- 中间状态传递错了。

LangSmith 用来记录和查看这些链路，让你知道每一步具体发生了什么。

### 4. Deep Agents：复杂智能体系统

Deep Agents 面向更复杂的任务型智能体，例如：

- 多步骤研究。
- 自动拆解任务。
- 长上下文压缩。
- 多个子 Agent 分工。
- 反思、自检和修正。

这不是第一阶段重点。你现在只需要知道：它是 Agent 能力继续向上的方向。

---

## 四、本课程的学习路线地图

本课程后续会按下面顺序推进：

```text
基础模型调用
  ↓
Prompt 与 Messages
  ↓
Runnable 链式组合
  ↓
结构化输出
  ↓
RAG 知识库问答
  ↓
Tools 与 Agent
  ↓
LangGraph 工作流
  ↓
调试、评估与工程化
  ↓
综合项目与 Deep Agents 拓展
```

学习时要记住一个原则：

> 先把“输入是什么、输出是什么、数据怎么流动”搞清楚，再让模型拥有更多自主决策能力。

如果基础链路没练熟，直接学 Agent 会很容易变成“能跑，但不知道为什么这样跑”。

---

## 五、环境准备

### 1. 推荐开发环境

建议使用：

- Python 3.11 或更高版本。
- uv。
- VS Code 或 PyCharm。
- PowerShell / Terminal。
- uv 管理的独立虚拟环境。
- `.env` 文件管理 API Key。

检查 Python 版本：

```powershell
python --version
```

如果你本机同时装了多个 Python，也可以尝试：

```powershell
py --version
```

### 2. 新建课程项目目录

建议后续课程统一放在一个目录中，例如：

```text
langchain_1x_course/
  lessons/
  examples/
  projects/
  .env
  pyproject.toml
  uv.lock
```

如果你还没有安装 uv，可以先参考官方安装方式：`https://docs.astral.sh/uv/getting-started/installation/`。

检查 uv 是否可用：

```powershell
uv --version
```

如果你现在只想先跑通第一个程序，可以用 uv 初始化一个课程项目：

```powershell
uv init langchain_1x_course
cd langchain_1x_course
```

如果目录已经存在，也可以进入目录后执行：

```powershell
uv init
```

### 3. 使用 uv 创建虚拟环境

Windows PowerShell：

```powershell
uv venv --python 3.11
.\.venv\Scripts\Activate.ps1
```

如果 PowerShell 提示脚本执行策略限制，可以临时使用：

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\.venv\Scripts\Activate.ps1
```

macOS / Linux：

```bash
uv venv --python 3.11
source .venv/bin/activate
```

虚拟环境激活后，命令行前面通常会出现 `(.venv)`。

补充说明：

- `uv venv` 会在当前项目中创建 `.venv`。
- 即使不手动激活虚拟环境，也可以使用 `uv run` 在项目环境中运行命令。
- 后续课程推荐优先使用 `uv run`，这样更不容易跑错 Python 环境。
- 运行 Python 脚本时，本课程统一使用 `uv run 脚本路径`。

---

## 六、安装依赖

本课程第一阶段推荐先安装以下依赖：

```powershell
uv add langchain langchain-openai python-dotenv
```

依赖说明：

| 包名 | 作用 |
| --- | --- |
| `langchain` | LangChain 1.x 主包 |
| `langchain-openai` | OpenAI 聊天模型集成包 |
| `python-dotenv` | 从 `.env` 文件读取环境变量 |

如果后续使用 RAG，还会安装向量数据库、文档解析、Embedding 等依赖。第一课先保持简单。

检查安装是否成功：

```powershell
uv run python -c "import langchain; print(langchain.__version__)"
```

---

## 七、配置 API Key

在项目根目录新建 `.env` 文件：

```text
OPENAI_API_KEY=你的_api_key
OPENAI_MODEL=gpt-5-nano
```

如果你使用的是 OpenAI 兼容接口，可能还需要配置：

```text
OPENAI_API_KEY=你的_api_key
OPENAI_BASE_URL=https://你的兼容接口地址/v1
OPENAI_MODEL=你的模型名
```

注意：

- `.env` 不要提交到公开仓库。
- API Key 不要写死在代码里。
- 如果你使用的不是 OpenAI 官方服务，要确认你的服务是否兼容 OpenAI Chat Completions API。

---

## 八、第一个 LangChain 程序

新建文件：

```text
examples/lesson_01_hello_langchain.py
```

写入以下代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain_openai import ChatOpenAI


load_dotenv()


model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.2,
)

response = model.invoke("请用三句话解释什么是 LangChain。")

print(response.content)
```

运行：

```powershell
uv run examples/lesson_01_hello_langchain.py
```

如果运行成功，你会看到模型返回一段关于 LangChain 的解释。

---

## 九、理解这段代码

### 1. `load_dotenv()`

这行代码会读取当前项目中的 `.env` 文件，把里面的配置加载到环境变量中。

也就是说，代码中不需要直接写：

```python
api_key="..."
```

这样更安全，也更方便切换环境。

### 2. `ChatOpenAI`

`ChatOpenAI` 是 LangChain 对 OpenAI 聊天模型的封装。

它的作用不是“重新实现一个模型”，而是提供一个统一接口，让你用 LangChain 的方式调用模型。

常见参数：

| 参数 | 含义 |
| --- | --- |
| `model` | 使用哪个模型 |
| `temperature` | 控制输出随机性，越低越稳定 |
| `timeout` | 请求超时时间 |
| `max_retries` | 失败重试次数 |
| `base_url` | OpenAI 兼容接口地址 |

### 3. `invoke()`

`invoke()` 是 LangChain 中非常重要的调用方法。

你可以先把它理解为：

```text
给组件一个输入，得到一个输出。
```

在这里：

```python
response = model.invoke("请用三句话解释什么是 LangChain。")
```

输入是一段用户问题，输出是一个 AI 消息对象。

### 4. `response.content`

模型返回的不是单纯字符串，而是一个消息对象。真正的文本内容通常在：

```python
response.content
```

在 LangChain 1.x 中，`AIMessage` 还可能包含 `text`、`content_blocks`、`tool_calls`、`usage_metadata` 等信息。

第一阶段先使用 `response.content` 获取原始文本内容；后面学习工具调用、多模态和 Agent 时，再进一步学习 `content_blocks` 和 `tool_calls`。

这也是 LangChain 和普通 HTTP 请求不一样的地方：它会把模型输入输出统一抽象成消息对象，方便后续做多轮对话、工具调用和 Agent。

---

## 十、Messages 入门

除了直接传入字符串，聊天模型也可以接收消息列表。

新建文件：

```text
examples/lesson_01_messages.py
```

代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain.messages import HumanMessage, SystemMessage
from langchain_openai import ChatOpenAI


load_dotenv()


model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.2,
)

messages = [
    SystemMessage(content="你是一名耐心、严谨的 Python 和 LangChain 老师。"),
    HumanMessage(content="请用初学者能听懂的方式解释 invoke() 是什么。"),
]

response = model.invoke(messages)

print(response.content)
```

运行：

```powershell
uv run examples/lesson_01_messages.py
```

这里出现了两类消息：

| 消息类型 | 作用 |
| --- | --- |
| `SystemMessage` | 设定模型角色、规则、回答风格 |
| `HumanMessage` | 用户输入 |

后面你还会见到：

| 消息类型 | 作用 |
| --- | --- |
| `AIMessage` | 模型回复 |
| `ToolMessage` | 工具调用后的结果 |

第一课先掌握 System 和 Human 即可。

---

## 十一、实践案例：学习概念解释助手

现在你来做一个很小但完整的实践案例：输入一个技术概念，让模型按固定结构解释。

新建文件：

```text
examples/lesson_01_concept_teacher.py
```

代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain.messages import HumanMessage, SystemMessage
from langchain_openai import ChatOpenAI


load_dotenv()


model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.2,
)


concept = input("请输入你想学习的概念：")

messages = [
    SystemMessage(
        content=(
            "你是一名高级编程老师，擅长把复杂技术讲清楚。"
            "回答要适合初学者，避免堆砌术语。"
        )
    ),
    HumanMessage(
        content=f"""
请解释这个技术概念：{concept}

请按以下结构回答：
1. 一句话定义
2. 通俗解释
3. 为什么它重要
4. 一个生活类比
5. 一个简单练习题
"""
    ),
]

response = model.invoke(messages)

print("\n=== 学习卡片 ===\n")
print(response.content)
```

运行：

```powershell
uv run examples/lesson_01_concept_teacher.py
```

可以输入：

```text
Runnable
```

或：

```text
RAG
```

这个案例虽然简单，但已经包含 LLM 应用的基本结构：

```text
用户输入 → Prompt 组织 → 模型调用 → 输出结果
```

---

## 十二、常见错误与排查

### 1. `ModuleNotFoundError: No module named 'langchain_openai'`

说明依赖没装好，执行：

```powershell
uv add langchain-openai
```

如果你使用 `uv run` 仍然报错，确认当前目录下存在 `pyproject.toml`，并且你是在项目根目录执行命令。

### 2. 找不到 API Key

检查：

- `.env` 是否在当前项目根目录。
- 变量名是否是 `OPENAI_API_KEY`。
- 是否调用了 `load_dotenv()`。
- 运行命令时所在目录是否正确。

### 3. 接口地址错误

如果使用 OpenAI 兼容接口，需要配置 `OPENAI_BASE_URL`，并确认地址通常以 `/v1` 结尾。

也可以在代码中显式传入：

```python
model = ChatOpenAI(
    model="你的模型名",
    base_url="https://你的接口地址/v1",
)
```

### 4. 模型名不可用

不同账号和不同服务商支持的模型名不一样。

如果默认的 `gpt-5-nano` 不可用，请在 `.env` 中把 `OPENAI_MODEL` 改成你当前服务支持的聊天模型名称。

### 5. 网络或代理问题

如果出现连接超时、DNS 错误、SSL 错误，需要检查：

- 当前网络是否能访问模型服务。
- 是否需要代理。
- API 服务商地址是否配置正确。

---

## 十三、本课小结

这一课你完成了 LangChain 学习的第一步：

- 认识了 LangChain 1.x 的整体生态。
- 分清了 LangChain、LangGraph、LangSmith、Deep Agents 的定位。
- 搭建了 Python 虚拟环境。
- 安装了 LangChain 相关依赖。
- 配置了 API Key。
- 跑通了第一个聊天模型调用。
- 初步理解了 `ChatOpenAI`、`invoke()`、Messages 和 `response.content`。

请记住：

> LangChain 的学习核心不是背 API，而是理解数据如何在模型、Prompt、工具、检索器和工作流之间流动。

---

## 十四、课后练习

### 练习 1：修改模型角色

把实践案例中的 `SystemMessage` 改成：

```text
你是一名面试官，擅长用面试题考察候选人对技术概念的理解。
```

观察输出风格有什么变化。

### 练习 2：限制输出格式

修改 `HumanMessage`，要求模型必须使用 Markdown 表格输出：

```text
请用 Markdown 表格解释这个概念，列包括：定义、应用场景、常见误区、练习题。
```

观察模型是否稳定遵守格式。

### 练习 3：比较 temperature

分别设置：

```python
temperature=0
temperature=0.8
```

用同一个问题调用模型 3 次，比较回答稳定性。

### 练习 4：整理学习笔记

请用自己的话回答：

1. LangChain 主要解决什么问题？
2. LangGraph 和 LangChain 有什么区别？
3. `invoke()` 是什么？
4. 为什么模型返回值不是简单字符串？
5. `SystemMessage` 和 `HumanMessage` 的区别是什么？

---

## 十五、下一课预告

下一课进入：

```text
第 2 课：Messages 与 Prompt 模板
```

你会系统学习：

- 多轮消息如何组织。
- PromptTemplate 怎么用。
- ChatPromptTemplate 怎么用。
- 如何把变量注入提示词。
- 如何写出更可维护的 Prompt。

第 1 课像是把教室、电源和黑板都准备好；第 2 课开始，我们就正式练“怎么把问题交给模型”。

---

## 十六、官方参考

- LangChain Python overview: `https://docs.langchain.com/oss/python/langchain/overview`
- LangChain v1 release notes: `https://docs.langchain.com/oss/python/releases/langchain-v1`
- ChatOpenAI integration: `https://docs.langchain.com/oss/python/integrations/chat/openai/`
- LangGraph v1 release notes: `https://docs.langchain.com/oss/python/releases/langgraph-v1`
