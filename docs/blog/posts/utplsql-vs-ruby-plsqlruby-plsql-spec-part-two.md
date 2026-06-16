---
title: "utPLSQL v2 vs. ruby-plsql/ruby-plsql-spec - part two (setup and basic reporting)"
date:
  created: 2015-06-24
slug: utplsql-v2-vs-ruby-plsqlruby-plsql-spec-part-two
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

In my [previous post](../posts/utplsql-vs-ruby-plsqlruby-plsql-spec-part-one.md) I have described the conceptual differences between UTPLSQL and ruby-plsql frameworks for unit testing of Oracle database code.
I have used a message\_api package and unit tests for that API using both frameworks as an example.
In this post I will focus on getting the tests to run and the feedback that we can we get from the tests using both frameworks.

<!-- more -->

# Before running tests

Before we can actually run the tests, we need both frameworks to be downloaded installed and set up.
I will not describe in details all the steps needed to get them configured but rather give some bullet points on what is required.
To have UTPLSQL installed and ready to use you will need to:

- have oracle client installed locally
- have some Oracle database accessible
- have SYS or SYSDBA privileges on that database (or have someone who has)
- download the [sources](http://sourceforge.net/projects/utplsql/files/utPLSQL/)
- create database user for UTPLSQL framework (recommended)
- grant the user proper privileges - [documentation](http://utplsql.sourceforge.net/Doc/fourstep.html) is not up to date, as user needs access to dbms\_pipe and utl\_file for installation to go smooth (requires connecting as SYSDBA or just SYS)
- install the UTPLSQL library packages, tables, synonyms etc.
- preferably have some Oracle client tool like [SQL Developer (free)](http://www.oracle.com/technetwork/developer-tools/sql-developer/downloads/index.html) or [PLSQL Developer](http://www.allroundautomations.com/plsqldev.html?gclid=CI-gxoqcpMYCFSSc2wodH2UAIw) (paid) or [TOAD](http://software.dell.com/products/toad-for-oracle/) (expensive) or other, to ease working with PLSQL code

To have ruby-plsql-spec installed you will need to:

- have Oracle client installed locally
- have have some Oracle database accessible
- download and install Ruby in version corresponding to Oracle client version (32 bits or 64 bits). This is crucial for ruby-oci8 library to work.
- download and install Ruby Developer Kit - this is needed to install ruby-oci8 library
- install ruby-plsql-spec library by issuing command: `gem install ruby-plsql-spec`
- install ruby-oci8 library and dependencies by issuing command: `gem install ruby-oci8`
- preferably have some Oracle client tool for working with PLSQL code
- preferably have some editor for working with RSpec files. To start simple [Notepad++](https://notepad-plus-plus.org/) will do. I started with Notepad++, then I used [RedCar](http://redcareditor.com/) (opensource but was unstable when i used it) and finally switched to paid [RubyMine by ItelliJ](https://www.jetbrains.com/ruby/), that speeded up things remarkably and improved the quality of tests. In fact the tool is so good that I often use it for PL/SQL and SQL code, due to the incredibly powerful code editor.

\
It's really good to keep in mind entire life-cycle of a project from the very beginning and try to keep the project structure simple and tidy.

# Organizing your workplace

Once you have framework(s) installed, you will need a place on your drive to store all the project files. It's also good to have some version control system in place, even when you're working on your own. I use free GIT repositories available in the cloud ([github](https://github.com/) or [bitbucket](https://bitbucket.org/))
It's noting, that unit tests should not coexist with your production code. Separate folder for tests will make your life easier and simplify automation of your project deployment.
Pattern of tests separation is applied by default with the RSpec unit testing framework. Even though you can have your tests placed anywhere you like, by default the RSpec is looking for tests in the "spec" folder of your project directory. The same goes to RSpec unit test file naming convention. Even though you can execute any file as a RSpec unit test, by default and by definition, the RSpec unit tests have a suffix "\_spec" and extension ".rb". All your RSpec unit tests should take a form of "TestSomething\_spec.rb" or "test\_something\_spec.rb". the first form is more Ruby'ish, the second is more Oracle'ish (as Oracle code is by default case insensitive) so the [snake\_case](https://en.wikipedia.org/wiki/Snake_case) naming convention is the preferred one for Oracle.
Similarly, UTPLSQL has a convention for naming the unit test packages. Though you can name your test packages however you like, by default they should be named with "UT\_" prefix. UTPLSQL does not state any standard for project files naming or location, as it is database-centric. I practice storing UTPLSQL unit tests in separate folder and since specification and body of package are separate database objects and are compiled independently, I have them stored in separate files. I use ".pks" and ".pkb" extensions to identify those, but there are few standards.
The folder structure of the project used for the purpose of this blog series will evolve into structure like below.
\

```
install/ - stores install scripts for setup of database users, and UTPLSQL library
spec/ - stores RSpec unit tests
  helpers/ - a placeholder helper libraries (common code) that can be re-used by unit tests
  message_api/ - organizing unit tests into folders by package names is quite useful when number of tests grow
    send_and_receive_spec.rb - one spec per one function/procedure or one functionality tested
tested_code/ - stores the PL/SQL code being tested (the production code)
  message_api.pks
  message_api.pkb
unit_tests/ - stores UTPLSQL unit tests packages
  ut_message_api.pks
  ut_message_api.pkb
```

ruby-plsql-spec comes with a command for initializing unit testing folder structure. All that needs to be done is to execute command `plsql-spec init` from within the project directory. The command creates the spec folder and initializes it with required files (`spec_helper.rb, database.yml` and some helper files).
The `database.yml` file is a [YAML](https://en.wikipedia.org/?title=YAML) file that provides database connection details for the tested database(s).
`database.yml` used for the purpose of those blog posts looks like this.

```ruby
default:
  username: tdd_test1
  password: tdd_test1
  database: xe
  # host: localhost
  # port: 1521
# Add other connection if needed.
# You can access them with plsql(:other) where :other is connection name specified below.
# other:
#   username: scott
#   password: tiger
#   database: xe

secondary:
  username: tdd_test1
  password: tdd_test1
  database: xe

different_user:
  username: tdd_test2
  password: tdd_test2
  database: xe
```

# Running tests with UTPLSQL

The UTPLSQL unit tests will not compile if there is no package to be tested, and UTPLSQL will not execute the tests if they do not compile.
That means that the code to be tested needs to be in place (at least it's complete specification) before we can execute a test.
That leads us to straight conclusion, that UTPLSQL **is not** fully supporting [Test Driven Development](http://gaboesquivel.com/blog/2014/differences-between-tdd-atdd-and-bdd/).
ruby-plsql-spec will actually execute the tests without the tested code living in the database. RSpec is designed for Behaviour Driven Development (BDD), which is shares the similar concept to TDD, which is: write test for requirement ,validate the requirement is not met, then implement the code.
Our UTPLSQL unit tests need to be compiled into the database.

```
>cd unit_tests
>sqlplus tdd_test1/tdd_test1@xe

SQL*Plus: Release 11.2.0.4.0 Production on Tue Jun 23 21:23:18 2015

Copyright (c) 1982, 2013, Oracle.  All rights reserved.

Connected to:
Oracle Database 11g Express Edition Release 11.2.0.2.0 - 64bit Production

SQL> @ut_message_api.pks

Package created.

SQL> @ut_message_api.pkb

Package body created.

SQL>
```

We can now execute the unit tests.

```
SQL> BEGIN
  2    utplsql.test('message_api');
  3  END;
  4  /

PL/SQL procedure successfully completed.
```

So where are the test results?
We need to make sure to enable DBMS OUTPUT, to be able to see the results of the tests, as by default UTPLSQL reports through DBMS\_OUTPUT.

```
SQL> SET SERVEROUTPUT ON 
SQL> BEGIN
  2    utplsql.test('message_api');
  3  END;
  4  /

Error UT-300005: Compile error: you must specify a directory with
utConfig.setdir!
Error compiling ut_message_api.pks located in "": ORA-20000: UT-300005: Compile
error: you must specify a directory with utConfig.setdir!
Please make sure the directory for utPLSQL is set by calling utConfig.setdir.
Your test package must reside in this directory.
Error UT-300005: Compile error: you must specify a directory with
utConfig.setdir!
Error compiling ut_message_api.pkb located in "": ORA-20000: UT-300005: Compile
error: you must specify a directory with utConfig.setdir!
Please make sure the directory for utPLSQL is set by calling utConfig.setdir.
Your test package must reside in this directory.
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

One thing worth noting is that even though we see a huge SUCCESS, there are some errors due to the fact that UTPLSQL tried to recompile the unit test packages before the testrun and it failed.
According to the documentation we need to disable this feature, that is enabled by default by calling `utConfig.autocompile(false);`
The problem is, that the configuration of UTPLSQL is stored on the database user level, not on the project level.
So if you want to have some settings for entire project that is using several users, you need to either setup those independently or setup one user and copy it to other users.
Unsolvable problem seems to be a case, where several projects would like to use the same user with different settings for unit testing.
Another thing to note is that **the report is covering only one test case for one procedure**. The amount of information and the formatting of the output makes the report really hard to read.
Another issue is that UTPLSQL is using DBMS\_OUTPUT to display the outcomes of tests. Well it's not really the problem of UTPLSQL itself, just that Oracle gives DBMS OUTPUT as the only stream for providing feedback to the calling client (like SQLPlus).
If the tested code, for some reason, is also using DBMS\_OUTPUT, then the feedback from tests will be mixed with output from the code itself.
UTPLSQL according to documentation comes along with 2 more build-in output reporters:
- HTML reporter
- FILE reporter
Both of those reporters do not use DBMS OUTPUT, but write test results as files to the **file system of the database server**. I don't know how strict policies for access to the database server instance file system are used in your company, but in almost all projects that I had opportunity to work on, the database servers (even development), were strongly protected from accessing the file system by database developers. This trend is growing. Regardless of the fact, I'm very interested in the capabilities of the HTML reporter so I've decided to give it a try.
We need to alter our database system to be able to write files to a specified directory and restart our database!
OK that is a one time operation, but still it's a bit much to set up unit testing reporting.

```
SQL> alter system set utl_file_dir='/tmp' scope = spfile;

System altered.

SQL>
```

Once that is done we need to change the reporter for the user that is used for unit testing. It's not easy to get through all the options and settings, but let's get this done.

```
SQL> BEGIN
  2    utConfig.setreporter('HTML');
  4    utConfig.setFileDir('/tmp');
  3  END;
  4  /
```

And run the tests again.

```
SQL> SET SERVEROUTPUT ON 
SQL> BEGIN
  2    utplsql.test('message_api');
  3  END;
  4  /

PL/SQL procedure successfully completed.

SQL>
```

So the procedure completed successfully without giving feedback on failure or success. There is no feedback on the file being generated to the database filesystem too.
On the database server in a new file was created `/tmp/20150623223711.html`

# Running tests with ruby-plsql-spec

With ruby-plsql-spec we have several options to run tests from the main project directory.

- use `plsql-spec run` command
- use `rspec` command

Both `plsql-spec` and `RSpec` commands have many options and I will not cover all of them here, but I will focus on the most interesting ones.
the main difference in the way tests are executed by RSpec compared to UTPLSQL is that Rspec, by default, executes all the tests in the project. That can be altered easily. You can point to a specific test file, specific test in a file or use tags on tests and then call all tests with specific tags.

```
>plsql-spec run
Running all specs from spec/
...

Finished in 0.07601 seconds (files took 1.36 seconds to load)
3 examples, 0 failures
```

By default we get the most minimalistic form of report, showing that there were three tests executed. Each dot represents one successful test.
We get information about time taken to execute the tests and number of failures.
If we want to get an html report, we change the call syntax for the project.

```
>plsql-spec run --html
Running all specs from spec/

Test results in test-results.html
```

The output informs us about the location of out test results. The are located in file `test-results.html` inside our main project directory.
Please have a look at the plain text and the html output from both frameworks.
Before finishing this part I think it's worth having a look at the code coverage reporting functionality included in ruby-plsql-spec.

```
>plsql-spec run --coverage
Running all specs from spec/
...

Finished in 0.07601 seconds (files took 1.38 seconds to load)
3 examples, 0 failures

Coverage report in coverage/index.html
```

With one simple command we have all our tests executed and code coverage report generated into [`coverage/index.html`](../../images/index.html) file.
You may download the [coverage report](../../images/coverage.zip) and see it on your local machine.
Coming up next
[- more reporting options
- reporting failing tests
- are the tests failing/passing when the should](../posts/utplsql-vs-ruby-plsqlruby-plsql-spec-part-three.md)
