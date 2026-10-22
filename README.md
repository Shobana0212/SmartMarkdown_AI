# SmartMarkdown_AI

## Mohan Raj I - 23MID0121
## Athiyamaan S - 23MID0245
## Shobana Balusamy - 23MID0212

## 1. Project overview

**SmartMarkdown AI** is a prototype for expiry-aware pricing of perishable grocery products. It simulates a single store, compares fixed-price and markdown policies, reports waste and financial outcomes, and provides a Streamlit dashboard. The project also contains a shopper-facing grocery storefront and a CSV-based daily inventory upload workflow.

The aim is to explore the trade-off between reducing expired stock and protecting revenue/profit. Results are generated from synthetic data and must not be presented as measured performance from a real store.

## 2. Project goals

- Simulate a store with 20 perishable products for up to 30 days.
- Model stock arrivals as batches with expiry dates.
- Make daily demand depend on baseline demand, freshness, price sensitivity and a shopper-response parameter.
- Compare fixed price with configurable expiry-aware markdowns under matched random seeds.
- Display revenue, gross simulation profit, margin, units wasted and product-level event history.
- Explore candidate discount schedules using a grid search.
- Demonstrate a shopper-facing “FreshMart” storefront and a validated CSV inventory upload interface.

## 3. Repository map

| File | Purpose |
|---|---|
| `app.py` | Main Streamlit simulation dashboard: controls, policy comparison, daily charts, event history, CSV export, assumptions panel and inventory upload/monitoring section. |
| `store.py` | Shopper-facing FreshMart demo with product search, category filter, deal-only filter, cart and demo checkout. |
| `simulator.py` | Core inventory, expiry, demand, selling, accounting and event simulation. Also contains a command-line grid-search experiment. |
| `data_generator.py` | Defines and writes the 20-row synthetic product catalogue to `products.csv`. |
| `products.csv` | Product attributes used by the simulator. |
| `select_policy.py` | Reads `policy_comparison.csv` and ranks markdown settings profit-first, using waste as a tie-breaker. |
| `test_selective.py` | Experiment script comparing fixed, markdown and selective policies over seeds 42–51. It prints averages; it is a script rather than a formal unit-test suite. |
| `requirements.txt` | Python dependencies: Streamlit, pandas, NumPy and Plotly. |
| `daily_fixed.csv`, `daily_markdown.csv` | Example daily simulation outputs for the fixed and markdown policies. |
| `events_fixed.csv`, `events_markdown.csv` | Example event logs (arrival, sale and expiry records). |
| `policy_comparison.csv` | Aggregated policy-search results over 10 seeds. |
| `data/synthetic_grocery_inventory.csv` | Separate 3,000-row synthetic inventory/demand dataset with scenario, stock, price, expiry-hours and contextual columns. |
| `data/daily_stock.csv` | Example stock-upload CSV template. |
| `data/current_inventory.csv` | Saved example inventory snapshot. |
| `data/inventory_upload_history/` | Archived snapshots from inventory uploads. |
| `DEV_LOG.md` | Present in the ZIP but empty. |
| `README.md` | Was empty in the supplied ZIP; this document is a proposed complete replacement. |

The ZIP also includes a local `.venv` environment. It is not needed in source control and should not be deployed; install dependencies from `requirements.txt` instead.

## 4. Requirements

- Python 3.10+ is a reasonable starting point; the supplied local environment was created with Python 3.13.
- `streamlit`
- `pandas`
- `numpy`
- `plotly`

Install in a clean virtual environment:

```bash
python -m venv .venv
# Windows PowerShell:
.venv\Scripts\Activate.ps1
# macOS/Linux:
source .venv/bin/activate

python -m pip install --upgrade pip
pip install -r requirements.txt
```

## 5. Run the applications

Run the simulation dashboard from the project root:

```bash
streamlit run app.py
```

Run the shopper-facing storefront separately:

```bash
streamlit run store.py
```

Run the discount-grid simulation from the project root:

```bash
python simulator.py
```

Then inspect the ranked policies:

```bash
python select_policy.py
```

The `test_selective.py` script is an experiment runner:

```bash
python test_selective.py
```

Keep the CSV files and Python modules in the project root unless you update the paths in the code.

## 6. Product catalogue

The catalogue has 20 products across four categories: Dairy, Bakery, Produce and Ready-to-eat. Each row includes:

- `product`
- `category`
- `shelf_life_days`
- `unit_cost`
- `base_price`
- `avg_daily_demand`
- `price_sensitivity`

The data is synthetic and hard-coded in `data_generator.py`. The generator asserts that there are 20 products, each has positive shelf life, and each base price is greater than unit cost.

## 7. Simulation logic and flow

For each policy run:

