---
name: show-k
description: |
  当用户说"显示K线"、"show K线"、"看K线"、"K线图"时，或 backtest-runner 回测完成后，
  **必须使用此 skill**。自动在回测日志中找到最新数据，生成并显示K线图表（含指标和交易记录）。
---

# Show-K Skill

## 步骤 1：确定回测日志目录

优先从上下文获取（backtest-runner 会传递），否则：
- 如果当前在某个策略 test 目录下，使用 `log/` 子目录
- 常见的回测日志在 `test/strategy_XXX/log/` 或项目根目录 `log/` 下

找到最新的回测日志目录：
```bash
latest_log=$(ls -dt test/strategy_TQA_V2/log/TQA_* test/strategy_TQA_V2_3_MeanRevertPulse/log/S_* log/TQA_* 2>/dev/null | head -1)
```

## 步骤 2：识别文件

在最新日志目录中查找：
- K线数据：`*_D1.csv` 或 `*_H1.csv`
- 交易记录：`*_trade.csv`
- 分布记录：`*_dist.csv`（可选）
- 资金曲线：`*_daily_list.csv`

## 步骤 3：资金曲线叠加

如果存在 `*_daily_list.csv`，将资金曲线合并到 D1/H1 数据中：

1. 读取 `*_daily_list.csv`，提取 `date` 和 `net` 列
2. 读取 K线数据 CSV，按日期 merge
3. 计算 NetCurve = price_mean + (net / 1000000.0 - 1.0) × 50000
   - `price_mean`：close 均价，使曲线对齐到价格量级
   - `50000`：盈利百分比放大器，1% 盈利 ≈ 500 图表单位波动
4. 保存为 `*_with_capital.csv`（或 `*_with_net.csv`）
5. 将 `NetCurve` 加入 `main_indicators`

```python
import pandas as pd

df = pd.read_csv(f"{log_dir}/Strategy_D1.csv")
daily = pd.read_csv(f"{log_dir}/Strategy_daily_list.csv")
daily['dt'] = pd.to_datetime(daily['date'])
df['dt'] = pd.to_datetime(df['datetime'])
df['date_only'] = df['dt'].dt.date.astype(str)
daily['date_only'] = daily['dt'].dt.date.astype(str)
merged = df.merge(daily[['date_only', 'net']], on='date_only', how='left')
merged['net'] = merged['net'].ffill()

init_cap = 1000000.0
price_ref = merged['close'].mean()
merged['NetCurve'] = price_ref + (merged['net'] / init_cap - 1.0) * 50000
merged = merged.drop(columns=['dt', 'date_only'])
merged.to_csv(f"{log_dir}/Strategy_D1_with_capital.csv", index=False)
```

## 步骤 4：K 线图核心修改（kline.py）

以下是 `vnpy/trader/ui/kline/kline.py` 中已完成的修改，新环境部署时需确保这些改动存在：

### 4a. NetCurve 紫色虚线样式

在 `add_indicator` 方法中，NetCurve 使用特殊样式：

```python
if indicator == 'NetCurve':
    from pyqtgraph import functions as fn
    self.main_indicator_colors[indicator] = [fn.mkPen({'color': '#8B00FF', 'width': 3, 'style': 2})]
else:
    self.main_indicator_colors[indicator] = self.main_color_pool[0]
    self.main_color_pool.append(self.main_color_pool.popleft())
```

- `#8B00FF`：深紫色，与红绿黄价格线区分
- `width: 3`：略粗于价格指标线
- `style: 2`：Qt.DashLine 虚线，不与 K 线混淆

### 4b. 交易箭头大小统一

平仓箭头（sell/cover）改为与开仓箭头一致的小箭头：

```python
# cover（多头平仓）
arrow = pg.ArrowItem(pos=(x, price), angle=45, brush='y', pen=None,
                     tipAngle=30, baseAngle=20, tailLen=10, tailWidth=2)
# sell（空头平仓）
arrow = pg.ArrowItem(pos=(x, price), angle=-45, brush='g', pen=None,
                     tipAngle=30, baseAngle=20, tailLen=10, tailWidth=2)
```

- 开仓箭头保持空心（`pen='w'`），平仓箭头实心（`pen=None`）
- 统一使用 `tipAngle=30, baseAngle=20`，不再使用 `headLen/headWidth`

### 4c. 子图 Y 轴锁定（全量数据 + 10% 留白 + 禁用自动缩放）

在 `plot_sub` 方法中，使用全量数据计算范围并锁定：

