将“普通用户 prompt”转化为更利于 LLM 理解和执行的“结构化提示”的一个可复用模板。它兼具元提示、上下文限定、输出格式提示，使用 Markdown 或 XML 方式展现。

---

## 🧠 方法一：Markdown + CO‑STAR 框架（兼具结构与清晰）

```markdown
# SYSTEM (系统提示)
你是一个擅长格式化回复并按要求输出结构化结果的智能助理。

# CONTEXT（上下文／目标）
- :contentReference[oaicite:1]{index=1}
- :contentReference[oaicite:2]{index=2}
- :contentReference[oaicite:3]{index=3}

# OBJECTIVE（任务目标）
:contentReference[oaicite:4]{index=4}

# STYLE（风格）
:contentReference[oaicite:5]{index=5}

# TONE（语气）
专业中带有教学风格，适合提示工程师阅读。

# AUDIENCE（受众）
:contentReference[oaicite:6]{index=6}

# RESPONSE（输出格式）
请输出两份示例：  
1. Markdown 结构版本；  
2. XML 结构版本；  
模板中应包含注释/占位字段。

# USER_PROMPT（待转化内容）
:contentReference[oaicite:7]{index=7}
```

备注说明：

* 前半部分用于引导 LLM 明确角色与预期；
* 后半部分包含用户要转化的内容；
* 整体让输出更具可控性、可复用性。

---

## ✅ 方法二：XML Frame + 元提示控制流程

```xml
<prompt>
  <system>
    你是提示工程专家，专职把用户 prompt 改写为带元提示、格式化清晰的模板。
  </system>

  <meta>
    <role>assistant</role>
    <instructions>
      包含角色设定、steps、示例、输出格式
    </instructions>
    <format>Markdown 或 XML</format>
  </meta>

  <user_original>
    <!-- 在这里插入用户原始 prompt -->
  </user_original>

  <template_examples>
    <markdown_example>
      <![CDATA[
      ### 🔧 模板（Markdown）

      **SYSTEM**: 你是…

      **USER_INPUT**:
      “原 prompt 推荐放这里”

      **TASK**:
      - 步骤 1：解析用户意图
      - 步骤 2：构造元提示
      - 步骤 3：输出结构化模板

      **OUTPUT_FORMAT**:
      - 使用 Markdown/ YAML / JSON
      ]]>
    </markdown_example>

    <xml_example>
      <![CDATA[
      <template>
        <system>…</system>
        <user>用户 prompt…</user>
        <steps>
          <step>解析意图</step>
          <step>构造元提示</step>
          <step>格式化输出</step>
        </steps>
        <output format="markdown_or_xml">…</output>
      </template>
    ]]>
    </xml_example>
  </template_examples>
</prompt>
```

---

### 🔍 为什么这样做效果更好？

1. **结构清晰**：CO‑STAR 框架确保每个关键信息都明确传达 ([panscook.com][1], [reddit.com][2], [wqw547243068.github.io][3])
2. **元提示增强**：添加 meta 指令让 LLM 自身充当“提示专家”，提升输出质量&#x20;
3. **格式化方式灵活**：Markdown vs XML 可选，依据任务性质选择&#x20;

---

## 🎯 实战演示

假设用户原文：

> “帮我写一个广告文案，宣传我们的新款智能手表，要吸引年轻人，语气幽默，附带产品特点列表。”

**1. Markdown 示例**

```markdown
# SYSTEM
你是广告文案专家，专注目标受众，输出风格化内容。

# USER_PROMPT
“帮我写一个广告文案，宣传我们的新款智能手表，要吸引年轻人，语气幽默，附带产品特点列表。”

# TASK
1. 理解目标：吸引年轻用户  
2. 风格：幽默俏皮  
3. 内容：  
   - 核心卖点  
   - 产品功能列表  
4. 格式：  
   - 标题  
   - 正文  
   - 产品特点 Bullet List  

# OUTPUT
请输出最终广告文案，使用中文，符合上述结构。
```

**2. XML 示例**

```xml
<template>
  <system>你是广告文案专家。</system>
  <user_prompt>帮我写一个广告文案，宣传我们的新款智能手表，要吸引年轻人，语气幽默，附带产品特点列表。</user_prompt>
  <task>
    <objective>写广告文案吸引年轻人</objective>
    <tone>幽默</tone>
    <structure>
      <title/>
      <body/>
      <features type="list"/>
    </structure>
  </task>
  <output_format>Markdown</output_format>
</template>
```

---

### ✅ 总结

* **模板化 + 元提示** 能显著提升 LLM 输出的质量与一致性。
* **结构清晰**（Markdown/XML/CO‑STAR 等）更易于扩展、维护与复用。
* **附模板示例** 帮助用户快速应用。

---

如需进一步个性化，如加入 chain‑of‑thought、few‑shot 示例、工具调用、JSON 输出等，可按此结构进一步扩展。我非常乐意帮你为任何 prompt 生成这种 LLM 🛠️ 风格的结构化模板。

[1]: https://www.panscook.com/2025/05/%E3%80%8Aprompt-engineering-for-llms%EF%BC%9A%E5%AE%9E%E7%94%A8%E6%8C%87%E5%8D%97%E3%80%8B/?utm_source=chatgpt.com "《Prompt Engineering for LLMs：实用指南》 – PAN用书"
[2]: https://www.reddit.com/r/LocalLLaMA/comments/1i2b2eo?utm_source=chatgpt.com "Meta Prompts - Because Your LLM Can Do Better Than Hello World"
[3]: https://wqw547243068.github.io/pe?utm_source=chatgpt.com "提示工程指南及技巧"

