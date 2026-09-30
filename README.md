# Mutual Fund Analyzer (India)

A single-file HTML app (`mutual-fund-analyzer.html`) that loads Indian mutual fund NAV data from MFapi.in, calculates risk and return metrics in the browser, and shows category leaderboards, a composite scorecard, charts and a head-to-head comparison.

It covers six equity categories: **Large Cap, Large & Mid Cap, Mid Cap, Small Cap, Flexi Cap, Multi Cap**, Direct Growth plans only.

---

## 1. Data source

| Item | Detail |
|---|---|
| Provider | MFapi.in, a free API built on AMFI data. No key or login. |
| Base URL | `https://api.mfapi.in/mf` |
| List all schemes | `GET /mf?limit=N&offset=M` |
| Latest NAV and category | `GET /mf/{schemeCode}/latest` |
| NAV history | `GET /mf/{schemeCode}?startDate=YYYY-MM-DD` |

The `/mf/search` endpoint is **not used**. It returns only about 15 results per query, ordered by oldest scheme code first, so it cannot list a whole category.

Nothing is stored on a server. Every load fetches fresh data and all calculations run in your browser.

---

## 2. How schemes are listed

Pressing **Refresh data** runs three steps. The status line shows progress.

### Step 1: list every scheme
- Requests `/mf?limit=2000&offset=0`. If the API rejects that size, it retries with `limit=100`.
- The page size is taken from the first response. Further pages are then requested six at a time until a short or empty page is returned.
- If a later page fails, loading stops there and the summary warns that the list may be incomplete.

### Step 2: shortlist and verify
1. **Name filter.** A scheme is kept only if its name contains "Direct" and "Growth", and does not match the exclusion pattern: IDCW, dividend, bonus, regular, ETF, index, Nifty, Sensex, FoF, segregated, payout, reinvest, "fund of".
2. **Broad prefilter.** The name must also match a wide pattern (`cap`, `blue chip`, `top 100`, `emerging`, `flexi`, `opportunit`, `frontier`, `equity`, `discovery`). This only limits how many schemes get a verification call. It does not decide the category.
3. **Category from the API.** Each remaining scheme is checked with `/latest`. Its `scheme_category` field decides the category:
   - must start with "Equity"
   - "Large & Mid" or "Large and Mid" becomes **Large & Mid Cap**
   - otherwise Large Cap, Mid Cap, Small Cap, Flexi Cap or Multi Cap by keyword
   - anything else (hybrid, sectoral, ELSS and so on) is dropped

   Because the API's own label is used, funds whose names do not include the category, such as "Bluechip" funds, are still classified correctly.
4. **Freshness.** A scheme whose latest NAV is more than 25 days old is skipped, which removes merged and closed funds.

### Step 3: download history
- If **Funds per category** is 10 or 25, funds are ranked by a fixed list of large fund houses first, then alphabetically, and the first N per category are kept. **All** keeps everything.
- NAV history is fetched with `startDate` set to about 5.1 years ago. If that parameter fails, the full history is fetched instead.
- Requests run eight at a time. A fund with fewer than 60 NAV points is skipped.

The final status line reports how many funds were loaded per category and how many were skipped and why.

---

## 3. Method of analysis

All figures come from NAV history only. Windows are measured back from each fund's latest NAV date.

### Per-fund metrics

| Metric | How it is calculated |
|---|---|
| 1Y / 3Y / 5Y return | Compound annual growth between the latest NAV and the NAV on or just before the start date. Shown only if the history reaches back to that date (10-day tolerance). |
| Volatility | Standard deviation of daily returns over the last 3 years, annualised by multiplying by the square root of 252. |
| Sharpe | (annualised mean daily return minus 6.5%) divided by volatility. Needs at least 200 daily observations. |
| Sortino | Same numerator, divided by annualised downside deviation (daily returns below the daily risk-free rate). |
| Max drawdown | Worst peak-to-trough fall of NAV over the last 3 years. |
| Age | Years from the first NAV loaded. Capped near 5 because only 5 years are downloaded. |

The risk-free rate is a constant, `RF = 6.5` in the code. Change it there if you prefer another value.

### Metrics versus the category average

The **category average** is the equal-weighted average daily return of the loaded funds in the same category. A date is used only if at least 3 funds have a return on it. This is a proxy benchmark, not an official index.

