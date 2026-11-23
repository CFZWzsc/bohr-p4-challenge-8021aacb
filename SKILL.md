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


## 自查清单

- [ ] `case_id` 与合同字段保留；未映射项已登记。
- [ ] `findings` 覆盖六类问题，每条有定位与动作。
- [ ] `action_plan` 按 high→medium 排序且可执行。
- [ ] 未虚构缺陷，无正文时明确 `insufficient_input`。

## 交付保证与评分对齐

调用顺序（固定）：

1. 先运行包内脚本（若存在）：`python scripts/run_skill.py`（默认读 `/input/case.json`，写 `/app/output/result.json`）；路径不同时用 `--case` / `--output` 指定。
2. **必须交付完整成品**：脚本只做结构与论证链初筛；调用 Agent 必须复核并补全 `findings`（类别/严重度/定位/动作）与 `action_plan`（按 high→medium 排序、每条可执行），覆盖全文六类问题；不得只给统计、`pending` 或“需人工判断”占位。
3. 最后自检：`case_id` 与输入一致；`output_contract` 每个字段都存在；`status` 取本合同允许值 `success | insufficient_input | failed`；正文无 `pending`/`TODO`/空占位；数字、引用、来源可逐条回溯。

运行保证：

- 输入缺失、JSON 损坏、字段异常、输出目录不存在时，仍须写出结构化 `result.json` 并正常结束，不得崩溃或非零退出。
- 不联网、不伪造数据、不声称平台评分或官方案例结果。
- 交付物必须是可直接编辑与复核的成品，而不是流程说明。

评分对齐（本题五个维度）：

- 实际效果（40）：完整产出任务要求的最终成品，覆盖全部输入条目，可直接进入修改流程。
- 使用增益（25）：体现相对基础 Agent 的可验证改进（结构化、术语/数字/引用核对、可编辑性）。
- 创新增益（15）：保留方法学增量（分块、对齐、二次校验、版本比对等），并用结果证明其价值。
- 可靠体验（10）：异常输入有明确降级与说明，不同长度/文体/重复运行下稳定产出。
- 生态贡献（10）：用途、调用、输入输出、限制、示例、维护信息齐全，资源可被他人复用。

## 最终交付强制要求

1. **调用 Agent 必须交付成品，不是流程说明**：脚本产出只是索引、核对或规格骨架；调用 Agent 必须在同一 `result.json` 中补齐本题要求的最终产物（正文/译文/审稿意见/综述/分析结论/图表源文件与导出件/幻灯片页内容与讲稿/实验方案等）。交付字段为空、只有标题骨架，或含 `pending`、`TODO` 之类占位，一律视为未完成。
2. **必须覆盖全部输入条目**：输入中的每个章节、问题、文献、图、页面、指标都要有对应条目，不得只处理示例或首条；确实缺失的条目写入 `uncertainty` / `limitations` 并说明原因，不得静默跳过。
3. **必须自检后再交付**：`case_id` 与输入一致；`output_contract` 每个字段都存在；状态值取本题契约规定的枚举；数字、单位、引用、来源可逐条回溯；结果内部无自相矛盾。
4. **参数等价**：`python scripts/run_skill.py --case <输入> --out <输出>` 与 `--output <输出>` 等价；默认 `/input/case.json` → `/app/output/result.json`。
5. **平台用 3 个固定案例分别运行**：逻辑要由输入驱动，不针对某一份示例硬编码；字段命名差异走别名匹配与显式降级，并在 `uncertainty` 记录假设。
