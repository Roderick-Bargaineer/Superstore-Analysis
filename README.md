# Superstore-Analysis
Superstore Profitability Analysis
/* =============================================================================
   PROJECT:  Superstore Profitability Analysis
   QUESTION: Superstore's sales are growing. Is the growth profitable, and
             what is driving profit leakage?
   DATA:     Superstore orders, Jan 2014 - Dec 2017 (9,994 line items)
   TOOLS:    DuckDB, SQL (CTEs, window functions, conditional aggregation)

   HEADLINE FINDING
   Margin erosion is driven by deep discounting, not by product mix. Line items
   discounted more than 20% lose money overall, while all other sales earn a
   21.8% margin.

   CONTENTS
   Part 0. Setup
   Part 1. Data quality checks and cleaned view
   Part 2. Growth and margin trend (year over year)
   Part 3. Why did margin move? Mix vs. rate decomposition
   Part 4. Discount analysis
   Part 5. Product performance
   Part 6. Geography
   Part 7. Customers and segments
   Part 8. Market deep dive: Michigan and Virginia

   NOTES
   - Each statement ends with a semicolon and has no blank lines inside it.
   - After Part 1, every query reads from orders_clean, not orders.
   ============================================================================= */


/* =============================================================================
   PART 0. SETUP
   Rename the source file to data/superstore.csv before running.
   ============================================================================= */

CREATE OR REPLACE TABLE orders AS
SELECT *
FROM read_csv_auto('data/superstore.csv');


/* =============================================================================
   PART 1. DATA QUALITY CHECKS
   ============================================================================= */

-- 1.1 Row counts and date range
SELECT
    COUNT(*) AS total_rows,
    COUNT(DISTINCT "Order ID") AS unique_orders,
    COUNT(DISTINCT "Customer ID") AS unique_customers,
    MIN("Order Date") AS first_order,
    MAX("Order Date") AS last_order
FROM orders;

-- 1.2 Missing values in key columns
SELECT
    COUNT(*) FILTER (WHERE sales IS NULL) AS missing_sales,
    COUNT(*) FILTER (WHERE profit IS NULL) AS missing_profit,
    COUNT(*) FILTER (WHERE quantity IS NULL) AS missing_quantity,
    COUNT(*) FILTER (WHERE discount IS NULL) AS missing_discount,
    COUNT(*) FILTER (WHERE "Customer ID" IS NULL) AS missing_customer_id,
    COUNT(*) FILTER (WHERE "Product ID" IS NULL) AS missing_product_id,
    COUNT(*) FILTER (WHERE "Order Date" IS NULL) AS missing_order_date,
    COUNT(*) FILTER (WHERE "Ship Date" IS NULL) AS missing_ship_date
FROM orders;

-- 1.3 Duplicate rows.
--     "Row ID" is unique by definition, so it MUST be excluded from the
--     comparison. Including it can never find a duplicate.
SELECT COUNT(*) AS duplicate_rows
FROM (
    SELECT
        ROW_NUMBER() OVER (
            PARTITION BY
                "Order ID", "Order Date", "Ship Date", "Ship Mode",
                "Customer ID", "Customer Name", segment, country, city, state,
                "Postal Code", region, "Product ID", category, "Sub-Category",
                "Product Name", sales, quantity, discount, profit
            ORDER BY "Row ID"
        ) AS rn
    FROM orders
)
WHERE rn > 1;

-- 1.4 Inspect the duplicate (Row IDs 3406 and 3407 in this dataset)
SELECT "Row ID", "Order ID", "Product Name", sales, quantity, profit
FROM orders
WHERE "Order ID" = 'US-2014-150119'
ORDER BY "Row ID";

-- 1.5 Product IDs that map to more than one product name
SELECT
    "Product ID",
    COUNT(DISTINCT "Product Name") AS name_variants
FROM orders
GROUP BY "Product ID"
HAVING COUNT(DISTINCT "Product Name") > 1
ORDER BY name_variants DESC;

