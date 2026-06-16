---
title: "utPLSQL v2 vs. ruby-plsql/ruby-plsql-spec - part three - (not) reporting failures"
date:
  created: 2015-06-28
slug: utplsql-v2-vs-ruby-plsqlruby-plsql-spec-part-three
categories:
  - "PLSQL"
  - "testing"
tags:
  - "TDD"
  - "utPLSQL v2"
  - "ruby-plsql"
  - "unit testing"
---

![UTPLSQL_vs_RSpec](../../images/UTPLSQL_vs_RSpec-300x56.png)

I have finished my [previous post](../posts/utplsql-vs-ruby-plsqlruby-plsql-spec-part-two.md) with comparison of basic reporting capabilities build into UTPLSQL and ruby-plsql frameworks for Oracle unit testing.

<!-- more -->

# Additional reporting options for ruby-plsql

So far I have been using the ruby-plsql-spec wrapper around RSpec to run unit tests, however ruby-plsql was created in a way, that developer is not limited to use the functionalities of ruby-plsql-spec only.
To run ruby-plsql unit tests we can use command `plsql-spec run`  that is provided with ruby-plsql-spec library or use the `rspec` command that comes with RSpec library.
Using RSpec allows richer build-in reporting capabilities.
A short look at `rspec --help` shows:

```
  **** Output ****

    -f, --format FORMATTER           Choose a formatter.
                                       [p]rogress (default - dots)
                                       [d]ocumentation (group and example names)
                                       [h]tml
                                       [j]son
                                       custom formatter class name
    -o, --out FILE                   Write output to a file instead of $stdout. This option applies
                                       to the previously specified --format, or the default format
                                       if no format is specified.
        --deprecation-out FILE       Write deprecation warnings to a file instead of $stderr.
    -b, --backtrace                  Enable full backtrace.
    -c, --[no-]color, --[no-]colour  Enable color in the output.
    -p, --[no-]profile [COUNT]       Enable profiling of examples and list the slowest examples (default: 10).
    -w, --warnings                   Enable ruby warnings
```

Just to remind, those are the outcomes of UTPLSQL reporter for execution **of a single test**.

```
SQL> SET SERVEROUTPUT ON 
SQL> BEGIN
  2    utplsql.test('message_api');
  3  END;
  4  /

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
SUCCESS: ".message_api"
.
> Individual Test Case Results:
>
SUCCESS - message_api.UT_SEND_RECEIVE_MSG: EQ "when I send a message and receive
it then I get a valid message number" Expected "1" and got "1"
>
SUCCESS - message_api.UT_SEND_RECEIVE_MSG: EQ "and I get a valid message date"
Expected "JUN-23-2015 12:00:00" and got "JUN-23-2015 12:00:00"
>
```

When we run `rpsec -fd`, we get a documentation format for **three unit tests** in a very human readable form as below.

```
cross session communication package
  allows receiving message sent in one session
  allows sending and receiving message across different sessions
  allows sending and receiving message across sessions of different users

Finished in 0.07851 seconds (files took 2.03 seconds to load)
3 examples, 0 failures
```

Using `rspec -fj -o rspec_test_results.json` we can generate output of tests into [rspec\_test\_results.json](../../images/rspec_test_results.json) file.
Using `rspec -fh -o rspec_test_results.html` we can generate output of tests into [rspec\_test\_results.html](../../images/rspec_test_results.html) file.
By installing library RspecJunitFormatter `gem install rspec_junit_formatter` we can have output generated into [rspec\_test\_results.xml](../../images/rspec_test_results.xml) XML format that is accepted by many Continuous Integration platforms like Jenkins.
All that needs to be done to have the XML JUnit-like output is to issue `rspec -f RspecJunitFormatter -o rspec_test_results.xml`.
I have described only the basic options of the formatting, there are many more formatters out there to be used, there are many different options of controlling how RSpec is executing tests like ordering, filtering, stop on first failure, dry-run and more.

# Handling test failures

