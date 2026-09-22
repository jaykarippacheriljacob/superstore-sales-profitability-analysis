# Sales & Profitability Analysis — Global Superstore
**Covers:** Veracity (messy-data judgment) · SQL, statistics, dashboarding

## The Question
Which product categories, regions, and discount levels are actually driving profit — and which ones are quietly losing money?

I started here because "sales are up" is a meaningless headline without knowing whether profit is following. Superstore data lets you test that gap directly.

## Data
- Source: [Global Superstore dataset, Kaggle](https://www.kaggle.com/datasets/vivek468/superstore-dataset-final)
- 9,994 rows (9,993 after removing 1 duplicate), 2014–2017, orders across US regions
- Fields: order/ship dates, category/sub-category, sales, discount, profit, region, segment

## Tools
SQL (SQLite, window functions) · Python/Pandas + statsmodels (cleaning, EDA, hypothesis testing) · Tableau Public (dashboard)

## Process

### 1. Cleaning & judgment calls
- Checked for duplicate order lines: found 1 exact duplicate, dropped it (data export artifact, not a real repeated order)
- No missing values in the real dataset — a cleaner starting point than typical tutorial datasets, so the "veracity" story here centers on outlier/threshold judgment rather than missing-data handling
- Verified ship dates never precede order dates (0 violations) — a basic sanity check that passed

### 2. Exploratory analysis
- 1,871 orders (18.7% of all orders) are net loss-making — a large enough share to be a real pattern, not noise
- Loss rate by discount level shows a sharp, non-gradual jump: 0% loss rate at no discount, staying under ~33% through 20% discount, then jumping to 87–100% loss rate at 30%+ discount
- This pattern holds across every category, not just one: at 30%+ discount, Office Supplies has a 100% loss rate (680 orders), Furniture 97.2% (542 orders), Technology 82.5% (171 orders, the most resilient of the three)

### 3. SQL — regional margin ranking (window functions)
Used `RANK() OVER (PARTITION BY Region ORDER BY margin)` to find the worst-margin sub-category within each region:

| Region | Worst sub-category | Margin |
|---|---|---|
| Central | Furnishings | -25.6% |
| East | Tables | -28.2% |
| South | Tables | -10.5% |
| West | Bookcases | -4.6% |

**Tables loses money in 3 of 4 regions** but is actually profitable in West (+1.75% margin, $1,483 profit on $84,755 sales) — a genuinely regional finding that a single overall ranking would have hidden. A blanket "stop discounting Tables" policy would be wrong specifically for the West region.

### 4. Hypothesis test — quantifying the discount threshold
Logistic regression predicting loss (binary) from discount level:
- Point-biserial correlation: r = 0.755 (strong positive relationship)
- Logistic regression: coefficient 27.73, p < 0.001 (highly significant)
- Pseudo R² = 0.64 — the model explains a substantial share of the loss/profit variation
- **Predicted loss probability crosses 50% at 26.3% discount** — closely matching the cliff observed visually in the EDA

### 5. Dashboard
Built in Tableau Public — combines the category/region profit breakdown with the discount/loss-rate cliff into one view.

- **Profit by Category & Region**: color-coded grid showing Furniture/Central as the sole loss-making cell (-2,871), against strong performers like Office Supplies/West (52,610) and Technology/East (47,462)
- **Discount vs Loss Rate**: bar chart confirming the statistical threshold found in the hypothesis test — loss rate stays under 15% through 20% discount (0.000, 0.043, 0.327, 0.137), then jumps to 91.6%+ at 30% and above, plateauing near 100%

[Live dashboard link](https://public.tableau.com/views/Tableau_17900853477400/SuperstoreProfitabilityDashboard?:language=en-US&publish=yes&:sid=&:redirect=auth&:display_count=n&:origin=viz_share_link)

## What I found
Discount level is a highly significant, strong predictor of whether an order loses money. Below ~20% discount, most orders are still profitable; above ~26%, an order is statistically more likely than not to be a loss, and by 30%+ discount, loss becomes close to certain (82–100% depending on category). This isn't a gradual erosion — it's a sharp threshold. The effect is broadly consistent across categories and regions, with two notable exceptions: Technology is comparatively more resilient to discounting than Furniture or Office Supplies, and Tables — a loss-maker in 3 of 4 regions — is actually profitable in the West region specifically.

## So what
**Cap discounts at 25%** as a default policy — this is a statistically defensible threshold (p < 0.001), not an arbitrary round number, since it sits right at the point where predicted loss probability crosses 50%. Enforce this most strictly on Office Supplies and Furniture, where the loss rate at high discount is most severe (97–100%). Technology can reasonably support a slightly higher ceiling given its relative resilience. Before applying a company-wide fix to Tables specifically, investigate what's different about pricing/discounting practices in the West region, since the sub-category is profitable there despite losing money everywhere else.

## Repo structure
```
/notebooks
  01_eda.ipynb              — cleaning, loss-rate discovery, discount cliff
  02_sql_analysis.ipynb     — SQLite window-function margin ranking by region
  03_hypothesis_test.ipynb  — logistic regression, statistical threshold
/data
  Sample - Superstore.csv
/dashboard
  superstore-dashboard.twbx — Tableau workbook (profit grid + discount/loss-rate chart)
README.md
```