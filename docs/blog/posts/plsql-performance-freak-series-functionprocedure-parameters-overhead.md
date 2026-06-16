---
title: "PL/SQL performance freak series - function/procedure parameters overhead"
date:
  created: 2014-05-18
slug: plsql-performance-freak-series-functionprocedure-parameters-overhead
categories:
  - "PLSQL"
  - "performance"
tags:
  - "PL/SQL"
  - "Performance"
---

When writing a PL/SQL code, I usually use following strategy:

- If the code that I'm calling is returning a calculated value, it should be a function.
- If the code that I'm calling is doing some DB operations and is not returning a value, it should be a procedure.

There are however some situations, when I tend to break the rule, and this is due to performance.

<!-- more -->

In Oracle PL/SQL, following rules apply to the parameters passed and returned from functions and procedures:

- All IN parameters of procedure are passed by pointer. Variable is not copied when program block is called. The program block however is not allowed to modify the variable value.
- All OUT and IN/OUT parameters are passed by value. Variable is copied when program block is called, so the program can modify a copy of the variable. When the program block completes without error, the original variable is overwritten with the modified copy. If the program block raises exception, the original value remains unchanged.
- Values returned by functions conform to the above rule.
- The OUT and IN/OUT parameters can be passed by reference if we use the NOCOPY directive for them. This is however not available for function return value.

I was aware of the above since a very long time, but it never came into my mind to measure the actual overhead that the copy operation may introduce to your code.
This post aims to show, the overhead you may encounter, when using functions, or procedures with OUT and IN/OUT parameters.
In the following example I create 3 test program units.
A function that just returns the passed value.

```sql
CREATE OR REPLACE FUNCTION tst_func(p IN VARCHAR2) RETURN VARCHAR2 IS
BEGIN
  RETURN p;
END;
/
```

An empty procedure using the IN/OUT parameter.

```sql
CREATE OR REPLACE PROCEDURE tst_proc(p IN OUT VARCHAR2) IS
BEGIN
  NULL;
END;
/
```

An empty procedure using IN/OUT parameter with NOCOPY directive.

```sql
CREATE OR REPLACE PROCEDURE tst_proc_nocopy(p IN OUT NOCOPY VARCHAR2) IS
BEGIN
  NULL;
END;
/
```

I used a simple testing procedure to run the test on varchar parameters passed to all three program units created with above code.

```sql
CREATE OR REPLACE PROCEDURE run_test(string_len INTEGER, iter INTEGER) IS
  t NUMBER;
  v VARCHAR2(32767);
  PROCEDURE putResults(start_time NUMBER, unitName VARCHAR2) IS
  BEGIN
    DBMS_OUTPUT.PUT_LINE('Took: '||RPAD((DBMS_UTILITY.GET_TIME-start_time)/100,6) ||'sec. for '||unitName);
  END;
BEGIN
  v := RPAD( ' ', string_len);
  DBMS_OUTPUT.PUT_LINE('Running test with '||LENGTH(v)||' VARCHAR2 variable.');
  
  t := DBMS_UTILITY.GET_TIME;
  FOR i IN 1 .. iter LOOP
    v:= tst_func(v);
  END LOOP;
  putResults(t,'v:= tst_func(v);');
  
  t := DBMS_UTILITY.GET_TIME;
  FOR i IN 1 .. iter LOOP
    tst_proc(v);
  END LOOP;
  putResults(t,'tst_proc(v);');

  t := DBMS_UTILITY.GET_TIME;
  FOR i IN 1 .. iter LOOP
    tst_proc_nocopy(v);
  END LOOP;
  putResults(t,'tst_proc_nocopy(v);');
END;
/
```

Now all that is left to be done, is to perform tests with different variable size

```sql
DECLARE
  ITER CONSTANT INTEGER := 10000000;
BEGIN
  DBMS_OUTPUT.PUT_LINE('Running test with '||iter||' repetitive calls');
  run_test(32767, ITER);
  run_test(3277, ITER);
  run_test(327, ITER);
  run_test(32, ITER);
  run_test(3, ITER);
  run_test(1, ITER);
END;
/
```

and check the results

```
Running test with 10000000 repetitive calls
Running test with 32767 VARCHAR2 variable.
Took: 59.58 sec. for v:= tst_func(v);
Took: 59.36 sec. for tst_proc(v);
Took: 2.67 sec. for tst_proc_nocopy(v);
Running test with 3277 VARCHAR2 variable.
Took: 7.1 sec. for v:= tst_func(v);
Took: 9.8 sec. for tst_proc(v);
Took: 2.58 sec. for tst_proc_nocopy(v);
Running test with 327 VARCHAR2 variable.
Took: 5.1 sec. for v:= tst_func(v);
Took: 5.01 sec. for tst_proc(v);
Took: 2.61 sec. for tst_proc_nocopy(v);
Running test with 32 VARCHAR2 variable.
Took: 4.41 sec. for v:= tst_func(v);
Took: 4.33 sec. for tst_proc(v);
Took: 2.64 sec. for tst_proc_nocopy(v);
Running test with 3 VARCHAR2 variable.
Took: 4.39 sec. for v:= tst_func(v);
Took: 4.39 sec. for tst_proc(v);
Took: 2.64 sec. for tst_proc_nocopy(v);
Running test with 1 VARCHAR2 variable.
Took: 4.36 sec. for v:= tst_func(v);
Took: 4.41 sec. for tst_proc(v);
Took: 2.65 sec. for tst_proc_nocopy(v);
```

Observations

- Function calls perform similar to procedure calls with IN/OUT parameters.
- The NOCOPY directive reduces the copy overhead and causes the call to be twice as fast then the function or regular procedure.
- The bigger the variable that is passed, the bigger the boost going up to 22 times faster for single VARCHAR2 parameter.

Conclusions
If you're writing a PL/SQL code that is to be very intensively used, and is to perform really good, use procedures with NOCOPY rather than functions, even if it makes your code not so readable, as it it when a variable is assigned a value returned by function.
Of course, if the function/procedure is to be called infrequently, you may forget about the overhead and continue with the good practices for code clarity.
As usual, everything is relative.
