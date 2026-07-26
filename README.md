# zepto-product-analysis-sql
SQL project analyzing Zepto product data using PostgreSQL. Includes data cleaning, aggregations, business insights, filtering, and pricing analysis.
# Zepto Product Analysis (PostgreSQL)

Exploratory SQL analysis of a Zepto product catalog — cleaning raw listing data and answering pricing, discount, and inventory questions using PostgreSQL.

Dataset source: [ADD LINK]
Rows: [ADD — run `select count(*) from ztable;` and drop the number in]

---

## Problem

Product export data from e-commerce platforms is rarely analysis-ready. In this dataset, prices are stored as paise instead of rupees, some rows have zero-value prices from bad exports, and identical products are split across multiple SKU rows (different pack sizes, different listings). Before any pricing or discount question can be answered honestly, the data has to be cleaned and validated.

This project does that cleaning in PostgreSQL, then runs category- and product-level analysis on top of it.

---

## Schema

```sql
create table ztable(
    id serial primary key,
    category varchar(255),
    name varchar(255) not null,
    mrp decimal(8,2),
    discountPercent decimal(5,2),
    availableQuantity integer,
    discountedSellingPrice decimal(8,2),
    weightInGms integer,
    outOfStock boolean,
    quantity integer
);
```

`category` and `name` were widened from `varchar(50)` to `varchar(255)` after the initial load truncated longer product names.

---

## Data validation and cleaning

- Checked every column for nulls before analysis, rather than assuming completeness:
  ```sql
  select * from ztable
  where name is null or category is null or mrp is null
     or discountPercent is null or availableQuantity is null
     or discountedSellingPrice is null or weightInGms is null
     or outOfStock is null or quantity is null;
  ```
- Removed rows with `mrp = 0` — these are broken listings, not genuinely free products.
- Converted `mrp` and `discountedSellingPrice` from paise to rupees (`/ 100.0`), since the source data stores currency in the smaller unit.
- Identified products duplicated across multiple SKU rows (same name, different listings — usually different pack sizes) using `GROUP BY name HAVING COUNT(id) > 1`.

---

## Questions answered

1. **How many distinct product categories exist, and how is stock distributed across them?**
   ```sql
   select distinct category, outOfStock, count(id)
   from ztable
   group by outOfStock, category
   order by outOfStock desc;
   ```

2. **Which product names appear as more than one SKU?**
   ```sql
   select name, count(id) as "Number of SKUs"
   from ztable
   group by name
   having count(id) > 1
   order by count(id) desc;
   ```

3. **Which products offer the best discounts to customers?**
   ```sql
   select distinct name, mrp, discountPercent
   from ztable
   order by discountPercent desc
   limit 10;
   ```

4. **Which high-value products are currently out of stock?**
   ```sql
   select distinct name, mrp
   from ztable
   where outOfStock = true and mrp > 300
   order by mrp desc;
   ```

5. **What is the estimated revenue per category?**
   ```sql
   select sum(discountedSellingPrice * availableQuantity) as total_revenue, category
   from ztable
   group by category
   order by total_revenue;
   ```
   Revenue here is `discounted price × available quantity` — a stock-based estimate, not actual sales data (this dataset has no order or transaction history).

6. **Which premium products (MRP > ₹500) are under-discounted (< 10%)?**
   ```sql
   select distinct name, mrp, discountPercent
   from ztable
   where mrp > 500 and discountPercent < 10
   order by mrp desc, discountPercent desc;
   ```

7. **Which categories offer the highest average discount?**
   ```sql
   select category, round(avg(discountPercent), 2) as avg_discount
   from ztable
   group by category
   order by avg_discount desc
   limit 5;
   ```

---

## Findings

[Run the queries above and put your real results here — this is the section a recruiter actually reads. Two or three sentences with real numbers beats the whole rest of the README. Example format:]

- Top discount category: **[name]**, averaging **[X]%** off MRP.
- **[N] products** priced above ₹300 are currently out of stock.
- **[Category]** leads estimated revenue at **₹[X]**, ahead of **[category]** in second.

---

## How to run

```bash
psql -U postgres -d your_db -f zepto_analysis.sql
```

The script is self-contained: it drops and recreates `ztable`, so re-running it resets the table before re-loading data.

---

## Notes / limitations

- Revenue figures are estimates based on listed stock, not confirmed sales — worth stating explicitly rather than letting a reader assume otherwise.
- `discountPercent` is taken as given in the source data rather than recalculated from `mrp` and `discountedSellingPrice`; a follow-up check would be to verify the two are consistent.

---

## Tools

PostgreSQL, SQL (aggregation, `GROUP BY` / `HAVING`, filtering, sorting, `DISTINCT`)
