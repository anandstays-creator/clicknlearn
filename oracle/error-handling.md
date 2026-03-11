# Error Handling in PL/SQL: A Mastery Roadmap Guide

## What is Error Handling?

**Error Handling** (also known as exception handling) is the systematic process of anticipating, detecting, and resolving runtime errors in PL/SQL programs. Its core purpose is to gracefully manage unexpected situations that occur during program execution, preventing application crashes and ensuring data integrity. Error handling solves the problem of unpredictable program behavior by providing structured mechanisms to capture, log, and respond to errors while maintaining control flow.

## How it works in PL/SQL

### Exception Block
The exception block is a dedicated section in a PL/SQL program where you define how to handle errors that occur in the execution section.

```plsql
DECLARE
  v_employee_name employees.last_name%TYPE;
BEGIN
  -- Execution section where errors might occur
  SELECT last_name INTO v_employee_name 
  FROM employees WHERE employee_id = 999; -- This might fail
  
EXCEPTION  -- Exception block starts here
  WHEN NO_DATA_FOUND THEN
    DBMS_OUTPUT.PUT_LINE('Employee not found');
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Unexpected error occurred');
END;
/
```

### Predefined Exceptions
PL/SQL provides built-in exceptions for common database errors, such as constraint violations or data not found scenarios.

```plsql
DECLARE
  v_salary employees.salary%TYPE;
BEGIN
  SELECT salary INTO v_salary 
  FROM employees WHERE employee_id = 100;
  
  -- Attempt invalid operation
  v_salary := v_salary / 0;
  
EXCEPTION
  WHEN ZERO_DIVIDE THEN  -- Predefined exception
    DBMS_OUTPUT.PUT_LINE('Division by zero attempted');
  WHEN TOO_MANY_ROWS THEN  -- Another predefined exception
    DBMS_OUTPUT.PUT_LINE('Query returned multiple rows');
END;
/
```

### User-Defined Exceptions
You can define custom exceptions to handle business logic errors specific to your application.

```plsql
DECLARE
  e_insufficient_funds EXCEPTION;  -- Declare custom exception
  PRAGMA EXCEPTION_INIT(e_insufficient_funds, -20001);  -- Associate with error code
  v_balance NUMBER := 500;
  v_withdrawal NUMBER := 1000;
BEGIN
  IF v_withdrawal > v_balance THEN
    RAISE e_insufficient_funds;  -- Raise custom exception
  END IF;
  
EXCEPTION
  WHEN e_insufficient_funds THEN
    DBMS_OUTPUT.PUT_LINE('Insufficient funds for withdrawal');
END;
/
```

### RAISE Statement
The RAISE statement explicitly triggers an exception, either predefined, user-defined, or re-raising the current exception.

```plsql
DECLARE
  e_invalid_age EXCEPTION;
  v_age NUMBER := 15;
BEGIN
  IF v_age < 18 THEN
    RAISE e_invalid_age;  -- Explicitly raise exception
  END IF;
  
EXCEPTION
  WHEN e_invalid_age THEN
    DBMS_OUTPUT.PUT_LINE('Age must be 18 or older');
    RAISE;  -- Re-raise to propagate to calling environment
END;
/
```

### RAISE_APPLICATION_ERROR
This procedure raises a user-defined error with a custom error message and error number (-20000 to -20999 range).

```plsql
CREATE OR REPLACE PROCEDURE process_order(p_order_id NUMBER) IS
  v_status orders.status%TYPE;
BEGIN
  SELECT status INTO v_status FROM orders WHERE order_id = p_order_id;
  
  IF v_status != 'PENDING' THEN
    RAISE_APPLICATION_ERROR(-20002, 
      'Order ' || p_order_id || ' is not in PENDING status');
  END IF;
  
  -- Process the order...
  
EXCEPTION
  WHEN NO_DATA_FOUND THEN
    RAISE_APPLICATION_ERROR(-20003, 'Order not found: ' || p_order_id);
END;
/
```

### SQLCODE and SQLERRM
These functions provide error code and message information for the most recent exception.

```plsql
DECLARE
  v_employee_id NUMBER := 999;
BEGIN
  DELETE FROM employees WHERE employee_id = v_employee_id;
  
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Error code: ' || SQLCODE);
    DBMS_OUTPUT.PUT_LINE('Error message: ' || SQLERRM);
    
    -- Log detailed error information
    INSERT INTO error_log (error_code, error_message, timestamp)
    VALUES (SQLCODE, SQLERRM, SYSTIMESTAMP);
END;
/
```

### Exception Propagation
Exceptions propagate through nested blocks until they find a handler or reach the calling environment.

```plsql
DECLARE
  PROCEDURE inner_procedure IS
  BEGIN
    RAISE_APPLICATION_ERROR(-20004, 'Error from inner procedure');
  END;
BEGIN
  BEGIN  -- Inner block
    inner_procedure;
  EXCEPTION
    WHEN OTHERS THEN
      DBMS_OUTPUT.PUT_LINE('Inner block caught: ' || SQLERRM);
      RAISE;  -- Propagate to outer block
  END;
  
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Outer block caught: ' || SQLERRM);
END;
/
```

### Logging Errors
Structured error logging captures exception details for debugging and auditing purposes.