```python
def plot_sub(self, xmin=0, xmax=-1):
    """重画持仓量子图"""
    if self.initCompleted:
        all_vals = []
        for indicator in list(self.sub_indicator_data.keys()):
            if indicator in self.sub_indicator_plots:
                data = self.sub_indicator_data[indicator]
                self.sub_indicator_plots[indicator].setData(data,
                                                            pen=self.sub_indicator_colors[indicator][0],
                                                            name=indicator)
                all_vals.extend(data)
        # 使用全量数据固定子图Y轴范围，10%留白
        if all_vals and len(all_vals) > 0:
            y_min = min(all_vals)
            y_max = max(all_vals)
            if y_max > y_min:
                pad = (y_max - y_min) * 0.1
                vb = self.pi_sub.getViewBox()
                vb.enableAutoRange(axis=pg.ViewBox.YAxis, enable=False)
                vb.setYRange(y_min - pad, y_max + pad, padding=0)
```

关键点：
- 使用**全量数据**（不是可见部分）计算范围，避免滚动时 Y 轴跳动
- `enableAutoRange(enable=False)` 彻底禁用自动缩放
- `setYRange` 配合 `padding=0` 精确锁定范围

### 4d. 子图初始化禁用自动缩放

在子图 ViewBox 初始化处：

```python
if self.display_sub:
    view = self.pi_sub.getViewBox()
    view.enableAutoRange(axis=pg.ViewBox.YAxis, enable=False)
```

不再设置 `setRange(yRange=(0, 100))`，改为禁用自动缩放，让 `plot_sub` 接管 Y 轴控制。

### 4e. 子图高度

在 `init_plot_sub` 中设置子图高度范围：

```python
self.lay_KL.nextRow()
self.lay_KL.addItem(self.pi_sub)
# 子图固定高度范围，保证指标可见
self.pi_sub.setMinimumHeight(120)
self.pi_sub.setMaximumHeight(250)
```

## 步骤 5：识别可用指标

读取 D1/H1 CSV 的第一行，排除标准列（datetime, open, high, low, close, turnover, volume, open_interest, net, NetCurve），其余为指标列。

## 步骤 6：创建 ui_display_klines.py

1. 模板参考：`test/strategy_TQA_V2_3_MeanRevertPulse/ui_display_klines.py`
2. 复制到日志目录所在位置，修改路径为日志目录中的实际文件
3. `main_indicators` 放通道类指标（H10, L10, H20, L20, LongMa, BollMid, BollUpper, BollLower, NetCurve）
4. `sub_indicators` 放副图指标（ATR 等）
5. `trade_include_symbols` 从 trade CSV 中提取实际品种
6. 如有资金曲线，数据文件指向 `*_with_capital.csv`

```python
kline_settings = {
    "D1_StrategyName":
        {
            "data_file": os.path.join(log_dir, "Strategy_D1_with_capital.csv"),
            "main_indicators": [
                'BollUpper', 'BollMid', 'BollLower',
                'H10', 'L10', 'H20', 'L20', 'LongMa',
                'NetCurve'
            ],
            "sub_indicators": ['ATR'],
            "trade_file": os.path.join(log_dir, "Strategy_trade.csv"),
            "dist_file": os.path.join(log_dir, "Strategy_dist.csv"),
            "dist_include_list": ["buy", "sell", "short", "cover"],
            "trade_include_symbols": ["SM88"]
        }
}
display_multi_grid(kline_settings)
```

## 步骤 7：执行并检查

```bash
python3 <复制后的ui_display_klines.py路径>
```

如果有报错，根据错误信息修复后重试，直到无报错。

## 常见问题

- **文件路径**：注意相对路径，ui_display_klines.py 所在目录和 log 目录的关系
- **列名**：必须与 CSV 实际列名一致
- **品种名**：trade CSV 中 symbol 列的值
- **资金曲线不显示**：检查 NetCurve 量级是否与价格对齐（应在价格范围 50%-150% 内）
- **资金曲线太平**：增大放大器（50000 可根据策略盈亏幅度调整）
- **子图指标被裁剪**：确认 kline.py 中 `enableAutoRange(enable=False)` 和 `setYRange` 代码存在
- **子图太小**：调整 `setMinimumHeight(120)` / `setMaximumHeight(250)` 数值
- **闪退**：不要使用 `self.lay_KL.ci.layout` 访问布局，该路径不存在；使用 `self.pi_sub.setMinimumHeight/setMaximumHeight` 控制高度

## 当前满意状态总结（2026-06-10）

| 特性 | 配置 |
|------|------|
| NetCurve 样式 | `#8B00FF` 紫色虚线，width=3 |
| 开仓箭头 | 空心，`tipAngle=30, baseAngle=20` |
| 平仓箭头 | 实心，`tipAngle=30, baseAngle=20` |
| 子图 Y 轴 | 全量数据范围 + 10% 留白，禁用自动缩放 |
| 子图高度 | minHeight=120, maxHeight=250 |
