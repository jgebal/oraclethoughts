---
title: "What is the equivalent of utassert eqtable / eqquery in utPLSQL v3"
date:
  created: 2017-10-30
slug: what-is-the-equivalent-of-utassert-eqtable-utplsql-v2-v3
categories:
  - "utPLSQL"
  - "testing"
tags:
  - "Oracle"
  - "PL/SQL"
  - "utPLSQL"
  - "unit testing"
---

In utPLSQL v2 you had to use 'quoted text to compare tables / queries.
utPLSQL v3 allows you to compare table data using native refcursors without usage of dynamic SQL.

<!-- more -->

In v2 with you would use syntax

```sql
procedure test_tables_v2 is
begin
  utassert.eqtable(
    'Delete rows',
    'EMPLOYEES',
    'ut_DEL1',
    'hire_date > date ''2017-08-01''','hire_date > date ''2017-08-01'''
  );
end;
/
```

or syntax:

```sql
procedure test_tables_v2 is
begin
   utassert.eqquery (
      'Update three columns',
      'select first_name, commission, hire_date from EMPLOYEE',
      'select first_name, commission, hire_date from ut_upd1'
   );
end;
/
```

In v3 you will use syntax:

```sql
procedure test_tables_v3 is
  l_expected sys_refcursor;
  l_actual sys_refcursor;
begin
  open l_actual for select * from employees where hire_date > date '2017-08-01' order by employee_id;
  open l_expected for select * from ut_del1 where hire_date > date '2017-08-01' order by employee_id;
  ut.expect(l_actual).to_equal(l_expected);
end;
```

Additionally v3 allows you to filter columns of cursors so you can decide which columns should not be compared (in this example we skip the LAST\_NAME column)

```sql
procedure test_tables_v3 is
  l_expected sys_refcursor;
  l_actual sys_refcursor;
begin
  open l_actual for select first_name, last_name, commission, hire_date from employees order by first_name;
  open l_expected for select first_name, commission, hire_date from employees order by first_name;
  ut.expect(l_actual).to_equal(l_expected, a_exclude=>'LAST_NAME' );
end;
```

`ut.expect` syntax, uses native Oracle cursors rather than dynamic SQL.
Thanks to that it's much easier to open a cursor that requires complex setup or an object to be passed as variable to the cursor.
The `ut.expect( cursor ).to_equal( cursor )` accepts and compares cursors with columns containing Clob, Blob as well as nested cursors, Oracle Object Types and Collections of Objects.
Additionally, you can exclude columns (or elements of complex types in cursor columns) that you do not wish to compare.
One thing to keep in mind, is that in utPLSQL v3, order matters. Cursor data is compared in a sorted manner.
This makes the comparison more strict, as you can check if data is properly ordered in the result-set.
It becomes useful when validating ordering of cursor data returned by procedure or function.
 
utPSLQL v2 documentaion on [utassert](http://utplsql.org/utPLSQL/v2.3.1/utassert.html#utassert.eqtable)
utPLSQL v3.0.3 documentation on [expectations](http://utplsql.org/utPLSQL/v3.0.3/userguide/expectations.html)
