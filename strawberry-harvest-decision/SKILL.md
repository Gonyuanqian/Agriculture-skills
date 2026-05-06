---
name: "strawberry-harvest-decision"
description: "草莓成熟度评估与采收决策。基于YOLO/PINN感知数据、光照诊断结果和历史趋势，生成采摘、补光、人工复核等农业操作建议。决策逻辑包含成熟度阈值、可信度检查和历史趋势佐证。"
---

# 草莓成熟度与采收决策 Skill v1.0.0

基于感知数据和诊断结果，为草莓温室生产生成可操作的农业建议。

## 核心阈值参数

| 参数 | 默认值 | 农业含义 |
|------|--------|---------|
| `harvest_threshold` | 0.80 | 矫正成熟度≥0.80 → 推荐采摘 |
| `observe_threshold` | 0.60 | 矫正成熟度≥0.60 → 进入观察窗口 |
| `low_light_threshold` | 0.30 | 场景光照<0.30 → 需要补光 |

## 成熟度数值映射

YOLO输出文本标签需映射为数值：

| 文本标签 | 数值 |
|---------|------|
| "mature" (成熟) | 0.95 |
| "semi_mature" (半熟) | 0.60 |
| "unripe" (未熟) | 0.25 |

## 决策树

```
输入：感知数据 + 诊断结果 + 历史记录
↓
计算平均矫正成熟度 (avg_maturity) 和场景光照 (scene_light)
↓
┌─ 可信度 = "low" 或 检测数量=0 ─────────────────────────────────────┐
│   → 人工复核：harvest=false, manual_review=true                    │
│   → 补光判断：scene_light<0.30 则 fill_light=true                  │
│   → 消息："建议人工复核，估计成熟度=XX，可信度=low"                │
└────────────────────────────────────────────────────────────────────┘
│
┌─ avg_maturity ≥ 0.80 AND 可信度∈{"high","medium"} ──────────────┐
│   → 推荐采摘：harvest=true                                       │
│   → 补光：如果可信度≠"high"且光照低则补光，否则不补光            │
│   → 消息："推荐采摘，成熟度=XX，历史记录N条佐证"                 │
└────────────────────────────────────────────────────────────────────┘
│
┌─ avg_maturity ≥ 0.60 ─────────────────────────────────────────────┐
│   → 继续观察：harvest=false                                       │
│   → 可信度="medium"则人工复核                                     │
│   → 消息："接近采摘窗口，建议继续观察"                            │
└────────────────────────────────────────────────────────────────────┘
│
┌─ 其他 (avg_maturity < 0.60) ────────────────────────────────────┐
│   → 等待：harvest=false                                          │
│   → 补光判断：scene_light<0.30 则补光                            │
│   → 消息："尚未成熟，矫正成熟度=XX，建议等待"                     │
└────────────────────────────────────────────────────────────────────┘
```

## 数据结构

```python
@dataclass
class DecisionResult:
    harvest: bool          # 是否推荐采摘
    fill_light: bool       # 是否需要补光
    manual_review: bool    # 是否需要人工复核
    message: str           # 面向用户的中文建议
    rationale: str         # 决策理由
    actions: list[str]     # 操作列表
```

## 决策结果示例

### 示例1：推荐采摘
```json
{
  "harvest": true,
  "fill_light": false,
  "manual_review": false,
  "message": "推荐采摘；矫正成熟度=0.92，佐证历史记录3条",
  "rationale": "成熟度高于采摘阈值，诊断可信度接受",
  "actions": ["harvest"]
}
```

### 示例2：继续观察（接近窗口）
```json
{
  "harvest": false,
  "fill_light": true,
  "manual_review": true,
  "message": "继续密切观察；矫正成熟度=0.65，可信度=medium",
  "rationale": "接近采摘范围，受益于另一轮观察",
  "actions": ["fill_light", "manual_review", "continue_observation"]
}
```

### 示例3：人工复核
```json
{
  "harvest": false,
  "fill_light": true,
  "manual_review": true,
  "message": "建议人工复核后再决定采摘；估计矫正成熟度=0.42，可信度=low",
  "rationale": "诊断可信度过低，无法自动推荐采摘",
  "actions": ["fill_light", "manual_review"]
}
```

### 示例4：等待成熟
```json
{
  "harvest": false,
  "fill_light": false,
  "manual_review": false,
  "message": "尚未成熟；矫正成熟度=0.32",
  "rationale": "成熟度低于观察阈值",
  "actions": ["continue_observation"]
}
```

## 操作列表对照

| 操作 | 含义 | 触发条件 |
|------|------|---------|
| `harvest` | 安排采摘 | 成熟度≥0.80且可信度高 |
| `fill_light` | 开启补光灯 | 光照<0.30 或 诊断出低光风险 |
| `manual_review` | 人工现场复核 | 可信度低 或 中等可信度+观察窗口 |
| `continue_observation` | 继续观察 | 未触发采摘且不需要复核 |
| `no_action` | 无需操作 | 无任何操作触发 |

## 多批次对比建议

当连续批次数据可用时，可计算成熟趋势：
- **稳定上升趋势** → 接近采摘窗口，加强监测频率
- **突然下降** → 检查是否存在病虫害或环境异常
- **停滞不变** → 检查养分供应或光照条件

## 引用

- 决策逻辑参考自 `decision_agent.py` 的 `RuleBasedDecisionAgent` 类
- 成熟度映射参考自 `decision_agent.py` 的 `_average_corrected_maturity()` 方法
