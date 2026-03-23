# AMP 指令微调数据集设计模板

本文档给出一个可直接落地的抗菌肽（AMP）指令微调数据集设计模板，用于构建面向 **检索问答、序列分析、候选生成、风险提示** 的中文/中英双语 AMP 助手模型。

## 1. 设计目标

该数据集建议支持以下四类核心能力：

1. **知识问答**：回答已知 AMP 的来源、活性、靶标、临床阶段、实验指标等。
2. **序列分析**：输入一条序列，输出理化性质、结构倾向、可能活性及风险提示。
3. **条件生成**：根据目标菌、长度、电荷、低溶血等约束生成候选序列。
4. **科研助手式表达**：将数据库事实、模型预测、设计建议区分清楚，并保持可解释性。

## 2. 推荐数据格式

建议优先使用 **JSONL**，每行一个样本，便于后续对接 LLaMA-Factory、Axolotl、Transformers Trainer 等框架。

### 2.1 基础格式（Alpaca 风格）

```json
{
  "instruction": "分析这条抗菌肽序列的基础特征，并判断其是否可能对革兰阴性菌有活性。",
  "input": "KWKSFIKKLTSAAKKVVTTAKPLISS",
  "output": "序列长度为26 aa，预测净电荷为正，具有较明显的两亲性和α-螺旋倾向。该序列富含Lys，符合典型阳离子抗菌肽特征。若结合同类序列经验，可能对部分革兰阴性菌具有活性，但具体MIC仍需实验验证。"
}
```

### 2.2 扩展格式（推荐）

如果你希望后续支持数据审计、分任务采样和质量控制，建议增加元数据字段。

```json
{
  "id": "amp_ins_000001",
  "task_type": "sequence_analysis",
  "instruction": "分析下面序列的结构和活性特征。",
  "input": "KWKSFIKKLTSAAKKVVTTAKPLISS",
  "output": "该序列长度为26 aa，偏阳离子，推测具备α-螺旋倾向和一定膜破坏潜力。建议进一步关注其对革兰阴性菌的MIC和溶血风险。",
  "source_type": "annotated_db",
  "source_ids": ["DRAMP:12345", "APD:AP00123"],
  "language": "zh",
  "grounding_level": "database_plus_rule",
  "safety_label": "requires_validation",
  "split": "train"
}
```

## 3. 推荐任务类型与样本配比

建议不要只做“生成新序列”，而是按任务分桶构建数据集。

| task_type | 任务说明 | 推荐占比 |
| --- | --- | --- |
| `knowledge_qa` | 已知肽问答、数据库事实问答 | 25% |
| `sequence_analysis` | 给定序列做理化/结构/活性分析 | 25% |
| `retrieval_summary` | 基于检索结果进行汇总和对比 | 15% |
| `candidate_generation` | 按约束生成候选 AMP | 20% |
| `safety_risk_notice` | 毒性、溶血、稳定性、可实验性提示 | 10% |
| `negative_example` | 拒答、纠错、证据不足说明 | 5% |

### 为什么这样分配

- **knowledge_qa** 保证模型能“说对已知事实”；
- **sequence_analysis** 保证模型能“看懂序列”；
- **candidate_generation** 才是设计能力的核心；
- **negative_example** 能减少模型幻觉和过度自信回答。

## 4. 各任务模板

---

## 4.1 知识问答模板（knowledge_qa）

### 模板 A：单条已知肽事实问答

```json
{
  "task_type": "knowledge_qa",
  "instruction": "请介绍抗菌肽 {name} 的已知活性和应用信息。",
  "input": "",
  "output": "{name} 的已知活性包括：{activity_summary}。主要靶标包括：{target_summary}。当前临床阶段为：{clinical_stage}。若需进一步判断其成药潜力，还应结合溶血性、稳定性和体内数据综合评估。"
}
```

### 模板 B：条件筛选型问答

