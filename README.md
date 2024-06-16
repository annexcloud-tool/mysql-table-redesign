# Production DB Schema Changes
## Approach 1(Shadow Table Approach)
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
