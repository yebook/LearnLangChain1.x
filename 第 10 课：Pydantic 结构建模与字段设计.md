# 第 10 课：Pydantic 结构建模与字段设计

> 本课关键词：**Pydantic、BaseModel、Field、字段描述、Literal、可选字段、列表字段、schema 复杂度**

---

## 一、本课目标

学完本课，你应该能做到：

1. 理解 schema 设计为什么会直接影响结构化输出质量。
2. 为 LLM 输出设计清晰、稳定、不过度复杂的 Pydantic 模型。
3. 正确使用 `Field(description=...)` 描述字段含义。
4. 使用 `Literal[...]` 表达有限候选值。
5. 使用 `类型 | None = Field(default=None, ...)` 表达可选字段。
6. 设计列表字段，并知道什么时候不该嵌套太深。
7. 完成一个简历信息 schema 设计示例。

前面你已经见过结构化输出的基本调用方式。现在要往前走一步：真正难的常常不是“怎么调用结构化输出”，而是“你给模型的结构到底设计得是否适合它稳定填写”。

---

## 二、为什么 schema 设计很重要

结构化输出让模型按固定字段返回数据，但 schema 不是随便列几个字段就够了。

想象你让模型从简历里提取信息。如果 schema 写成这样：

```python
class ResumeInfo(BaseModel):
    info: str
    level: str
    result: str
```

这三个字段都太含糊：

- `info` 是基础信息、工作经历，还是全部内容？
- `level` 是学历等级、能力等级，还是候选人评级？
- `result` 是提取结果、推荐结论，还是处理状态？

模型并不会自动知道你的业务含义。字段名和字段描述越模糊，输出越容易飘。

更好的 schema 应该把业务含义说清楚：

```python
class ResumeInfo(BaseModel):
    candidate_name: str | None = Field(default=None, description="候选人姓名，如果简历中没有出现则为 None")
    highest_education: str | None = Field(default=None, description="候选人最高学历，例如本科、硕士、博士")
    years_of_experience: int | None = Field(default=None, description="候选人总工作年限，无法判断时为 None")
```

这不是为了代码好看，而是为了让模型、校验器和后续程序对“每个字段是什么意思”达成一致。

---

## 三、一个好 schema 要回答什么问题

设计 Pydantic schema 时，先不要急着写代码。你可以先问自己几个问题：

```text
这个字段是否真的会被后续程序使用？
这个字段是必填，还是可能缺失？
这个字段应该是自由文本，还是有限候选值？
这个字段是单个值，还是列表？
这个字段能不能从原文中稳定判断？
字段描述是否足够具体，能避免模型误解？
```

好 schema 的目标不是“覆盖所有可能信息”，而是让模型稳定返回对程序有用的数据。

在 LLM 应用里，schema 同时扮演三种角色：

| 角色 | 说明 |
| --- | --- |
| 给模型的任务说明 | 字段名和描述告诉模型要抽取什么 |
| 给程序的类型约定 | 类型标注告诉程序会拿到什么形态的数据 |
| 给校验器的检查规则 | Pydantic 会根据类型和默认值校验结果 |

---

## 四、示例 1：查看 Pydantic 生成的 schema

先不调用模型，只看 Pydantic 会把你的模型转换成什么结构。

新建文件：

```text
examples/lesson_10_schema_preview.py
```

代码：

```python
from pprint import pprint
from typing import Literal

from pydantic import BaseModel, Field


class ResumeInfo(BaseModel):
    candidate_name: str | None = Field(
        default=None,
        description="候选人姓名，如果简历中没有出现则为 None",
    )
    highest_education: Literal["高中", "大专", "本科", "硕士", "博士", "未知"] = Field(
        description="候选人最高学历，只能从给定选项中选择"
    )
    years_of_experience: int | None = Field(
        default=None,
        description="候选人总工作年限，无法判断时为 None",
    )
    core_skills: list[str] = Field(
        description="候选人的核心技能列表，只提取简历中明确出现的技能"
    )


pprint(ResumeInfo.model_json_schema())
```

