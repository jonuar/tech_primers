# PL/SQL Cheatsheet

## Mental Model

PL/SQL (Procedural Language/SQL) is Oracle's procedural extension to SQL. The core idea: embed SQL inside a block-structured language with variables, conditionals, loops, and exception handling. Code runs **inside the database engine** — no round trips, no serialization overhead. The unit of work is the **block** (`DECLARE → BEGIN → EXCEPTION → END`), and larger units are **procedures**, **functions**, **packages**, and **triggers**. Everything is transactional by default.

---

## Minimal Setup

```sql
-- Connect with SQL*Plus
sqlplus username/password@hostname:1521/service_name

-- Or SQL Developer / DBeaver (GUI)

-- Enable output (required to see DBMS_OUTPUT)
SET SERVEROUTPUT ON;

-- Anonymous block — runs once, not stored
BEGIN
    DBMS_OUTPUT.PUT_LINE('Hello, Oracle!');
END;
/  -- slash executes the block
```

---

## Core Concepts

### 1. Block Structure

```sql
DECLARE
    -- Variable declarations
    v_name      VARCHAR2(100);
    v_salary    NUMBER(10, 2) := 0;
    v_today     DATE := SYSDATE;
    v_active    BOOLEAN := TRUE;
    c_tax_rate  CONSTANT NUMBER := 0.15;  -- constant

BEGIN
    -- Executable statements
    v_name := 'Joshua';
    v_salary := 5000;

    DBMS_OUTPUT.PUT_LINE('Name: ' || v_name);
    DBMS_OUTPUT.PUT_LINE('Net: ' || (v_salary * (1 - c_tax_rate)));

EXCEPTION
    -- Error handling
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('No rows returned');
    WHEN TOO_MANY_ROWS THEN
        DBMS_OUTPUT.PUT_LINE('More than one row returned');
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('Error: ' || SQLERRM);
        RAISE;  -- re-raise the exception
END;
/
```

### 2. Variables & Types

```sql
DECLARE
    -- Scalar types
    v_id        NUMBER;
    v_name      VARCHAR2(200);
    v_price     NUMBER(10, 2);
    v_date      DATE;
    v_timestamp TIMESTAMP;
    v_flag      BOOLEAN;
    v_clob      CLOB;    -- large text

    -- %TYPE — inherit column type (best practice — survives schema changes)
    v_salary    employees.salary%TYPE;
    v_dept_name departments.department_name%TYPE;

    -- %ROWTYPE — inherit entire row structure
    v_emp       employees%ROWTYPE;

    -- Record type
    TYPE t_person IS RECORD (
        name   VARCHAR2(100),
        age    NUMBER,
        email  VARCHAR2(200)
    );
    v_person t_person;

BEGIN
    -- Assign
    v_person.name := 'Joshua';
    v_person.age  := 28;

    -- SELECT INTO — assigns query result to variable (must return exactly 1 row)
    SELECT salary INTO v_salary FROM employees WHERE employee_id = 100;
    SELECT * INTO v_emp FROM employees WHERE employee_id = 100;

    DBMS_OUTPUT.PUT_LINE(v_emp.first_name || ' earns ' || v_emp.salary);
END;
/
```

### 3. Control Flow

```sql
-- IF / ELSIF / ELSE
IF v_salary > 10000 THEN
    v_grade := 'A';
ELSIF v_salary > 5000 THEN
    v_grade := 'B';
ELSE
    v_grade := 'C';
END IF;

-- CASE expression
v_grade := CASE
    WHEN v_salary > 10000 THEN 'A'
    WHEN v_salary > 5000  THEN 'B'
    ELSE 'C'
END;

-- CASE statement
CASE v_department
    WHEN 'IT'      THEN process_it(v_emp);
    WHEN 'HR'      THEN process_hr(v_emp);
    ELSE process_default(v_emp);
END CASE;

-- Loops
LOOP
    v_counter := v_counter + 1;
    EXIT WHEN v_counter >= 10;   -- or: IF condition THEN EXIT; END IF;
END LOOP;

WHILE v_counter < 10 LOOP
    v_counter := v_counter + 1;
END LOOP;

FOR i IN 1..10 LOOP              -- i is implicitly declared
    DBMS_OUTPUT.PUT_LINE(i);
END LOOP;

FOR i IN REVERSE 10..1 LOOP     -- count down
    DBMS_OUTPUT.PUT_LINE(i);
END LOOP;
```

