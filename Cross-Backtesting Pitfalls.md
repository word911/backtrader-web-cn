# Cross-Backtesting Pitfalls

从其他平台转移到Backtrader上回测的一些常见错误

[https://www.backtrader.com/blog/posts/2019-09-04-donchian-across-platforms/donchian-across-platforms/](https://www.backtrader.com/blog/posts/2019-09-04-donchian-across-platforms/donchian-across-platforms/)


Somethings that tends to repeat itself in the [backtrader community ](https://community.backtrader.com/) is that a user explains the will to replicate the backtesting results attained in, for example, [TradingView ](https://www.tradingview.com/), quite popular these days, or some other backtesting platform.
**在Backtrader社区有些问题往往一再重复，一些社区用户表达了复现其他平台回测结果的意愿，比如现在非常流行的TradingView以及其他回测平台。**

Without really knowing the language used in *TradingView*, named `Pinescript`, and having null exposure to the internal of the backtesting engine, there is still a way to let users know, that coding across platform has to be taken with pinch of salt.
**在不了解 TradingView 所使用的 Pinescript 语言及其回测引擎内部结构的情况下，跨平台进行编码回测必须慎之又慎。【相同策略在不同平台回测结果可能大不相同。】**

## Indicators: Not always faithful to the sources 指标：并不总是忠实于原始定义

When a new indicator is being implemented for *backtrader*, either directly for the distribution or as a snippet for the website, a lot of emphasis is taken in respecting the original definition. The `RSI` is a good example.
**当一个新技术指标实现于Backtrade时，无论是直接使用出版物【原文】还是【引用】来自网站的片段，【作者】都非常强调尊重【技术指标的】原始定义。**  `**RSI**` **指标就是一个很好的例子。**

* Welles Wilder designed the `RSI` using  the `Modified Moving Average` (aka `Smoothed Moving Average`, see [Wikipedia - Modified Moving Average     ](https://en.wikipedia.org/wiki/Moving_average#Modified_moving_average))
  **威尔斯·怀尔德（Welles Wilder）设计了使用修正移动平均线（又名平滑移动平均线，参见维基百科：修改后的移动平均线）的RSI指标**
* In spite of which, a number of platforms give the users something called  `RSI` but using a classic `Exponential Moving Average` and not what the     book says.
  **尽管如此，许多平台仍为用户提供了一些所谓的 RSI 指标，但这些指标并非经典书中的 RSI 指标，而是使用指数移动平均线 (EMA) 进行计算的。**
* Given that both averages are exponential, the differences are not huge, but  it is **NOT** what Welles Wilder defined. It may still be useful, it may even be better, but it is **NOT** the `RSI`. And the documentation (when  available) fails to mentions that.
  **两类移动平均线虽然都是基于指数级变换，但仍然存在一些差异。使用 EMA 计算的 RSI 指标具有一定的参考价值，甚至可能在某些情况下比传统的 RSI 指标更加有效。但投资者需要注意，这并非威尔斯·怀尔德定义的 RSI 指标。【平台上的】文档（如果有的话）也并未提示这一点**。

The default configuration for the `RSI` in *backtrader* is to use the `MMA` to be faithful to the sources, but which moving average to use is a parameter which can be changed via subclassing or during run-time instantiation to use the `EMA` or even a *Simple Moving Average*.
**Backtrader中**​`**RSI**`​**的默认使用MMA（Modified Moving Average），这是忠实于原著的。但是，根据需要可以调整使用的移动平均线，可以通过子类化或在运行时实例化期间进行更改，以使用**​`**EMA**`​ **（Exponential Moving Average）甚至简单移动平均线。**


## An example: The Donchian Channels 一个例子：唐奇安通道

The Wikipedia definition: [Wikipedia - Donchian Channel ](https://en.wikipedia.org/wiki/Donchian_channel)). It is just text and it makes no mention of using channel breakouts as a trading signal.
**维基百科的定义：维基百科 - Donchian Channel 。它只是叙述【通道的含义】，没有提到使用通道突破作为交易信号。**

Another two definitions:
**另外两处定义：**

* [StockCharts - School - Price Channels](https://school.stockcharts.com/doku.php?id=technical_indicators:price_channels)
* [IncredibleCharts - Donchian Channels](https://www.incrediblecharts.com/indicators/donchian_channels.php)

These two references explicitly state, that the data for the calculation of the channel does not include the current bar, because if it did ... breakouts would not be reflected. Here is a sample chart from *StockCharts*
**这两个参考文献明确指出，用于计算通道的数据不包括当前柱线，因为如果它确实包括【当日柱线的话】......【当日】突破就无法显示出来。这是来自StockCharts的示例图表**

![!StockCharts - Donchian Channels- Breakouts](https://www.backtrader.com/blog/posts/2019-09-04-donchian-across-platforms/stockcharts-donchian-breakouts.png)

Going now to *TradingView*. First the link
**现在转到TradingView。首先，链接如下**

* [TradingView - Wiki - Donchian Channels](https://www.tradingview.com/wiki/Donchian_Channels_(DC))

And a chart from that page.
**还有该页面的图表。**

![!TradingView - Donchian Channels - No](assets/tradingview-donchian-no-breakouts-20240127225252-3qr51fu.png)[https://www.backtrader.com/blog/posts/2019-09-04-donchian-across-platforms/tradingview-donchian-no-breakouts.png](https://www.backtrader.com/blog/posts/2019-09-04-donchian-across-platforms/tradingview-donchian-no-breakouts.png)

Even *Investopedia* uses a chart from *TradingView* that shows **no breakout**. Here: [Investopedia - Donchian Channels - https://www.investopedia.com/terms/d/donchianchannels.asp ](https://www.investopedia.com/terms/d/donchianchannels.asp)
**甚至 Investopedia 也使用来自 TradingView 的图表，该图表无法显示突破。链接如下： Investopedia - Donchian Channels -**  [https://www.investopedia.com/terms/d/donchianchannels.asp](https://www.investopedia.com/terms/d/donchianchannels.asp)

As some people would put it ... **Blistering Barnacles!!!!** . Because there are **no** breakouts to be seen in the *TradingView* charts. This means that the implementation of the indicator is using the **current** price bar for the calculation of the channels.
**正如一些人所说......天呐【来自海绵宝宝】!!!.因为在TradingView图表中看不到突破。这意味着指标的实现使用了当前价格柱线来计算【唐奇安】通道。**


## The Donchian Channels in *backtrader* Backtrader中的唐奇安通道

There is no `DonchianChannels` implementation in the standard *backtrader* distribution, but it can be quickly crafted. A parameter will be the deciding factor as to whether the current bar will be used for the channel calculation or not.
**标准Backtrader版本中没有(实现)唐奇安通道，但【我们】可以快速实现。【我们可以设定】一个参数，决定通道计算时是否纳入当前柱线。**

```python
class DonchianChannels(bt.Indicator):
    '''
    Params Note:
      - ``lookback`` (default: -1)

        If `-1`, the bars to consider will start 1 bar in the past and the
        current high/low may break through the channel.

        If `0`, the current prices will be considered for the Donchian
        Channel. This means that the price will **NEVER** break through the
        upper/lower channel bands.
    '''

    alias = ('DCH', 'DonchianChannel',)

    lines = ('dcm', 'dch', 'dcl',)  # dc middle, dc high, dc low
    params = dict(
        period=20,
        lookback=-1,  # consider current bar or not
    )

    plotinfo = dict(subplot=False)  # plot along with data
    plotlines = dict(
        dcm=dict(ls='--'),  # dashed line
        dch=dict(_samecolor=True),  # use same color as prev line (dcm)
        dcl=dict(_samecolor=True),  # use same color as prev line (dch)
    )

    def __init__(self):
        hi, lo = self.data.high, self.data.low
        if self.p.lookback:  # move backwards as needed
            hi, lo = hi(self.p.lookback), lo(self.p.lookback)  # 这里是关键

        self.l.dch = bt.ind.Highest(hi, period=self.p.period)
        self.l.dcl = bt.ind.Lowest(lo, period=self.p.period)
        self.l.dcm = (self.l.dch + self.l.dcl) / 2.0  # avg of the above
```

Using it with `lookback=-1` a sample chart looks like this (zoomed in)
`lookback=-1`时，示例图如下所示（已放大）

![!Backtrader - Donchian Channels -](assets/bt-donchian-lookback-1-breakouts-20240127225252-548g6oo.png)[https://www.backtrader.com/blog/posts/2019-09-04-donchian-across-platforms/bt-donchian-lookback-1-breakouts.png](https://www.backtrader.com/blog/posts/2019-09-04-donchian-across-platforms/bt-donchian-lookback-1-breakouts.png)

One can clearly see the breakouts, whereas there are no breakouts in the `lookback=0` version.
上图中，我们可以清楚地看到突破，而 在`lookback=0`时无法看到突破，如下图。

![!Backtrader - Donchian Channels -](assets/bt-donchian-lookback-0-no-breakouts-20240127225252-yzqgv7k.png)[https://www.backtrader.com/blog/posts/2019-09-04-donchian-across-platforms/bt-donchian-lookback-0-no-breakouts.png](https://www.backtrader.com/blog/posts/2019-09-04-donchian-across-platforms/bt-donchian-lookback-0-no-breakouts.png)

## Coding Implications  编码含义

The programmer first goes to the commercial platform and implements a strategy using *The Donchian Channels*. Because the chart does not show breakouts, a comparison of the current price value has to be done against the previous channel value. Something as
**试想一个量化程序员打开商业平台，编码实现唐奇安通道突破策略时。由于图表从未显示突破，因此，必须将【当前价格】与【前一日的通道值】进行比较。如下：**

```
if price0 > channel_high_1:
    sell()
elif price0 < channel_low_1:
    buy()
```

The current price, i.e.: `price0` is being compared to the high/low channel values of `1` period ago (hence the `_1` suffix)
**当前价格即**​`**price0**` **与** `**1**` **前段的高/低通道值进行比较（因此加入**  `**_1**` **后缀）**

Being a cautious programmer and unaware that the implementation of the *Donchian Channels* in *backtrader* defaults to having breakout, the code is ported and looks like this
 **【由于并不知道Backtrader中唐奇安通道默认已实现突破功能，如上】，作为一个谨慎的量化程序员，【简单】移植【商业平台上的】代码可能会是这样：**

```
    def __init__(self):
        self.donchian = DonchianChannels()

    def next(self):
        if self.data[0] > self.donchian.dch[-1]:
            self.sell()
        elif self.data[0] < self.donchian.dcl[-1]:
            self.buy()
```

Which is wrong!!! Because the breakout happens at the same moment of the comparison. The correct code:
 **【将这个简单的移植代码用于Backtrader平台】却是错误的!!因为突破发生在【价格与通道上下沿】比较的同一时刻。正确的代码应该是：**

```
    def __init__(self):
        self.donchian = DonchianChannels()

    def next(self):
        if self.data[0] > self.donchian.dch[0]:
            self.sell()
        elif self.data[0] < self.donchian.dcl[0]:
            self.buy()
```

Although this is just a small example, it shows how backtesting results may differ because an indicator has been coded with a `1` bar difference. It may not seem much, but it will for sure make a difference when the wrong trade is started.
**虽然这只是一个小例子，但它显示了回测结果可能会有差异，因为指标的编码有1个柱线的差异。这看起来问题不大，但当错误交易【信号及指令】启动时，结果肯定会大不一样。**
