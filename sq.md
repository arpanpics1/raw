## Sqoop Script: Oracle to Hive

Here's a basic Sqoop import script:

```bash
sqoop import \
  --connect jdbc:oracle:thin:@//oracle-host:1521/ORCL \
  --username your_oracle_user \
  --password your_oracle_password \
  --table ORACLE_TABLE_NAME \
  --hive-import \
  --hive-table hive_db.hive_table_name \
  --hive-overwrite \
  --num-mappers 4 \
  --fields-terminated-by ',' \
  --lines-terminated-by '\n' \
  --null-string '\\N' \
  --null-non-string '\\N'
```

**Key parameters explained:**

- `--connect` — JDBC connection string to your Oracle DB
- `--num-mappers` — number of parallel map tasks (tune based on your cluster)
- `--hive-overwrite` — replaces existing Hive table data on each run
- `--null-string / --null-non-string` — handles NULLs correctly in Hive

---

### Incremental Load (for ongoing syncs)

```bash
sqoop import \
  --connect jdbc:oracle:thin:@//oracle-host:1521/ORCL \
  --username your_oracle_user \
  --password your_oracle_password \
  --table ORACLE_TABLE_NAME \
  --hive-import \
  --hive-table hive_db.hive_table_name \
  --incremental lastmodified \
  --check-column LAST_UPDATED_DATE \
  --last-value '2024-01-01 00:00:00' \
  --num-mappers 4
```

---

## Verifying Sqoop is Properly Set Up

**1. Check Sqoop version**
```bash
sqoop version
```

**2. Test Oracle JDBC connectivity**
```bash
sqoop list-tables \
  --connect jdbc:oracle:thin:@//oracle-host:1521/ORCL \
  --username your_oracle_user \
  --password your_oracle_password
```
This confirms Sqoop can reach Oracle and authenticate. You should see a list of tables printed.

**3. Check Hadoop connectivity**
```bash
hadoop fs -ls /
```
Sqoop depends on HDFS — this confirms Hadoop is reachable.

**4. Verify the Oracle JDBC driver is in place**
```bash
ls $SQOOP_HOME/lib/ | grep ojdbc
```
You should see something like `ojdbc8.jar`. If it's missing, download it from Oracle and drop it into `$SQOOP_HOME/lib/`.

**5. Run a quick eval query**
```bash
sqoop eval \
  --connect jdbc:oracle:thin:@//oracle-host:1521/ORCL \
  --username your_oracle_user \
  --password your_oracle_password \
  --query "SELECT COUNT(*) FROM ORACLE_TABLE_NAME"
```
If this returns a row count, your end-to-end Oracle connection is working.

**6. Verify Hive integration**
```bash
hive -e "SHOW DATABASES;"
```
Confirms Hive is accessible from the same node Sqoop runs on.

---

### Common Issues to Watch For

| Issue | Likely Cause |
|---|---|
| `ojdbc` class not found | JDBC driver missing from `$SQOOP_HOME/lib/` |
| Connection refused | Wrong host/port or Oracle listener not running |
| Permission denied on HDFS | Hadoop user doesn't have write access |
| Hive table not created | Missing `--hive-import` or Hive metastore misconfigured |
| Mapper failures | Too many `--num-mappers` for the split column cardinality |