Unit tests are being created to give a fast and clear feedback, when something goes wrong and a requirement for code is no longer met.
Let's have a look, how UTPLSQL and ruby-plsql (with RSpec) are reporting test failures and errors.
There are quite significant differences on how each of them is reporting errors and also **when and how they fail**.

# Graceful failure

This is the scenario, where a unit test is actually doing it's job by failing. The test fails to indicate that a requirement that is being verified, is not met.
For this scenario I will use a very simple code and similarly simple tests.
The tested code is:

```sql
CREATE OR REPLACE FUNCTION sum_two_numbers(a NUMBER, b NUMBER) RETURN NUMBER IS
  BEGIN
    RETURN a + b + 1;
  END;
/
```

The UTPLSQL unit test for this code is:

```sql
CREATE OR REPLACE PACKAGE ut_sum_two_numbers
IS
  PROCEDURE ut_setup;
  PROCEDURE ut_teardown;

  PROCEDURE ut_sum_two_numbers;
END ut_sum_two_numbers;
/
CREATE OR REPLACE PACKAGE BODY ut_sum_two_numbers
IS
  PROCEDURE ut_setup
  IS
    BEGIN
      NULL;
    END;

  PROCEDURE ut_teardown
  IS
    BEGIN
      NULL;
    END;

  PROCEDURE ut_sum_two_numbers
  IS
    BEGIN
      utAssert.eq ( 'sum_two_numbers returns 10 for arguments 5 and 5', sum_two_numbers(5,5), 10 );
    END;

END ut_sum_two_numbers;
/
```

And the ruby-plsql unit test for this code, doing exactly the same check is:

```ruby
require_relative 'spec_helper'

describe 'sum_two_numbers' do

  it 'returns 10 for arguments 5 and 5' do
    expect( plsql.sum_two_numbers(5,5) ).to eq 10
  end

end
```

The over-sizing of UPLSQL is clearly visible when handling small test-case.
Output from UTPLSQL test run.

```sql
SET SERVEROUPTUT ON
BEGIN
  utConfig.autocompile(false);
  utConfig.setreporter(NULL);
  utplsql.test('sum_two_numbers');
END;
/
```

```
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
FAILURE: ".sum_two_numbers"
.
> Individual Test Case Results:
>
FAILURE - sum_two_numbers.UT_SUM_TWO_NUMBERS: EQ "sum_two_numbers returns 10 for
arguments 5 and 5" Expected "10" and got "11"
>
>
> Errors recorded in utPLSQL Error Log:
>
> NONE FOUND

PL/SQL procedure successfully completed.
```

Output from ruby-plsql-spec test run.

```
rspec -fd spec/sum_two_numbers_spec.rb
```

```
sum_two_numbers
  returns 10 for arguments 5 and 5 (FAILED - 1)

Failures:

  1) sum_two_numbers returns 10 for arguments 5 and 5
     Failure/Error: expect( plsql.sum_two_numbers(5,5) ).to eq 10

       expected: 10
            got: 11

       (compared using ==)
     # ./spec/sum_two_numbers_spec.rb:6:in `block (2 levels) in <top (required)>'

Finished in 0.0335 seconds (files took 2.41 seconds to load)
1 example, 1 failure

Failed examples:

rspec ./spec/sum_two_numbers_spec.rb:5 # sum_two_numbers returns 10 for arguments 5 and 5
```

At first glance both reports seem to be similar. There are significant differences though.

- UTPLSQL is showing the package and procedure name, assertion name and type and the values compared
- RSpec is showing the spec file location and name, name of the test, assertion type, values compared and the line number for failure
- UTPLSQL **has no summary of tests**, we don't know how many tests were executed, how many succeeded an how many failed
- RSpec provides a summary report showing all that is needed to investigate the failures
- UTPLSQL announces big status at the **beginning** of textual report
- RSpec displays summary report at the **end** of textual report

# Reporting on large number of tests

The differences might seem slight and of a low value until you run a large set of tests.
I will simulate 100 tests by simply calling the function 100 times with different set of parameters.
I've created a separate function that causes wrong result for one value of parameter 'a'.

```sql
CREATE OR REPLACE FUNCTION sum_two_numbers_100_times(a NUMBER, b NUMBER) RETURN NUMBER IS
  BEGIN
    RETURN CASE WHEN a = 41 THEN 42 + b ELSE a + b END;
  END;