### 4. Cursors

Cursors iterate over multi-row query results.

```sql
-- Implicit cursor — for single-row SELECT INTO
SELECT salary INTO v_sal FROM employees WHERE employee_id = 100;
-- SQL%ROWCOUNT, SQL%FOUND, SQL%NOTFOUND — attributes of last implicit cursor

-- Explicit cursor
DECLARE
    CURSOR c_employees IS
        SELECT employee_id, first_name, salary
        FROM employees
        WHERE department_id = 60
        ORDER BY salary DESC;

    v_emp c_employees%ROWTYPE;
BEGIN
    OPEN c_employees;
    LOOP
        FETCH c_employees INTO v_emp;
        EXIT WHEN c_employees%NOTFOUND;
        DBMS_OUTPUT.PUT_LINE(v_emp.first_name || ': ' || v_emp.salary);
    END LOOP;
    CLOSE c_employees;    -- always close
END;
/

-- Cursor FOR loop — cleaner, auto-opens/fetches/closes
BEGIN
    FOR rec IN (SELECT first_name, salary FROM employees WHERE department_id = 60) LOOP
        DBMS_OUTPUT.PUT_LINE(rec.first_name || ': ' || rec.salary);
    END LOOP;
END;
/

-- Parameterized cursor
CURSOR c_dept_employees(p_dept_id NUMBER) IS
    SELECT * FROM employees WHERE department_id = p_dept_id;

FOR rec IN c_dept_employees(60) LOOP
    -- process rec
END LOOP;
```

### 5. Procedures & Functions

```sql
-- Stored procedure
CREATE OR REPLACE PROCEDURE update_salary (
    p_employee_id  IN  NUMBER,
    p_percentage   IN  NUMBER,
    p_new_salary   OUT NUMBER
) AS
    v_current_salary employees.salary%TYPE;
BEGIN
    SELECT salary INTO v_current_salary
    FROM employees WHERE employee_id = p_employee_id;

    p_new_salary := v_current_salary * (1 + p_percentage / 100);

    UPDATE employees
    SET salary = p_new_salary
    WHERE employee_id = p_employee_id;

    COMMIT;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RAISE_APPLICATION_ERROR(-20001, 'Employee ' || p_employee_id || ' not found');
END update_salary;
/

-- Call procedure
DECLARE
    v_new_sal NUMBER;
BEGIN
    update_salary(100, 10, v_new_sal);
    DBMS_OUTPUT.PUT_LINE('New salary: ' || v_new_sal);
END;
/

-- Function — must return a value, can be used in SQL
CREATE OR REPLACE FUNCTION get_annual_salary (
    p_employee_id IN NUMBER
) RETURN NUMBER AS
    v_salary employees.salary%TYPE;
BEGIN
    SELECT salary * 12 INTO v_salary
    FROM employees WHERE employee_id = p_employee_id;
    RETURN v_salary;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN NULL;
END get_annual_salary;
/

-- Use in SQL
SELECT first_name, get_annual_salary(employee_id) AS annual_sal FROM employees;
```

### 6. Packages

Packages group related procedures, functions, types, and variables. Best practice for organizing PL/SQL code.

