# 第 5 课：Runnable 核心接口深入

> 本课关键词：**Runnable、invoke、batch、stream、RunnableLambda、输入输出约定、统一接口**

---

## 一、本课目标

学完本课，你应该能做到：

1. 理解 LangChain 为什么把很多组件都设计成 Runnable。
2. 理解 Runnable 的输入输出约定。
3. 掌握 `invoke()`、`batch()`、`stream()` 的基本用法和适用场景。
4. 使用 `RunnableLambda` 把普通 Python 函数包装成 Runnable。
5. 完成一个单步文本改写实践。

前四课你已经分别学过：

- `ChatPromptTemplate` 可以接收变量，输出 PromptValue。
- `ChatOpenAI` 可以接收字符串或消息，输出 `AIMessage`。
- `StrOutputParser` 可以接收模型消息，输出字符串。
- `with_structured_output()` 可以让模型输出 Pydantic 对象。

你可能已经注意到一个共同点：这些组件都可以被 `invoke()` 调用。

这不是巧合，而是 LangChain 的核心设计之一：Runnable。

---

## 二、为什么会有 Runnable

先想一个开发问题。

你正在构建一个 LLM 应用，流程可能是：

```text
用户输入
  ↓
Prompt 模板
  ↓
聊天模型
  ↓
输出解析器
  ↓
业务代码
```

如果每个组件都有完全不同的调用方式，代码会变成这样：

```text
Prompt 用 format()
Model 用 call()
Parser 用 parse()
Retriever 用 search()
Tool 用 execute()
```

这样会带来几个麻烦：

- 学一个组件就要记一种调用方式。
- 很难把多个组件稳定接起来。
- 批量处理、流式输出、异步调用很难统一。
- 调试和追踪时，很难用同一种方式观察每一步。
- 后续组合复杂链路时，代码会越来越散。

Runnable 的设计意图，就是给 LangChain 组件一个统一的“工作单元”协议。

你可以把 Runnable 理解成：

> 一个可以接收输入、产生输出，并且支持统一调用方式的组件。

官方参考文档中对 Runnable 的描述是：它是一个可以被 `invoke`、`batch`、`stream`、`transform` 和组合的工作单元。

这也是为什么 LangChain 中很多看起来不同的东西都能用同一种方式调用：

| 组件 | 输入 | 输出 |
| --- | --- | --- |
| Prompt Template | 变量字典 | PromptValue |
| Chat Model | 字符串 / 消息 / PromptValue | AIMessage |
| Output Parser | AIMessage | 字符串或其他结构 |
| RunnableLambda | 任意指定输入 | 函数返回值 |

Runnable 让 LangChain 的组件从“各自为政”变成“可以组合的积木”。

---

## 三、Runnable 的核心输入输出思维

学习 Runnable，最重要的不是背类名，而是建立输入输出思维。

每个 Runnable 都可以这样看：

```text
Input → Runnable → Output
```

例如：

```text
{"topic": "LangChain"} → ChatPromptTemplate → PromptValue
```

```text
"解释 Runnable" → ChatOpenAI → AIMessage
```

```text
AIMessage → StrOutputParser → str
```

只要你能说清楚一个组件的输入和输出，就能更容易把它接到下一个组件。

后面学习 LCEL 链式组合时，你会看到：

```python
chain = prompt | model | parser
```

这行代码能成立，是因为：

```text
prompt 的输出，正好能作为 model 的输入。
model 的输出，正好能作为 parser 的输入。
```

本课先不急着深入 `|` 管道组合。你现在只需要先把每个 Runnable 的输入输出看清楚。

---

## 四、示例 1：Prompt 也是 Runnable

第 2 课你已经用过 `ChatPromptTemplate.invoke()`。现在从 Runnable 的角度重新看它。

新建文件：

```text
examples/lesson_05_prompt_is_runnable.py
```

代码：

```python
from langchain_core.prompts import ChatPromptTemplate


prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "你是一名简洁的技术解释助手。"),
        ("human", "请用{style}风格解释：{topic}"),
    ]
)

prompt_value = prompt.invoke(
    {
        "style": "初学者能听懂的",
        "topic": "Runnable",
    }
)

print("Prompt 输出类型：")
print(type(prompt_value))

print("\nPrompt 生成的消息：")
for message in prompt_value.messages:
    print(type(message).__name__, "=>", message.content)
```

运行：

```powershell
uv run examples/lesson_05_prompt_is_runnable.py
```

观察输出：

- 输入是变量字典。
- 输出是 PromptValue。
- PromptValue 中包含真正要发送给聊天模型的消息。

这说明 Prompt 模板不是普通字符串工具，而是一个有输入输出约定的 Runnable。

---

## 五、示例 2：模型也是 Runnable

聊天模型也实现了 Runnable 接口。