```

I've added new test procedure to UTPLSQL tests package

```sql
  PROCEDURE ut_sum_two_numbers_100_times
  IS
    BEGIN
      FOR i IN 1 .. 100 LOOP
        utAssert.eq ( 'sum_two_numbers returns '||(i+10)||' for arguments '||i||' and 10', sum_two_numbers_100_times(i,10), (i+10) );
      END LOOP;
    END;
```

and a new test in a loop to my spec file.

```ruby
  100.times do |i|
    it "returns #{i+10} for arguments #{i} and 10" do
      expect( plsql.sum_two_numbers_100_times(i,10) ).to eq i+10
    end
  end
```

When looking at the test outputs, keep in mind that using a text output to console makes the console scroll, and that you can see once the tests finish, are the very last lines of the output.
To mimic, what you will see, it's best to scroll to the end of the report output block.

```
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
FAILURE: ".sum_two_numbers"
.
> Individual Test Case Results:
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS: EQ "sum_two_numbers returns 10 for
arguments 5 and 5" Expected "10" and got "10"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 11 for arguments 1 and 10" Expected "11" and got "11"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 12 for arguments 2 and 10" Expected "12" and got "12"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 13 for arguments 3 and 10" Expected "13" and got "13"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 14 for arguments 4 and 10" Expected "14" and got "14"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 15 for arguments 5 and 10" Expected "15" and got "15"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 16 for arguments 6 and 10" Expected "16" and got "16"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 17 for arguments 7 and 10" Expected "17" and got "17"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 18 for arguments 8 and 10" Expected "18" and got "18"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 19 for arguments 9 and 10" Expected "19" and got "19"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 20 for arguments 10 and 10" Expected "20" and got "20"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 21 for arguments 11 and 10" Expected "21" and got "21"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 22 for arguments 12 and 10" Expected "22" and got "22"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 23 for arguments 13 and 10" Expected "23" and got "23"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 24 for arguments 14 and 10" Expected "24" and got "24"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 25 for arguments 15 and 10" Expected "25" and got "25"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 26 for arguments 16 and 10" Expected "26" and got "26"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 27 for arguments 17 and 10" Expected "27" and got "27"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 28 for arguments 18 and 10" Expected "28" and got "28"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 29 for arguments 19 and 10" Expected "29" and got "29"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 30 for arguments 20 and 10" Expected "30" and got "30"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 31 for arguments 21 and 10" Expected "31" and got "31"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 32 for arguments 22 and 10" Expected "32" and got "32"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 33 for arguments 23 and 10" Expected "33" and got "33"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 34 for arguments 24 and 10" Expected "34" and got "34"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 35 for arguments 25 and 10" Expected "35" and got "35"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 36 for arguments 26 and 10" Expected "36" and got "36"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 37 for arguments 27 and 10" Expected "37" and got "37"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 38 for arguments 28 and 10" Expected "38" and got "38"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 39 for arguments 29 and 10" Expected "39" and got "39"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 40 for arguments 30 and 10" Expected "40" and got "40"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 41 for arguments 31 and 10" Expected "41" and got "41"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 42 for arguments 32 and 10" Expected "42" and got "42"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 43 for arguments 33 and 10" Expected "43" and got "43"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 44 for arguments 34 and 10" Expected "44" and got "44"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 45 for arguments 35 and 10" Expected "45" and got "45"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 46 for arguments 36 and 10" Expected "46" and got "46"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 47 for arguments 37 and 10" Expected "47" and got "47"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 48 for arguments 38 and 10" Expected "48" and got "48"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 49 for arguments 39 and 10" Expected "49" and got "49"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 50 for arguments 40 and 10" Expected "50" and got "50"
>
FAILURE - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 51 for arguments 41 and 10" Expected "51" and got "52"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 52 for arguments 42 and 10" Expected "52" and got "52"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 53 for arguments 43 and 10" Expected "53" and got "53"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 54 for arguments 44 and 10" Expected "54" and got "54"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 55 for arguments 45 and 10" Expected "55" and got "55"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 56 for arguments 46 and 10" Expected "56" and got "56"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 57 for arguments 47 and 10" Expected "57" and got "57"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 58 for arguments 48 and 10" Expected "58" and got "58"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 59 for arguments 49 and 10" Expected "59" and got "59"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 60 for arguments 50 and 10" Expected "60" and got "60"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 61 for arguments 51 and 10" Expected "61" and got "61"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 62 for arguments 52 and 10" Expected "62" and got "62"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 63 for arguments 53 and 10" Expected "63" and got "63"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 64 for arguments 54 and 10" Expected "64" and got "64"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 65 for arguments 55 and 10" Expected "65" and got "65"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 66 for arguments 56 and 10" Expected "66" and got "66"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 67 for arguments 57 and 10" Expected "67" and got "67"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 68 for arguments 58 and 10" Expected "68" and got "68"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 69 for arguments 59 and 10" Expected "69" and got "69"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 70 for arguments 60 and 10" Expected "70" and got "70"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 71 for arguments 61 and 10" Expected "71" and got "71"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 72 for arguments 62 and 10" Expected "72" and got "72"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 73 for arguments 63 and 10" Expected "73" and got "73"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 74 for arguments 64 and 10" Expected "74" and got "74"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 75 for arguments 65 and 10" Expected "75" and got "75"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 76 for arguments 66 and 10" Expected "76" and got "76"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 77 for arguments 67 and 10" Expected "77" and got "77"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 78 for arguments 68 and 10" Expected "78" and got "78"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 79 for arguments 69 and 10" Expected "79" and got "79"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 80 for arguments 70 and 10" Expected "80" and got "80"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 81 for arguments 71 and 10" Expected "81" and got "81"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 82 for arguments 72 and 10" Expected "82" and got "82"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 83 for arguments 73 and 10" Expected "83" and got "83"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 84 for arguments 74 and 10" Expected "84" and got "84"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 85 for arguments 75 and 10" Expected "85" and got "85"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 86 for arguments 76 and 10" Expected "86" and got "86"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 87 for arguments 77 and 10" Expected "87" and got "87"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 88 for arguments 78 and 10" Expected "88" and got "88"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 89 for arguments 79 and 10" Expected "89" and got "89"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 90 for arguments 80 and 10" Expected "90" and got "90"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 91 for arguments 81 and 10" Expected "91" and got "91"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 92 for arguments 82 and 10" Expected "92" and got "92"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 93 for arguments 83 and 10" Expected "93" and got "93"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 94 for arguments 84 and 10" Expected "94" and got "94"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 95 for arguments 85 and 10" Expected "95" and got "95"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 96 for arguments 86 and 10" Expected "96" and got "96"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 97 for arguments 87 and 10" Expected "97" and got "97"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 98 for arguments 88 and 10" Expected "98" and got "98"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 99 for arguments 89 and 10" Expected "99" and got "99"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 100 for arguments 90 and 10" Expected "100" and got "100"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 101 for arguments 91 and 10" Expected "101" and got "101"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 102 for arguments 92 and 10" Expected "102" and got "102"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 103 for arguments 93 and 10" Expected "103" and got "103"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 104 for arguments 94 and 10" Expected "104" and got "104"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 105 for arguments 95 and 10" Expected "105" and got "105"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 106 for arguments 96 and 10" Expected "106" and got "106"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 107 for arguments 97 and 10" Expected "107" and got "107"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 108 for arguments 98 and 10" Expected "108" and got "108"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 109 for arguments 99 and 10" Expected "109" and got "109"
>
SUCCESS - sum_two_numbers.UT_SUM_TWO_NUMBERS_100_TIMES: EQ "sum_two_numbers
returns 110 for arguments 100 and 10" Expected "110" and got "110"
>
>
> Errors recorded in utPLSQL Error Log:
>
> NONE FOUND

