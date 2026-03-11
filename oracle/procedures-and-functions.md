Of course. Here is a detailed explanation of Procedures and Functions in PL/SQL, structured as you've requested.

---

### **What is Procedures and Functions?**

In PL/SQL, **Procedures** and **Functions** are named, reusable blocks of code stored within the Oracle database. They are the core building blocks for modular programming in PL/SQL and are often collectively referred to as **subprograms** or **stored procedures/functions**. Their core purpose is to promote code organization, reusability, and security. They solve the problem of repetitive, inline code by encapsulating specific business logic into discrete, callable units. This allows developers to "write once, use many times," simplifying application development and maintenance.

---

### **How it works in PL/SQL**

#### **Creating Procedures**
A procedure is created to perform an action. It does not have to return a value directly to the calling environment (though it can via `OUT` parameters). It is defined using the `CREATE OR REPLACE` syntax.

```plsql
CREATE OR REPLACE PROCEDURE greet_employee (
    p_emp_id IN employees.employee_id%TYPE
)
IS
    v_name employees.last_name%TYPE;
BEGIN
    -- Action: Select a value and perform an action (DBMS_OUTPUT)
    SELECT last_name INTO v_name
    FROM employees
    WHERE employee_id = p_emp_id;

    DBMS_OUTPUT.PUT_LINE('Hello, ' || v_name);
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('Employee not found.');
END greet_employee;
/
```

#### **Procedure Parameters**
Parameters allow you to pass data into and out of a procedure. They make procedures flexible and adaptable. There are three modes: `IN`, `OUT`, and `IN OUT`.

#### **IN Parameters**
An `IN` parameter is the default mode. It is used to pass a value **into** the procedure. The value acts as a constant within the procedure and cannot be modified.

```plsql
CREATE OR REPLACE PROCEDURE increase_salary (
    p_emp_id IN employees.employee_id%TYPE, -- IN parameter
    p_percent IN NUMBER                     -- IN parameter
)
IS
BEGIN
    -- The procedure uses the IN parameters to perform an update.
    UPDATE employees
    SET salary = salary * (1 + p_percent/100)
    WHERE employee_id = p_emp_id;

    COMMIT;
END increase_salary;
/
```

#### **OUT Parameters**
An `OUT` parameter is used to pass a value **out of** the procedure back to the calling environment (e.g., another PL/SQL block). The initial value is NULL inside the procedure.

```plsql
CREATE OR REPLACE PROCEDURE get_employee_info (
    p_emp_id  IN  employees.employee_id%TYPE,
    o_name    OUT employees.last_name%TYPE,   -- OUT parameter
    o_salary  OUT employees.salary%TYPE       -- OUT parameter
)
IS
BEGIN
    -- The procedure populates the OUT parameters.
    SELECT last_name, salary
    INTO o_name, o_salary
    FROM employees
    WHERE employee_id = p_emp_id;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        o_name := 'Unknown';
        o_salary := 0;
END get_employee_info;
/
```

#### **IN OUT Parameters**
An `IN OUT` parameter is a combination of both. A value is passed **into** the procedure, and potentially modified, and the new value is passed **out** back to the caller.

```plsql
CREATE OR REPLACE PROCEDURE format_phone_number (
    io_phone_num IN OUT VARCHAR2 -- IN OUT parameter
)
IS
BEGIN
    -- Takes a phone number in, reformats it, and passes it out.
    io_phone_num := '(' || SUBSTR(io_phone_num, 1, 3) || ') ' ||
                    SUBSTR(io_phone_num, 4, 3) || '-' ||
                    SUBSTR(io_phone_num, 7);
END format_phone_number;
/
```

#### **Creating Functions**
A function is similar to a procedure but is designed to **return a single value** directly via a `RETURN` statement. This makes them ideal for use in SQL expressions.

```plsql
CREATE OR REPLACE FUNCTION get_annual_salary (
    p_monthly_salary NUMBER
) RETURN NUMBER -- Datatype of the return value is specified here
IS
BEGIN
    -- The function computes and returns a value.
    RETURN (p_monthly_salary * 12);
END get_annual_salary;
/
```

