# 第 6 课：LCEL 链式组合

> 本课关键词：**LCEL、RunnableSequence、管道组合、prompt | model | parser、链式编排、输入输出流转**

---

## 一、本课目标

学完本课，你应该能做到：

1. 理解 LCEL 为什么是 LangChain 中重要的链式组合方式。
2. 掌握 `|` 管道操作符的基本用法。
3. 说清楚 `prompt | model | parser` 中每一段的输入和输出。
4. 理解 `RunnableSequence` 的设计意义。
5. 使用 LCEL 构建一个可复用的基础文本处理链。
6. 使用同一个链完成 `invoke()`、`batch()` 和 `stream()` 调用。

第 5 课你已经知道：Prompt、Model、Parser 都可以看成 Runnable。

这一课要解决新的问题：

> 既然每个组件都能单独调用，怎样把它们自然、稳定、可读地组合成一条完整流程？

---

## 二、为什么会有 LCEL

先回到一个最常见的 LLM 应用流程：

```text
用户输入
  ↓
Prompt 模板
  ↓
聊天模型
  ↓
输出解析器
  ↓
业务可直接使用的结果
```

如果不用链式组合，你可能会这样写：

```python
prompt_value = prompt.invoke({"topic": "LCEL"})
message = model.invoke(prompt_value)
text = parser.invoke(message)
print(text)
```

这段代码没有错，而且很适合理解每一步发生了什么。

但当流程变长之后，问题会出现：

- 每增加一步，都要手动保存中间变量。
- 批量处理时，需要自己循环和组织结果。
- 流式输出时，需要确认每一步是否能处理流。
- 调试时，流程结构分散在一堆临时变量里。
- 复用时，很难把这段流程当成一个整体传给别的模块。

LCEL 的设计意图，是让你用声明式方式描述一条 Runnable 链路。

你不再逐行控制“先调用谁，再把结果传给谁”，而是把流程写成：

```python
chain = prompt | model | parser
```

这行代码表达的是：

```text
把输入交给 prompt，
prompt 的输出交给 model，
model 的输出交给 parser，
parser 的输出作为最终结果。
```

LCEL 解决的不是“能不能调用模型”这个问题，而是“怎样把多个组件组合成清晰、可复用、可调用的程序单元”。

---

## 三、LCEL 是什么

LCEL 是 **LangChain Expression Language** 的缩写，可以理解为 LangChain 的表达式组合语言。

它最常见的写法是使用 `|`：

```python
chain = step_1 | step_2 | step_3
```

这里的 `|` 不是命令行里的管道，也不是集合运算。它在 LangChain 中被重载为 Runnable 的组合操作符。

当你写：

```python
chain = prompt | model | parser
```

LangChain 会把它组织成一个 `RunnableSequence`。

`RunnableSequence` 的含义很直接：

> 按顺序执行多个 Runnable，前一个 Runnable 的输出作为后一个 Runnable 的输入。

更重要的是，组合后的 `chain` 本身仍然是一个 Runnable。

所以它依然可以这样调用：

```python
chain.invoke(...)
chain.batch(...)
chain.stream(...)
```

这就是 LCEL 的核心价值：你把多个组件组合起来之后，并没有失去 Runnable 的统一接口。

---

## 四、示例 1：不用 LCEL 手动执行三步

先写一个不用 LCEL 的版本。这样你能看清楚 LCEL 省掉的不是“必要知识”，而是重复的流程胶水代码。

新建文件：

```text
examples/lesson_06_manual_steps.py
```

代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI


load_dotenv()


prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "你是一名擅长解释技术概念的技术解释助手。"),
        ("human", "请用{style}风格解释：{topic}"),
    ]
)

model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.2,
)

parser = StrOutputParser()

prompt_value = prompt.invoke(
    {
        "style": "初学者能听懂的",
        "topic": "LCEL",
    }
)

message = model.invoke(prompt_value)
text = parser.invoke(message)

