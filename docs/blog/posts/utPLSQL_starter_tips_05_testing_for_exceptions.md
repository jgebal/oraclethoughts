---
title: "Testing for exceptions with --%throws"
date:
  created: 2026-08-08
slug: utPLSQL_starter_tips_testing_exceptions
categories:
  - "PLSQL"
  - "utPLSQL"
  - "testing"
tags:
  - "utPLSQL"
  - "unit testing"
---

# Testing for exceptions with --%throws

Throwing exceptions is a common practice of handling situations when the program meets unsupported combination of data or conditions.
Oracle PL/SQL language provides several mechanisms for raising and capturing exceptions.

utPLSQL provides the `--%throws` annotation that allows for verification of expected exceptions. 
Exception testing in utPLSQL is done through this annotation, not through `ut.expect(...)`.

<!-- more -->

The annotation accepts a lists the error codes, error variable names or predefined oracle exception names.

## Testing application errors by numbers 

Once of ways for the program to return an exception is by calling the `raise_application_error` procedure;
That procedure allows program to raise exception with a specific error number and error message;

utPLSQL allows developers to test for a specific application error by pointing to:
- the application error number
- the variable / constant that holds the specific application error number

The code below raises application error when value of `p_status` is unexpected.

```sql hl_lines="3 14"
create or replace package order_processing as

  error_number_variable   integer := -20002;

  procedure check_status(p_status varchar2);
end;
/

create or replace package body order_processing as

  procedure check_status(p_status varchar2) is
  begin
    if p_status not in ('NEW','PROCESSING','DONE') then
      raise_application_error( error_number_variable, 'nvalid order status' );
    end if;
  end;
end;
/
```

The unit tests below are verifying the behavior.
The verification can be done by referencing exception by **error number** or by pointing to a **numeric variable or constant** that returns the error number.

```sql hl_lines="5 9"
create or replace package test_orders as
  --%suite

  --%test(Rejects an invalid status by number)
  --%throws(-20002)
  procedure test_invalid_status_by_value;

  --%test(Rejects an invalid status by variable (error number))
  --%throws(order_processing.error_number_variable)
  procedure test_invalid_status_by_variable;
end;
/

create or replace package body test_orders as
              
  procedure test_invalid_status_by_value is
  begin
    order_processing.check_status(p_status => 'INVALID_VALUE');
  end;

  procedure test_invalid_status_by_variable is
  begin
    order_processing.check_status(p_status => 'INVALID_VALUE');
  end;
end;
/
```

After test execution
```sql
begin
    ut.run('test_orders');
end;
/
```

Both of the tests are passing. Application errors can be tested by error number or error number variable or constant. 
```
test_orders
  Rejects an invalid status by number [,014 sec]
  Rejects an invalid status by variable (error number) [,006 sec]

Finished in ,022376 seconds
2 tests, 0 failed, 0 errored, 0 disabled, 0 warning(s)
```

The test passes when `check_status` raises error `-20002`. It fails when:

- the call completes without raising any exception
- an exception with different error code is raised

Please note that the `--%throws` annotation is only applicable to the `--%test` procedure.

## Testing for exception variables

The same business logic can be implemented in another way, using Oracles `exception` variables.
The variables can be initialized with `pragma exception_init` or can be used uninitialized.

```sql hl_lines="3 4 5 17 24"
create or replace package order_processing as

  uninitialized_exception exception;
  initialized_exception   exception;
  pragma exception_init( initialized_exception, -20003);

  procedure check_status(p_status varchar2);
  procedure check_status_v2(p_status varchar2);
end;
/

create or replace package body order_processing as

  procedure check_status(p_status varchar2) is
  begin
    if p_status not in ('NEW','PROCESSING','DONE') then
      raise uninitialized_exception;
    end if;
  end;

  procedure check_status_v2(p_status varchar2) is
  begin
    if p_status not in ('NEW','PROCESSING','DONE') then
      raise initialized_exception;
    end if;
  end;
end;
/
```

To reference an uninitialized exception, the `--%throws` annotation must point to the exception variable name.
To reference an initialized exception, the `--%throws` annotation can point to the exception variable name or to the exception number.

```sql hl_lines="5 9 13"
create or replace package test_orders as
  --%suite

  --%test(Rejects an invalid status by uninitialized exception variable)
  --%throws(order_processing.uninitialized_exception)
  procedure test_invalid_status_by_uninitialized_exception_variable;

  --%test(Rejects an invalid status by initialized exception variable)
  --%throws(order_processing.initialized_exception)
  procedure test_invalid_status_by_initialized_exception_variable;

  --%test(Rejects an invalid status by exception number)
  --%throws(-20003)
  procedure test_invalid_status_by_initialized_exception_number;
end;
/

create or replace package body test_orders as
  
  procedure test_invalid_status_by_uninitialized_exception_variable is
  begin
    order_processing.check_status(p_status => 'INVALID_VALUE');
  end;
  procedure test_invalid_status_by_initialized_exception_variable is
  begin
    order_processing.check_status_v2(p_status => 'INVALID_VALUE');
  end;
  procedure test_invalid_status_by_initialized_exception_number is
  begin
    order_processing.check_status_v2(p_status => 'INVALID_VALUE');
  end;
end;
/
```

After test execution
```sql
begin
    ut.run('test_orders');
end;
/
```

Test is passing. utPLSQL allows developers to test against uninitialized exception variables (new in [utPLSQL 3.2.3](https://github.com/utPLSQL/utPLSQL/releases/tag/v.3.2.3) )
```
test_orders
  Rejects an invalid status by uninitialized exception variable [,009 sec]
  Rejects an invalid status by initialized exception variable [,008 sec]
  Rejects an invalid status by exception number [,011 sec]

Finished in ,030689 seconds
3 tests, 0 failed, 0 errored, 0 disabled, 0 warning(s)
```

## Oracle predefined exceptions and multiple exceptions

`--%throws` takes a comma-separated list, so a test can allow more than one acceptable error code. It also accepts predefined Oracle exception names such as `DUP_VAL_ON_INDEX`:

```sql hl_lines="2 3 4 7 11"
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