-- 1.6 Value ranges and logical checks
SELECT
    MIN(sales) AS min_sales,
    MAX(sales) AS max_sales,
    MIN(profit) AS min_profit,
    MAX(profit) AS max_profit,
    MIN(quantity) AS min_quantity,
    MAX(quantity) AS max_quantity,
    MIN(discount) AS min_discount,
    MAX(discount) AS max_discount,
    COUNT(*) FILTER (WHERE "Ship Date" < "Order Date") AS ship_before_order,
    COUNT(*) FILTER (WHERE profit < 0) AS loss_making_lines,
    ROUND(AVG(CASE WHEN profit < 0 THEN 1.0 ELSE 0 END) * 100, 1) AS pct_lines_loss_making
FROM orders;

-- 1.7 Cleaned view used for ALL analysis below.
--     Keeps one copy of any exact duplicate. Loss-making lines are kept: they
--     are real business outcomes, not errors.
CREATE OR REPLACE VIEW orders_clean AS
SELECT * EXCLUDE (rn)
FROM (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY
                "Order ID", "Order Date", "Ship Date", "Ship Mode",
                "Customer ID", "Customer Name", segment, country, city, state,
                "Postal Code", region, "Product ID", category, "Sub-Category",
                "Product Name", sales, quantity, discount, profit
            ORDER BY "Row ID"
        ) AS rn
    FROM orders
)
WHERE rn = 1;


/* =============================================================================
   PART 2. GROWTH AND MARGIN TREND
   ============================================================================= */

-- 2.1 Year-over-year sales, profit, and margin
WITH yearly AS (
    SELECT
        EXTRACT(YEAR FROM "Order Date") AS order_year,
        COUNT(DISTINCT "Order ID") AS orders,
        SUM(sales) AS total_sales,
        SUM(profit) AS total_profit
    FROM orders_clean
    GROUP BY EXTRACT(YEAR FROM "Order Date")
)
SELECT
    order_year,
    orders,
    ROUND(total_sales, 0) AS total_sales,
    ROUND(total_profit, 0) AS total_profit,
    ROUND(total_profit / total_sales * 100, 2) AS margin_pct,
    ROUND((total_sales / LAG(total_sales) OVER (ORDER BY order_year) - 1) * 100, 1) AS sales_growth_pct,
    ROUND((total_profit / total_sales - LAG(total_profit / total_sales) OVER (ORDER BY order_year)) * 100, 2) AS margin_change_pts
FROM yearly
ORDER BY order_year;


/* =============================================================================
   PART 3. WHY DID MARGIN MOVE? MIX VS. RATE DECOMPOSITION
   Splits each year's margin change into:
   - mix effect:  shift in sales share between sub-categories (at prior margins)
   - rate effect: change in margin WITHIN each sub-category (at current mix)
   mix_effect + rate_effect = total change in margin (percentage points).
   ============================================================================= */

-- 3.1 Margin change decomposition by year
WITH yearly_sub AS (
    SELECT
        EXTRACT(YEAR FROM "Order Date") AS order_year,
        "Sub-Category" AS sub_category,
        SUM(sales) AS sub_sales,
        SUM(profit) AS sub_profit
    FROM orders_clean
    GROUP BY EXTRACT(YEAR FROM "Order Date"), "Sub-Category"
),
shares AS (
    SELECT
        order_year,
        sub_category,
        sub_sales / SUM(sub_sales) OVER (PARTITION BY order_year) AS sales_share,
        sub_profit / sub_sales AS margin
    FROM yearly_sub
),
compare AS (
    SELECT
        b.order_year,
        a.sales_share AS share_prev,
        b.sales_share AS share_curr,
        a.margin AS margin_prev,
        b.margin AS margin_curr
    FROM shares a
    JOIN shares b
        ON a.sub_category = b.sub_category
        AND b.order_year = a.order_year + 1
)
SELECT
    order_year,
    ROUND(SUM(share_curr * margin_curr - share_prev * margin_prev) * 100, 2) AS total_margin_change_pts,
    ROUND(SUM((share_curr - share_prev) * margin_prev) * 100, 2) AS mix_effect_pts,
    ROUND(SUM(share_curr * (margin_curr - margin_prev)) * 100, 2) AS rate_effect_pts
FROM compare
GROUP BY order_year
ORDER BY order_year;