PL/SQL procedure successfully completed.
```

The ruby-plsql-spec resutls are very dense, when using the default, progress reporting format.

```
rspec spec/sum_two_numbers_spec.rb
```

```
Running specs from spec\sum_two_numbers_spec.rb
..........................................F..........................................................

Failures:

  1) sum_two_numbers returns 51 for arguments 41 and 10
     Failure/Error: expect( plsql.sum_two_numbers_100_times(i,10) ).to eq i+10

       expected: 51
            got: 52

       (compared using ==)
     # ./spec/sum_two_numbers_spec.rb:11:in `block (3 levels) in <top (required)>'

Finished in 0.70409 seconds (files took 1.46 seconds to load)
101 examples, 1 failure

Failed examples:

rspec './spec/sum_two_numbers_spec.rb[1:43]' # sum_two_numbers returns 51 for arguments 41 and 10

Failing tests!
```

If you compare the two reports, you'll clearly notice the huge difference in readability of those two. The RSpec clearly shows what failed and where it failed. In UTPLSQL the information is lost in the amount of SUCCESS noise.

# UTPLSQL failing to run tests

 **One of the biggest problems with UTPLSQL is the fact that it can't run when dependencies are broken.**
This makes developers move away from using this tool. As the number of tests grow and database packages grow, one small change to package signature (change of specification) will invalidate all tests against this package and the tests will simply not execute.
When a change is done to PLSQL package, tests would first need to be fixed before they can be executed. This simple fact makes the existence of the tests a headache for a developer and decreases their value down to almost zero. The tests are unable to give a fast feedback.
Consider the package `message_api` code I've used in the [first article](../posts/utplsql-vs-ruby-plsqlruby-plsql-spec-part-one.md) of this series.
The package consists of two methods

```sql
  PROCEDURE send_msg (p_number IN NUMBER, p_text IN VARCHAR2, p_date IN DATE DEFAULT SYSDATE);
  PROCEDURE receive_msg (p_number OUT NUMBER, p_text OUT VARCHAR2, p_date OUT DATE);
