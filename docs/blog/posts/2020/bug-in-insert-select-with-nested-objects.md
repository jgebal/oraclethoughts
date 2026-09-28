---
title: "BUG in INSERT ... SELECT with nested objects"
date:
  created: 2020-04-05
slug: bug-in-insert-select-with-nested-objects
categories:
  - "SQL"
tags:
  - "SQL"
  - "exception handling"
---

I came across a very nasty bug in Oracle SQL engine while working on some new code.
It took me a while to figure out what the problem is as the SQL code I was working on was acting weird.
The work was related to refactoring and consolidating an existing code. As part of the effort we aimed to achieve two things. First, simplify the code and second, improve performance.
The code was already there and it was very well tested with unit tests using utPLSQL.
Test cases covered all the join conditions and all column transformations done as part of delivered functionality.
Thanks to the testing, we've managed to capture the issue while developing new code. It took me quite some time and head scratching to figure that it's actually an oracle bug.
Creating an isolated test-case to reproduce the unexpected behavior really helped. With that I could confirm with 100% certainty that it's not coding issue but an actual BUG.
I must mention that within my ~20 years of career as an SQL and PL/SQL developer I've never seen a bug like that.

<!-- more -->

As a result of that bug, query data is getting messed up when inserting into a table.
The bug occurs only in very specific conditions and below examples illustrate this.
Followup examples demonstrate workarounds for the bug.
Regardless of workarounds available, the bug is really serious as it doesn't raise any exceptions, simply bad data gets inserted into tables.

## Test case setup

We will need to setup two user defined types and a nested table.

```sql
create or replace type nested_item force as object (
  some_id integer,
  unit_cost number
);
/

create or replace type item force as object (
  id integer,
  sub_item nested_item
);
/

create or replace type items force as table of item;
/
```

With the above setup we have a nested table `items` containing a list of `item` objects.
Each `item` object contains `id` and `sub_item` of type `nested_item`.
To make examples easier to read, we will create a function that generates some data and returns them as `items` nested table type.

```sql
create or replace function get_items(
  p_how_many positiven := 10
) return items is
  l_items items := items( );
begin
  for i in 1 .. p_how_many loop
    l_items.extend;
    l_items( l_items.last ) :=
      item(
        id       => i,
        sub_item =>
          nested_item(
            some_id   => mod( i - 1, 5 ) + 1,
            unit_cost => i / 10
          )
      );
  end loop;
  return l_items;
end;
/
```

## Test cases

### GROUP BY nested object attribute

With function created, we can now run a query on the nested table returned by function.
**Note.**
The SELECT statement does a `GROUP BY OBJS.SUB_ITEM.SOME_ID`. So we are grouping by attribute of a nested object.

```sql
select
    objs.id,
    objs.sub_item.some_id,
    sum( objs.sub_item.unit_cost )
  from table (get_items( 10 ) ) objs
 group by
    objs.id,
    objs.sub_item.some_id
 order by
    objs.id,
    objs.sub_item.some_id;
```

The above query gives correct results.

| ID  | SUB\_ITEM.SOME\_ID | SUM(OBJS.SUB\_ITEM.UNIT\_COST) |
|-----|--------------------|--------------------------------|
| 1   | 1                  | 0.1                            |
| 2   | 2                  | 0.2                            |
| 3   | 3                  | 0.3                            |
| 4   | 4                  | 0.4                            |
| 5   | 5                  | 0.5                            |
| 6   | 1                  | 0.6                            |
| 7   | 2                  | 0.7                            |
| 8   | 3                  | 0.8                            |
| 9   | 4                  | 0.9                            |
| 10  | 5                  | 1                              |

Let us populate a table using the above query. To do that, we will first create the TARGET table.

```sql
create table target (
  test_case_number integer,
  item_id          integer,
  some_id          integer,
  some_value       number
);
/
```

We can now perform an INSERT statement and populate table with data.

```sql
insert into target (
  test_case_number,
  item_id,
  some_id,
  some_value
)
select
    1,
    objs.id,
    objs.sub_item.some_id,
    sum( objs.sub_item.unit_cost )
  from table (get_items( 10 ) ) objs
 group by
    objs.id,
    objs.sub_item.some_id
 order by
    objs.id,
    objs.sub_item.some_id;

commit;
```

TARGET table, when inspected shows correct results (the same as the base query)

```sql
select
    item_id,
    some_id,
    some_value
  from target
 where test_case_number = 1
 order by
    item_id,
    some_id;
```

