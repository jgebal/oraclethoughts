---
title: "utPLSQL starter tips - Testing query output with cursor comparison"
date:
  created: 2026-07-28
slug: utPLSQL_starter_tips_testing_query_output
categories:
  - "PLSQL"
  - "utPLSQL"
  - "testing"
tags:
  - "utPLSQL"
  - "unit testing"
---

# Testing query output with cursor comparison

utPLSQL can compare results of two queries in single line call to `ut.expect()`. Within your test, define the SQL queries as `SYS_REFCURSOR` variables.
<!-- more -->

Open one cursor with the expected rows (usually a static fixture data), open a second cursor with the tested SQL query, and pass both to `ut.expect()`.

```sql
declare
  l_expected sys_refcursor;
  l_actual   sys_refcursor;
begin
  open l_expected for
    select 1 as id, 'Alice' as name from dual union all
    select 2 as id, 'Bob'   as name from dual;

  open l_actual for
    select id, name
      from customers
     where status = 'ACTIVE'
     order by id;

  ut.expect(l_actual).to_equal(l_expected);
end;
```

## How failure output works

When rows differ, utPLSQL reports which rows were expected but absent, which were present but unexpected, and which differed in column values.
Rows are also considered different if column order is different or if datatype of a column is different.

## Matching options

**Ordered comparison (default):** rows must appear in the same order in both cursors.

**Unordered comparison:** row order is ignored.

```sql
ut.expect(l_actual).to_equal(l_expected).unordered();
```

**Key-based matching:** when order differs between cursors, use `join_by()` to match rows on a key column rather than on row position.

```sql
ut.expect(l_actual).to_equal(l_expected).join_by('ID');
```

**Unordered columns (uc):** when columns are allowed to have different order between expected and actual dataset - column matching by name not by position in result set
```sql
ut.expect(l_actual).to_equal(l_expected).unordered_columns();
ut.expect(l_actual).to_equal(l_expected).uc();
```

**Include/exclude columns:** when the tested cursor has columns that you do not wish or cannot compare in the test
```sql
ut.expect(l_actual).to_equal(l_expected).exclude('CREATE_DATE,LAST_EDIT_DATE');
ut.expect(l_actual).to_equal(l_expected).include('NAME,DESCRIPTION');
```

## Notes on fixture design

Cursor comparison is most valuable and readable when fixture data is minimal and explicit. 
Use `UNION ALL` selects from `DUAL` for expected data rather than querying from tables that other tests might modify. 
This keeps expected values readable and the test self-contained.

## Further reading

- [Cursor comparison reference](https://www.utplsql.org/utPLSQL/latest/userguide/expectations.html#comparing-cursors-object-types-nested-tables-and-varrays)
- [Advanced cursor comparison](https://www.utplsql.org/utPLSQL/latest/userguide/advanced_data_comparison.html)
