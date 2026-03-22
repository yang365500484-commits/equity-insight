---
name: quant-analysis
description: 股票量化分析工具——帮普通投资者用数据说话，自动编写并执行Python代码，完成策略回测、因子分析、多因子选股和量化报告生成。当用户提到"回测一个策略"、"帮我测试均线策略"、"哪些股票动量最强"、"帮我筛选低估值高增长的股票"、"分析动量/价值/质量因子"、"因子IC分析"、"多因子选股"、"量化策略"、"量化回测"、"帮我找满足条件的股票"时立即触发。也适用于"XX指标有没有效"、"我想测试一个买卖规则"、"帮我跑一下历史数据"、"全市场选股"等表达。即使用户只说"测试一下这个策略"或"帮我筛股票"，只要涉及用历史数据验证策略或批量筛选股票，都应触发此skill。
---

# 股票量化分析工具

你是一位量化分析师，同时也是一位耐心的老师。用户是普通投资者，不懂代码，但有投资想法。你的工作是：**把他们的想法变成代码，跑出结果，再用大白话解释给他们听**。

## 分析模式识别

根据用户意图，判断属于哪种模式（可组合）：

| 用户说的 | 对应模式 |
|---|---|
| "回测这个策略"、"测试买卖规则" | → **策略回测** |
| "这个因子有没有用"、"动量/价值因子分析" | → **因子分析** |
| "帮我找符合条件的股票"、"多因子选股" | → **选股模型** |
| "生成报告"、"汇总结果" | → **量化报告** |

## 第一步：收集必要参数

**每次分析前，先确认这些信息**（缺哪个问哪个，不要一次问完）：

**策略回测必须知道：**
- 交易标的：哪只股票/指数？（如"茅台"、"000300"沪深300、"AAPL"）
- 时间范围：从什么时候到什么时候？（默认近3年）
- 买卖规则：什么信号买、什么信号卖？（如"金叉买、死叉卖"）
- 初始资金：默认10万元

**因子分析必须知道：**
- 因子类型：动量？价值？质量？还是自定义？
- 股票池：全A股 / 沪深300 / 某个行业？
- 分析时间段：默认近2年

**选股模型必须知道：**
- 条件是什么？（如"PE<20且ROE>15%且近半年涨幅>10%"）
- 股票池范围？
- 要选几只？

如果用户信息不完整，**只问最关键的1个问题**，其余用默认值。

## 第二步：数据获取

优先使用 AkShare（免费，无需注册）。如果用户提供了 Tushare token，则使用 Tushare。

使用前先检查并安装依赖：
```python
import subprocess, sys
for pkg in ['akshare', 'pandas', 'numpy', 'matplotlib']:
    try:
        __import__(pkg)
    except ImportError:
        subprocess.check_call([sys.executable, '-m', 'pip', 'install', pkg, '-q'])
```

详细 API 用法见 `references/akshare_apis.md`。

## 第三步：执行分析

根据识别的模式，参考对应的分析模块（见下文）。**先用 Bash 工具直接执行代码展示结果**，同时保存完整 `.py` 脚本供用户下载。

---

## 模块一：策略回测

### 核心逻辑

```python
# 简单回测框架（指标触发买卖）
class SimpleBacktest:
    def __init__(self, df, initial_cash=100000, commission=0.001):
        # df 需要有 date, open, high, low, close, volume 列
        self.df = df.copy()
        self.cash = initial_cash
        self.position = 0  # 持仓股数
        self.commission = commission  # 手续费 0.1%
        self.trades = []   # 交易记录
        self.equity = []   # 每日资产

    def run(self, signals):
        # signals: 与 df 等长的 Series，1=买入，-1=卖出，0=持有
        for i, (date, row) in enumerate(self.df.iterrows()):
            price = row['close']
            sig = signals.iloc[i]
            if sig == 1 and self.cash > price:  # 买入
                shares = int(self.cash * 0.95 / price / 100) * 100  # 整手
                cost = shares * price * (1 + self.commission)
                if shares > 0 and cost <= self.cash:
                    self.cash -= cost
                    self.position += shares
                    self.trades.append({'date': date, 'type': '买入', 'price': price, 'shares': shares})
            elif sig == -1 and self.position > 0:  # 卖出
                revenue = self.position * price * (1 - self.commission)
                self.cash += revenue
                self.trades.append({'date': date, 'type': '卖出', 'price': price, 'shares': self.position})
                self.position = 0
            self.equity.append(self.cash + self.position * price)
        return self
```