-- 3.2 Which sub-categories' margins fell the most in 2017?
WITH yearly_sub AS (
    SELECT
        EXTRACT(YEAR FROM "Order Date") AS order_year,
        "Sub-Category" AS sub_category,
        SUM(profit) / SUM(sales) AS margin
    FROM orders_clean
    GROUP BY EXTRACT(YEAR FROM "Order Date"), "Sub-Category"
)
SELECT
    sub_category,
    ROUND(MAX(margin) FILTER (WHERE order_year = 2016) * 100, 1) AS margin_2016_pct,
    ROUND(MAX(margin) FILTER (WHERE order_year = 2017) * 100, 1) AS margin_2017_pct,
    ROUND((MAX(margin) FILTER (WHERE order_year = 2017) - MAX(margin) FILTER (WHERE order_year = 2016)) * 100, 1) AS change_pts
FROM yearly_sub
GROUP BY sub_category
ORDER BY change_pts ASC;


/* =============================================================================
   PART 4. DISCOUNT ANALYSIS
   ============================================================================= */

-- 4.1 Profitability by discount band (whole company)
SELECT
    CASE
        WHEN discount = 0 THEN '1: No discount'
        WHEN discount <= 0.10 THEN '2: 1-10%'
        WHEN discount <= 0.20 THEN '3: 11-20%'
        WHEN discount <= 0.30 THEN '4: 21-30%'
        ELSE '5: Over 30%'
    END AS discount_band,
    COUNT(*) AS line_items,
    ROUND(SUM(sales), 0) AS total_sales,
    ROUND(SUM(profit), 0) AS total_profit,
    ROUND(SUM(profit) / SUM(sales) * 100, 1) AS margin_pct
FROM orders_clean
GROUP BY discount_band
ORDER BY discount_band;

-- 4.2 Deep-discount lines (over 20%) vs. everything else
SELECT
    CASE WHEN discount > 0.20 THEN 'Discount over 20%' ELSE 'Discount 20% or less' END AS discount_group,
    COUNT(*) AS line_items,
    ROUND(SUM(sales), 0) AS total_sales,
    ROUND(SUM(profit), 0) AS total_profit,
    ROUND(SUM(profit) / SUM(sales) * 100, 1) AS margin_pct
FROM orders_clean
GROUP BY discount_group
ORDER BY discount_group;

-- 4.3 Profit lost on deep-discount lines, by year
SELECT
    EXTRACT(YEAR FROM "Order Date") AS order_year,
    COUNT(*) AS deep_discount_lines,
    ROUND(SUM(profit), 0) AS profit_on_deep_discount_lines
FROM orders_clean
WHERE discount > 0.20
GROUP BY EXTRACT(YEAR FROM "Order Date")
ORDER BY order_year;

-- 4.4 Scenario: what if no line had been discounted more than 20%?
--     Assumes the same units sold at the higher price, so this is an UPPER
--     BOUND (a higher price would likely reduce volume). Cost per line is
--     inferred as sales - profit.
SELECT
    ROUND(SUM(profit), 0) AS actual_profit,
    ROUND(SUM(CASE WHEN discount > 0.20 THEN sales * 0.80 / (1 - discount) - (sales - profit) ELSE profit END), 0) AS profit_if_capped_at_20pct,
    ROUND(SUM(CASE WHEN discount > 0.20 THEN sales * 0.80 / (1 - discount) - (sales - profit) ELSE profit END) - SUM(profit), 0) AS upper_bound_uplift
FROM orders_clean;

-- 4.5 Discount behavior by sub-category: share of deep-discount lines
SELECT
    "Sub-Category" AS sub_category,
    COUNT(*) AS line_items,
    ROUND(AVG(CASE WHEN discount > 0.20 THEN 1.0 ELSE 0 END) * 100, 1) AS pct_lines_over_20pct_discount,
    ROUND(SUM(profit) / SUM(sales) * 100, 1) AS margin_pct
FROM orders_clean
GROUP BY "Sub-Category"
ORDER BY margin_pct ASC;


/* =============================================================================
   PART 5. PRODUCT PERFORMANCE
   ============================================================================= */

-- 5.1 Category performance
SELECT
    category,
    ROUND(SUM(sales), 0) AS total_sales,
    ROUND(SUM(profit), 0) AS total_profit,
    ROUND(SUM(profit) / SUM(sales) * 100, 1) AS margin_pct
