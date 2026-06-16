---
title: "PL/SQL performance freak series – alternatives to PL/SQL function calls from SQL"
date:
  created: 2014-06-03
slug: plsql-performance-freak-series-alternatives-to-plsql-function-calls-from-sql
categories:
  - "PLSQL"
  - "SQL"
  - "performance"
tags:
  - "PL/SQL"
  - "Performance"
---

Last week I revealed the numbers standing behind the overhead of calling a Pl/SQL function from within an SQL statement.
I've left two questions open:
- Is it always a performance issue, when you call a PL/SQL function from a SQL statement?
- What can be done to maintain the function encapsulation (have the code DRY) and keep high performance?

<!-- more -->

Lets take it one by one.
To see if it is always an issue, lets run some tests. We will be running the SQL statements with inlined calculation formula and PL/SQL function call using different SQL result sizes and different number of executions.
For simplicity and clarity. The function i used does not use DETERMINISTIC nor RESULT\_CACHE keyword. In the examples, i assume that you run the function on data unique data, so that each function call is unique.
I've made those assumptions only to clearly isolate the issue with PL/SQL function call overhead. If the function calls are repeatable, you could benefit from the DETERMINISTIC or RESULT\_CACHE, but this is out of scope of the article, so I will not investigate it further now.
In my previous post I compared the simple SQL statement performance with or without PL/SQL function call. The comparison was done on 1,2 million rows. how does it look when we process less rows with our queries? Is it still a pain? Do we need to worry? How much of a pain is it? Let's check.
I will use the function implementation as before.

```sql
CREATE OR REPLACE FUNCTION get_val( val IN INTEGER ) RETURN INTEGER IS
BEGIN
  RETURN CASE mod( val  , 2 ) WHEN 1 THEN 6 ELSE 3 END;
END;
/
```

The following script should give us some answers

```sql
set timing on
set feedback off
set pagesize 0

PROMPT Calculate using function call from SQL on 100 rows data sets 10000 times
DECLARE
  res INTEGER := 0;
BEGIN
  FOR r IN 1 .. 10000 LOOP
    SELECT sum(get_val(ROWNUM)) val
      INTO res
      FROM dual CONNECT BY LEVEL <= 100;
  END LOOP;
dbms_output.put_line(res);
END;
/

PROMPT Calculate using inline calculation in SQL on 100 rows data sets 10000 times
DECLARE
  res INTEGER := 0;
BEGIN
  FOR r IN 1 .. 10000 LOOP
  SELECT sum( CASE mod( ROWNUM , 2 ) WHEN 1 THEN 6 ELSE 3 END ) val
    INTO res
    FROM dual CONNECT BY LEVEL <= 100;
  END LOOP;
dbms_output.put_line(res);
END;
/

PROMPT Calculate using function call from SQL on 1 row data sets 100000 times
DECLARE
  res INTEGER := 0;
BEGIN
  FOR r IN 1 .. 100000 LOOP
  SELECT sum(get_val(ROWNUM)) val
    INTO res
    FROM dual;
  END LOOP;
dbms_output.put_line(res);
END;
/

PROMPT Calculate using inline calculation in SQL on 1 row data sets 100000 times
DECLARE
  res INTEGER := 0;
BEGIN
  FOR r IN 1 .. 100000 LOOP
  SELECT sum( CASE mod( ROWNUM , 2 ) WHEN 1 THEN 6 ELSE 3 END ) val
    INTO res
    FROM dual;
  END LOOP;
dbms_output.put_line(res);
END;
/
```

And the results give us some answers

```
Calculate using function call from SQL on 100 rows data sets 10000 times
Elapsed: 00:00:10.523
Calculate using inline calculation in SQL on 100 rows data sets 10000 times
Elapsed: 00:00:01.572
Calculate using function call from SQL on 1 row data sets 100000 times
Elapsed: 00:00:05.891
Calculate using inline calculation in SQL on 1 row data sets 100000 times
Elapsed: 00:00:03.841
```

The overhead is there, but it is less visible, when the data volumes that the SQL query processes are small.
I leave it up to you to decide if this is a reason good enough, to use PL/SQL functions in SQL statements.
In my daily work I use the following pattern.
If it is small and is not called frequently, I don't hesitate to use all the goodies of PL/SQL in SQL.
If it is big or called really often, I avoid this approach as much as possible.
Now, lets go to the second question.
What can be done to maintain the function encapsulation (have the code DRY) and keep high performance?
There are several ways to keep the performance:
- Keep everything inlined in SQL queries
- Encapsulate the calculation with Views
- Encapsulate the calculation using Virtual Column (Oracle 11g and above)
- Use PL/SQL to do the calculation
- Use Dynamic SQL and and PL/SQL functions to build the calculation formula (not to calculate the actual value)
Inlining the calculation can be fine, if you use it only in one SQL statement, if it is 2 or more, and the formula is actually complex, it would be better to encapsulate it somehow.
You could use View over a table to create a facade over the calculation logic that is done purely in SQL.
This approach is OK, as long as the formula is used on one table. If it is two or more table, you end up having the same formula scattered across many views which could turn into maintenance hell if the formula needs to be changed.
The script below will test the performance of the view vs the performance of inline calculation.

