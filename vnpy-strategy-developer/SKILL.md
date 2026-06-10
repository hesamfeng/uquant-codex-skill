---
name: vnpy-strategy-developer
description: |
  基于 vn.py 框架的量化交易策略开发工作流。当用户说"写策略"、"开发新策略"、"创建策略"、
  "帮我写个策略"、"基于XXX策略写一个新策略"、"策略开发"、"vnpy策略"时，**必须使用此 skill**。
  适用于所有基于 vnpy/app/cta_strategy_pro 框架的期货/股票策略开发任务。
---

# vn.py 量化策略开发工作流

## 学习阶段（必须执行）

1. **文件理解优先级**：
   - 先学习 `vnpy/component/cta_line_bar.py`（K线与指标核心文件）
   - 再学习 `vnpy/app/cta_strategy_pro/template.py`（策略模板）
   - 最后参考 `vnpy/app/cta_strategy_pro/strategies/strategy_TQA_V2.py`

2. **指标命名规范**：
   - 所有K线指标参数必须严格参考 `vnpy/component/cta_line_bar.py`
   - 搜索模式 `rg 'para_+指标名'` 获取完整参数命名列表
   - 严禁自行创造参数名

## 策略开发流程

### 阶段 1：策略逻辑确认
- 详细询问用户策略逻辑：入场条件、出场条件、风控规则
- **必须多次询问是否有补充逻辑**
- 只有用户明确回答"没有其他逻辑了"才能进入代码编写阶段
- 如果逻辑太复杂，拒绝并给出简化建议

### 阶段 2：策略文件创建
- `cp strategy_TQA_V2.py strategy_新策略名.py`
- 新文件名必须反映策略核心逻辑
- 复制完成后告知用户新文件路径

### 阶段 3：核心代码修改

**必须修改的函数**：
- `__init__()`：重点修改 ctalinebar 指标设置
- 回调函数 `on_bar_XXX()`：实现具体交易逻辑
- `export_klines()`：导出新策略所需K线指标

**严禁修改的函数（除非用户明确要求）**：
`on_bar`, `dingding`, `on_init`, `init_data`, `on_order`, `on_trade`, `on_tick`, `check_change_Month`, `tns_buy`, `tns_short`, `tns_cover`, `tns_sell`, `tns_get_grid`

### 阶段 4：代码修改规范
- 每次只修改一个函数，渐进式调整
- 每次修改最多不超过 100 行
- 先完整复制 V2 策略，再逐个函数修改
- 严禁删除未明确提及的函数

### 阶段 5：回测验证
- 代码修改完成后，**使用 `backtest-runner` skill** 配置并执行回测
- `backtest-runner` 会自动创建 test 目录、复制模板、配置回测参数、运行回测并报告结果

## 策略思维规范
- **价格中立原则**：不能对标的物价格有任何前提假设
- **数据驱动**：所有决策基于历史数据和规则
- 遵循最小改动原则，大幅改动需用户同意

## 交互规范
- 始终用中文交流
- 每个步骤清晰说明进展
- 不确定时主动询问用户
