# Generate VQA

如果你已经跑通 `vqa_config.yaml`，可以按下面流程系统验证“多种 VQA 生成方法”（更准确地说：多种 **VQA 生成策略组合**）。

## 1) 先明确可对比维度

当前 VQA 入口仍是 `generate.params.method: vqa`，可变化的核心维度主要有：

- **分区策略（partition）**：例如 `anchor_bfs`（图像锚点）、`ece`、`dfs`、`leiden`。
- **大模型配置**：不同模型、不同推理后端（API / vLLM / SGLang / HF）。
- **数据格式**：`ChatML`、`Sharegpt`、`Alpaca`。
- **提示词模板**：在 `graphgen/templates/generation/vqa_generation.py` 中可扩展不同 prompt 版本进行 A/B。

> 建议把“方法”定义为：`partition × model/backend × prompt_version × data_format`。

---

## 2) 构建实验矩阵（推荐最小集合）

先做一个 2×2×2 的小规模矩阵，快速得到结论：

- partition: `anchor_bfs`, `ece`
- model: `Model-A`, `Model-B`
- prompt: `v1`, `v2`

共 8 组。每组固定同一批输入样本和随机种子（若后端支持），保证可比性。

---

## 3) 为每组方法准备配置文件

以 `examples/generate/generate_vqa/vqa_config.yaml` 为基线，复制出多份：

- `vqa_anchor_bfs_modelA_v1.yaml`
- `vqa_ece_modelA_v1.yaml`
- `vqa_anchor_bfs_modelB_v2.yaml`
- ...

关键改动位置：

- `nodes.partition.params.method`
- `nodes.partition.params.method_params`
- `nodes.generate.params.data_format`
- 环境变量中的模型/后端（见根目录 `README.md` 的后端配置示例）

---

## 4) 运行生成与评测

### 4.1 运行单组 VQA 生成

```bash
python3 -m graphgen.run --config_file examples/generate/generate_vqa/<your_config>.yaml
```

### 4.2 在同一条 pipeline 中加入评测节点（推荐）

在配置里追加 `evaluate` 节点（参考 `examples/evaluate/evaluate_qa/qa_evaluation_config.yaml`），可先启用：

- `length`
- `mtld`

后续再加 LLM-as-a-judge 指标（如 `reward_score`、`uni_score`）。

---

## 5) 建议的对比指标

至少分三层：

1. **基础质量**：长度分布、词汇多样性（MTLD）、重复率。
2. **任务质量**：
   - 问题是否依赖图像（不是纯文本就能回答）
   - 答案可验证性（是否能从图像/文本证据中定位）
   - 幻觉率（无依据事实）
3. **业务可用性**：人工抽样打分（正确性、清晰度、难度、覆盖度）。

建议每组抽样 100 条做人评，自动指标+人评联合决策。

---

## 6) 推荐评测流程（可直接执行）

1. 固定输入集（训练/验证拆分）。
2. 批量跑完所有方法组。
3. 统一导出 `cache/output` 下结果并重命名归档。
4. 计算自动指标（length/mtld/奖励模型分）。
5. 对每组随机抽样做人评（双人标注更稳）。
6. 输出 leaderboard（总分 + 分维度分数）。

---

## 7) 结果汇报模板（给领导）

建议用一页表格说明：

- 方法配置（partition/model/prompt）
- 自动指标（MTLD、reward）
- 人评指标（正确性、图像相关性、可验证性）
- 结论（最优方法 + 次优备选 + 适用场景）

这样既能说明“验证了多种方法”，也能落地到最终可选型方案。
