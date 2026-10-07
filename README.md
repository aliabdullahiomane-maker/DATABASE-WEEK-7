# Sales Database Security and Audit Logging Lab

## Student Information

- Name: Ali Abdullahi
- Database: `sales`
- Database system: MySQL Community Server
- MySQL version: 8.0.46
- Operating system: Windows
- Shell: Windows PowerShell
- Main table audited: `student`

---

## 1. Lab Objective

The objective of this lab was to implement:

1. Trigger-based audit logging.
2. JSON storage of old and new row values.
3. A hierarchical category tree.
4. Recursive common table expressions.
5. Versioned database migrations with Flyway.
6. MySQL roles and least-privilege permissions.
7. A restricted API database user.

---

## 2. Environment Setup

The MySQL command-line client was started from Windows PowerShell using:

```powershell
& "C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql.exe" -u root -p
```

The database was selected with:

```sql
USE sales;
```

The database and MySQL version were confirmed using:

```sql
SELECT DATABASE(), VERSION();
```

Result:

```text
DATABASE(): sales
VERSION(): 8.0.46
```

---

## 3. Existing Database Tables

The database already contained the following tables:

```text
customers
employees
offices
orderdetails
orders
payments
productlines
products
student
```

The `student` table was inspected with:

```sql
DESCRIBE student;
```

Structure:

| Column | Type | Key |
|---|---|---|
| `id` | `int` | Primary key |
| `fullName` | `varchar(100)` | None |
| `age` | `int` | None |

Initial student data included:

```text
1 | Alice Johnson | 19
2 | Bob Smith     | 20
3 | Carol Davis   | 22
```

---

# 4. Audit Logging

## 4.1 Audit Table

The audit table was created to record:

- The affected table.
- The operation.
- The old row.
- The new row.
- The user who made the change.
- The time of the change.

SQL used:

```sql
CREATE TABLE audit_log (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    tbl VARCHAR(128) NOT NULL,
    op VARCHAR(20) NOT NULL,
    old_row JSON NULL,
    new_row JSON NULL,
    changed_by VARCHAR(255) NOT NULL,
    changed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

The `old_row` and `new_row` columns use MySQL's native `JSON` data type.

---

## 4.2 Insert Audit Trigger

```sql
DELIMITER $$

CREATE TRIGGER trg_student_insert_audit
AFTER INSERT ON student
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (
        tbl,
        op,
        old_row,
        new_row,
        changed_by
    )
    VALUES (
        'student',
        'INSERT',
        NULL,
        JSON_OBJECT(
            'id', NEW.id,
            'fullName', NEW.fullName,
            'age', NEW.age
        ),
        CURRENT_USER()
    );
END$$

DELIMITER ;
```

---

## 4.3 Update Audit Trigger

```sql
DELIMITER $$

CREATE TRIGGER trg_student_update_audit
AFTER UPDATE ON student
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (
        tbl,
        op,
        old_row,
        new_row,
        changed_by
    )
    VALUES (
        'student',
        'UPDATE',
        JSON_OBJECT(
            'id', OLD.id,
            'fullName', OLD.fullName,
            'age', OLD.age
        ),
        JSON_OBJECT(
            'id', NEW.id,
            'fullName', NEW.fullName,
            'age', NEW.age
        ),
        CURRENT_USER()
    );
END$$

DELIMITER ;
```

---

## 4.4 Delete Audit Trigger

```sql
DELIMITER $$

CREATE TRIGGER trg_student_delete_audit
AFTER DELETE ON student
FOR EACH ROW
BEGIN
    INSERT INTO audit_log (
        tbl,
        op,
        old_row,
        new_row,
        changed_by
    )
    VALUES (
        'student',
        'DELETE',
        JSON_OBJECT(
            'id', OLD.id,
            'fullName', OLD.fullName,
            'age', OLD.age
        ),
        NULL,
        CURRENT_USER()
    );
END$$

