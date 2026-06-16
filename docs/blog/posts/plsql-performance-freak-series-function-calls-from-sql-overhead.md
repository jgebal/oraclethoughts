---
title: "PL/SQL performance freak series - function calls from SQL overhead"
date:
  created: 2014-05-25
slug: plsql-performance-freak-series-function-calls-from-sql-overhead
categories:
  - "SQL"
  - "performance"
tags:
  - "PL/SQL"
  - "Performance"
---

In my [previous post](../posts/plsql-performance-freak-series-functionprocedure-parameters-overhead.md "PL/SQL performance freak series – function/procedure parameters overhead") I've shown and measured the performance loss on passing the parameters to and from procedure/function call inside PL/SQL code.
In this article I'm about to reveal another bottleneck that is often forgotten and not so easy to overcome.

<!-- more -->

Suppose you need to have some value calculated. The formula is straight matematical calculation but the calculation will be used in multiple SQL statements across system and is therefore a perfect candidate for a PL/SQL function.
I will use this sample formula:
For every Odd value return 6, for every Even value return 3.

```sql
case mod( X , 2 ) when 1 then 6 else 3 end
```

So the PL/SQL implementing the formula is:

```sql
CREATE OR REPLACE FUNCTION get_val( val IN INTEGER ) RETURN INTEGER IS
BEGIN
  RETURN CASE mod( val  , 2 ) WHEN 1 THEN 6 ELSE 3 END;
END;
/
```

One may either write the formula directly in each SQL statement, that needs it, or use the implemented function to keep the code [DRY](http://en.wikipedia.org/wiki/Don't_repeat_yourself).
Let us compare the performance of the inline version of the formula in SQL statement vs. function call.
Simple script will do the trick.

```sql
set timing on
PROMPT Calculate using function call from SQL
SELECT sum(get_val(ROWNUM)) val FROM dual CONNECT BY LEVEL < 1200000;

PROMPT Calculate using inline calculation in SQL
SELECT sum( CASE mod( ROWNUM , 2 ) WHEN 1 THEN 6 ELSE 3 END ) 
  FROM dual CONNECT BY LEVEL < 1200000;
```

And the results are:

```
Calculate using function call from SQL
5399997 

Elapsed: 00:00:15.562
```

```
Calculate using inline calculation in SQL
5399997

Elapsed: 00:00:01.212
```

The SELECT statement with function call was **over 10 times slower** than the inline calculation (when running  on 1.2 million rows).
As [Tom Kyte explained in his answer](https://asktom.oracle.com/pls/apex/f?p=100:11:0::::P11_QUESTION_ID:60122715103602), SQL and PL/SQL are two separate languages with two separate runtime engines and each function call causes a context switch.
How would the query perform, if we would have 2 function calls in it?

```sql
set timing on
PROMPT Calculate using function call from SQL
SELECT sum(get_val(ROWNUM)) val, sum(get_val(-ROWNUM)) val_1 FROM dual CONNECT BY LEVEL < 1200000;

PROMPT Calculate using inline calculation in SQL
SELECT sum( CASE mod( ROWNUM , 2 ) WHEN 1 THEN 6 ELSE 3 END ) val
       sum( CASE mod( -ROWNUM , 2 ) WHEN 1 THEN 6 ELSE 3 END ) val_1
  FROM dual CONNECT BY LEVEL < 1200000;
```

See the results

```
Calculate using function call from SQL
5399997 3599997 

Elapsed: 00:00:29.260
```

```
Calculate using inline calculation in SQL
5399997 5399997

Elapsed: 00:00:01.210
```

As expected, with two function calls from SQL statements, the overhead was doubled. Now the code with function calls runs almost 30 times slower than the inline version.
Now that the issue was clearly shown, some questions should be answered.
- Is it always an issue?
- What can be done to maintain the function encapsulation (have the code DRY) and keep high performance?
I will give some answers to that on my next post.