```

What will happen, when I will add or remove or change the ordering of parameters?
Lets give it a try with adding a parameter to the `receive_msg` procedure.

```sql
  PROCEDURE receive_msg (p_number OUT NUMBER, p_text OUT VARCHAR2, p_date OUT DATE, p_received_on OUT TIMESTAMP);
```

When I execute my ruby-plsql-spec tests I get the following results

```
Running specs from spec\message_api\send_and_receive_spec.rb
FF.

Failures:

  1) cross session communication package allows receiving message sent in one session
     Failure/Error: expect( plsql.message_api.receive_msg ).to eq( a_message )

       expected: {:p_number=>1, :p_text=>"a message", :p_date=>2015-06-27 00:00:00.000000000 +0100}
            got: {:p_number=>1, :p_text=>"a message", :p_date=>2015-06-27 00:00:00.000000000 +0100, :p_received_on=>NULL}

       (compared using ==)

       Diff:
       @@ -1,4 +1,5 @@
        :p_date => 2015-06-27 00:00:00.000000000 +0100,
        :p_number => 1,
       +:p_received_on => NULL,
        :p_text => "a message",
     # ./spec/message_api/send_and_receive_spec.rb:8:in `block (2 levels) in <top (required)>'

  2) cross session communication package allows sending and receiving message across different sessions
     Failure/Error: expect( received_values ).to eq( message_values )

       expected: [1, "a message", 2015-06-27 00:00:00.000000000 +0100]
            got: [1, "a message", 2015-06-27 00:00:00.000000000 +0100, NULL]

       (compared using ==)
     # ./spec/message_api/send_and_receive_spec.rb:15:in `block (2 levels) in <top (required)>'