print(text)
```

运行：

```powershell
uv run examples/lesson_06_manual_steps.py
```

观察这三行：

```python
prompt_value = prompt.invoke(...)
message = model.invoke(prompt_value)
text = parser.invoke(message)
```

它们正好对应：

```text
dict → PromptValue → AIMessage → str
```

这条输入输出链，是理解 LCEL 的基础。

---

## 五、示例 2：使用 `prompt | model | parser`

现在把上面的三步改成 LCEL。

新建文件：

```text
examples/lesson_06_basic_lcel_chain.py
```

代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI


load_dotenv()


prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "你是一名擅长解释技术概念的技术解释助手。"),
        ("human", "请用{style}风格解释：{topic}"),
    ]
)

model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.2,
)

parser = StrOutputParser()

chain = prompt | model | parser

result = chain.invoke(
    {
        "style": "初学者能听懂的",
        "topic": "LCEL",
    }
)

print(result)
```

运行：

```powershell
uv run examples/lesson_06_basic_lcel_chain.py
```

你会看到最终输出已经是字符串，而不是 `AIMessage`。

原因是链路最后接了 `StrOutputParser()`：

```text
Prompt 模板负责生成消息
聊天模型负责生成 AIMessage
StrOutputParser 负责提取最终文本
```

这也是 `prompt | model | parser` 的经典结构。

---

## 六、理解每一段的职责

学习 LCEL 时，不要只记住写法。你要能说清楚每一段负责什么。

| 链路片段 | 输入 | 输出 | 职责 |
| --- | --- | --- | --- |
| `prompt` | 变量字典 | PromptValue | 把运行时变量填入 Prompt 模板 |
| `model` | PromptValue / 消息 / 字符串 | AIMessage | 调用聊天模型生成回复 |
| `parser` | AIMessage | 字符串 | 把模型消息转换成业务更方便使用的文本 |

这条链成立的前提是：

```text
上一段的输出类型，必须能被下一段接收。
```

你可以把 LCEL 看成一种“输入输出对齐”的编排方式。

只要每一段的接口对得上，LangChain 就能把它们组合成更大的 Runnable。

---

## 七、示例 3：查看链的类型

为了确认 `|` 背后发生了什么，可以打印链的类型。

新建文件：

```text
examples/lesson_06_chain_type.py
```

代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI


load_dotenv()


prompt = ChatPromptTemplate.from_template("请用一句话解释：{topic}")

model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.2,
)

parser = StrOutputParser()

chain = prompt | model | parser

print(type(chain))
```

运行：

```powershell
uv run examples/lesson_06_chain_type.py
```

你通常会看到类似：

```text
<class 'langchain_core.runnables.base.RunnableSequence'>
```

这说明：

- `prompt | model | parser` 创建的是顺序链。
- 顺序链本身也是 Runnable。
- 你可以继续把它接到其他 Runnable 后面。

---

## 八、`|` 和 `.pipe()` 的关系

官方参考中也提供了 `.pipe()` 形式。

下面两种写法表达的含义相同：

```python
chain = prompt | model | parser
```

```python
chain = prompt.pipe(model).pipe(parser)
```

本教材优先使用 `|`，因为它更贴近数据从左到右流动的阅读顺序。

如果你所在团队不喜欢操作符重载，或者希望代码更显式，可以使用 `.pipe()`。

无论哪种写法，核心都不变：

```text
前一个 Runnable 的输出，会成为下一个 Runnable 的输入。
```

---

## 九、示例 4：同一个链支持 `batch()`

LCEL 的好处之一是：链组合完成后，整条链也可以批量调用。

新建文件：

```text
examples/lesson_06_chain_batch.py
```

代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI


load_dotenv()


prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "你是一名技术文章标题编辑。"),
        ("human", "请为下面这段内容生成一个中文标题：\n\n{text}"),
    ]
)

model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.3,
)

parser = StrOutputParser()

title_chain = prompt | model | parser

texts = [
    {"text": "Runnable 让不同 LangChain 组件拥有统一调用方式。"},
    {"text": "LCEL 可以把 Prompt、Model、Parser 组合成可复用链路。"},
    {"text": "结构化输出能让模型结果更适合进入业务系统。"},
]

titles = title_chain.batch(texts, config={"max_concurrency": 2})

for title in titles:
    print("-", title)
```

运行：

```powershell
uv run examples/lesson_06_chain_batch.py
```

请观察：

- 你没有自己写 `for` 循环逐个调模型。
- 每个输入仍然是符合 Prompt 变量要求的字典。
- 输出是字符串列表，因为链路最后是 `StrOutputParser()`。

