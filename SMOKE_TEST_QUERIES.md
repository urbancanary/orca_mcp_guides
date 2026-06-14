# Ask Orca — Smoke-Test Query Set (Tom's marketing app)

50 plain-English queries a marketing / client-service user (Tom) would conceivably type into the **Ask Orca** app, with the **expected answer** and **expected display type** for each. Hand this to a smoke-testing agent to run against Orca and flag where the responses don't match.

The fund under test is **wnbf** (Wealthy Nations Sovereign IG Bond Fund — emerging-market sovereign investment-grade bonds), `client_id = "guinness"`.

## How to call

```
POST https://orca-mcp.x-trillion.com/call
{
  "tool": "orca_query",
  "args": { "query": "<the question>", "client_id": "guinness" },
  "include_display": true
}
```

### The display flag (important)
**Orca has a display flag.** When `include_display: true` is set, Orca returns a `display` object with **ready-to-draw typed blocks** instead of (or alongside) raw data:

| Block `type` | What it is | Tom's app draws it as |
|---|---|---|
| `metric_strip` | headline figures (Yield, Duration, Cash %…) | a small metrics table |
| `chart` (`chart_type: pie\|bar`) | a series to plot | a chart |
| `table` | columns + rows | a table |
| `factsheet` | label/value rows for one entity (a bond) | a fact sheet |
| `text` | a narrative answer | a paragraph |

So the smoke test should check **two things per query**:
1. **Right answer** — does the content actually answer the question?
2. **Right block type** — for a "show me a chart" question, did Orca return a `chart` block? For "table", a `table`? For a single bond, a `factsheet`? A question that wants a number should not come back as a 5-block portfolio dump.