| ITEM\_ID | SOME\_ID | SOME\_VALUE |
|----------|----------|-------------|
| 1        | 1        | 0.1         |
| 2        | 2        | 0.2         |
| 3        | 3        | 0.3         |
| 4        | 4        | 0.4         |
| 5        | 5        | 0.5         |
| 6        | 1        | 0.6         |
| 7        | 2        | 0.7         |
| 8        | 3        | 0.8         |
| 9        | 4        | 0.9         |
| 10       | 5        | 1           |

**Conclusion**
INSERT as SELECT from a nested table of objects with nested object attribute used in GROUP BY expression works correctly.

### GROUP BY nested object attribute and join with a table

Let us add a join to a lookup table and see the results.
To do that we will use a `LOOKUP` table populated with 5 rows.

```sql
create table lookup (
some_id integer,
multiplier number,
constraint lookup_pk primary key ( some_id )
);
/

insert into lookup (
some_id, multiplier
)
select rownum, 6 - rownum
from dual
connect by level <= 5;

commit;

select * from lookup;
```

| SOME\_ID | MULTIPLIER |
|----------|------------|
| 1        | 5          |
| 2        | 4          |
| 3        | 3          |
| 4        | 2          |
| 5        | 1          |

Selecting from a nested table of objects, join with `LOOKUP` table and group results by attribute of nested object

```sql
select
  objs.id as id,
  objs.sub_item.some_id as some_id,
  sum( objs.sub_item.unit_cost * lookup.multiplier ) as some_value
  from
    table (get_items( 10 )) objs
    join lookup
    on lookup.some_id = objs.sub_item.some_id
 group by objs.id, objs.sub_item.some_id
 order by
   objs.id,
   objs.sub_item.some_id;
```

| ITEM\_ID | SOME\_ID | SOME\_VALUE |
|----------|----------|-------------|
| 1        | 1        | 0.5         |
| 2        | 2        | 0.8         |
| 3        | 3        | 0.9         |
| 4        | 4        | 0.8         |
| 5        | 5        | 0.5         |
| 6        | 1        | 3           |
| 7        | 2        | 2.8         |
| 8        | 3        | 2.4         |
| 9        | 4        | 1.8         |
| 10       | 5        | 1           |

**Conclusion**
Joining to the lookup table generates correct results\
Now the same as above, but this time we will INSERT the data from the query into the `TARGET` table.

```sql
insert into target (
  test_case_number,
  item_id,
  some_id,
  some_value
  )
select
  2 as test_case_number,
  objs.id as id,
  objs.sub_item.some_id as some_id,
  sum( objs.sub_item.unit_cost * lookup.multiplier ) as some_value
  from
    table (get_items( 10 )) objs
    join lookup
    on lookup.some_id = objs.sub_item.some_id
 group by objs.id, objs.sub_item.some_id
 order by
   objs.id,
   objs.sub_item.some_id;

commit;

select
  item_id,
  some_id,
  some_value
  from target
 where test_case_number = 2
 order by
   item_id,
   some_id;
```

This is where we hit the **BUG**.
Data in column `SOME_ID` is wrong.
Instead of getting value of nested object attribute for each row, the insert statement gets the value of the last nested object attribute from collection.

| ITEM\_ID | SOME\_ID | SOME\_VALUE |
|----------|----------|-------------|
| 1        | 5        | 0.5         |
| 2        | 5        | 0.8         |
| 3        | 5        | 0.9         |
| 4        | 5        | 0.8         |
| 5        | 5        | 0.5         |
| 6        | 5        | 3           |
| 7        | 5        | 2.8         |
| 8        | 5        | 2.4         |
| 9        | 5        | 1.8         |
| 10       | 5        | 1           |

## Workarounds

### Nested inline view

Encapsulating table of objects in an inline view and introducing column aliases for queried attributes works around that bug.

```sql
insert into target (
  test_case_number,
  item_id,
  some_id,
  some_value
  )
select
  3 as test_case_number,
  objs.id,
  objs.some_id,
  sum( objs.unit_cost * lookup.multiplier ) as some_value
  from (
    select
      o.id as id,
      o.sub_item.some_id as some_id,
      o.sub_item.unit_cost as unit_cost
     from table ( get_items( 10 ) ) o
    ) objs
    join lookup
    on lookup.some_id = objs.some_id
 group by objs.id, objs.some_id
 order by
   objs.id,
   objs.some_id;

commit;

select
  item_id,
  some_id,
  some_value
  from target
 where test_case_number = 3
 order by
   item_id,
   some_id;
```