```json
{
  "task_type": "knowledge_qa",
  "instruction": "列出已知对 {target_organism} 有活性，且长度在 {len_min}-{len_max} aa 的抗菌肽。",
  "input": "",
  "output": "符合条件的候选包括：{candidate_list}。其中可优先关注 {top_candidate}，因为其具有 {reason_summary}。"
}
```

### 适合的数据来源

- 三库整合后的 `annotated_amp_database`
- 临床注释表
- 活性注释和实验结果表

---

## 4.2 序列分析模板（sequence_analysis）

### 模板 A：基础分析

```json
{
  "task_type": "sequence_analysis",
  "instruction": "请分析以下序列的基础理化特征，并判断其是否像典型抗菌肽。",
  "input": "{sequence}",
  "output": "该序列长度为 {length} aa，预测净电荷为 {net_charge}，疏水性指标为 {hydrophobicity}，{secondary_structure_hint}。综合来看，它{amp_likelihood_statement}。"
}
```

### 模板 B：面向靶标的分析

```json
{
  "task_type": "sequence_analysis",
  "instruction": "请评估这条序列是否可能对 {target_organism} 有活性，并说明依据。",
  "input": "{sequence}",
  "output": "从序列特征看，该肽 {feature_summary}。若与已知针对 {target_organism} 的阳离子两亲性肽进行比较，它{comparison_statement}。因此可初步判断 {activity_judgement}，但仍需实验验证。"
}
```

### 输出要点

建议输出中包含以下几个固定维度：

- 长度
- 净电荷
- 疏水性
- 二级结构倾向
- 是否具有典型 AMP 特征
- 活性判断的依据
- 风险或验证提示

---

## 4.3 检索增强汇总模板（retrieval_summary）

这类样本用于训练模型“读懂检索结果并归纳”。

### 模板

```json
{
  "task_type": "retrieval_summary",
  "instruction": "基于给定候选记录，总结哪些抗菌肽更适合用于 {use_case}。",
  "input": "候选记录：\n1. {record_1}\n2. {record_2}\n3. {record_3}",
  "output": "综合比较，{best_candidate} 更适合用于 {use_case}。主要原因包括：{reason_1}；{reason_2}；{reason_3}。如果优先考虑低溶血风险，则应重点关注 {risk_adjusted_candidate}。"
}
```

### 适用场景

- RAG 检索后总结
- 多条候选 AMP 对比
- 临床阶段/毒性/活性综合排序说明

---

## 4.4 候选生成模板（candidate_generation）

这部分最关键，但也最容易让模型“编得很像”。建议强制输出固定格式。

### 模板 A：单候选生成

```json
{
  "task_type": "candidate_generation",
  "instruction": "生成一个针对 {target_organism} 的候选抗菌肽，要求长度 {len_min}-{len_max} aa，净电荷不低于 {min_charge}，并尽量降低溶血风险。",
  "input": "",
  "output": "候选序列：{generated_sequence}\n设计理由：该序列长度为 {length} aa，预计净电荷为 {charge}，疏水残基占比适中，具有一定两亲性和α-螺旋潜力。\n风险提示：该序列为模型设计结果，并非实验验证结论，建议进一步进行毒性、溶血性和MIC测试。"
}
```

### 模板 B：多候选生成 + 排序

```json
{
  "task_type": "candidate_generation",
  "instruction": "请给出 3 条针对 {target_organism} 的候选 AMP，并按预期成药性从高到低排序。",
  "input": "",
  "output": "候选1：{seq_1}；理由：{reason_1}。\n候选2：{seq_2}；理由：{reason_2}。\n候选3：{seq_3}；理由：{reason_3}。\n说明：以上均为设计候选，需通过活性、毒性和稳定性实验进一步确认。"
}
```

### 生成任务数据构造建议

生成类样本建议混合三种来源：

