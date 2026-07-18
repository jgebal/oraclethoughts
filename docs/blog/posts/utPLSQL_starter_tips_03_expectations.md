---
title: "utPLSQL starter tips - The expectations API"
date:
  created: 2026-07-18
slug: utPLSQL_starter_tips_annotations_configure_tests
categories:
  - "PLSQL"
  - "utPLSQL"
  - "testing"
tags:
  - "utPLSQL"
  - "unit testing"
---

utPLSQL provides expectations syntax through the `ut.expect()` methods. This post covers the most commonly used expectation matchers and how failure output works.
<!-- more -->

# The utPLSQL expectations API


Expectations (assertions) in utPLSQL are written using the `ut.expect()` syntax. The actual value (value that is tested) is passed to `ut.expect()`, and a matcher method describes the expected value.

utPLSQL uses the method chaining design pattern to make an expectation read like a meaningful sentence.

```plsql
declare
    l_count integer := 3;
    l_name varchar2(10) := 'SMITH';
    l_salary number := 4567.32;
    l_result integer := 99;
    l_status varchar2(10) := 'NEW';
    l_results sys.odcinumberlist := sys.odcinumberlist(1,2,3,4);
    l_ratio  number := 3.1415926535897;
begin
    ut.expect(l_count).to_equal(5);
    ut.expect(l_name).to_equal('SMITH');
    ut.expect(l_salary).to_be_greater_than(0);
    ut.expect(l_result).to_be_not_null();
    
    -- Negation: prefix any to_ with not_to_
    ut.expect(l_status).not_to_equal('CLOSED');
    
    -- Collection and cursor size
    ut.expect(l_results).to_have_count(3);
    
    -- Approximate numeric comparison
    ut.expect(l_ratio).be_within(0.001).of_(3.14159);
end;
```

## Failure output

When an expectation fails, utPLSQL prints the expected and actual values with their data types:

```
FAILURE
  Actual: 3 (number) was expected to equal: 5 (number)
```

All expectations in a test are collected before the test is finished. A single test can have multiple expectation failures, all of which are reported as part of test result.

## Null handling

By default, utPLSQL treats null values as equal so `ut.expect(to_char(null)).to_equal(to_char(null))` passes but also the explicit `ut.expect(null).to_be_null()` passes.

This behavior can be changed by passing second argument to the `to_equal()` matcher.
Calling `ut.expect(to_char(null)).to_equal(to_char(null), false)` or more verbosely `ut.expect(to_char(null)).to_equal(to_char(null), a_nulls_are_equal => false)` will cause the expectation to fail when comparing null wit null.

## Common matchers

| Matcher                 | Tests that actual...                |
|-------------------------|-------------------------------------|
| `to_equal(x)`           | equals x                            |
| `to_be_null()`          | is null                             |
| `to_be_not_null()`      | is not null                         |
| `to_be_true()`          | is TRUE (boolean)                   |
| `to_be_false()`         | is FALSE (boolean)                  |
| `to_be_greater_than(x)` | is greater than x                   |
| `to_be_less_than(x)`    | is less than x                      |
| `to_be_between(a, b)`   | is between a and b inclusive        |
| `to_be_like(pattern)`   | matches the LIKE pattern            |
| `to_match(pattern)`     | matches the regexp pattern          |
| `to_have_count(n)`      | collection or cursorhas n elements  |
| `to_be_empty()`         | collection or cursor has zero rows  |

For more details see the full [expectations reference guide at utplsql.org](https://utplsql.org/utPLSQL/latest/userguide/expectations).

