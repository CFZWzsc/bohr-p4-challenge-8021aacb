---
name: manuscript-structure-diagnoser
description: Diagnose a full manuscript at section, paragraph and sentence level for missing sections, heading hierarchy, cross-section duplication, claims without evidence and conclusions that exceed the reported results, returning locators plus executable edit actions. Use when a task asks for structural and logical revision rather than language polishing.
---

# 论文结构与逻辑诊断

## 用途与适用条件

对一篇完整稿件做**结构化诊断**：定位到章节/段落/句子，指出属于哪一类结构问题，并给出作者可直接执行的编辑动作（移到哪、与哪段合并、哪句需要收窄）。

适用：

- 任务是"结构/逻辑/论证链优化"，需要诊断报告而不是重写稿。
- 需要覆盖研究问题、章节结构、论证链、段落关系、重复内容、结论边界六类。
- 需要每条判断可指回原文位置，便于作者逐条采纳/驳回。

不适用：纯语言润色、纯格式排版、需要评价学科内容正确性。

## 输入

| 输入 | 位置 | 说明 |
|-|-|-|
| `case.json` | `/input/case.json`（`--case` 可覆盖） | 稿件取自 `manuscript` / `draft` / `document` / `text` / `content` / `article`；支持字符串或段落列表 |
| `task.md` | `/input/task.md`（自动探测） | 任务约束（目标体裁、必须章节） |
| 输出 | `/app/output/result.json`（`--out` 可覆盖） | 结构化结果 |

## 处理流程（对应评分维度）

1. **结构解析**（诊得准）：识别 Markdown 与 `2.1` 编号标题，建立 `S{n}:P{m}` 定位；重复同章子标题、编号子节归入当前章节，空章节不单独计数。
2. **章节覆盖**：对照 Abstract/Introduction/Related Work/Methods/Results/Discussion/Conclusion 检查缺章（有 Introduction 时 Related Work 可缺省）与顺序偏离。
3. **研究问题**：检测 `we aim/investigate`、`本研究旨在`、问句等线索；缺失即高优先级。
4. **段落与重复**（段落关系）：>260 词的长段提示拆分；跨章节句子做词项 Jaccard ≥0.7 的重复检测，给出左右定位与相似度。
5. **论证链**：引言/讨论中的结论性断言若无 `[n]` 或 `\cite`，标 `claim_without_evidence`；结果段落无任何数值标 `results_without_numbers`。
6. **结论边界**：讨论/结论中因果动词无限定语、或 all/every/always 类全称表述，标 `conclusion_boundary` 并建议收窄。
7. **动作排序**：按 high → medium 排序，重编 `F01…`，同步 evidence；`action_plan` 给出执行顺序。

## 输出契约

- `case_id`：原样保留。
- `status`：`success`（有稿件）/ `insufficient_input`（无正文）/ `failed`。
- `result`：`structure`（章节、定位、段数、词数、占比）、`findings`（类别/严重度/定位/细节/动作/引文）、`action_plan`、`duplicates`、`research_question_detected`、`limitations`。
- 顶层另置 `structure`、`findings`、`action_plan` 便于契约映射。
- `evidence`（诊断→原文定位）、`uncertainty`、`errors`、`stats`（章节/段落/词数/问题/高严重度计数）、`contract_notes`。

**output_contract 处理**：声明字段逐一置于顶层；`findings`、`structure`、`action_plan` 及别名（`issues`、`actions` 等）会映射；无法产出字段置 `null` 并记录，不虚构缺陷。

## 异常处理

| 情况 | 处置 | status |
|-|-|-|
| `case.json` 缺失/非法 | 结构化失败，标注阶段，退出码 2 | `failed` |
| 无正文 | 不产出诊断，说明缺什么 | `insufficient_input` |
| 缺章节/无参考文献 | 记为缺章或提示，不推断内容 | `success` |
| 无 Methods | 额外提示无法核对结果与方法对应 | `success` |
| 编号子节/重复子标题 | 归入当前章节，不制造空章节 | `success` |
| 重复检测无命中 | 在 `uncertainty` 说明同义改写可能漏检 | `success` |

## 限制

- 只诊断文本结构，不判断学科结论是否正确、不核对图表数值。
- 重复检测基于词项 Jaccard，同义改写、翻译式重复可能漏检。
- 段落切分依赖空行；无空行的长文会被视为一段并触发长段提示。
- 不自动改稿，交付物是诊断 + 动作清单，作者仍需自行编辑。

## 最小调用示例（示例，非官方案例结果）

```json
{
  "case_id": "demo-001",
  "output_contract": {"findings": "array", "action_plan": "array", "status": "string"},
  "manuscript": "# Abstract\n\nWe report a trial.\n\n# Introduction\n\nThis study aims to test a new inhaler.\n\n# Results\n\nThe intervention improved FEV1.\n\n# Discussion\n\nThe inhaler causes lasting improvement in all patients."
}
```

```bash
python scripts/run_skill.py --case /input/case.json --out /app/output/result.json --work /app/work
```

示意输出（**示例**）：`status=success`，`findings` 含 `missing_section`(methods)、`results_without_numbers`(S3:P1)、`conclusion_boundary`(S4:P1)；`action_plan` 按 high→medium 排序给出动作。