运行：

```powershell
uv run examples/lesson_10_schema_preview.py
```

这里出现了几个新写法，需要逐个看清楚。

`model_json_schema()` 是 Pydantic 模型的方法，用来查看这个模型对应的 JSON Schema。它不会调用大模型，只是帮助你检查字段名、类型、候选值和描述是否已经进入 schema。

`Literal["高中", "大专", ...]` 来自 Python 的 `typing` 模块，表示这个字段只能取这些固定值之一。它很适合表达分类、等级、状态、优先级这类有限选项。

`str | None` 表示这个字段的值可以是字符串，也可以是 `None`。后面的 `Field(default=None, ...)` 表示如果没有这个字段，可以默认设为 `None`。这两个部分含义不同：

```text
类型 | None：值允许为空。
default=None：字段缺失时使用 None 作为默认值。
```

`list[str]` 表示字符串列表，适合技能、标签、职责、证书这类多项结果。

请重点观察输出里的 `description`、`enum`、`type` 等信息。它们就是模型和程序会共同依赖的结构说明。

---

## 五、字段名：让业务含义浮出来

字段名要具体，不要只写抽象名词。

不推荐：

```python
name: str
level: str
items: list[str]
result: str
```

更推荐：

```python
candidate_name: str
highest_education: str
core_skills: list[str]
recommendation_reason: str
```

字段名应该让人一眼知道这个字段要装什么。

命名建议：

| 场景 | 推荐命名 |
| --- | --- |
| 人名 | `candidate_name`、`contact_name` |
| 分类结果 | `ticket_type`、`intent_category` |
| 等级 | `priority_level`、`risk_level` |
| 列表 | `core_skills`、`missing_information` |
| 原因说明 | `reason`、`recommendation_reason` |

字段名越清楚，后续程序越容易读；字段名越含糊，Prompt 里就要花更多力气解释。

---

## 六、字段描述：写给模型看的细粒度说明

`Field(description=...)` 不只是文档注释。对于结构化输出，它也是给模型的重要提示。

字段描述应该说明：

- 这个字段表示什么。
- 什么时候填具体值。
- 什么时候填 `None` 或默认值。
- 是否只能从原文中提取。
- 是否允许推断。
- 输出单位或格式是什么。

例如：

```python
years_of_experience: int | None = Field(
    default=None,
    description="候选人总工作年限，只填写整数年；如果无法从简历判断则为 None",
)
```

这个描述比下面这种更好：

```python
years_of_experience: int | None = Field(description="工作年限")
```

因为它额外说明了：

- 输出整数年。
- 无法判断时返回 `None`。
- 不鼓励模型随便猜。

字段描述越接近业务规则，结构化输出越容易进入后续程序。

---

## 七、示例 2：简历 schema 设计

现在设计一个更完整的简历信息 schema。

新建文件：

```text
examples/lesson_10_resume_schema.py
```

代码：

```python
from os import getenv
from typing import Literal

from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field


load_dotenv()


class ResumeInfo(BaseModel):
    candidate_name: str | None = Field(
        default=None,
        description="候选人姓名，如果简历中没有出现则为 None",
    )
    current_title: str | None = Field(
        default=None,
        description="候选人当前或最近一份工作的职位名称，如果无法判断则为 None",
    )
    highest_education: Literal["高中", "大专", "本科", "硕士", "博士", "未知"] = Field(
        description="候选人最高学历，只能从给定选项中选择；无法判断时选择 未知"
    )
    years_of_experience: int | None = Field(
        default=None,
        description="候选人总工作年限，只填写整数年；无法从简历判断时为 None",
    )
    core_skills: list[str] = Field(
        description="候选人的核心技能列表，只提取简历中明确出现的技能"
    )
    match_level: Literal["high", "medium", "low", "unknown"] = Field(
        description="与 Python 后端开发岗位的匹配程度：high、medium、low 或 unknown"
    )
    match_reason: str = Field(
        description="用一句中文说明匹配程度的主要依据"
    )


model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0,
)

structured_model = model.with_structured_output(ResumeInfo)

resume_text = """
候选人：李明
最近职位：后端开发工程师
教育背景：2018 年毕业于某大学计算机科学与技术本科。
工作经历：5 年 Python 后端开发经验，熟悉 FastAPI、PostgreSQL、Redis 和 Docker。
项目经验：参与过企业内部知识库系统和数据同步平台建设。
"""

result = structured_model.invoke(
    f"请从下面简历中提取结构化信息，并评估其与 Python 后端开发岗位的匹配程度：\n\n{resume_text}"
)

print(result)
print(result.candidate_name)
print(result.core_skills)
```