1. Load and validate the product catalogue.
2. Create two deterministic random generators: one for arrivals (`seed + 100003`) and one for baseline demand (`seed`).
3. Pre-generate baseline demand for each product and day from a Poisson distribution.
4. For every simulated day and product:
   - Remove batches whose `expiry_day <= current_day`; log expired units and their cost.
   - On day indices 0, 2, 4, ... receive a new batch.
   - Determine the days remaining until the earliest-expiring available batch.
   - Choose the selling price according to the selected policy.
   - Apply freshness and price-response adjustments to baseline demand.
   - Sell stock using FEFO (First Expired, First Out).
   - Record revenue, sold cost and an event row.
5. After the horizon, calculate ending stock and aggregate KPIs.
6. Return a summary, daily DataFrame and event-history DataFrame.

### Batch arrivals

A new batch is received every two days (`day % 2 == 0`). Its quantity is sampled from a normal distribution:

- mean = `avg_daily_demand * 2`
- standard deviation = `max(1, avg_daily_demand * 0.2)`
- quantity is rounded and bounded below by 1

The batch expiry day is `arrival_day + shelf_life_days`.

### Expiry convention

A batch with shelf life 2 arriving on day index 0 may sell on day indices 0 and 1. It expires before sales on day index 2. Expired batches are removed before that day's sales.

### Demand assumptions

- Baseline demand for each product/day is sampled from a Poisson distribution.
- The weekday factor is 1.15 when `day % 7` is 5 or 6; otherwise it is 1.00. This is based on the simulation day index, not a calendar date.
- Freshness factor is 0.90 when the earliest batch has 1 day or less remaining; 0.97 when exactly 2 days remain; otherwise 1.00.
- Price elasticity is represented by:

  `price_factor = (price / base_price) ** (-price_sensitivity * discount_response)`

- Adjusted demand is the rounded baseline demand after freshness and price factors.
- `discount_response` defaults to 1.0. The dashboard allows a range of 0.5× to 1.5×. A value of 0.5 means the price-related demand response is half as strong in the model; it does not come from observed shopper research.

### Selling rule

Stock is sold FEFO: the batch with the earliest expiry is sold first. The markdown price is calculated from the minimum `expiry_day` among currently available batches.

## 8. Pricing policies

### Fixed price

The product sells at its base price, with no markdown discount.

### Markdown

The `markdown_discount()` function uses the following thresholds:

- If `days_left <= 1`, use `one_day_discount`.
- Else if `days_left <= 2`, use `two_day_discount`.
- Otherwise, use no discount.

Both discounts are configurable. The simulator function defaults are 20% for two days remaining and 40% for one day remaining. The dashboard controls in `app.py` currently initialize to 5% and 10%. Always record the actual values used in an experiment.

### Selective policy

The simulator also accepts `policy="selective"`. It checks stock cover (stock quantity divided by average daily demand) and applies capped discounts only when expiry is close and stock cover is high. In the supplied implementation, the one-day discount is capped at 10%, and the two-day discount at 5%. This policy is exercised by `test_selective.py`, but the main dashboard comparison runs fixed and markdown.

## 9. Financial definitions

- **Revenue** = sum of `units sold * selling price`.
- **Cost of sold goods** = units sold multiplied by unit cost.
- **Cost of expired goods** = expired units multiplied by unit cost.
- **Gross simulation profit** = revenue − cost of sold goods − cost of expired goods.
- **Profit margin** = gross simulation profit / revenue × 100, or 0 if revenue is zero.
- **Units wasted** = units removed from batches after expiry.
- **Ending stock** = remaining units at the end of the simulation. Stock that has not yet expired by the end is not counted as waste.
- **Units marked down** = units sold when the active discount is greater than zero.

The model does not include rent, wages, utilities, payment fees, disposal costs, replenishment logistics, tax, price rounding rules beyond display/output rounding, or substitution between products. The profit figure is therefore not net business profit.

## 10. Dashboard guide (`app.py`)

The simulation dashboard provides:

- Random seed input (0–999999; default 42).
- Simulation duration slider (7–30 days; default 30).
- Two-day and one-day discount sliders.
- Shopper response strength slider (0.5×–1.5×).
- Run-comparison button.
- KPI cards for markdown profit, units wasted, waste reduction and margin.
- Fixed-versus-markdown comparison table.
- Daily revenue, profit and waste charts.
- Product/event filtering by product and policy.
- CSV export of the visible event history.
- Expandable assumptions/limitations text.

The app checks that the one-day discount is at least as high as the two-day discount before running the comparison.

### Daily inventory upload section

The main app also contains a CSV upload and inventory monitoring workflow. Required input columns:

- `product`
- `batch_id`
- `quantity`
- `expiry_date`
- `unit_cost`
- `base_price`

The workflow normalizes column names, validates required fields, non-negative integer quantities, non-negative costs/prices, valid expiry dates, and unique batch IDs within the uploaded file. Uploading replaces the current inventory snapshot rather than adding to it. The previous snapshot and the uploaded snapshot are archived in `data/inventory_upload_history/`. The saved inventory view reports batches, units in stock, expired units, units expiring within two days and an expiry status.

**Implementation boundary:** the uploaded inventory tracker is currently a separate operational section; the simulation's generated inventory and the uploaded live-style snapshot are not integrated into one shared stock engine.