下面是一个强化后的复用模板，集成了 **Chain-of-Thought 推理**、**Few-Shot 示例**、**工具调用**、**JSON 输出**，采用 Markdown 和 XML 两种结构展示，可直接用于提示工程场景。

---

## 🛠 Markdown 结构版

````markdown
# SYSTEM
:contentReference[oaicite:1]{index=1}

# METAPROMPT（元提示）
- :contentReference[oaicite:2]{index=2}
- 🧩 任务分解：
  1. :contentReference[oaicite:3]{index=3}
  2. :contentReference[oaicite:4]{index=4}
  3. :contentReference[oaicite:5]{index=5}
  4. :contentReference[oaicite:6]{index=6}
- :contentReference[oaicite:7]{index=7}

# FEW‑SHOT 示例
### Example 1
**USER**: “…任务描述 1…”
**ASSISTANT**:
```text
Let's think step‑by‑step:
1. 分析…
2. 推理…
3. 得出结论。
Result: …最终结果1…
````

### Example 2

**USER**: “…任务描述 2…”
**ASSISTANT**:

```text
Let's think step‑by‑step:
1. 第一步…
2. 第二步…
3. 结论…
Result: …最终结果2…
```

# PROMPT TEMPLATE

````text
SYSTEM: 你是……
{Few‑Shot 示例}

USER: {用户原始 prompt 内容}

ASSISTANT:
Let's think step‑by‑step:
1. …
2. …
3. …

CALL_TOOL("tool_name", {{ "param1": "{value1}", ... }})

输出 JSON：
```json
{
  "analysis_steps": ["…", "..."],
  "tool_response": { /* tool 调用返回 */ },
  "final_answer": "…"
}
````

````

---

## 🧩 XML 结构版

```xml
<prompt>
  <system>你是提示工程专家，负责生成带 CoT、Few‑Shot、工具调用和 JSON 输出的结构化提示。</system>

  <meta>
    <role>assistant</role>
    <features>
      <CoT>Let’s think step-by-step</CoT>
      <FewShot examples="2+"/>
      <ToolCall name="tool_name" params="json"/>
      <OutputFormat>JSON</OutputFormat>
    </features>
  </meta>

  <examples>
    <example id="1">
      <user>…任务描述 1…</user>
      <assistant>
        <![CDATA[
        Let's think step‑by‑step:
        1. 分析…
        2. 推理…
        3. 结论：…
        Result: …最终结果1…
        ]]>
      </assistant>
    </example>
    <example id="2">
      <user>…任务描述 2…</user>
      <assistant>
        <![CDATA[
        Let's think step‑by‑step:
        1. ...
        2. ...
        3. 结论…
        Result: …最终结果2…
        ]]>
      </assistant>
    </example>
  </examples>

  <template>
    <user_original>{这里插入用户原始 prompt}</user_original>
    <assistant_template>
      <![CDATA[
      Let's think step‑by‑step:
      1. …
      2. …
      3. …

      CALL_TOOL("tool_name", { "param1": "{value1}", ... })

      输出 JSON：
      {
        "analysis_steps": [ "…", "…" ],
        "tool_response": /* tool 返回 */,
        "final_answer": "…"
      }
      ]]>
    </assistant_template>
  </template>
</prompt>
````

---

## ✅ 为什么这样设计更优

1. **Chain‑of‑Thought**：通过 “Let’s think step‑by‑step” 引导模型进行中间推理，可提升复杂任务准确性 ([promptingguide.ai][1], [prompthub.us][2])。
   &#x20;。
2. **工具调用**：显式 `CALL_TOOL(...)` 可与函数/环境集成，实现动态信息获取。
3. **JSON 输出**：让 LLM 严格按照结构化数据返回，更易于下游程序解析。
4. 完整融合 **角色设定 → 示例演示 → 模板设计 → 输出规范**，大幅提升提示工程质量。

---

## 🧩 应用示例（假设用户 prompt）

* **用户原始 prompt**:
  “帮我分析下面新闻标题是否有倾向性，给出理由，并查一下当天股市开盘价。”

* **生成提示**:

  ```text
  SYSTEM: 你是提示工程专家，…
  (插入 Few‑Shot 示例)

  USER: 帮我分析下面新闻标题是否有倾向性，给出理由，并查一下当天股市开盘价。

  ASSISTANT:
  Let's think step‑by‑step:
  1. 判断语气/措辞中的倾向性。
  2. …

  CALL_TOOL("get_stock_open", { "ticker": "AAPL", "date": "2025‑06‑11" })

  输出 JSON：
  {
    "analysis_steps": [ ... ],
    "stock_open_price": …,
    "bias_conclusion": "…"
  }
  ```

你可以基于此模板灵活调整 Few‑Shot 数量、CoT 深度、工具名称和 JSON schema，覆盖多种复杂任务。

[1]: https://www.promptingguide.ai/techniques/cot.en?utm_source=chatgpt.com "Chain-of-Thought Prompting | Prompt Engineering Guide<!-- -->"
[2]: https://www.prompthub.us/blog/chain-of-thought-prompting-guide?utm_source=chatgpt.com "Chain of Thought Prompting Guide"

