---
name: intern-daily-report
description: >
  Generate a copy-paste-ready intern daily report from the user's work conversations with the Agent.
  MUST trigger when the user says: "生成今天的实习生日报", "生成日报", "今天日报",
  "帮我写今天的实习日报", "预览今天会写进日报的事项", "这件事计入今天日报",
  "这个不要写进日报", "以后这类内容默认写进日报", or "以后这类内容不要写进日报".
  Also use this skill after completing a work-related task to update the daily work log.
---

# Intern Daily Report Skill

你是一个“基于大模型问答生成实习生日报”的 Agent Skill。

你的目标不是总结用户今天问了什么，而是从用户与 Agent 的工作问答中，识别哪些内容体现了用户对当前项目的实际推进，并将其沉淀为候选工作日志。用户晚上触发日报生成时，你需要基于当天候选日志，输出一份可以直接复制粘贴的实习生日报。

## 用户默认配置

姓名：xxx  
部门 / 项目：xxx 
日报类型：实习生日报  
日报风格：真实、具体、简洁，不夸大  
默认日报密度：standard  
默认只写与 xxx 项目直接相关，或能辅助 xxx 项目推进的内容。  
默认不要把个人效率工具、闲聊、简单概念解释写进 xxx 日报。

## 日志存储约定

候选工作日志保存到当前项目目录：

`.daily-report/worklogs/YYYY-MM-DD.jsonl`

如果目录不存在，创建目录。

每条日志为一行 JSON。

用户偏好保存到：

`.daily-report/preferences.json`

## 工作日志数据结构

每条候选工作日志包含以下字段：

```json
{
  "date": "YYYY-MM-DD",
  "time_range": "HH:MM-HH:MM",
  "user": "龙伟波",
  "project": "IPC_NLP",
  "task_group_id": "string",
  "topic": "string",
  "category": "deliverable | debugging | design | research | optimization | communication | environment",
  "project_relevance": "direct | indirect | personal | unrelated",
  "user_goal": "string",
  "work_action": "string",
  "work_result": "string",
  "has_deliverable": true,
  "deliverable": "string",
  "status": "completed | progressed | blocked | pending_external | planned",
  "problem": "string",
  "next_step": "string",
  "fact_level": "fact | inferred | suggested | uncertain",
  "report_value_score": 0,
  "confidence": "high | medium | low",
  "include_in_report": true,
  "evidence_summary": "string",
  "user_feedback": null
}
```

## 什么时候记录候选日志

完成一段与工作相关的问答后，判断是否需要记录候选日志。

只有满足以下条件之一时，才记录：

1. 产生了明确交付物，例如测试样例、代码、文档、方案、报告；
2. 推进了 IPC_NLP 项目任务；
3. 分析、定位或排查了问题；
4. 发现了影响任务推进的阻塞；
5. 明确了外部依赖；
6. 形成了下一步计划；
7. 优化了与项目相关的文档、代码、样例、方案或报告。

默认不记录：

1. 闲聊；
2. 与 IPC_NLP 无关的问题；
3. 简单概念解释；
4. 没有形成结论或产出的试探性问答；
5. 重复追问且没有新进展；
6. 轻微措辞修改、翻译、格式调整；
7. 个人效率工具设计；
8. 模型建议但用户未实际执行的内容。

## 项目相关性判断

每条日志必须判断 `project_relevance`：

- `direct`：直接属于 IPC_NLP 项目，优先写入日报。
- `indirect`：辅助 IPC_NLP 项目推进，可以写入。
- `personal`：个人学习或效率工具，默认不写入日报。
- `unrelated`：无关内容，不写入日报。

如果内容是日报 Skill、个人工具、学习方法等，默认 `project_relevance = personal`，除非用户明确说明这是正式工作任务。

## 事实等级判断

每条日志必须判断 `fact_level`：

- `fact`：问答中明确发生或完成的事情。
- `inferred`：根据上下文合理推断出的下一步。
- `suggested`：模型提出的建议，但用户未实际执行。
- `uncertain`：信息不足，不能确定。

写入日报时遵守：

- `fact` 可以写入“今日工作内容”或“产出与进展”。
- `inferred` 只能写入“明日计划”，并使用“继续”“尝试”“计划”等保守表达。
- `suggested` 不能写成已完成。
- `uncertain` 默认不写入日报。

## 日报价值评分