1. **真实已知肽改写**：把已知 AMP 的条件反推成设计任务；
2. **模板重写样本**：用规则生成高质量 instruction/output；
3. **人工审核样本**：抽样人工修订，提高表达质量和科学性。

---

## 4.5 风险提示与拒答模板（safety_risk_notice / negative_example）

这是非常重要但经常被忽略的部分。

### 模板 A：证据不足时的回答

```json
{
  "task_type": "negative_example",
  "instruction": "请断定这条新序列一定可以在体内治疗耐药菌感染。",
  "input": "{sequence}",
  "output": "无法仅凭序列直接断定其一定具备体内治疗效果。体内疗效还取决于稳定性、毒性、药代动力学、免疫反应和给药方式等因素。当前最多只能做基于序列特征的初步推测，仍需实验和动物模型验证。"
}
```

### 模板 B：安全边界提示

```json
{
  "task_type": "safety_risk_notice",
  "instruction": "请评估该序列的应用风险。",
  "input": "{sequence}",
  "output": "该序列可能具备一定抗菌肽特征，但应用风险仍包括：潜在溶血性、细胞毒性、蛋白酶降解风险和体内稳定性不足。建议将其作为研究候选而非直接临床结论。"
}
```

## 5. 推荐字段设计

如果你希望后续更方便做数据过滤、训练采样和误差分析，建议每条样本至少保留以下字段：

| 字段名 | 含义 |
| --- | --- |
| `id` | 样本唯一编号 |
| `task_type` | 任务类型 |
| `instruction` | 指令 |
| `input` | 输入内容 |
| `output` | 标准答案 |
| `source_type` | 来源类型，如 database / synthetic / manual |
| `source_ids` | 溯源 ID 列表 |
| `language` | `zh` / `en` / `zh-en` |
| `grounding_level` | database_only / database_plus_rule / model_assisted |
| `quality_score` | 样本质量分 |
| `safety_label` | 是否涉及高风险生成或需验证 |
| `split` | train / valid / test |

## 6. 输出风格规范

为了让模型更像一个科研助手，而不是“会瞎编的聊天机器人”，建议统一输出风格。

### 6.1 必须遵守的风格要求

- **优先陈述事实，再给预测。**
- **明确区分数据库证据与模型推断。**
- **生成结果必须附带风险提示。**
- **避免使用“确定有效”“必然低毒”之类绝对化表述。**
- **结尾尽量附一句“仍需实验验证”。**

### 6.2 推荐措辞模板

可在 `output` 中高频复用：

- “根据现有数据库信息……”
- “从序列特征推测……”
- “该结论属于初步判断……”
- “仍需结合 MIC、溶血性和稳定性实验验证……”

## 7. 训练集 / 验证集 / 测试集切分建议

推荐按 **肽级别** 而不是样本级别切分，避免同一条序列以不同问法同时出现在训练集和测试集。

### 推荐比例

- `train`: 80%
- `valid`: 10%
- `test`: 10%

### 切分原则

1. 同一 `amp_id` 只能出现在一个 split 中；
2. 同一家族肽最好做部分隔离，避免过拟合；
3. 生成任务可额外保留一个“高难度测试集”，专门评估新约束组合。

## 8. 数据质量控制清单

在正式训练前，建议对数据做一次 QC。

### 8.1 文本质量

- 是否有重复样本
- 是否有空输出
- 是否存在明显模板错误
- 中英文术语是否统一
- 输出是否包含自相矛盾描述

### 8.2 科学性质量

- 预测值是否与序列特征严重冲突
- 活性标签是否与来源记录一致
- 临床阶段、毒性、MIC 是否有误映射
- 是否错误地把“设计结果”写成“实验结论”

### 8.3 安全性质量

- 是否存在鼓励过度确定性结论的样本
- 是否缺少风险提示
- 是否把未验证新序列写成可直接应用方案

## 9. 最小可用样本规模建议

不同阶段建议如下：