```sql
set feedback off
set pagesize 0
CREATE TABLE t1 AS SELECT rownum AS rn
  FROM dual CONNECT BY LEVEL < 1200000;

CREATE OR REPLACE VIEW v1 AS 
  SELECT CASE mod( ROWNUM , 2 ) WHEN 1 THEN 6 ELSE 3 END val FROM t1;

set timing on
PROMPT Calculate using inline calculation in SQL
SELECT sum( CASE mod( ROWNUM , 2 ) WHEN 1 THEN 6 ELSE 3 END ) FROM t1;

PROMPT Calculate using calculation in View
SELECT sum(val) from v1;

SET TIMING OFF
DROP TABLE t1;
DROP VIEW v1;
```

The performance of the view is nearly as good as the performance of the plain SQL query. The view was slightly slower. I have repeated the test couple of times with similar results.

```
Calculate using inline calculation in SQL
 5399997 
Elapsed: 00:00:00.660

Calculate using calculation in View
 5399997 
Elapsed: 00:00:00.731
```

You could create a table with virtual column to achieve the same goal as with the view. [This feature is available since Oracle 11g](http://www.oracle-base.com/articles/11g/virtual-columns-11gr1.php). Same as with view, if the formula is used on many tables, you will need to have it defined in many places and you may end up having maintenance issues.

```sql
set feedback off
set pagesize 0
CREATE TABLE t1 AS
  SELECT rownum AS rn
    FROM dual CONNECT BY LEVEL < 1200000;

CREATE TABLE t2(rn NUMBER, val NUMBER GENERATED ALWAYS AS (CASE mod( rn , 2 ) WHEN 1 THEN 6 ELSE 3 END) VIRTUAL);

INSERT INTO t2 (rn) SELECT rn from t1;
COMMIT;

set timing on
 
PROMPT Calculate using inline calculation in SQL
SELECT sum( CASE mod( ROWNUM , 2 ) WHEN 1 THEN 6 ELSE 3 END )
  FROM t1;

PROMPT Calculate using calculation ON virtual column
SELECT sum( val )
  FROM t2;

SET TIMING OFF
DROP TABLE t1;
DROP TABLE t2;
```

Interesting thing with virtual column, is that the calculation was actually executed faster than with a regular query.
I've tested it a couple of times and the results were always in favour of calculation on virtual column.

```
Calculate using inline calculation in SQL
 5399997 

Elapsed: 00:00:00.687
Calculate using calculation ON virtual column
 5399997 

Elapsed: 00:00:00.444
```

The solutions that allow you to have the business logic encapsulated in single place and avoid the performance trade-off are:
- Use PL/SQL to do the work and bulk collect the data.
- Use Dynamic SQL and function

```sql
set feedback off
set serveroutput on
set pagesize 0
CREATE TABLE t1 AS SELECT rownum AS rn
  FROM dual CONNECT BY LEVEL <= 1200000;

CREATE OR REPLACE FUNCTION get_val(val IN INTEGER) RETURN INTEGER IS
BEGIN
  RETURN CASE mod( val , 2 ) WHEN 1 THEN 6 ELSE 3 END;
END;
/

CREATE OR REPLACE FUNCTION get_val_str(col_nm VARCHAR2) RETURN VARCHAR2 IS
BEGIN
  RETURN 'case mod('||col_nm||',2) when 1 then 6 else 3 end';
END;
/

set timing on

PROMPT Calculate using function call in SQL
SELECT sum( get_val(rn) ) val FROM t1;

PROMPT Calculate using PL/SQL with bulk collect
DECLARE
  TYPE t_num_tab IS TABLE OF INTEGER;
  val_lst t_num_tab;
  res INTEGER := 0;
BEGIN
  SELECT rn BULK COLLECT INTO val_lst FROM t1;
  FOR i IN val_lst.FIRST .. val_lst.LAST loop
    res := res + get_val(val_lst(i));
  END loop;
  dbms_output.put_line(res);
END;
/

PROMPT Calculate using dynamic SQL with with function call for inlining calculation
DECLARE
  res INTEGER;
BEGIN
  EXECUTE IMMEDIATE
    'SELECT sum('||get_val_str('rn')||') val FROM t1'
    INTO res;
  dbms_output.put_line(res);
END;
/

PROMPT Calculate using inline calculation in SQL
SELECT sum( CASE mod( rn , 2 ) WHEN 1 THEN 6 ELSE 3 END ) val
  FROM t1;
  
set timing off
DROP FUNCTION get_val;
DROP TABLE t1;
```

```
Calculate using function call in SQL
5400000 

Elapsed: 00:00:13.769
Calculate using PL/SQL with bulk collect
Elapsed: 00:00:01.759
5400000

Calculate using dynamic SQL with with function call for inlining calculation
Elapsed: 00:00:00.447
5400000

Calculate using inline calculation in SQL
5400000 

Elapsed: 00:00:00.442
```

The approach witch Native Dynamic SQL has no visible performance impact.
The PL/SQL processing with bulk collect has a noticeable overhead. It is 4 times slower than the pure SQL/native dynamic SQL.
It is however still 8 times faster than the PL/SQL function call from within SQL.
There are many options to further investigate different variants and aspects of each approach.
I'll leave it up to you to decide which one is your favourite and suits your needs best.
