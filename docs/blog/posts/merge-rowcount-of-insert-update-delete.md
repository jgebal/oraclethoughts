---
title: "Oracle MERGE operation - get number of rows inserted / updated deleted"
date:
  created: 2015-06-13
slug: merge-rowcount-of-insert-update-delete
categories:
  - "SQL"
tags:
  - "Merge"
  - "Oracle"
  - "SQL"
---

[![mergesign](../../images/mergesign-300x200.jpg)](../../images/mergesign.jpg)
Oracle database does not support ability to obtain number of rows inserted/updated/deleted by a merge operation.
The only value you can obtain is the total number of rows affected by merge operation.

<!-- more -->

Consider the following example.
Setup.

```sql
CREATE TABLE emp(id INTEGER PRIMARY KEY, first_name VARCHAR2(50));

INSERT INTO emp
SELECT rownum AS id, 'emp '||rownum AS first_name
  FROM DUAL
CONNECT BY LEVEL <= 50;

COMMIT;
```

Code:

```sql
BEGIN
  MERGE INTO emp dst
    USING (SELECT rownum AS id, 'emp '||rownum AS first_name
             FROM DUAL
          CONNECT BY LEVEL <= 100
         ) src
      ON (src.id = dst.id)
    WHEN MATCHED THEN
      UPDATE
         SET dst.first_name = src.first_name
      DELETE
       WHERE src.id <= 10
    WHEN NOT MATCHED THEN
      INSERT (dst.id, dst.first_name)
      VALUES (src.id, src.first_name);
  DBMS_OUTPUT.PUT_LINE( SQL%ROWCOUNT || ' rows processed.');
  ROLLBACK;
END;
/
```

All that you can get is the overall number of rows processed by merge statement.
So I've created a helper package that will allow counting of rows inserted/updated/deleted by the merge operation.
Sample usages of the package.

```sql
BEGIN
  MERGE INTO emp dst
    USING (SELECT rownum AS id, 'emp '||rownum AS first_name
             FROM DUAL
          CONNECT BY LEVEL <= 100
         ) src
      ON (src.id = dst.id)
    WHEN MATCHED THEN
      UPDATE
         SET dst.first_name = src.first_name
       WHERE merge_row_count.upd() > 0
      DELETE
       WHERE src.id <= 10 AND merge_row_count.del() > 0
    WHEN NOT MATCHED THEN
      INSERT (dst.id, dst.first_name)
      VALUES (src.id, src.first_name)
       WHERE merge_row_count.ins() > 0;
  DBMS_OUTPUT.PUT_LINE( SQL%ROWCOUNT                    || ' rows processed.');
  DBMS_OUTPUT.PUT_LINE( merge_row_count.get_inserted()  || ' rows inserted.');
  DBMS_OUTPUT.PUT_LINE( merge_row_count.get_updated()   || ' rows updated.' );
  DBMS_OUTPUT.PUT_LINE( merge_row_count.get_deleted()   || ' rows deleted.' );
  ROLLBACK;
END;
/
```

In the above example the package function is called from within the MERGE statement one call for each UPDATE, DELETE and INSERT operation is done
For performance reasons it's better to have your merge statements make as little [SQL - PL/SQL context switching as possible](../posts/plsql-performance-freak-series-function-calls-from-sql-overhead.md). You may call the merge operation wit a counter used only on the part that is likely to process less rows.
If your code is suppose to mainly update existing rows and sometimes insert new rows it might be better to use calls only to

```
merge_row_count.ins()
```

```sql
BEGIN
  MERGE INTO emp dst
    USING (SELECT rownum AS id, 'emp '||rownum AS first_name
             FROM DUAL
            CONNECT BY LEVEL <= 100
          ) src
       ON (src.id = dst.id)
    WHEN MATCHED THEN
      UPDATE
         SET dst.first_name = src.first_name
    WHEN NOT MATCHED THEN
      INSERT (id, first_name)
      VALUES (src.id, src.first_name)
       WHERE merge_row_count.ins() > 0;
  DBMS_OUTPUT.PUT_LINE( SQL%ROWCOUNT );
  DBMS_OUTPUT.PUT_LINE( merge_row_count.get_inserted(SQL%ROWCOUNT)  || ' rows inserted.');
  DBMS_OUTPUT.PUT_LINE( merge_row_count.get_updated(SQL%ROWCOUNT)   || ' rows updated.' );
  ROLLBACK;
END;
/
```

If your code is suppose to mainly insert new rows and sometimes update existing rows it might be better to use calls only to

```
merge_row_count.upd()
```

```sql
BEGIN
  MERGE INTO emp dst
    USING (SELECT rownum AS id, 'emp '||rownum AS first_name
             FROM DUAL
            CONNECT BY LEVEL <= 100
          ) src
       ON (src.id = dst.id)
    WHEN MATCHED THEN
      UPDATE
         SET dst.first_name = src.first_name
       WHERE merge_row_count.upd() > 0
    WHEN NOT MATCHED THEN
      INSERT (id, first_name)
      VALUES (src.id, src.first_name);
  DBMS_OUTPUT.PUT_LINE( SQL%ROWCOUNT );
  DBMS_OUTPUT.PUT_LINE( merge_row_count.get_inserted(SQL%ROWCOUNT)  || ' rows inserted.');
  DBMS_OUTPUT.PUT_LINE( merge_row_count.get_updated(SQL%ROWCOUNT)   || ' rows updated.' );
  ROLLBACK;
END;
/
```

The code can be downloaded "as is" from [my github project.](https://github.com/jgebal/merge_row_count)
Feel free to modify according to your own needs or contribute if you like the idea.
