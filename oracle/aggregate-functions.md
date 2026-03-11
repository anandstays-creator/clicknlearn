# Aggregate Functions in PL/SQL

## **What is Aggregate Functions?**

Aggregate functions in PL/SQL are SQL functions that operate on sets of rows to return a single summary value. They perform calculations across multiple rows grouped by one or more columns, enabling powerful data analysis capabilities directly within database operations. Unlike scalar functions that operate on single values, aggregate functions process entire result sets to provide statistical summaries, counts, and groupings.

Common aggregate functions include COUNT, SUM, AVG, MIN, and MAX, which are typically used in conjunction with GROUP BY and HAVING clauses for advanced data aggregation.

## **How it works in PL/SQL**

### COUNT Function
The COUNT function returns the number of rows that match specified criteria. It can count all rows (`COUNT(*)`), distinct values (`COUNT(DISTINCT column)`), or non-null values in a specific column (`COUNT(column)`).

```sql
DECLARE
    v_total_count NUMBER;
    v_non_null_count NUMBER;
    v_distinct_count NUMBER;
BEGIN
    -- Count all rows in employees table
    SELECT COUNT(*) INTO v_total_count FROM employees;
    
    -- Count non-null values in department_id column
    SELECT COUNT(department_id) INTO v_non_null_count FROM employees;
    
    -- Count distinct department values
    SELECT COUNT(DISTINCT department_id) INTO v_distinct_count FROM employees;
    
    DBMS_OUTPUT.PUT_LINE('Total employees: ' || v_total_count);
    DBMS_OUTPUT.PUT_LINE('Employees with departments: ' || v_non_null_count);
    DBMS_OUTPUT.PUT_LINE('Unique departments: ' || v_distinct_count);
END;
/
```

### SUM Function
The SUM function calculates the total sum of numeric values in a specified column. It ignores NULL values and returns NULL if all values are NULL or if no rows are selected.

```sql
DECLARE
    v_total_salary NUMBER;
    v_dept_salary NUMBER;
BEGIN
    -- Sum of all salaries in employees table
    SELECT SUM(salary) INTO v_total_salary FROM employees;
    
    -- Sum salaries by department using GROUP BY
    FOR dept_rec IN (
        SELECT department_id, SUM(salary) as dept_total
        FROM employees 
        WHERE department_id IS NOT NULL
        GROUP BY department_id
    ) LOOP
        DBMS_OUTPUT.PUT_LINE('Dept ' || dept_rec.department_id || 
                           ': $' || dept_rec.dept_total);
    END LOOP;
END;
/
```

### AVG Function
The AVG function calculates the average value of numeric expressions, excluding NULL values from the calculation. The result precision depends on the data type being averaged.

```sql
DECLARE
    v_avg_salary NUMBER;
    v_avg_commission NUMBER;
BEGIN
    -- Average salary across all employees
    SELECT AVG(salary) INTO v_avg_salary FROM employees;
    
    -- Average commission (NULLs are excluded from calculation)
    SELECT AVG(NVL(commission_pct, 0)) INTO v_avg_commission FROM employees;
    
    -- Average salary by job title with GROUP BY
    FOR job_rec IN (
        SELECT job_id, AVG(salary) as avg_salary, COUNT(*) as emp_count
        FROM employees 
        GROUP BY job_id
        HAVING COUNT(*) > 1  -- Only jobs with multiple employees
    ) LOOP
        DBMS_OUTPUT.PUT_LINE('Job ' || job_rec.job_id || 
                           ': Avg Salary $' || ROUND(job_rec.avg_salary, 2) ||
                           ' (' || job_rec.emp_count || ' employees)');
    END LOOP;
END;
/
```

### MIN Function
The MIN function returns the minimum value from a set of values. It works with numeric, date, and character data types, and ignores NULL values.

```sql
DECLARE
    v_min_salary NUMBER;
    v_min_hire_date DATE;
    v_min_job_title VARCHAR2(50);
BEGIN
    -- Minimum salary in the company
    SELECT MIN(salary) INTO v_min_salary FROM employees;
    
    -- Earliest hire date
    SELECT MIN(hire_date) INTO v_min_hire_date FROM employees;
    
    -- Minimum job title alphabetically by department
    FOR dept_rec IN (
        SELECT department_id, MIN(job_id) as min_job
        FROM employees 
        WHERE department_id IS NOT NULL
        GROUP BY department_id
    ) LOOP
        DBMS_OUTPUT.PUT_LINE('Dept ' || dept_rec.department_id || 
                           ': First job alphabetically: ' || dept_rec.min_job);
    END LOOP;
END;
/
```

