---
name: "agri-data-standard"
description: "设施农业（温室作物）数据标准格式规范。定义了从图像采集到智能体决策全链路的数据结构：地块管理、图像感知、目标检测、场景摘要、诊断结果、决策建议、历史记忆。适用于草莓、番茄、蓝莓、黄瓜等设施栽培作物。"
---

# 设施农业数据标准格式 Skill v1.0.0

定义温室作物监测系统的完整数据链路标准格式，包括地块管理、图像感知、目标检测、场景摘要、诊断、决策、记忆存储。

## 数据架构总览

```
地块 (plot)
  └─ 批次 (batch)
       └─ 图像采集 (image)
            ├─ 感知结果 (PerceptionResult)
            │    ├─ 目标检测列表 (StrawberryDetection[])
            │    │    ├─ 边界框 (BoundingBox)
            │    │    ├─ 成熟度 (原始/矫正后)
            │    │    ├─ 置信度
            │    │    └─ 光照特征 (亮度/对比度/阴影)
            │    └─ 场景摘要 (SceneSummary)
            │         ├─ 平均光照
            │         ├─ 时间段
            │         └─ 场景备注
            ├─ 诊断结果 (DiagnosisResult)
            ├─ 决策结果 (DecisionResult)
            └─ 记忆记录 (MemoryRecord) - 保存完整链路
```

## 详细数据结构

### 1️⃣ 地块与采集元数据

```python
image_id: str          # 唯一标识，如 "sample_morning_001"
plot_id: str           # 地块编号，如 "plot_A1"
plant_batch_id: str    # 种植批次，如 "batch_20260401"
captured_at: str       # 采集时间，ISO格式 "2026-04-27T08:30:00"
```

**设计原则：**
- 地块(plot)是空间维度：A1区、A2区、B1区...
- 批次(batch)是时间维度：20260401批次、20260410批次...
- 图像(image)是检测单元：一次采集对应一条完整记录

### 2️⃣ 感知结果 (PerceptionResult)

```python
@dataclass
class PerceptionResult:
    image_id: str                          # 图像标识
    timestamp: str                         # 采集时间戳 ISO格式
    source: str                            # 数据来源 "yolo_pinn"/"mock_yolo_v1"
    detections: list[StrawberryDetection]  # 草莓检测列表
    scene_summary: SceneSummary            # 场景摘要
```

### 3️⃣ 目标检测 (StrawberryDetection)

```python
@dataclass
class StrawberryDetection:
    bbox: BoundingBox              # 边界框 (x1, y1, x2, y2) 像素坐标
    maturity_raw: float            # 原始成熟度 (0~1)
    maturity_corrected: float      # PINN矫正后成熟度 (0~1)
    confidence: float              # 检测置信度 (0~1)
    light_features: dict           # 光照特征字典

    # light_features 包含字段:
    # {
    #   "avg_brightness": 155.0,    // 平均亮度 (0~255)
    #   "contrast": 0.72,           // 对比度 (0~1)
    #   "shadow_ratio": 0.25        // 阴影占比 (0~1)
    # }
```

**成熟度文本↔数值对照表：**

| 文本标签 | 数值 | 含义 | 建议操作 |
|---------|------|------|---------|
| "mature" | 0.95 | 完全成熟 | 可采摘 |
| "semi_mature" | 0.60 | 半成熟 | 继续观察 |
| "unripe" | 0.25 | 未成熟 | 等待 |

### 4️⃣ 场景摘要 (SceneSummary)

```python
@dataclass
class SceneSummary:
    avg_light: float       # 场景平均光照 (0~255)
    time_period: str       # 时间段: "morning"/"noon"/"evening"/"night"/"afternoon"
    notes: str             # 中文场景备注
```

**已知场景分类：**

| 时间段 | 典型光照范围 | 常见问题 |
|--------|------------|---------|
| `morning` (早晨 6:00-9:00) | 140~165 | 侧逆光、阴影 |
| `noon` (正午 11:00-14:00) | 195~235 | 过曝、眩光 |
| `evening` (傍晚 17:00-19:00) | 78~88 | 光照不足 |
| `night` (夜间 + 补光) | 108~122 | 人工光可控 |
| `afternoon` (阴天) | 92~108 | 均匀无阴影 |

### 5️⃣ 诊断结果 (DiagnosisResult)

