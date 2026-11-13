# Superstore Profitability Analysis

**Is Superstore's sales growth profitable, and what is driving profit leakage?**

An analysis of 9,994 order line items (Jan 2014 – Dec 2017) using SQL (DuckDB). The work covers data validation, year-over-year margin analysis, a mix vs. rate decomposition, a discount scenario model, and geographic and customer profitability.

---

## Executive Summary

Sales grew **51.5%** from 2014 to 2017 ($484K to $733K) and profit grew **88.6%** ($49.6K to $93.4K), lifting margin from **10.2% to 12.7%**. But 2017 shows a warning sign: **sales grew 20.4% while profit grew only 14.2%, and margin fell 0.68 pts from its 2016 peak of 13.4%.**

The main driver is discounting. **Line items discounted more than 20% lose money in aggregate (-37.3% margin, -$135K profit), while all other sales earn a 21.8% margin.**

**Key findings**

1. **Growth is real but margin slipped in 2017.** Margin fell 0.68 pts in 2017. Product mix *helped* (+1.17 pts) but within-sub-category margin rates *hurt* more (-1.86 pts), so the decline is a pricing and discount problem, not a product-mix problem. *(Query 3.1)*
2. **Deep discounts cause most of the losses.** Lines discounted over 20% are 13.9% of line items and 15.8% of sales, but 1,347 of those 1,392 lines (97%) lose money, and they account for **89% of all line-level losses** ($138.5K of $156.1K). *(Queries 4.2, 4.3)*
3. **The effect holds within product categories.** Deep discounts appear in 9 of 17 sub-categories. In 8 of those 9, deep-discount lines have negative margins while non-deep lines in the same sub-category are profitable (for example, Binders: -106.6% vs. +41.1%). Holding sub-category mix constant, non-deep margin is +22.1% vs. -48.5% for deep. *(Query 4.1 and extension)*
4. **Losses are concentrated.** Binders, Tables and Machines hold 73% of deep-discount losses. Three sub-categories (Tables, Bookcases, Supplies) lose money outright, for -$22.4K combined. Furniture earns a 2.5% margin vs. 17% for the other categories. *(Query 5.3)*
5. **10 of 49 states lose money (-$98.2K combined).** Texas (-$25.7K), Ohio (-$17.0K), Pennsylvania (-$15.6K) and Illinois (-$12.6K) lead. These states have a deep-discount share of 34.6% of lines vs. 13.9% overall. *(Query 6.3)*
6. **Customer profit is concentrated and 1 in 5 customers loses money.** 155 of 793 customers (19.5%) are unprofitable, costing $71.2K. The top 10% of customers generate 58% of net profit. *(Queries 7.3, 7.4)*

## Recommendations

| # | Recommendation | Estimated impact | Basis |
|---|----------------|------------------|-------|
| 1 | Cap discounts at 20%, starting with Binders, Machines and Tables | **+$135K to +$219K profit** (see range below) | Query 4.4 |
| 2 | Review discount approval in the 10 loss-making states, starting with Texas, Ohio, Pennsylvania and Illinois | Up to $98K of state-level losses in scope | Query 6.3 |
| 3 | Reprice or review Tables and Bookcases (loss-making even before deep discounts are isolated) | Up to $21K of losses in scope | Query 5.3 |
| 4 | Review the 155 unprofitable customers for discount terms | Up to $71K of customer-level losses in scope | Query 7.3 |

> **How to read the range for Recommendation 1.** The **$219K** upper bound assumes customers buy the same units at the higher price. The **$135K** lower bound assumes the deep-discount lines simply aren't sold (volume loss of 100%), which removes the losses but forgoes any revenue. The true outcome likely sits between them. The 20% cap stays above $135K at every volume-loss level.
>
> Even with the cap, 198 of the 1,392 repriced lines would still lose money because their cost exceeds the 20%-off price.

## Sensitivity: What if volume drops when discounts are capped?

Change in total profit (vs. $286K baseline) by discount cap and share of volume lost on repriced lines:

| Discount cap | 0% volume loss | 10% | 25% | 50% | 75% | 100% |
|--------------|----------------|-----|-----|-----|-----|------|
| 10% | +$389K | +$355K | +$303K | +$216K | +$130K | +$44K |
| **20%** | **+$219K** | **+$211K** | **+$198K** | **+$177K** | **+$156K** | **+$135K** |
| 30% | +$146K | +$144K | +$141K | +$136K | +$130K | +$125K |

A 20% cap is the most defensible choice: it removes nearly all of the losses without repricing the 3,709 lines at 11–20% discount, which are still profitable (11.6% margin). A 10% cap is riskier because it reprices profitable lines too, and at high volume loss it gives up most of the gain.

## Why 20%?