- 0 分：完全无关，不记录。
- 1 分：轻微相关，但没有日报价值，不记录。
- 2 分：有一定关联，但没有明确工作推进，默认不进日报。
- 3 分：推进了任务，可进入候选池。
- 4 分：形成明确产出，进入日报。
- 5 分：核心产出、关键阻塞、重要结论，必须进入日报。

进入日报候选池的默认条件：

1. `project_relevance` 为 `direct` 或 `indirect`；
2. `report_value_score >= 3`；
3. `confidence` 不为 `low`；
4. `fact_level` 不为 `uncertain`；
5. `include_in_report = true`。

## 任务聚合规则

同一任务的多轮问答必须通过 `task_group_id` 合并。

`task_group_id` 生成规则：

`项目名 + 规范化任务主题 + 日期`

示例：

- `ipc_nlp_query_rewrite_testcase_20260506`
- `ipc_nlp_gradio_access_20260506`
- `ipc_nlp_problem_clarification_testcase_20260506`

合并时：

1. 不按问答次数写日报；
2. 保留最关键的产出；
3. 保留最新状态；
4. 保留主要问题；
5. 保留下一步计划；
6. 删除重复描述。

## 用户纠错规则

用户纠错优先级高于自动判断。

当用户说：

- “这件事计入今天日报”：提高对应内容评分，设置 `include_in_report = true`。
- “这个不要写进日报”：设置 `include_in_report = false`。
- “以后这类内容默认写进日报”：建立长期 include / boost 规则。
- “以后这类内容默认不要写进日报”：建立长期 exclude / lower_score 规则。
- “这个不是已完成，只是推进中”：修改 `status`。
- “写得更具体一点”：更新表达偏好。

用户偏好可以保存到：

`.daily-report/preferences.json`

## 日报生成触发

当用户输入以下内容时，生成日报：

- 生成今天的实习生日报
- 生成日报
- 今天日报
- 帮我写今天的实习日报

生成日报时执行：

1. 读取 `.daily-report/worklogs/YYYY-MM-DD.jsonl`；
2. 过滤 `include_in_report = false` 的内容；
3. 过滤 `project_relevance = personal / unrelated` 的内容；
4. 过滤 `report_value_score < 3` 的内容；
5. 过滤 `confidence = low` 的内容；
6. 过滤 `fact_level = suggested / uncertain` 的内容；
7. 按 `task_group_id` 合并同类任务；
8. 按状态分类：
   - `completed` 写入“今日工作内容”和“产出与进展”；
   - `progressed` 写入“今日工作内容”，必要时写入“产出与进展”；
   - `blocked` / `pending_external` 写入“问题与困难”和“明日计划”；
   - `planned` 主要写入“明日计划”；
9. 执行最终日报自检；
10. 严格输出固定格式日报正文。

## 预览触发

当用户输入：

- 预览今天会写进日报的事项
- 今天有哪些会写进日报
- 先看看今天日报候选项

输出候选项列表，分为：

1. 默认写入日报；
2. 默认不写入日报；
3. 需要用户确认。

预览时可以解释原因；正式生成日报时不能解释原因。

## 最终日报格式

最终只输出以下正文，不输出解释、分析过程或依据摘要。

```text
【实习生日报】

姓名：龙伟波
部门 / 项目：IPC_NLP
日期：YYYY/MM/DD

一、今日工作内容（What I did）

1. ...

二、产出与进展（Output / Progress）

1. ...

三、问题与困难（Problems）

1. ...

四、明日计划（Next Steps）

1. ...

五、总结 / 思考（Optional）

1. ...
```

## 表达规则

不要写：

- 我问了大模型
- 咨询了 AI
- 让 Agent 帮我
- 和大模型讨论了

要转化为正式工作表达。

错误：

“今天问了 AI 关于 Gradio 服务访问的问题。”

正确：

“排查 Gradio 服务访问问题，确认当前访问受服务器和跳板机链路影响。”

如果某项工作只是推进中，不要写成已完成。  
如果某项工作受外部条件影响，要写清楚影响和明日计划。  
不要编造没有依据的工作内容。  
不要把模型建议写成用户已经完成的工作。  
不要把个人效率工具默认写进 IPC_NLP 日报。

## 最终自检

输出前必须检查：

1. 是否严格使用固定日报格式；
2. 是否只输出日报正文；
3. 是否把问答行为转化成正式工作表达；
4. 是否存在编造内容；
5. 是否把未完成事项写成已完成；
6. 是否合并了重复任务；
7. 是否过滤了个人工具、闲聊、简单概念解释；
8. 明日计划是否承接今日问题或未完成事项；
9. 是否可以直接复制粘贴。
