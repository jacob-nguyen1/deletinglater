# Kalshi Daily-High Favourite Bot: Design

A rules-based bot for Kalshi's daily high-temperature markets. It uses no weather model: it buys the market's own favourite bracket when the market is confident, and holds it to settlement. The edge is the favourite-longshot bias. Favourites settle YES somewhat more often than their price implies, and longshots less often.

## 1. Markets

- **Series:** Kalshi daily high temperature (`KXHIGH*`), one event per city per day. The brackets are 2°F ranges plus two open-ended tails.
- **21 stations:** New York (Central Park), Chicago (Midway), Miami, Austin, Los Angeles, Denver, Houston (Hobby), Dallas/Fort Worth, Atlanta, Seattle, Phoenix, Philadelphia, Minneapolis, San Antonio, New Orleans, Las Vegas, San Francisco, Boston, Oklahoma City, Louisville, Washington DC (Reagan).
- **Settlement:** the NWS Daily Climate Report (CLI) maximum. The climate day runs midnight to midnight in local **standard** time, which is 1 AM to 1 AM on the clock during daylight time.
- **Highs only.** Daily *low* markets backtested well but lost money live (9 of 16 won against ~80% break-even), and the backtest has no fall data for them. Don't trade them.

## 2. The rule

| Step | Setting |
|---|---|
| Decision time | **7 hours before 09:00 local standard time**, i.e. 02:00 local standard time (03:00 on the clock during daylight time) |
| Pick | Read the fresh order book of every bracket in the event and take the bracket with the highest mid ((bid + ask) / 2) |
| Buy only if | mid ≥ **65¢** and ask ≤ **95¢** and spread (ask − bid) ≤ **8¢** |
| Order | Buy YES, immediate-or-cancel, limit = **ask + 1¢**. Accept partial fills and don't chase. |
| Exit | None. Hold to settlement. |
| Frequency | One order per event, ever |

The 09:00 anchor is when the day's heating starts to settle the outcome. Before that, prices mostly reflect forecasts. A decision window of about 20 minutes after decision time is enough. If the machine is asleep or offline then, skip the event.

## 3. Evidence

**Method:** hourly Kalshi candles and settled results, July 2025 to September 2026. The period is split into select (Jul 2025 to Apr 2026), validate (May to mid-Jul 2026) and test (mid-Jul to Sep 2026). Rules were fixed on select and checked on the later periods. Uncertainty comes from a bootstrap over whole trading days, because same-day bets share weather surprises. An edge only counts if the whole 95% range is above zero.

**Overall:** 426 qualifying bets. **81.7% won**, against a break-even of about 77% (the average price paid plus fee).

**By price band (mid at decision time):**

| Favourite's price | Bets | Won | Return per $ staked |
|---|---|---|---|
| 65–69¢ | 152 | 73% | +5.1% |
| 70–74¢ | 93 | 75% | +1.8% |
| 75–79¢ | 54 | 83% | +5.6% |
| 80–84¢ | 45 | 98% | +17.4% |
| 85¢+ | 82 | 95% | +4.9% |

- Below a 65¢ mid the rule loses money: 60–64¢ came out −4.4% per $ in the select period.
- Brackets under 40¢ lose about 3.5¢ per contract (the longshot side of the bias).
- Spreads of 9¢ or more lost in every period.

**Lead time** (2026, profit per contract by hours before 09:00):

| 5h | 6h | 7h | 8h | 12h | 16h |
|---|---|---|---|---|---|
| +2.9¢ | +4.7¢ | +4.6¢ | +4.8¢ | +7.3¢ | +7.8¢ |

It's a broad plateau from about 6 to 16 hours. Earlier decisions earn more per bet but qualify fewer events, so total profit is about the same. Inside 2 hours the edge is gone.

**Live results** for this exact rule (65¢ floor, roughly 80 settled bets): about 82% won, in line with the backtest.

## 4. Why 7 hours (not 6)

