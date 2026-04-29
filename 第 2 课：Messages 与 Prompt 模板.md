# 第 2 课：Messages 与 Prompt 模板

> 本课关键词：**SystemMessage、HumanMessage、AIMessage、PromptTemplate、ChatPromptTemplate、MessagesPlaceholder、提示词即接口**

---

## 一、本课目标

学完本课，你应该能做到：

1. 理解聊天模型为什么使用 Messages，而不是只使用普通字符串。
2. 掌握 `SystemMessage`、`HumanMessage`、`AIMessage` 的职责。
3. 掌握 `PromptTemplate` 的基础用法。
4. 掌握 `ChatPromptTemplate` 的基础用法。
5. 能写出一个可复用、可维护的 Prompt 模板，并通过 `uv run` 运行示例代码。

上一课我们已经跑通了最小模型调用：

```python
response = model.invoke("请用三句话解释什么是 LangChain。")
```

这一课要进一步学习：怎样把提示词从“临时字符串”升级成“稳定接口”。

---

## 二、为什么 Prompt 不能一直手写字符串

先想一个很真实的小需求。

你现在要做一个“技术概念解释器”，用户每次输入一个概念，程序都要让模型按固定要求解释：

- 学习者是谁。
- 要解释哪个概念。
- 输出要包含定义、类比和练习题。

如果只做一次，我们很容易这样写：

```python
question = "什么是 LangChain？"
prompt = f"请用初学者能听懂的方式解释：{question}"
```

这在小脚本里没问题。

但你马上会遇到几个困难。

### 困难 1：输入会变化

今天用户问：

```text
LangChain
```

明天用户问：

```text
Runnable
```

后天用户又希望面向不同水平的人解释：

```text
Python 初学者
后端工程师
产品经理
```

如果每次都手写完整 Prompt，代码很快会变乱。

### 困难 2：Prompt 结构会重复

你真正想复用的是这个结构：

```text
请用{学习者水平}能听懂的方式解释{概念}。
要求包含定义、类比、练习题。
```

其中会变化的是：

- 学习者水平。
- 技术概念。

不应该每次都重写整段提示词。

### 困难 3：变量越多，字符串拼接越容易错

一开始只有一个变量：

```python
concept = "LangChain"
```

后来可能变成三个变量：

```python
audience = "Python 初学者"
concept = "Runnable"
output_style = "通俗但准确"
```

再后来还可能有：

- 回答长度。
- 输出语言。
- 是否包含代码。
- 是否包含练习题。

这时如果继续用普通字符串拼接，容易出现变量漏传、格式混乱、Prompt 到处复制的问题。

### 困难 4：Prompt 难以像函数一样复用

一个普通函数可以这样复用：

```python
def explain(concept, audience):
    ...
```

那 Prompt 也应该有类似能力：

```text
固定模板 + 动态参数 = 每次生成不同 Prompt
```

LangChain 的 Prompt 模板就是为了解决这个问题。

它让我们把 Prompt 拆成两部分：

```text
不变的任务结构：模板
会变化的输入：变量
```

所以，Prompt 模板不是为了让代码看起来高级，而是为了解决真实开发中这几个问题：

- Prompt 散落在代码各处，难维护。
- 变量越来越多，字符串拼接容易出错。
- System 规则、用户问题、历史对话混在一起。
- 很难复用同一个 Prompt 结构。
- 很难检查一个模型调用到底收到了什么输入。

你可以把 Prompt 模板理解成：

> 给模型输入设计的函数签名。

普通函数有参数、有返回值；Prompt 模板也应该有清晰的输入变量、角色分工和输出要求。

---

## 三、Messages 基础复习

聊天模型通常不是只看一段字符串，而是看一组消息。

常见消息类型：