```python
@dataclass
class DiagnosisResult:
    reliability: str              # "high"/"medium"/"low"
    reason: str                   # 诊断理由（中文）
    confidence_score: float       # 置信度 0.0~1.0
    evidence_points: list[str]    # 证据列表
    risks: list[str]              # 识别到的风险
    recommendations: list[str]    # 建议操作
```

### 6️⃣ 决策结果 (DecisionResult)

```python
@dataclass
class DecisionResult:
    harvest: bool          # 是否采摘
    fill_light: bool       # 是否补光
    manual_review: bool    # 是否人工复核
    message: str           # 面向用户的建议（中文）
    rationale: str         # 决策理由
    actions: list[str]     # 操作列表 ["harvest"/"fill_light"/"manual_review"/"continue_observation"]
```

### 7️⃣ 历史记录 (MemoryRecord)

```python
@dataclass
class MemoryRecord:
    image_id: str              # 图像标识
    timestamp: str             # 记录时间
    perception: PerceptionResult   # 感知结果
    diagnosis: DiagnosisResult     # 诊断结果
    decision: DecisionResult       # 决策结果
    metadata: dict                 # 扩展元数据
        # 包含: plot_id, plant_batch_id 等
```

## 完整数据示例

以下是一条图像采集→感知→诊断→决策的完整数据记录：

```json
{
  "image_id": "sample_morning_001",
  "plot_id": "plot_A1",
  "plant_batch_id": "batch_20260401",
  "captured_at": "2026-04-27T08:30:00",
  "perception": {
    "image_id": "sample_morning_001",
    "timestamp": "2026-04-27T08:30:00",
    "source": "yolo_pinn",
    "detection_count": 8,
    "detections": [
      {
        "bbox": {"x1": 120, "y1": 85, "x2": 200, "y2": 180},
        "maturity_raw": 0.95,
        "maturity_corrected": 0.95,
        "confidence": 0.95,
        "light_features": {"avg_brightness": 155, "contrast": 0.72}
      }
    ],
    "maturity_distribution": {"mature": 6, "semi_mature": 1, "unripe": 1},
    "scene_summary": {
      "avg_light": 151.0,
      "time_period": "morning",
      "notes": "早晨阳光充足，光照条件良好"
    }
  },
  "diagnosis": {
    "reliability": "high",
    "confidence_score": 0.82,
    "reason": "平均置信度=0.91；场景光照=151.0",
    "evidence_points": [
      "平均置信度=0.91",
      "光照条件良好",
      "与历史趋势一致"
    ],
    "risks": [],
    "recommendations": [
      "当前观测可用于采收判断"
    ]
  },
  "decision": {
    "harvest": true,
    "fill_light": false,
    "manual_review": false,
    "message": "推荐采摘；矫正成熟度=0.92",
    "rationale": "成熟度高于采摘阈值，诊断可信度接受",
    "actions": ["harvest"]
  }
}
```

## 模拟测试数据规范

设计模拟数据时，建议覆盖以下 **10个核心农业场景**：

| 场景 | 光照 | 成熟度 | 置信度 | 测试目的 |
|------|------|--------|--------|---------|
| ① 早晨充足光照 | 高 | 高 | 高 | 正常采摘流程 |
| ② 正午强逆光 | 极高 | 中 | 中 | 过曝矫正 |
| ③ 傍晚低光照 | 低 | 低 | 低 | 低光补建议 |
| ④ 夜间补光 | 中 | 中 | 中 | 人工光可靠性 |
| ⑤ 早晨逆光阴影 | 低 | 中 | 低 | 阴影矫正 |
| ⑥ 阴天均匀光 | 中 | 高 | 高 | 均匀光评估 |
| ⑦ 高成熟度集中 | 高 | 极高 | 高 | 集中采摘建议 |
| ⑧ 低成熟度早期 | 高 | 低 | 高 | 等待守候 |
| ⑨ 病虫害风险 | 中 | 中 | 中 | 异常检测 |
| ⑩ 最佳采摘窗口 | 极高 | 极高 | 极高 | 最优决策 |

## 扩展指南

将此标准适配到其他作物时需修改：
- **番茄**：调整成熟度阈值（harvest ≥ 0.75 因变色期较长）
- **蓝莓**：增加果实直径字段，调整采摘阈值（harvest ≥ 0.85）
- **黄瓜**：增加长度测量，替换成熟度为"尺寸等级"

## 引用

- 数据格式定义参考自 `schemas.py` 中的 `dataclass` 定义
- 模拟数据规范参考自 `sample_data/batch_001.py` 的10个测试场景