`batch()` 适合处理多个相互独立的输入，例如批量生成标题、批量摘要、批量分类。

---

## 十、示例 5：同一个链支持 `stream()`

如果链路中的步骤支持流式处理，LCEL 链也可以用 `stream()`。

新建文件：

```text
examples/lesson_06_chain_stream.py
```

代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI


load_dotenv()


prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "你是一名写作教练，回答要清晰、分点。"),
        ("human", "请给初学者解释：{topic}"),
    ]
)

model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.2,
)

parser = StrOutputParser()

chain = prompt | model | parser

for chunk in chain.stream({"topic": "为什么 LCEL 适合组合 LangChain 组件"}):
    print(chunk, end="", flush=True)
```

运行：

```powershell
uv run examples/lesson_06_chain_stream.py
```

观察输出体验：

- `invoke()` 会等完整结果生成后一次性返回。
- `stream()` 会把内容逐段输出。

在命令行助手、聊天界面、长文本生成场景里，流式输出能显著改善等待体验。

---

## 十一、实践案例：文本处理链

现在做一个更接近真实项目的小练习：把一段粗糙说明改写成更适合文档的版本。

目标链路：

```text
输入原始说明
  ↓
Prompt 组织改写要求
  ↓
Model 生成改写结果
  ↓
Parser 提取文本
```

新建文件：

```text
examples/lesson_06_text_polish_chain.py
```

代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI


load_dotenv()


prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            "你是一名技术文档编辑，擅长把口语化说明改写成清晰、克制、准确的文档表达。",
        ),
        (
            "human",
            "请将下面内容改写为面向初学者的技术文档段落：\n\n{raw_text}",
        ),
    ]
)

model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.2,
)

parser = StrOutputParser()

polish_chain = prompt | model | parser

result = polish_chain.invoke(
    {
        "raw_text": "这个 LCEL 就是把几个东西用竖线连起来，然后数据会一路传下去，写起来比较省事。"
    }
)

print(result)
```

运行：

```powershell
uv run examples/lesson_06_text_polish_chain.py
```

请重点观察两件事：

1. 输入只需要提供 `raw_text`，不需要关心 PromptValue 和 AIMessage。
2. 链路内部仍然是清晰的三段：Prompt、Model、Parser。

这就是 LCEL 在项目中的价值：对外暴露简单输入输出，对内保留清晰可组合的步骤。

---

## 十二、什么时候适合使用 LCEL

LCEL 适合这些场景：

- Prompt、Model、Parser 这类线性流程。
- 文本生成、摘要、改写、分类、抽取等单向处理链。
- 需要复用同一条链做 `invoke()`、`batch()` 或 `stream()`。
- 希望把多个 Runnable 封装成一个更大的 Runnable。
- 想让链路结构比手写中间变量更清晰。

LCEL 不适合把所有复杂系统都硬写成一条长链。

如果流程中出现这些情况，后续更适合使用 LangGraph：

- 有循环。
- 有复杂分支。
- 有人工确认。
- 有步骤重试和状态恢复。
- 有多个 Agent 或长任务规划。

你可以先记住一句话：

> 线性组合优先用 LCEL，复杂状态流程后面用 LangGraph。

---

## 十三、常见错误与排查

### 1. 忘记最后接输出解析器

如果你写：

```python
chain = prompt | model
```

那么：

```python
result = chain.invoke({"topic": "LCEL"})
```

返回的通常是 `AIMessage`。

如果你希望得到字符串，应该写：

```python
chain = prompt | model | StrOutputParser()
```

### 2. 输入字典的键和 Prompt 变量不一致

如果 Prompt 中写的是：

```python
"请解释：{topic}"
```

调用时就要传：

```python
chain.invoke({"topic": "LCEL"})
```

如果传成：

```python
chain.invoke({"name": "LCEL"})
```

Prompt 模板无法找到需要的 `topic`。

### 3. 上一步输出不能被下一步接收

LCEL 不是魔法。它不会让任意两个组件都自动兼容。

你要确认：

```text
A 的输出类型，B 能接收。
```

排查方法仍然很简单：

```python
value = prompt.invoke({"topic": "LCEL"})
print(type(value))
```

