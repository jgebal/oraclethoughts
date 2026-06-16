---
title: "utPLSQL v2 vs. ruby-plsql/ruby-plsql-spec – implicit datatype conversion traps ... continued"
date:
  created: 2015-08-11
slug: utplsql-v2-vs-ruby-plsqlruby-plsql-spec-implicit-datatype-conversion-traps-continued
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
I've finished my [previous post](../posts/utplsql-vs-ruby-plsqlruby-plsql-spec-implicit-datatype-conversion-traps.md) a bit too soon and was not precise on the ruby-plsql unite test results analysis.
I've decided to dig a bit deeper to validate that ruby-plsql (RSpec) actually support datatype mismatch exceptions where utPLSQL unit testing fails due to oracle implicit datatype conversion.

<!-- more -->

# Retrospective

Lets have a look once more at where we landed.
There is a package with two functions.

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

The package was initially working on the below table structure.

```sql
CREATE TABLE customers (
  customer_id NUMBER,
  customer_no NUMBER(10,0),
  customer_valid_flag VARCHAR2(1),
  CONSTRAINT chk_valid_flag CHECK (customer_valid_flag IN ('0','1'))
);
```

But the table structure was changed:

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

and so the datatypes returned by functions have changed.
We have following ruby-plsql unit tests.

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

And the tests are now failing.

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

The thing is, the test did not fail because the function return datatype has changed.
The tests have actually failed on the insert statement due to the fact that the table structure was changed.
That got me thinking. Those little and simple tests already have some dependency to table CUSTOMERS, though the CUSTOMERS table itself is not being tested. The tests should validate API regardless of the table structure.
So what can be done?

# Plain SQL initialization

Since it is Ruby that guards datatypes more than Oracle, I can use Oracle to do the inserts and take advantage of the Oracle "feature" of implicit datatype conversion.
Let's give it a try with get\_customer\_no function

```ruby
describe 'get customer number using pure SQL setup' do

  before(:all) do
    plsql.execute <<-SQL
      INSERT INTO customers(customer_id, customer_no,customer_valid_flag)
      VALUES (-1, 123456, '1')
    SQL
  end

  it 'returns expected customer number as a numeric value' do
    expect( plsql.some_table_api.get_customer_no(-1) ).to eq 123456
  end

  it 'returns expected customer number as a numeric value through assignment' do
    result = plsql.some_table_api.get_customer_no(-1)
    expect( result ).to eq 123456
  end

end
```

And the results of tests are:

```
C:\Users\Jacek\RubymineProjects\utplsql_vs_plsql_spec>rspec spec\some_table_api\get_customer_no_pure_sql_spec.rb
FF

Failures:

  1) get customer number using pure SQL setup returns expected customer number as a numeric value
     Failure/Error: expect( plsql.some_table_api.get_customer_no(-1) ).to eq 123456

       expected: 123456
            got: "123456"

       (compared using ==)
     # ./spec/some_table_api/get_customer_no_pure_sql_spec.rb:11:in `block (2 levels) in <top (required)>'

  2) get customer number using pure SQL setup returns expected customer number as a numeric value through assignment
     Failure/Error: expect( result ).to eq 123456

       expected: 123456
            got: "123456"

       (compared using ==)
     # ./spec/some_table_api/get_customer_no_pure_sql_spec.rb:16:in `block (2 levels) in <top (required)>'

Finished in 0.03701 seconds (files took 1.75 seconds to load)
2 examples, 2 failures

Failed examples:

rspec ./spec/some_table_api/get_customer_no_pure_sql_spec.rb:10 # get customer number using pure SQL setup returns expected customer number as a numeric value
rspec ./spec/some_table_api/get_customer_no_pure_sql_spec.rb:14 # get customer number using pure SQL setup returns expected customer number as a numeric value through assignment

C:\Users\Jacek\RubymineProjects\utplsql_vs_plsql_spec>
```

After overcoming the tests failure on COUSTOMERS table, the tests are still failing.
The tests have failed on the assertions, as they should. We expected to get an integer and received a string since the datatypes of functions have changed. Cool!