```sql
-- Package specification (public API)
CREATE OR REPLACE PACKAGE employee_pkg AS
    -- Public type
    TYPE t_emp_rec IS RECORD (id NUMBER, name VARCHAR2(100), salary NUMBER);

    -- Public constant
    c_max_salary CONSTANT NUMBER := 50000;

    -- Public procedure/function headers
    PROCEDURE hire(p_name VARCHAR2, p_salary NUMBER, p_dept_id NUMBER);
    FUNCTION  get_salary(p_employee_id NUMBER) RETURN NUMBER;
    PROCEDURE raise_salary(p_employee_id NUMBER, p_pct NUMBER);
END employee_pkg;
/

-- Package body (implementation)
CREATE OR REPLACE PACKAGE BODY employee_pkg AS

    -- Private variable (only accessible within the package)
    g_last_hire_date DATE;

    PROCEDURE hire(p_name VARCHAR2, p_salary NUMBER, p_dept_id NUMBER) AS
    BEGIN
        INSERT INTO employees (first_name, salary, department_id, hire_date)
        VALUES (p_name, p_salary, p_dept_id, SYSDATE);
        g_last_hire_date := SYSDATE;
        COMMIT;
    END hire;

    FUNCTION get_salary(p_employee_id NUMBER) RETURN NUMBER AS
        v_salary employees.salary%TYPE;
    BEGIN
        SELECT salary INTO v_salary FROM employees WHERE employee_id = p_employee_id;
        RETURN v_salary;
    EXCEPTION
        WHEN NO_DATA_FOUND THEN RETURN NULL;
    END get_salary;

    PROCEDURE raise_salary(p_employee_id NUMBER, p_pct NUMBER) AS
    BEGIN
        UPDATE employees
        SET salary = salary * (1 + p_pct/100)
        WHERE employee_id = p_employee_id;
        IF SQL%ROWCOUNT = 0 THEN
            RAISE_APPLICATION_ERROR(-20001, 'Employee not found');
        END IF;
    END raise_salary;

END employee_pkg;
/

-- Call package members
BEGIN
    employee_pkg.hire('Joshua', 8000, 60);
    employee_pkg.raise_salary(100, 5);
    DBMS_OUTPUT.PUT_LINE(employee_pkg.get_salary(100));
END;
/
```

### 7. Exception Handling

```sql
DECLARE
    -- User-defined exception
    e_negative_salary EXCEPTION;
    PRAGMA EXCEPTION_INIT(e_negative_salary, -20100);  -- bind to error number

BEGIN
    IF v_salary < 0 THEN
        RAISE_APPLICATION_ERROR(-20100, 'Salary cannot be negative');
    END IF;

EXCEPTION
    WHEN e_negative_salary THEN
        DBMS_OUTPUT.PUT_LINE('Caught: ' || SQLERRM);
    WHEN NO_DATA_FOUND THEN
        -- Handle missing row
        NULL;  -- explicit no-op
    WHEN DUP_VAL_ON_INDEX THEN
        -- Handle unique constraint violation
        ROLLBACK;
        RAISE;  -- re-raise to caller
    WHEN OTHERS THEN
        -- Catch-all — log and re-raise
        DBMS_OUTPUT.PUT_LINE('SQLCODE: ' || SQLCODE);
        DBMS_OUTPUT.PUT_LINE('SQLERRM: ' || SQLERRM);
        DBMS_OUTPUT.PUT_LINE('Backtrace: ' || DBMS_UTILITY.FORMAT_ERROR_BACKTRACE);
        ROLLBACK;
        RAISE;
END;
/
```

### 8. Triggers

```sql
-- Row-level trigger — fires for each affected row
CREATE OR REPLACE TRIGGER trg_audit_salary
    AFTER UPDATE OF salary ON employees
    FOR EACH ROW
BEGIN
    IF :NEW.salary != :OLD.salary THEN
        INSERT INTO salary_audit (employee_id, old_salary, new_salary, changed_at)
        VALUES (:OLD.employee_id, :OLD.salary, :NEW.salary, SYSDATE);
    END IF;
END;
/

-- Statement-level trigger — fires once per DML statement
CREATE OR REPLACE TRIGGER trg_restrict_weekend
    BEFORE INSERT OR UPDATE OR DELETE ON employees
BEGIN
    IF TO_CHAR(SYSDATE, 'DY') IN ('SAT', 'SUN') THEN
        RAISE_APPLICATION_ERROR(-20002, 'DML not allowed on weekends');
    END IF;
END;
/

-- Disable / enable trigger
ALTER TRIGGER trg_audit_salary DISABLE;
ALTER TRIGGER trg_audit_salary ENABLE;
```

### 9. Collections

