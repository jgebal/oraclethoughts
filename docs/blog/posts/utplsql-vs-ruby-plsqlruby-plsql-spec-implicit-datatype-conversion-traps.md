---
title: "utPLSQL v2 vs. ruby-plsql/ruby-plsql-spec – implicit datatype conversion traps"
date:
  created: 2015-08-11
slug: utplsql-v2-vs-ruby-plsqlruby-plsql-spec-implicit-datatype-conversion-traps
categories:
  - "PLSQL"
  - "ruby-plsql"
  - "testing"
tags:
  - "TDD"
  - "utPLSQL v2"
  - "ruby-plsql"
  - "unit testing"
---

[![UTPLSQL_vs_RSpec](../../images/UTPLSQL_vs_RSpec.png)](../posts/utplsql-vs-ruby-plsqlruby-plsql-spec-implicit-datatype-conversion-traps.md)
This post is a continuation of the utPLSQL vs. ruby-plsql series, you might want to have a look at my previous post for introduction and some basics.

<!-- more -->

# Foreword

Recently I've spent quite a lot of time investigating utPLSQL and ruby-plsql. Working on my previous articles made me think about very foundations of PL/SQL language.
Sadly, the longer I think about it, the more holes in the foundations of the language I see. Oracle made a large effort to make PL/SQL incredibly efficient and highly integrated with SQL language. That gives the power of processing data in a procedural form within the database engine, but during the journey something bad has happened, some strange or simply bad design decisions were made that never got fixed. I don't know why they were not fixed, my only guess is that because of the old-school backward-compatibility approach, Oracle cannot and dare not to change the foundations of PL/SQL language, as it might cause their customers some headache. Looking at all the modern languages and frameworks I realise that the only thing that is stopping progress if the attachment to historical solutions. You simply can't provide dramatically better solution without changes to the foundations. Google, Apple, Microsoft seem to already know that. When will it come to Oracle?
I'm really tempted to change the subject of this article and elaborate on the dark sides of PL/SQL language. After using it for over 15 years I've finally came to realise, that some of the things with the language are simply wrong and some of basic things are missing.
All of this thinking began some time ago and covers many different subject areas like polymorphism, exception handling, operators, primitve datatypes, object and array datatypes, derived datatypes [(%TYPE)](http://docs.oracle.com/cd/E11882_01/appdev.112/e25519/type_attribute.htm) and implicit datatype conversions.
Though all of those mentioned above are very interesting and I'd like to spent some time to organize my thoughts and put them into some meaningful statements, today I'll focus on derived datatypes and implicit datatype conversion issue in context of Unit Testing.

# The derived datatype in Oracle

Oracle PL/SQL comes with a great feature that allows developer to declare a variable or constant with datatype derived from a different database object.
In my career I've seen many different approaches and standards being used in projects. Some were demanding usage of derived datatypes, some were strictly forbidding the usage of those. Personally I think that each feature as a potentially valid use-case, but you really need to be aware of the potential consequences. Specially when you're dealing with a huge enterprise system that is developed and maintained by multiple teams with many dependencies.
While thinking about Unit Testing and detecting changes/errors that should captured by unit tests for PL/SQL code, the usage of derived datatype scenario came into my mind as a best way to demonstrate the issue of implicit datatype conversion.
Consider the following table exists in your database and is populated with data

```sql
CREATE TABLE customers (
  customer_id NUMBER,
  customer_no NUMBER(10,0),
  customer_valid_flag VARCHAR2(1),
  CONSTRAINT chk_valid_flag CHECK (customer_valid_flag IN ('0','1'))
);

INSERT INTO customers VALUES (1, 123, '1');

INSERT INTO customers VALUES (2, 234, '0');

COMMIT;
```

And you have created an API package for the table using derived dataypes.

```sql
CREATE OR REPLACE PACKAGE some_table_api AS

  FUNCTION get_customer_no(
    p_customer_id customers.customer_id%TYPE
  ) RETURN customers.customer_no%TYPE;
  FUNCTION get_customer_valid_flag(
    p_customer_id customers.customer_id%TYPE
  ) RETURN customers.customer_valid_flag%TYPE;

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

END some_table_api;
/
```

You've probably noticed that entire package is depending on structure of the CUSTOMERS table.
All the variables, input parameters and return datatypes are derived from CUSTOMERS table using the %TYPE keyword.
But what is the real value behind using the %TYPE here?
I tend to think that this kind of implementation makes this package resistant to underlying datatype changes in the table structure, but it is actually moving the problem to the level above, the caller of the function.
The API functions in this package are created to be used by someone. It can be another PL/SQL code, it can be code of Java application or a remote database calling the function.
Whoever is now using the function, he is to worry about the datatype compatibility, not me - my hands are clean and my code works! **Haha!**
Well... being a professional developer I think I should care a bit more and think a bit more about potential customers / users of the function.
Maybe I should have something to validate that the change will not break the contract of the API.
By the contract I mean, that my function get\_customer\_valid\_flag is supposed to return VARCHAR.
I'd like to have test validating that it is returning a VARCHAR value.
The below examples are a bit abstract but they cam into my mind when thinking of implicit datatype conversion.

# The problem of implicit datatype conversion in Oracle

Ok, let's write a set of simple unit tests to validate the package works as expected.

```sql
CREATE OR REPLACE PACKAGE ut_some_table_api IS
  PROCEDURE ut_setup;
  PROCEDURE ut_teardown;

  PROCEDURE ut_get_cust_no_fixed_types;

  PROCEDURE ut_get_cust_valid_fixed_types;

  PROCEDURE ut_get_cust_no_inheritance;

  PROCEDURE ut_get_cust_valid_inheritance;

END;
/
```

```sql
CREATE OR REPLACE PACKAGE BODY ut_some_table_api IS
  PROCEDURE ut_setup IS
    BEGIN
      INSERT INTO customers
              ( customer_id, customer_no, customer_valid_flag )
       VALUES ( -1, 12345678, '1' );
    END;

  PROCEDURE ut_teardown IS
    BEGIN
      ROLLBACK;
    END;

  PROCEDURE ut_get_cust_no_fixed_types IS
    v_result NUMBER;
    BEGIN
      v_result := some_table_api.get_customer_no(-1);
      utAssert.eq ( 'returns expected customer number', v_result, 12345678 );
    END;

  PROCEDURE ut_get_cust_valid_fixed_types IS
    v_result VARCHAR2(1);
    BEGIN
      v_result := some_table_api.get_customer_valid_flag(-1);
      utAssert.eq ('returns expected customer validity flag', v_result, '1' );
    END;

  PROCEDURE ut_get_cust_no_inheritance IS
    BEGIN
      utAssert.eq ( 'returns expected customer number', some_table_api.get_customer_no(-1), 12345678 );
    END;

  PROCEDURE ut_get_cust_valid_inheritance IS
    BEGIN
      utAssert.eq ( 'returns expected customer number', some_table_api.get_customer_valid_flag(-1), '1' );
    END;

END;
/
```

As you can see. I've made 2 tests for each function.
First two are using variables with fixed datatypes of VARCHAR2(1) and NUMBER, last two are calling the functions directly inline of utAssert call, so no datatypes and variables are declared in tests.
Let's have a look at and make sure the tests run successfully.

```
SQL> BEGIN  utConfig.autocompile(false);  utplsql.test('some_table_api');END;
  2  /
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
SUCCESS: ".some_table_api"
.
> Individual Test Case Results:
>
SUCCESS - some_table_api.UT_GET_CUST_NO_FIXED_TYPES: EQ "returns expected
customer number" Expected "12345678" and got "12345678"
>
SUCCESS - some_table_api.UT_GET_CUST_NO_INHERITANCE: EQ "returns expected
customer number" Expected "12345678" and got "12345678"
>
SUCCESS - some_table_api.UT_GET_CUST_VALID_FIXED_TYPES: EQ "returns expected
customer validity flag" Expected "1" and got "1"
>
SUCCESS - some_table_api.UT_GET_CUST_VALID_INHERITANCE: EQ "returns expected
customer number" Expected "1" and got "1"
>
>
> Errors recorded in utPLSQL Error Log:
>
```

Great. So all of my tests are passing.
We can also have similar tests written in ruby-plsql.
Let's have a look.

```ruby
describe 'get customer number' do

  before(:all) do
    plsql.customers.insert({customer_id: -1, customer_no: 123456, customer_valid_flag: '1' })
  end

  it 'returns expected customer number as a numeric value' do
    expect( plsql.some_table_api.get_customer_no(-1) ).to eq 123456
  end

  it 'returns expected customer number as a numeric value through assignment' do
    result = plsql.some_table_api.get_customer_no(-1)
    expect( result ).to eq 123456
  end

end

describe 'get customer valid flag' do

  before(:all) do
    plsql.customers.insert({customer_id: -1, customer_no: 123456, customer_valid_flag: '1' })
  end

  it 'returns expected customer validity flag as a character' do
    expect( plsql.some_table_api.get_customer_valid_flag(-1) ).to eq '1'
  end

  it 'returns expected customer validity flag as a character through assignment' do
    result = plsql.some_table_api.get_customer_valid_flag(-1)
    expect( result ).to eq '1'
  end
end
```

And let's see if those tests actually run successfully.

```
C:\Users\Jacek\RubymineProjects\utplsql_vs_plsql_spec>rspec spec\some_table_api\*
....

Finished in 0.07801 seconds (files took 1.5 seconds to load)
4 examples, 0 failures
```

Now. The column CUSTOMER\_VALID\_FLAG is actually storing an integer value of 0 or 1 but the datatype in table is VARCHAR2(1). This seems wrong and after a while we want to adjust the table, also business users came back to us with a feedback that CUSTOMER\_NO is supposed to allow character values in it.
So we will change the structure of the CUSTOMERS table to fit the new needs.
I actually will create a new table, move all the data, drop the old one and rename new to old.

```sql
CREATE TABLE customers_new (
  customer_id NUMBER,
  customer_no VARCHAR2(30),
  customer_valid_flag NUMBER(1,0)
);

INSERT INTO customers_new SELECT * FROM customers;

DROP TABLE customers;

RENAME customers_new TO customers;
```

Ok, so we have a new table structure in place, the package should do OK after the changes, but the API is actually working on different datatypes now.
So how will my tests behave after the change?
Lets check utPLSQL.

```
SQL> BEGIN  utConfig.autocompile(false);  utplsql.test('some_table_api');END;
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
SUCCESS: ".some_table_api"
.
> Individual Test Case Results:
>
SUCCESS - some_table_api.UT_GET_CUST_NO_FIXED_TYPES: EQ "returns expected
customer number" Expected "12345678" and got "12345678"
>
SUCCESS - some_table_api.UT_GET_CUST_NO_INHERITANCE: EQ "returns expected
customer number" Expected "12345678" and got "12345678"
>
SUCCESS - some_table_api.UT_GET_CUST_VALID_FIXED_TYPES: EQ "returns expected
customer validity flag" Expected "1" and got "1"
>
SUCCESS - some_table_api.UT_GET_CUST_VALID_INHERITANCE: EQ "returns expected
customer number" Expected "1" and got "1"
>
>
> Errors recorded in utPLSQL Error Log:
>
> NONE FOUND

PL/SQL procedure successfully completed.

SQL>
```

and ruby-plsql

```
C:\Users\Jacek\RubymineProjects\utplsql_vs_plsql_spec>rspec spec\some_table_api\*
FFFF

Failures:

  1) get customer number returns expected customer number as a numeric value
     Failure/Error: plsql.customers.insert({customer_id: -1, customer_no: 123456, customer_valid_flag: '1' })
     TypeError:
       can't convert Fixnum into String
     # ./spec/some_table_api/get_customer_no_spec.rb:4:in `block (2 levels) in <top (required)>'

  2) get customer number returns expected customer number as a numeric value through assignment
     Failure/Error: plsql.customers.insert({customer_id: -1, customer_no: 123456, customer_valid_flag: '1' })
     TypeError:
       can't convert Fixnum into String
     # ./spec/some_table_api/get_customer_no_spec.rb:4:in `block (2 levels) in <top (required)>'

  3) get customer valid flag returns expected customer validity flag as a character
     Failure/Error: plsql.customers.insert({customer_id: -1, customer_no: 123456, customer_valid_flag: '1' })
     TypeError:
       can't convert Fixnum into String
     # ./spec/some_table_api/get_customer_valid_flag_spec.rb:4:in `block (2 levels) in <top (required)>'

  4) get customer valid flag returns expected customer validity flag as a character through assignment
     Failure/Error: plsql.customers.insert({customer_id: -1, customer_no: 123456, customer_valid_flag: '1' })
     TypeError:
       can't convert Fixnum into String
     # ./spec/some_table_api/get_customer_valid_flag_spec.rb:4:in `block (2 levels) in <top (required)>'