| 消息类型 | 角色 | 典型用途 |
| --- | --- | --- |
| `SystemMessage` | 系统规则 | 设定角色、边界、输出风格和约束 |
| `HumanMessage` | 用户输入 | 表达用户的问题、任务和补充信息 |
| `AIMessage` | 模型回复 | 保存模型已经说过的话 |
| `ToolMessage` | 工具结果 | 保存工具调用后的返回结果，后面 Agent 阶段再学 |

第一阶段重点掌握前三个。

### 1. SystemMessage

`SystemMessage` 负责告诉模型“你是谁、该遵守什么规则”。

示例：

```python
SystemMessage(content="你是一名严谨的 LangChain 老师，回答要清晰、分层、适合初学者。")
```

适合放在 System 中的内容：

- 角色定位。
- 回答风格。
- 安全边界。
- 输出格式要求。
- 不确定时如何处理。

不适合放太多业务数据。业务数据通常放在 Human 或后续 RAG 的上下文里。

### 2. HumanMessage

`HumanMessage` 表示用户输入。

示例：

```python
HumanMessage(content="请解释 PromptTemplate 和 ChatPromptTemplate 的区别。")
```

适合放在 Human 中的内容：

- 用户问题。
- 当前任务。
- 待处理文本。
- 业务上下文。
- 输出要求。

### 3. AIMessage

`AIMessage` 表示模型之前的回复，常用于多轮对话。

示例：

```python
AIMessage(content="当然，我们先从 Messages 开始讲。")
```

如果要模拟历史对话，就可以把 Human 和 AI 消息交替放入消息列表。

---

## 四、示例 1：手写 Messages

新建文件：

```text
examples/lesson_02_manual_messages.py
```

代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain.messages import AIMessage, HumanMessage, SystemMessage
from langchain_openai import ChatOpenAI


load_dotenv()


model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.2,
)

messages = [
    SystemMessage(content="你是一名耐心的 LangChain 老师，擅长循序渐进地讲解。"),
    HumanMessage(content="我刚开始学习 LangChain。"),
    AIMessage(content="很好，我们先从模型调用、Messages 和 Prompt 模板开始。"),
    HumanMessage(content="请解释 SystemMessage 和 HumanMessage 的区别。"),
]

response = model.invoke(messages)

print(response.content)
```

运行：

```powershell
uv run examples/lesson_02_manual_messages.py
```

观察重点：

- 模型会参考前面的对话历史。
- `SystemMessage` 会影响回答风格。
- `AIMessage` 能让模型知道前面已经讲过什么。

---

## 五、PromptTemplate：字符串模板

现在我们正式引入第一个新概念：`PromptTemplate`。

先不要急着记 API。先看它解决什么问题。

上一节的问题可以总结成一句话：

> 我们希望保留一段固定 Prompt 结构，只在运行时传入不同参数。

这就是 `PromptTemplate` 的作用。

`PromptTemplate` 是 LangChain 中用于生成普通字符串 Prompt 的模板工具。它会把模板里的占位符，例如 `{audience}`、`{concept}`，替换成运行时传入的真实值。

你可以把它类比成一个函数：

```python
def build_prompt(audience, concept):
    return f"请用{audience}能听懂的方式解释：{concept}"
```

只不过 `PromptTemplate` 是 LangChain 体系里的 Prompt 构建方式。它可以被 `invoke()` 调用，也可以和模型、解析器继续组合。

它适合这类场景：

- 单轮文本处理。
- 摘要、改写、翻译。
- 不是严格聊天结构的简单任务。

它的核心思路是：

```text
模板字符串 + 变量字典 → 渲染后的 Prompt
```

例如：

```text
模板：请用{audience}能听懂的方式解释：{concept}
变量：audience = Python 初学者，concept = Runnable
结果：请用Python 初学者能听懂的方式解释：Runnable
```

### 示例 2：用 PromptTemplate 生成提示词

新建文件：

```text
examples/lesson_02_prompt_template.py
```

代码：

```python
from langchain_core.prompts import PromptTemplate