先单独调用每一步，看清楚输入输出，再组合成链。

### 4. 把 LCEL 链写得过长

LCEL 让链式组合很方便，但链太长也会降低可读性。

如果一条链已经很难一眼看清，可以拆成几个有名字的小链：

```python
draft_chain = draft_prompt | model | parser
review_chain = review_prompt | model | parser
```

清晰的命名比把所有步骤塞进一行更重要。

### 5. 误以为 `batch()` 是服务商的离线批处理

和第 5 课一样，LCEL 链上的 `batch()` 通常表示对多个输入进行客户端并发调用。

它适合多个独立请求的批量处理，但不等同于模型服务商提供的离线 Batch API。

---

## 十四、本课小结

这一课你把 Runnable 从“单个组件”推进到了“链式组合”。

你需要记住：

- LCEL 是 LangChain 中组合 Runnable 的表达式方式。
- `prompt | model | parser` 会创建顺序执行的 Runnable 链。
- 这个链通常是 `RunnableSequence`。
- 链中的关键规则是：前一步输出必须能作为后一步输入。
- 组合后的链本身仍然是 Runnable。
- 因此同一条链可以继续使用 `invoke()`、`batch()` 和 `stream()`。
- `StrOutputParser()` 常用于把模型消息转换成普通字符串。

LCEL 的价值不是少写几行代码，而是让 LLM 应用的数据流更清晰、更容易复用、更容易扩展。

---

## 十五、课后练习

### 练习 1：手动三步改成 LCEL

创建：

```text
examples/lesson_06_rewrite_manual_to_lcel.py
```

要求：

- 先用 `prompt.invoke()`、`model.invoke()`、`parser.invoke()` 手动执行。
- 再改成 `prompt | model | parser`。
- 比较两种写法的输出是否一致。

### 练习 2：批量生成摘要

创建：

```text
examples/lesson_06_batch_summaries.py
```

要求：

- 准备 3 段短文本。
- 使用 `summary_chain.batch(...)` 批量生成一句话摘要。
- 设置 `config={"max_concurrency": 2}`。
- 打印每段文本对应的摘要。

### 练习 3：流式生成解释

创建：

```text
examples/lesson_06_stream_explanation.py
```

要求：

- 创建 `prompt | model | StrOutputParser()`。
- 使用 `stream()` 解释一个你不熟悉的 LangChain 概念。
- 用 `print(chunk, end="", flush=True)` 输出。

### 练习 4：分析输入输出

选择本课任意一个链，写下注释：

```text
输入：
prompt 输出：
model 输出：
parser 输出：
最终结果：
```

不要跳过这一步。LCEL 的熟练程度，很大程度来自你是否能稳定判断数据在链路中的形态。

### 练习 5：封装一个小链

创建一个变量名清晰的小链，例如：

```python
title_chain = prompt | model | parser
```

要求：

- 输入一段文章。
- 输出一个标题。
- 分别用 `invoke()` 和 `batch()` 调用。
- 思考：如果后续项目要复用它，应该把它放在哪个模块里？

---

## 十六、下一课预告

下一课进入：

```text
第 7 课：输出解析与格式转换
```

你会继续学习：

- Output Parser 在链路中负责什么。
- 为什么模型消息通常还需要转换成业务可用的数据。
- 如何把模型输出解析成字符串、列表、JSON 或 Pydantic 对象。
- 在已经学过 `with_structured_output()` 的前提下，什么时候仍然需要 Output Parser。

第 6 课解决的是线性链式组合；第 7 课继续深入链路最后一步：怎样把模型输出变成程序更容易使用的结果。

---

## 十七、官方参考

- LangChain Core Runnables API Reference: `https://reference.langchain.com/python/langchain-core/runnables`
- Runnable Reference: `https://reference.langchain.com/python/langchain-core/runnables/base/Runnable`
- RunnableSequence Reference: `https://reference.langchain.com/python/langchain-core/runnables/base/RunnableSequence`
- LangChain Models - Invocation / Stream / Batch: `https://docs.langchain.com/oss/python/langchain/models`
- LLMChain Deprecated Notice: `https://api.python.langchain.com/en/latest/langchain/chains/langchain.chains.llm.LLMChain.html`