```plsql
CREATE TABLE error_log (
  log_id NUMBER PRIMARY KEY,
  error_code NUMBER,
  error_message VARCHAR2(4000),
  backtrace CLOB,
  call_stack CLOB,
  timestamp TIMESTAMP,
  username VARCHAR2(30)
);

CREATE OR REPLACE PROCEDURE log_error IS
  PRAGMA AUTONOMOUS_TRANSACTION;  -- Ensures log commit even if main transaction rolls back
BEGIN
  INSERT INTO error_log (
    log_id, error_code, error_message, backtrace, 
    call_stack, timestamp, username
  ) VALUES (
    error_seq.NEXTVAL, SQLCODE, SQLERRM, 
    DBMS_UTILITY.FORMAT_ERROR_BACKTRACE(),
    DBMS_UTILITY.FORMAT_CALL_STACK(),
    SYSTIMESTAMP, USER
  );
  COMMIT;
END;
/

CREATE OR REPLACE PROCEDURE risky_operation IS
BEGIN
  -- Some operation that might fail
  RAISE PROGRAM_ERROR;
  
EXCEPTION
  WHEN OTHERS THEN
    log_error;  -- Call logging procedure
    RAISE;  -- Re-raise after logging
END;
/
```

## Why is Error Handling important?

1. **Data Integrity (ACID Compliance)**: Ensures database transactions maintain atomicity and consistency by properly handling errors that could leave data in an inconsistent state.

2. **Maintainability (DRY Principle)**: Centralized exception handling reduces code duplication and makes error management consistent across the application.

3. **User Experience (Fail-Gracefully Pattern)**: Provides meaningful error messages and alternative pathways instead of abrupt failures, enhancing application reliability.

## Advanced Nuances

### Exception Propagation in Nested Programs
When exceptions propagate through multiple nested procedures, the call stack and backtrace become crucial for debugging:

```plsql
-- Advanced propagation example with detailed logging
CREATE OR REPLACE PROCEDURE level3 IS
BEGIN
  RAISE VALUE_ERROR;
END;

CREATE OR REPLACE PROCEDURE level2 IS
BEGIN
  level3;
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Level2: ' || DBMS_UTILITY.FORMAT_ERROR_BACKTRACE());
    RAISE;
END;

CREATE OR REPLACE PROCEDURE level1 IS
BEGIN
  level2;
EXCEPTION
  WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Level1: ' || DBMS_UTILITY.FORMAT_ERROR_STACK());
    RAISE;
END;
```

### Autonomous Transaction Logging
Using autonomous transactions for error logging ensures that error details are preserved even when the main transaction rolls back:

```plsql
CREATE OR REPLACE PACKAGE error_mgmt AS
  PROCEDURE log_detailed_error(p_additional_info VARCHAR2);
END error_mgmt;

CREATE OR REPLACE PACKAGE BODY error_mgmt AS
  PROCEDURE log_detailed_error(p_additional_info VARCHAR2) IS
    PRAGMA AUTONOMOUS_TRANSACTION;
  BEGIN
    INSERT INTO error_log_detail (
      error_code, error_msg, backtrace, call_stack, additional_info
    ) VALUES (
      SQLCODE, SQLERRM, 
      DBMS_UTILITY.FORMAT_ERROR_BACKTRACE(),
      DBMS_UTILITY.FORMAT_CALL_STACK(),
      p_additional_info
    );
    COMMIT;
  END;
END error_mgmt;
```

### Bulk Operation Exception Handling
For bulk operations, Oracle provides the `SAVE EXCEPTIONS` clause and `SQL%BULK_EXCEPTIONS` collection:

```plsql
DECLARE
  TYPE id_list IS TABLE OF NUMBER;
  v_ids id_list := id_list(100, 200, 300);
  e_bulk_errors EXCEPTION;
  PRAGMA EXCEPTION_INIT(e_bulk_errors, -24381);
BEGIN
  FORALL i IN 1..v_ids.COUNT SAVE EXCEPTIONS
    UPDATE employees SET salary = salary * 1.1 
    WHERE employee_id = v_ids(i);
    
EXCEPTION
  WHEN e_bulk_errors THEN
    FOR j IN 1..SQL%BULK_EXCEPTIONS.COUNT LOOP
      DBMS_OUTPUT.PUT_LINE(
        'Error ' || j || ': Index ' || 
        SQL%BULK_EXCEPTIONS(j).ERROR_INDEX || ' - ' ||
        SQLERRM(-SQL%BULK_EXCEPTIONS(j).ERROR_CODE)
      );
    END LOOP;
END;
/
```

## How this fits the Roadmap

Within the "PL/SQL Programming" section of the Advanced PL/SQL Mastery roadmap, Error Handling serves as a **fundamental building block** that sits between basic syntax/mastery and advanced programming concepts. 

**Prerequisites**: This topic builds upon basic PL/SQL block structure, variable declaration, and control structures.

**Unlocks**: Mastery of error handling is essential for progressing to:
- Advanced package design and modular programming
- Robust transaction management patterns
- Performance optimization with bulk operations
- Sophisticated debugging and logging frameworks
- Enterprise-level application architecture

Error handling forms the foundation for writing production-grade PL/SQL code that is reliable, maintainable, and professional—qualities expected of senior PL/SQL developers.