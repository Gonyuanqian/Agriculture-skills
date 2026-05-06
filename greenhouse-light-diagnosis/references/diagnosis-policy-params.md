# DiagnosisPolicy 参数详解

## 完整参数列表

```python
@dataclass
class DiagnosisPolicy:
    low_light_threshold: float = 0.30         # 低光照阈值
    bright_light_threshold: float = 0.80      # 过曝阈值
    shadow_ratio_threshold: float = 0.45      # 阴影占比阈值
    correction_gap_threshold: float = 0.15    # RAW与矫正差距阈值
    low_confidence_threshold: float = 0.60    # 低置信度阈值
    high_confidence_threshold: float = 0.85   # 高置信度阈值
    stable_history_gap_threshold: float = 0.10  # 历史稳定偏差阈值
```

## 农作物调参参考

| 作物 | low_light | bright_light | shadow_ratio | correction_gap | harvest_threshold |
|------|-----------|--------------|--------------|----------------|-------------------|
| 🍓 草莓 | 0.30 | 0.80 | 0.45 | 0.15 | 0.80 |
| 🍅 番茄 | 0.25 | 0.85 | 0.40 | 0.12 | 0.75 |
| 🫐 蓝莓 | 0.20 | 0.90 | 0.50 | 0.10 | 0.85 |
| 🥒 黄瓜 | 0.20 | 0.85 | 0.35 | 0.10 | N/A(按尺寸) |

## 扣除分数参考表

| 诊断发现 | 扣分 | 触发条件 |
|---------|------|---------|
| 低置信度 | -0.35 | avg_confidence < low_confidence_threshold |
| 矫正差距大 | -0.20 | avg_gap > correction_gap_threshold |
| 低光照 | -0.20 | avg_brightness < low_light_threshold |
| 过曝 | -0.10 | avg_brightness > bright_light_threshold |
| 阴影过多 | -0.15 | avg_shadow_ratio > shadow_ratio_threshold |
| 历史一致 | +0.10 | 与历史均值偏差 ≤ 0.10 |
| 历史偏差 | -0.10 | 与历史均值偏差 > 0.10 |

## 可靠度等级映射

| 最终分数 | 等级 | 含义 |
|---------|------|------|
| ≥ 0.80 | high | 数据可靠，可直接用于决策 |
| ≥ 0.55 | medium | 数据部分可信，需谨慎使用 |
| < 0.55 | low | 数据不可靠，建议人工介入 |