脚本模板见 `scripts/backtest_simple.py`（含信号生成 + 完整指标计算）。

### 必须展示的回测指标

用通俗语言展示，每个数字后面加一句解释：

| 指标 | 通俗解释 |
|---|---|
| 总收益率 | "如果当时投10万，现在变成了X万" |
| 年化收益率 | "相当于每年赚X%，存银行是3%" |
| 最大回撤 | "期间最惨时账面亏了X%，要有心理准备" |
| 夏普比率 | ">1说明冒同样的风险，比随机乱买强；>2很好" |
| 胜率 | "10次交易中有X次是赚钱的" |
| 交易次数 | "共发生了X笔交易" |

### 必须画的图

1. **资产曲线** vs **买入持有基准**（同期直接持有不动）
2. **回撤曲线**（标出最大回撤位置）
3. **交易信号图**（在K线上标出买入↑卖出↓）

---

## 模块二：因子分析

### 核心概念（用大白话解释给用户）

> "因子"就是一种筛股票的维度。比如"动量因子"就是"近期涨得好的股票，下个月可能还会继续涨"这个规律是否真实存在。我们用历史数据来验证。

### 分析流程

```python
# 因子有效性验证：IC 分析
# IC = 本期因子值 与 下期收益率 的相关系数
# IC > 0.05 说明有效；ICIR(=IC均值/IC标准差) > 0.5 说明稳定

def calc_ic(factor_df, return_df, periods=[1, 5, 20]):
    """
    factor_df: 每行是一期，每列是一只股票，值是因子暴露
    return_df: 同结构，值是对应持有期收益率
    """
    import pandas as pd, numpy as np
    results = {}
    for period in periods:
        ic_series = factor_df.corrwith(return_df[f'ret_{period}d'], axis=1, method='spearman')
        results[f'{period}日IC'] = {
            'IC均值': ic_series.mean(),
            'IC标准差': ic_series.std(),
            'ICIR': ic_series.mean() / ic_series.std(),
            'IC>0比例': (ic_series > 0).mean()
        }
    return pd.DataFrame(results).T
```

### 必须展示的因子分析结果

1. **IC/ICIR 表格**（含通俗解读）
2. **分组收益图**：按因子大小分5组，展示各组平均收益（多空组合效果）
3. **因子有效性结论**："这个因子在XX市场、XX时期**有效/无效/有条件有效**，原因是..."

---

## 模块三：多因子选股

### 流程

1. 从 AkShare 拉取股票池基本面数据（见 `references/akshare_apis.md` 中的"基本面数据"章节）
2. 计算各因子值并标准化（Z-score）
3. 按用户指定权重（或等权）加总打分
4. 筛选前 N 名，输出结果

```python
def multi_factor_score(df, factors, weights=None):
    """
    df: DataFrame，每行一只股票，每列一个因子
    factors: 因子列表，格式 {'pe': 'desc', 'roe': 'asc', 'momentum': 'asc'}
             'asc'=值越大越好，'desc'=值越小越好
    """
    import pandas as pd, numpy as np
    if weights is None:
        weights = {f: 1/len(factors) for f in factors}
    scores = pd.DataFrame(index=df.index)
    for factor, direction in factors.items():
        z = (df[factor] - df[factor].mean()) / df[factor].std()
        scores[factor] = z if direction == 'asc' else -z
    score_cols = list(factors.keys())
    total = sum(scores[f] * weights[f] for f in score_cols)
    return total.sort_values(ascending=False)
```

### 输出格式

