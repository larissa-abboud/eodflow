# Finance concepts used in EODFlow

Plain-language notes. Know these well enough to explain them in an interview.

## Trade
An agreement to **buy** or **sell** a quantity of an instrument (e.g. a share identified by its ISIN) at a price, on a date, inside a **portfolio** (a "book").

## Position
The net quantity you hold after all trades.
Buy 100, buy 50, sell 30 → **position = +120** (long).
A negative position is **short**: you sold something you didn't own, and you gain if the price falls.

## Mark-to-market (revaluation)
Valuing a position at **today's market price**, not at the price you paid.
`market value = position × closing price`
120 shares × €50 = **€6,000**.

## P&L (profit and loss)
How much the portfolio gained or lost **today**.
- **Unrealised P&L** comes from price moves on positions you still hold:
  `position × (today's price − yesterday's price)` → 120 × (50 − 48) = **+€240**
- **Realised P&L** is locked in when you sell: `quantity sold × (sell price − average cost)`
- Daily P&L = unrealised + realised. (This project starts with unrealised P&L only.)

## Value-at-Risk (VaR): optional extra, but know the idea for the interview
"With **95%** confidence, the portfolio will **not lose more than X** in **one day**."

**Historical VaR** (the simplest method to explain):
1. Take the last **250** daily price returns for each instrument: `return = (P_t / P_t-1) − 1`.
2. For each of those 250 days, compute what **today's portfolio** would have gained or lost if that day happened again: `Σ position × today's price × return`.
3. You now have 250 hypothetical P&Ls. Sort them from worst to best.
4. The 95% VaR is the loss at the **5th percentile**: roughly the 13th worst value (5% of 250 = 12.5).

Example: if the 13th worst hypothetical P&L is −€1,800, then **1-day 95% VaR = €1,800**.

## End-of-day (EOD) processing
After markets close, banks run a batch: load closing prices → revalue all positions → compute P&L → compute risk → produce reports for traders, risk managers and regulators. Steps depend on each other, which is why it is modelled as a **dependency graph**. Running it fast and reliably every night is a real engineering problem.

## Why `BigDecimal`, not `double`
`0.1 + 0.2` in `double` is `0.30000000000000004`. In finance, rounding errors add up across millions of trades. `BigDecimal` gives exact decimal arithmetic with an explicit rounding mode.

## Where this connects to Murex
MX.3 covers **trading** (capturing trades), **risk management** (P&L, VaR, sensitivities) and **processing** (lifecycle, settlement, EOD batches) for banks and asset managers. EODFlow is a tiny, simplified model of the trading → risk → EOD part.
