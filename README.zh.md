# Polymarket 5 分钟 Up/Down 开奖结果爬虫

一个极小、几乎零依赖、注释详尽的参考实现,用来读取 Polymarket 5 分钟加密
up/down 合约(BTC / ETH / SOL / XRP)的**早期「Outcome: Up / Down」信号** ——
*在*链上 oracle 结算之前就拿到。

它完全独立运行:**没有交易机器人、没有券商 SDK、没有 API key、不需要账号。**
指给它一个合约,返回 `up` / `down`。

> 唯一的源文件 `polymarket_outcome_scraper.py` 是按教程方式写的 —— 读文件顶部那段
> 很长的模块 docstring,里面完整解释了*为什么*需要这个方法、以及它*怎么*工作。

English docs see [README.md](README.md)

## 为什么要用浏览器?

这个「及时的答案」用别的办法是真的很难拿到:

| 数据源 | 问题 |
| --- | --- |
| 链上 oracle(`closed` / `outcomePrices`) | 权威,但慢 —— 结算要滞后好几分钟。 |
| 原始 HTML(`curl` 页面) | 「Outcome: Up」这段文字不在 HTML 里,也不在任何单个 XHR 里;它是在浏览器里算出来的。 |
| 前端渲染 | 在 K 线收盘后约 20 秒内就知道答案。✅ |

所以我们*变成*前端:用无头 Chromium 加载页面,让 React 渲染,再用一小段正则
从实时 DOM 里把开奖结果读出来。

## 5 个关键思路

1. **确定性 URL** —— `https://polymarket.com/event/{asset}-updown-5m-{t_start}`,其中 `t_start` 是该 K 线的 UNIX 开盘时间,按 5 分钟边界对齐(UTC)。
2. **用 Playwright 在无头 Chromium 里渲染**。
3. **在浏览器内部轮询 DOM** —— 用 `page.wait_for_function(probe, polling=100)`,文字一出现就立刻返回,而不是按固定定时器把 HTML 拉回 Python。
4. **整个进程复用一个浏览器**;用 `page.goto()` 导航(SPA 快速跳转,JS/CSS 留缓存)。
5. **拦截笨重/无关的请求**(图片、字体、媒体、第三方分析),但**放行第一方 JS/CSS** —— 拦了这些 React 就渲染不出来。

此外还有:带退避的重试直到截止时间、定期回收浏览器以控制内存、以及一个内存缓存。

## 安装

```bash
pip install playwright
playwright install chromium     # 一次性下载浏览器
```

## 命令行用法

```bash
# 最近一根已收盘的 BTC K 线:
python polymarket_outcome_scraper.py --asset btc

# 按 UNIX 开盘时间指定某根 K 线:
python polymarket_outcome_scraper.py --asset eth --t-start 1716900000

# 显示浏览器(调试用):
python polymarket_outcome_scraper.py --asset btc --no-headless
```

解析到开奖结果时退出码为 `0`,否则非零 —— 方便写进 shell 脚本。

## Python 用法

```python
import asyncio
from polymarket_outcome_scraper import OutcomeScraper

async def main():
    async with OutcomeScraper(asset="btc") as scraper:
        outcome, status, attempts = await scraper.get_outcome_with_retry(
            t_start=1716900000,
        )
        print(outcome)   # 'up' / 'down' / None

asyncio.run(main())
```

## 注意事项与伦理

- 读取的是**公开、已渲染**的信息。请保持适度的请求频率,遵守 Polymarket 的服务条款,不要猛刷网站。
- 前端标签是**早期指标**,不是最终的链上结算;如果对正确性要求极高,请再与权威数据核对。
- 如果 Polymarket 改了页面结构,更新 `_OUTCOME_DOM_PROBE_JS` 里的正则即可 —— 解析逻辑只在这一处。

## 还需要更多?(PRO 版 & PTB 版)

这个仓库是**最小、教程式的参考实现**:5 分钟 BTC/ETH/SOL/XRP 开奖结果,除 Playwright 外零依赖,可以从头读到尾搞懂它*怎么*工作。如果你已经超出它的能力范围,有两个开箱即用的版本接着往下做:

**➡️ [PTB & Outcome Scraper — PRO](https://bluewhalequantlab.gumroad.com/l/polymarket-settled-outcome-up-or-down)** —— 完整版:**全部周期(`5m`/`15m`/`1h`/`4h`/`1d`)、全币种**(BTC、ETH、SOL、XRP、DOGE,可扩展),并带有完整的 slug 工厂,能构建那些纯时间戳搞不定、按美东时间对齐的 `1h`/`4h`/`1d` 合约。返回 PTB、Outcome、收盘价的干净对象。

**➡️ [T0 Price-To-Beat Resolver](https://bluewhalequantlab.gumroad.com/l/polymarket-t0-price-to-beat-ptb-resolver)** —— 在 **T0 拿到开盘 Price To Beat**,实时,在周期开盘那一刻就拿到(而不是结算之后)。Polymarket 的 API 对加密 Up/Down 合约不提供 PTB;本方案从平台用来结算的同一条 Chainlink oracle 流和 Binance K 线里解析出来。这正是大多数 Up/Down 机器人卡住的那一块。

两者都是独立的 Python,中英文文档齐全,提供 14 天「修复或退款」保证。

## License

随便用。注明出处感谢,不强求。
