---
name: "greenhouse-light-diagnosis"
description: "温室作物光照诊断与数据可靠性评估。基于光照分析、阴影检测、置信度评估和历史趋势对比，诊断作物图像数据的可靠性，识别低光、过曝、阴影覆盖等风险，生成诊断建议。适用于草莓、番茄、蓝莓等设施栽培作物。"
---

# 温室作物光照诊断 Skill v1.0.0

基于温室环境感知数据（YOLO/PINN或其他视觉检测），诊断数据可靠性和光照相关问题，识别风险并生成建议。

## 核心阈值参数

以下阈值经过草莓温室场景验证，可根据具体作物调整：

| 参数 | 默认值 | 农业含义 |
|------|--------|---------|
| `low_light_threshold` | 0.30 | 平均光照<0.30 → 低光风险，需补光 |
| `bright_light_threshold` | 0.80 | 平均光照>0.80 → 过曝风险，可能有眩光 |
| `shadow_ratio_threshold` | 0.45 | 阴影占比>0.45 → 建议调整拍摄角度 |
| `correction_gap_threshold` | 0.15 | PINN矫正差>0.15 → 光照严重干扰成熟度判断 |
| `low_confidence_threshold` | 0.60 | 置信度<0.60 → 建议重新采集图像 |
| `high_confidence_threshold` | 0.85 | 置信度≥0.85 → 数据可信 |
| `stable_history_gap` | 0.10 | 与历史均值偏差>0.10 → 发出异常信号 |

## 诊断流程

```
输入：感知数据(PerceptionResult) + 历史记录(MemoryRecord[])
↓
1. 计算平均置信度、平均矫正差、平均亮度、平均阴影比
↓
2. 逐一检查每个阈值，累加/扣减可靠度分数 (初始1.0)
   - 置信度<0.60 → -0.35
   - 矫正差>0.15 → -0.20
   - 亮度<0.30 → -0.20 (低光)
   - 亮度>0.80 → -0.10 (过曝)
   - 阴影>0.45 → -0.15
   - 与历史一致 → +0.10
   - 与历史偏差>0.10 → -0.10
↓
3. 映射可靠度分数到等级标签
   - ≥0.80 → "high" (高)
   - ≥0.55 → "medium" (中)
   - <0.55 → "low" (低)
↓
输出：DiagnosisResult
```

## 数据结构

```python
@dataclass
class DiagnosisResult:
    reliability: str          # "high" / "medium" / "low"
    reason: str               # 诊断理由（中文描述）
    confidence_score: float   # 0.0 ~ 1.0
    evidence_points: list[str]   # 证据列表
    risks: list[str]             # 识别到的风险
    recommendations: list[str]   # 建议操作
```

## 诊断输出示例

```json
{
  "reliability": "medium",
  "confidence_score": 0.67,
  "reason": "检测到 6 颗草莓；平均置信度=0.84；平均矫正差=0.05；场景光照=0.22；",
  "evidence_points": [
    "平均置信度=0.84",
    "平均矫正差=0.05",
    "场景光照=0.22",
    "与3条历史记录一致"
  ],
  "risks": [
    "低光照观测（光照<0.30，建议在更强自然光或补光下重新拍摄）"
  ],
  "recommendations": [
    "在更强自然光或补光条件下重新拍摄",
    "当前观测可用于参考，但建议谨慎对待成熟度判断"
  ]
}
```

## 常见诊断场景

### 场景A：早晨充足光照
- 亮度>0.30，置信度高(>0.85)，阴影少
- → 可靠性"high"，建议用于采收判断

### 场景B：正午强逆光
- 亮度>0.80，阴影多(>0.45)，置信度中等
- → 可靠性"low"/"medium"，建议调整角度

### 场景C：傍晚/阴天低光
- 亮度<0.30，置信度偏低(<0.85)
- → 可靠性"medium"，建议补光

### 场景D：夜间人工补光
- 亮度适中(0.30~0.80)，置信度中等
- → 可靠性"medium"，光照人工可控

### 场景E：连续阴雨天
- 亮度均匀(<0.80)，置信度尚可
- → 可靠性"medium"~"high"，关注湿度衍生病虫害

## 光照特征字段

每个草莓检测对象包含以下光照特征：

```json
{
  "avg_brightness": 155.0,     // 平均亮度（0~255）
  "contrast": 0.72,             // 对比度
  "shadow_ratio": 0.25         // 阴影占比
}
```

## 引用

- 阈值设计参考自 `rule_based_diagnosis.py` 的 `DiagnosisPolicy` 类
- 历史趋势分析逻辑参考自 `_analyze_history()` 方法