```sql
-- Associative array (index-by table) — key-value store
DECLARE
    TYPE t_salary_map IS TABLE OF NUMBER INDEX BY VARCHAR2(50);
    v_salaries t_salary_map;
BEGIN
    v_salaries('Joshua') := 8000;
    v_salaries('Maria')  := 9500;
    DBMS_OUTPUT.PUT_LINE(v_salaries('Joshua'));  -- 8000
END;
/

-- Nested table — dynamic array, can be stored in DB
DECLARE
    TYPE t_names IS TABLE OF VARCHAR2(100);
    v_names t_names := t_names('Alice', 'Bob', 'Charlie');
BEGIN
    v_names.EXTEND;                -- add one slot
    v_names(4) := 'Diana';
    DBMS_OUTPUT.PUT_LINE('Count: ' || v_names.COUNT);
    FOR i IN v_names.FIRST..v_names.LAST LOOP
        DBMS_OUTPUT.PUT_LINE(v_names(i));
    END LOOP;
END;
/

-- BULK COLLECT — fetch multiple rows into collection (faster than row-by-row)
DECLARE
    TYPE t_ids   IS TABLE OF employees.employee_id%TYPE;
    TYPE t_names IS TABLE OF employees.first_name%TYPE;
    v_ids   t_ids;
    v_names t_names;
BEGIN
    SELECT employee_id, first_name
    BULK COLLECT INTO v_ids, v_names
    FROM employees WHERE department_id = 60;

    -- FORALL — batch DML (much faster than loop)
    FORALL i IN 1..v_ids.COUNT
        UPDATE employees SET salary = salary * 1.1 WHERE employee_id = v_ids(i);
    COMMIT;
END;
/
```

---

## Most-Used Patterns

### Dynamic SQL

```sql
-- EXECUTE IMMEDIATE for DDL and dynamic queries
EXECUTE IMMEDIATE 'CREATE TABLE temp_' || v_suffix || ' (id NUMBER)';

-- With bind variables (prevents SQL injection)
EXECUTE IMMEDIATE
    'UPDATE employees SET salary = :1 WHERE employee_id = :2'
    USING v_new_salary, v_emp_id;

-- Dynamic SELECT INTO
EXECUTE IMMEDIATE
    'SELECT ' || v_column || ' FROM employees WHERE employee_id = :1'
    INTO v_result
    USING v_emp_id;
```

### Autonomous Transaction

```sql
-- Procedure that commits independently — useful for audit logging
CREATE OR REPLACE PROCEDURE log_error (p_msg VARCHAR2) AS
    PRAGMA AUTONOMOUS_TRANSACTION;  -- independent from caller's transaction
BEGIN
    INSERT INTO error_log (message, logged_at) VALUES (p_msg, SYSDATE);
    COMMIT;  -- commits only this proc's work
END;
/
```

---

## Gotchas

- **`SELECT INTO` must return exactly one row** — zero rows raises `NO_DATA_FOUND`, multiple rows raises `TOO_MANY_ROWS`. Always handle both in `EXCEPTION`.
- **Always close explicit cursors** — or use cursor FOR loops which close automatically.
- **`COMMIT`/`ROLLBACK` in procedures** — don't commit inside reusable procedures unless they're autonomous. Let the caller control the transaction.
- **`RAISE_APPLICATION_ERROR` numbers** — must be between -20000 and -20999. Anything outside this range is an Oracle internal error code.
- **`WHEN OTHERS` without `RAISE`** — silently swallows errors. Always re-raise or log in a catch-all handler.
- **`BULK COLLECT` without `LIMIT`** — can fill memory on large tables. Use `LIMIT 1000` and loop: `FETCH c BULK COLLECT INTO v_data LIMIT 1000;`.
- **`NULL` comparisons** — `NULL = NULL` is `NULL` (not `TRUE`). Use `IS NULL` / `IS NOT NULL`.

---

## Quick Links

- [Oracle PL/SQL Docs](https://docs.oracle.com/en/database/oracle/oracle-database/21/lnpls/)
- [Oracle Live SQL](https://livesql.oracle.com) — free online Oracle environment
- [DBMS_OUTPUT](https://docs.oracle.com/en/database/oracle/oracle-database/21/arpls/DBMS_OUTPUT.html)
- [Oracle Error Codes](https://docs.oracle.com/en/database/oracle/oracle-database/21/errmg/)
