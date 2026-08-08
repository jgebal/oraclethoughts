---
title: "Testing exception raising with --%throws"
date:
  created: 2026-08-08
slug: utPLSQL_starter_tips_testing_query_output
categories:
  - "PLSQL"
  - "utPLSQL"
  - "testing"
tags:
  - "utPLSQL"
  - "unit testing"
---

# Testing exception raising with --%throws

utPLSQL provides the `--%throws` annotation for testing that a procedure raises an expected exception. 
Exception testing in utPLSQL is done through this annotation, not through `ut.expect(...)`.

The annotation accepts a lists the error codes, error variable names or predefined oracle exception names.
<!-- more -->

## User defined exception numbers 

You can use inl

```sql
create procedure set_order_status(p_id interer, p_status varchar2) is
begin
  --verify input 
  if p_status not in ('NEW','OPEN','PROCESSING','SHIPPING','CLOSED','CANCELLED') then
    raise_aplication_error( -20001, 'Invalid order status' );
  end if;
  --actual business logic below
end;
/

create or replace package test_orders as
  --%suite(Order status)

  --%test(Rejects an invalid status)
  --%throws(-20001)
  procedure test_invalid_status;
end;
/

create or replace package body test_orders as
  procedure test_invalid_status is
  begin
    set_order_status(p_id => 99, p_status => 'INVALID_VALUE');
  end;
end;
/
```

The test passes when `set_order_status` raises error `-20001`. It fails when:

- the call completes without raising any exception
- an exception with different error code is raised

Because `--%throws` only applies to the annotated `--%test` procedure itself, it cannot be attached to an arbitrary expression or call inline. The code under test is simply called from the body of the test procedure, as shown above.

## Oracle exception error codes

`--%throws` accepts any Oracle error code, not only application errors raised with `raise_application_error`:

```sql
create or replace package test_division as
  --%suite(Division)

  --%test(Fails when dividing by zero)
  --%throws(-1476)
  procedure test_zero_divisor;
end;
/

create or replace package body test_division as
  procedure test_zero_divisor is
    l_result number;
  begin
    l_result := 23/0;
  end;
end;
/
```

## Various and multiple exceptions

`--%throws` takes a comma-separated list, so a test can allow more than one acceptable error code. It also accepts package-level exception constants and variables, and predefined Oracle exception names such as `DUP_VAL_ON_INDEX`:

```sql
create or replace package test_exceptions as
  e_num_custom_error integer := -20145;
  e_custom_exception exception;
  pragma exception_init( e_custom_exception, -20146 );
  
  --%test(Raises one of several validation errors)
  --%throws(e_num_custom_error, e_custom_exception, -20189)
  procedure test_validation_errors;
  
  --%test(Raises a predefined Oracle exception)
  --%throws(DUP_VAL_ON_INDEX)
  procedure test_duplicate_key;
end;
/

create or replace package body test_division as

  procedure test_validation_errors is
    v_day pls_integer := extract(day from sysdate);
  begin
    case mod(v_day, 3)
        when 1 then raise_aplication_error( e_num_custom_error, 'Test error' );
        when 2 then raise e_custom_exception;
        when 0 then raise_aplication_error(-20189, 'Another test error');
    end case;
  end;
  
  procedure test_zero_divisor is
  begin
    raise dup_val_on_index;
  end;
end;
/
```

## Further reading

- [The --%throws annotation](https://www.utplsql.org/utPLSQL/latest/userguide/annotations.html#throws)
- [Expectations reference](https://www.utplsql.org/utPLSQL/latest/userguide/expectations.html#expecting-exceptions)