FROM orders_clean
GROUP BY category
ORDER BY total_sales DESC;

-- 5.2 Sub-category performance, ranked by profit within each category
SELECT
    category,
    "Sub-Category" AS sub_category,
    ROUND(SUM(sales), 0) AS total_sales,
    ROUND(SUM(profit), 0) AS total_profit,
    ROUND(SUM(profit) / SUM(sales) * 100, 1) AS margin_pct,
    RANK() OVER (PARTITION BY category ORDER BY SUM(profit) DESC) AS profit_rank_in_category
FROM orders_clean
GROUP BY category, "Sub-Category"
ORDER BY category, profit_rank_in_category;

-- 5.3 Loss-making and low-margin sub-categories (margin under 5%)
SELECT
    "Sub-Category" AS sub_category,
    ROUND(SUM(sales), 0) AS total_sales,
    ROUND(SUM(profit), 0) AS total_profit,
    ROUND(SUM(profit) / SUM(sales) * 100, 1) AS margin_pct
FROM orders_clean
GROUP BY "Sub-Category"
HAVING SUM(profit) / SUM(sales) < 0.05
ORDER BY margin_pct ASC;


/* =============================================================================
   PART 6. GEOGRAPHY
   ============================================================================= */

-- 6.1 Region performance with deep-discount share
SELECT
    region,
    ROUND(SUM(sales), 0) AS total_sales,
    ROUND(SUM(profit), 0) AS total_profit,
    ROUND(SUM(profit) / SUM(sales) * 100, 1) AS margin_pct,
    ROUND(AVG(CASE WHEN discount > 0.20 THEN 1.0 ELSE 0 END) * 100, 1) AS pct_lines_over_20pct_discount
FROM orders_clean
GROUP BY region
ORDER BY total_profit DESC;

-- 6.2 State performance with deep-discount share (all states)
SELECT
    region,
    state,
    COUNT(DISTINCT "Order ID") AS orders,
    ROUND(SUM(sales), 0) AS total_sales,
    ROUND(SUM(profit), 0) AS total_profit,
    ROUND(SUM(profit) / SUM(sales) * 100, 1) AS margin_pct,
    ROUND(AVG(CASE WHEN discount > 0.20 THEN 1.0 ELSE 0 END) * 100, 1) AS pct_lines_over_20pct_discount
FROM orders_clean
GROUP BY region, state
ORDER BY total_profit DESC;

-- 6.3 Loss-making states and the total profit they cost
SELECT
    state,
    ROUND(SUM(sales), 0) AS total_sales,
    ROUND(SUM(profit), 0) AS total_profit,
    ROUND(SUM(profit) / SUM(sales) * 100, 1) AS margin_pct,
    ROUND(AVG(CASE WHEN discount > 0.20 THEN 1.0 ELSE 0 END) * 100, 1) AS pct_lines_over_20pct_discount
FROM orders_clean
GROUP BY state
HAVING SUM(profit) < 0
ORDER BY total_profit ASC;

-- 6.4 City performance: top 5 and bottom 5 by profit
(
    SELECT 'Top 5' AS grp, state, city, ROUND(SUM(sales), 0) AS total_sales, ROUND(SUM(profit), 0) AS total_profit, ROUND(SUM(profit) / SUM(sales) * 100, 1) AS margin_pct
    FROM orders_clean
    GROUP BY state, city
    ORDER BY SUM(profit) DESC
    LIMIT 5
)
UNION ALL
(
    SELECT 'Bottom 5' AS grp, state, city, ROUND(SUM(sales), 0) AS total_sales, ROUND(SUM(profit), 0) AS total_profit, ROUND(SUM(profit) / SUM(sales) * 100, 1) AS margin_pct
    FROM orders_clean
    GROUP BY state, city
    ORDER BY SUM(profit) ASC
    LIMIT 5
);


/* =============================================================================
   PART 7. CUSTOMERS AND SEGMENTS
   ============================================================================= */

-- 7.1 Segment performance
SELECT
    segment,
    COUNT(DISTINCT "Customer ID") AS customers,
    ROUND(SUM(sales), 0) AS total_sales,
    ROUND(SUM(profit), 0) AS total_profit,
    ROUND(SUM(profit) / SUM(sales) * 100, 1) AS margin_pct
