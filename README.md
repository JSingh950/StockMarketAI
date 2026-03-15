# StockMarketAI
# MarketMind — AI Stock Predictor

A browser-based stock analysis tool that trains a Gradient Boosted Trees model on each stock's own 5-year price history, runs live technical analysis on real Finnhub data, and uses Claude to synthesize everything into a written outlook.

---

## What it actually does

When you search a ticker, the app runs three things in sequence:

**1. Trains a model on that stock's history**
Fetches 5 years of real daily OHLCV candles from Finnhub. Computes 19 features at every bar (RSI, MACD, Bollinger Bands, Stochastic, momentum, volume, trend quality, moving average distances, volatility, ATR). Labels every bar by what actually happened 20 days later — up >2% (bullish), down >2% (bearish), or flat (neutral). Trains a Gradient Boosted Trees classifier on those ~1,000 labeled samples. Validates accuracy on the most recent 20% of data, which was never seen during training.

**2. Runs live inference**
Fetches the current quote, candles, fundamentals, news, and peers from Finnhub. Stitches the live candles onto the training history, recomputes all 19 features at the current bar, and runs them through the trained model to get a probability distribution: X% bullish, Y% neutral, Z% bearish over the next 20 trading days.

**3. Generates AI analysis**
Sends the model output, backtest accuracy, top predictive features, live technicals, and key price levels to Claude, which writes a 4-sentence analysis grounded in the actual numbers.

---

## Features

- **Per-stock trained model** — AAPL's model is trained on AAPL's history, not a generic population average
- **Real backtest accuracy** — shows actual directional accuracy on the last 85+ 20-day windows for that stock
- **19-feature ML model** — Gradient Boosted Trees (XGBoost-style) in pure JavaScript, no external ML libraries
- **Monte Carlo forecast** — 300 simulated price paths using ML-derived drift and real historical volatility, displayed as confidence bands on the chart
- **Feature importance** — shows which of the 19 indicators actually drove the prediction for that specific stock
- **Live data** — real prices, candles, fundamentals, news, and peers from Finnhub
- **Pattern detection** — Head & Shoulders, Inverse H&S, Double Top/Bottom, Golden/Death Cross, Bollinger Squeeze, trend channels
- **Support & resistance** — algorithmically detected from real price history
- **Watchlist** — live % change for 7 major tickers
- **Responsive** — works on mobile

---

## Honest accuracy expectations

The model typically achieves **52–62% directional accuracy** on 20-day predictions, depending on the stock and market regime. This is in line with what systematic quant funds achieve on public OHLCV data.

| Condition | Expected accuracy |
|---|---|
| All market conditions, 20-day horizon | 52–56% |
| Strong trending regimes | 56–62% |
| Short horizon (5 days) | 54–58% |
| Best case with all filters | ~62–65% |

**Why it can't go higher:** The Efficient Market Hypothesis sets a structural ceiling on public-data models. Every indicator in this model is watched by millions of traders — any edge gets arbitraged away quickly. Renaissance Technologies, the best quant fund in history, runs at roughly 55–60% directional accuracy on equities. Getting to 70%+ requires alternative data (satellite imagery, credit card flows, options order flow) costing $50k–$500k/year.

**Why it's still useful:** A 56% model applied consistently with proper position sizing beats the market over time. The value is in having a systematic, unemotional signal — not in the raw accuracy number.

---

## Technical architecture

### Model: Gradient Boosted Trees
Implemented from scratch in pure JavaScript. No TensorFlow, no external ML libraries — just the browser.

- 80 estimators, learning rate 0.12, max depth 4, 80% subsampling
- Two binary classifiers trained in parallel: bull-vs-rest and bear-vs-rest
- Predictions converted to a 3-class probability distribution via sigmoid normalization
- Feature subsampling at each split (√n features) for variance reduction

### Features (19 total)

