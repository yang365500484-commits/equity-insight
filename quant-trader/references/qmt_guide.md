# 国金证券 QMT（迅投量化平台）部署指南

## QMT 是什么？

国金证券提供的量化交易平台，客户端叫"迅投QMT"。
- 支持 Python 策略编写
- 可以直接在 A 股账户下单
- 提供实时行情数据（不需要额外数据源）
- 免费，开通国金证券账户即可申请

**申请方式**：致电国金证券客服或在 App 内申请"量化交易权限"

---

## 安装和环境配置

### 1. 下载 QMT
从国金证券官网下载 miniQMT（轻量版，适合个人用户）或完整版 QMT。

### 2. 安装 XtQuant Python 包
QMT 自带 Python 环境，也可以用系统 Python：
```bash
pip install xtquant
```

或者使用 QMT 内置的包管理器安装。

### 3. 找到账户 ID
登录 QMT 客户端 → 账户管理 → 查看"资金账号"，形如 `88888888`

### 4. 找到 userdata 路径
QMT 安装目录下有 `userdata_mini` 文件夹，路径通常是：
```
C:\国金证券QMT交易端\userdata_mini\
```
或
```
C:\迅投量化交易终端\userdata_mini\
```

---

## 如何运行 Claude 生成的策略代码

### 方式一：QMT 内置编辑器（推荐新手）
1. 打开 QMT 客户端
2. 点击"策略编辑"→ 新建策略
3. 粘贴 Claude 生成的代码
4. 修改参数区域（账户ID、股票代码等）
5. 先点"模拟运行"，确认无误后切"实盘运行"

### 方式二：外部 Python 脚本
1. 确保 QMT 客户端已登录且在运行
2. 在命令行运行：
```bash
python your_strategy.py
```
注意：外部脚本需要 QMT 客户端同时运行

---

## 设置定时任务（每天自动执行）

### 在 QMT 中设置
策略设置 → 运行时间 → 添加时间点：
- `09:31`：开盘后第一分钟
- `14:50`：收盘前10分钟（推荐，避免尾盘波动）
- `15:00`：收盘后复盘

### 用 Windows 任务计划（外部脚本方式）
```
控制面板 → 管理工具 → 任务计划程序 → 创建基本任务
触发器：每个工作日 14:50
操作：启动程序 → python → 你的策略文件路径
```

---

## XtQuant 核心 API 速查

### 行情数据
```python
from xtquant import xtdata

# 获取K线数据
bars = xtdata.get_market_data(
    field_list=['open', 'high', 'low', 'close', 'volume'],
    stock_list=['600519.SH'],
    period='1d',      # '1m'分钟 / '5m' / '1d'日线
    count=60          # 获取最近60根K线
)
close_series = bars['close']['600519.SH']

# 获取实时报价
tick = xtdata.get_full_tick(['600519.SH'])
last_price = tick['600519.SH']['lastPrice']
bid_price = tick['600519.SH']['bidPrice'][0]  # 买一价
ask_price = tick['600519.SH']['askPrice'][0]  # 卖一价

# 订阅实时行情（回调模式）
def on_tick(datas):
    for code, data in datas.items():
        print(f"{code}: {data['lastPrice']}")
xtdata.subscribe_quote('600519.SH', period='tick', callback=on_tick)
```

### 账户查询
```python
from xtquant.xttrader import XtQuantTrader
from xtquant.xttype import StockAccount

trader = XtQuantTrader('C:/qmt_path/userdata_mini', '88888888')
trader.start()
account = StockAccount('88888888')

# 查询资金
asset = trader.query_stock_asset(account)
print(f"可用资金: {asset.cash:.2f}")
print(f"总资产: {asset.total_asset:.2f}")

# 查询持仓
positions = trader.query_stock_positions(account)
for pos in positions:
    print(f"{pos.stock_code}: 持仓{pos.volume}股, 成本{pos.open_price:.2f}, 现价{pos.market_value/pos.volume:.2f}")

# 查询当日委托
orders = trader.query_stock_orders(account)
```

### 下单
```python
# 买入
# 参数：账户, 代码, 买卖方向(23=买/24=卖), 数量, 报价方式(11=限价), 价格, 备注, 策略名
trader.order_stock(
    account,
    '600519.SH',
    23,          # 23=买入
    100,         # 100股（1手）
    11,          # 11=限价单
    1750.00,     # 限价价格
    '均线策略买入',
    'my_strategy'
)

# 卖出
trader.order_stock(account, '600519.SH', 24, 100, 11, 1800.00, '止盈卖出', '')

# 撤单
orders = trader.query_stock_orders(account, cancelable_only=True)
for order in orders:
    trader.cancel_order_stock(account, order.order_id)
```

### 股票代码规则（A股）
```
上交所：代码 + .SH   例：600519.SH（贵州茅台）
深交所：代码 + .SZ   例：000001.SZ（平安银行）
```

---

## 模拟盘 vs 实盘切换

```python
# 模拟盘（推荐先测试）
account = StockAccount('88888888', 'STOCK')  # 查看QMT中模拟账户的账号

# 实盘
account = StockAccount('88888888')
```

**强烈建议**：所有新策略先在模拟盘运行至少1周，确认逻辑正确再切实盘。

---

## 常见报错处理

| 报错 | 原因 | 解决 |
|---|---|---|
| `连接失败` | QMT客户端未登录 | 先打开QMT客户端并登录 |
| `账户不存在` | 账户ID填错 | 在QMT账户管理里确认资金账号 |
| `下单失败: 资金不足` | 可用资金不够 | 减少买入数量或检查冻结资金 |
| `非交易时间` | 在非交易时段下单 | 正常，交易时间9:30-11:30, 13:00-15:00 |
| `get_market_data返回空` | 代码格式错误 | A股需要加.SH/.SZ后缀 |
| `xtquant模块找不到` | 未安装 | `pip install xtquant` 或用QMT内置Python |
