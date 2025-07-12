# 🧾 **ORACLE DBA SESSION MANAGEMENT **

---

## **1. INTRODUCTION TO SESSION MANAGEMENT IN ORACLE**

### **Description:**

Session management is a critical aspect of Oracle Database administration, involving the monitoring and control of database sessions, resource contention, and ensuring that multiple sessions can perform operations concurrently without compromising data integrity. Effective session management addresses issues like blocking, deadlocks, and latch contention, ensuring smooth database operations and maintaining optimal performance. This guide outlines the key steps for managing sessions, resolving blocking situations, and addressing deadlocks and latch contention.

---

## **2. CONNECT AS SYSDBA AND CREATE A DEMO USER**

### **Description:**

Before performing any session management tasks, it’s essential to have a user with the necessary privileges. In this section, we create a demo user with adequate privileges to perform various data operations and simulate session management scenarios.

### **Step 2.1: Open PuTTY or Terminal**

```bash
$ sqlplus / as sysdba
```

### **Step 2.2: Execute SQL to Create Demo User**

```sql
CREATE USER demo_user IDENTIFIED BY demo123;
GRANT CONNECT, RESOURCE TO demo_user;
GRANT UNLIMITED TABLESPACE TO demo_user;
```

📘 **Explanation:**

* This step creates a new user, `demo_user`, with essential privileges and unlimited tablespace access to perform data operations and session management tasks.

---

## **3. CREATE EMPLOYEE TABLE**

### **Description:**

Once the user is created, the next step is to create a table where we will perform session-level locking and blocking simulations. The `employee` table will serve as a basic dataset for demonstrating these concepts.

### **Step 3.1: PuTTY Session for Demo User**

```bash
$ sqlplus demo_user/demo123
```

### **Step 3.2: Create Employee Table**

```sql
CREATE TABLE employee (
    emp_id   NUMBER PRIMARY KEY,
    name     VARCHAR2(100),
    dept     VARCHAR2(50),
    salary   NUMBER
);
```

📘 **Explanation:**

* This step creates an `employee` table to hold employee-related data, which will later be used to simulate locking, blocking, and deadlock scenarios.

---

## **4. INSERT 15 RECORDS INTO EMPLOYEE TABLE**

### **Description:**

To simulate real-world scenarios of concurrent access, we insert sample data into the `employee` table. These records will help demonstrate how row-level locks, session blocking, and other concurrency issues manifest in Oracle.

### **Step 4.1: Insert Data into Employee Table**

```sql
INSERT INTO employee VALUES (1,  'Amit Kumar',     'IT',      5500);
INSERT INTO employee VALUES (2,  'Emily Clark',    'HR',      5200);
INSERT INTO employee VALUES (3,  'Rahul Sharma',   'Finance', 6000);
INSERT INTO employee VALUES (4,  'Priya Verma',    'IT',      5700);
INSERT INTO employee VALUES (5,  'John Davis',     'HR',      5300);
INSERT INTO employee VALUES (6,  'Anjali Nair',    'Finance', 6100);
INSERT INTO employee VALUES (7,  'Michael Scott',  'IT',      5900);
INSERT INTO employee VALUES (8,  'Deepa Mehta',    'HR',      5400);
INSERT INTO employee VALUES (9,  'Suresh Menon',   'Finance', 6200);
INSERT INTO employee VALUES (10, 'Sarah Taylor',   'IT',      5600);
INSERT INTO employee VALUES (11, 'Karan Singh',    'HR',      5100);
INSERT INTO employee VALUES (12, 'Neha Dubey',     'Finance', 6300);
INSERT INTO employee VALUES (13, 'Arjun Shetty',   'IT',      5800);
INSERT INTO employee VALUES (14, 'Jessica Brown',  'HR',      5500);
INSERT INTO employee VALUES (15, 'Mohit Bansal',   'Finance', 6400);

COMMIT;
```

📘 **Explanation:**

* Inserts 15 records into the `employee` table, which will be used in subsequent sections to demonstrate how locking and blocking occur when multiple sessions access the same data.

---

## **5. SIMULATE BLOCKING SESSION**

### **Description:**

