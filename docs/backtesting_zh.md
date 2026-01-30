# Backtesting（回测）
<!-- 
===================================================
回测是验证交易策略性能的核心功能
通过历史数据模拟交易，评估策略的盈利能力
===================================================
-->

This page explains how to validate your strategy performance by using Backtesting.
<!-- 本页解释如何使用回测来验证你的策略表现 -->

Backtesting requires historic data to be available.
To learn how to get data for the pairs and exchange you're interested in, head over to the [Data Downloading](data-download.md) section of the documentation.
<!-- 
回测需要历史数据支持。
要了解如何获取你感兴趣的交易对和交易所的数据，
请参阅文档的"数据下载"部分。
-->

Backtesting is also available in [webserver mode](freq-ui.md#backtesting), which allows you to run backtests via the web interface.
<!-- 回测也可以在网页服务器模式下进行，通过Web界面运行回测 -->

## Backtesting command reference（回测命令参考）

--8<-- "commands/backtesting.md"

## Test your strategy with Backtesting（使用回测测试你的策略）

Now you have good Entry and exit strategies and some historic data, you want to test it against
real data. This is what we call [backtesting](https://en.wikipedia.org/wiki/Backtesting).
<!-- 
现在你已经有了好的入场和出场策略以及一些历史数据，
你想用真实数据来测试它。这就是我们所说的"回测"。
-->

Backtesting will use the crypto-currencies (pairs) from your config file and load historical candle (OHLCV) data from `user_data/data/<exchange>` by default.
If no data is available for the exchange / pair / timeframe combination, backtesting will ask you to download them first using `freqtrade download-data`.
For details on downloading, please refer to the [Data Downloading](data-download.md) section in the documentation.
<!-- 
回测将使用配置文件中的加密货币（交易对），
默认从 user_data/data/<交易所> 加载历史K线（OHLCV）数据。
如果没有可用数据，回测会提示你先使用 freqtrade download-data 下载数据。
-->

The result of backtesting will confirm if your bot has better odds of making a profit than a loss.
<!-- 回测结果将确认你的机器人盈利的可能性是否大于亏损 -->

All profit calculations include fees, and freqtrade will use the exchange's default fees for the calculation.
<!-- 所有利润计算都包含手续费，freqtrade将使用交易所的默认手续费进行计算 -->

!!! Warning "Using dynamic pairlists for backtesting（使用动态交易对列表进行回测）"
    Using dynamic pairlists is possible (not all of the handlers are allowed to be used in backtest mode), however it relies on the current market conditions - which will not reflect the historic status of the pairlist.
    Also, when using pairlists other than StaticPairlist, reproducibility of backtesting-results cannot be guaranteed.
    Please read the [pairlists documentation](plugins.md#pairlists) for more information.
    <!-- 
    可以使用动态交易对列表（但并非所有处理程序都允许在回测模式下使用），
    但它依赖于当前市场条件 - 这不会反映交易对列表的历史状态。
    使用 StaticPairlist 以外的交易对列表时，无法保证回测结果的可重复性。
    -->

    To achieve reproducible results, best generate a pairlist via the [`test-pairlist`](utils.md#test-pairlist) command and use that as static pairlist.
    <!-- 要获得可重复的结果，最好通过 test-pairlist 命令生成交易对列表，并用作静态列表 -->

!!! Note
    By default, Freqtrade will export backtesting results to `user_data/backtest_results`.
    The exported trades can be used for [further analysis](#further-backtest-result-analysis) or can be used by the [plotting sub-command](plotting.md#plot-price-and-indicators) (`freqtrade plot-dataframe`) in the scripts directory.
    <!-- 
    默认情况下，Freqtrade 会将回测结果导出到 user_data/backtest_results。
    导出的交易记录可用于进一步分析或用于绘图子命令。
    -->


### Starting balance（起始余额）

Backtesting will require a starting balance, which can be provided as `--dry-run-wallet <balance>` or `--starting-balance <balance>` command line argument, or via `dry_run_wallet` configuration setting.
This amount must be higher than `stake_amount`, otherwise the bot will not be able to simulate any trade.
<!-- 
回测需要起始余额，可以通过以下方式提供：
- 命令行参数：--dry-run-wallet <余额> 或 --starting-balance <余额>
- 配置设置：dry_run_wallet
这个金额必须高于 stake_amount，否则机器人将无法模拟任何交易。
-->

### Dynamic stake amount（动态下注金额）

Backtesting supports [dynamic stake amount](configuration.md#dynamic-stake-amount) by configuring `stake_amount` as `"unlimited"`, which will split the starting balance into `max_open_trades` pieces.
Profits from early trades will result in subsequent higher stake amounts, resulting in compounding of profits over the backtesting period.
<!-- 
回测支持动态下注金额，将 stake_amount 配置为 "unlimited"，
这会将起始余额分成 max_open_trades 份。
早期交易的利润会导致后续更高的下注金额，从而在回测期间实现复利增长。
-->

### Example backtesting commands（回测命令示例）

<!-- ========== 基本回测命令 ========== -->
With 5 min candle (OHLCV) data (per default)
<!-- 使用5分钟K线数据（默认） -->

```bash
freqtrade backtesting --strategy AwesomeStrategy
# --strategy AwesomeStrategy：指定策略类名
# 策略文件位于 user_data/strategies 目录下
```

Where `--strategy AwesomeStrategy` / `-s AwesomeStrategy` refers to the class name of the strategy, which is within a python file in the `user_data/strategies` directory.

---

<!-- ========== 使用1分钟K线 ========== -->
With 1 min candle (OHLCV) data
<!-- 使用1分钟K线数据 -->

```bash
freqtrade backtesting --strategy AwesomeStrategy --timeframe 1m
# --timeframe 1m：使用1分钟K线周期
```

---

<!-- ========== 自定义起始资金 ========== -->
Providing a custom starting balance of 1000 (in stake currency)
<!-- 提供1000的自定义起始余额（以计价货币为单位） -->

```bash
freqtrade backtesting --strategy AwesomeStrategy --dry-run-wallet 1000
# --dry-run-wallet 1000：设置模拟钱包初始资金为1000
```

---

<!-- ========== 使用不同的历史数据目录 ========== -->
Using a different on-disk historical candle (OHLCV) data source
<!-- 使用不同的磁盘历史K线数据源 -->

Assume you downloaded the history data from the Binance exchange and kept it in the `user_data/data/binance-20180101` directory. 
You can then use this data for backtesting as follows:
<!-- 
假设你从币安交易所下载了历史数据并保存在 user_data/data/binance-20180101 目录。
你可以按以下方式使用这些数据进行回测：
-->

```bash
freqtrade backtesting --strategy AwesomeStrategy --datadir user_data/data/binance-20180101 
# --datadir：指定历史数据目录
```

---

<!-- ========== 比较多个策略 ========== -->
Comparing multiple Strategies
<!-- 比较多个策略 -->

```bash
freqtrade backtesting --strategy-list SampleStrategy1 AwesomeStrategy --timeframe 5m
# --strategy-list：同时回测多个策略进行比较
# SampleStrategy1 和 AwesomeStrategy 是策略类名
```

Where `SampleStrategy1` and `AwesomeStrategy` refer to class names of strategies.

---

<!-- ========== 禁止导出交易记录 ========== -->
Prevent exporting trades to file
<!-- 阻止将交易导出到文件 -->

```bash
freqtrade backtesting --strategy backtesting --export none --config config.json 
# --export none：不导出交易记录
# 只有在确定不需要分析或绘图时才使用此选项
```

Only use this if you're sure you'll not want to plot or analyze your results further.

---

<!-- ========== 自定义导出目录 ========== -->
Exporting trades to file specifying a custom directory
<!-- 将交易导出到指定的自定义目录 -->

```bash
freqtrade backtesting --strategy backtesting --export trades --backtest-directory=user_data/custom-backtest-results
# --backtest-directory：指定回测结果保存目录
```

---

Please also read about the [strategy startup period](strategy-customization.md#strategy-startup-period).
<!-- 请同时阅读"策略启动期"相关内容 -->

---

<!-- ========== 自定义手续费 ========== -->
Supplying custom fee value
<!-- 提供自定义手续费值 -->

Sometimes your account has certain fee rebates (fee reductions starting with a certain account size or monthly volume), which are not visible to ccxt.
To account for this in backtesting, you can use the `--fee` command line option to supply this value to backtesting.
This fee must be a ratio, and will be applied twice (once for trade entry, and once for trade exit).
<!-- 
有时你的账户有一定的手续费折扣（从特定账户规模或月交易量开始的费用减免），
这些对ccxt不可见。为了在回测中考虑这一点，可以使用 --fee 命令行选项。
这个费用必须是比率形式，将应用两次（入场一次，出场一次）。
-->

For example, if the commission fee per order is 0.1% (i.e., 0.001 written as ratio), then you would run backtesting as the following:
<!-- 例如，如果每笔订单的佣金费率是0.1%（即0.001），则按以下方式运行回测： -->

```bash
freqtrade backtesting --fee 0.001
# --fee 0.001：设置手续费为0.1%（买入和卖出各收取一次）
```

!!! Note
    Only supply this option (or the corresponding configuration parameter) if you want to experiment with different fee values. By default, Backtesting fetches the default fee from the exchange pair/market info.
    <!-- 只有在需要实验不同费用值时才使用此选项。默认情况下，回测从交易所获取默认费用。 -->

---

<!-- ========== 使用时间范围 ========== -->
Running backtest with smaller test-set by using timerange
<!-- 使用timerange运行较小测试集的回测 -->

Use the `--timerange` argument to change how much of the test-set you want to use.
<!-- 使用 --timerange 参数来控制使用多少测试数据 -->

For example, running backtesting with the `--timerange=20190501-` option will use all available data starting with May 1st, 2019 from your input data.
<!-- 例如，使用 --timerange=20190501- 选项将使用从2019年5月1日开始的所有可用数据 -->

```bash
freqtrade backtesting --timerange=20190501-
# --timerange=20190501-：从2019年5月1日开始
```

You can also specify particular date ranges.
<!-- 你也可以指定特定的日期范围 -->

The full timerange specification:
<!-- 完整的时间范围规范： -->

- Use data until 2018/01/31: `--timerange=-20180131`
  <!-- 使用截至2018/01/31的数据 -->
- Use data since 2018/01/31: `--timerange=20180131-`
  <!-- 使用从2018/01/31开始的数据 -->
- Use data since 2018/01/31 till 2018/03/01 : `--timerange=20180131-20180301`
  <!-- 使用从2018/01/31到2018/03/01的数据 -->
- Use data between POSIX / epoch timestamps 1527595200 1527618600: `--timerange=1527595200-1527618600`
  <!-- 使用POSIX时间戳之间的数据 -->

## Understand the backtesting result（理解回测结果）

The most important in the backtesting is to understand the result.
<!-- 回测中最重要的是理解结果 -->

A backtesting result will look like that:
<!-- 回测结果如下所示： -->

```
                                                 BACKTESTING REPORT（回测报告）                                                  
┏━━━━━━━━━━━━━━━┳━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┓
┃          Pair ┃ Trades ┃ Avg Profit % ┃ Tot Profit USDT ┃ Tot Profit % ┃    Avg Duration ┃  Win  Draw  Loss  Win% ┃
┃    （交易对）  ┃(交易数)┃  (平均利润%) ┃  (总利润USDT)   ┃  (总利润%)   ┃   (平均持仓时长) ┃  (胜 平 负 胜率)        ┃
┡━━━━━━━━━━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━┩
│ LTC/USDT:USDT │     16 │          1.0 │          56.176 │         5.62 │        16:16:00 │   16     0     0   100 │
│ ETC/USDT:USDT │     12 │         0.72 │          30.936 │         3.09 │         9:55:00 │   11     0     1  91.7 │
│ ETH/USDT:USDT │      8 │         0.66 │          17.864 │         1.79 │ 1 day, 13:55:00 │    7     0     1  87.5 │
│ XLM/USDT:USDT │     10 │         0.31 │          11.054 │         1.11 │        12:08:00 │    9     0     1  90.0 │
│ BTC/USDT:USDT │      8 │         0.21 │           7.289 │         0.73 │ 3 days, 1:24:00 │    6     0     2  75.0 │
│ XRP/USDT:USDT │      9 │        -0.14 │          -7.261 │        -0.73 │        21:18:00 │    8     0     1  88.9 │
│ DOT/USDT:USDT │      6 │         -0.4 │          -9.187 │        -0.92 │         5:35:00 │    4     0     2  66.7 │
│ ADA/USDT:USDT │      8 │        -1.76 │         -52.098 │        -5.21 │        11:38:00 │    6     0     2  75.0 │
│         TOTAL │     77 │         0.22 │          54.774 │         5.48 │        22:12:00 │   67     0    10  87.0 │
│        (合计) │        │              │                 │              │                 │                        │
└───────────────┴────────┴──────────────┴─────────────────┴──────────────┴─────────────────┴────────────────────────┘

<!-- 
=== 表格字段解释 ===
Pair: 交易对名称
Trades: 该交易对的交易次数
Avg Profit %: 每笔交易的平均利润百分比
Tot Profit USDT: 该交易对总利润（以USDT计）
Tot Profit %: 相对于起始资金的总利润百分比
Avg Duration: 平均持仓时间
Win/Draw/Loss: 盈利/平局/亏损的交易次数
Win%: 胜率
-->

                                               LEFT OPEN TRADES REPORT（未平仓交易报告）                                                
┏━━━━━━━━━━━━━━━┳━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┓
┃          Pair ┃ Trades ┃ Avg Profit % ┃ Tot Profit USDT ┃ Tot Profit % ┃     Avg Duration ┃  Win  Draw  Loss  Win% ┃
┡━━━━━━━━━━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━┩
│ BTC/USDT:USDT │      1 │        -4.14 │          -9.930 │        -0.99 │ 17 days, 8:00:00 │    0     0     1     0 │
│ ETC/USDT:USDT │      1 │        -4.24 │         -15.365 │        -1.54 │         10:40:00 │    0     0     1     0 │
│ DOT/USDT:USDT │      1 │        -5.29 │         -19.125 │        -1.91 │         11:30:00 │    0     0     1     0 │
│         TOTAL │      3 │        -4.56 │         -44.420 │        -4.44 │  6 days, 2:03:00 │    0     0     3     0 │
└───────────────┴────────┴──────────────┴─────────────────┴──────────────┴──────────────────┴────────────────────────┘
<!-- 
这些是回测结束时仍然持仓的交易
系统会强制平仓(force_exit)以展示完整的回测情况
-->

                                                ENTER TAG STATS（入场标签统计）                                                
┏━━━━━━━━━━━┳━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ Enter Tag ┃ Entries ┃ Avg Profit % ┃ Tot Profit USDT ┃ Tot Profit % ┃ Avg Duration ┃  Win  Draw  Loss  Win% ┃
┃ (入场标签)┃ (入场数)┃              │                 │              │              │                        ┃
┡━━━━━━━━━━━╇━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━┩
│     OTHER │      77 │         0.22 │          54.774 │         5.48 │     22:12:00 │   67     0    10  87.0 │
│     TOTAL │      77 │         0.22 │          54.774 │         5.48 │     22:12:00 │   67     0    10  87.0 │
└───────────┴─────────┴──────────────┴─────────────────┴──────────────┴──────────────┴────────────────────────┘
<!-- 按入场标签(enter_tag)分类的统计，帮助分析不同入场信号的表现 -->

                                                EXIT REASON STATS（退出原因统计）                                                 
┏━━━━━━━━━━━━━┳━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ Exit Reason ┃ Exits ┃ Avg Profit % ┃ Tot Profit USDT ┃ Tot Profit % ┃    Avg Duration ┃  Win  Draw  Loss  Win% ┃
┃ (退出原因)  ┃(退出数)│              │                 │              │                 │                        ┃
┡━━━━━━━━━━━━━╇━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━┩
│         roi │    67 │         1.05 │         242.179 │        24.22 │        15:49:00 │   67     0     0   100 │
│ exit_signal │     4 │        -2.23 │         -31.217 │        -3.12 │  1 day, 8:38:00 │    0     0     4     0 │
│  force_exit │     3 │        -4.56 │         -44.420 │        -4.44 │ 6 days, 2:03:00 │    0     0     3     0 │
│   stop_loss │     3 │       -10.14 │        -111.768 │       -11.18 │  1 day, 3:05:00 │    0     0     3     0 │
│       TOTAL │    77 │         0.22 │          54.774 │         5.48 │        22:12:00 │   67     0    10  87.0 │
└─────────────┴───────┴──────────────┴─────────────────┴──────────────┴─────────────────┴────────────────────────┘
<!-- 
退出原因说明：
- roi: 达到ROI止盈目标
- exit_signal: 退出信号触发
- force_exit: 强制退出（回测结束）
- stop_loss: 止损触发
-->
```

<!-- 
===============================================================
                    SUMMARY METRICS 汇总指标详解
===============================================================
-->

```
                          SUMMARY METRICS（汇总指标）                          
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ Metric（指标）                ┃ Value（值）                      ┃
┡━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┩
│ Backtesting from              │ 2025-07-01 00:00:00             │  ← 回测开始时间
│ Backtesting to                │ 2025-08-01 00:00:00             │  ← 回测结束时间
│ Trading Mode                  │ Isolated Futures                │  ← 交易模式（隔离保证金期货）
│ Max open trades               │ 3                               │  ← 最大同时持仓数
│                               │                                 │
│ Total/Daily Avg Trades        │ 77 / 2.48                       │  ← 总交易数/日均交易数
│ Starting balance              │ 1000 USDT                       │  ← 起始余额
│ Final balance                 │ 1054.774 USDT                   │  ← 最终余额
│ Absolute profit               │ 54.774 USDT                     │  ← 绝对利润
│ Total profit %                │ 5.48%                           │  ← 总利润百分比
│ CAGR %                        │ 87.36%                          │  ← 年复合增长率
│ Sortino                       │ 2.48                            │  ← 索提诺比率（越高越好，衡量下行风险调整后收益）
│ Sharpe                        │ 3.75                            │  ← 夏普比率（越高越好，衡量风险调整后收益）
│ Calmar                        │ 40.99                           │  ← 卡玛比率（年化收益/最大回撤）
│ SQN                           │ 0.69                            │  ← 系统质量数（Van Tharp提出）
│ Profit factor                 │ 1.29                            │  ← 盈亏比（盈利交易总额/亏损交易总额）
│ Expectancy (Ratio)            │ 0.71 (0.04)                     │  ← 期望值（每笔交易的平均预期收益）
│ Avg. daily profit             │ 1.767 USDT                      │  ← 日均利润
│ Avg. stake amount             │ 345.016 USDT                    │  ← 平均下注金额
│ Total trade volume            │ 53316.954 USDT                  │  ← 总交易量
│                               │                                 │
│ Long / Short trades           │ 67 / 10                         │  ← 做多/做空交易数
│ Long / Short profit %         │ 8.94% / -3.47%                  │  ← 做多/做空利润百分比
│ Long / Short profit USDT      │ 89.425 / -34.651                │  ← 做多/做空利润金额
│                               │                                 │
│ Best Pair                     │ LTC/USDT:USDT 5.62%             │  ← 表现最好的交易对
│ Worst Pair                    │ ADA/USDT:USDT -5.21%            │  ← 表现最差的交易对
│ Best trade                    │ ETC/USDT:USDT 2.00%             │  ← 最佳单笔交易
│ Worst trade                   │ ADA/USDT:USDT -10.17%           │  ← 最差单笔交易
│ Best day                      │ 26.91 USDT                      │  ← 最佳单日收益
│ Worst day                     │ -47.741 USDT                    │  ← 最差单日亏损
│ Days win/draw/lose            │ 20 / 6 / 5                      │  ← 盈利/平局/亏损天数
│ Max Consecutive Wins / Loss   │ 36 / 3                          │  ← 最大连胜/连亏次数
│ Rejected Entry signals        │ 258                             │  ← 被拒绝的入场信号（因达到max_open_trades）
│ Entry/Exit Timeouts           │ 0 / 0                           │  ← 入场/出场超时次数
│                               │                                 │
│ Min balance                   │ 1003.168 USDT                   │  ← 最低余额
│ Max balance                   │ 1149.421 USDT                   │  ← 最高余额
│ Max % of account underwater   │ 8.23%                           │  ← 账户最大水下百分比
│ Absolute drawdown             │ 94.647 USDT (8.23%)             │  ← 绝对回撤（金额和百分比）
│ Drawdown duration             │ 9 days 08:50:00                 │  ← 最大回撤持续时间
│ Drawdown start                │ 2025-07-22 15:10:00             │  ← 回撤开始时间
│ Drawdown end                  │ 2025-08-01 00:00:00             │  ← 回撤结束时间
│ Market change                 │ 30.51%                          │  ← 市场变化（所有交易对的平均涨跌幅）
└───────────────────────────────┴─────────────────────────────────┘
```

### Backtesting report table（回测报告表）

The first table contains all trades the bot made, including "left open trades".
<!-- 第一个表格包含机器人进行的所有交易，包括"未平仓交易" -->

The last line will give you the overall performance of your strategy,
here:
<!-- 最后一行给出策略的整体表现： -->

```
│         TOTAL │     77 │         0.22 │          54.774 │         5.48 │        22:12:00 │   67     0    10  87.0 │
```

The bot has made `77` trades for an average duration of `22:12:00`, with a performance of `5.48%` (profit), that means it has earned a total of `54.774 USDT` starting with a capital of 1000 USDT.
<!-- 
机器人进行了77笔交易，平均持仓时间为22小时12分钟，
总收益率为5.48%，即从1000 USDT的本金赚取了54.774 USDT。
-->

The column `Avg Profit %` shows the average profit for all trades made.
The column `Tot Profit %` shows instead the total profit % in relation to the starting balance.
<!-- 
Avg Profit %：显示所有交易的平均利润百分比
Tot Profit %：显示相对于起始余额的总利润百分比
-->

In the above results, we have a starting balance of 1000 USDT and an absolute profit of 54.774 USDT - so the `Tot Profit %` will be `(54.774 / 1000) * 100 ~= 5.48%`.
<!-- 在上述结果中，起始余额1000 USDT，绝对利润54.774 USDT，所以 Tot Profit % = (54.774/1000)*100 ≈ 5.48% -->

Your strategy performance is influenced by your entry strategy, your exit strategy, and also by the `minimal_roi` and `stop_loss` you have set.
<!-- 你的策略表现受入场策略、出场策略以及设置的 minimal_roi 和 stop_loss 影响 -->

For example, if your `minimal_roi` is only `"0":  0.01` you cannot expect the bot to make more profit than 1% (because it will exit every time a trade reaches 1%).
<!-- 例如，如果 minimal_roi 只设为 "0": 0.01，你不能期望机器人获得超过1%的利润（因为每次达到1%就会退出） -->

```json
"minimal_roi": {
    "0":  0.01
},
```

On the other hand, if you set a too high `minimal_roi` like `"0":  0.55`
(55%), there is almost no chance that the bot will ever reach this profit.
<!-- 另一方面，如果设置过高的 minimal_roi 如 "0": 0.55（55%），机器人几乎不可能达到这个利润目标 -->

### Left open trades table（未平仓交易表）

The second table contains all trades the bot had to `force_exit` at the end of the backtesting period to present you the full picture.
This is necessary to simulate realistic behavior, since the backtest period has to end at some point, while realistically, you could leave the bot running forever.
<!-- 
第二个表格包含回测结束时机器人必须强制平仓的所有交易。
这是为了模拟真实行为而必要的，因为回测期间必须在某个时间点结束，
而实际上，你可以让机器人一直运行。
-->

### Enter tag stats table（入场标签统计表）

The third table provides a breakdown of trades by their entry tags (e.g., `enter_long`, `enter_short`), showing the number of entries, average profit percentage, total profit in the stake currency, total profit percentage, average duration, and the number of wins, draws, and losses for each tag.
<!-- 
第三个表格按入场标签分类显示交易情况，
包括：入场次数、平均利润百分比、总利润、平均持仓时间、胜平负统计。
-->

### Exit reason stats table（退出原因统计表）

The fourth table contains a recap of exit reasons (e.g., `exit_signal`, `roi`, `stop_loss`, `force_exit`). This table can tell you which area needs additional work (e.g., if many `exit_signal` trades are losses, you should work on improving the exit signal or consider disabling it).
<!-- 
第四个表格是退出原因汇总（如 exit_signal、roi、stop_loss、force_exit）。
这个表格可以告诉你哪个方面需要改进。
例如，如果很多 exit_signal 交易是亏损的，你应该改进退出信号或考虑禁用它。
-->

### Summary metrics（汇总指标详解）

<!-- 
==================================================================
                    关键指标解释
==================================================================
-->

The last element of the backtest report is the summary metrics table.
It contains key metrics about the performance of your strategy on backtesting data.
<!-- 回测报告的最后一部分是汇总指标表，包含策略在回测数据上表现的关键指标 -->

- `Backtesting from` / `Backtesting to`: Backtesting range (usually defined with the `--timerange` option).
  <!-- 回测时间范围（通常通过 --timerange 选项定义） -->
  
- `Trading Mode`: Spot or Futures trading.
  <!-- 交易模式：现货或期货 -->
  
- `Max open trades`: Setting of `max_open_trades` (or `--max-open-trades`) - or number of pairs in the pairlist (whatever is lower).
  <!-- 最大同时持仓数：max_open_trades 设置值或交易对数量（取较小值） -->
  
- `Total/Daily Avg Trades`: Identical to the total trades of the backtest output table / Total trades divided by the backtesting duration in days (this will give you information about how many trades to expect from the strategy).
  <!-- 总交易数/日均交易数：可以预估策略的交易频率 -->
  
- `Starting balance`: Start balance - as given by dry-run-wallet (config or command line).
  <!-- 起始余额：由 dry-run-wallet 配置或命令行参数指定 -->
  
- `Final balance`: Final balance - starting balance + absolute profit.
  <!-- 最终余额 = 起始余额 + 绝对利润 -->
  
- `Absolute profit`: Profit made in stake currency.
  <!-- 绝对利润：以计价货币计算的利润 -->
  
- `Total profit %`: Total profit. Calculated as `(End capital − Starting capital) / Starting capital`.
  <!-- 总利润百分比 = (最终资金 - 起始资金) / 起始资金 -->
  
- `CAGR %`: Compound annual growth rate.
  <!-- 年复合增长率：假设这个收益率持续一年会是多少 -->
  
- `Sortino`: Annualized Sortino ratio.
  <!-- 年化索提诺比率：只考虑下行波动的风险调整收益指标，越高越好 -->
  
- `Sharpe`: Annualized Sharpe ratio.
  <!-- 年化夏普比率：风险调整后的收益指标，越高越好。>1好，>2很好，>3优秀 -->
  
- `Calmar`: Annualized Calmar ratio.
  <!-- 年化卡玛比率 = 年化收益率 / 最大回撤，越高越好 -->
  
- `SQN`: System Quality Number (SQN) - by Van Tharp.
  <!-- 系统质量数（由Van Tharp提出）：衡量交易系统质量。>2好，>3优秀，>5超级 -->
  
- `Profit factor`: Sum of the profits of all winning trades divided by the sum of the losses of all losing trades.
  <!-- 盈亏比 = 盈利交易总额 / 亏损交易总额。>1盈利，>1.5好，>2优秀 -->
  
- `Expectancy (Ratio)`: Expectancy ratio, which is the average profit or loss per trade. A negative expectancy ratio means that your strategy is not profitable.
  <!-- 期望值：每笔交易的平均预期收益。负值表示策略不盈利 -->
  
- `Avg. daily profit`: Average profit per day, calculated as `(Total Profit / Backtest Days)`.
  <!-- 日均利润 = 总利润 / 回测天数 -->
  
- `Avg. stake amount`: Average stake amount, either `stake_amount` or the average when using dynamic stake amount.
  <!-- 平均下注金额：使用动态下注时为平均值 -->
  
- `Total trade volume`: Volume generated on the exchange to reach the above profit.
  <!-- 总交易量：在交易所产生的总交易额 -->
  
- `Long / Short trades`: Split long/short trade counts.
  <!-- 做多/做空交易次数 -->
  
- `Best Pair` / `Worst Pair`: Best and worst performing pair.
  <!-- 表现最好/最差的交易对 -->
  
- `Best trade` / `Worst trade`: Biggest single winning trade and biggest single losing trade.
  <!-- 最佳/最差单笔交易 -->
  
- `Best day` / `Worst day`: Best and worst day based on daily profit.
  <!-- 最佳/最差单日收益 -->
  
- `Days win/draw/lose`: Winning / Losing days (draws are usually days without closed trades).
  <!-- 盈利/平局/亏损天数（平局通常是没有平仓交易的日子） -->
  
- `Max Consecutive Wins / Loss`: Maximum consecutive wins/losses in a row.
  <!-- 最大连胜/连亏次数 -->
  
- `Rejected Entry signals`: Trade entry signals that could not be acted upon due to `max_open_trades` being reached.
  <!-- 被拒绝的入场信号：因达到 max_open_trades 限制而无法执行的入场信号 -->
  
- `Entry/Exit Timeouts`: Entry/exit orders which did not fill (only applicable if custom pricing is used).
  <!-- 入场/出场超时：未成交的订单（仅在使用自定义定价时适用） -->
  
- `Min balance` / `Max balance`: Lowest and Highest Wallet balance during the backtest period.
  <!-- 回测期间的最低/最高钱包余额 -->
  
- `Max % of account underwater`: Maximum percentage your account has decreased from the top since the simulation started.
  <!-- 账户最大水下百分比：从最高点下跌的最大幅度 -->
  
- `Absolute drawdown`: Maximum absolute drawdown experienced.
  <!-- 绝对回撤：经历的最大绝对回撤 -->
  
- `Drawdown duration`: Duration of the largest drawdown period.
  <!-- 最大回撤持续时间 -->
  
- `Drawdown start` / `Drawdown end`: Start and end datetime for the largest drawdown.
  <!-- 最大回撤的开始/结束时间 -->
  
- `Market change`: Change of the market during the backtest period.
  <!-- 市场变化：回测期间所有交易对从第一根到最后一根K线的平均涨跌幅 -->

### Daily / Weekly / Monthly / Yearly breakdown（日/周/月/年分解）

You can get an overview over daily, weekly, monthly, or yearly results by using the `--breakdown <>` switch.
<!-- 使用 --breakdown 开关可以获得按日/周/月/年分类的结果概览 -->

To visualize monthly and yearly breakdowns, you can use the following:
<!-- 要查看月度和年度分解，可以使用以下命令： -->

``` bash
freqtrade backtesting --strategy MyAwesomeStrategy --breakdown month year
# --breakdown month year：显示月度和年度收益分解
```

### Backtest result caching（回测结果缓存）

To save time, by default backtest will reuse a cached result from within the last day when the backtested strategy and config match that of a previous backtest. To force a new backtest despite existing result for an identical run specify `--cache none` parameter.
<!-- 
为节省时间，默认情况下，如果策略和配置与之前的回测匹配，
回测会重用最近一天内的缓存结果。
要强制进行新的回测，请指定 --cache none 参数。
-->

!!! Warning
    Caching is automatically disabled for open-ended timeranges (`--timerange 20210101-`), as freqtrade cannot ensure reliably that the underlying data didn't change.
    <!-- 对于开放式时间范围，缓存自动禁用，因为无法确保底层数据没有变化 -->

### Further backtest-result analysis（进一步分析回测结果）

To further analyze your backtest results, freqtrade will export the trades to file by default.
You can then load the trades to perform further analysis as shown in the [data analysis](strategy_analysis_example.md#load-backtest-results-to-pandas-dataframe) backtesting section.
<!-- 
Freqtrade 默认会将交易导出到文件。
你可以加载这些交易进行进一步分析。
-->

Also, you can use freqtrade in [webserver mode](freq-ui.md#backtesting) to visualize the backtest results in a web interface.
<!-- 你也可以使用网页服务器模式在Web界面中可视化回测结果 -->

### Backtest output file（回测输出文件）

The output file freqtrade produces is a zip file containing the following files:
<!-- Freqtrade 生成的输出文件是一个 zip 文件，包含以下文件： -->

- The backtest report in json format（JSON格式的回测报告）
- The market change data in feather format（feather格式的市场变化数据）
- A copy of the strategy file（策略文件副本）
- A copy of the strategy parameters (if a parameter file was used)（策略参数副本）
- A sanitized copy of the config file（经过处理的配置文件副本）

## Assumptions made by backtesting（回测做出的假设）

Since backtesting lacks some detailed information about what happens within a candle, it needs to take a few assumptions:
<!-- 由于回测缺少K线内部价格变动的详细信息，需要做一些假设： -->

<!-- 
==================================================================
                    关键假设列表（务必理解！）
==================================================================
-->

- Exchange [trading limits](#trading-limits-in-backtesting) are respected
  <!-- 遵守交易所的交易限制 -->
  
- Entries happen at open-price unless a custom price logic has been specified
  <!-- 除非指定了自定义价格逻辑，否则以开盘价入场 -->
  
- All orders are filled at the requested price (no slippage) as long as the price is within the candle's high/low range
  <!-- 只要价格在K线的高低范围内，所有订单都以请求价格成交（无滑点） -->
  
- Exit-signal exits happen at open-price of the consecutive candle
  <!-- 退出信号在下一根K线的开盘价执行 -->
  
- Exits free their trade slot for a new trade with a different pair
  <!-- 退出后释放交易槽位，可用于其他交易对 -->
  
- Exit-signal is favored over Stoploss, because exit-signals are assumed to trigger on candle's open
  <!-- 退出信号优先于止损，因为假设退出信号在开盘时触发 -->
  
- ROI:
  <!-- ROI止盈相关假设 -->
  - Exits are compared to high - but the ROI value is used (e.g. ROI = 2%, high=5% - so the exit will be at 2%)
    <!-- 与最高价比较，但使用ROI值退出（如ROI=2%，high=5%，则以2%退出） -->
  - Exits are never "below the candle", so a ROI of 2% may result in an exit at 2.4% if low was at 2.4% profit
    <!-- 退出不会"低于K线"，如果最低价在2.4%利润，则以2.4%退出 -->
    
- Stoploss exits happen exactly at stoploss price, even if low was lower
  <!-- 止损精确在止损价执行，即使最低价更低 -->
  
- Stoploss is evaluated before ROI within one candle. So you can often see more trades with the `stoploss` exit reason comparing to the results obtained with the same strategy in the Dry Run/Live Trade modes
  <!-- 在同一根K线内，止损优先于ROI评估。这可能导致回测中止损交易比实盘多 -->
  
- Low happens before high for stoploss, protecting capital first
  <!-- 对于止损，假设先触及最低价再触及最高价，优先保护资金 -->
  
- Trailing stoploss:
  <!-- 移动止损相关假设 -->
  - Trailing Stoploss is only adjusted if it's below the candle's low (otherwise it would be triggered)
    <!-- 只有当移动止损低于K线最低价时才调整 -->
  - High happens first - adjusting stoploss
    <!-- 先到达最高价 - 调整止损 -->
  - Low uses the adjusted stoploss (so exits with large high-low difference are backtested correctly)
    <!-- 然后用调整后的止损检查最低价 -->
    
- Evaluation sequence (if multiple signals happen on the same candle):
  <!-- 同一根K线内多个信号的评估顺序 -->
  - Exit-signal（退出信号）
  - Stoploss（止损）
  - ROI（止盈）
  - Trailing stoploss（移动止损）

Taking these assumptions, backtesting tries to mirror real trading as closely as possible. However, backtesting will **never** replace running a strategy in dry-run mode.
Also, keep in mind that past results don't guarantee future success.
<!-- 
基于这些假设，回测尽量接近真实交易。
但是，回测永远无法替代模拟盘测试。
另外，请记住：过去的结果不保证未来的成功。
-->

### Trading limits in backtesting（回测中的交易限制）

Exchanges have certain trading limits, like minimum (and maximum) base currency, or minimum/maximum stake (quote) currency.
<!-- 交易所有一定的交易限制，如最小/最大基础货币或计价货币 -->

Backtesting (as well as live and dry-run) does honor these limits, and will ensure that a stoploss can be placed below this value - so the value will be slightly higher than what the exchange specifies.
<!-- 回测（以及实盘和模拟盘）会遵守这些限制 -->

Freqtrade has however no information about historic limits.
This can lead to situations where trading-limits are inflated by using a historic price, resulting in minimum amounts > 50\$.
<!-- 
Freqtrade 没有历史限制信息。
这可能导致使用历史价格时交易限制被放大，导致最小金额超过50美元。
-->

## Improved backtest accuracy（提高回测精度）

One big limitation of backtesting is it's inability to know how prices moved intra-candle (was high before close, or vice-versa?).
So assuming you run backtesting with a 1h timeframe, there will be 4 prices for that candle (Open, High, Low, Close).
<!-- 
回测的一大限制是无法知道K线内部价格如何变动（先到最高价还是先到收盘价？）
假设使用1小时K线，每根K线只有4个价格（开高低收）。
-->

While backtesting does take some assumptions (read above) about this - this can never be perfect, and will always be biased in one way or the other.
To mitigate this, freqtrade can use a lower (faster) timeframe to simulate intra-candle movements.
<!-- 
回测做了一些假设，但永远无法完美。
为缓解这个问题，freqtrade 可以使用更小的周期来模拟K线内部波动。
-->

To utilize this, you can append `--timeframe-detail 5m` to your regular backtesting command.
<!-- 要使用此功能，在回测命令后添加 --timeframe-detail 5m -->

``` bash
freqtrade backtesting --strategy AwesomeStrategy --timeframe 1h --timeframe-detail 5m
# --timeframe 1h：主交易周期为1小时
# --timeframe-detail 5m：使用5分钟数据模拟K线内部波动
# 这将使回测更精确，但需要更多内存和时间
```

This will load 1h data (the main timeframe) as well as 5m data (detail timeframe) for the selected timerange.
<!-- 这将加载1小时数据（主周期）和5分钟数据（详细周期） -->

The strategy will be analyzed with the 1h timeframe.
Candles where activity may take place (there's an active signal, the pair is in a trade) are evaluated at the 5m timeframe.
<!-- 策略以1小时周期分析，但有活动的K线（有信号或持仓）会用5分钟周期评估 -->

This will allow for a more accurate simulation of intra-candle movements - and can lead to different results, especially on higher timeframes.
<!-- 这允许更精确地模拟K线内部波动，可能导致不同结果，尤其在较大周期时 -->

All callback functions (`custom_exit()`, `custom_stoploss()`, ... ) will be running for each 5m candle once the trade is opened (so 12 times in the above example of 1h timeframe, and 5m detailed timeframe).
<!-- 所有回调函数在开仓后会对每根5分钟K线运行（上例中每小时运行12次） -->

`--timeframe-detail` must be smaller than the original timeframe, otherwise backtesting will fail to start.
<!-- --timeframe-detail 必须小于主周期，否则回测会失败 -->

Obviously this will require more memory (5m data is bigger than 1h data), and will also impact runtime (depending on the amount of trades and trade durations).
Also, data must be available / downloaded already.
<!-- 这需要更多内存和运行时间，且数据必须已下载 -->

!!! Tip
    You can use this function as the last part of strategy development, to ensure your strategy is not exploiting one of the [backtesting assumptions](#assumptions-made-by-backtesting). Strategies that perform similarly well with this mode have a good chance to perform well in dry/live modes too (although only forward-testing (dry-mode) can really confirm a strategy).
    <!-- 
    可以在策略开发的最后阶段使用此功能，
    确保策略没有利用回测假设的漏洞。
    在此模式下表现同样好的策略，在实盘中表现良好的可能性更大。
    -->

## Backtesting multiple strategies（回测多个策略）

To compare multiple strategies, a list of Strategies can be provided to backtesting.
<!-- 要比较多个策略，可以提供策略列表 -->

This is limited to 1 timeframe value per run. However, data is only loaded once from disk so if you have multiple
strategies you'd like to compare, this will give a nice runtime boost.
<!-- 每次运行只能使用1个时间周期。但数据只加载一次，所以比较多个策略时运行更快 -->

All listed Strategies need to be in the same directory, unless also `--recursive-strategy-search` is specified, where sub-directories within the strategy directory are also considered.
<!-- 所有策略需在同一目录，除非指定 --recursive-strategy-search 来搜索子目录 -->

``` bash
freqtrade backtesting --timerange 20180401-20180410 --timeframe 5m --strategy-list Strategy001 Strategy002 --export trades
# --strategy-list：指定多个策略进行对比回测
# 结果将包含所有策略的对比汇总表
```

This will save the results to `user_data/backtest_results/backtest-result-<datetime>.json`, including results for both `Strategy001` and `Strategy002`.
There will be an additional table comparing win/losses of the different strategies (identical to the "Total" row in the first table).
<!-- 结果保存到 backtest_results 目录，包含所有策略的结果和对比表 -->

## Next step（下一步）

Great, your strategy is profitable. What if the bot can give you the optimal parameters to use for your strategy?
Your next step is to learn [how to find optimal parameters with Hyperopt](hyperopt.md)
<!-- 
太好了，你的策略是盈利的。
如果机器人能给你策略的最优参数呢？
下一步是学习如何使用 Hyperopt 寻找最优参数。
-->
