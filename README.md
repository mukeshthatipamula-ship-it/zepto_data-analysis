# Zepto — MSSQL Data Analysis

## 1. Setup: create database and table

```sql
-- create database and switch context
CREATE DATABASE zepto_prj;
GO
USE zepto_prj;
GO

-- drop table if exists (T-SQL pattern)
IF OBJECT_ID('dbo.zepto', 'U') IS NOT NULL
    DROP TABLE dbo.zepto;
GO

-- create table in Microsoft SQL Server
CREATE TABLE dbo.zepto (
    sku_id INT IDENTITY(1,1) PRIMARY KEY,
    category VARCHAR(120) NULL,
    name VARCHAR(150) NOT NULL,
    mrp DECIMAL(10,2) NULL,
    discountPercent DECIMAL(6,2) NULL,
    availableQuantity INT NULL,
    discountedSellingPrice DECIMAL(10,2) NULL,
    weightInGms INT NULL,
    outOfStock BIT NULL,
    quantity INT NULL
);

## 2. Basic data exploration queries

### 2.1 Count rows

```sql
SELECT COUNT(*) AS total_rows
FROM dbo.zepto;
```

### 2.2 Sample data (first 10 rows)

```sql
SELECT TOP (10) *
FROM dbo.zepto;
```

### 2.3 Find rows with NULLs in important columns

```sql
SELECT *
FROM dbo.zepto
WHERE name IS NULL
   OR category IS NULL
   OR mrp IS NULL
   OR discountPercent IS NULL
   OR discountedSellingPrice IS NULL
   OR weightInGms IS NULL
   OR availableQuantity IS NULL
   OR outOfStock IS NULL
   OR quantity IS NULL;
```

### 2.4 Distinct categories

```sql
SELECT DISTINCT category
FROM dbo.zepto
ORDER BY category;
```

### 2.5 Products in-stock vs out-of-stock

```sql
SELECT outOfStock,
       COUNT(sku_id) AS sku_count
FROM dbo.zepto
GROUP BY outOfStock;
```

### 2.6 Names appearing multiple times (duplicate SKUs)

```sql
SELECT name,
       COUNT(sku_id) AS number_of_skus
FROM dbo.zepto
GROUP BY name
HAVING COUNT(sku_id) > 1
ORDER BY COUNT(sku_id) DESC;
```

---

## 3. Data cleaning steps

### 3.1 Identify products where price is zero

```sql
SELECT *
FROM dbo.zepto
WHERE mrp = 0 OR discountedSellingPrice = 0;
```

### 3.2 Remove rows with `mrp = 0` (use with care — consider backing up first)

```sql
DELETE FROM dbo.zepto
WHERE mrp = 0;
```

> **Safety tip:** wrap destructive operations in a transaction during development:

```sql
BEGIN TRANSACTION;
DELETE FROM dbo.zepto WHERE mrp = 0;
-- SELECT COUNT(*) FROM dbo.zepto; -- preview
-- ROLLBACK TRANSACTION; -- or COMMIT after verification
COMMIT TRANSACTION;
```

### 3.3 Convert paise to rupees (if source values are in paise)

```sql
-- ensure decimal division and preserve precision
UPDATE dbo.zepto
SET mrp = CAST(mrp AS DECIMAL(10,2)) / 100.0,
    discountedSellingPrice = CAST(discountedSellingPrice AS DECIMAL(10,2)) / 100.0;
```

**Note:** If `mrp` or `discountedSellingPrice` are already stored in rupees, DO NOT run the conversion.

---

## 4. Analysis queries (business questions)

### Q1 — Top 10 best-value products by discount percentage

```sql
SELECT DISTINCT TOP (10) name, mrp, discountPercent
FROM dbo.zepto
ORDER BY discountPercent DESC;
```

**Explanation:** high `discountPercent` suggests strong markdown; `DISTINCT` removes exact duplicate rows by name.

### Q2 — High MRP products that are out of stock (MRP > ₹300)

```sql
SELECT DISTINCT name, mrp
FROM dbo.zepto
WHERE outOfStock = 1
  AND mrp > 300
ORDER BY mrp DESC;
```

### Q3 — Estimated revenue per category (using availableQuantity)

```sql
SELECT category,
       SUM(discountedSellingPrice * CAST(availableQuantity AS DECIMAL(18,2))) AS total_revenue
FROM dbo.zepto
GROUP BY category
ORDER BY total_revenue DESC; -- descending to show largest revenue first
```

**Assumption:** `availableQuantity` represents the sellable units in stock.

### Q4 — Products with MRP > ₹500 but < 10% discount

```sql
SELECT DISTINCT name, mrp, discountPercent
FROM dbo.zepto
WHERE mrp > 500
  AND discountPercent < 10
ORDER BY mrp DESC, discountPercent DESC;
```

### Q5 — Top 5 categories with highest average discount

```sql
SELECT TOP (5) category,
       ROUND(AVG(discountPercent), 2) AS avg_discount
FROM dbo.zepto
GROUP BY category
ORDER BY avg_discount DESC;
```

### Q6 — Price per gram for products >= 100 g, sorted by best value

```sql
SELECT DISTINCT name, weightInGms, discountedSellingPrice,
       ROUND(discountedSellingPrice / NULLIF(CAST(weightInGms AS DECIMAL(18,4)), 0), 2) AS price_per_gram
FROM dbo.zepto
WHERE weightInGms >= 100
ORDER BY price_per_gram ASC; -- lowest price/gram = best value
```

**Note:** `NULLIF(..., 0)` prevents divide-by-zero.

### Q7 — Weight category buckets (Low, Medium, Bulk)

```sql
SELECT DISTINCT name, weightInGms,
       CASE WHEN weightInGms < 1000 THEN 'Low'
            WHEN weightInGms < 5000 THEN 'Medium'
            ELSE 'Bulk'
       END AS weight_category
FROM dbo.zepto;
```

### Q8 — Total inventory weight per category

```sql
SELECT category,
       SUM(CAST(weightInGms AS BIGINT) * CAST(availableQuantity AS BIGINT)) AS total_weight_in_grams
FROM dbo.zepto
GROUP BY category
ORDER BY total_weight_in_grams DESC;
```

**Note:** multiplications use `BIGINT` to avoid overflow when numbers are large.

---

## 5. Performance & production considerations

* Add indexes to speed up common filters and aggregations. Example:

```sql
CREATE INDEX IX_zepto_category ON dbo.zepto(category);
CREATE INDEX IX_zepto_name ON dbo.zepto(name);
CREATE INDEX IX_zepto_mrp ON dbo.zepto(mrp);
```

* Consider a computed column for `price_per_gram` if queries use it often, and index the computed column.
* For very large tables, maintain statistics and consider partitioning by category or another business key.
* When performing large UPDATE/DELETE operations, batch them (e.g. with `TOP (10000)`) to avoid long locks.