## 11. Shopper-facing storefront (`store.py`)

The storefront is a separate Streamlit app with:

- Product cards and category labels.
- Search by product name.
- Category filter.
- “Show SmartMarkdown deals only” filter.
- Session-based shopping cart, remove/clear actions and a demo checkout.

The storefront currently assigns illustrative discounts from product shelf-life attributes (5% if catalogue shelf life is at most 2 days; 10% if at most 1 day). These discounts do **not** use current batch expiry dates, the simulation's per-product events or actual uploaded inventory. The demo order does not collect payment and should not be described as a production checkout.

## 12. Optimization and sensitivity

### Discount grid search

`simulator.py` searches the following candidate values over seeds 42–51:

- Two-day discount: 5%, 10%, 15%, 20%.
- One-day discount: 10%, 20%, 30%, 40%, 50%.
- Combinations where the one-day discount is below the two-day discount are skipped.
- Fixed price is the baseline.
- Results are averaged by policy and discount pair and written to `policy_comparison.csv`.

`select_policy.py` ranks available markdown settings by average profit, then lower waste. It is a profit-first ranking, not a multi-objective optimizer.

### Shopper response sensitivity

The simulator supports `discount_response`; the dashboard exposes it. A direct sensitivity experiment should run both policies at 1.0× and 0.5× with the same seeds and compare KPI differences. The supplied `policy_comparison.csv` is a discount-grid output, not a precomputed response-strength sensitivity report.

### Category-specific optimization

Category-specific discount optimization is not implemented in the supplied grid search. It is a stretch goal: run candidate schedules separately by category and compare results across matched seeds.

## 13. Supplied data and experiment snapshot

The ZIP contains a 20-product catalogue, 30-row daily summaries for fixed and markdown policies, event logs for both policies, a policy-comparison table, a separate 3,000-row synthetic inventory dataset and example uploaded inventory files.

The supplied `policy_comparison.csv` reports the following **synthetic, 10-seed averages**:

| Scenario | 2-day discount | 1-day discount | Avg. profit | Avg. waste | Avg. revenue | Avg. margin |
|---|---:|---:|---:|---:|---:|---:|
| Fixed price | 0% | 0% | ₹55,680.10 | 127.5 units | ₹150,964.30 | 36.88% |
| Markdown, 5% / 10% | 5% | 10% | ₹55,082.02 | 60.1 units | ₹150,871.82 | 36.51% |
| Markdown, lowest waste in grid | 20% | 50% | ₹48,184.74 | 1.7 units | ₹144,449.94 | 33.35% |

The 5% / 10% rule reduces average waste by about 52.86% and average profit by about 1.07% versus the fixed-price baseline. The 20% / 50% rule reduces average waste by about 98.67% and average profit by about 13.46%. The lowest-waste setting is not the profit-maximizing setting.

These averages are based on the supplied CSV and should be treated as experiment outputs, not guaranteed or real-world outcomes. The one-seed daily/event files are examples and should not be conflated with the 10-seed averages.

## 14. Limitations and risks

1. All product attributes and demand are synthetic.
2. The model uses a simplified price-elasticity equation, not a trained machine-learning model.
3. The simulation is a single-store, single-product-catalogue model; it does not model customer-level baskets or product substitution.
4. The model uses fixed replenishment cadence and quantity distribution rather than a replenishment optimizer.
5. Ending stock that has not expired by the final day is not counted as waste.
6. The storefront is not linked to the simulated or uploaded current inventory.
7. The inventory upload feature stores a CSV snapshot locally; it is not a database-backed multi-user inventory service.
8. The selective policy exists in the simulator, but is not part of the main dashboard's two-policy comparison.
9. The optimizer currently ranks profit first and uses waste as a tie-breaker; it does not calculate a formal Pareto frontier.
10. No live store, POS, payment or customer data is connected.

## 15. Suggested next steps

1. Add automated unit tests for expiry timing, FEFO, discount thresholds, accounting identities and deterministic same-seed comparisons.
2. Connect the storefront to the current inventory snapshot and actual simulated expiry dates.
3. Add category-specific policy search for Dairy, Bakery, Produce and Ready-to-eat.
4. Add a paired sensitivity report comparing response strength 1.0× and 0.5× over multiple seeds.
5. Add a configurable business objective (profit-first, waste-first or weighted score) and display trade-offs.
6. Persist inventory and simulation runs in a database for deployment.
7. Add input validation, logging, deployment secrets/configuration and automated tests before production use.

## 16. Reproducibility checklist

Before presenting a result, record:

- Seed(s)
- Simulation days
- Two-day discount
- One-day discount
- Shopper response strength
- Policy names
- Whether the result is a single seed or average across multiple seeds
- Source CSV/version and date of experiment

## 17. Disclaimer

SmartMarkdown AI is an academic/prototype simulation. It is not a production pricing engine, live inventory service or financial forecast. All recommendations depend on the synthetic assumptions and should be validated against real store data before operational use.