运行：

```powershell
uv run examples/lesson_10_resume_schema.py
```

请观察：

- `highest_education` 只能从固定中文选项里选。
- `match_level` 只能从 `high / medium / low / unknown` 中选。
- `years_of_experience` 可以是整数，也可以是 `None`。
- `core_skills` 是列表，不是逗号分隔字符串。

这个示例的重点是 schema 设计，而不是再次学习结构化输出调用。你要关注的是：字段名、字段类型和字段描述怎样共同约束模型输出。

---

## 八、什么时候用 `Literal`

`Literal` 适合字段值只有少数候选项的场景。

例如：

```python
priority: Literal["low", "medium", "high"]
```

它表达的不是“给模型一个建议”，而是“这个字段只能属于这些值”。

适合使用 `Literal` 的场景：

| 场景 | 示例 |
| --- | --- |
| 紧急程度 | `Literal["low", "medium", "high"]` |
| 工单类型 | `Literal["bug", "feature", "consulting", "other"]` |
| 是否推荐 | `Literal["yes", "no", "unknown"]` |
| 匹配等级 | `Literal["high", "medium", "low", "unknown"]` |

不适合使用 `Literal` 的场景：

- 姓名。
- 公司名。
- 自由文本原因。
- 任意技能列表。
- 任意地址或链接。

如果一个字段本来就应该自由填写，就不要硬塞进枚举选项里。

---

## 九、可选字段：不要让模型被迫编造

真实文本经常缺信息。比如简历里可能没有手机号、没有期望薪资、没有毕业年份。

如果字段写成必填：

```python
expected_salary: str = Field(description="候选人期望薪资")
```

模型可能会为了满足 schema 而猜一个值。

更稳妥的写法是：

```python
expected_salary: str | None = Field(
    default=None,
    description="候选人期望薪资；如果简历中没有出现则为 None",
)
```

这给了模型一个合法出口：没有信息时返回 `None`。

可选字段特别适合：

- 联系方式。
- 薪资期望。
- 毕业年份。
- 当前公司。
- 证书编号。
- 项目上线时间。

不要把所有字段都设成必填。必填字段应该是业务上确实必须有、且输入文本大概率能提供的信息。

---

## 十、列表字段：只放同一种东西

`list[str]` 很适合表示多个同类信息。

例如：

```python
core_skills: list[str] = Field(
    description="候选人的核心技能列表，只提取简历中明确出现的技能"
)
```

好的列表字段应该满足：

- 列表里的元素类型一致。
- 字段描述说明应该放什么。
- 不把多个维度混到一个列表里。

不推荐：

```python
items: list[str] = Field(description="候选人的所有信息")
```

更推荐：

```python
core_skills: list[str] = Field(description="候选人的核心技能列表")
project_names: list[str] = Field(description="候选人参与过的项目名称列表")
certifications: list[str] = Field(description="候选人证书列表")
```

如果列表里的每一项都需要多个字段，例如项目需要名称、职责、技术栈，可以后续再设计嵌套对象。本课先优先练习简单稳定的列表字段。

---

## 十一、避免过度复杂的 schema

