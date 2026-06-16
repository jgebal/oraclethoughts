---
title: "utPLSQL v2 vs. ruby-plsql/ruby-plsql-spec - part one"
date:
  created: 2015-06-14
slug: utplsql-v2-vs-ruby-plsqlruby-plsql-spec-part-one
categories:
  - "PLSQL"
  - "SQL"
  - "testing"
tags:
  - "TDD"
  - "utPLSQL v2"
  - "ruby-plsql"
  - "unit testing"
---

![UTPLSQL_vs_RSpec](../../images/UTPLSQL_vs_RSpec-300x56.png)

# Foreword

[Unit Testing](https://en.wikipedia.org/wiki/Unit_testing) is around for quite a while. Since it started to become more and more popular, quite a few tools became available for Oracle database to allow unit testing of the database.
There are the UI based tools like Quest Code Tester (now Dell Code Tester for Oracle), Oracle SQL Developer unit testing. There is [DBFit by Goyko Adzic](http://gojko.net/fitnesse/dbfit/) for regression and functional testing of databases (including but not limited to Oracle), there are probably many more of that kind, that I'm not aware of.
There are also pure programming language based frameworks like **UTPLSQL** and **ruby-plsql**. There are probably (and hopefully) many more of that sort that I'm not aware of.
Unit tests are meant to be created and maintained by software developers and are to help developers keep their code clean and valid throughout entire project life cycle.
UI based frameworks for unit testing are putting a high abstraction and tight facade between the code and the developer. Those frameworks tend to limit the richness and variety of possible implementations for a test case and therefore are not suited for software developers. The time and cost of development and maintenance for UI-based unit tests is usually way beyond the benefits.
Variety of program units that are possible to develop is infinite and the only limitations to what the unit can do are the programming language boundaries and developers creativity. For this reason itself, it is best to use a programming language of similar or higher flexibility to describe and test the behaviour for a unit.
In this series of article I will focus on two programming language-based unit testing frameworks for Oralce, that I got familiar with and had opportunity to use.

<!-- more -->

---

# History and numbers

[UTPLSQL](http://utplsql.sourceforge.net/) created by Steven Feuerstein in 2000 - yes 15 years ago!
It was maintained till 2005, then the project died for 8 years. It was revitalised in 2014. It had [13 releases](http://sourceforge.net/projects/utplsql/files/utPLSQL/) up till 06.2015.
The total downloads of the project according to [sourceforge stats](http://sourceforge.net/projects/utplsql/files/utPLSQL/stats/timeline?dates=2000-06-15+to+2015-06-21) is ~47 000
It seems to become quite popular among Oracle developers and therefore revitalized. I heard people in Poland knowing at least what it is. I'm aware of few quite large companies that actually use it as a tool of choice.
[ruby-plsql](https://github.com/rsim/ruby-plsql) created by [Raimonds Simanovskis](http://blog.rayapps.com/tags/ruby-plsql/) was first released in 2008, eight years after UTPLSQL.
Since then it had [15 releases](https://github.com/rsim/ruby-plsql/releases) till 06.2015.
It was downloaded over 68 000 times as a Ruby gem, [according to rubygems.org](https://rubygems.org/search?utf8=%E2%9C%93&query=ruby-plsql)
[ruby-plsql-spec](https://github.com/rsim/ruby-plsql-spec) created by [Raimonds Simanovskis](http://blog.rayapps.com/tags/ruby-plsql/) is a neat utility library on top of ruby-plsql that aims to ease all the fuss around setup for RSpec unit testing easier. It also provides [code-coverage reporting](http://blog.rayapps.com/2010/10/05/ruby-plsql-spec-gem-and-code-coverage-reporting/).
It had 4 releases and is currently in version 0.4.0.
It was downloaded over 5 000 times as a Ruby gem, [according to rubygems.org](https://rubygems.org/search?utf8=%E2%9C%93&query=ruby-plsql-spec)
[RSpec](https://www.relishapp.com/rspec) is an industry standard Ruby Unit Testing framework for [Behaviour Driven Development (BDD)](https://en.wikipedia.org/wiki/RSpec).
It was first released in 07.2005 and since then had 159 versions published.
It was downloaded over 24 000 000 times [according to rubygems.org](https://rubygems.org/gems/rspec)

---

# Thanks

Before I dig into the details, I'd like to give a big thank you to Steven Feuerstein for creating UTPLSQL.
Creation of UTPLSQL was an enormous achievement. It was the first successful initiative to take Oracle database Unit Testing seriously and provide set of tools for writing unit tests for SQL and PLSQL code.
I'd like to thank Raimonds Simanovskis for taking the challenge to build a more flexible future-proofed and feature rich library for Oracle unit testing. Thanks to the library, within months, two database developers without any former knowledge of Ruby, RSpec and the library, managed to create over 1500 valuable unit tests for a complex, computation intensive, legacy database system. Tests were automatically pulled from version control system into continuous integration server and were actually delivering continuous feedback for the 6-person database development team. The tests became indicators of the quality and stability of the developed database and provided a safety harness for huge refactoring and performance improvements, that no one would dare to do without the tests giving certainty that all the functional requirements are still met.

# A note on naming used in comparison

RSpec, ruby-plsql and ruby-plsql-spec along with some other Ruby libraries build a framework for Oracle unit testing within Ruby programming language. Those libraries provide different sets of functionalities that comprise the whole utility. In the below sections I will be comparing UTPLSQL to Ruby, RSpec, ruby-plsql or ruby-plsql-spec, depending on which features are compared.

---

# The Conceptual differences between UTPLSQL / ruby-plsql

---

UTPLSQL is based on the Oracle database, SQL and PL/SQL procedural programming language.

RSpec, ruby-plsql and ruby-plsql-spec are libraries of Ruby, object oriented, dynamic language.

\

UTPLSQL library itself resides within the database that is to be tested.

ruby-plsql, ruby-plsql-spec and all the dependant libraries are part of Ruby gems within Ruby installation outside of database. Much like Oracle Client software, they live on the "client" side.

\

UTPLSQL unit tests must compiled into the database in the same schema/user as the tested program units.

RSpec tests reside outside of database and only contact the database to perform the testing operation.

\

UTPLSQL unit tests executed from within database that is to be tested.

RSpec unit tests are executed from the project source control directory.

\

UTPLSQL tests are executed within a context of a single session/database connection.

ruby-plsql manages database connections by using Oracle OCI8 library (Ruby/Rubinius) or Oracle JDBC driver (JRuby). Any test can refer to multiple users/connections/databases if needed.

---

## Implications

The conceptual differences are fundamental.
ruby-plsql is not invasive. The tests live outside of the database and only contact the database to perform the testing operation, their existence has no direct impact on the database state.
UTPLSQL is invasive. The library itself as well as the tests live inside the database that it is testing. It affects the overall health of the database, when tests are invalidated. It needs to coexist with the product code. The very existence of unit tests and unit testing library on DEV database make it incompatible with all non-DEV databases. The databases are simply different. Importing non-DEV database to DEV database will wipe-out all the tests (and probably the testing library too).
ruby-plsql test can refer to multiple users/connections/databases within single test if needed. It manages database connections by using Oracle OCI8 library (Ruby/Rubinius) or Oracle JDBC driver (JRuby) and allows creation of cross-database / cross session / cross user tests. Unit test is on top of database connection.
UTPLSQL tests are executed within a context of a single session/database connection. The dependency is inverted and the test is executed within the context of a particular connection, so the framework does not allow such flexibility. Of course you can use some external tools to manage connections and run tests in different contexts but that is not part of the framework and still doesn't allow a single tests to reference several connections.
In most of the cases, using UTPLSQL, you would connect to the database to execute all of your tests in the context of a user (the tester). This is what the framework supports and that seems to be simplest approach.
But what if you would like to test something across two users?

## Example code and tests

We have a requirement to have a database package for sending a message between different database sessions.
The package could look like the one below.

```sql
CREATE OR REPLACE PACKAGE message_api AS
  --Source from https://oracle-base.com/articles/misc/dbms_pipe
  PROCEDURE send_msg (
    p_number  IN  NUMBER,
    p_text    IN  VARCHAR2,
    p_date    IN  DATE DEFAULT SYSDATE);
  PROCEDURE receive_msg (
    p_number  OUT  NUMBER,
    p_text    OUT  VARCHAR2,
    p_date    OUT  DATE);
END message_api;
/

CREATE OR REPLACE PACKAGE BODY message_api AS
  --Source from https://oracle-base.com/articles/misc/dbms_pipe
  PROCEDURE send_msg (
    p_number  IN  NUMBER,
    p_text    IN  VARCHAR2,
    p_date    IN  DATE DEFAULT SYSDATE
  ) AS
    l_status  NUMBER;
    BEGIN
      DBMS_PIPE.pack_message(p_number);
      DBMS_PIPE.pack_message(p_text);
      DBMS_PIPE.pack_message(p_date);

      l_status := DBMS_PIPE.send_message('message_pipe');
      IF l_status != 0 THEN
        RAISE_APPLICATION_ERROR(-20001, 'message_pipe error');
      END IF;
    END;

  PROCEDURE receive_msg (
    p_number  OUT  NUMBER,
    p_text    OUT  VARCHAR2,
    p_date    OUT  DATE
  ) AS
    l_result  INTEGER;
    BEGIN
      l_result := DBMS_PIPE.receive_message (
          pipename => 'message_pipe', timeout  => DBMS_PIPE.maxwait);
      IF l_result = 0 THEN
        -- Message received successfully.
        DBMS_PIPE.unpack_message(p_number);
        DBMS_PIPE.unpack_message(p_text);
        DBMS_PIPE.unpack_message(p_date);
      ELSE
        RAISE_APPLICATION_ERROR(-20002,
           'message_api.receive was unsuccessful. Return result: ' || l_result);
      END IF;
    END;
END message_api;
/
```

With ruby-plsql you can create a set of tests to cover different scenarios.

```ruby
require_relative '../spec_helper'

describe 'cross session communication package' do

  it 'allows receiving message sent in one session' do
    a_message = { p_number: 1, p_text: 'a message', p_date: Time.today }
    plsql.message_api.send_msg( a_message )
    expect( plsql.message_api.receive_msg ).to eq( a_message )
  end

  it 'allows sending and receiving message across different sessions' do
    message_values = [ 1, 'a message', Time.today ]
    plsql(:default).message_api.send_msg( *message_values )
    received_values = plsql(:secondary).message_api.receive_msg.values
    expect( received_values ).to eq( message_values )
  end

  it 'allows sending and receiving message across sessions of different users' do
    # a very PL/SQL'ish style of code
    a_number = 1
    a_text = 'a message'
    a_date = Time.today
    plsql.message_api.send_msg( a_number, a_text, a_date)

    result = plsql(:different_user).message_api.receive_msg

    expect( result[:p_number] ).to eq( a_number )
    expect( result[:p_text] ).to eq( a_text )
    expect( result[:p_date] ).to eq( a_date )
  end

end
```

We can create an UTPLSQL package to test that the package is sending message.
The UTPLSQL will not allwo us however to test it across different sessions (the inverted dependency).
So the example below will represent only the first test case we have created in ruby-plsql.

```sql
CREATE OR REPLACE PACKAGE ut_message_api IS
  PROCEDURE ut_setup;
  PROCEDURE ut_teardown;

  PROCEDURE ut_send_receive_msg;
END;
/
CREATE OR REPLACE PACKAGE BODY ut_message_api IS
  PROCEDURE ut_setup IS
    BEGIN
      NULL;
    END;

  PROCEDURE ut_teardown IS
    BEGIN
      NULL;
    END;

  PROCEDURE ut_send_receive_msg IS
    l_num  NUMBER;
    l_txt  VARCHAR2(32767);
    l_date DATE;
    l_num_expected  NUMBER;
    l_txt_expected  VARCHAR2(32767);
    l_date_expected DATE;
    BEGIN
      l_num_expected := 1;
      l_txt_expected := 'a message';
      l_date_expected := TRUNC( SYSDATE );

      message_api.send_msg( l_num_expected, l_txt_expected, l_date_expected );

      message_api.receive_msg( l_num, l_txt, l_date );

      utAssert.eq ( 'when I send a message and receive it then I get a valid message number', l_num, l_num_expected );
      utAssert.eq ( 'and I get a valid message date', l_date, l_date_expected );
      utAssert.eq ( 'and I get a valid message text', l_txt, l_txt_expected );
    END;

END;
/
```

\
Looking at those two implementations of unit tests.
RSpec syntax is much closer to natural language. It is far more readable and dense.
It took 32 lines of code for 3 tests (one test was done over-verbose on purpose). There is very little noise in tests when you read them.
UTPLSQL required 41 lines of code to write one test. There is noise in the code and it really hard to put a meaningful text when calling utAssert.
The ruby-plsql unit tests are able to cover all the required test cases, as multi-session/multi-user tests are possible to implement.
UTPLSQL is not able to cover the required scenarios of multi-session/multi-user communication.
Ruby syntax is flexible enough so that you can start writing tests with syntax that is most fitted for you. It's possible to write very verbose tests. It is also possible to write very compact tests, once you learn how to do it.
**Coming up next:**

- [Setup and basic reporting](../posts/utplsql-vs-ruby-plsqlruby-plsql-spec-part-two.md)
- [(Not) reporting failures](../posts/utplsql-vs-ruby-plsqlruby-plsql-spec-part-three.md)
- [Language limitations](../posts/utplsql-vs-ruby-plsqlruby-plsql-spec-implicit-datatype-conversion-traps.md)
- Modularity
- Lexical differences
- Structuring and organizing
- and hopefully more.

**to be continued...**
