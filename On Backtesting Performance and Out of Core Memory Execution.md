# On Backtesting Performance and Out of Core Memory Execution
关于回测的性能和内存外执行的情况

[https://www.backtrader.com/blog/2019-10-25-on-backtesting-performance-and-out-of-memory/on-backtesting-performance-and-out-of-memory/](https://www.backtrader.com/blog/2019-10-25-on-backtesting-performance-and-out-of-memory/on-backtesting-performance-and-out-of-memory/)


本文主要是为了回应 [https://redit.com/r/algotrading](https://redit.com/r/algotrading) 最近的两个帖子。

* 其一是质疑Backtrader无法处理160万根K线: [reddit/r/algotrading - A performant backtesting system? ](https://www.reddit.com/r/algotrading/comments/dlfujr/a_performant_backtesting_system/)
* 另一个询问哪种工具可以回测8000只股票？: [reddit/r/algotrading - Backtesting libs that supports 1000+ stocks?     ](https://www.reddit.com/r/algotrading/comments/dmv51t/backtesting_libs_that_supports_1000_stocks/)  
* 作者想找到一个支持“内核/内存外”的回测框架，“因为数据量巨大，显然无法将所有数据加载到内存中”

本文中，我们将用Backtrader一起解决这些问题

##  两百万根K线

为了做到这一点，第一件事是产生足够数量的K线数据。鉴于第一个帖子谈论的是 77 只股票和 160万根K线，这相当于每只股票约有20,779根K线，因此我们将执行以下操作以获得对应的数据

* 为100只股票生成K线
* 每只股票产生20,000根K线

即：100 个文件，共 200万根K线。

代码如下：

```python
import numpy as np
import pandas as pd

COLUMNS = ['open', 'high', 'low', 'close', 'volume', 'openinterest']
CANDLES = 20000
STOCKS = 100

dateindex = pd.date_range(start='2010-01-01', periods=CANDLES, freq='15min')

for i in range(STOCKS):
    data = np.random.randint(10, 20, size=(CANDLES, len(COLUMNS)))
    df = pd.DataFrame(data * 1.01, dateindex, columns=COLUMNS)
    df = df.rename_axis('datetime')
    df.to_csv('candles{:02d}.csv'.format(i))
```

这将生成 100 个文件，从开始的`candles00.csv` ，一直到`candles99.csv` 。实际值并不重要。文件拥有标准 `datetime` 、 `OHLCV` 组件（和 `OpenInterest` ）才是最重要的。

## 测试系统

* 硬件/操作系统：将使用配备Intel i7和32GB内存的Windows10 15.6 英寸笔记本电脑。
* Python: CPython `3.6.1` 和 `pypy3 6.0.0`
* 其他：测试时，本电脑运行的应用程序占用大约20%的CPU运行时间，这些持续运行的程序大致包括如Chrome（102个进程），Edge，Word，Powerpoint，Excel和一些其他的应用。

## Backtrader的默认配置

让我们回顾一下Backtrader的默认运行时配置是什么：

* 如果可能，请预加载所有数据馈送(data feeds)
* 如果所有数据馈送均可以预先加载，请在批处理模式下运行（名为 `runonce`）
* 首先，请预先计算所有指标
* 以步进方式执行策略逻辑以及经纪人（的交易）

## 在默认批处理 `runonce` 模式下执行上述挑战问题

我们的测试脚本（完整源代码见底部）将打开这 100 个文件，并使用默认的Backtrader配置处理。运行结果如下：

```
$ ./two-million-candles.py
Cerebro Start Time:          2019-10-26 08:33:15.563088
Strat Init Time:             2019-10-26 08:34:31.845349
Time Loading Data Feeds:     76.28
Number of data feeds:        100
Strat Start Time:            2019-10-26 08:34:31.864349
Pre-Next Start Time:         2019-10-26 08:34:32.670352
Time Calculating Indicators: 0.81
Next Start Time:             2019-10-26 08:34:32.671351
Strat warm-up period Time:   0.00
Time to Strat Next Logic:    77.11
End Time:                    2019-10-26 08:35:31.493349
Time in Strategy Next Logic: 58.82
Total Time in Strategy:      58.82
Total Time:                  135.93
Length of data feeds:        20000
```

内存使用：观察到的峰值为 348 MB

大部分时间实际上花在预加载数据上（98.63秒），其余时间花在策略【执行】上（包含每次迭代中的经纪人交易）（73.63秒）。总时间173.26秒 。

依据这个特定运算过程，（Backtrader的处理）性能如下：

* 整个运行时间内的处理速度为 14,713根K线/秒，

划线说明：上面两个 reddit 帖子中第一个帖子里提到Backtrader无法处理160万根K线的表述是错误的。


### 用pypy能否提升速度

由于帖子声称使用pypy没有提升性能，让我们看看使用它时发生了什么。

```
$ ./two-million-candles.py
Cerebro Start Time:          2019-10-26 08:39:42.958689
Strat Init Time:             2019-10-26 08:40:31.260691
Time Loading Data Feeds:     48.30
Number of data feeds:        100
Strat Start Time:            2019-10-26 08:40:31.338692
Pre-Next Start Time:         2019-10-26 08:40:31.612688
Time Calculating Indicators: 0.27
Next Start Time:             2019-10-26 08:40:31.612688
Strat warm-up period Time:   0.00
Time to Strat Next Logic:    48.65
End Time:                    2019-10-26 08:40:40.150689
Time in Strategy Next Logic: 8.54
Total Time in Strategy:      8.54
Total Time:                  57.19
Length of data feeds:        20000
```

天哪！运行总时间已从135.93秒减少到57.19秒。性能提高了一倍多。

性能数据：34,971根K线/秒

内存使用：峰值为 269 MB。

这就 pypy 是对标准 CPython 解释器的重要改进。


## 处理不在内存内(out of core memory)的 200万根K线

Backtrader有几个用于执行回测的配置选项可以解决这些问题，包括【设置】优化缓冲区和仅使用所需的最少数据集【的选项】（理想情况下可以减少到只包含1根K线的缓存，但显然这只会在理想情况下，因为指标的计算不能只包括1根K线）

要使用的选项是 `**exactbars=True**` 。（这是在实例化​`Cerebro`时或调用`run`函数时的参数）​`exactbars`文档如下：

```
  `True` or `1`: all “lines” objects reduce memory usage to the
  automatically calculated minimum period.
  `True`或`1`：所有的"lines"对象将减少内存使用量到（需要自动计算出指标的）最小周期。

  If a Simple Moving Average has a period of 30, the underlying data
  will have always a running buffer of 30 bars to allow the
  calculation of the Simple Moving Average
  如果一个简单移动平均线的周期为30，底层数据将始终保持一个只含30根K线的缓存，
  以便计算简单移动平均线。

  * This setting will deactivate `preload` and `runonce`
    这个设置将禁用"preload"和"runonce"。 

  * Using this setting also deactivates **plotting**
    使用这个设置也将禁用绘图。
```

为了最大程度地优化，绘图将被禁用，因此还需要进行如下设置：`stdstats=False`。【stdstats这个选项设置为false】将禁用【Backtrader的】标准观察者【Observer】，【观察者主要用于观察回测中的现金、资产净值、交易等；这些信息对于绘图很有用，禁用后，这些信息将不再提供】。

```
$ ./two-million-candles.py --cerebro exactbars=False,stdstats=False
Cerebro Start Time:          2019-10-26 08:37:08.014348
Strat Init Time:             2019-10-26 08:38:21.850392
Time Loading Data Feeds:     73.84
Number of data feeds:        100
Strat Start Time:            2019-10-26 08:38:21.851394
Pre-Next Start Time:         2019-10-26 08:38:21.857393
Time Calculating Indicators: 0.01
Next Start Time:             2019-10-26 08:38:21.857393
Strat warm-up period Time:   0.00
Time to Strat Next Logic:    73.84
End Time:                    2019-10-26 08:39:02.334936
Time in Strategy Next Logic: 40.48
Total Time in Strategy:      40.48
Total Time:                  114.32
Length of data feeds:        20000
```

性能数据： 17,494根K线/秒

内存占用：75 MB（从回测会话开始到结束始终保持稳定）

让我们与之前的未优化运行进行比较

* 与之前需要使用76秒预加载数据不同，回测是立即开始的，因为在该模式下数据不需要预加载
* 优化后总时间是114.32秒，相较于之前135.93秒，提升了15.90%。
* 相对于优化前占用内存348Mb，优化后仅占用75Mb，内存使用率方面改进了68.5% = (1 - 75/348)。

**注意**

我们可以在程序代码中导入1亿根K线，内存消耗量仍将保持在75Mb；


### 优化模式下使用 `pypy`运行

现在我们知道了如何优化，让使用 `pypy` 运行一次。

```
$ ./two-million-candles.py --cerebro exactbars=True,stdstats=False
Cerebro Start Time:          2019-10-26 08:44:32.309689
Strat Init Time:             2019-10-26 08:44:32.406689
Time Loading Data Feeds:     0.10
Number of data feeds:        100
Strat Start Time:            2019-10-26 08:44:32.409689
Pre-Next Start Time:         2019-10-26 08:44:32.451689
Time Calculating Indicators: 0.04
Next Start Time:             2019-10-26 08:44:32.451689
Strat warm-up period Time:   0.00
Time to Strat Next Logic:    0.14
End Time:                    2019-10-26 08:45:38.918693
Time in Strategy Next Logic: 66.47
Total Time in Strategy:      66.47
Total Time:                  66.61
Length of data feeds:        20000
```

性能数据： 30,025根K线/秒

内存占用：稳定在49 Mb

将其与之前的等效运行进行比较：

* 66.61秒与114.32秒，运行时间缩短41.73%
* 49 Mbytes与75 Mbytes，改进34.6%内存占用。

**注意**

在pypy运行的场景下，批处理(runonce)模式消耗57.19秒，与一般模式相比，在运行时间方面并无提升。这是意料之中的，因为在预加载时，计算器指示是在矢量化模式下完成的，而这正是pypy的just-in-time编译器的优势所在。
In this case `pypy` has not been able to beat its own time compared to the batch (`runonce`) mode, which was `57.19` seconds. This is to be expected, because when preloading, the calculator indications are done in **vectorized** mode and that's where the JIT of `pypy` excels

无论如何，pypy仍然做得很好，并且在内存消耗方面有重要的改进

## 完整的交易运行

该程序代码在100个标的上创建移动平均线指标，并执行基于移动平均线交叉的多空策略(回测)，我们使用pypy来运行该程序，并已知在批处理模式下会更佳，结果如下。

```
$ ./two-million-candles.py --strat indicators=True,trade=True
Cerebro Start Time:          2019-10-26 08:57:36.114415
Strat Init Time:             2019-10-26 08:58:25.569448
Time Loading Data Feeds:     49.46
Number of data feeds:        100
Total indicators:            300
Moving Average to be used:   SMA
Indicators period 1:         10
Indicators period 2:         50
Strat Start Time:            2019-10-26 08:58:26.230445
Pre-Next Start Time:         2019-10-26 08:58:40.850447
Time Calculating Indicators: 14.62
Next Start Time:             2019-10-26 08:58:41.005446
Strat warm-up period Time:   0.15
Time to Strat Next Logic:    64.89
End Time:                    2019-10-26 09:00:13.057955
Time in Strategy Next Logic: 92.05
Total Time in Strategy:      92.21
Total Time:                  156.94
Length of data feeds:        20000
```

性能数据： 12,743根K线/秒

内存占用：观察到峰值1300 Mbytes

执行时间明显增加了（指标+交易），但为什么内存使用量增加？

在得出任何结论之前，我们运行它，创建指标，但不【调用经纪人】交易

```
$ ./two-million-candles.py --strat indicators=True
Cerebro Start Time:          2019-10-26 09:05:55.967969
Strat Init Time:             2019-10-26 09:06:44.072969
Time Loading Data Feeds:     48.10
Number of data feeds:        100
Total indicators:            300
Moving Average to be used:   SMA
Indicators period 1:         10
Indicators period 2:         50
Strat Start Time:            2019-10-26 09:06:44.779971
Pre-Next Start Time:         2019-10-26 09:06:59.208969
Time Calculating Indicators: 14.43
Next Start Time:             2019-10-26 09:06:59.360969
Strat warm-up period Time:   0.15
Time to Strat Next Logic:    63.39
End Time:                    2019-10-26 09:07:09.151838
Time in Strategy Next Logic: 9.79
Total Time in Strategy:      9.94
Total Time:                  73.18
Length of data feeds:        20000
```

性能数据：27,329根K线/秒

内存使用：600 Mbytes（在优化后的exactbars模式下执行相同的操作只会消耗60 Mbytes内存，但执行时间会增加，`pypy` 无法进一步优化）

基于以上结果：交易发生时内存使用量确实会增加。原因是 `Order` 和 `Trade` 对象将由`broker`【经纪人实例】创建、传递及保存

**注意**

考虑到【本实验中的】数据集包含的是随机值，这会产生大量的【均线】交叉，因此`order`订单和`trade`交易的数量非常多。对于常规数据集，不应出现类似如此多的均线交叉。

## 结论

### 上述帖子中的陈述有误

已通过上述方法证明reddit的两个帖子内的叙述不成立，通过我们的实验，Backtrader可以处理 160万根甚至更多的K线。

### General 通常情况

1. Backtrader 可以使用默认配置轻松处理200万根K线（预加载数据到内存）
2. Backtrader 可以在优化模式下运行回测【不对数据进行预加载】，【优化模式】将减少缓存将【内存占用】降到最低限度，优化模式下的K线数据大部分不在内存中。
3. 在优化的非预加载模式下进行回测时，内存消耗的增加来自于【broker，经纪人】生成的各类管理信息【如order、trade】。
4. 即使在交易过程中，要不断地计算指标，同时经纪人也在不断发出各类信息【可能阻碍程序运行速度】，Backtrader的处理速度仍是12473根K线/秒
5. 尽可能地使用 `pypy` （例如，如果不需要绘图）

### 使用 Python 和/或backtrader的几种情况

基于 `pypy` 运行环境、启用交易、使用随机数据集（高于平时的交易数量）后，处理200万根K线耗费：

* 156.94秒，即：2分37秒

考虑到这是在同时运行多个其他程序的笔记本电脑中完成的，可以得出结论，200万根K线可以正常完成。

### 在8000个股票情况的场景下【Backtrader】如何？

执行时间按80倍缩放，因此：

* 在这个随机集的情景下，耗费12,560 seconds（或210分钟 或 3小时30分钟）

即使假设生成的标准数据集产生将较少的操作【均线交叉】，回测仍可能需要消耗3到4小时。

在交易时，由于经纪人的操作，内存使用量也会增加，并且可能需要几个G。

**注意**

不能在其他场景下简单的乘以80，因为示例程序用的是随机数据进行交易【其一致性是可以保证的】。无论如何，【为程序预留】所需的内存是很重要的。

因此，仅使用Backtrader作为【大规模数据的】研究和回测工具的工作流程似乎很牵强。

## 关于工作流程的讨论

使用Backtrader时，需要考虑两个标准工作流程

* 第一种流程，研究和回测均使用Backtrader，即：研究和回测合二为一
* 第二种流程，先使用python pandas对【数据和策略】进行大致研究，了解【策略】想法是否正确；然后再使用Backtrader回测，以尽可能准确地进行验证；这样可以将庞大的数据集【导致的内存占用】减少到更适合于一般内存大小的场景。

**提示**

可以考虑使用类似 `dask` 包替换 `pandas`，用作核外内存执行。

## 测试脚本

这里是源代码

```python
#!/usr/bin/env python
# -*- coding: utf-8; py-indent-offset:4 -*-
###############################################################################
import argparse
import datetime

import backtrader as bt


class St(bt.Strategy):
    params = dict(
        indicators=False,
        indperiod1=10,
        indperiod2=50,
        indicator=bt.ind.SMA,
        trade=False,
    )

    def __init__(self):
        self.dtinit = datetime.datetime.now()
        print('Strat Init Time:             {}'.format(self.dtinit))
        loaddata = (self.dtinit - self.env.dtcerebro).total_seconds()
        print('Time Loading Data Feeds:     {:.2f}'.format(loaddata))

        print('Number of data feeds:        {}'.format(len(self.datas)))
        if self.p.indicators:
            total_ind = self.p.indicators * 3 * len(self.datas)
            print('Total indicators:            {}'.format(total_ind))
            indname = self.p.indicator.__name__
            print('Moving Average to be used:   {}'.format(indname))
            print('Indicators period 1:         {}'.format(self.p.indperiod1))
            print('Indicators period 2:         {}'.format(self.p.indperiod2))

            self.macross = {}
            for d in self.datas:
                ma1 = self.p.indicator(d, period=self.p.indperiod1)
                ma2 = self.p.indicator(d, period=self.p.indperiod2)
                self.macross[d] = bt.ind.CrossOver(ma1, ma2)

    def start(self):
        self.dtstart = datetime.datetime.now()
        print('Strat Start Time:            {}'.format(self.dtstart))

    def prenext(self):
        if len(self.data0) == 1:  # only 1st time
            self.dtprenext = datetime.datetime.now()
            print('Pre-Next Start Time:         {}'.format(self.dtprenext))
            indcalc = (self.dtprenext - self.dtstart).total_seconds()
            print('Time Calculating Indicators: {:.2f}'.format(indcalc))

    def nextstart(self):
        if len(self.data0) == 1:  # there was no prenext
            self.dtprenext = datetime.datetime.now()
            print('Pre-Next Start Time:         {}'.format(self.dtprenext))
            indcalc = (self.dtprenext - self.dtstart).total_seconds()
            print('Time Calculating Indicators: {:.2f}'.format(indcalc))

        self.dtnextstart = datetime.datetime.now()
        print('Next Start Time:             {}'.format(self.dtnextstart))
        warmup = (self.dtnextstart - self.dtprenext).total_seconds()
        print('Strat warm-up period Time:   {:.2f}'.format(warmup))
        nextstart = (self.dtnextstart - self.env.dtcerebro).total_seconds()
        print('Time to Strat Next Logic:    {:.2f}'.format(nextstart))
        self.next()

    def next(self):
        if not self.p.trade:
            return

        for d, macross in self.macross.items():
            if macross > 0:
                self.order_target_size(data=d, target=1)
            elif macross < 0:
                self.order_target_size(data=d, target=-1)

    def stop(self):
        dtstop = datetime.datetime.now()
        print('End Time:                    {}'.format(dtstop))
        nexttime = (dtstop - self.dtnextstart).total_seconds()
        print('Time in Strategy Next Logic: {:.2f}'.format(nexttime))
        strattime = (dtstop - self.dtprenext).total_seconds()
        print('Total Time in Strategy:      {:.2f}'.format(strattime))
        totaltime = (dtstop - self.env.dtcerebro).total_seconds()
        print('Total Time:                  {:.2f}'.format(totaltime))
        print('Length of data feeds:        {}'.format(len(self.data)))


def run(args=None):
    args = parse_args(args)

    cerebro = bt.Cerebro()

    datakwargs = dict(timeframe=bt.TimeFrame.Minutes, compression=15)
    for i in range(args.numfiles):
        dataname = 'candles{:02d}.csv'.format(i)
        data = bt.feeds.GenericCSVData(dataname=dataname, **datakwargs)
        cerebro.adddata(data)

    cerebro.addstrategy(St, **eval('dict(' + args.strat + ')'))
    cerebro.dtcerebro = dt0 = datetime.datetime.now()
    print('Cerebro Start Time:          {}'.format(dt0))
    cerebro.run(**eval('dict(' + args.cerebro + ')'))


def parse_args(pargs=None):
    parser = argparse.ArgumentParser(
        formatter_class=argparse.ArgumentDefaultsHelpFormatter,
        description=(
            'Backtrader Basic Script'
        )
    )

    parser.add_argument('--numfiles', required=False, default=100, type=int,
                        help='Number of files to rea')

    parser.add_argument('--cerebro', required=False, default='',
                        metavar='kwargs', help='kwargs in key=value format')

    parser.add_argument('--strat', '--strategy', required=False, default='',
                        metavar='kwargs', help='kwargs in key=value format')


    return parser.parse_args(pargs)


if __name__ == '__main__':
    run()
```
