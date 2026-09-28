---
title: "utPLSQL v2 vs. ruby-plsql - exception handling"
date:
  created: 2015-08-17
slug: utplsql-v2-vs-ruby-plsql-exception-handling
categories:
  - "PLSQL"
  - "ruby-plsql"
  - "testing"
tags:
  - "PL/SQL"
  - "utPLSQL v2"
  - "ruby-plsql"
  - "unit testing"
---

I recently use utPLSQL in my daily work as a testing framework and I've noticed that the framework is doing quite bad job on exception handling on the tested code.
I'll try to demonstrate it with a simple/yet realistic scenario.

<!-- more -->

# Scenario

Consider a CUSTOMERS table created and populated with following script

```sql
DROP TABLE customers;

CREATE TABLE customers (
  customer_id NUMBER,
  customer_no NUMBER(10,0),
  customer_valid_flag VARCHAR2(1),
  CONSTRAINT chk_valid_flag CHECK (customer_valid_flag IN ('0','1'))
);
ALTER TABLE customers ADD CONSTRAINT customers_pk PRIMARY KEY (customer_id) USING INDEX;

INSERT INTO customers VALUES (1, 123, '1');

INSERT INTO customers VALUES (2, 234, '0');

COMMIT;
```

We have an API package to populate and access the CUSTOMERS table data.

```sql
CREATE OR REPLACE PACKAGE some_table_api AS

  FUNCTION get_customer_no(
    p_customer_id customers.customer_id%TYPE
  ) RETURN customers.customer_no%TYPE;
  FUNCTION get_customer_valid_flag(
    p_customer_id customers.customer_id%TYPE
  ) RETURN customers.customer_valid_flag%TYPE;
  PROCEDURE add_customer(
    p_customer_id         customers.customer_id%TYPE,
    p_customer_no         customers.customer_no%TYPE,
    p_customer_valid_flag customers.customer_valid_flag%TYPE
  );
END some_table_api;
/
```

```sql
CREATE OR REPLACE PACKAGE BODY some_table_api AS

  FUNCTION get_customer_no(
    p_customer_id customers.customer_id%TYPE
  ) RETURN customers.customer_no%TYPE IS
    v_result customers.customer_no%TYPE;
    BEGIN
      SELECT c.customer_no INTO v_result
        FROM customers c  WHERE c.customer_id = p_customer_id;

      RETURN v_result;
    END;

  FUNCTION get_customer_valid_flag(
    p_customer_id customers.customer_id%TYPE
  ) RETURN customers.customer_valid_flag%TYPE IS
    v_result customers.customer_valid_flag%TYPE;
    BEGIN
      SELECT c.customer_valid_flag INTO v_result
      FROM customers c WHERE c.customer_id = p_customer_id;

      RETURN v_result;
    END;

  PROCEDURE add_customer(
    p_customer_id         customers.customer_id%TYPE,
    p_customer_no         customers.customer_no%TYPE,
    p_customer_valid_flag customers.customer_valid_flag%TYPE
  ) IS
  BEGIN
    INSERT INTO customers
      ( customer_id, customer_no, customer_valid_flag )
    VALUES
      (p_customer_id, p_customer_no, p_customer_valid_flag);
  END;

END some_table_api;
/
```

# Unit tests

The above API Package is subject to unit testing.
To keep it short, I'll focus on testing of the add\_customer function.
Let's create a simple test to verify that when I call the procedure add\_customer, the customer row is added to the table.
Using utPLSQL.

```sql
CREATE OR REPLACE PACKAGE ut_some_table_api_new IS

  PROCEDURE ut_setup;

  PROCEDURE ut_teardown;

  PROCEDURE ut_add_customer;

END;
/
CREATE OR REPLACE PACKAGE BODY ut_some_table_api_new IS
  PROCEDURE ut_setup IS
    BEGIN
      NULL;
    END;

  PROCEDURE ut_teardown IS
    BEGIN
      ROLLBACK;
    END;

  PROCEDURE ut_add_customer IS
      v_row_count   INTEGER;
      v_customer_id INTEGER := -1;
    BEGIN
      SELECT COUNT(1) INTO v_row_count FROM customers
       WHERE customer_id = v_customer_id;
      utAssert.eq ( 'GIVEN customer does not exist', v_row_count, 0 );

      some_table_api.add_customer( v_customer_id, 123, '1' );

      SELECT COUNT(1) INTO v_row_count FROM customers
      WHERE customer_id = v_customer_id;
      utAssert.eq ( 'THEN customer is added', v_row_count, 1 );
    END;

END;
/
```

