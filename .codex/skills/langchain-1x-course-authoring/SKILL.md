---
name: langchain-1x-course-authoring
description: Use this skill whenever creating, revising, reviewing, or extending this project's LangChain 1.x self-study textbook, including lesson markdown files, course outlines, runnable examples, exercises, project practice, SDK usage, uv commands, and official-documentation checks. This skill is required for every course chapter or lesson in this repository, even if the user only says "continue the next lesson" or asks for a small correction to course content.
---

# LangChain 1.x Course Authoring

Use this skill for all work on this repository's LangChain 1.x learning materials. The goal is to produce a rigorous self-study textbook that takes learners from zero to project practice with current LangChain 1.x APIs.

## Core Standard

Treat the materials as a textbook, not a loose note collection.

Every lesson must be:

- Accurate against current official documentation.
- Written directly for self-study learners, not as a teacher's lecture script or instructor notes.
- Progressive from concept to code to practice.
- Consistent with the course outline and previous lessons.
- Free of outdated LangChain patterns unless explicitly marked as historical or migration context.
- Runnable with the project's uv-based workflow.

## Required Research Step

Before creating or materially revising any lesson:

1. Check the relevant official documentation.
2. Prefer official LangChain, LangGraph, LangSmith, OpenAI, provider, or uv documentation over blogs and old tutorials.
3. If browsing is available, browse official docs for current API details.
4. If browsing is not available, state that the lesson should be rechecked against official docs before publication.
5. Cite official reference links in the lesson's "官方参考" section when useful.

Official sources to prefer:

- LangChain Python docs: `https://docs.langchain.com/oss/python/langchain/overview`
- LangChain v1 release notes: `https://docs.langchain.com/oss/python/releases/langchain-v1`
- LangChain messages docs: `https://docs.langchain.com/oss/python/langchain/messages`
- LangChain integrations: `https://docs.langchain.com/oss/python/integrations/`
- LangGraph docs: `https://docs.langchain.com/oss/python/langgraph/overview`
- LangGraph v1 release notes: `https://docs.langchain.com/oss/python/releases/langgraph-v1`
- LangSmith docs: `https://docs.smith.langchain.com/`
- uv docs: `https://docs.astral.sh/uv/`

## SDK And Code Rules

Follow current LangChain 1.x patterns.

- Use `uv` for environment and execution.
- Use `uv add ...` for dependencies.
- Use `uv run examples/file_name.py` for running lesson examples.
- Avoid `pip install ...` in lessons unless explaining migration or alternatives.
- Avoid `python examples/file_name.py` as the primary run command.
- Do not hard-code one fixed model name inside example scripts.
- Read model names from `.env` with `OPENAI_MODEL`.
- Use a reasonable default such as `getenv("OPENAI_MODEL", "gpt-5-nano")`, while explaining learners can replace it with their provider's supported model.
- Use `.env` for API keys and endpoint configuration.
- Never put real API keys in examples.

Preferred model setup pattern:

```python
from os import getenv

from dotenv import load_dotenv
from langchain_openai import ChatOpenAI


load_dotenv()


model = ChatOpenAI(
    model=getenv("OPENAI_MODEL", "gpt-5-nano"),
    temperature=0.2,
)
```

Preferred `.env` pattern:

```text
OPENAI_API_KEY=你的_api_key
OPENAI_MODEL=gpt-5-nano
```

For OpenAI-compatible endpoints:

```text
OPENAI_API_KEY=你的_api_key
OPENAI_BASE_URL=https://你的兼容接口地址/v1
OPENAI_MODEL=你的模型名
```

## Import Rules

Use current public LangChain 1.x imports when the official docs expose them.

- Prefer `from langchain.messages import HumanMessage, SystemMessage, AIMessage` for message classes in beginner lessons.
- Use `from langchain_openai import ChatOpenAI` for OpenAI chat model integration.
- Use `from langchain_core.prompts import PromptTemplate, ChatPromptTemplate, MessagesPlaceholder` for prompt templates unless current official docs recommend a newer public entry point.
- Avoid old imports or deprecated classes from pre-1.0 tutorials.

