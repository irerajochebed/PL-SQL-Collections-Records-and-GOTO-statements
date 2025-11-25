# PL-SQL-Collections-Records-and-GOTO-statements
 defining a problem that demonstrates PL/SQL Collections, Records, and GOTO statements. 

# PL/SQL Concepts Demonstration: Employee Bonus Management System
# 1. Problem Definition

Objective:
To create a PL/SQL procedure that calculates and assigns year-end bonuses to employees based on their department and performance rating. The system must demonstrate the use of:

    Records: To hold composite employee data.

    Collections: To manage and process multiple employee records efficiently.

    GOTO Statement: To implement a simple, controlled jump for handling a specific error condition (e.g., an invalid department ID).

Scenario:
The HR department needs an automated year-end bonus calculator. The bonus is calculated as a percentage of the employee's salary, determined by two factors:

    1.Department: Base bonus rate.

    2.Performance Rating: Multiplier on the base rate.

The procedure should process a list of employees, calculate their individual bonuses, and output the results. If an employee from an unrecognized department is encountered, the logic should skip the calculation and log a message using a GOTO statement


# RECORD
emp_record_type

A composite data type holding an employee's ID, name, salary, department, rating, and calculated bonus.
Groups related fields into a single unit. Makes code cleaner and more manageable than using individual variables.

# COLLECTION
emp_table_type
(Nested Table)

A list (nested table) of emp_record_type. Used with BULK COLLECT to fetch all employee data into memory at once.
Allows efficient batch processing of multiple rows. Superior to processing one row at a time in a cursor FOR loop for larger datasets.

#GOTO
GOTO skip_calculation

Used to jump out of the main calculation logic for a specific, known error condition (invalid department).
Use with extreme caution. It can create "spaghetti code" that is hard to debug and maintain. In 99% of cases, IF-THEN-ELSE or EXCEPTION handling is a better, clearer choice. Here, it's used to demonstrate its functionality in a controlled manner.

# 2. Database Setup (Assumed Table Structure)

 -- Example Employee Table Structure (Your actual table might differ)
CREATE TABLE employees (
    emp_id      NUMBER PRIMARY KEY,
    emp_name    VARCHAR2(100),
    salary      NUMBER,
    dept_id     VARCHAR2(10),
    performance_rating NUMBER -- Assume a scale, e.g., 1-5
);

-- Sample Data for Testing
INSERT INTO employees VALUES (1, 'Alice Smith', 75000, 'IT', 4);
INSERT INTO employees VALUES (2, 'Bob Jones', 65000, 'HR', 3);
INSERT INTO employees VALUES (3, 'Charlie Brown', 80000, 'IT', 5);
INSERT INTO employees VALUES (4, 'Diana Prince', 90000, 'INVALID_DEPT', 2); -- For GOTO demo
INSERT INTO employees VALUES (5, 'Edward King', 55000, 'FINANCE', 4);
COMMIT;

# 3. PL/SQL Solution Code
The following anonymous block contains the complete solution.

SET SERVEROUTPUT ON;

DECLARE
    -- STEP 1: Define a RECORD Type to hold employee data and calculated bonus.
    TYPE emp_record_type IS RECORD (
        emp_id      employees.emp_id%TYPE,
        emp_name    employees.emp_name%TYPE,
        salary      employees.salary%TYPE,
        dept_id     employees.dept_id%TYPE,
        performance_rating employees.performance_rating%TYPE,
        calculated_bonus NUMBER
    );
    
    -- STEP 2: Define a COLLECTION (Nested Table) of the record type.
    TYPE emp_table_type IS TABLE OF emp_record_type;
    
   -- Declare the collection variable.
    l_employees emp_table_type;
        
   -- Variable to store the base bonus rate.
    l_bonus_rate NUMBER;
    
   -- Exception for invalid department.
    invalid_department EXCEPTION;
    PRAGMA EXCEPTION_INIT(invalid_department, -20001);