#### **Function Return Values**
The `RETURN` statement immediately ends the function's execution and passes the specified value back to the caller. A function must contain at least one `RETURN` statement.

```plsql
CREATE OR REPLACE FUNCTION is_eligible_for_bonus (
    p_emp_id IN employees.employee_id%TYPE
) RETURN VARCHAR2 -- Returns a string ('YES' or 'NO')
IS
    v_salary employees.salary%TYPE;
    v_result VARCHAR2(3) := 'NO';
BEGIN
    SELECT salary INTO v_salary
    FROM employees WHERE employee_id = p_emp_id;

    -- Logic to determine return value
    IF v_salary > 10000 THEN
        v_result := 'YES';
    END IF;

    RETURN v_result; -- The single value is returned here
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RETURN 'NO';
END is_eligible_for_bonus;
/
-- Usage in a SELECT statement:
-- SELECT employee_id, is_eligible_for_bonus(employee_id) FROM employees;
```

#### **Autonomous Transactions**
An autonomous transaction is a independent transaction started within another main transaction. It allows you to commit or rollback operations (e.g., logging) without affecting the main transaction's commit/rollback state. This is declared with `PRAGMA AUTONOMOUS_TRANSACTION`.

```plsql
CREATE OR REPLACE PROCEDURE log_action (
    p_message IN VARCHAR2
)
IS
    PRAGMA AUTONOMOUS_TRANSACTION; -- This makes the procedure autonomous
BEGIN
    INSERT INTO audit_log (log_date, message)
    VALUES (SYSDATE, p_message);
    -- This COMMIT only affects the INSERT above, not the main transaction.
    COMMIT;
END log_action;
/
```

---

### **Why is Procedures and Functions important?**

1.  **Modularity and the DRY Principle (Don't Repeat Yourself):** They encapsulate business logic into a single, reusable unit. This eliminates code duplication, making applications easier to debug, test, and maintain.
2.  **Abstraction and Security:** Subprograms provide a controlled interface to database operations. Applications can call a procedure without needing direct table access, allowing security policies to be enforced and underlying table structures to be changed without impacting dependent code.
3.  **Performance and Scalability:** Stored subprograms are compiled and stored in the database, reducing parsing overhead. Furthermore, moving data-intensive logic to the database server minimizes network traffic in client-server architectures, leading to better scalability.

---

### **Advanced Nuances**

1.  **Function Purity and Determinism:** A function used in a SQL statement must be "pure" (no DML on the table being queried) to avoid the dreaded "mutating table" error. Declaring a function as `DETERMINISTIC` (if its output depends solely on its inputs) can also optimize performance in certain contexts, like function-based indexes.
2.  **`NOCOPY` Hint for Performance:** For large `OUT` and `IN OUT` parameters (like collections), passing by value can be expensive. The `NOCOPY` compiler hint instructs PL/SQL to pass by reference, which can drastically improve performance by avoiding the deep-copy of data, though it introduces the risk of side effects if exceptions occur.
    ```plsql
    PROCEDURE process_large_data (io_data IN OUT NOCOPY very_large_collection) ...
    ```
3.  **Calling Conventions in SQL:** While functions can be called from SQL, there are restrictions. For instance, a function with `OUT` or `IN OUT` parameters cannot be called from a SQL query. This is a key differentiator from procedures, which cannot be called directly in a SQL statement.

---

### **How this fits the Roadmap**

**Procedures and Functions** are the absolute foundation of the "PL/SQL Programming" section. Mastering them is a prerequisite for nearly every topic that follows:

*   **Prerequisite for:** This concept builds directly upon understanding of basic PL/SQL block structure, variable declaration, and control structures (loops, conditionals).
*   **Unlocks Advanced Topics:** It is the essential building block for more complex constructs like **Packages** (which group related procedures/functions), advanced API development, and **Database Triggers** (which are special types of stored procedures). Understanding parameter passing and transaction control (including autonomous transactions) is also critical for mastering more sophisticated patterns like **Application Contexts** and **Fine-Grained Access Control**. Ultimately, this mastery is necessary for designing scalable, maintainable, and secure database applications.