| 阶段 | 建议规模 | 目标 |
| --- | --- | --- |
| 原型验证 | 2,000 - 5,000 条 | 验证输出格式和任务覆盖 |
| 初版微调 | 10,000 - 30,000 条 | 形成稳定科研助手表达 |
| 增强版本 | 50,000+ 条 | 提升泛化能力和任务细分表现 |

## 10. 建议目录结构

```text
training_data/
├── raw/
│   ├── amp_knowledge_qa.jsonl
│   ├── amp_sequence_analysis.jsonl
│   ├── amp_candidate_generation.jsonl
│   └── amp_negative_examples.jsonl
├── merged/
│   └── amp_instruction_tuning_all.jsonl
├── splits/
│   ├── train.jsonl
│   ├── valid.jsonl
│   └── test.jsonl
└── reports/
    ├── data_profile.md
    └── quality_check.csv
```

## 11. 一组可直接复用的样本模板

### 示例 1：知识问答

```json
{
  "id": "amp_ins_qa_0001",
  "task_type": "knowledge_qa",
  "instruction": "请概述肽 DRAMP_X 的活性特点。",
  "input": "",
  "output": "根据数据库记录，DRAMP_X 主要表现出抗菌活性，重点靶标包括金黄色葡萄球菌和大肠杆菌。若进一步评估其应用潜力，建议同时查看 MIC、溶血性和稳定性数据。",
  "source_type": "annotated_db",
  "source_ids": ["DRAMP:DRAMP_X"],
  "language": "zh",
  "grounding_level": "database_only",
  "quality_score": 0.95,
  "safety_label": "low_risk",
  "split": "train"
}
```

### 示例 2：序列分析

```json
{
  "id": "amp_ins_seq_0001",
  "task_type": "sequence_analysis",
  "instruction": "请分析该序列是否符合典型阳离子抗菌肽特征。",
  "input": "KWKSFIKKLTSAAKKVVTTAKPLISS",
  "output": "该序列长度适中，富含 Lys 残基，预测整体带正电，并具有一定两亲性和 α-螺旋倾向，符合典型阳离子抗菌肽的常见特征。但其实际活性和溶血风险仍需实验验证。",
  "source_type": "annotated_db",
  "source_ids": ["SEQ:demo_0001"],
  "language": "zh",
  "grounding_level": "database_plus_rule",
  "quality_score": 0.96,
  "safety_label": "requires_validation",
  "split": "train"
}
```

### 示例 3：候选生成

```json
{
  "id": "amp_ins_gen_0001",
  "task_type": "candidate_generation",
  "instruction": "生成一个针对 MRSA 的候选抗菌肽，长度 20-24 aa，净电荷不低于 +5，并尽量降低溶血风险。",
  "input": "",
  "output": "候选序列：KWRKAFKKIAKKLAAKFLKK\n设计理由：该序列长度为20 aa，预计带较强正电，兼具疏水和阳离子残基，符合典型膜作用型 AMP 的常见构型。\n风险提示：该结果为模型生成候选，并非实验验证结论，建议进一步测试其 MIC、溶血性及稳定性。",
  "source_type": "synthetic_template",
  "source_ids": ["RULE:gen_template_v1"],
  "language": "zh",
  "grounding_level": "model_assisted",
  "quality_score": 0.90,
  "safety_label": "requires_validation",
  "split": "train"
}
```

## 12. 最终建议

如果你要尽快启动 LoRA/QLoRA 微调，建议按以下顺序制作数据：

1. 先从 **knowledge_qa + sequence_analysis** 开始；
2. 再加入 **retrieval_summary**，让模型学会基于证据组织表达；
3. 最后加入 **candidate_generation + negative_example**，提升设计能力和安全边界；
4. 每一轮训练后都抽样人工审查高风险输出，持续回流修正样本。

这样做的好处是：模型先学会“说对”和“解释清楚”，再学“生成新序列”，整体成功率会高很多。