DELIMITER ;
```

---

## 4.5 Trigger Verification

The triggers were verified with:

```sql
SHOW TRIGGERS LIKE 'student';
```

The following triggers were present:

```text
trg_student_insert_audit
trg_student_update_audit
trg_student_delete_audit
```

---

## 4.6 Audit Test

The update operation was tested with:

```sql
UPDATE student
SET fullName = 'Kofi M.'
WHERE id = 1;
```

The delete operation was tested with:

```sql
DELETE FROM student
WHERE id = 3;
```

The audit records were inspected with:

```sql
SELECT
    tbl,
    op,
    JSON_UNQUOTE(JSON_EXTRACT(old_row, '$.fullName')) AS was,
    JSON_UNQUOTE(JSON_EXTRACT(new_row, '$.fullName')) AS now,
    changed_by,
    changed_at
FROM audit_log
ORDER BY changed_at DESC, id DESC;
```

Result:

```text
student | DELETE | Carol Davis   | NULL   | root@localhost
student | UPDATE | Alice Johnson | Kofi M. | root@localhost
```

This confirmed that both update and delete operations were recorded correctly.

---

# 5. Category Tree

## 5.1 Categories Table

The category table uses a self-referencing foreign key. The `parent_id` column points to the parent category in the same table.

```sql
CREATE TABLE categories (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    parent_id INT NULL,
    CONSTRAINT fk_categories_parent
        FOREIGN KEY (parent_id)
        REFERENCES categories(id)
);
```

---

## 5.2 Category Data

The following hierarchy was created:

```text
Electronics
├── Computers
│   └── Laptops
└── Phones
```

The root category was inserted with:

```sql
INSERT INTO categories (name, parent_id)
VALUES ('Electronics', NULL);
```

The child categories were then inserted using the parent IDs.

---

## 5.3 Recursive CTE Query

The hierarchy was queried using:

```sql
WITH RECURSIVE tree AS (
    SELECT
        id,
        name,
        parent_id,
        0 AS depth,
        CAST(name AS CHAR(1000)) AS sort_path
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    SELECT
        c.id,
        c.name,
        c.parent_id,
        t.depth + 1,
        CONCAT(t.sort_path, ' > ', c.name)
    FROM categories c
    JOIN tree t
        ON c.parent_id = t.id
)
SELECT
    CONCAT(REPEAT('  ', depth), name) AS category,
    depth
FROM tree
ORDER BY sort_path;
```

Expected output:

```text
Electronics
  Computers
    Laptops
  Phones
```

The recursive query successfully displayed the category hierarchy.

---

# 6. Versioned Migrations

## 6.1 Migration Folder

The migration files were organized in the following folder:

```text
migrations/
├── V1__core_tables.sql
├── V2__audit_log.sql
└── V3__categories.sql
```

The files use Flyway's versioned naming format:

```text
V<version>__<description>.sql
```

---

## 6.2 Migration Descriptions

### V1__core_tables.sql

This migration represents the core application tables, including the `student` table.

### V2__audit_log.sql

This migration creates:

- The `audit_log` table.
- The insert audit trigger.
- The update audit trigger.
- The delete audit trigger.

### V3__categories.sql

This migration creates:

- The `categories` table.
- The self-referencing parent foreign key.

---

## 6.3 Flyway Database URL

The target database URL is:

```text
jdbc:mysql://localhost:3306/sales
```

Example PowerShell commands:

```powershell
flyway `
  -url="jdbc:mysql://localhost:3306/sales" `
  -user="root" `
  -locations="filesystem:.\migrations" `
  migrate
```

To inspect migration status:

```powershell
flyway `
  -url="jdbc:mysql://localhost:3306/sales" `
  -user="root" `
  -locations="filesystem:.\migrations" `
  info
```

The Flyway migration files are stored separately from the manual SQL execution so that schema changes can be versioned and applied in order.

---

# 7. Least-Privilege Security

## 7.1 Application Roles

Two roles were created:

```sql
CREATE ROLE IF NOT EXISTS 'app_read'@'localhost';

CREATE ROLE IF NOT EXISTS 'app_write'@'localhost';
```

The read role is intended for users that only need to view data.

The write role is intended for the application API and allows normal data changes without schema administration.

---

## 7.2 Read Role Permissions

```sql
GRANT SELECT
ON sales.*
TO 'app_read'@'localhost';
```

The `app_read` role can read data from the `sales` database.

---

## 7.3 Write Role Permissions

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON sales.*
TO 'app_write'@'localhost';
```