A blocking session occurs when one session holds a lock on a resource (such as a row or table), preventing other sessions from accessing or modifying that resource. This section demonstrates how one session can block another by holding a lock.

### **Step 5.1: PuTTY Session 1 – Locking a Row**

```bash
$ sqlplus demo_user/demo123
```

#### **Lock the Row (Blocking Session)**

```sql
SELECT * FROM employee WHERE emp_id = 5 FOR UPDATE;
```

Alternatively:

```sql
UPDATE employee SET salary = 5000 WHERE emp_id = 5;
SELECT SYS_CONTEXT('USERENV', 'SID') AS my_sid FROM dual;
```

📘 **Explanation:**

* This session locks the row with `emp_id = 5`, which prevents other sessions from modifying or accessing the row until the transaction is committed or rolled back.

### **Step 5.2: PuTTY Session 2 – Attempt to Update the Same Row (Blocked Session)**

```bash
$ sqlplus demo_user/demo123
```

#### **Attempt to Update the Same Row**

```sql
UPDATE employee SET salary = 10000 WHERE emp_id = 5;
```

📘 **Explanation:**

* **Session 2** will be blocked because **Session 1** has already locked the row with `emp_id = 5`. **Session 2** will remain blocked until **Session 1** completes the transaction.

---

## **6. CHECK SESSIONS – AS SYSDBA**

### **Description:**

As a DBA, it's important to monitor and identify which sessions are blocked and which are causing the blocking. This step provides the necessary SQL queries to examine session activity and resolve blocking issues.

### **Step 6.1: PuTTY Session 3 – Connect as SYSDBA**

```bash
$ sqlplus / as sysdba
```

### **Step 6.2: Identify Blocked and Blocking Sessions**

```sql
SELECT sid, serial#, username, blocking_session, prev_sql_id
FROM v$session
WHERE blocking_session IS NOT NULL;
```

📘 **Explanation:**

* This query identifies sessions that are being blocked (`SID`) and the sessions causing the blockage (`blocking_session`).

---

### **Step 6.3: View SQL of the Blocking Session**

```sql
SELECT sid, serial#, username, sql_id, prev_sql_id
FROM v$session
WHERE sid = <Blocking_Session_SID>;
```

📘 **Explanation:**

* This query provides details about the SQL statements running in the blocking session. You can use `SQL_ID` to trace the query causing the block.

---

### **Step 6.4: Get Full SQL Text for Current and Previous SQL**

```sql
-- Current SQL (if not NULL)
SELECT sql_text
FROM v$sql
WHERE sql_id = '<SQL_ID>';

-- Previous SQL
SELECT sql_text
FROM v$sql
WHERE sql_id = '<PREV_SQL_ID>';
```

📘 **Explanation:**

* This query retrieves the full SQL text for the blocking session, allowing you to diagnose the source of the block.

---

## **7. KILL BLOCKING SESSION**

### **Description:**

If a blocking session is causing significant delays or holding critical resources, it may be necessary to terminate the session. This section explains how to forcibly kill a blocking session to resolve a blockage.

### **Step 7.1: Kill the Blocking Session**

```sql
ALTER SYSTEM KILL SESSION '<SID>,<Serial#>' IMMEDIATE;
```

📘 **Explanation:**

* This command terminates the blocking session immediately, allowing the blocked session to proceed.

---

## **8. SIMULATE DEADLOCK**

### **Description:**

A deadlock occurs when two or more sessions are waiting on each other to release locks, creating a cycle of dependencies. Oracle automatically detects deadlocks and resolves them by rolling back one of the sessions. This section demonstrates how a deadlock occurs and how it’s handled by Oracle.

### **Step 8.1: Deadlock Simulation**

1. **Session 1:** Lock row 1 and try to update row 2.
2. **Session 2:** Lock row 2 and try to update row 1.

#### **Session 1:**

```sql
UPDATE employee SET salary = 5000 WHERE emp_id = 1;
```

#### **Session 2:**

```sql
UPDATE employee SET salary = 6000 WHERE emp_id = 2;
```

Now, run conflicting updates in both sessions:

