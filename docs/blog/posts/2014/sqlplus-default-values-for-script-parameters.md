---
title: "SQL*Plus default values for script parameters"
date:
  created: 2014-01-22
slug: sqlplus-default-values-for-script-parameters
categories:
  - "SQL*Plus"
tags:
  - "define default"
  - "sqlplus"
  - "SQL"
---

While working heavily on script automation I came across the issue of having an SQLPlus script with optional parameter. Oracle does not allow that by default, and one needs to use a dirty trick to make it work. Vladimir made a great article on how to achieve that. Thank you Vlad, this is really helpful.
[Vladimir's Diary: On SQL\*Plus Defines](http://vbegun.blogspot.nl/2008/04/on-sqlplus-defines.html).

<!-- more -->

I'll just add a use case, as there is not much more to add.
I have a script that i'd like to run on the connected user's credentials and by default in the connected users schema.
I'd also like it to accept a user's schema name, to execute in as a parameter.
To achieve this i'll use the Vladimir's solution

```sql
SET ECHO OFF
SET FEEDBACK OFF
SET HEADING OFF
SET PAGESIZE 0
SET VERIFY OFF
COLUMN 1 NEW_VALUE 1
SELECT '' "1" FROM DUAL WHERE ROWNUM = 0;
REM Just to use more meaningfull variable, i will give it a name

DEF WORK_SCHEMA_NAME='&1'

BEGIN
  IF '&&WORK_SCHEMA_NAME' IS NOT NULL THEN
    EXECUTE IMMEDIATE 'ALTER SESSION SET CURRENT_SCHEMA=&&WORK_SCHEMA_NAME';
  END IF;
END;
/

SELECT 'Now i''m working on schema: '
       || SYS_CONTEXT('USERENV','CURRENT_SCHEMA')
       || ' as user: '|| USER
  FROM DUAL;

PROMPT I can do my processing here
```