BEGIN
    DBMS_OUTPUT.PUT_LINE('*** EMPLOYEE BONUS CALCULATION REPORT ***');
    DBMS_OUTPUT.PUT_LINE('=========================================');

    -- STEP 3: Bulk collect data from the EMPLOYEES table into the collection.
    -- This is efficient for processing multiple rows.
    
   SELECT emp_id, emp_name, salary, dept_id, performance_rating, NULL
   BULK COLLECT INTO l_employees
   FROM employees
   ORDER BY emp_id;

   -- STEP 4: Loop through the collection to calculate each employee's bonus.
    FOR i IN 1..l_employees.COUNT
    LOOP
        DBMS_OUTPUT.PUT_LINE('--- Processing: ' || l_employees(i).emp_name || ' (ID: ' || l_employees(i).emp_id || ') ---');
        
   -- Reset bonus rate for each employee.
        l_bonus_rate := 0;
        
   -- STEP 5: Use GOTO to handle an invalid department.
        -- First, check for the invalid condition.
        IF l_employees(i).dept_id = 'INVALID_DEPT' THEN
            DBMS_OUTPUT.PUT_LINE('   ERROR: Invalid department encountered.');
            -- Instead of a complex loop exit, we use GOTO to jump to a specific label for this record.
            GOTO skip_calculation;
        END IF;

   -- STEP 6: Determine the base bonus rate based on department.
        -- This demonstrates a simple lookup logic.
        CASE l_employees(i).dept_id
            WHEN 'IT' THEN l_bonus_rate := 0.10; -- 10% base bonus
            WHEN 'HR' THEN l_bonus_rate := 0.08; -- 8% base bonus
            WHEN 'FINANCE' THEN l_bonus_rate := 0.09; -- 9% base bonus
            ELSE
                -- If no other department matches, raise a custom exception.
                -- This is an alternative to the GOTO check above, shown for completeness.
                RAISE invalid_department;
        END CASE;

   -- STEP 7: Calculate the final bonus using the performance multiplier.
      l_employees(i).calculated_bonus := l_employees(i).salary *
                                         l_bonus_rate *
                                         (l_employees(i).performance_rating / 5); -- e.g., Rating 5 = 100% of base rate.

   -- Output the successful calculation.
        DBMS_OUTPUT.PUT_LINE('   Department: ' || l_employees(i).dept_id);
        DBMS_OUTPUT.PUT_LINE('   Base Rate: ' || (l_bonus_rate * 100) || '%');
        DBMS_OUTPUT.PUT_LINE('   Performance Multiplier: ' || (l_employees(i).performance_rating / 5));
        DBMS_OUTPUT.PUT_LINE('   Calculated Bonus: $' || ROUND(l_employees(i).calculated_bonus, 2));
        
   -- STEP 5 (cont.): The label we jump to if the department is invalid.
        -- This skips the calculation logic and continues with the next iteration.
        <<skip_calculation>>
        NULL; -- A NULL statement is required after a label.

   END LOOP;

   DBMS_OUTPUT.PUT_LINE('=========================================');
   DBMS_OUTPUT.PUT_LINE('*** BONUS RUN COMPLETE ***');

EXCEPTION
    -- Exception handler for the custom invalid_department exception.
    WHEN invalid_department THEN
        DBMS_OUTPUT.PUT_LINE('A serious error occurred: Unhandled invalid department.');
        DBMS_OUTPUT.PUT_LINE('SQLERRM: ' || SQLERRM);
    WHEN OTHERS THEN
        DBMS_OUTPUT.PUT_LINE('An unexpected error occurred:');
        DBMS_OUTPUT.PUT_LINE('Error Code: ' || SQLCODE);
        DBMS_OUTPUT.PUT_LINE('Error Message: ' || SQLERRM);
END;
/

# 4. Expected Output & Explanation
When executed with the provided sample data, the output will be:

*** EMPLOYEE BONUS CALCULATION REPORT ***
=========================================
--- Processing: Alice Smith (ID: 1) ---
   Department: IT
   Base Rate: 10%
   Performance Multiplier: .8
   Calculated Bonus: $6000
--- Processing: Bob Jones (ID: 2) ---
   Department: HR
   Base Rate: 8%
   Performance Multiplier: .6
   Calculated Bonus: $3120
--- Processing: Charlie Brown (ID: 3) ---
   Department: IT
   Base Rate: 10%
   Performance Multiplier: 1
   Calculated Bonus: $8000
--- Processing: Diana Prince (ID: 4) ---
   ERROR: Invalid department encountered.
--- Processing: Edward King (ID: 5) ---
   Department: FINANCE
   Base Rate: 9%
   Performance Multiplier: .8
   Calculated Bonus: $3960
=========================================
*** BONUS RUN COMPLETE ***