template = PromptTemplate.from_template(
    """
请用{audience}能听懂的方式解释下面这个概念：

概念：{concept}

要求：
1. 先用一句话定义
2. 再用一个生活类比
3. 最后给一个练习题
"""
)

prompt_value = template.invoke(
    {
        "audience": "Python 初学者",
        "concept": "LangChain Runnable",
    }
)

print(prompt_value.text)
```

运行：

```powershell
uv run examples/lesson_02_prompt_template.py
```

这里先不调用模型，只观察模板渲染后的结果。

重点理解：

- `{audience}` 和 `{concept}` 是模板变量。
- `template.invoke({...})` 传入的是变量字典。
- 字典里的 key 必须和模板变量名一致。
- 返回的 `prompt_value` 是 LangChain 的 PromptValue，不只是普通字符串。

```python
template.invoke({...})
```

Prompt 模板也可以被 `invoke()` 调用。它接收变量字典，输出一个 PromptValue。

这就是 LangChain 统一接口思想的早期体现。

到这里，你应该能看出动态参数传递的价值：

```text
同一个模板，可以服务不同用户、不同概念、不同输出要求。
```

这就是从“写死 Prompt”升级到“可复用 Prompt 接口”的第一步。

---

## 六、PromptTemplate + Model

接下来把 Prompt 模板和模型连起来。

新建文件：

```text
examples/lesson_02_prompt_template_with_model.py
```

代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain_core.prompts import PromptTemplate
from langchain_openai import ChatOpenAI


load_dotenv()


model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.2,
)

template = PromptTemplate.from_template(
    """
请用{audience}能听懂的方式解释下面这个概念：

概念：{concept}

要求：
1. 一句话定义
2. 通俗解释
3. 一个生活类比
4. 一个练习题
"""
)

prompt_value = template.invoke(
    {
        "audience": "Python 初学者",
        "concept": "LangChain PromptTemplate",
    }
)

response = model.invoke(prompt_value)

print(response.content)
```

运行：

```powershell
uv run examples/lesson_02_prompt_template_with_model.py
```

这一版的数据流是：

```text
变量字典 → PromptTemplate → PromptValue → Chat Model → AIMessage → 文本内容
```

---

## 七、ChatPromptTemplate：聊天模板

现在再看一个新的困难。

`PromptTemplate` 已经能解决“固定模板 + 动态参数”的问题，但它生成的是一整段普通字符串。

如果任务只是摘要、翻译、改写，这通常够用。

但聊天模型还有一个更重要的特点：它区分不同角色的消息。

比如我们希望：

```text
System：你是一名严谨的 LangChain 老师。
Human：请解释 ChatPromptTemplate。
```

如果继续用普通字符串，可能会写成：

```text
你是一名严谨的 LangChain 老师。

用户问题：请解释 ChatPromptTemplate。
```

这虽然也能运行，但会遇到几个问题：

- System 规则和用户问题混在同一个字符串里。
- 后续加入历史对话时，消息角色不清楚。
- 做多轮聊天、Agent、工具调用时，不利于维护消息结构。
- 很难检查最终传给模型的是哪些角色消息。

所以，当我们面对聊天模型时，更推荐使用 `ChatPromptTemplate`。

它解决的是：

```text
既要支持动态参数，又要保留 system / human / ai 等消息角色。
```

换句话说，`PromptTemplate` 主要解决“字符串模板化”；`ChatPromptTemplate` 解决“聊天消息模板化”。

因为它能明确表达：

- 哪些内容是 System 规则。
- 哪些内容是用户输入。
- 哪些内容是历史对话。
- 哪些内容是模型曾经回复过的。

### 示例 3：基础 ChatPromptTemplate

新建文件：

```text
examples/lesson_02_chat_prompt_template.py
```

代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI


load_dotenv()


model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.2,
)

prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "你是一名{subject}老师，回答要清晰、准确、适合{level}。"),
        ("human", "请解释这个概念：{concept}"),
    ]
)