新建文件：

```text
examples/lesson_05_model_is_runnable.py
```

代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain_openai import ChatOpenAI


load_dotenv()


model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.2,
)

message = model.invoke("请用一句话解释 LangChain Runnable。")

print("模型输出类型：")
print(type(message))

print("\n模型输出文本：")
print(message.text)
```

运行：

```powershell
uv run examples/lesson_05_model_is_runnable.py
```

观察输出：

- 输入是字符串。
- 输出是 `AIMessage`。
- 你可以用 `message.text` 获取文本内容。

这说明模型调用也符合：

```text
Input → Runnable → Output
```

---

## 六、示例 3：Parser 也是 Runnable

第 3 课你已经用过 `StrOutputParser`。现在从 Runnable 角度看它。

新建文件：

```text
examples/lesson_05_parser_is_runnable.py
```

代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain_core.output_parsers import StrOutputParser
from langchain_openai import ChatOpenAI


load_dotenv()


model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.2,
)

parser = StrOutputParser()

message = model.invoke("请用一句话说明输出解析器的作用。")
text = parser.invoke(message)

print("解析前类型：")
print(type(message))

print("\n解析后类型：")
print(type(text))

print("\n解析后文本：")
print(text)
```

运行：

```powershell
uv run examples/lesson_05_parser_is_runnable.py
```

观察输出：

- Parser 的输入是模型消息。
- Parser 的输出是普通字符串。
- Parser 把“取出文本”这一步变成了一个可组合的工作单元。

---

## 七、`invoke()`：处理一个输入

`invoke()` 是 Runnable 最基础的调用方式。

它适合：

- 一次处理一个用户问题。
- 一次生成一个 Prompt。
- 一次调用一个模型。
- 一次解析一个模型输出。

你可以把它理解成：

```text
给一个输入，得到一个输出。
```

例如：

```python
output = runnable.invoke(input)
```

为什么 LangChain 把这个方法统一叫 `invoke()`，而不是每个组件各叫各的名字？

因为统一命名能让你在切换组件时少改变思维：

```python
prompt.invoke(...)
model.invoke(...)
parser.invoke(...)
structured_model.invoke(...)
```

这些组件做的事情不同，但调用形态一致。

这就是 Runnable 设计带来的第一个收益：降低组合成本。

---

## 八、`batch()`：处理多个独立输入

真实应用里，你经常不是只处理一条数据。

例如：

- 批量改写 20 条文案。
- 批量摘要 100 篇短文本。
- 批量给用户反馈分类。
- 批量生成标题。

如果只会 `invoke()`，就可能写循环：

```python
results = []
for text in texts:
    results.append(model.invoke(text))
```

这样能工作，但没有表达出“这是批量处理”的意图。

Runnable 提供 `batch()`，让你可以把多个输入一次交给同一个组件：

```text
list[Input] → Runnable.batch() → list[Output]
```

### 示例 4：模型批量调用

新建文件：

```text
examples/lesson_05_model_batch.py
```

代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain_openai import ChatOpenAI


load_dotenv()


model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.2,
)

questions = [
    "请用一句话解释 PromptTemplate。",
    "请用一句话解释 AIMessage。",
    "请用一句话解释 StrOutputParser。",
]

responses = model.batch(
    questions,
    config={"max_concurrency": 2},
)

for index, response in enumerate(responses, start=1):
    print(f"{index}. {response.text}")
```

运行：

```powershell
uv run examples/lesson_05_model_batch.py
```

观察输出：

- 输入是字符串列表。
- 输出是 `AIMessage` 列表。
- `max_concurrency` 可以限制并发数量。

注意：官方文档说明，LangChain 的 `batch()` 通常是在客户端并行化多个调用；它不同于 OpenAI、Anthropic 等服务商自己的离线 Batch API。

---

## 九、`stream()`：边生成边接收

`invoke()` 会等模型完整生成完，再一次性返回结果。

如果回答很长，用户就要一直等。

这时可以使用 `stream()`。

`stream()` 的设计目标是改善体验：

```text
模型正在生成 → 程序陆续收到 chunk → 用户逐步看到内容
```

在聊天模型中，`stream()` 通常返回多个 `AIMessageChunk`。

### 示例 5：模型流式输出

新建文件：

```text
examples/lesson_05_model_stream.py
```

代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain_openai import ChatOpenAI


load_dotenv()


model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.2,
)

for chunk in model.stream("请用 5 句话解释为什么 Runnable 很重要。"):
    print(chunk.text, end="", flush=True)

print()
```

运行：

```powershell
uv run examples/lesson_05_model_stream.py
```

观察输出：

- 文本会逐步出现在终端里。
- 每次循环得到的是一个输出 chunk。
- 这里用 `chunk.text` 打印当前文本片段。