* **Session 1**: `UPDATE employee SET salary = 7000 WHERE emp_id = 2;`
* **Session 2**: `UPDATE employee SET salary = 8000 WHERE emp_id = 1;`

📘 **Explanation:**

* Oracle automatically detects the deadlock and resolves it by rolling back one of the sessions involved in the cycle.

### **Step 8.2: Resolution of Deadlock**

Oracle resolves deadlocks by automatically rolling back one of the sessions. You can check the `alert.log` for more information on the deadlock:

```bash
tail -f /u01/app/oracle/diag/rdbms/<dbname>/<dbname>/trace/alert_<dbname>.log
```

Alternatively, use the following to find the sessions involved in a deadlock:

```sql
SELECT * FROM v$lock WHERE request > 0;
SELECT * FROM v$session WHERE sid = <sid>;
```

---

## **9. SIMULATE LATCH CONTENTION**

### **Description:**

Latch contention occurs when multiple sessions attempt to access the same memory structure, causing performance issues. Latches are synchronization mechanisms that protect critical memory areas like the shared pool and buffer cache. This section demonstrates how latch contention can impact performance and how to resolve it.

### **Step 9.1: Query Latch Information**

```sql
SELECT latch#, name, get_requests, wait_requests, sleeps
FROM v$latch
WHERE name = 'shared pool';
```

📘 **Explanation:**

* This query helps identify latch contention in the shared pool. A high number of `wait_requests` suggests that sessions are competing for access to shared memory structures.

---

### **Step 9.2: Resolution of Latch Contention**

1. **Increase Memory Allocation**:
   Increase the shared pool size to reduce contention:

   ```sql
   ALTER SYSTEM SET shared_pool_size = <new_size> SCOPE = BOTH;
   ```

2. **Optimize SQL Queries**:
   Inefficient queries can exacerbate latch contention. Use the SQL Tuning Advisor to optimize queries.

3. **Remove Unused PL/SQL**:
   Removing unused procedures and functions can help reduce contention for memory structures in the shared pool.

---

## **10. TYPES OF LOCKS, BLOCKS, AND LATCHES IN ORACLE**

### **Comparison Table:**

| **Type**               | **Description**                                                       | **Example**                                           | **Resolution**                                                             |
| ---------------------- | --------------------------------------------------------------------- | ----------------------------------------------------- | -------------------------------------------------------------------------- |
| **Row-Level Lock**     | Prevents concurrent modification of a row.                            | `UPDATE employee SET salary = 5000 WHERE emp_id = 5;` | Commit or rollback the transaction. Use `v$session` to identify blockages. |
| **Table-Level Lock**   | Prevents concurrent modification or access of a table.                | `LOCK TABLE employee IN EXCLUSIVE MODE;`              | Commit or rollback the transaction.                                        |
| **Exclusive Lock (X)** | Prevents all other sessions from accessing or modifying the resource. | `UPDATE employee SET salary = 5000 WHERE emp_id = 5;` | Wait until the session commits or rolls back.                              |
| **Shared Lock (S)**    | Allows others to read, but not modify the resource.                   | `SELECT * FROM employee WHERE emp_id = 5 FOR UPDATE;` | Wait until the session commits or rolls back.                              |
| **Blocking Session**   | A session holding locks on resources, blocking other sessions.        | Session 1 locks a row, Session 2 waits.               | Identify the blocking session using `v$session` and resolve it.            |
| **Latch**              | Lightweight synchronization mechanism to protect memory structures.   | `v$latch` query for memory contention.                | Increase memory allocation or optimize SQL.                                |
| **Deadlock**           | A situation where two sessions wait for each other indefinitely.      | Session 1 waits for Session 2’s lock and vice versa.  | Oracle detects and resolves deadlocks automatically.                       |

---

### **Final Notes**:

* **Locks** prevent conflicting access to database resources, ensuring data integrity and consistency.
* **Latches** protect Oracle's internal memory structures from concurrent access, improving system stability.
* **Blocking sessions** occur when one session holds a lock, preventing others from proceeding.
* **Deadlocks** are resolved by Oracle automatically by rolling back one of the involved sessions.
* **Latch contention** can be reduced by optimizing memory allocations and SQL queries.

