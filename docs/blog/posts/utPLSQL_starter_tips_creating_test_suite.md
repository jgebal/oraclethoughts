---
title: "utPLSQL starter tips - creating test suite"
date:
  created: 2026-06-10
slug: utPLSQL_starter_tips_creating_a_test
categories:
  - "PLSQL"
  - "utPLSQL"
  - "testing"
tags:
  - "utPLSQL"
  - "unit testing"
---

Testing PL/SQL code does not need to mean writing throwaway scripts or using a complicated UI. utPLSQL gives you a proper test framework — right inside database.

A utPLSQL test is just a package annotated as `--%suite` with procedures annotated as `--%test`. The framework finds them automatically. Here is the smallest possible passing test — a single check that 1 + 1 equals 2.

```sql
create or replace package ut_math as
  -- %suite(Math utilities)

  -- %test(Addition works)
  procedure test_addition;
end;
/

create or replace package body ut_math as
  procedure test_addition is
  begin
    ut.expect(1 + 1).to_equal(2);
  end;
end;
/
```

To run th test suite call:
```sql
set serveroutput on
begin
  ut.run('ut_math');
end;
```

Install utPLSQL into a dedicated framework schema, create test package in the schema were your code lives, run it. 

For more details check the framework documentation.

#utPLSQL #OracleDev #PLSQL #TDD #DatabaseTesting