prompt_value = prompt.invoke(
    {
        "subject": "LangChain",
        "level": "初学者",
        "concept": "ChatPromptTemplate",
    }
)

response = model.invoke(prompt_value)

print(response.content)
```

运行：

```powershell
uv run examples/lesson_02_chat_prompt_template.py
```

这里的：

```python
("system", "...")
("human", "...")
```

会被 LangChain 转换成对应的消息对象。

---

## 八、查看 ChatPromptTemplate 生成了什么

很多初学者写 Prompt 时最大的问题是：不知道模型最终收到了什么。

我们可以先打印模板生成的消息。

新建文件：

```text
examples/lesson_02_inspect_chat_prompt.py
```

代码：

```python
from langchain_core.prompts import ChatPromptTemplate


prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "你是一名{subject}老师，回答要适合{level}。"),
        ("human", "请解释这个概念：{concept}"),
    ]
)

prompt_value = prompt.invoke(
    {
        "subject": "LangChain",
        "level": "初学者",
        "concept": "Messages",
    }
)

for message in prompt_value.messages:
    print(type(message).__name__)
    print(message.content)
    print("---")
```

运行：

```powershell
uv run examples/lesson_02_inspect_chat_prompt.py
```

你应该能看到类似结构：

```text
SystemMessage
你是一名LangChain老师，回答要适合初学者。
---
HumanMessage
请解释这个概念：Messages
---
```

这个检查习惯非常重要。以后调试 RAG、Agent、LangGraph 时，也要经常看中间输入。

---

## 九、MessagesPlaceholder：放入历史对话

再看一个多轮对话里的真实困难。

前面的 `ChatPromptTemplate` 能固定写好 system 和 human 消息：

```python
prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "你是一名 LangChain 老师。"),
        ("human", "{question}"),
    ]
)
```

但如果用户已经聊了几轮，问题就来了：

```text
第 1 轮 Human：我刚学完 invoke。
第 1 轮 AI：invoke 是给组件输入并得到输出的统一方法。
第 2 轮 Human：我又学了 SystemMessage 和 HumanMessage。
第 2 轮 AI：它们分别用于系统规则和用户输入。
第 3 轮 Human：那 ChatPromptTemplate 解决了什么问题？
```

这里的历史消息数量不是固定的。

有时 2 条，有时 20 条。我们不可能在模板里提前写死每一条历史消息。

这时需要一个“动态插槽”：

```text
system 固定
history 运行时放入一组历史消息
human 当前问题
```

这个动态插槽就是 `MessagesPlaceholder`。

它的作用是：

> 在 ChatPromptTemplate 中预留一个位置，运行时再放入一组消息。

所以它解决的是：

```text
固定聊天模板中，如何动态插入不定数量的历史消息。
```

### 示例 4：带历史对话的 Prompt

新建文件：

```text
examples/lesson_02_messages_placeholder.py
```

代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain.messages import AIMessage, HumanMessage
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_openai import ChatOpenAI


load_dotenv()


model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.2,
)

prompt = ChatPromptTemplate.from_messages(
    [
        ("system", "你是一名 LangChain 老师。请结合历史对话回答用户的新问题。"),
        MessagesPlaceholder(variable_name="history"),
        ("human", "{question}"),
    ]
)

history = [
    HumanMessage(content="我刚学完 invoke。"),
    AIMessage(content="很好，invoke 是给 LangChain 组件输入并得到输出的统一方法。"),
    HumanMessage(content="我又学了 SystemMessage 和 HumanMessage。"),
    AIMessage(content="它们分别用于系统规则和用户输入。"),
]

prompt_value = prompt.invoke(
    {
        "history": history,
        "question": "那 ChatPromptTemplate 解决了什么问题？",
    }
)

response = model.invoke(prompt_value)

print(response.content)
```

运行：

```powershell
uv run examples/lesson_02_messages_placeholder.py
```