Using ruby-plsql.

```ruby
describe 'Add a new customer' do

  it 'adds a new customer to the table' do
    customer_id = -1
    #GIVEN
    expect( plsql.customers.count( :customer_id => customer_id) ).to eq( 0 )
    #WHEN
    plsql.some_table_api.add_customer( customer_id, 123, 'Y' )
    #THEN
    expect( plsql.customers.count(:customer_id => customer_id) ).to eq( 1 )
  end

end
```

Let us now execute the tests to make sure they validate the procedure.
First utPLSQL.

```
SQL> SET SERVEROUTPUT ON
SQL> BEGIN  utConfig.autocompile(false);  utplsql.run('ut_some_table_api_new');END;
  2  /
.
>    SSSS   U     U   CCC     CCC   EEEEEEE   SSSS     SSSS
>   S    S  U     U  C   C   C   C  E        S    S   S    S
>  S        U     U C     C C     C E       S        S
>   S       U     U C       C       E        S        S
>    SSSS   U     U C       C       EEEE      SSSS     SSSS
>        S  U     U C       C       E             S        S
>         S U     U C     C C     C E              S        S
>   S    S   U   U   C   C   C   C  E        S    S   S    S
>    SSSS     UUU     CCC     CCC   EEEEEEE   SSSS     SSSS
.
SUCCESS: ".ut_some_table_api_new"
.
> Individual Test Case Results:
>
SUCCESS - ut_some_table_api_new.UT_ADD_CUSTOMER: EQ "GIVEN customer does not
exist" Expected "0" and got "0"
>
SUCCESS - ut_some_table_api_new.UT_ADD_CUSTOMER: EQ "THEN customer is added"
Expected "1" and got "1"
>
```

And ruby-plsql

```
C:\Users\Jacek\RubymineProjects\utplsql_vs_plsql_spec>rspec -fd spec\some_table_api\add_customer_spec.rb

Add a new customer
  adds a new customer to the table

Finished in 0.04701 seconds (files took 1.95 seconds to load)
1 example, 0 failures

C:\Users\Jacek\RubymineProjects\utplsql_vs_plsql_spec>
```

All good. both test frameworks do their job.

# Catching exceptions

Lets have a look on how the unit tests will behave when a change to the CUSTOMERS table structure will be introduced.
We've just added a unique constraint to the CUSTOMERS table.

```sql
ALTER TABLE customers ADD CONSTRAINT customers_uk UNIQUE (customer_no) USING INDEX;
```

Our utPLSQL unit tests start to fail.

```
SQL> BEGIN  utConfig.autocompile(false);  utplsql.run('ut_some_table_api_new');END;
  2  /
.
>  FFFFFFF   AA     III  L      U     U RRRRR   EEEEEEE
>  F        A  A     I   L      U     U R    R  E
>  F       A    A    I   L      U     U R     R E
>  F      A      A   I   L      U     U R     R E
>  FFFF   A      A   I   L      U     U RRRRRR  EEEE
>  F      AAAAAAAA   I   L      U     U R   R   E
>  F      A      A   I   L      U     U R    R  E
>  F      A      A   I   L       U   U  R     R E
>  F      A      A  III  LLLLLLL  UUU   R     R EEEEEEE
.
FAILURE: ".ut_some_table_api_new"
.
> Individual Test Case Results:
>
SUCCESS - ut_some_table_api_new.UT_ADD_CUSTOMER: EQ "GIVEN customer does not
exist" Expected "0" and got "0"
>
FAILURE - ut_some_table_api_new.UT_ADD_CUSTOMER: Unable to run
ut_some_table_api_new.UT_ADD_CUSTOMER: ORA-00001: unique constraint
(TDD_TEST1.CUSTOMERS_UK) violated
>
>
> Errors recorded in utPLSQL Error Log:
>
> NONE FOUND

PL/SQL procedure successfully completed.

SQL>
```