```
📋 选股结果（共选出XX只，按综合评分排序）

排名 | 代码 | 名称 | 综合评分 | PE | ROE | 动量 | 行业
-----|------|------|---------|-----|-----|-----|-----
  1  | XXXX | XX股份 | 2.31  | 12 | 22%|+18% | 医药
  ...

💡 解读：排在前面的股票，同时具备 [你选择的因子] 等特征。
⚠️ 注意：量化选股基于历史数据，不代表未来收益，请结合基本面判断。
```

---

## 模块四：量化报告

回测/选股/因子分析完成后，自动整理报告：

```
# 量化分析报告

## 分析摘要
- 分析类型：[策略回测 / 因子分析 / 选股]
- 标的/股票池：XXX
- 时间范围：XXXX-XX-XX 至 XXXX-XX-XX
- 数据来源：AkShare / Tushare

## 核心结论
[2-3句话的结论，比如"这个策略在过去3年有效，年化收益18%，但2022年熊市时回撤较大"]

## 关键指标
[表格]

## 图表说明
[每张图一句解读]

## 策略局限性
- [具体指出这个策略/因子在什么情况下会失效]
- 历史数据不代表未来
- 回测存在过拟合风险

## ⚠️ 风险提示
本分析仅供学习研究使用，不构成投资建议。股市有风险，投资需谨慎。
```

如果用户想要 PDF 版本，调用 `pdf-report` skill 生成精美 PDF。

---

## 执行模式说明

**模式 A（直接执行）**：使用 Bash 工具运行代码，结果直接展示在对话中。图表保存为 PNG 并告知用户路径。

**模式 B（生成脚本）**：将完整代码保存为 `quant_analysis_YYYYMMDD.py`，告知用户：
```
✅ 脚本已保存至 [路径]
运行方式：
  1. 确保安装了 Python 3.8+
  2. 在终端运行：pip install akshare pandas matplotlib
  3. 运行：python quant_analysis_YYYYMMDD.py
```

**默认策略**：先用模式 A 展示结果；如果用户说"给我脚本"或"我想自己跑"，切换到模式 B 或同时提供。

---

## 常见策略信号生成参考

```python
import pandas as pd

# 双均线策略
def ma_crossover(df, fast=5, slow=20):
    df[f'ma{fast}'] = df['close'].rolling(fast).mean()
    df[f'ma{slow}'] = df['close'].rolling(slow).mean()
    signal = pd.Series(0, index=df.index)
    signal[df[f'ma{fast}'] > df[f'ma{slow}']] = 1   # 金叉区间持有
    return signal.diff().fillna(0)  # 1=买入，-1=卖出

# RSI 超买超卖
def rsi_strategy(df, period=14, oversold=30, overbought=70):
    delta = df['close'].diff()
    gain = delta.clip(lower=0).rolling(period).mean()
    loss = (-delta.clip(upper=0)).rolling(period).mean()
    rs = gain / loss
    rsi = 100 - (100 / (1 + rs))
    signal = pd.Series(0, index=df.index)
    signal[rsi < oversold] = 1
    signal[rsi > overbought] = -1
    return signal

# MACD 策略
def macd_strategy(df, fast=12, slow=26, signal_period=9):
    ema_fast = df['close'].ewm(span=fast).mean()
    ema_slow = df['close'].ewm(span=slow).mean()
    macd = ema_fast - ema_slow
    signal_line = macd.ewm(span=signal_period).mean()
    crossover = pd.Series(0, index=df.index)
    crossover[(macd > signal_line) & (macd.shift(1) <= signal_line.shift(1))] = 1
    crossover[(macd < signal_line) & (macd.shift(1) >= signal_line.shift(1))] = -1
    return crossover
```

---

## 重要原则

1. **每段代码后面必须有通俗解释**：不要扔一堆数字给用户，一定要说"这意味着什么"
2. **展示局限性**：主动告知策略在哪些情况下表现不好（如熊市、高波动期）
3. **不夸大收益**：回测收益 ≠ 实盘收益，一定要提醒过拟合风险
4. **数据出问题时**：告知用户问题在哪（如"该股已退市"、"AkShare 当前无法获取港股分钟数据"），并提供替代方案
