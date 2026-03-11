# Conditional Logic in PL/SQL

## What is Conditional Logic?

Conditional logic, also known as **control flow** or **branching logic**, refers to the programming construct that enables different code paths to execute based on specific conditions. Its core purpose is to make decisions dynamically during program execution, solving the problem of having rigid, linear code that can't adapt to varying data inputs or runtime scenarios. Conditional logic is fundamental to creating intelligent, responsive applications.

## How it works in PL/SQL

### IF THEN ELSE
The IF-THEN-ELSE statement evaluates a Boolean condition and executes different code blocks based on the result.

```sql
DECLARE
    v_salary NUMBER := 75000;
    v_grade VARCHAR2(10);
BEGIN
    -- Simple IF-THEN-ELSE structure
    IF v_salary > 80000 THEN
        v_grade := 'HIGH';
        DBMS_OUTPUT.PUT_LINE('Employee has high salary');
    ELSE
        v_grade := 'STANDARD';
        DBMS_OUTPUT.PUT_LINE('Employee has standard salary');
    END IF;
END;
/
```

### ELSIF Statement
ELSIF allows for multiple conditional branches without nesting multiple IF statements.

```sql
DECLARE
    v_score NUMBER := 85;
    v_grade VARCHAR2(2);
BEGIN
    -- Multiple conditions using ELSIF
    IF v_score >= 90 THEN
        v_grade := 'A';
    ELSIF v_score >= 80 THEN
        v_grade := 'B';
    ELSIF v_score >= 70 THEN
        v_grade := 'C';
    ELSIF v_score >= 60 THEN
        v_grade := 'D';
    ELSE
        v_grade := 'F';
    END IF;
    
    DBMS_OUTPUT.PUT_LINE('Grade: ' || v_grade);
END;
/
```

### CASE Statement
#### Simple CASE
Simple CASE compares an expression to a set of simple values.

```sql
DECLARE
    v_department VARCHAR2(20) := 'IT';
    v_bonus_multiplier NUMBER;
BEGIN
    -- Simple CASE evaluates a single expression against multiple values
    CASE v_department
        WHEN 'IT' THEN
            v_bonus_multiplier := 1.5;
        WHEN 'SALES' THEN
            v_bonus_multiplier := 2.0;
        WHEN 'HR' THEN
            v_bonus_multiplier := 1.2;
        ELSE
            v_bonus_multiplier := 1.0;
    END CASE;
    
    DBMS_OUTPUT.PUT_LINE('Bonus multiplier: ' || v_bonus_multiplier);
END;
/
```

#### Searched CASE
Searched CASE evaluates multiple Boolean conditions.

```sql
DECLARE
    v_sales NUMBER := 150000;
    v_commission_rate NUMBER;
BEGIN
    -- Searched CASE evaluates multiple Boolean conditions
    CASE
        WHEN v_sales > 200000 THEN
            v_commission_rate := 0.15;
        WHEN v_sales > 100000 THEN
            v_commission_rate := 0.10;
        WHEN v_sales > 50000 THEN
            v_commission_rate := 0.05;
        ELSE
            v_commission_rate := 0.02;
    END CASE;
    
    DBMS_OUTPUT.PUT_LINE('Commission rate: ' || v_commission_rate);
END;
/
```

### NULLIF Function
NULLIF returns NULL if two expressions are equal, otherwise returns the first expression.

```sql
DECLARE
    v_old_salary NUMBER := 50000;
    v_new_salary NUMBER := 50000;
    v_salary_change NUMBER;
BEGIN
    -- NULLIF prevents division by zero and handles equal values gracefully
    v_salary_change := (v_new_salary - v_old_salary) / NULLIF(v_old_salary, 0);
    
    IF v_salary_change IS NULL THEN
        DBMS_OUTPUT.PUT_LINE('No salary change or zero base salary');
    ELSE
        DBMS_OUTPUT.PUT_LINE('Salary change: ' || ROUND(v_salary_change * 100, 2) || '%');
    END IF;
END;
/
```

### COALESCE Function
COALESCE returns the first non-NULL value from a list of expressions.

```sql
DECLARE
    v_primary_contact VARCHAR2(50);
    v_secondary_contact VARCHAR2(50) := 'backup@company.com';
    v_fallback_contact VARCHAR2(50) := 'support@company.com';
    v_final_contact VARCHAR2(50);
BEGIN
    -- COALESCE selects first non-NULL value in sequence
    v_final_contact := COALESCE(v_primary_contact, v_secondary_contact, v_fallback_contact);
    
    DBMS_OUTPUT.PUT_LINE('Contact: ' || v_final_contact);
END;
/
```

## Why is Conditional Logic Important?

1. **DRY Principle (Don't Repeat Yourself)**: Reduces code duplication by centralizing conditional logic, making maintenance easier and preventing inconsistencies.
2. **Single Responsibility Principle (SOLID)**: Encapsulates decision-making logic within dedicated structures, improving code organization and testability.
3. **Scalability**: Enables complex business rules to be implemented efficiently, allowing applications to handle increasingly sophisticated requirements without exponential code growth.

## Advanced Nuances

### 1. Short-Circuit Evaluation
PL/SQL uses short-circuit evaluation in conditional statements, meaning it stops evaluating conditions as soon as the result is determined:

```sql
DECLARE
    v_value NUMBER := 10;
    v_result BOOLEAN;
BEGIN
    -- Second condition never evaluates due to short-circuiting
    IF v_value > 5 OR (1/0 = 1) THEN  -- No division by zero error
        DBMS_OUTPUT.PUT_LINE('Short-circuit prevents error');
    END IF;
END;
/
```

### 2. Boolean Logic in CASE Statements
CASE statements can incorporate complex Boolean expressions and even function calls:

```sql
DECLARE
    v_employee_id NUMBER := 100;
    v_performance_rating VARCHAR2(20);
BEGIN
    CASE
        WHEN v_employee_id IN (SELECT employee_id FROM high_performers) THEN
            v_performance_rating := 'EXCELLENT';
        WHEN EXISTS (SELECT 1 FROM employees WHERE employee_id = v_employee_id AND hire_date < ADD_MONTHS(SYSDATE, -24)) THEN
            v_performance_rating := 'EXPERIENCED';
        ELSE
            v_performance_rating := 'STANDARD';
    END CASE;
END;
/
```

### 3. NULL Handling Gotchas
Understanding how different conditional constructs handle NULL values is crucial:

```sql
DECLARE
    v_value NUMBER;
BEGIN
    -- These behave differently with NULL values
    IF v_value = 10 THEN  -- NULL comparison returns NULL (treated as FALSE)
        DBMS_OUTPUT.PUT_LINE('This will not execute');
    END IF;
    
    IF v_value IS NULL THEN  -- Correct NULL check
        DBMS_OUTPUT.PUT_LINE('This will execute');
    END IF;
END;
/
```

## How this Fits the Roadmap

Within the "PL/SQL Programming" section, Conditional Logic serves as a **fundamental building block** that sits between basic syntax and advanced programming constructs. It's a prerequisite for:

- **Error Handling**: Conditional logic enables sophisticated exception handling and recovery mechanisms
- **Dynamic SQL**: Essential for building flexible queries based on runtime conditions
- **Package Design**: Complex business rules within packages rely heavily on conditional constructs
- **Performance Optimization**: Enables targeted optimizations based on data conditions

Mastering conditional logic unlocks more advanced topics like **refactoring complex logic**, **dynamic program flow**, and **sophisticated business rule engines**, forming the foundation for creating robust, enterprise-grade PL/SQL applications.