很多结构化输出不稳定，并不是因为模型差，而是 schema 一开始设计得太复杂。

例如你想一次性抽取：

```text
候选人基础信息
教育经历
每段工作经历
每个项目经历
技能熟练度
岗位匹配分析
薪资建议
面试问题
风险点评
```

如果全部塞进一个深层嵌套 schema，模型更容易漏字段、混字段、输出不一致。

更好的方式是分阶段：

```text
第一步：抽取基础信息和核心技能。
第二步：抽取项目经历。
第三步：做岗位匹配评分。
```

schema 设计有一个很实用的原则：

> 先让结果稳定，再让结构变丰富。

对初学阶段来说，建议优先使用：

- 8 个以内的顶层字段。
- 1 层列表。
- 少量 `Literal` 候选值。
- 清晰的 `None` 规则。
- 简短但具体的字段描述。

---

## 十二、示例 3：本地校验 schema 设计

Pydantic 不只服务于模型输出，也可以在本地校验普通 Python 数据。

新建文件：

```text
examples/lesson_10_local_validation.py
```

代码：

```python
from typing import Literal

from pydantic import BaseModel, Field, ValidationError


class CandidateScore(BaseModel):
    candidate_name: str = Field(description="候选人姓名")
    match_level: Literal["high", "medium", "low"] = Field(description="匹配等级")
    reasons: list[str] = Field(description="匹配原因列表")


valid_data = {
    "candidate_name": "李明",
    "match_level": "high",
    "reasons": ["5 年 Python 经验", "熟悉 FastAPI"],
}

invalid_data = {
    "candidate_name": "王芳",
    "match_level": "very_high",
    "reasons": "经验丰富",
}

print(CandidateScore.model_validate(valid_data))

try:
    CandidateScore.model_validate(invalid_data)
except ValidationError as error:
    print("\n校验失败：")
    print(error)
```

运行：

```powershell
uv run examples/lesson_10_local_validation.py
```

这里出现两个新写法：

`model_validate()` 是 Pydantic 模型的方法，用来把普通字典校验并转换成模型对象。

`ValidationError` 是 Pydantic 校验失败时抛出的异常。上面的 `invalid_data` 有两个问题：

- `match_level` 不在 `Literal["high", "medium", "low"]` 允许的范围内。
- `reasons` 期望是 `list[str]`，但传入的是字符串。

这个例子不调用大模型，但它能帮助你理解：schema 不是装饰品，它真的会检查数据形态。

---

## 十三、字段设计清单

写 Pydantic schema 时，可以按这个清单检查：

```text
字段名是否表达业务含义？
字段描述是否说明了取值规则？
缺失时是否允许 None？
分类字段是否适合用 Literal？
列表字段是否只放同一种东西？
字段数量是否过多？
是否把后续程序不会用到的字段也放进来了？
是否让模型被迫推断它无法知道的信息？
```

如果一个字段只是“以后可能有用”，先不要急着加。结构化输出的第一目标是稳定和可消费，不是一次性收集所有信息。

---

## 十四、常见错误与排查

### 1. 字段名太抽象

`result`、`info`、`data` 这类字段名通常太宽泛。

改成 `candidate_name`、`core_skills`、`match_reason` 这样的业务字段，模型和程序都会更容易理解。

### 2. 字段描述只重复字段名

不推荐：

```python
match_level: str = Field(description="匹配等级")
```

更推荐：

```python
match_level: Literal["high", "medium", "low", "unknown"] = Field(
    description="与 Python 后端开发岗位的匹配程度；无法判断时选择 unknown"
)
```

描述要补充规则，而不是重复名字。

### 3. 忘记给缺失信息留 `None`

如果输入文本没有某个信息，模型需要一个合法出口。

写成：

```python
phone: str | None = Field(default=None, description="手机号；没有出现则为 None")
```

比强制 `phone: str` 更稳。

### 4. 枚举候选值太多

