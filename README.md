# UK Household Food Analysis (1974–2023)
## Author
Irina Yadrikova  
Data Analyst Portfolio
## Project Description
This project analyses household food consumption and expenditure in the UK using PostgreSQL. It transforms and normalises raw Excel data, builds a relational data model, and runs analytical SQL queries to surface trends and insights.
## Real-World Scenario
**Project**: 
You work for a public policy agency or food industry group that wants to understand:

*“How has household food consumption and expenditure changed over time, and which food groups are becoming more or less cost-efficient?”*

*- What food groups saw the biggest change in quantity or cost between 2019–2022?*

*- Did food costs increase more during recession years?*

*- Which categories became less affordable (quantity down, cost up)?*

## Tools Used

- PostgreSQL (pgAdmin)
- SQL (data modeling & queries)
- VS Code
- Git + GitHub
---

## Data Sources
- `UK_HouseHold_consumption_24.xlsx`
- `UK_HouseHold_expendature.xlsx`

---

## Data Structure (Star Schema)

- **Fact Tables**:
  - `household_quantity_long`
  - `household_expenditure_long`

- **Dimension Tables**:
  - `dim_year` (time)
  - `dim_food` (food categories)

---

## Setup Steps

1. Create PostgreSQL database (`my_project_db`)
2. Use pgAdmin to:
   - Import cleaned CSVs
   - Run `CREATE TABLE` statements
   - Define primary/foreign keys
3. Load data using pgAdmin's **Import Tool**
4. Join fact tables to dimensions via SQL

---
### Create the Table - `CREATE TABLE` 

```sql
CREATE TABLE dim_year (
    year INTEGER PRIMARY KEY,
    decade INTEGER,
    is_recession BOOLEAN,
    notes TEXT
);

```
#### Composite Primary Key
```sql
CREATE TABLE household_quantity_long (
    code TEXT,
    code_level INTEGER,
    food_category TEXT,
    food_group TEXT,
    major_food_code TEXT,
    minor_food_code TEXT,
    units TEXT,
    rse_indicator TEXT,
    percent_change_2021_22 NUMERIC,
    percent_change_2019_20 NUMERIC,
    significance TEXT,
    trend_2019_20 TEXT,
    reliability_3yr_se NUMERIC,
    year INTEGER,
    quantity NUMERIC,
    PRIMARY KEY (code, year)
);

```
```sql
DROP TABLE IF EXISTS household_expenditure_long;

CREATE TABLE household_expenditure_long (
    code TEXT,
    code_level INTEGER,
    food_category TEXT,
    food_group TEXT,
    major_food_code TEXT,
    minor_food_code TEXT,
    units TEXT,
    rse_indicator TEXT,
    percent_change_2021_22 NUMERIC,
    percent_change_2019_20 NUMERIC,
    significance TEXT,
    trend_2019_20 TEXT,
    reliability_3yr_se NUMERIC,
    year INTEGER,
    expenditure NUMERIC,
    PRIMARY KEY (code, year)
);

```
#### Logical Relationships:
- household_quantity_long.year → dim_year.year
- household_expenditure_long.year → dim_year.year
- household_quantity_long.code + year ↔ - household_expenditure_long.code + year (not a formal FK, but used for joining)
```sql
-- Add FK from quantity to dim_year
ALTER TABLE household_quantity_long
ADD CONSTRAINT fk_quantity_year
FOREIGN KEY (year)
REFERENCES dim_year(year);

-- Add FK from expenditure to dim_year
ALTER TABLE household_expenditure_long
ADD CONSTRAINT fk_expenditure_year
FOREIGN KEY (year)
REFERENCES dim_year(year);
```

```sql
CREATE TABLE dim_food (
    code TEXT PRIMARY KEY,
    code_level INTEGER,
    food_category TEXT,
    food_group TEXT,
    major_food_code TEXT,
    minor_food_code TEXT,
    units TEXT
);
```

## 📌 SQL Highlights


### Create a Combined Fact Table
Join quantity and expenditure into a single view:

```sql
CREATE VIEW food_fact AS
SELECT
    q.code,
    q.food_category,
    q.food_group,
    q.major_food_code,
    q.minor_food_code,
    q.units,
    q.year,
    q.quantity,
    e.expenditure,
    y.decade,
    y.is_recession
FROM household_quantity_long q
JOIN household_expenditure_long e
    ON q.code = e.code AND q.year = e.year
JOIN dim_year y
    ON q.year = y.year;
    ```
🎯 This is now your core analytical dataset: one row per food_code x year, with all relevant data.
STEP 2: Calculate Cost per Unit Over Time
Add a metric that shows £ per unit consumed:
```sql
SELECT
    food_group,
    food_category,
    year,
    ROUND(SUM(expenditure) / NULLIF(SUM(quantity), 0), 2) AS cost_per_unit
FROM food_fact
GROUP BY food_group, food_category, year
ORDER BY food_group, food_category, year;
```

✅ This shows how expensive it is to consume a unit of each food category — a KPI a senior analyst would highlight.
STEP 3: Identify Trends Over Time
See how cost per unit has changed over decades or key events:
``` sql
SELECT
    decade,
    food_group,
    ROUND(AVG(expenditure / NULLIF(quantity, 0)), 2) AS avg_cost_per_unit
FROM food_fact
GROUP BY decade, food_group
ORDER BY decade, food_group;
```


📊 Business use: "Which food groups got more expensive in the 2020s?"
STEP 4: Flag Inflation-Driven Inefficiencies
Flag cases where expenditure is increasing but quantity is decreasing:
```sql
SELECT
    code,
    food_category,
    year,
    quantity,
    expenditure,
    LAG(quantity) OVER (PARTITION BY code ORDER BY year) AS prev_quantity,
    LAG(expenditure) OVER (PARTITION BY code ORDER BY year) AS prev_expenditure
FROM food_fact
```
QUALIFY
    (expenditure > prev_expenditure) AND
    (quantity < prev_quantity);
🧠 This is senior-level diagnostic logic: highlighting potential inflationary trends or consumer behavior shifts.
STEP 5: Rank Food Groups by Volatility
Which food groups have the most unstable cost per unit?
```sql
SELECT
    food_group,
    ROUND(STDDEV_POP(expenditure / NULLIF(quantity, 0)), 2) AS cost_volatility
FROM food_fact
GROUP BY food_group
ORDER BY cost_volatility DESC;
```

📈 Use this insight to guide policy around price stability or subsidies.
BONUS: Create an Executive Summary Table
```sql 
SELECT
    food_group,
    COUNT(DISTINCT code) AS food_items,
    ROUND(AVG(quantity), 2) AS avg_quantity,
    ROUND(AVG(expenditure), 2) AS avg_expenditure,
    ROUND(AVG(expenditure / NULLIF(quantity, 0)), 2) AS avg_cost_per_unit
FROM food_fact
GROUP BY food_group
ORDER BY avg_cost_per_unit DESC;
```


### 📈 Trend: Cost per Unit Over Time
```sql
SELECT food_category, year,
  ROUND(SUM(expenditure)/NULLIF(SUM(quantity), 0), 2) AS cost_per_unit
FROM food_fact
GROUP BY food_category, year
ORDER BY year;
```

###📉 Recession Impact Analysis
```sql
SELECT year, is_recession, AVG(quantity) AS avg_quantity
FROM food_fact
JOIN dim_year USING (year)
GROUP BY year, is_recession;
```
