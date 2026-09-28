---
title: "Beware of raise_application_error from within exception when others"
date:
  created: 2016-05-19
slug: beware-of-raise_application_error-from-within-exception-when-others
categories:
  - "PLSQL"
tags:
  - "PL/SQL"
  - "exception handling"
---

Most of use-cases for current Oracle databases involve some kind of application server that is managing database connection pool.
The pool is keeping database connections open for long time to avoid the overhead of connect/disconnect handshakes.
Taking into consideration, that deployment of database changes, is more and more often required to be done seamlessly, with minimum or even zero down time, the changes must be applied in a way that they do not significantly impact the open connections and do not enforce the connection pool recycling.
This is related to the fact that we want to minimize the scale of impact of particular deployment on a working, living environment.

<!-- more -->

### The stateful package

As some of you may know, and rest might just about to find out, some of the database packages can be stateful within a session.
This means, that package itself holds in memory data that is session-specific.
To give simple example, lets have a look at this package.

```sql
create or replace package stateful_package is

   function get_number return number;

   procedure set_number( value number );

end;
/

create or replace package body stateful_package is

   number_value number; --global variable that causes the package to become stateful

   function get_number return number is
   begin
      return stateful_package.number_value;
   end;

   procedure set_number( value number ) is
   begin
      stateful_package.number_value := set_number.value;
   end;

end;
/
```

Let's see what happens when we invoke the procedure and function from the package from two different sessions.

```sql
--SESSION 1
set serveroutput on
begin
  stateful_package.set_number(3);
  dbms_output.put_line( 'stateful_package.get_number() returns ['||stateful_package.get_number()||']' );
end;
/
stateful_package.get_number() returns [3]

PL/SQL procedure successfully completed.

SQL>
```

Statefulness means that the state of the package is held within the session between calls to PLSQL code.
That is the main difference between the function/procedure level variables and package level variables.
Lest see what we get by another call to the get\_number function within session 1.

```sql
--SESSION 1
begin
  dbms_output.put_line( 'stateful_package.get_number() returns ['||stateful_package.get_number()||']' );
end;
/
stateful_package.get_number() returns [3]

PL/SQL procedure successfully completed.

SQL>
```

So what will happen if we call the function get\_number from a separate database session?

```sql
--SESSION 2
set serveroutput on
begin
  dbms_output.put_line( 'stateful_package.get_number() returns ['||stateful_package.get_number()||']' );
end;
/
stateful_package.get_number() returns []

PL/SQL procedure successfully completed.

SQL>
```

We clearly see that the state of package (number\_value = 3) is held only within SESSION 1.

### Fun with production installations

Now the fun part starts, when we would like to deploy a change or recompile a package.

Note
Package recompilation may occur implicitly, if the object that the package is referencing (weather it is a table/view or another package) gets altered or recompiled.

Our deployment process runs in separate session (session 3).
The process causes the package to be recomplied.

```sql
--SESSION 3
alter package stateful_package compile body;

Package body altered.

SQL>
```

 
What has happened to sessions 1 and 2, since they were open before the session 3 kicked in and recompiled the package?

```sql
--SESSION 1
begin
  dbms_output.put_line( 'stateful_package.get_number() returns ['||stateful_package.get_number()||']' );
end;
/

begin
*
ERROR at line 1:
ORA-04068: existing state of packages has been discarded
ORA-04061: existing state of package body "GENERIC_UTIL.STATEFUL_PACKAGE" has
been invalidated
ORA-04065: not executed, altered or dropped package body
"GENERIC_UTIL.STATEFUL_PACKAGE"
ORA-06508: PL/SQL: could not find program unit being called:
"GENERIC_UTIL.STATEFUL_PACKAGE"
ORA-06512: at line 2

SQL>
```

Session state has changed for session 1. Oracle has cleaned the memory area related to the package, as the package was re-compiled.
The ORA-04068 is an exception that is recoverable, this means, that the next time we call the code within session 1, it will still run.
The important part is, that the previously set state is gone.

```sql
--SESSION 1
begin
  dbms_output.put_line( 'stateful_package.get_number() returns ['||stateful_package.get_number()||']' );
end;
/

stateful_package.get_number() returns []

SQL>
```

Exactly the same thing happened to session 2. Function call failed the first time and recovered when called again.

### Exception raising and handling

