# Overnight vs Intraday Returns in US Equities (2016–2026)

Do US stocks earn their returns while the market is closed or while it is open? This project splits every daily equity returns into an **overnight** part (previous close to today's open) and an **intraday** part (today's open to today's close). It then backtests strategies that hold only one of the two, and tests whether the monthly ranking pattern from Lou, Polk & Skouras (2019), *"A tug of war"*, still appears in recent data.

**Headline results:**
- **Before costs, the overnight return dominates.** An equal-weight portfolio of all stocks earns about **+19% a year** holding only overnight. Holding only intraday loses about **−3% a year**. Buy and hold makes +14% and SPY +13.6%.
- **Ranking makes the effect much stronger.** Buying last month's overnight winners and shorting the losers using a Long/Short top/bottom 2 bucket earns **+41% a year** before costs and **+38%** after regulatory fees.
- **The effect lives in smaller cap companies** Restricted to the 1,000 largest stocks (reranked monthly), the same Long/Short earns only **+8.7% a year** after fees, below SPY.
- **Trading costs decide it.** The strategies trade every stock twice a day. A commission of just **0.05% per trade** turns the ranked Long/Short negative.

All results are backtests on historical data, before spreads and taxes, and subject to the limitations below.

---

## Contents

1. [Data](#data)
2. [Definitions and method](#definitions-and-method)
3. [Cost model](#cost-model)
4. [Results](#results)
   1. [Base model: all 4,000 stocks](#1-base-model-all-4000-stocks)
   2. [Common stocks (3,500) vs 500 most traded](#2-common-stocks-3500-vs-500-most-traded)
   3. [Ranked strategy: all stocks](#3-ranked-strategy-all-stocks)
   4. [Ranked strategy: 1,000 largest stocks](#4-ranked-strategy-1000-largest-stocks)
5. [Limitations](#limitations)
6. [Repository layout](#repository-layout)
7. [How to run](#how-to-run)
8. [References](#references)

---

## Data

| Database | What it holds |
|---|---|
| `history.parquet` | Daily OHLCV bars: 8.06M rows, 4,000 US-listed symbols, 2016-09-19 to 2026-09-18 (2,514 trading days). Columns: `symbol, name, cik, date, open, high, low, close, volume`. Prices are split-adjusted. |
| `SPY_D_history.json` | Daily SPY bars (2016-09-20 to 2026-09-18), used only as a cost-free benchmark. |

**Cleaning:**
- **No-trade bars dropped:** about 98k bars where open = high = low = close with no volume.
- **Extreme moves excluded:** in the portfolio views, any stock-day whose overnight or intraday move exceeds ±50% is treated as a likely data error and left out (1,206 overnight moves exceed this, one of them about 649,000×).
- **Common stocks only, where noted:** "common stocks" excludes names containing *Warrant, Right, Unit, Preferred, Depositary Shares, Notes, Debentures* or `%`, which leaves about 3,500.

---

## Definitions and method

For each stock on each trading day:

| Measure | Formula |
|---|---|
| Overnight change ($) | `open_t − close_{t−1}` |
| Overnight return (%) | `open_t / close_{t−1} − 1` |
| Intraday change ($) | `close_t − open_t` |
| Intraday return (%) | `close_t / open_t − 1` |

**Strategies:**
- **Overnight:** buy at every close and sell at the next open.
- **Intraday:** buy at every open and sell at that day's close.
- **Buy and hold:** hold the same stocks all the time (close to close).
- **SPY:** buy SPY and hold it. It is a benchmark only and is never charged costs.

**Portfolios** are equal-weight and rebalanced daily, with $1,000 invested at the start. Returns compound daily.

**Ranked strategy** (following Lou, Polk & Skouras, 2019):
1. At each month end, take every stock with at least 15 trading days that month.
2. Rank them by that month's compounded overnight return (overnight test) or intraday return (intraday test).
3. Split them into **10 equal groups**, where group 1 holds last month's top 10%.
4. Through the next month, each group runs the matching strategy (overnight or intraday), equal-weight and rebalanced daily.
5. The **long-short** buys groups 1–2 and shorts groups 9–10, with each leg sized at full portfolio value.

**Top-1,000 variant:** before ranking, each month keeps only the 1,000 largest common stocks. The file has no shares outstanding, so size is measured as the average daily **dollar volume** (close × volume) over the ranking month.

---

## Cost model

Unless stated otherwise, "after costs" means **regulatory fees only**: no bid-ask spread, no commission and no borrow fee.

| Cost | How it is charged |
|---|---|
| SEC Section 31 fee | 0.00278% of every sale ($27.80 per $1M, the top of its 2016–2026 range) |
| FINRA Trading Activity Fee | $0.000166 per share sold, capped at $8.30 per trade |
| Commission (excluded in this run) | `--commission-pct`, charged on every buy and sell |
| Short borrow (excluded in this run) | `--borrow-pct`, an annual rate charged daily on the short leg |

**How fees hit each trade:**
- **Long position:** daily net return = `(1 + r)(1 − fee − c) / (1 + c) − 1`
- **Short position:** daily net return = `−r − fee − 2c − borrow/252`. The fee is paid on the opening short sale.
- **Every stock is charged separately**, every day. The ranked long-short over all stocks makes roughly 2,600 trades a day.

---

## Results

Unless noted, the period is Sep 2016 to Sep 2026, portfolios start with $1,000, "per year" is the compound annual return, and SPY is never charged costs.

### 1. Base model: all 4,000 stocks

Equal-weight across every symbol in the file, including warrants and preferreds, with ±50% days removed.

<img width="940" height="334" alt="image" src="https://github.com/user-attachments/assets/5eccc7cf-e0d5-46a1-b11c-e9a2f440e754" />

| Strategy | Per year (before) | $1,000 → (before) | Per year (after fees) | $1,000 → (after fees) | Worst drop |
|---|---|---|---|---|---|
| Overnight | **+19.0%** | $5,675 | **+17.7%** | $5,101 | −28% |
| Buy and hold | +13.6% | $3,569 | +13.6% | $3,565 | −43% |
| SPY | +13.6% | $3,552 | +13.6% | $3,552 | −34% |
| Intraday | −3.4% | $709 | −4.4% | $637 | −48% (−52% after fees) |

Almost all of the stock market's return over this period came overnight. Holding stocks only during trading hours lost money.

**Per-stock view** (all 4,000 stocks, $1,000 in each, before costs):
- **Overnight:** made money in 82% of stocks.
- **Intraday:** made money in only 31%.
- **Best of the three:** overnight was the best strategy for 2,654 of 4,000 stocks.

### 2. Common stocks (3,500) vs 500 most traded

The same strategies on two universes: the roughly 3,500 common stocks, and the 500 with the highest median dollar volume.

<img width="940" height="648" alt="image" src="https://github.com/user-attachments/assets/309fde08-f570-45dc-9a09-6af419271fff" />

| Strategy | 3,500 common, before | 3,500 common, after fees | 500 most traded, before | 500 most traded, after fees |
|---|---|---|---|---|
| Overnight | **+19.2%** ($5,766) | **+18.0%** ($5,196) | +12.6% ($3,257) | +11.7% ($3,011) |
| Buy and hold | +14.2% ($3,753) | +14.2% ($3,749) | **+16.8%** ($4,691) | **+16.8%** ($4,688) |
| SPY | +13.6% ($3,552) | +13.6% | +13.6% | +13.6% |
| Intraday | −3.2% ($725) | −4.2% ($653) | +3.9% ($1,458) | +3.0% ($1,348) |

**Overnight only beats buy and hold in the broad universe.** Among the 500 most liquid stocks, overnight trails both buy and hold and SPY, even before costs.

**Break-even trading cost** for the overnight strategy (full round trip, 3,500 common stocks):
- To beat SPY: about **0.019%**
- To make money at all: about **0.07%**

### 3. Ranked strategy: all stocks

Each group holds about 326 stocks (range 228–397), because not every one of the 4,000 symbols has 15 trading days every month.

**Before costs**

<img width="940" height="405" alt="image" src="https://github.com/user-attachments/assets/91fbbc96-d776-4e36-b033-ff9486122b5f" />
<img width="940" height="464" alt="image" src="https://github.com/user-attachments/assets/7eec66d5-d963-481c-9618-f1aeec2d17da" />

**After fees**

<img width="940" height="377" alt="image" src="https://github.com/user-attachments/assets/30e7300b-986f-4b29-9dc3-5a85a337effc" />
<img width="940" height="439" alt="image" src="https://github.com/user-attachments/assets/5711644e-563b-496c-adb1-63b497d0eab6" />


| Portfolio (Nov 2016 to Sep 2026) | Overnight, before | Overnight, after fees | Intraday, before | Intraday, after fees |
|---|---|---|---|---|
| Group 1 (last month's top 10%) | **+75.3%** ($252,460) | **+72.7%** ($218,093) | +11.9% ($3,030) | +10.6% ($2,688) |
| Group 10 (bottom 10%) | +2.8% ($1,313) | +1.5% ($1,155) | −27.1% ($45) | −28.2% ($38) |
| Long 1–2 / short 9–10 | **+41.0%** ($29,490) | **+37.6%** ($23,200) | +27.9% ($11,263) | +24.8% ($8,883) |
| Long-short worst drop | −6.7% | −6.9% | −9.3% | −9.6% |
| Buy and hold | +14.0% ($3,637) | +14.0% | +14.0% | +14.0% |
| SPY | +13.8% ($3,584) | +13.8% | +13.8% | +13.8% |

**The groups line up almost perfectly in rank order.** This confirms the persistence in Lou, Polk & Skouras: stocks with high overnight returns last month keep earning high overnight returns.

**Commission sensitivity** (after fees, plus a commission on every buy and sell):

| Commission per trade | Overnight long-short | Intraday long-short | Overnight group 1 |
|---|---|---|---|
| 0% | +37.6% | +24.8% | +72.7% |
| 0.01% | +24.4% | +12.9% | +64.2% |
| 0.05% | −16.9% | −24.6% | +34.3% |

**The extreme groups are small, cheap and thinly traded.** Groups 1 and 10 have a median price of about $15–18 and dollar volume of about $5–6M. The middle groups are about $35 and $17M. Real spreads are widest exactly where the returns are highest.

### 4. Ranked strategy: 1,000 largest stocks

Each month the universe is the 1,000 largest common stocks by dollar volume, and each group holds 97–101 stocks.

**Before costs**

<img width="1193" height="503" alt="image" src="https://github.com/user-attachments/assets/4c8a536a-3477-4c47-a001-42a2c1057fb4" />
<img width="1132" height="539" alt="image" src="https://github.com/user-attachments/assets/dc362be7-acb6-4c25-abb3-5d0f3f31fbab" />

**After fees**

<img width="1184" height="504" alt="image" src="https://github.com/user-attachments/assets/e1ceda5e-f784-4358-9c84-88e6a97297c4" />
<img width="1132" height="536" alt="image" src="https://github.com/user-attachments/assets/0aec5cfd-db14-4574-a0a6-49fe9dc293fe" />

| Portfolio (Nov 2016 to Sep 2026) | Overnight, before | Overnight, after fees | Intraday, before | Intraday, after fees |
|---|---|---|---|---|
| Group 1 (top 100) | **+24.4%** ($8,562) | **+23.3%** ($7,866) | −0.3% ($971) | −1.1% ($894) |
| Group 10 (bottom 100) | +9.3% ($2,405) | +8.4% ($2,212) | −10.4% ($339) | −11.2% ($311) |
| Long 1–2 / short 9–10 | +10.5% ($2,675) | +8.7% ($2,271) | +5.2% ($1,652) | +3.5% ($1,403) |
| Long-short worst drop | −12.9% | −14.5% | −22.3% | −25.7% |
| Buy and hold (same 1,000) | +11.5% ($2,931) | +11.5% | +11.5% | +11.5% |
| SPY | +13.8% ($3,584) | +13.8% | +13.8% | +13.8% |

**Among large stocks, most of the edge disappears.**
- **Overnight:** last month's overnight winners (group 1) still beat SPY. But the bottom group no longer loses, so the long-short falls below both SPY and buy and hold.
- **Intraday:** the only clear signal left is that the bottom group keeps losing.

---

## Limitations

- **Survivorship bias:** the databased is 4000 of the largest companies by market cap listed in mid September, so companies that were delisted are not included. 
- **No dividends:** prices are split-adjusted but not dividend-adjusted, for both the stocks and SPY.
- **Opening price:** the open is the daily opening print. Lou, Polk & Skouras use the average traded price over the first half hour (9:30–10:00), which is less noisy. Noisy opens, especially in thin stocks, may partly create the overnight pattern. But limited by intraday data accessibility.
- **Size proxy:** market cap is approximated by dollar volume. The "500 most traded" universe ranks on median dollar volume over the whole period, which uses future information. The top-1,000 variant uses only the ranking month.
- **Costs:** the headline after-cost results include regulatory fees only which is realistic given zero-commission brokers. But the model also leave out bid-ask spreads, any potential market impact of each trade, short borrow cost, margin interest and taxes (daily trading produces short-term gains).
- **±50% filter:** this rule removes obvious data errors but can also remove some genuine moves.
- **Statistical significance:** not yet tested formally.

---

## Repository layout

| Path | Purpose |
|---|---|

---

## How to run


## References

- Lou, D., Polk, C., & Skouras, S. (2019). *A tug of war: Overnight versus intraday expected returns.* Journal of Financial Economics, 134, 192–213.