`Literal` 很适合少量固定选项。如果候选值几十个，模型可能仍然容易选错。

候选值太多时，可以考虑：

- 先做粗分类。
- 后续再细分。
- 或让模型输出自由文本，再由程序做映射。

### 5. 嵌套太深

深层嵌套 schema 看起来很完整，但更难稳定生成。

可以先拆成多步链路：基础信息、项目经历、匹配分析分别处理。

### 6. 把推断和抽取混在一起

“候选人姓名”是抽取；“是否适合岗位”是判断。

两者可以放在同一个 schema 中，但字段描述要说清楚哪些字段必须来自原文，哪些字段允许基于信息进行判断。

---

## 十五、本课小结

这一课你把结构化输出推进到了 schema 设计层面。

你需要记住：

- schema 质量会直接影响结构化输出稳定性。
- 字段名要表达业务含义。
- `Field(description=...)` 要说明取值规则，而不只是重复字段名。
- `Literal[...]` 适合有限候选值。
- `类型 | None = Field(default=None, ...)` 适合可能缺失的信息。
- `list[str]` 适合同类多项信息。
- schema 不要一开始就设计得太复杂。
- 可以用 `model_json_schema()` 检查 schema 结构，用 `model_validate()` 理解 Pydantic 校验。

请把 Pydantic schema 当成“模型和程序之间的接口设计”，而不是简单的字段列表。

---

## 十六、课后练习

### 练习 1：设计简历基础信息 schema

创建：

```text
examples/lesson_10_resume_basic_schema.py
```

要求定义 `ResumeBasicInfo`：

- `candidate_name`
- `phone`
- `email`
- `highest_education`
- `core_skills`

要求：

- 可能缺失的字段使用 `类型 | None = Field(default=None, ...)`。
- 学历字段使用 `Literal`。
- 技能字段使用 `list[str]`。
- 打印 `ResumeBasicInfo.model_json_schema()`。

### 练习 2：改进字段描述

找出下面 schema 的问题，并重写：

```python
class BadCandidate(BaseModel):
    name: str = Field(description="名字")
    level: str = Field(description="等级")
    items: list[str] = Field(description="东西")
```

要求：

- 字段名更具体。
- 描述说明取值规则。
- 分类字段使用 `Literal`。

### 练习 3：岗位匹配 schema

设计 `JobMatchResult`：

- `match_level`
- `match_score`
- `matched_requirements`
- `missing_requirements`
- `reason`

思考：

- 哪些字段应该是列表？
- 哪些字段应该用 `Literal`？
- `match_score` 是整数、浮点数，还是文本？为什么？

### 练习 4：本地校验错误

基于本课 `CandidateScore` 示例，故意传入错误数据：

- 错误的 `match_level`。
- 错误的 `reasons` 类型。
- 缺失 `candidate_name`。

观察 Pydantic 的报错信息。

### 练习 5：控制 schema 复杂度

设计一个“项目经历抽取”schema。

先写一个过度复杂版本，再删减到 6 个以内字段。

回答：

```text
哪些字段是后续程序真的需要的？
哪些字段只是看起来有用？
哪些字段会迫使模型过度推断？
```

---

## 十七、下一课预告

下一课进入：

```text
第 11 课：信息抽取器实战
```

你会把本课的 schema 设计能力用于更完整的信息抽取任务：从自然语言中抽取固定字段，处理列表、日期、联系方式等常见字段，并把抽取链封装成可复用能力。

---

## 十八、官方参考

- LangChain Structured output: `https://docs.langchain.com/oss/python/langchain/structured-output`
- LangChain OpenAI integration - Structured output: `https://docs.langchain.com/oss/python/integrations/chat/openai`
- Pydantic Fields: `https://docs.pydantic.dev/latest/concepts/fields/`
- Pydantic BaseModel API: `https://docs.pydantic.dev/latest/api/base_model/`
- Python `typing.Literal`: `https://docs.python.org/3/library/typing.html#typing.Literal`
