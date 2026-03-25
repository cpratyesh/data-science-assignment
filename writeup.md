# Write-Up — Trader Performance vs Market Sentiment
**Primetrade.ai Intern Assignment**

---

## What I Did

I started by just loading both datasets and checking if they'd actually align cleanly — they did, which was a relief. The trader data covers May 2023 to May 2025, and the Fear/Greed index has daily entries going back to 2018, so there was no issue with coverage gaps.

One thing I noticed early: a lot of rows have `Closed PnL = 0`. That's not missing data — it's open trades that haven't been closed yet, or trades where fees exactly offset gains. I kept them in for trade count and behavioral metrics, but filtered them out when computing win rates so I wasn't inflating or deflating the numbers.

I aggregated everything to daily per-account level, which felt like the right granularity — individual trades are too noisy, and weekly would've lost too much signal. I also simplified the 5-class sentiment (Extreme Fear → Greed) into 3 groups for cleaner comparisons, but kept the full 5-class breakdown for the heatmap since that's where the nuance actually shows up.

---

## What Surprised Me

Honestly, I expected Greed days to be better for traders — more confidence, bigger moves, more opportunity. The data says the opposite.

**Fear days average $5,185 PnL vs $4,144 on Greed days.** That's a 25% difference and it held consistently across months, not just a few outlier days. My best guess is that Fear periods come with higher volatility, so the traders who stay disciplined get paid more per trade even if they're not winning more often — win rates are basically the same (~36%) across all sentiment regimes.

What really caught my attention was the **long bias flipping**. On Fear days traders are more long (35.8%) than on Greed days (31.8%). That feels backwards until you think about it — these are active traders on a derivatives platform, not retail sentiment followers. They're fading the crowd. That's probably a big part of why Fear days pay out better.

The other thing I didn't expect: **trade frequency is 37% higher on Fear days**. More trades, bigger PnL, more bullish — during the days everyone else is panicking. That pattern is hard to ignore.

---

## Segmentation

I ran K-Means with k=3 on account-level features (total PnL, trade count, win rate, avg size, long ratio). The three clusters that came out made intuitive sense:

- **Consistent Winners (16 accounts)** — ~45% win rate, moderate trade count, stable across all sentiment regimes. These look like systematic traders. Their performance barely changes between Fear and Greed, which to me suggests they're not reacting to sentiment at all — they have rules and they follow them.
- **High Frequency (5 accounts)** — massive trade volume, biggest absolute PnL, but more volatile. Their edge seems to come from volume, not from being right more often.
- **Cautious Traders (11 accounts)** — lower size, lower frequency, and noticeably worse on Fear days specifically. They seem to pull back exactly when the opportunity is highest.

---

## Strategy Ideas

**1. Treat Fear as a signal to be more active, not less**
The data pretty clearly shows that pulling back during Fear is the wrong move — at least for this trader base. A simple rule: when Fear/Greed drops below 35, don't reduce position count. If anything, maintain or slightly increase long exposure. The Cautious segment seems to be doing the opposite and it's costing them.

**2. Greed days are a good time to be picky**
When the index goes above 70, trade frequency naturally drops in this dataset — and that seems correct. Forcing trades during euphoric markets doesn't seem to help win rate (it barely moves). High Frequency traders specifically might benefit from a hard cap on daily trade count during Greed phases, something like "no more than X new positions when index > 70."

---

## Model

I threw together a Random Forest to predict whether tomorrow would be profitable, using today's sentiment score, trade count, win rate, position size, long ratio, PnL and fees. Got **70% accuracy** on the test set, which is better than I expected honestly. The most important features were today's PnL and win rate — so recent performance is a stronger signal than sentiment alone. Sentiment (the fear/greed value) was mid-tier in importance, which actually makes sense given that the behavioral differences between regimes aren't massive, just consistent.

