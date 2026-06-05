## Sqoop Import with SQL Query Filter

Use `--query` instead of `--table` when you need custom SQL with WHERE conditions:

```bash
sqoop import \
  --connect jdbc:oracle:thin:@//oracle-host:1521/ORCL \
  --username your_oracle_user \
  --password your_oracle_password \
  --query "SELECT emp_id, emp_name, dept, salary, hire_date
           FROM EMPLOYEES
           WHERE dept = 'SALES'
           AND hire_date >= TO_DATE('2023-01-01', 'YYYY-MM-DD')
           AND \$CONDITIONS" \
  --split-by emp_id \
  --hive-import \
  --hive-table hive_db.employees_sales \
  --num-mappers 4 \
  --target-dir /user/hive/warehouse/employees_sales
```

---

### ⚠️ Critical Rule: `$CONDITIONS` is Mandatory

When using `--query`, you **must** append `AND \$CONDITIONS` (or `WHERE \$CONDITIONS`) at the end of your SQL. Sqoop injects this placeholder to split data across mappers.

```sql
-- If your query has a WHERE clause:
WHERE dept = 'SALES' AND \$CONDITIONS

-- If your query has NO WHERE clause:
WHERE \$CONDITIONS
```

---

### More Filter Examples

**1. Date range filter**
```bash
--query "SELECT * FROM ORDERS
         WHERE order_date BETWEEN TO_DATE('2024-01-01','YYYY-MM-DD')
                               AND TO_DATE('2024-12-31','YYYY-MM-DD')
         AND \$CONDITIONS"
```

**2. Filter with JOIN across tables**
```bash
--query "SELECT o.order_id, o.order_date, c.customer_name, o.amount
         FROM ORDERS o
         JOIN CUSTOMERS c ON o.customer_id = c.customer_id
         WHERE o.status = 'COMPLETED'
         AND \$CONDITIONS"
```

**3. Filter with IN list**
```bash
--query "SELECT * FROM PRODUCTS
         WHERE category IN ('ELECTRONICS', 'APPLIANCES')
         AND status = 'ACTIVE'
         AND \$CONDITIONS"
```

**4. Incremental filter using a timestamp**
```bash
--query "SELECT * FROM TRANSACTIONS
         WHERE updated_at > TO_TIMESTAMP('2024-06-01 00:00:00', 'YYYY-MM-DD HH24:MI:SS')
         AND \$CONDITIONS"
```

**5. Filter with subquery**
```bash
--query "SELECT * FROM EMPLOYEES
         WHERE dept_id IN (SELECT dept_id FROM DEPARTMENTS WHERE region = 'WEST')
         AND \$CONDITIONS"
```

---

### Key Rules When Using `--query`

| Rule | Detail |
|---|---|
| Use `--split-by` | Required with `--query` when `--num-mappers > 1`. Pick a numeric, indexed column |
| No `--table` | `--query` and `--table` are mutually exclusive |
| Quote carefully | Escape `$CONDITIONS` as `\$CONDITIONS` in bash to prevent shell expansion |
| Use `--target-dir` | Specify an HDFS output directory explicitly |
| Single mapper fallback | If no good split column exists, set `--num-mappers 1` and drop `--split-by` |