适合使用 `stream()` 的场景：

- 聊天应用。
- 长答案生成。
- Web 页面实时展示。
- 希望用户更快看到反馈的交互。

不适合使用 `stream()` 的场景：

- 你必须等完整结果才能解析。
- 下游逻辑不支持分块处理。
- 结构化输出场景中需要完整对象。

---

## 十、RunnableLambda：把普通函数变成 Runnable

到目前为止，你看到的 Runnable 都来自 LangChain 内置组件。

但真实项目里，你也会有自己的 Python 函数，例如：

```python
def clean_text(text: str) -> str:
    return text.strip()
```

如果这个函数只是普通函数，就不能直接参与 LangChain 的统一调用体系。

`RunnableLambda` 的作用是：

> 把一个普通 Python callable 包装成 Runnable。

它解决的是“自定义业务逻辑如何接入 LangChain 组件体系”的问题。

### 示例 6：包装普通函数

新建文件：

```text
examples/lesson_05_runnable_lambda.py
```

代码：

```python
from langchain_core.runnables import RunnableLambda


def normalize_topic(topic: str) -> str:
    return topic.strip().replace(" ", "_").lower()


normalizer = RunnableLambda(normalize_topic)

single_result = normalizer.invoke("  LangChain Runnable  ")
batch_result = normalizer.batch(
    [
        "  Prompt Template ",
        " AI Message ",
        " Structured Output ",
    ]
)

print("单个结果：")
print(single_result)

print("\n批量结果：")
print(batch_result)
```

运行：

```powershell
uv run examples/lesson_05_runnable_lambda.py
```

观察输出：

- 普通函数被包装后，也有了 `invoke()`。
- 同一个逻辑也能使用 `batch()`。
- 这为后续把业务函数接入链路打基础。

---

## 十一、实践案例：单步文本改写器

现在你来完成一个本课实践案例：把用户输入的一段文字改写成指定风格。

这一版暂时不使用 `|` 管道组合，而是手动执行每一步，目的是看清楚每个 Runnable 的输入和输出。

新建文件：

```text
examples/lesson_05_text_rewriter.py
```

代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnableLambda
from langchain_openai import ChatOpenAI


load_dotenv()


def clean_user_text(text: str) -> str:
    return text.strip()


cleaner = RunnableLambda(clean_user_text)

prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "你是一个文本改写助手。请只输出改写后的文本，不要额外解释。",
        ),
        (
            "human",
            """
请将下面文本改写成 {style} 风格：

{text}
""",
        ),
    ]
)

model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.3,
)

parser = StrOutputParser()

raw_text = input("请输入要改写的文本：")
style = input("请输入目标风格：")

cleaned_text = cleaner.invoke(raw_text)
prompt_value = prompt.invoke(
    {
        "text": cleaned_text,
        "style": style,
    }
)
message = model.invoke(prompt_value)
rewritten_text = parser.invoke(message)