If a lower-level `langchain_core` import is used, explain why it is appropriate.

## Lesson Structure

A lesson markdown file should usually contain:

1. Title and keywords.
2. Lesson goals.
3. Concept explanation.
4. Why the concept matters in real applications.
5. Minimal runnable example.
6. Code explanation.
7. Slightly richer practice case.
8. Common mistakes and troubleshooting.
9. Lesson summary.
10. Homework exercises.
11. Next lesson preview.
12. Official references.

Use this structure flexibly, but do not skip runnable examples for coding lessons.

Write each section so a learner can read it alone and know what to do next. Avoid relying on a teacher being present to explain missing steps.

## Teaching Style

Write as a self-study textbook author speaking directly to the learner.

- Explain from first principles before naming abstractions.
- Introduce important new concepts by helping learners understand the design intent: why this abstraction exists, what complexity it removes, what problem it solves for application developers, and why the library designers likely shaped it this way.
- Use problem-driven explanation when it helps, but do not force every concept into a rigid template. Choose the explanation path that best reveals the concept's purpose.
- When teaching dynamic APIs such as prompt variables, parsers, tools, retrievers, or graph state, explain what stays fixed, what changes at runtime, and why that separation makes the system easier to maintain.
- Define terms when they first appear.
- Connect each lesson to the course roadmap.
- Tell learners what to observe after running code.
- Include "why" and "when to use it", not only "how".
- Keep examples small enough for beginners but realistic enough to transfer to projects.
- Use Chinese as the main teaching language.
- Keep technical identifiers, package names, and code in English.
- Prefer reader-facing wording such as "你将看到", "请运行", "观察输出", "思考一下", "你现在可以".
- Avoid teacher-facing wording such as "本课讲授", "课堂上可以", "教师应", "引导学生", "讲义", or "授课时".
- Use "示例", "实践案例", "练习" instead of "课堂案例" when creating new lessons, unless explicitly discussing a live class context.

For each major concept, try to answer several of these questions naturally:

```text
为什么会有这个东西？
如果没有它，开发者会遇到什么麻烦？
它把哪部分复杂度封装起来了？
它让代码在可读性、复用性、可维护性或可组合性上变好了什么？
它适合解决什么问题，不适合解决什么问题？
它的核心输入、输出、变化点是什么？
怎样用一个最小例子验证它的价值？
```

## Accuracy Guardrails

Before finalizing a lesson, check:

- Does every command use the current project convention?
- Are all code examples internally consistent?
- Are imports current and official-doc-aligned?
- Is the model name configurable through `.env`?
- Are outdated APIs avoided?
- Are concepts sequenced from simple to complex?
- Does the lesson include practical exercises?
- Does the lesson include official references?
- Does it avoid claiming API behavior that may have changed unless verified?

If unsure about a current API, verify with official docs before writing the final content.

## Runnable Example Conventions

Use an `examples/lesson_nn_name.py` naming convention in lessons.

Examples:

```text
examples/lesson_01_hello_langchain.py
examples/lesson_02_chat_prompt_template.py
examples/lesson_03_output_parser.py
```

Run commands must be:

```powershell
uv run examples/lesson_nn_name.py
```

When a script only demonstrates template rendering and does not call a model, say so explicitly.

## Content Progression Rules

Preserve the course progression:

1. Model calls and environment setup.
2. Messages and prompt templates.
3. Model outputs and parsing.
4. Runnable and LCEL composition.
5. Structured output.
6. RAG.
7. Tools.
8. Agents with current LangChain 1.x APIs.
9. LangGraph workflows.
10. Debugging, tracing, evaluation, and engineering.
11. Integrated projects.
12. Deep Agents expansion.

Avoid teaching Agent or LangGraph details too early unless used only as context.

## Review Checklist For Existing Lessons

When the user asks to review or revise a lesson, inspect for:

- Old SDK usage.
- Hard-coded model names.
- Non-uv commands.
- Unclear self-study instructions.
- Missing observations after code execution.
- Missing troubleshooting.
- Missing exercises.
- Missing official references.
- Mismatch with the master course outline.

Prioritize correctness and learner reproducibility over making the prose sound impressive.
