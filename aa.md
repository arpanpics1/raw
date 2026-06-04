Part 1 — Write the 4 SQL scripts
Open Hue's Query Editor (SQL Editor tab) and write + save each script separately.

Script 1 — 01_create_staging.sql
sql
CREATE TABLE IF NOT EXISTS staging_orders (
    order_id    INT,
    customer_id INT,
    amount      DECIMAL(10,2),
    order_date  DATE
);


Script 2 — 02_pull_staging_data.sql
sql
INSERT INTO staging_orders
SELECT order_id, customer_id, amount, order_date
FROM source_orders
WHERE order_date = CURRENT_DATE - INTERVAL 1 DAY;


Script 3 — 03_update_historical.sql
sql
INSERT INTO historical_orders
SELECT * FROM staging_orders;


Script 4 — 04_drop_staging.sql
sql
DROP TABLE IF EXISTS staging_orders;