And our ruby-plsql tests start to fail too

```
C:\Users\Jacek\RubymineProjects\utplsql_vs_plsql_spec>rspec -fd spec\some_table_api\add_customer_spec.rb

Add a new customer
  adds a new customer to the table (FAILED - 1)

Failures:

  1) Add a new customer adds a new customer to the table
     Failure/Error: plsql.some_table_api.add_customer( customer_id, 123, '1' )
     OCIError:
       ORA-00001: unique constraint (TDD_TEST1.CUSTOMERS_UK) violated
       ORA-06512: at "TDD_TEST1.SOME_TABLE_API", line 31
       ORA-06512: at line 2
     # stmt.c:250:in oci8lib_191.so
     # ./spec/some_table_api/add_customer_spec.rb:8:in `block (2 levels) in <top (required)>'

Finished in 0.04401 seconds (files took 1.7 seconds to load)
1 example, 1 failure

Failed examples:

rspec ./spec/some_table_api/add_customer_spec.rb:3 # Add a new customer adds a new customer to the table

C:\Users\Jacek\RubymineProjects\utplsql_vs_plsql_spec>
```

Let's have a closer look at the failures and what kind of information they provide.
When we get rid of all the noise from utPLSQL report, we see that there is an error message there.

```
FAILURE - ut_some_table_api_new.UT_ADD_CUSTOMER: Unable to run
ut_some_table_api_new.UT_ADD_CUSTOMER: ORA-00001: unique constraint
(TDD_TEST1.CUSTOMERS_UK) violated
```

The failure informs about the unit testing package procedure "ut\_some\_table\_api\_new.UT\_ADD\_CUSTOMER" failing.
It also gives a piece of information on what is wrong "ORA-00001: unique constraint (TDD\_TEST1.CUSTOMERS\_UK) violated".
Not bad, but the information is not sufficient to easily nail the piece of code that is failing.
On the other hand, ruby-plsql gives us the following:

```
  1) Add a new customer adds a new customer to the table
     Failure/Error: plsql.some_table_api.add_customer( customer_id, 123, '1' )
     OCIError:
       ORA-00001: unique constraint (TDD_TEST1.CUSTOMERS_UK) violated
       ORA-06512: at "TDD_TEST1.SOME_TABLE_API", line 31
       ORA-06512: at line 2
     # stmt.c:250:in oci8lib_191.so
     # ./spec/some_table_api/add_customer_spec.rb:8:in `block (2 levels) in <top (required)>'