| ITEM\_ID | SOME\_ID | SOME\_VALUE |
|----------|----------|-------------|
| 1        | 1        | 0.5         |
| 2        | 2        | 0.8         |
| 3        | 3        | 0.9         |
| 4        | 4        | 0.8         |
| 5        | 5        | 0.5         |
| 6        | 1        | 3           |
| 7        | 2        | 2.8         |
| 8        | 3        | 2.4         |
| 9        | 4        | 1.8         |
| 10       | 5        | 1           |

### Named subquery with ORDER BY

Another alternative solution is encapsulating the whole select statement into a named subquery using `WITH` clause.

```sql
insert into target (
  test_case_number,
  item_id,
  some_id,
  some_value
  )
  with
    input_data as (
    select
      4 as test_case_number,
      objs.id,
      objs.sub_item.some_id,
      sum( objs.sub_item.unit_cost * lookup.multiplier )
      from
        table (get_items( 10 )) objs
        join lookup
        on lookup.some_id = objs.sub_item.some_id
     group by objs.id, objs.sub_item.some_id
     order by
       objs.id,
       objs.sub_item.some_id
    )
select *
  from input_data;

commit;

select
  item_id,
  some_id,
  some_value
  from target
 where test_case_number = 4
 order by
   item_id,
   some_id;
```

| ITEM\_ID | SOME\_ID | SOME\_VALUE |
|----------|----------|-------------|
| 1        | 1        | 0.5         |
| 2        | 2        | 0.8         |
| 3        | 3        | 0.9         |
| 4        | 4        | 0.8         |
| 5        | 5        | 0.5         |
| 6        | 1        | 3           |
| 7        | 2        | 2.8         |
| 8        | 3        | 2.4         |
| 9        | 4        | 1.8         |
| 10       | 5        | 1           |

**Note**
This solution only works if the order by is used on the with clause. If we remove the ORDER BY from WITH clause the results are bad again.

```sql
insert into target (
  test_case_number,
  item_id,
  some_id,
  some_value
  )
  with
    input_data as (
    select
      5 as test_case_number,
      objs.id,
      objs.sub_item.some_id,
      sum( objs.sub_item.unit_cost * lookup.multiplier )
      from
        table (get_items( 10 )) objs
        join lookup
        on lookup.some_id = objs.sub_item.some_id
     group by objs.id, objs.sub_item.some_id
    )
select *
  from input_data
 order by 1, 2
;

commit;

select
  item_id,
  some_id,
  some_value
  from target
 where test_case_number = 5
 order by
   item_id,
   some_id;
```

| ITEM\_ID | SOME\_ID | SOME\_VALUE |
|----------|----------|-------------|
| 1        | 5        | 0.5         |
| 2        | 5        | 0.8         |
| 3        | 5        | 0.9         |
| 4        | 5        | 0.8         |
| 5        | 5        | 0.5         |
| 6        | 5        | 3           |
| 7        | 5        | 2.8         |
| 8        | 5        | 2.4         |
| 9        | 5        | 1.8         |
| 10       | 5        | 1           |

### Named subquery with MATERIALIZE hint

Interestingly, when the named subquery is materialized, the results are correct again.

```sql
insert into target (
  test_case_number,
  item_id,
  some_id,
  some_value
  )
  with
    input_data as (
    select /*+ materialize */
      6 as test_case_number,
      objs.id,
      objs.sub_item.some_id,
      sum( objs.sub_item.unit_cost * lookup.multiplier )
      from
        table (get_items( 10 )) objs
        join lookup
        on lookup.some_id = objs.sub_item.some_id
     group by objs.id, objs.sub_item.some_id
    )
select *
  from input_data
 order by 1, 2
;

commit;

select
  item_id,
  some_id,
  some_value
  from target
 where test_case_number = 6
 order by
   item_id,
   some_id;
```

| ITEM\_ID | SOME\_ID | SOME\_VALUE |
|----------|----------|-------------|
| 1        | 1        | 0.5         |
| 2        | 2        | 0.8         |
| 3        | 3        | 0.9         |
| 4        | 4        | 0.8         |
| 5        | 5        | 0.5         |
| 6        | 1        | 3           |
| 7        | 2        | 2.8         |
| 8        | 3        | 2.4         |
| 9        | 4        | 1.8         |
| 10       | 5        | 1           |

## Cleanup of created objects

```sql
drop function get_items;
drop type items;
drop type item;
drop type keys_obj;
drop table lookup;
drop table target;
```

## Tested database versions

The above code produced same (bad) results on following versions of Oracle DB that I've checked against:

- 12c release 1
- 12c release 2
- 18c
- 19c
- All of above is also reproducible on LiveSQL see  [this demo script](https://livesql.oracle.com/apex/livesql/s/jwqcc9zd751kh1wbj62wzhi8c)