The `app_write` role can:

- Read rows.
- Insert rows.
- Update rows.
- Delete rows.

The role does not have permission to:

- Create tables.
- Alter tables.
- Drop tables.
- Create users.
- Grant privileges.

The role grants were verified with:

```sql
SHOW GRANTS FOR 'app_write'@'localhost';
```

Result:

```text
GRANT SELECT, INSERT, UPDATE, DELETE
ON `sales`.* TO `app_write`@`localhost`
```

---

# 8. API User

## 8.1 Create the API User

```sql
CREATE USER IF NOT EXISTS 'api'@'localhost'
IDENTIFIED BY 'strong-secret';
```

The password was used only for this local lab environment.

---

## 8.2 Assign the Write Role

```sql
GRANT 'app_write'@'localhost'
TO 'api'@'localhost';
```

The role was enabled by default:

```sql
SET DEFAULT ROLE 'app_write'@'localhost'
TO 'api'@'localhost';
```

---

## 8.3 API Login

The API user connected from Windows PowerShell using:

```powershell
& "C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql.exe" -u api -p sales
```

The API session was verified with:

```sql
SELECT CURRENT_USER(), DATABASE(), CURRENT_ROLE();
```

Result:

```text
CURRENT_USER(): api@localhost
DATABASE(): sales
CURRENT_ROLE(): app_write@localhost
```

This confirmed that the API user was connected to the correct database with the correct role active.

---

# 9. Security Testing

## 9.1 Read Test

The API user successfully ran:

```sql
SELECT * FROM student;
```

Result:

```text
1 | Kofi M.   | 19
2 | Bob Smith | 21
```

---

## 9.2 Update Test

The API user successfully ran:

```sql
UPDATE student
SET age = 21
WHERE id = 2;
```

The command completed successfully.

The result showed:

```text
Rows matched: 1
Changed: 0
Warnings: 0
```

The row already contained the value `21`, so no data value changed. The account still had permission to execute the update operation.

---

## 9.3 Audit Verification After API Update

The audit record was inspected with:

```sql
SELECT
    tbl,
    op,
    JSON_UNQUOTE(JSON_EXTRACT(old_row, '$.fullName')) AS old_name,
    JSON_UNQUOTE(JSON_EXTRACT(new_row, '$.fullName')) AS new_name,
    changed_by,
    changed_at
FROM audit_log
ORDER BY id DESC
LIMIT 1;
```

Result:

```text
student | UPDATE | Bob Smith | Bob Smith | root@localhost
```

This confirms that the update trigger recorded the operation.

---

## 9.4 Schema Permission Test

The API user attempted to create a table:

```sql
CREATE TABLE security_test (
    id INT
);
```

The command was denied with:

```text
ERROR 1142 (42000): CREATE command denied to user 'api'@'localhost' for table 'security_test'
```

This confirms that the API user can modify application data but cannot create database tables.

---

# 10. Final Results

The following lab requirements were completed:

- Trigger-based audit logging.
- JSON storage of old and new row values.
- Insert, update, and delete auditing.
- Hierarchical category modeling.
- Recursive category-tree querying.
- Versioned migration file organization.
- Read and write roles.
- API user creation.
- Least-privilege access control.
- Successful denial of unauthorized table creation.

---

# 11. Submission Files

The recommended submission structure is:

```text
sales-audit-lab/
├── README.md
├── 01_audit_logging.sql
├── 02_category_tree.sql
├── 03_security.sql
├── migrations/
│   ├── V1__core_tables.sql
│   ├── V2__audit_log.sql
│   └── V3__categories.sql
└── screenshots/
    ├── audit-triggers.png
    ├── audit-results.png
    ├── category-tree.png
    ├── role-grants.png
    └── security-denied.png
```

---

# 12. Important Security Note

The password `strong-secret` was used for this lab only. Before using this database in a real application, change it:

```sql
ALTER USER 'api'@'localhost'
IDENTIFIED BY 'replace-with-a-strong-private-password';
```

Do not commit database passwords to GitHub or include real passwords in a public repository.