```

We have full error stack trace from Oracle, includint the line number of the tested SOME\_TABLE\_API code, where the exception was raised.
We also have line number of the ruby-plsql unit test, where it failed.
This way it's much easier to trace the exception and figure out what happened.
**utPLSQL falls behind ruby-plsql in this match**, as ruby-plsql gives a full stack trace of the exception that was not captured.

# Testing for exceptions

Now, assuming that adding the constraint was deliberate and valid action, it would be good to have our tests fixed and also extend them to cover the scenario of unique constraint violation.
Let's give it a try with utPLSQL.
I've changed the existing test to validate that the customer with given ID and No does not exist.
I've added 2 more tests to check that ADD\_CUSTOMERS fails when a customer exits with given ID or number.
I've also extracted the SELECT statement into a function, as it is used all over the place and I've added a ROLLBACK statement after each test, to make them atomic.
[sql highlight="55,70"]
CREATE OR REPLACE PACKAGE ut\_some\_table\_api\_new IS
PROCEDURE ut\_setup;
PROCEDURE ut\_teardown;
PROCEDURE ut\_add\_customer;
PROCEDURE ut\_add\_customer\_fail\_id;
PROCEDURE ut\_add\_customer\_fail\_no;
END;
/
CREATE OR REPLACE PACKAGE BODY ut\_some\_table\_api\_new IS
PROCEDURE ut\_setup IS
BEGIN
NULL;
END;
PROCEDURE ut\_teardown IS
BEGIN
ROLLBACK;
END;
FUNCTION count\_customers( p\_customer\_id INTEGER, p\_customer\_no INTEGER ) RETURN INTEGER IS
v\_row\_count INTEGER;
BEGIN
SELECT COUNT(1) INTO v\_row\_count FROM customers
WHERE customer\_id = p\_customer\_id OR customer\_no = p\_customer\_no;
RETURN v\_row\_count;
END;
PROCEDURE ut\_add\_customer IS
v\_customer\_id INTEGER := -1;
v\_customer\_no INTEGER := 1234;
BEGIN
utAssert.eq ( 'GIVEN customer does not exist', count\_customers( v\_customer\_id, v\_customer\_no ), 0 );
some\_table\_api.add\_customer( v\_customer\_id, v\_customer\_no, '1' );
utAssert.eq ( 'THEN customer is added', count\_customers( v\_customer\_id, v\_customer\_no ), 1 );
ROLLBACK;
END;
PROCEDURE ut\_add\_customer\_fail\_id IS
v\_customer\_id INTEGER := -1;
v\_customer\_no INTEGER := 1234;
BEGIN
utAssert.eq ( 'GIVEN customer does not exist', count\_customers( v\_customer\_id, v\_customer\_no ), 0 );
some\_table\_api.add\_customer( v\_customer\_id, v\_customer\_no, '1' );
utAssert.eq ( 'WHEN customer is added', count\_customers( v\_customer\_id, v\_customer\_no ), 1 );
utAssert.throws('THEN the add\_customer fails on the same CUSTOMER\_ID'
,'some\_table\_api.add\_customer( '||v\_customer\_id||', '||(v\_customer\_no+1)||', ''1'' )'
,'DUP\_VAL\_ON\_INDEX');
ROLLBACK;
END;
PROCEDURE ut\_add\_customer\_fail\_no IS
v\_customer\_id INTEGER := -1;
v\_customer\_no INTEGER := 1234;
BEGIN
utAssert.eq ( 'GIVEN customer does not exist', count\_customers( v\_customer\_id, v\_customer\_no ), 0 );
some\_table\_api.add\_customer( v\_customer\_id, v\_customer\_no, '1' );
utAssert.eq ( 'WHEN customer is added', count\_customers( v\_customer\_id, v\_customer\_no ), 1 );
utAssert.throws('THEN the add\_customer fails on the same CUSTOMER\_NO'
,'some\_table\_api.add\_customer( '||(v\_customer\_id-1)||', '||v\_customer\_no||', ''1'' )'
,-1);
ROLLBACK;
END;
END;
/
[/sql]
So what do we get from those tests now?

```
SQL> BEGIN  utConfig.autocompile(false);  utplsql.run('ut_some_table_api_new');END;
  2  /
.
>    SSSS   U     U   CCC     CCC   EEEEEEE   SSSS     SSSS
>   S    S  U     U  C   C   C   C  E        S    S   S    S
>  S        U     U C     C C     C E       S        S
>   S       U     U C       C       E        S        S
>    SSSS   U     U C       C       EEEE      SSSS     SSSS
>        S  U     U C       C       E             S        S
>         S U     U C     C C     C E              S        S
>   S    S   U   U   C   C   C   C  E        S    S   S    S
>    SSSS     UUU     CCC     CCC   EEEEEEE   SSSS     SSSS
.
SUCCESS: ".ut_some_table_api_new"
.
> Individual Test Case Results:
>
SUCCESS - ut_some_table_api_new.UT_ADD_CUSTOMER: EQ "GIVEN customer does not
exist" Expected "0" and got "0"
>
SUCCESS - ut_some_table_api_new.UT_ADD_CUSTOMER: EQ "THEN customer is added"
Expected "1" and got "1"
>
SUCCESS - ut_some_table_api_new.UT_ADD_CUSTOMER_FAIL_ID: EQ "GIVEN customer does
not exist" Expected "0" and got "0"
>
SUCCESS - ut_some_table_api_new.UT_ADD_CUSTOMER_FAIL_ID: EQ "WHEN customer is
added" Expected "1" and got "1"
>
SUCCESS - ut_some_table_api_new.UT_ADD_CUSTOMER_FAIL_ID: RAISES "THEN the
add_customer fails on the same CUSTOMER_ID" Result: Block
"some_table_api.add_customer( -1, 1235, '1' )" raises  Exception
"DUP_VAL_ON_INDEX
>
SUCCESS - ut_some_table_api_new.UT_ADD_CUSTOMER_FAIL_NO: EQ "GIVEN customer does
not exist" Expected "0" and got "0"
>
SUCCESS - ut_some_table_api_new.UT_ADD_CUSTOMER_FAIL_NO: EQ "WHEN customer is
added" Expected "1" and got "1"
>
SUCCESS - ut_some_table_api_new.UT_ADD_CUSTOMER_FAIL_NO: THROWS "THEN the
add_customer fails on the same CUSTOMER_NO" Result: Block
"some_table_api.add_customer( -2, 1234, '1' )" raises  Exception "-1
>
>
> Errors recorded in utPLSQL Error Log:
>
> NONE FOUND