### MAX Function
The MAX function returns the maximum value from a set of values. Like MIN, it handles various data types and excludes NULL values from consideration.

```sql
DECLARE
    v_max_salary NUMBER;
    v_max_commission NUMBER;
BEGIN
    -- Maximum salary
    SELECT MAX(salary) INTO v_max_salary FROM employees;
    
    -- Maximum commission percentage by department
    FOR dept_rec IN (
        SELECT department_id, 
               MAX(commission_pct) as max_commission,
               COUNT(*) as total_employees,
               COUNT(commission_pct) as commissioned_employees
        FROM employees 
        WHERE department_id IS NOT NULL
        GROUP BY department_id
        HAVING MAX(commission_pct) IS NOT NULL
    ) LOOP
        DBMS_OUTPUT.PUT_LINE('Dept ' || dept_rec.department_id || 
                           ': Max commission ' || dept_rec.max_commission ||
                           ' (' || dept_rec.commissioned_employees || 
                           '/' || dept_rec.total_employees || ' employees)');
    END LOOP;
END;
/
```

### GROUP BY Clause
The GROUP BY clause groups rows that have the same values in specified columns into summary rows. It's essential for combining aggregate functions with categorical data analysis.

```sql
DECLARE
    CURSOR dept_summary_cur IS
        SELECT department_id, 
               COUNT(*) as emp_count,
               AVG(salary) as avg_salary,
               MIN(hire_date) as earliest_hire,
               MAX(salary) as max_salary
        FROM employees
        WHERE department_id IS NOT NULL
        GROUP BY department_id
        ORDER BY department_id;
BEGIN
    FOR dept_rec IN dept_summary_cur LOOP
        DBMS_OUTPUT.PUT_LINE(
            'Department ' || dept_rec.department_id || 
            ': ' || dept_rec.emp_count || ' employees | ' ||
            'Avg Salary: $' || ROUND(dept_rec.avg_salary, 2) ||
            ' | Max Salary: $' || dept_rec.max_salary ||
            ' | First Hire: ' || TO_CHAR(dept_rec.earliest_hire, 'DD-MON-YYYY')
        );
    END LOOP;
END;
/
```

### HAVING Clause
The HAVING clause filters groups created by the GROUP BY clause based on aggregate conditions. Unlike WHERE which filters rows before grouping, HAVING filters after aggregation.

```sql
DECLARE
    CURSOR high_volume_jobs_cur IS
        SELECT j.job_title,
               COUNT(e.employee_id) as employee_count,
               AVG(e.salary) as avg_salary,
               SUM(e.salary) as total_salary_budget
        FROM employees e
        JOIN jobs j ON e.job_id = j.job_id
        GROUP BY j.job_title
        HAVING COUNT(e.employee_id) > 5  -- Only jobs with more than 5 employees
           AND AVG(e.salary) > 5000      -- And average salary > 5000
        ORDER BY total_salary_budget DESC;
BEGIN
    DBMS_OUTPUT.PUT_LINE('High Volume/High Salary Jobs:');
    DBMS_OUTPUT.PUT_LINE('================================');
    
    FOR job_rec IN high_volume_jobs_cur LOOP
        DBMS_OUTPUT.PUT_LINE(
            RPAD(job_rec.job_title, 30) ||
            'Employees: ' || LPAD(job_rec.employee_count, 3) ||
            ' | Avg Salary: $' || LPAD(ROUND(job_rec.avg_salary), 6) ||
            ' | Total Budget: $' || LPAD(ROUND(job_rec.total_salary_budget), 10)
        );
    END LOOP;
END;
/
```

### GROUPING SETS
GROUPING SETS allow multiple grouping combinations in a single query, providing a more efficient alternative to multiple UNION ALL queries. They enable hierarchical and multi-dimensional analysis.

