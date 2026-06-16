---
title: "SQLPlus ERRORLOGGING issues with error log on rollback resolved"
date:
  created: 2014-01-30
slug: sqlplus-errorlogging-issues-with-error-log-on-rollback-resolved
categories:
  - "SQL*Plus"
tags:
  - "error logging"
  - "rollback"
  - "sqlplus"
  - "SQL"
  - "exception handling"
---

Today I came up with idea to overcome the issue: [SQLPlus ERRORLOGGING does not keep error log on rollback](../posts/sqlplus-errorlogging-does-not-keep-error-log-on-rollback.md "SQLPlus ERRORLOGGING does not keep error log on rollback").
The resolution is to use autonomous transactions to log the errors reported by SQL Plus.
What we need to do is to somehow catch the error that is about to be logged and wrap it in an autonomous transaction.

<!-- more -->

The features that we will use are:

- a VIEW
- a ["INSTEAD OF" TRIGGER](http://docs.oracle.com/cd/E11882_01/appdev.112/e25519/triggers.htm#LNPLS20041)
- a procedure that works in an [autonomous transaction](http://docs.oracle.com/cd/B28359_01/server.111/b28318/transact.htm#CNCPT417)

Create the error logging table.

```sql
CREATE TABLE sperrorlog(
  username VARCHAR(256),
  timestamp TIMESTAMP,
  script VARCHAR(1024),
  identifier VARCHAR(256),
  message CLOB,
  statement CLOB
);
```

Run a script to prove the ROLLBACK statement will cause the errors disappear from the error log.

```sql
SET ERRORLOGGING ON
INSERT INTO bad_table(a) VALUES(1);
SET ERRORLOGGING OFF
ROLLBACK;
SELECT timestamp, script, substr(message,1,100) message
  FROM sperrorlog;
```

```
SQL*Plus: Release 11.2.0.2.0 Production on Wed Jan 1 23:22:20 2014

Copyright (c) 1982, 2011, Oracle.  All rights reserved.

Use "connect username/password@XE" to connect to the database.
SQL> conn hr/hr 
Connected.
SQL> truncate table sperrorlog;

Table truncated.

SQL> SET ERRORLOGGING ON
SQL> INSERT INTO bad_table(a) VALUES(1);
INSERT INTO bad_table(a) VALUES(1)
            *
ERROR at line 1:
ORA-00942: table or view does not exist

SQL> SET ERRORLOGGING OFF
SQL> ROLLBACK;

Rollback complete.

SQL> SELECT timestamp, script, substr(message,1,100) message FROM sperrorlog;

no rows selected

SQL>
```

The error log table is empty.
A little cheat will do the trick.

```sql
ALTER TABLE sperrorlog RENAME TO t_sperrorlog;

CREATE VIEW sperrorlog AS SELECT * FROM t_sperrorlog;

CREATE OR REPLACE PROCEDURE sperrorlog_prc(
  username VARCHAR, timestamp TIMESTAMP, script VARCHAR,
  identifier VARCHAR, message CLOB, statement CLOB
) IS
 PRAGMA AUTONOMOUS_TRANSACTION;
BEGIN
  INSERT INTO t_sperrorlog
  VALUES (username, timestamp, script, identifier, message, statement);
  COMMIT;
END;
/

CREATE OR REPLACE TRIGGER sperrorlog_trg
  INSTEAD OF INSERT ON sperrorlog FOR EACH ROW
 CALL sperrorlog_prc(
   :NEW.username,:NEW.timestamp, :NEW.script,
   :NEW.identifier, :NEW.message, :NEW.statement);
/
```

Run the same script and see the results.

```sql
SET ERRORLOGGING ON
INSERT INTO bad_table(a) VALUES(1);
SET ERRORLOGGING OFF
ROLLBACK;
SELECT timestamp, script, substr(message,1,100) message
  FROM sperrorlog;
```

```
SQL*Plus: Release 11.2.0.2.0 Production on Wed Jan 1 23:22:20 2014

Copyright (c) 1982, 2011, Oracle.  All rights reserved.

Use "connect username/password@XE" to connect to the database.
SQL> conn hr/hr 
Connected.
SQL> truncate table sperrorlog;

Table truncated.

SQL> SET ERRORLOGGING ON
SQL> INSERT INTO bad_table(a) VALUES(1);
INSERT INTO bad_table(a) VALUES(1)
            *
ERROR at line 1:
ORA-00942: table or view does not exist

SQL> SET ERRORLOGGING OFF
SQL> ROLLBACK;

Rollback complete.

SQL> SELECT timestamp, script, substr(message,1,100) message FROM sperrorlog;

TIMESTAMP
---------------------------------------------------------------------------
SCRIPT
--------------------------------------------------------------------------------
MESSAGE
--------------------------------------------------------------------------------
30-JAN-14 01.44.27.000000 PM

ORA-00942: table or view does not exist
SQL>
```

Logging is now done into SPERRORLOG VIEW via trigger, that executes insert into T\_SPERRORLOG table in autonomous transaction.
Even if the transaction is rolled back, the error logs are persisted.