注意：

- `history` 必须是一组消息。
- `MessagesPlaceholder` 很适合多轮对话。
- 后面做聊天机器人、Agent、LangGraph 状态时会经常遇到类似结构。

---

## 十、提示词即接口

从这一课开始，要逐渐形成一个重要意识：

> Prompt 不是随手写给模型看的作文，而是程序和模型之间的接口。

一个好的 Prompt 接口应该尽量清晰：

| 设计项 | 问题 |
| --- | --- |
| 角色 | 模型应该以什么身份回答？ |
| 任务 | 模型到底要完成什么？ |
| 输入 | 模型会收到哪些变量？ |
| 约束 | 有哪些不能做或必须遵守的规则？ |
| 输出 | 应该返回什么结构或格式？ |
| 失败处理 | 不确定、缺信息、无法判断时怎么办？ |

### 不太好的 Prompt

```text
帮我看看这个内容。
```

问题：

- 没有角色。
- 没有任务边界。
- 没有输出格式。
- 没有评价标准。

### 更好的 Prompt

```text
你是一名资深 Python 老师。

请阅读下面的学习笔记，完成三件事：
1. 找出不准确的地方
2. 给出更正后的解释
3. 设计一道检查理解的练习题

学习笔记：
{note}

如果笔记内容不足以判断，请明确说明“信息不足”，不要编造。
```

这个 Prompt 更像接口，因为它说明了：

- 谁来做。
- 做什么。
- 输入是什么。
- 输出什么。
- 不确定时怎么办。

---

## 十一、实践案例：技术概念学习卡片生成器

现在把本课知识组合起来，做一个比上一课更规范的实践案例。

新建文件：

```text
examples/lesson_02_concept_card.py
```

代码：

```python
from os import getenv

from dotenv import load_dotenv
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI


load_dotenv()


model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.2,
)

prompt = ChatPromptTemplate.from_messages(
    [
        (
            "system",
            """
你是一名高级技术课程老师。
你的任务是把复杂技术概念讲得准确、清晰、适合初学者。
如果用户的问题不够明确，请先指出缺失信息，再给出合理学习建议。
""",
        ),
        (
            "human",
            """
请为下面这个技术概念生成一张学习卡片。

概念：{concept}
学习者水平：{level}
学习目标：{goal}

请按以下结构输出：
1. 一句话定义
2. 核心作用
3. 它解决的问题
4. 一个简单例子
5. 常见误区
6. 一个课后练习
""",
        ),
    ]
)

concept = input("请输入技术概念：")
level = input("请输入学习者水平：")
goal = input("请输入学习目标：")

prompt_value = prompt.invoke(
    {
        "concept": concept,
        "level": level,
        "goal": goal,
    }
)

response = model.invoke(prompt_value)

print("\n=== 技术概念学习卡片 ===\n")
print(response.content)
```

运行：

```powershell
uv run examples/lesson_02_concept_card.py
```

可以输入：

```text
技术概念：ChatPromptTemplate
学习者水平：刚学完 Python 函数
学习目标：能理解它和普通字符串 Prompt 的区别
```

这个案例的重点不是模型回答多漂亮，而是 Prompt 结构更清楚了。

---

## 十二、常见错误与排查

### 1. 变量名不匹配

如果模板里写了：

```python
"请解释：{concept}"
```

调用时却传：

```python
{"topic": "LangChain"}
```

就会报错，因为模板需要的是 `concept`，不是 `topic`。

正确写法：

```python
{"concept": "LangChain"}
```

### 2. 花括号冲突

Prompt 模板中 `{name}` 会被识别为变量。

如果你真的想输出花括号，例如 JSON 示例：

```text
{"name": "张三"}
```

需要小心，因为模板可能把 `{` 和 `}` 当成变量边界。初学阶段建议先少在普通 Prompt 模板中直接写复杂 JSON 示例；后面学结构化输出时会用更规范的方法解决。

