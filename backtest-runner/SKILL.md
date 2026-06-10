---
name: backtest-runner
description: |
  当用户说"帮我回测"、"跑回测"、"回测一下"、"运行回测"时需要运行回测脚本时，
  **必须使用此 skill**。自动识别策略名称、定位 test 目录、配置 run_One.py 并执行回测。
  回测完成后自动调用 show-k 显示 K 线图。
---

# 回测运行器

## 工作流程

### 1. 策略识别
从用户指令或上下文中提取策略类名（如 `Strategy_TQA_V2`）。

### 2. 目录检查
- 在 `test/` 下查找对应策略名的目录（如 `test/strategy_TQA_V2/`）
- 如果目录不存在，创建目录并从 `test/strategy_TQA_V2/` 复制模板文件：
  - `run_One.py`（单品种回测）

### 3. 配置回测脚本

修改 `run_One.py` 中的配置：

**类名**：将 `class_name` 设为正确的策略类名。

**回测时间**：
```python
test_setting['start_date'] = '20180101'
test_setting['end_date'] = datetime.now().strftime('%Y%m%d')
```

**品种选择**：修改 `SELECTED_SYMBOLS` 列表。

**滑点设置**：
```python
symbol_info['slippage'] = 0
symbol_info['commission_rate'] = 0.0001
```

**资金配置**：
```python
test_setting['init_capital'] = 1000000
test_setting['percent_limit'] = 100
```

### 4. 执行回测
```bash
cd test/strategy_XXX && python3 run_One.py
```

### 5. 报告结果
回测完成后，从终端输出中提取关键指标向用户报告：
- 期初/期末资金、平仓盈亏
- 总交易次数、胜率
- 最大回撤、Sharpe 比率

### 6. 显示 K 线（必须执行）
回测结果报告完成后，**必须使用 `show-k` skill** 显示 K 线图。将回测目录路径传递给 show-k。

## 注意事项
- 不修改用户未指定的配置项
- 使用真实数据，不伪造
- 路径参考：`/Users/hewenfeng/Desktop/vnpy-uquantorg-Future-251220`