```sql
DECLARE
    CURSOR multi_dimension_cur IS
        SELECT 
            department_id,
            job_id,
            COUNT(*) as emp_count,
            AVG(salary) as avg_salary,
            -- GROUPING function identifies the aggregation level
            GROUPING(department_id) as dept_grouped,
            GROUPING(job_id) as job_grouped
        FROM employees
        WHERE department_id IS NOT NULL
        GROUP BY GROUPING SETS (
            (department_id, job_id),  -- Detail by dept and job
            (department_id),           -- Summary by department only
            (job_id),                  -- Summary by job only
            ()                         -- Grand total
        )
        ORDER BY GROUPING(department_id), department_id, 
                 GROUPING(job_id), job_id;
BEGIN
    FOR agg_rec IN multi_dimension_cur LOOP
        IF agg_rec.dept_grouped = 1 AND agg_rec.job_grouped = 1 THEN
            DBMS_OUTPUT.PUT_LINE('GRAND TOTAL: ' || agg_rec.emp_count || 
                               ' employees, Avg Salary: $' || ROUND(agg_rec.avg_salary, 2));
        ELSIF agg_rec.dept_grouped = 1 THEN
            DBMS_OUTPUT.PUT_LINE('JOB SUMMARY (' || agg_rec.job_id || '): ' || 
                               agg_rec.emp_count || ' employees');
        ELSIF agg_rec.job_grouped = 1 THEN
            DBMS_OUTPUT.PUT_LINE('DEPT SUMMARY (' || agg_rec.department_id || '): ' || 
                               agg_rec.emp_count || ' employees');
        ELSE
            DBMS_OUTPUT.PUT_LINE('DEPT ' || agg_rec.department_id || 
                               ' JOB ' || agg_rec.job_id || ': ' || 
                               agg_rec.emp_count || ' employees');
        END IF;
    END LOOP;
END;
/
```

## **Why is Aggregate Functions important?**

1. **Performance Optimization (Single-Query Principle)**: Aggregate functions enable complex data summarization in a single database round-trip, eliminating the need for multiple individual queries and reducing network overhead.

2. **Data Integrity (ACID Compliance)**: By performing calculations directly in the database, aggregates maintain transactional consistency and avoid phantom read issues that can occur with application-level calculations.

3. **Scalability (Divide and Conquer Pattern)**: GROUP BY with HAVING clauses allows partitioning large datasets into manageable groups, enabling efficient processing of massive volumes of data through parallel execution.

## **Advanced Nuances**

**NULL Handling Variations**: Different aggregate functions handle NULLs differently. COUNT(*) includes NULLs, COUNT(column) excludes them, while AVG and SUM ignore NULLs entirely. Understanding these nuances is critical for accurate reporting.

```sql
-- Demonstration of NULL handling in aggregates
SELECT 
    COUNT(*) as total_rows,                    -- Includes NULLs: 100
    COUNT(commission_pct) as non_null_comms,   -- Excludes NULLs: 35
    AVG(commission_pct) as avg_non_null,      -- Avg of non-NULLs: 0.15
    AVG(NVL(commission_pct, 0)) as avg_all    -- Avg including NULLs as 0: 0.0525
FROM employees;
```

**Window Function Integration**: Advanced usage involves combining aggregates with window functions for running totals and moving averages without collapsing rows:

```sql
-- Using aggregates as window functions
SELECT employee_id, department_id, salary,
       AVG(salary) OVER (PARTITION BY department_id) as dept_avg_salary,
       SUM(salary) OVER (PARTITION BY department_id) as dept_total_salary,
       salary - AVG(salary) OVER (PARTITION BY department_id) as diff_from_avg
FROM employees;
```

## **How this fits the Roadmap**

Within the "Advanced SQL Features" section, aggregate functions serve as the foundation for sophisticated data analysis capabilities. They are prerequisites for understanding **analytic functions** (LEAD, LAG, RANK), **window functions** for running calculations, and **pivoting operations** for cross-tabular reporting.

Mastery of aggregate functions unlocks advanced topics like **materialized views** with aggregate summaries, **query rewrite** capabilities for optimization, and **OLAP operations** for business intelligence applications. This knowledge directly supports subsequent roadmap sections on performance tuning and data warehousing techniques.