| Metric | How it is calculated |
|---|---|
| Beta | Covariance of the fund's daily returns with the category average, divided by the variance of the category average. Needs more than 150 common days. |
| Alpha | Annualised: (fund mean return minus risk-free) minus beta times (category mean return minus risk-free). |
| Downside capture | Average fund return divided by average category return, on days the category average was negative (times 100). Below 100 means the fund fell less. Needs more than 20 such days. |
| Consistency | Share of the last 36 months in which the fund's monthly return beat the category average. Needs at least 12 months. |

The fund itself is included in its category average. With few funds loaded, this pulls alpha, beta and consistency toward neutral.

### Composite score (0 to 100)

Each component is the fund's **percentile rank within its category**, then weighted:

| Component | Weight |
|---|---|
| 3Y return | 30% |
| Sharpe | 25% |
| Downside (average of max drawdown percentile and downside capture percentile) | 20% |
| Consistency | 15% |
| Alpha | 10% |

A missing value gets a neutral percentile of 0.5, so funds with short histories are not punished or rewarded for data they don't have. Percentiles depend on which funds are loaded, so scores change if you change **Funds per category**.

### Filters and flags
- **Top quartile:** score at or above the 75th percentile of its category. Applied only where the category has at least 4 funds.
- **Emerging (<3Y):** age under 3 years. These funds have no 3Y or 5Y return and are shown with a badge.

### Charts and comparison
- **Category dispersion:** each dot is a fund's 3Y return. The dark tick is the category median.
- **Return vs volatility:** 3Y return against volatility.
- **Radar (up to 4 funds):** five axes scaled to fixed ranges: alpha (-4 to 6), low beta, low volatility relative to the category, Sharpe (0 to 1.6), and downside protection (from downside capture). A larger area is better.
- **Return correlation:** correlation of daily returns over common days (at least 100). It shows how similarly two funds behave. It is **not** holdings overlap, which needs portfolio disclosures that NAV data does not contain.

### Executive summary and recommendations
Both are generated by rules from the funds currently in view, for example the top scorer, best Sharpe, shallowest drawdown, funds with downside capture above 105, and funds at least 15% more volatile than their category. They are not opinions about any specific investment.

### Chat box
It is a keyword lookup over the loaded rows (best Sharpe, lowest drawdown, consistency, emerging funds, best return, top score, or a fund name). No external model is used and no data leaves your browser except the MFapi requests.

---

## 4. Limitations

- **Only NAV-derived metrics.** Expense ratio, Morningstar ratings and pillars, active share, holdings and manager history are not in the data, so they are not shown.
- **Survivorship.** Only currently active funds are listed, so closed or merged funds are not in the comparison. Past averages are therefore somewhat flattering.
- **Proxy benchmark.** Alpha, beta, downside capture and consistency are relative to the loaded funds, not to an index such as the Nifty 50.
- **Age is capped** at about 5 years.
- **Data timing.** MFapi.in updates daily from AMFI. The header shows the latest NAV date and the time you fetched.
- **Not investment advice.** Past performance does not guarantee future results.

---

## 5. Running it

### Locally in VS Code
1. Save the file as `mutual-fund-analyzer.html`.
2. Install the **Live Server** extension, right-click the file and choose **Open with Live Server**.
3. Press **Refresh data**. A full load makes several hundred requests and downloads roughly 10 MB, so it can take a minute or two. Choose 10 or 25 funds per category for a faster load.

### Streamlit Community Cloud
Create `app.py` next to the HTML file:

```python
import streamlit as st, streamlit.components.v1 as c
st.set_page_config(page_title="Mutual Fund Analyzer", layout="wide")
c.html(open("mutual-fund-analyzer.html", encoding="utf-8").read(), height=2800, scrolling=True)
```

Add `streamlit` to `requirements.txt`, push the three files to GitHub, and deploy from share.streamlit.io. The HTML can also be hosted as-is on GitHub Pages or Netlify.

### Troubleshooting
| Symptom | Likely cause and fix |
|---|---|
| "Could not load fund data" | The browser or the host is blocking requests to `api.mfapi.in` (the Claude preview page does this). Run it locally or on GitHub Pages or Streamlit. |
| Fewer funds than expected | Read the status line for skipped counts. A page failure during listing means the list may be incomplete: press Refresh again. |
| Slow load | Reduce **Funds per category** to 10 or 25. |

---

## 6. Where to change things in the code

| What | Where |
|---|---|
| Risk-free rate | `RF` constant near the top of the script |
| Excluded plan types | `BAD` regular expression |
| Shortlist prefilter | `PRE` regular expression |
| Category mapping | `CATMAP` |
| Preferred fund houses when limiting funds | `PREF` list |
| Freshness cut-off (25 days) | `build()` and the `load()` step 2 check |
| Score weights | `stage2()`, the line that sets `f.score` |