| Discount band | Lines | % of lines | Margin | % of lines losing money |
|---------------|-------|-----------|--------|-------------------------|
| None | 4,798 | 48.0% | 29.5% | 0.0% |
| 1–10% | 94 | 0.9% | 16.6% | 4.3% |
| 11–20% | 3,709 | 37.1% | 11.6% | 14.0% |
| 21–30% | 226 | 2.3% | -10.1% | 91.6% |
| 31–50% | 310 | 3.1% | -24.8% | 91.6% |
| Over 50% | 856 | 8.6% | -119.2% | 100.0% |

Profitability breaks sharply above 20%, which is why it is the threshold used throughout.

## A natural comparison: Michigan and Virginia

Michigan and Virginia are the #4 and #5 states by total profit ($24.5K and $18.6K, about 115 orders each), and they **barely discount at all: average discount is 0.7% in Michigan and 0.0% in Virginia, with zero deep-discount lines**. Their margins are 32.1% and 26.3%, versus -15% to -22% in the heavily discounting states such as Texas and Ohio. Michigan and Virginia are not unique: other low-discount states such as Indiana (34.3% margin) and Georgia (33.1%) look similar. Across all 49 states, average discount and margin are almost perfectly inversely related (correlation of -0.96). This is a correlation across states, not proof of cause, but it is consistent with the line-level findings.

Caution: each state has only about 30 orders a year, so year-to-year swings (for example Virginia's sales dropping from $26.7K in 2016 to $7.6K in 2017) may reflect a few large orders.

## Visuals



1. Sales, profit and margin by year
2. Margin by discount band
3. Mix vs. rate effect by year
4. Profit by state (loss-making states highlighted)
5. Customer profit concentration by decile

---

## Method

| Step | What was done | SQL |
|------|---------------|-----|
| 1. Data quality | Row counts, nulls, duplicates, logical checks | Part 1 |
| 2. Trend | Year-over-year sales, profit and margin | Part 2 |
| 3. Decomposition | Split margin change into mix and rate effects | Part 3 |
| 4. Discounts | Profitability by discount band; capped-discount scenario | Part 4 |
| 5. Products | Category and sub-category performance | Part 5 |
| 6. Geography | Region, state and city performance | Part 6 |
| 7. Customers | Segments, profit concentration, unprofitable customers | Part 7 |
| 8. Deep dive | Michigan and Virginia: top-5 profit states with near-zero discounting | Part 8 |

## Data and Assumptions

- **Source:** Superstore sample dataset, 9,994 rows, Jan 2014 – Dec 2017. *[Add link and license/source.]*
- **Cleaning:** 1 exact duplicate removed (Row IDs 3406 and 3407), leaving 9,993 line items. No nulls, no negative sales or quantities, and no ship dates before order dates were found. Loss-making lines were kept because they are real business outcomes, not errors.
- **Product IDs:** 32 Product IDs map to more than one product name (IDs were reused across different products), so product-level analysis uses Sub-Category rather than Product ID.
- **Profit definition:** The dataset has no cost fields. Profit is as reported in the source and is treated as gross/contribution profit, **not** net income. Margin = profit ÷ sales. Cost per line is inferred as sales − profit.
- **"Deep discount":** discount greater than 20% (see "Why 20%?" above).
- **Mix vs. rate decomposition:** by sub-category. Mix effect = (current share − prior share) × prior margin. Rate effect = current share × (current margin − prior margin). The two sum to the total margin change.
- **Scenario method:** list price = sales ÷ (1 − discount). Repriced sales = list price × (1 − cap). Cost is held constant. Volume-loss scenarios scale the repriced lines' new profit down by the share of units lost.
- **Scenario limits:** The scenario ignores customer churn, halo effects (discounted items driving other purchases), and competitor response. Treat results as a range, not a forecast.
- **Causation:** The discount findings show a strong, consistent association that holds within sub-categories. They do not prove causation, because discounts may be targeted at products that were already low-margin, and the dataset has no list-price or cost detail to test this.

## Limitations and Next Steps

- Add a simple 2018 forecast or actual-vs-target view to cover the planning side of FP&A.
- Build a dashboard (Tableau / Power BI / Excel) for non-technical stakeholders.
- Investigate *why* Texas, Ohio, Pennsylvania and Illinois discount so heavily (average discounts of 32–39% vs. 15.6% overall). The pattern looks like a regional pricing practice rather than a product effect.
- Michigan and Virginia have about 30 orders a year each, so year-to-year swings may reflect a few large orders.

## How to Reproduce

1. Install [DuckDB](https://duckdb.org/docs/installation/).
2. Download the dataset and save it as `data/superstore.csv`. https://www.kaggle.com/datasets/vivek468/superstore-dataset-final*
3. Run `duckdb < sql/superstore_analysis.sql`, or run the script section by section.

## Repository Structure

```
Superstore-Analysis/
├── README.md
├── REPORT.md
├── sql/
│   └── superstore_analysis.sql
├── data/
│   └── superstore.csv
└── images/
```

## Tools

DuckDB · SQL (CTEs, window functions, conditional aggregation)