There are situations, where developer would be tempted to use custom exception to cover one or more exception that can be raised within Oracle.
For example, insert of a row into a table could fail for multiple reasons and listing all possible exceptions that could be raised while doing an insert operation can be countless.
Oracle does not provide exception hierarchies. We just have a very long list of possible exceptions.
Because of that, we can't simply trap exception category, we need to list all the possible exceptions.
I'll use a different example of exception handling that faces the same issue.
When converting from string to date using Oracle TO\_DATE function we can encounter very long list of exceptions.
So most of the suggestions are to surround the TO\_DATE with a block containing the WHEN OTHERS exception handler.
[(Oracle community) how to catch date errors and continue processing in a PL/SQL procedure](https://community.oracle.com/thread/961729?start=0&tstart=0)
[(stackoverflow) What exact exception to be caugth while calling TO\_DATE in pl/sql code](http://stackoverflow.com/questions/20042038/what-exact-exception-to-be-caugth-while-calling-to-date-in-pl-sql-code)
Let's create a package, that will be invoking to\_date conversion and use two approaches.

```sql
create or replace package conversion_package is

   function to_date_supress_error(
      date_string varchar2,
      date_format varchar2  := 'dd-mon-yyyy',
      date_when_error date := to_date('01-01-0001','dd-mm-yyyy')
   ) return date;

   function to_date_overwrite_error(
        char_literal in varchar2,
        date_format  in varchar2
   ) return date;
end;
/

create or replace package body conversion_package is

   function to_date_supress_error(
      date_string varchar2,
      date_format varchar2  := 'dd-mon-yyyy',
      date_when_error date  := to_date('01-01-0001','dd-mm-yyyy')
   ) return date is
   begin
      stateful_package.set_number( coalesce( stateful_package.get_number(), 0 ) + 1 );
      return to_date( to_date_supress_error.date_string, to_date_supress_error.date_format );
   exception
      when others then
         return to_date_supress_error.date_when_error;
   end;

   function to_date_overwrite_error(
        char_literal in varchar2,
        date_format  in varchar2
   ) return date is
   begin
      stateful_package.set_number( coalesce( stateful_package.get_number(), 0 ) + 1 );
      return to_date( to_date_overwrite_error.char_literal, to_date_overwrite_error.date_format );
   exception
      when others then
         raise_application_error( -20001, 'Not a valid date' );
   end;

end;
/
```

Interesting things start to happen, when we either suppress or overwrite oracle stack trace in the when others exception handler, specially when the code that we catch on can raise ORA-04068.

```sql
--SESSION 1
begin
  dbms_output.put_line( conversion_package.to_date_supress_error( '18-05-2016', 'dd-mm-yyyy') );
end;
/
18-MAY-16

PL/SQL procedure successfully completed.

SQL>
```

```sql
--SESSION 2
begin
  dbms_output.put_line( conversion_package.to_date_overwrite_error( '19-05-2016', 'dd-mm-yyyy') );
end;
/
19-MAY-16

PL/SQL procedure successfully completed.

SQL>
```

Let's have a look what happens, when we recompile the stateful\_package and invoke functions from the conversion\_package in sessions 1 and 2.

```sql
--SESSION 3
alter package stateful_package compile body;

Package body altered.

SQL>
```

```sql
--SESSION 1
begin
  dbms_output.put_line( conversion_package.to_date_supress_error( '18-05-2016', 'dd-mm-yyyy') );
end;
/
01-JAN-01

PL/SQL procedure successfully completed.

SQL>
```

```sql
--SESSION 2
begin
  dbms_output.put_line( conversion_package.to_date_overwrite_error( '19-05-2016', 'dd-mm-yyyy') );
end;
/
begin
*
ERROR at line 1:
ORA-20001: Not a valid date
ORA-06512: at "GENERIC_UTIL.CONVERSION_PACKAGE", line 25
ORA-06512: at line 2

SQL>
```

Session 1 has suppressed the ORA-04068 exception and Session 2 has overwritten it with custom application error ORA-20001.
**The worrying part however, is that this time the exception is unrecoverable, no matter how many times we try to execute the functions.**

### End notes

This behaviour is not often observable, but one needs to be very careful when dealing with stateful packages and exception handlers.
My personal preference would be to have a exception class hierarchy in Oracle so we could safely trap particular groups of exceptions.
Without that, developers need to use the absolutely too generic "OTHERS" exception, that is too broad to be safely used in day to day programming.
The only valid real-life implementation of when others exception handler I see is similar to this one

```sql
begin
   --the code that does stuff
exception
   when others then
     --do the transaction handling (rollback?)
     --close open cursor(s)
     --do the logging of exception
     RAISE; -- mandatory re-propagation of original exception
end;
```