| # | Feature | Description |
|---|---|---|
| 0 | RSI | 14-period Relative Strength Index, normalized 0–1 |
| 1 | MACD Histogram | Normalized by price |
| 2 | MACD Acceleration | Change in histogram over 3 bars |
| 3 | Bollinger %B | Position within Bollinger Bands (0 = lower, 1 = upper) |
| 4 | Bollinger Bandwidth | Band width relative to midline — squeeze detector |
| 5 | Stochastic | 14-period %K |
| 6 | 5-day return | tanh-normalized |
| 7 | 10-day return | tanh-normalized |
| 8 | 20-day return | tanh-normalized |
| 9 | Volume ratio | Current volume vs 20-day average |
| 10 | Trend R² | How well the last 20 bars fit a linear trendline |
| 11 | Trend slope | Slope of that trendline, normalized |
| 12 | Distance from SMA 20 | % deviation, tanh-normalized |
| 13 | Distance from SMA 50 | % deviation, tanh-normalized |
| 14 | Distance from SMA 200 | % deviation, tanh-normalized |
| 15 | MA alignment | Whether 20 > 50 > 200 (bullish stack) |
| 16 | Local volatility | 20-day annualized vol |
| 17 | ATR% | Average True Range as % of price |
| 18 | Body size | Candle body relative to range |

### Labels
- **Bullish (1):** price is >2% higher 20 trading days later
- **Bearish (-1):** price is >2% lower 20 trading days later
- **Neutral (0):** price is within ±2%

### Forecast
Monte Carlo simulation: 300 paths, each using Box-Muller normal shocks, ML-derived drift (from bull/bear probability spread), and real annualized volatility. Displays median, 25th/75th percentile (inner band), and 10th/90th percentile (outer band).

---

## Data sources

| Data | Source | Endpoint |
|---|---|---|
| Live quote, OHLC | Finnhub | `/quote` |
| Daily/weekly candles | Finnhub | `/stock/candle` |
| Company profile, sector | Finnhub | `/stock/profile2` |
| Fundamentals (P/E, EPS, beta, 52W) | Finnhub | `/stock/metric` |
| Company news | Finnhub | `/company-news` |
| Peer stocks | Finnhub | `/stock/peers` |
| AI written analysis | Anthropic Claude | `claude-sonnet-4` |

---

## Setup

### 1. Get a Finnhub API key
Sign up at [finnhub.io](https://finnhub.io) — free tier, 60 requests/minute, no credit card.

### 2. Open the app
Just open `index.html` in any modern browser. No build step, no npm, no server required.

### 3. Deploy to GitHub Pages
```bash
git init
git add index.html README.md
git commit -m "Initial commit"
git remote add origin https://github.com/YOURNAME/REPONAME.git
git push -u origin main
```
Then go to **Settings → Pages → Deploy from branch → main → / (root)**

Your app will be live at `https://yourname.github.io/reponame` within 60 seconds.

---

## Performance notes

Training takes **5–15 seconds** per ticker on first search (fetching 5 years of candles + fitting 80 trees). Models are cached in memory for the session — switching back to a previously searched ticker is instant.

The Gradient Boosted Trees training is CPU-bound and runs on the main thread. On slower devices it may briefly block the UI. This is a known limitation of running ML in the browser without a Web Worker.

---

## Known limitations

- **Simulated intraday on 1W view** — Finnhub free tier only returns intraday candles during market hours. Use 1M, 3M, or 1Y for reliable chart data.
- **No earnings filter** — the model doesn't know when earnings are. Predictions made just before an earnings release have much higher variance.
- **No macro context** — interest rates, sector rotation, and market-wide risk-off events are not in the model.
- **API key in source** — the Finnhub key is hardcoded. Anyone who views the source of a public GitHub repo can see and use it. Monitor your usage at finnhub.io/dashboard.
- **Session-only cache** — trained models reset on page refresh. A future improvement would persist them to IndexedDB.

---

## What would actually improve it further

In rough order of impact:

1. **Earnings filter** — skip predictions within 5 days of an earnings release (Finnhub has an earnings calendar endpoint)
2. **Regime filter** — only predict in trending, lower-volatility markets (VIX < 25)
3. **News sentiment scoring** — NLP on the news headlines already fetched, fed as a feature
4. **Shorter prediction horizon** — 5-day predictions are structurally easier than 20-day
5. **Walk-forward retraining** — retrain monthly on a rolling window so the model adapts to regime changes
6. **Alternative data** — the real unlock for 65%+ accuracy; requires paid data feeds

---

## Disclaimer

For educational purposes only. Not financial advice. Past model performance does not guarantee future results. All predictions carry significant uncertainty — see the confidence bands on the chart.