FROM orders_clean
GROUP BY segment
ORDER BY total_sales DESC;

-- 7.2 Top 10 customers by sales, with their profit rank
WITH customer_totals AS (
    SELECT
        "Customer ID" AS customer_id,
        "Customer Name" AS customer_name,
        COUNT(DISTINCT "Order ID") AS orders,
        SUM(sales) AS total_sales,
        SUM(profit) AS total_profit
    FROM orders_clean
    GROUP BY "Customer ID", "Customer Name"
),
ranked AS (
    SELECT
        *,
        RANK() OVER (ORDER BY total_sales DESC) AS sales_rank,
        RANK() OVER (ORDER BY total_profit DESC) AS profit_rank
    FROM customer_totals
)
SELECT
    customer_name,
    orders,
    ROUND(total_sales, 0) AS total_sales,
    ROUND(total_profit, 0) AS total_profit,
    ROUND(total_profit / total_sales * 100, 1) AS margin_pct,
    sales_rank,
    profit_rank
FROM ranked
WHERE sales_rank <= 10
ORDER BY sales_rank;

-- 7.3 How many customers are unprofitable, and what do they cost?
WITH customer_totals AS (
    SELECT "Customer ID", SUM(profit) AS total_profit
    FROM orders_clean
    GROUP BY "Customer ID"
)
SELECT
    COUNT(*) AS total_customers,
    COUNT(*) FILTER (WHERE total_profit < 0) AS unprofitable_customers,
    ROUND(SUM(total_profit) FILTER (WHERE total_profit < 0), 0) AS profit_lost_on_unprofitable_customers
FROM customer_totals;

-- 7.4 Profit concentration: share of total profit by customer decile
WITH customer_totals AS (
    SELECT "Customer ID", SUM(profit) AS total_profit
    FROM orders_clean
    GROUP BY "Customer ID"
),
deciles AS (
    SELECT
        total_profit,
        NTILE(10) OVER (ORDER BY total_profit DESC) AS profit_decile
    FROM customer_totals
)
SELECT
    profit_decile,
    ROUND(SUM(total_profit), 0) AS decile_profit,
    ROUND(SUM(total_profit) / SUM(SUM(total_profit)) OVER () * 100, 1) AS pct_of_total_profit
FROM deciles
GROUP BY profit_decile
ORDER BY profit_decile;


/* =============================================================================
   PART 8. MARKET DEEP DIVE: MICHIGAN AND VIRGINIA
   Both are high-margin, low-volume states (about 30 orders a year each), so
   year-to-year swings may reflect a handful of large orders.
   ============================================================================= */

-- 8.1 Annual performance
SELECT
    state,
    EXTRACT(YEAR FROM "Order Date") AS order_year,
    COUNT(DISTINCT "Order ID") AS orders,
    COUNT(DISTINCT "Customer ID") AS active_customers,
    ROUND(SUM(sales), 0) AS total_sales,
    ROUND(SUM(profit), 0) AS total_profit,
    ROUND(SUM(profit) / SUM(sales) * 100, 1) AS margin_pct
FROM orders_clean
WHERE state IN ('Michigan', 'Virginia')
GROUP BY state, EXTRACT(YEAR FROM "Order Date")
ORDER BY state, order_year;

-- 8.2 New vs. returning customers in Virginia by year
WITH virginia AS (
    SELECT
        "Customer ID" AS customer_id,
        EXTRACT(YEAR FROM "Order Date") AS order_year
    FROM orders_clean
    WHERE state = 'Virginia'
),
first_year AS (
    SELECT customer_id, MIN(order_year) AS first_order_year
    FROM virginia
    GROUP BY customer_id
)
SELECT
    v.order_year,
    COUNT(DISTINCT v.customer_id) AS active_customers,
    COUNT(DISTINCT v.customer_id) FILTER (WHERE v.order_year = f.first_order_year) AS new_customers,
    COUNT(DISTINCT v.customer_id) FILTER (WHERE v.order_year > f.first_order_year) AS returning_customers
FROM virginia v
JOIN first_year f
    ON v.customer_id = f.customer_id
GROUP BY v.order_year
ORDER BY v.order_year;
