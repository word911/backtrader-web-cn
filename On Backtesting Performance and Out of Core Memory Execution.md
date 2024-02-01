# On Backtesting Performance and Out of Core Memory Execution
关于回测的性能和内存外执行的情况

[https://www.backtrader.com/blog/2019-10-25-on-backtesting-performance-and-out-of-memory/on-backtesting-performance-and-out-of-memory/](https://www.backtrader.com/blog/2019-10-25-on-backtesting-performance-and-out-of-memory/on-backtesting-performance-and-out-of-memory/)



There have been two recent [https://redit.com/r/algotrading](https://redit.com/r/algotrading) threads which are the inspiration for this article.

本文主要是为了回应 [https://redit.com/r/algotrading](https://redit.com/r/algotrading) 最近的两个帖子。

* A thread with a bogus claim that *backtrader* cannot cope with 1.6M candles: [reddit/r/algotrading - A performant backtesting system? ](https://www.reddit.com/r/algotrading/comments/dlfujr/a_performant_backtesting_system/)

  其一是质疑Backtrader无法处理160万根K线
  
* And another one asking for something which can backtest a universe of 8000 stocks: [reddit/r/algotrading - Backtesting libs that supports 1000+ stocks?     ](https://www.reddit.com/r/algotrading/comments/dmv51t/backtesting_libs_that_supports_1000_stocks/)

  另一个询问哪种工具可以同时回测8000只股票？
  
* With the author asking about a framework that can backtest '"out-of-core/memory" *,*   *&quot;because obviously it cannot load all this data into memory&quot;* 

  作者想找到一个支持“内核/内存外”的回测框架，“因为数据量巨大，显然无法将所有数据加载到内存中”

We'll be of course addressing these concepts with *backtrader*

本文中，我们将用Backtrader一起解决这些问题



## The 2M Candles  两百万根K线

In order to do this, the first thing is to generate that amount of candles. Given that the first poster talks about 77 stocks and 1.6M candles, this would amount to 20,779 candles per stock, so we'll do the following to have nice numbers

为了做到这一点，第一件事是产生足够数量的K线数据。鉴于第一个帖子谈论的是 77 只股票和 160万根K线，这相当于每只股票约有20,779根K线，因此我们将执行以下操作以获得对应的数据

* Generate candles for 100 stocks
  为100只股票生成K线
* Generate 20,000 candles per stock
  每只股票产生20,000根K线

I.e.: 100 files totaling 2M candles.
即：100 个文件，共 200万根K线。

The script

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

This generates 100 files, starting with `candles00.csv` and going all the way up to `candles99.csv`. The actual values are not important. Having the standard `datetime`, `OHLCV` components (and `OpenInterest`) is what matters.
这将生成 100 个文件，从开始的`candles00.csv` ，一直到`candles99.csv` 。实际值并不重要。文件拥有标准 `datetime` 、 `OHLCV` 组件（和 `OpenInterest` ）才是最重要的。

## The test system 测试系统

* *Hardware/OS*: A *Windows 10* 15.6" laptop with an Intel i7 and 32 Gbytes of memory will be used.
  硬件/操作系统：将使用配备Intel i7和32GB内存的Windows10 15.6 英寸笔记本电脑。
* *Python*: CPython `3.6.1` and `pypy3 6.0.0`
  Python：CPython `3.6.1` 和 `pypy3 6.0.0`
* *Misc*: an application running constantly and taking around 20% of the CPU. The usual suspects like Chrome (102 processes), Edge, Word, Powerpoint, Excel and some minor application are running
  其他：本电脑运行的应用程序占用大约 20% CPU运行时间，这些持续运行的程序大致包括如Chrome（102个进程），Edge，Word，Powerpoint，Excel和一些其他的应用。

## *backtrader* default configuration Backtrader 默认配置

Let's recall what the default run-time configuration for *backtrader* is:
**让我们回顾一下Backtrader的默认运行时配置是什么：**

* Preload all data feeds if possible
  **如果可能，请预加载所有数据馈送**
* If all data feeds can be preloaded, run in batch mode (named `runonce`)
  **如果所有数据馈送均可以预先加载，请在批处理模式下运行（名为** `**runonce**`  **）**
* Precalculate all indicators first
  **首先预先计算所有指标**
* Go through the strategy logic and broker step-by-step
  以步进方式执行策略逻辑以及经纪人（的交易）

## Executing the challenge in the default batch `runonce` mode 在默认批处理 `runonce` 模式下执行上述挑战问题

Our test script (see at the bottom for the full source code) will open those 100 files and process them with the default *backtrader* configuration.
我们的测试脚本（==完整源代码见底部==）将打开这 100 个文件，并使用默认的Backtrader配置处理。

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

**Memory Usage**: A peak of 348 Mbytes was observed
内存使用：观察到的峰值为 348 MB

Most of the time is actually spent preloading the data (`98.63` seconds), spending the rest in the strategy, which includes going through the broker in each iteration (`73.63` seconds). The total time is `173.26` seconds.
大部分时间实际上花在预加载数据上（`98.63`秒），其余时间花在策略【执行】上（包含每次迭代中的经纪人交易）（`73.63`秒）。总时间 `173.26`秒 。

Depending on how you want to calculate it the performance is:
依据这个特定运算过程，（Backtrader的处理）性能如下：

* `14,713` candles/second considering the entire run time
  整个运行时间内的处理速度为 `14,713` 根K线/秒，

**Bottomline**: the claim in the 1^st^ of the two reddit thread above that *backtrader* cannot handle 1.6M candles is *FALSE*.

划线说明：上面两个 reddit 帖子中第一个帖子里提到Backtrader无法处理160万根K线的表述是错误的。


### Doing it with `pypy`  用`pypy`能否提升速度

Since the thread claims that using `pypy` didn't help, let's see what happens when using it.
由于帖子声称使用 `pypy` 没有帮助（无法提升性能），让我们看看使用它时会发生什么。

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

Holy Cow! The total time has gone down to `57.19` seconds in total from `135.93` seconds. The performance has more than **doubled**.
天哪！运行总时间已从 `135.93` 秒减少到 `57.19` 秒。性能提高了一倍多。

The performance: `34,971` candles/second
性能数据： `34,971`根K线/秒

**Memory Usage**: a peak of 269 Mbytes was seen.
内存使用：峰值为 269 MB。

This is also an important improvement over the standard CPython interpreter.
这也是对标准 CPython 解释器的重要改进。


## Handling the 2M candles out of core memory 处理不在**内存内**的 200万根K线

All of this can be improved if one considers that *backtrader* has several configuration options for the execution of a backtesting session, including optimizing the buffers and working only with the minimum needed set of data (ideally with just buffers of size `1`, which would only happen in ideal scenarios)
**Backtrader有几个用于执行回测的配置选项可以解决这些问题，包括【设置】优化缓冲区和仅使用所需的最少数据集【的选项】（理想情况下可以减少到只包含1根K线的缓存，但显然这只会在理想情况下[因为指标的计算不能只包括1根K线]）**

The option to be used will be `exactbars=True`. From the documentation for `exactbars` (which is a parameter given to `Cerebro` during either instantiation or when invoking `run`)
**要使用的选项是** `**exactbars=True**`  **。（这是在实例化**​`**Cerebro**`​**时或调用** `**run**`​**函数时的参数）**​`**exactbars**`​**文档如下：**

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

For the sake of maximum optimization and because plotting will be disabled, the following will be used too: `stdstats=False`, which disables the standard *Observers* for cash, value and trades (useful for plotting, which is no longer in scope)
**为了最大程度地优化，绘图将被禁用，因此还需要进行如下设置：**  `**stdstats=False**`  **。【stdstats这个选项设置为false】将禁用【Backtrader的】标准观察者【Observer】，【观察者主要用于观察回测中的现金、资产净值、交易等；这些信息对于绘图很有用，禁用后，这些信息将不再提供（不在讨论范围内）】。**

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

The performance: `17,494` candles/second
性能： `17,494` 根K线/秒

Memory Usage: 75 Mbytes (stable from the beginning to the end of the backtesting session)
内存占用：75 MB（从回测会话开始到结束始终保持稳定）

Let's compare to the previous non-optimized run
让我们与之前的未优化运行进行比较

* Instead of spending over `76` seconds preloading data, backtesting starts     immediately, because the data is not preloaded
  与之前需要使用76秒预加载数据不同，回测是立即开始的，因为在该模式下数据不需要预加载
* The total time is `114.32` seconds vs `135.93`. An improvement of `15.90%`.
  优化后总时间是114.32秒，相较于之前135.93秒，提升了15.90%。
* An improvement in memory usage of `68.5%`.
  相对于优化前占用内存348Mb，优化后仅占用75Mb，内存使用率方面改进了68.5% = (1 - 75/348)。

**Note 注意**

We could have actually thrown 100M candles to the script and the amount of memory consumed would have remained fixed at `75 Mbytes`
我们可以在程序代码中导入1亿根K线，内存消耗量将保持在75Mb；


### Doing it again with `pypy`  优化模式下使用 `pypy`运行

Now that we know how to optimize, let's do it the `pypy` way.
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

The performance: `30,025` candles/second
性能： `30,025` 根K线/秒

Memory Usage: constant at `49 Mbytes`
内存占用：稳定在 `49 Mb`

Comparing it to the previous equivalent run:
将其与之前的等效运行进行比较：

* `66.61` seconds vs `114.32` or a `41.73%` improvement in run time
  `66.61` 秒与 `114.32` 秒，运行时间缩短 `41.73%`
* `49 Mbytes` vs `75 Mbytes` or a `34.6%` improvement.
  `49 Mbytes` 与 `75 Mbytes` ， 改进`34.6%`内存占用。

**Note  注意**

In this case `pypy` has not been able to beat its own time compared to the batch (`runonce`) mode, which was `57.19` seconds. This is to be expected, because when preloading, the calculator indications are done in **vectorized** mode and that's where the JIT of `pypy` excels
**在pypy运行的场景下，批处理(runonce)模式消耗57.19秒，与一般模式相比，在运行时间方面并无提升。这是意料之中的，因为在预加载时，计算器指示是在矢量化模式下完成的，而这就是pypy的just-in-time编译器的优势所在**

It has, in any case, still done a very good job and there is an important **improvement** in memory consumption
无论如何，【pypy】仍然做得很好，并且在内存消耗方面有重要的改进

## A complete run with trading 完整的交易运行

The script can create indicators (moving averages) and execute a *short/long* strategy on the 100 data feeds using the crossover of the moving averages. Let's do it with `pypy`, and knowing that it is better with the batch mode, so be it.
该程序代码在100个数据馈送(100个标的)上计算[创建]移动平均线指标，并执行基于移动平均线交叉的多空策略(回测)，我们使用pypy来运行该程序，并已知在批处理模式下会更佳，结果如下。

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

The performance: `12,743` candles/second
性能： `12,743` 根K线/秒

**Memory Usage**: A peak of `1300 Mbytes` was observed.
内存占用：观察到峰值 `1300 Mbytes`

The execution time has obviously increased (indicators + trading), but why the memory usage increase?
执行时间明显增加了（指标+交易），但为什么内存使用量增加？