### What "pass" means
- ✅ answers the question, in the **expected display type**, with sensible values (consistent decimals, real numbers).
- ⚠️ answers but **wrong block type** (e.g. returns the full portfolio dashboard for a narrow question — the current known failure, backlog #1647).
- ❌ errors / `no-data` / leaks a raw error string into a text block (backlog #1646), or returns nothing.

### Known current failure modes to watch
- Orca tends to return the **same full 5-block payload** (snapshot + country table/pie + rating table/pie) for *any* broad portfolio query, and **errors on narrow/facet-specific wording** (#1647).
- Some queries leak `Error: 'country'` into a `text` block instead of a clean error (#1646).
- No working **as-of / point-in-time** parameter yet (#1643).
- Charts should carry the **real % weight**, not a renormalised share.

---

## The 50 queries

### A — Portfolio snapshot / overview
| # | Query | Expected answer | Expected block |
|---|---|---|---|
| 1 | What is the yield and duration of wnbf? | Yield ≈ 5.71%, Duration ≈ 9.29y | `metric_strip` |
| 2 | How much cash is held in wnbf? | Cash ≈ 2.27% of portfolio | `metric_strip` |
| 3 | What is the total value (NAV) of wnbf? | Total portfolio value (currency figure) | `metric_strip` / `text` |
| 4 | How many holdings are in wnbf? | A count (≈ number of bonds) | `metric_strip` / `text` |
| 5 | Give me a portfolio snapshot of wnbf. | Yield, duration, cash, NAV, # holdings, avg rating | `metric_strip` |
| 6 | What is the average credit rating of wnbf? | Avg rating (e.g. A3/BAA1) / WARF | `metric_strip` / `text` |

### B — Country / geographic exposure
| # | Query | Expected answer | Expected block |
|---|---|---|---|
| 7 | What are the largest country exposures in wnbf? | Top countries by weight (Mexico ≈19.6%, Saudi ≈18.1%…) | `chart` (pie) + `table` |
| 8 | Show the country allocation for wnbf as a chart. | Same country weights, plotted | `chart` (pie) |
| 9 | Give me the country weights as a table. | Country / % / value rows | `table` |
| 10 | Which country is the biggest position in wnbf? | Single answer (Mexico ≈19.6%) | `text` / `metric_strip` |
| 11 | How much of wnbf is in Latin America? | Aggregated regional % | `text` / `metric_strip` |

### C — Credit rating / quality
| # | Query | Expected answer | Expected block |
|---|---|---|---|
| 12 | Show the rating allocation for wnbf. | Rating buckets (BBB ≈25.9%, AA ≈22.7%…) | `chart` (pie) + `table` |
| 13 | Rating profile of wnbf (as a table). | Rating / % / value / IG-HY bucket | `table` |
| 14 | What percentage of wnbf is below investment grade? | Sub-IG % (sum of HY buckets) | `text` / `metric_strip` |
| 15 | What is the WARF of wnbf? | WARF figure | `metric_strip` / `text` |
| 16 | Is wnbf mostly investment grade? | Yes + IG % | `text` |

### D — Yield / income
| # | Query | Expected answer | Expected block |
|---|---|---|---|
| 17 | What is the yield on wnbf? | ≈ 5.71% | `metric_strip` / `text` |
| 18 | Which bonds have the highest yield? | Top-yield holdings | `table` |
| 19 | What is the average spread of wnbf? | Avg spread (bp) | `metric_strip` / `text` |
| 20 | What income does wnbf generate? | Running yield / coupon income | `text` / `metric_strip` |

### E — Duration / maturity / rate risk
| # | Query | Expected answer | Expected block |
|---|---|---|---|
| 21 | What is the duration of wnbf? | ≈ 9.29y | `metric_strip` / `text` |
| 22 | Show the maturity profile of wnbf. | Maturity buckets | `chart` (bar) / `table` |
| 23 | What is the longest-dated bond in wnbf? | Bond with max maturity | `factsheet` / `text` |
| 24 | How sensitive is wnbf to interest rates? | Duration + plain explanation | `text` |

### F — Specific bonds (ISIN / issuer)
| # | Query | Expected answer | Expected block |
|---|---|---|---|
| 25 | Tell me about ISIN XS1709535097. | Bond fact sheet (issuer, coupon, maturity, price, yield, rating, country) | `factsheet` |
| 26 | What details do you have on US168863CE60? | Bond fact sheet | `factsheet` |
| 27 | What is the yield on XS1709535097? | Single bond yield | `metric_strip` / `text` |
| 28 | What is the price of US168863CE60? | Single bond price | `metric_strip` / `text` |
| 29 | What is our position in Pemex? | Issuer holding(s) | `table` / `factsheet` |
| 30 | Show the Mexican government bonds we hold. | Filtered holdings | `table` |

### G — Relative value (cheap / rich)
| # | Query | Expected answer | Expected block |
|---|---|---|---|
| 31 | Which bonds are cheap or rich in wnbf? | Cheap/rich list with notches | `table` |
| 32 | What is the cheapest bond we hold? | Single most-cheap bond | `factsheet` / `text` |
| 33 | Show the bonds trading rich. | Rich bonds | `table` |
| 34 | What is the predicted vs actual spread for XS1709535097? | RVM predicted spread + actual | `factsheet` / `text` |

### H — Holdings / concentration
| # | Query | Expected answer | Expected block |
|---|---|---|---|
| 35 | What are the top 10 holdings in wnbf? | Top 10 by weight | `table` |
| 36 | List all holdings in wnbf. | Full holdings | `table` |
| 37 | What is the largest single position in wnbf? | Biggest holding | `factsheet` / `text` |
| 38 | How concentrated is wnbf? | Top-N concentration % | `text` / `metric_strip` |

### I — Issuer type / sector
| # | Query | Expected answer | Expected block |
|---|---|---|---|
| 39 | What is the split between sovereign and quasi-sovereign in wnbf? | Issuer-type breakdown | `chart` / `table` |
| 40 | How much of wnbf is in corporates vs sovereigns? | Breakdown | `chart` / `table` |
| 41 | Show the issuer-type allocation for wnbf. | Issuer-type weights | `chart` / `table` |

### J — Currency / hedging
| # | Query | Expected answer | Expected block |
|---|---|---|---|
| 42 | What currencies is wnbf exposed to? | Currency breakdown (likely mostly USD) | `chart` / `table` / `text` |
| 43 | Is wnbf currency-hedged? | Hedging status | `text` |

### K — Performance / returns (likely not yet supported — should degrade gracefully)
| # | Query | Expected answer | Expected block |
|---|---|---|---|
| 44 | What is the year-to-date return of wnbf? | YTD return % (or a clean "not available yet") | `metric_strip` / `text` |
| 45 | How has wnbf performed this year? | Performance narrative / series | `chart` / `text` |
| 46 | What was wnbf's return last month? | Monthly return | `metric_strip` / `text` |

### L — ESG / NFA (x-trillion enrichment)
| # | Query | Expected answer | Expected block |
|---|---|---|---|
| 47 | What is the NFA / ESG rating profile of wnbf? | NFA-stars distribution | `chart` / `table` |
| 48 | Which holdings have the best ESG (NFA) scores? | Top NFA-stars holdings | `table` |

### M — Client-friendly / marketing narrative
| # | Query | Expected answer | Expected block |
|---|---|---|---|
| 49 | Draft a short client update on wnbf positioning. | 1–2 paragraph narrative (key exposures, yield, quality) | `text` |
| 50 | Explain in plain English why wnbf suits a cautious income investor. | Narrative tying yield + IG quality + diversification | `text` |

---

## What a good result looks like across the set
- **Numbers match** the snapshot the app already shows (yield 5.71%, duration 9.29, cash 2.27%, country/rating weights).
- **Block type matches intent**: "as a chart" → `chart`; "as a table" → `table`; a single bond → `factsheet`; a number → `metric_strip`/`text`; not a 5-block dump for everything.
- **Narrow questions don't error** — they answer the narrow thing, or degrade to the dashboard *with* a note, never `Error: 'country'`.
- **Consistency**: the same query returns the same answer on repeat (no random snapshot drift).