print("\n=== 改写结果 ===\n")
print(rewritten_text)
```

运行：

```powershell
uv run examples/lesson_05_text_rewriter.py
```

可以输入：

```text
请输入要改写的文本：LangChain 可以帮助开发者更方便地构建大模型应用。
请输入目标风格：面向产品经理的通俗说明
```

观察这个流程：

```text
raw_text → cleaner → cleaned_text
变量字典 → prompt → prompt_value
prompt_value → model → AIMessage
AIMessage → parser → rewritten_text
```

你现在已经看到了一个完整的 Runnable 思维：

```text
每一步都有输入，每一步都有输出。
只要输出能接上下一个输入，流程就能继续。
```

---

## 十二、`invoke`、`batch`、`stream` 怎么选

你可以先用这个表判断：

| 方法 | 输入 | 输出 | 适合场景 |
| --- | --- | --- | --- |
| `invoke()` | 单个输入 | 单个输出 | 普通单次调用 |
| `batch()` | 输入列表 | 输出列表 | 批量处理独立任务 |
| `stream()` | 单个输入 | 输出片段迭代器 | 需要边生成边展示 |

几个判断问题：

- 只有一个用户请求？先用 `invoke()`。
- 有很多条互相独立的数据？考虑 `batch()`。
- 输出很长，想让用户早点看到？考虑 `stream()`。
- 下游必须等完整对象？先用 `invoke()`。
- 需要批量但担心并发太高？给 `batch()` 传 `config={"max_concurrency": n}`。

---

## 十三、常见错误与排查

### 1. 忘记看输入输出类型

错误思路：

```python
text = prompt.invoke({"topic": "Runnable"})
message = parser.invoke(text)
```

这里 `prompt.invoke()` 输出的是 PromptValue，不是模型消息；`StrOutputParser` 通常需要接模型输出。

排查方法：

```python
print(type(value))
```

学习 Runnable 时，`type()` 是非常有用的调试工具。

### 2. 把 `batch()` 当成服务商离线 Batch API

LangChain 的 `batch()` 通常是对多个输入做客户端并发调用。

它适合提高多个独立请求的处理效率，但不等同于 OpenAI 或其他服务商提供的离线批处理 API。

### 3. 在不支持流式处理的下游使用 `stream()`

`stream()` 返回的是一段一段的 chunk。

如果你的后续逻辑必须一次性拿到完整结果，就先用 `invoke()`。

### 4. 对 `RunnableLambda` 包装复杂副作用函数

`RunnableLambda` 很适合包装简单、清晰、输入输出明确的函数。

如果函数会修改文件、访问数据库、调用外部服务，后续要加入错误处理、日志和权限控制。本课先只包装纯文本处理函数。

### 5. 误以为 RunnableLambda 适合所有流式场景

官方参考文档说明，`RunnableLambda` 更适合不需要原生流式处理的普通 callable。

如果你的自定义逻辑需要一边接收 chunk、一边产出 chunk，后续可以了解 `RunnableGenerator`。本课先把重点放在最常见的同步函数包装上。

### 6. 混淆“能 invoke”和“输出类型一样”

很多组件都能 `invoke()`，但输出类型不一样：

- Prompt 输出 PromptValue。
- Model 输出 AIMessage。
- Parser 输出字符串。
- 结构化模型输出 Pydantic 对象。

统一接口不等于统一输出类型。

---

## 十四、本课小结

这一课你进入了 LangChain 的核心编排思维：

- Runnable 是 LangChain 中可调用、可批量、可流式、可组合的工作单元。
- Runnable 的关键是输入输出约定。
- `invoke()` 处理单个输入。
- `batch()` 处理多个独立输入。
- `stream()` 适合边生成边接收输出。
- Prompt、Model、Parser 都可以看成 Runnable。
- `RunnableLambda` 可以把普通 Python 函数包装成 Runnable。
- 只要每一步输出能接上下一个输入，后续就能组合成链。

请记住：

> Runnable 的价值不是多一个类名，而是让不同组件拥有同一种调用协议。

---

## 十五、课后练习

### 练习 1：检查三个组件的输入输出

创建：

```text
examples/lesson_05_io_debugger.py
```

要求：

- 创建一个 `ChatPromptTemplate`。
- 调用 `prompt.invoke()` 并打印类型。
- 调用 `model.invoke()` 并打印类型。
- 调用 `parser.invoke()` 并打印类型。
- 用自己的话写下注释：每一步的输入和输出是什么。

### 练习 2：批量标题生成

创建：

```text
examples/lesson_05_batch_titles.py
```

要求：

- 准备 5 段短文本。
- 使用 `model.batch()` 为每段文本生成标题。
- 设置 `config={"max_concurrency": 2}`。
- 打印每个标题。

### 练习 3：流式故事生成

创建：

```text
examples/lesson_05_stream_story.py
```

要求：

- 使用 `model.stream()` 生成一个短故事。
- 用 `print(chunk.text, end="", flush=True)` 实时输出。
- 观察它和 `invoke()` 的体验区别。

### 练习 4：包装自己的函数

创建：

```text
examples/lesson_05_custom_runnable.py
```

要求：

- 写一个普通函数，接收字符串，返回清洗后的字符串。
- 用 `RunnableLambda` 包装它。
- 分别用 `invoke()` 和 `batch()` 调用。

### 练习 5：整理学习笔记

请用自己的话回答：

1. 为什么 LangChain 需要 Runnable？
2. `invoke()`、`batch()`、`stream()` 分别适合什么场景？
3. 为什么说 Runnable 的核心是输入输出约定？
4. `RunnableLambda` 解决什么问题？
5. 为什么“统一接口”不等于“输出类型一样”？

---

## 十六、下一课预告

下一课进入：

```text
第 6 课：LCEL 链式组合
```

你会学习：

- 为什么可以写 `prompt | model | parser`。
- `RunnableSequence` 的设计意义。
- 如何把本课手动执行的多步骤流程改成链式组合。
- 为什么链式组合能天然支持 `invoke()`、`batch()` 和 `stream()`。

第 5 课解决“每个组件如何统一调用”；第 6 课开始解决“多个组件如何优雅组合”。

---

## 十七、官方参考

- LangChain Models - Invocation: `https://docs.langchain.com/oss/python/langchain/models`
- LangChain Core Runnables Reference: `https://reference.langchain.com/python/langchain-core/runnables`
- Runnable Reference: `https://reference.langchain.com/python/langchain-core/runnables/base/Runnable`
- RunnableLambda Reference: `https://reference.langchain.com/python/langchain-core/runnables/base/RunnableLambda`