PL/SQL procedure successfully completed.

SQL>
```

Let's give it a try with ruby-plsql
[ruby highlight="24,34"]
describe 'Add a new customer' do
def count\_customers(customer\_id, customer\_no)
plsql.customers.count( 'WHERE customer\_id = :customer\_id OR customer\_no = :customer\_no', customer\_id, customer\_no )
end
it 'adds a new customer to the table' do
customer\_id, customer\_no = -1, 1234
#GIVEN
expect( count\_customers( customer\_id, customer\_no ) ).to eq( 0 )
#WHEN
plsql.some\_table\_api.add\_customer( customer\_id, customer\_no, '1' )
#THEN
expect( count\_customers( customer\_id, customer\_no ) ).to eq( 1 )
end
it 'fails to add a new customer on PK' do
customer\_id, customer\_no = -1, 1234
#GIVEN
expect( count\_customers( customer\_id, customer\_no ) ).to eq( 0 )
#WHEN
plsql.some\_table\_api.add\_customer( customer\_id, customer\_no, '1' )
#THEN
expect{ plsql.some\_table\_api.add\_customer( customer\_id, customer\_no+1, '1' ) }.to raise\_exception(/CUSTOMERS\_PK/)
end
it 'fails to add a new customer on UK' do
customer\_id, customer\_no = -1, 1234
#GIVEN
expect( count\_customers( customer\_id, customer\_no ) ).to eq( 0 )
#WHEN
plsql.some\_table\_api.add\_customer( customer\_id, customer\_no, '1' )
#THEN
expect{ plsql.some\_table\_api.add\_customer( customer\_id-1, customer\_no, '1' ) }.to raise\_exception(/ORA-00001/)
end
end
[/ruby]
And the results.

```
C:\Users\Jacek\RubymineProjects\utplsql_vs_plsql_spec>rspec -fd spec\some_table_api\add_customer_spec.rb

Add a new customer
  adds a new customer to the table
  fails to add a new customer on PK
  fails to add a new customer on UK

Finished in 0.07101 seconds (files took 1.82 seconds to load)
3 examples, 0 failures

C:\Users\Jacek\RubymineProjects\utplsql_vs_plsql_spec>
```

With utPLSQL it is possible to check for a predefined Oracle exception by name (like DUP\_VAL\_ON\_INDEX) or any exception by specific exception number.
The trouble is, that to check for an exception you need to pass the tested code as a string literal.
If your procedure is accessing complex types that can't be easily expressed with literals (object/clob/blob/cursor) you simply cant use it.
ruby-plsql (well actually RSpec) has an assertion on exception text. Matching can be done using regular expression. So you can validate that the raised exception (stack trace) contains the information you expect.
When you are testing code for exceptions with ruby-plsql, you actually are injecting a code block into the "expect" routine using the block brackets "{ }".
In utPLSQL you're also injecting the code to be executed into the "raises" assertion. In PL/SQL however, to dynamically inject code into another code, you pass it as a string. This makes the unit tests less readable and introduces many limitations.

# Tricky one

How can we make an assertion/expectation using UTPLSQL and / or ruby-plsql that the code beeing tested is not raising exception, so that when it's not raising any exceptions it's passing and when it's raising an exception, it fails?
I'll leave it up to you, but believe me it's not a straight answer for UTPLSQL.