The order books are thin. The typical book holds about $40 of contracts within 1¢ of the ask, and 1 in 10 books has under $3 at the best ask.
- Two bots firing in the same minute split that liquidity. Replaying 116 real books, the second order in line fills 75–85% of a $7–14 order instead of 89–94%, and pays a bit more. At $30 per order it fills only about half.
- Books refill fast. After 151 real fills, the price was unchanged and the depth back to normal within 15 minutes.
- So deciding an hour apart avoids the clash, and the edge at 7h is about the same as at 6h.

## 5. Sizing (flexible)

Stake a fixed **fraction of available cash** per bet, recomputed at every order, so bets shrink after losses and grow after wins. Use fractional contracts (0.01 steps).

**Reference setting:** 5% of cash when the mid is ≥ 75¢, and **2.5% when it's 65–74¢**. The 65–74¢ band's edge has been shrinking: +10.7% per $ in select, +3.5% in validate, +1.2% in test, and about 0% live. The 75¢+ band has held up.

**One-year simulations from $300** (2000 bootstraps of trading days):

| Sizing | Median after 1 yr | Chance > $1,000 | Chance of ever dropping below $215 | Typical worst drop |
|---|---|---|---|---|
| 5% on all bets | $1,537 | 75% | 10% | 32% |
| 5% / 2.5% (reference) | $1,136 | 62% | 1% | 20% |
| Only bets ≥ 75¢, 5% | $747 | 11% | 0% | 13% |

These use the full backtest. Using only the most recent quarter, which has the weaker 65–74¢ edge, the reference setting comes out best: median $877, against $784 for the other two.

**Guidance:**
- 2–5% per bet is the sensible range.
- At 10% or more, drawdowns roughly double for little extra growth. Each bet's ceiling is also set by book depth (see section 4), not by cash.
- Bets in the same region on the same day (e.g. several Texas cities) did *not* lose together in the backtest, so no regional limit is needed. Bad days were nationwide, and fractional sizing already covers them.

## 6. Costs and mechanics

- **Fee:** ceil(0.07 × P × (1 − P)) dollars per contract (1–2¢ at these prices). Break-even win rate = price + fee.
- **API:** Kalshi Trade API v2 with signed requests (key ID + private key). The private key stays on the machine running the bot. Fresh order books come from the batch `markets/orderbooks` endpoint. The YES ask = 100 − best NO bid.
- **Balance:** weather markets trade on exchange shard 0, so size from that shard's balance, not the total.
- **Duplicates:** derive `client_order_id` from the event ticker (e.g. a UUID5), so a retry or restart can't place a second order.
- **Maintenance:** Kalshi pauses trading Thursdays 3–5 AM ET. That hits the East Coast decisions during daylight time, so retry until trading resumes.
- **Uptime:** the machine must be awake and online at decision time. A missed window means a skipped bet, not a late one.

## 7. Risks and known weak spots

- **The edge is small and noisy.** The per-bet standard deviation of return is about 50%. A 20-bet stretch can swing ±23 points of win rate, so judge the rule on 100+ bets, not on a week.
- **The payoff is lopsided.** A win at 70¢ pays about $0.30 per $1 risked, and a loss costs the whole $1. Expect frequent small wins and occasional losses worth 3–4 wins.
- **Seasonality.** Win rates were highest in spring (~90%), lower in summer (72–77%), and December 2025 was negative (−15% per $). Winter is the least-tested season.
- **Edge decay.** More bots in these markets will compress the favourite premium. The 65–74¢ band already shows it.
- **Capacity.** At ask + 1¢, the whole strategy supports roughly $4–5k of profit per year across everyone running it. Bigger orders walk the book and eat the edge.

## 8. Safety rails

- Dry-run mode by default; real orders only with an explicit flag.
- Kill switch: a `STOP` file in the working directory blocks all orders.
- A hard max stake per order as a bug seatbelt, and a daily order cap (history never had more than ~13 qualifying events in a day).
- Pause automatically if total account value (cash + open positions) falls below about 2/3 of what was deposited, then review.
- Review results monthly against the backtest bands. Don't re-tune the rule on a few weeks of live results; a new idea becomes a separate, separately tracked rule.