### 3. 忘记调用 `load_dotenv()`

如果代码需要调用模型，却没有读取 `.env`，可能会找不到 API Key。

确认代码里有：

```python
from dotenv import load_dotenv

load_dotenv()
```

### 4. 把字符串模板当成聊天模板

`PromptTemplate` 更适合普通字符串任务。

`ChatPromptTemplate` 更适合聊天模型，因为它能明确区分 system、human、ai 等角色。

本课程后续多数聊天模型示例会优先使用 `ChatPromptTemplate`。

### 5. 历史对话不是消息列表

`MessagesPlaceholder` 需要的是消息列表，例如：

```python
[
    HumanMessage(content="你好"),
    AIMessage(content="你好，我可以帮你什么？"),
]
```

不要直接传一整段历史字符串，除非你就是想把它当成普通文本上下文。

---

## 十三、本课小结

这一课你完成了从“手写提示词”到“模板化提示词”的升级：

- Messages 用来表达聊天模型的角色化输入。
- `SystemMessage` 负责规则和角色。
- `HumanMessage` 负责用户任务。
- `AIMessage` 负责历史回复。
- `PromptTemplate` 适合普通字符串模板。
- `ChatPromptTemplate` 适合聊天模型模板。
- `MessagesPlaceholder` 可以动态插入历史消息。
- Prompt 应该被当成程序和模型之间的接口来设计。

请记住：

> 写 Prompt 不是堆更多文字，而是把角色、任务、输入、约束和输出讲清楚。

---

## 十四、课后练习

### 练习 1：改造上一课代码

把第 1 课的 `lesson_01_concept_teacher.py` 改造成 `ChatPromptTemplate` 版本。

要求：

- 使用 `system` 消息定义老师角色。
- 使用 `human` 消息接收用户输入。
- 至少包含 3 个变量。
- 使用 `uv run` 运行。

### 练习 2：设计一个代码解释 Prompt

创建：

```text
examples/lesson_02_code_explainer.py
```

要求输入：

- 编程语言。
- 代码片段。
- 学习者水平。

要求输出：

- 代码整体作用。
- 逐段解释。
- 关键语法。
- 可能的改进建议。

### 练习 3：加入历史对话

基于 `MessagesPlaceholder` 写一个多轮学习助手。

要求：

- 预置两轮历史对话。
- 用户再提出一个新问题。
- 模型必须结合历史对话回答。

### 练习 4：整理学习笔记

请用自己的话回答：

1. 为什么聊天模型需要 Messages？
2. `PromptTemplate` 和 `ChatPromptTemplate` 的区别是什么？
3. `MessagesPlaceholder` 解决什么问题？
4. 为什么说 Prompt 是接口？
5. 一个好的 Prompt 通常应该包含哪些要素？

---

## 十五、下一课预告

下一课进入：

```text
第 3 课：模型输出与基础解析
```

你会学习：

- 模型返回值到底是什么。
- `AIMessage.content` 和完整消息对象的区别。
- 为什么自由文本不适合程序直接使用。
- 字符串输出解析的基础方式。
- 为第 4 课结构化输出做准备。

第 2 课解决“怎么把问题规范地交给模型”；第 3 课开始解决“模型答完以后，程序怎么接住结果”。

---

## 十六、官方参考

- LangChain Messages: `https://docs.langchain.com/oss/python/langchain/messages`
- LangChain Core Prompts Reference: `https://reference.langchain.com/python/langchain-core/prompts`
- ChatPromptTemplate Reference: `https://reference.langchain.com/python/langchain-core/prompts/#langchain_core.prompts.chat.ChatPromptTemplate`
- MessagesPlaceholder Reference: `https://reference.langchain.com/python/langchain-core/prompts/#langchain_core.prompts.chat.MessagesPlaceholder`
- LangChain v1 release notes: `https://docs.langchain.com/oss/python/releases/langchain-v1`
