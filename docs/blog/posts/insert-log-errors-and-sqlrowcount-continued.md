---
title: "INSERT ... LOG ERRORS and SQL%ROWCOUNT-continued"
date:
  created: 2020-03-17
slug: insert-log-errors-and-sqlrowcount-continued
categories:
  - "SQL"
tags:
  - "SQL"
  - "exception handling"
---

In my [previous post](../posts/insert-log-errors-and-sqlrowcount.md) I have described solution allowing you to obtain count of error rows that get inserted into error table when using Oracle SQL syntax of `INSERT INTO ... SELECT ... FROM ... LOG ERRORS`.
The solution provided had a bug related to resetting counters.

<!-- more -->

You might say, that the problem was with how the function `dml_utils.get_count` was implemented. But in fact the problem is related to what requirements.
The requirement was to be able to access counter value multiple times after the counter was populated in SQL Statement.
At the same time we would like to be able to control when the counter gets reset.
Ideally, invocation of next SQL statement that involves the counter should reset the counter.
This is exactly how it was implemented, however I've forgot to consider that a SQL statement will only call counter if data is processed.
So if no rows were processed by SQL, the counter should be reset anyways ans so the call to `dml_utils.get_count` should return 0 rows.
Below are the important parts of the flaky code I've created.

```sql
create or replace package body dml_utils as

  type row_count_info is record (
    cnt         row_count := 1,
    needs_reset boolean := false
  );
  type table_row_counts is table of row_count_info index by object_name;
  
  g_first_count row_count_info;

  g_row_counts_for_table table_row_counts;

  procedure count_row( p_table_name object_name ) is
  begin
    if g_row_counts_for_table.exists( p_table_name ) 
      and not g_row_counts_for_table( p_table_name ).needs_reset then
      g_row_counts_for_table( p_table_name ).cnt := g_row_counts_for_table( p_table_name ).cnt + 1;
    else
      g_row_counts_for_table( p_table_name ) := g_first_count;
    end if;
  end;

  function get_count( p_table_name object_name := no_object ) return row_count is
    l_result row_count := 0;
  begin
    if g_row_counts_for_table.exists( p_table_name ) then
      g_row_counts_for_table( p_table_name ).needs_reset := true;
      l_result  := g_row_counts_for_table( p_table_name ).cnt;
    end if;
    return l_result;
  end;
 
end;
/
```

The `count_row` will:

- if counter exists and the counter dose not need reset (default), increment the counter value by 1
- otherwise initialize counter with value of 1

The `get_count` will:

- return the count captured by counter
- set `needs_reset := true;` so that next call to `count_row` would reset counter and in effect restart counting

This logic is flaky as it assumes that each SQL statement, that potentially can call counter, will call it.
The logic above doesn't consider 0 rows-processed SQL statements and in those cases, counter will report bad values.
The simple way to prevent this problem is to give full control over the counters to developer.
So instead of using `get_count` and expecting the counters to reset automatically, developers sill have two different functions at hand and also a procedure:

- function `peek_count` - returns a value of counter specified item without resetting counter
- function `pop_count` - returns a value of counter specified item and reset counter
- procedure `reset_count` - resets counter

Full source of `dml_util` package

```sql
create or replace package dml_utils as

  subtype object_name is varchar2(258);
  subtype row_count is naturaln;
  no_object constant object_name := '[{no table}]';

  /**
    Adds row to a counter for specified item
  */
  procedure count_row( p_name object_name );

  /**
    Adds row to a counter for specified item and returns 1
  */
  function count_row( p_name object_name := no_object ) return integer;

  /**
    Adds row to a counter for specified item and returns the provided p_txt
  */
  function count_row( p_name object_name, p_txt varchar2 ) return varchar2;

  /**
    Returns a value of counter specified item without resetting counter
  */
  function peek_count( p_name object_name := no_object ) return row_count;

  /**
    Returns a value of counter specified item and resets counter
  */
  function pop_count( p_name object_name := no_object ) return row_count;

  /**
    Resets counter for specified item
  */
  procedure reset_count( p_name object_name := no_object );

  /**
    Return statement to create a trigger for counting rows on a specified table
  */
  function  generate_count_trigger( p_name object_name ) return varchar2;

  /**
    Creates a trigger for counting rows on a specified table
  */
  procedure create_count_trigger( p_name object_name );

  /**
    Drops trigger for counting rows on a specified table
  */
  procedure drop_count_trigger( p_name object_name );
end;
```

```sql
create or replace package body dml_utils as

  type table_row_counts is table of row_count index by object_name;
  g_items_counter table_row_counts;

  function get_counter_value( p_name object_name, p_reset boolean ) return row_count is
    l_result row_count := 0;
  begin
    if g_items_counter.exists( p_name ) then
      l_result := g_items_counter( p_name );
    end if;
    if p_reset then
      reset_count( p_name );
    end if;
    return l_result;
  end;

  procedure count_row( p_name object_name ) is
  begin
    if g_items_counter.exists( p_name ) then
      g_items_counter( p_name ) := g_items_counter( p_name ) + 1;
    else
      g_items_counter( p_name ) := 1;
    end if;
  end;

  function count_row( p_name object_name := no_object ) return integer is
  begin
    count_row( p_name );
    return 1;
  end;

  function count_row( p_name object_name, p_txt varchar2 ) return varchar2 is
  begin
    count_row( p_name );
    return p_txt;
  end;

  function peek_count( p_name object_name := no_object ) return row_count is
  begin
    return get_counter_value( p_name, false );
  end;

  function pop_count( p_name object_name := no_object ) return row_count is
  begin
    return get_counter_value( p_name, true );
  end;

  procedure reset_count( p_name object_name := no_object) is
  begin
    if g_items_counter.exists( p_name ) then
      g_items_counter.delete( p_name );
    end if;
  end;

  function generate_count_trigger( p_name object_name ) return varchar2 is
    l_table_name   object_name := dbms_assert.sql_object_name( p_name );
    l_trigger_name object_name := l_table_name||'_AI';
  begin
    return '
     create or replace trigger '||l_trigger_name
      ||' before insert on '||l_table_name||q'[ for each row
     declare
     begin
       dml_utils.count_row(']'||l_table_name||q'[');
     end;]';
  end;

  procedure create_count_trigger( p_name object_name ) is
    pragma autonomous_transaction;
  begin
    execute immediate generate_count_trigger(p_name);
  end;

  procedure drop_count_trigger( p_name object_name ) is
    pragma autonomous_transaction;
  begin
    execute immediate 'drop trigger '||dbms_assert.sql_object_name( p_name )||'_AI';
  end;

end;
```

With the above implementation, even the counters work correctly, even if no rows are counted.

```sql
create table t1 (id integer primary key , v1 integer, v2 integer);
call dbms_errlog.create_error_log('t1');
call dml_utils.create_count_trigger('ERR$_T1');

declare
  l_insert_count integer;
  l_error_count integer;
begin
  insert
    into t1(id, v1, v2)
  select 1,2,3
    from dual
    connect by level <=5
      log errors reject limit unlimited;

  ut.expect(sql%rowcount).to_equal( 1 );
  ut.expect(dml_utils.pop_count('ERR$_T1')).to_equal( 4 );

  insert
    into t1(id, v1, v2)
  select 2,3,4
    from dual
      log errors reject limit unlimited;

  ut.expect(sql%rowcount).to_equal( 1 );
  ut.expect(dml_utils.pop_count('ERR$_T1')).to_equal( 0 );
end;
/
```