Finished in 0.025 seconds (files took 1.48 seconds to load)
4 examples, 4 failures

Failed examples:

rspec ./spec/some_table_api/get_customer_no_spec.rb:7 # get customer number returns expected customer number as a numeric value
rspec ./spec/some_table_api/get_customer_no_spec.rb:11 # get customer number returns expected customer number as a numeric value through assignment
rspec ./spec/some_table_api/get_customer_valid_flag_spec.rb:7 # get customer valid flag returns expected customer validity flag as a character
rspec ./spec/some_table_api/get_customer_valid_flag_spec.rb:11 # get customer valid flag returns expected customer validity flag as a character through assignment

C:\Users\Jacek\RubymineProjects\utplsql_vs_plsql_spec>
```

# Conclusions

upPLSQL unit tests went through smoothly without any indication of potential issues. For me **it's a big STOP sign**, as the testing framework should allow me to validate that I get exactly what I expect.
ruby-plsql unit tests failed on data setup (inserting test data). The tests captured that the datatypes are not appropriate. Amazingly, Ruby programming language is far more strict on datatypes than Oracle even though it is a dynamic language that supports [duck-typing](https://en.wikipedia.org/wiki/Duck_typing).
Oracle SQL and PL/SQL, due to the build-in implicit datatype conversion treats integers as strings, strings as dates etc. interchangeably. The feature that was supposed to make developers lives easier actually becomes a big pain as you cant disable it and have no control over it.

## Update

I've decided to write a continuation of this post, you may read it [here](../posts/utplsql-vs-ruby-plsqlruby-plsql-spec-implicit-datatype-conversion-traps-continue.md).
