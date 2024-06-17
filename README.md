# Production DB Schema Changes
## Adding Indexes in MySQL
This document provides an overview of the different approaches available for adding indexes in MySQL.

### Overview
Indexes are crucial for improving the performance of database queries by allowing the database engine to find rows quickly. MySQL provides several algorithms for adding indexes to a table, each with specific characteristics and use cases.

### Goal
* Low __Latency and Downtime__ with a __large Table Size__

## Approach 1 (Direct Add Index to MySQL)

### Algorithms for Adding Indexes
1. __INPLACE__
    * **Description:** Modifies the table in place without making a full copy. Generally, it does not lock the table for long periods.
    * **Use Case:** Preferred for large tables due to efficiency and minimal downtime. Suitable for adding an index without significantly affecting table availability for reads and writes.
2. __COPY__
    * **Description:** Creates a full copy of the table with the new index and then replaces the original table with the new one. Requires more space and can be slower.
    * **Use Case:** Suitable for smaller tables where downtime and additional space requirements are acceptable. Often used when INPLACE is not supported.
3. __INSTANT__
    * **Description:** Adds indexes instantly without modifying the table data. Does not require a table copy or significant downtime. Availability depends on the table's storage engine and MySQL version.
    * **Use Case:** Ideal for scenarios where immediate availability is crucial and downtime needs to be minimized. Available only in specific scenarios and MySQL versions.
4. __DEFAULT__
    * **Description:** MySQL uses the default method for adding indexes when no specific algorithm is mentioned. The default method varies based on the storage engine, MySQL version, and specific table characteristics.
    * **Use Case:** General fallback when specific algorithm requirements are not critical, relying on MySQL to choose the most appropriate method.

```sql
-- Using INPLACE algorithm
ALTER TABLE my_table ADD INDEX my_index (my_column) ALGORITHM=INPLACE;

-- Using COPY algorithm
ALTER TABLE my_table ADD INDEX my_index (my_column) ALGORITHM=COPY;

-- Using INSTANT algorithm
ALTER TABLE my_table ADD INDEX my_index (my_column) ALGORITHM=INSTANT;
```
### Considerations for Choosing an Algorithm
1. __Table Size:__ For large tables, INPLACE or INSTANT are preferred to minimize downtime and avoid the overhead of copying the entire table.
2. __Table Locking:__ COPY algorithm locks the table for a longer period, impacting availability. INPLACE and INSTANT generally provide better concurrency.

### Recommendations
1. __[&check;]__ Use INPLACE for most scenarios involving large tables where you need to add an index without significant downtime.
2. __[&cross;]__ Use INSTANT when supported and immediate index creation is necessary.
3. __[&cross;]__ Use COPY for small tables or when INPLACE and INSTANT are not supported.


## Approach 2 (Percona Toolkit)
1. Use tool like pt-online-schema-change from Percona Toolkit to add indexes without downtime.
2. It creates a copy of the table, applies the changes, and swaps the tables.
3. **pt-online-schema-change** [documentation] (https://docs.percona.com/percona-toolkit/pt-online-schema-change.html)
```shell
pt-online-schema-change --alter "ADD INDEX (column_name)" D=database,t=table_name --chunk-size 1000 --execute
```

## Approach 3 (Shadow Table Approach)
1. Create a new table (new_table) with the desired schema changes based on the old table (old_table).
```python
CREATE TABLE new_table LIKE old_table;
```
2. Migrate Data Incrementally
  * Migrate data from the old table to the new table in batches to minimize locking and performance impact.
  * python script for data migration

```python
import mysql.connector
import time

batch_size = 1000  # Number of rows per batch
last_processed_id = 0  # Track the last processed ID
sleep_time = 1  # Time to sleep between batches to reduce load

# Database connection
conn = mysql.connector.connect(
    host='your_host',
    user='your_user',
    password='your_password',
    database='your_database'
)
cursor = conn.cursor()

while True:
    # Fetch a batch of data
    select_query = f"""
    SELECT * FROM old_table
    WHERE id > {last_processed_id}
    ORDER BY id ASC
    LIMIT {batch_size}
    """
    cursor.execute(select_query)
    rows = cursor.fetchall()
    
    if not rows:
        break  # Exit loop when no more rows to process
    
    # Insert batch into new table
    for row in rows:
        insert_query = """
        INSERT INTO new_table (column1, column2, new_column)
        VALUES (%s, %s, %s)  -- Adjust as per your new table schema
        """
        cursor.execute(insert_query, (row[1], row[2], None))  # Adjust as per columns

    # Commit transaction
    conn.commit()
    
    # Update last processed ID
    last_processed_id = rows[-1][0]  # Assuming 'id' is the first column
    
    # Sleep to reduce load
    time.sleep(sleep_time)

cursor.close()
conn.close()
```

3. Synchronize Data During Migration
  * Ensure any new changes to old_table are also reflected in new_table during the migration period. This can be achieved using triggers or by dual writes.
  * Create triggers to ensure any changes to the old table are also applied to the new table:

```sql
DELIMITER //

CREATE TRIGGER after_old_table_insert
AFTER INSERT ON old_table
FOR EACH ROW
BEGIN
    INSERT INTO new_table (column1, column2, new_column)
    VALUES (NEW.column1, NEW.column2, NULL);  -- Adjust as per your schema
END;
//

CREATE TRIGGER after_old_table_update
AFTER UPDATE ON old_table
FOR EACH ROW
BEGIN
    UPDATE new_table
    SET column1 = NEW.column1, column2 = NEW.column2  -- Adjust as per your schema
    WHERE id = NEW.id;
END;
//

CREATE TRIGGER after_old_table_delete
AFTER DELETE ON old_table
FOR EACH ROW
BEGIN
    DELETE FROM new_table WHERE id = OLD.id;
END;
//

DELIMITER ;
```

4. Switch to the New Table
  * RENAME TABLE old_table TO old_table_backup;
  * RENAME TABLE new_table TO old_table;
