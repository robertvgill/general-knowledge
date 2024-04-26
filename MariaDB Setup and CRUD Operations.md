# MariaDB Setup and CRUD Operations

# Table of Contents

1. [Introduction](#introduction)
2. [Setting Up MariaDB](#setting-up-mariadb)
3. [Creating a Database and Table](#creating-a-database-and-table)
4. [CRUD Operations](#crud-operations)
5. [Conclusion](#conclusion)

## Introduction
This guide will walk you through the steps on setting up a MariaDB engine, creating a database and a table, and performing common CRUD (Create, Read, Update, Delete) operations.

## Setting Up MariaDB

**1. Install MariaDB:**

Follow the installation instructions for your operating system from the [official MariaDB website](https://mariadb.org/download/).

**2. Start the MariaDB Server:**

Once installed, start the MariaDB server. The method may vary depending on your operating system. For instance:
```
sudo systemctl start mariadb
```

**3. Access the MariaDB Shell:**
Use the following command to access the MariaDB shell:
```
mysql -u root -p
```
Enter the root password when prompted.

## Creating a Database and Table

**4. Create a Database:**
Within the MariaDB shell, execute the following SQL command to create a new database:
```sql
CREATE DATABASE my_database;
```

**5. Use the Database:**
Switch to the newly created database:
```sql
USE my_database;
```

**6. Create a Table:**
Define a table structure and create it within the selected database. For example:
```sql
CREATE TABLE my_table (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255),
    email VARCHAR(255)
);
```

## CRUD Operations

**7. Insert Data (Create):**

To add records into the table, use the `INSERT INTO` statement. For instance:
```sql
INSERT INTO my_table (name, email) VALUES ('John Doe', 'jdoe@example.com');
```

**8. Retrieve Data (Read):**

Retrieve data from the table using the `SELECT` statement. Example:
```sql
SELECT * FROM my_table;
```
Example Output:
```sql
MariaDB [my_database]> SELECT * FROM my_table;
+----+----------+----------------------------+
| id | name     | email                      |
+----+----------+----------------------------+
|  1 | John Doe | jdoe@example.com |
+----+----------+----------------------------+
1 row in set (0.007 sec)

MariaDB [my_database]>
```

**9. Update Data (Update):**

Update existing records using the `UPDATE` statement. For example:
```sql
UPDATE my_table SET email = 'john.doe@example.com' WHERE id = 1;
```

**10. Delete Data (Delete):**

Remove records from the table using the `DELETE` statement. Example:
```sql
DELETE FROM my_table WHERE id = 1;
```

## Conclusion
You have successfully set up a MariaDB engine, created a database, and a table, and learned how to perform common CRUD operations.