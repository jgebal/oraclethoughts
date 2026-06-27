---
title: "utPLSQL starter tips - Annotations configure tests"
date:
  created: 2026-06-27
slug: utPLSQL_starter_tips_annotations_configure_tests
categories:
  - "PLSQL"
  - "utPLSQL"
  - "testing"
tags:
  - "utPLSQL"
  - "unit testing"
---

You configure utPLSQL test suite in special comments — called annotations — right inside the PL/SQL package specification.

Annotations are comment lines starting with `--%`. They tell the framework what is a suite, what is a test, what procedures need to be called for setup or cleanup logic. The framework reads them at runtime so no extra configuration is needed.
<!-- more -->

Here is an example of a test suite declaration containing various annotations.

```sql
create or replace package ut_orders_processing as

    --%suite(Order processing)
    --%suitepath(app.orders)
    
    --%beforeall
    procedure setup_suite;
    
    --%afterall
    procedure teardown_suite;

    --%beforeeach
    procedure reset_state;
    
    procedure seed_cancelled;

    --%test(New orders are processed)
    procedure test_new_orders_processing;

    --%test(Cancelled orders are excluded)
    --%beforetest(seed_cancelled)
    procedure test_cancelled_exclusion;
    
end;
/
```

Read more about the annotations in the [documentation](https://www.utplsql.org/utPLSQL/latest/userguide/annotations.html).

