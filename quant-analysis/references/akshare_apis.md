# AkShare 常用 API 速查手册

> AkShare 是免费开源的金融数据库，无需注册，`pip install akshare` 即可使用。
> 文档：https://akshare.akfamily.xyz/

---

## A股数据

### 个股日线数据
```python
import akshare as ak

# 日线行情（前复权）
df = ak.stock_zh_a_hist(
    symbol='600519',        # 股票代码（不带市场前缀）
    period='daily',         # 'daily' / 'weekly' / 'monthly'
    start_date='20220101',  # YYYYMMDD 格式
    end_date='20241231',
    adjust='qfq'            # 'qfq'前复权 / 'hfq'后复权 / ''不复权
)
# 返回列：日期、开盘、收盘、最高、最低、成交量、成交额、振幅、涨跌幅、涨跌额、换手率
```

### 沪深 300 / 上证指数等指数日线
```python
# 常用指数代码：
# 沪深300: '000300'  上证50: '000016'  中证500: '000905'
# 上证指数: 'sh000001'  深成指: 'sz399001'
df = ak.stock_zh_index_daily(symbol='sh000300')
# 返回列：date, open, close, high, low, volume
```

### 全 A 股股票列表
```python
stock_list = ak.stock_zh_a_spot_em()
# 返回：代码、名称、最新价、涨跌幅、市值等实时行情
```

### A股基本面数据（用于选股）
```python
# PE、PB、市值等估值数据
valuation = ak.stock_a_lg_indicator(symbol='600519')

# 财务指标（ROE、毛利率等）——需要指定季度
# 日期格式：'20231231'（年报）/ '20230930'（三季报）
fin = ak.stock_financial_abstract_ths(symbol='600519', indicator='按年度')

# 沪深300成分股
hs300 = ak.index_stock_cons(symbol='000300')
```

### 行业板块数据
```python
# 东方财富行业板块成分股
industry_stocks = ak.stock_board_industry_cons_em(symbol='银行')

# 所有行业列表
industries = ak.stock_board_industry_name_em()
```

---

## 港股数据

```python
# 港股日线（需要完整代码，如腾讯='00700'）
df = ak.stock_hk_hist(
    symbol='00700',
    period='daily',
    start_date='20220101',
    end_date='20241231',
    adjust='qfq'
)

# 港股列表
hk_stocks = ak.stock_hk_spot_em()
```

---

## 美股数据

```python
# 美股日线（代码大小写均可）
df = ak.stock_us_hist(
    symbol='AAPL',
    period='daily',
    start_date='20220101',
    end_date='20241231',
    adjust='qfq'
)

# 美股列表（纳斯达克 / 纽交所等）
us_stocks = ak.stock_us_spot_em()
```

---

## 宏观数据（辅助判断大盘环境）

```python
# 融资融券余额（市场情绪指标）
margin = ak.stock_margin_sz_summary_em()

# 北向资金流入（外资动向）
north_flow = ak.stock_em_hsgt_north_net_flow_in()

# 上交所成交量（市场活跃度）
vol = ak.stock_sse_deal_daily(date='20240101')
```

---

## 常见问题处理

| 问题 | 原因 | 解决方案 |
|---|---|---|
| 返回空 DataFrame | 股票已退市 / 日期无数据 | 检查代码和日期范围 |
| `KeyError: '日期'` | AkShare 版本差异 | 用 `df.columns` 查看实际列名 |
| 数据获取超时 | 网络问题 | 加 `retry` 逻辑或稍后重试 |
| 港股代码报错 | 需要补零 | '700' → '00700' |
| 美股实时价获取失败 | 非交易时间 | 用 `stock_us_hist` 获取昨日收盘 |

---

## Tushare Pro 接入（需要 token）

```python
import tushare as ts
ts.set_token('你的token')
pro = ts.pro_api()

# 日线数据（A股，复权方式在接口设置）
df = pro.daily(ts_code='600519.SH', start_date='20220101', end_date='20241231')

# 基本面：市值、PE、PB
daily_basic = pro.daily_basic(ts_code='600519.SH', trade_date='20241231')

# 财务数据
income = pro.income(ts_code='600519.SH', period='20231231')
balance = pro.balancesheet(ts_code='600519.SH', period='20231231')
cashflow = pro.cashflow(ts_code='600519.SH', period='20231231')
```