Finished in 0.06301 seconds (files took 1.63 seconds to load)
3 examples, 2 failures

Failed examples:

rspec ./spec/message_api/send_and_receive_spec.rb:5 # cross session communication package allows receiving message sent in one session
rspec ./spec/message_api/send_and_receive_spec.rb:11 # cross session communication package allows sending and receiving message across different sessions
```

The tests results clearly indicate what is happening

- three tests were executed
- two tests failed
- first test failed on comparison of parameter-value pairs and we see a diff of expected/got
- second test failed on comparison of list of returned values and we see that the returned values array contained more than expected
- third test passed, as **it was badly written**. The test was comparing individual elements of return values and a new value returned was simply not tested within the test. This single test is showing one of traps you can come into when unit testing. **One must really understand what is really tested in a test, to make the test reliable and valuable.**
- we see the names of failed tests and the lines at which those test failed

utplsql.test('message\_api') I get the following results

```
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
FAILURE: ".message_api"
.
> Individual Test Case Results:
>
FAILURE - .: Unable to run ut_message_api.ut_SETUP: ORA-04063: package body
"TDD_TEST1.UT_MESSAGE_API" has errors
ORA-06508: PL/SQL: could not find program
unit being called: "TDD_TEST1.UT_MESSAGE_API"
>
FAILURE - .: utPLSQL.test failure: User-Defined Exception
>
>
> Errors recorded in utPLSQL Error Log:
>
> NONE FOUND
BEGIN  utConfig.autocompile(false);  utplsql.test('message_api');END;
*
ERROR at line 1:
ORA-06510: PL/SQL: unhandled user-defined exception
ORA-06512: at "UTP.UTASSERT2", line 149
ORA-06512: at "UTP.UTASSERT", line 49
ORA-06512: at "UTP.UTPLSQL", line 952
ORA-06510: PL/SQL: unhandled user-defined exception
ORA-06512: at "UTP.UTASSERT2", line 149
ORA-06512: at "UTP.UTASSERT", line 49
ORA-06512: at "UTP.UTPLSQL", line 426
ORA-04063: package body "TDD_TEST1.UT_MESSAGE_API" has errors
ORA-06508: PL/SQL: could not find program unit being called:
"TDD_TEST1.UT_MESSAGE_API"
ORA-06512: at "UTP.UTPLSQL", line 1109
ORA-06510: PL/SQL: unhandled user-defined exception
ORA-06512: at "UTP.UTASSERT2", line 149
ORA-06512: at "UTP.UTASSERT", line 49
ORA-06512: at "UTP.UTPLSQL", line 426
ORA-04063: package body "TDD_TEST1.UT_MESSAGE_API" has errors
ORA-06508: PL/SQL: could not find program unit being called:
"TDD_TEST1.UT_MESSAGE_API"
ORA-06512: at line 1
```

The tests results show failure. But what has really happened?

- No tests were executed
- The tests cannot be executed as the test package is invalid
- There is a stacktrace of errors that allows us to trace how tests are invoked by UTPLSQL but has no value in terms of finding out why there is a problem with test execution

\
This example is clearly illustrating problems and difficulties that can be encountered when using UTPLSQL. If we would have a package containing 10,20,50 methods, then **UTPLSQL will fail to test all those methods just because one of them has changed it's signature**. This single fact makes UTPLSQL a bad option when making choice of unit testing framework. The problem is not related to the way UTPLSQL was build, but is strictly related to the way Oracle handles package dependencies.
There is a way to work around this issue. We can have one test package per one tested method. But that would mean that to test 50 methods in one package you would need to create 50 unit test packages. This is a huge overkill and would create a big mess in the tested database schema, as the number of test packages would dominate the schema source code objects.
Coming up next ... UTPLSQL not failing where it should.
