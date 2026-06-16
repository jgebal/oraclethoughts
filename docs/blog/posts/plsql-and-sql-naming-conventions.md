---
title: "PL/SQL and SQL naming conventions"
date:
  created: 2016-04-29
slug: plsql-and-sql-naming-conventions
categories:
  - "PLSQL"
  - "SQL"
tags:
  - "PL/SQL"
  - "naming"
---

Some time back I've read [Mike Smithers Blog on SQL and PL/SQL standards](https://mikesmithers.wordpress.com/2011/10/22/oracle-sql-and-plsql-coding-standards-%E2%80%93-cat-herding-for-dummies/). I really like reading his blog. He is a great story-teller.
Being Oracle developer for over 15 years should make me comply with all of the mostly demanding standards there are. My nature however always tells me to look at the balance the costs and benefits of my actions.
Mike pointed out a good amount of issues, when it comes to introducing coding standards and how important it is to keep it simple.
Naming conventions can also become a true bottleneck and make the database structures and code change-resistant.

<!-- more -->
Consider such simple convention for naming database objects.

```
Table            -> _TBL
View             -> _VW
Synonym          -> _SYN
Package          -> _PCK
Function         -> FUN_
Procedure        -> PRC_
Input Parameter  -> PI_
Output Parameter -> PO_
Variable         -> V_
Constant         -> CON_
Exception        -> EXC_
Object Type      -> TYP_
Collection Type  -> COL_
Record type      -> REC_
PLSQL table type -> TAB_
```

**What is the real reason behind having such standards?**

Such naming conventions allow us to have multiple objects that represent the same thing within one namespace (one database schema) and still be able to distinguish them.
So we can have:

```sql
  table customer_tbl
  view customers_vw
  synonym customers_syn
  package customers_pck
```

Such conventions were first established when databases themself were born, when developers were using plain text editors with no syntax highlighting.
The conventions were to ease developers work and provide within the object name, the information about object type. There is no other easy way of doing that when you're doomed to use plain text editor.
Currently we have many great IDE's that allow us to navigate and inspect database objects and code with single key stroke or mouse click so this part is no longer true, so this reasoning is no longer true.
The side effect of those standards is degradation of program readability as it becomes more of an encoded "code" not a text that is easy to read.

```sql
  function fun_get_cust_nm( pi_cust_id customers_tbl.cust_id%type ) return cust_tbl.cust_nm%type IS
    v_cust_nm cust_tbl.cust_nm%type;
  begin
    select cust_nm
      into v_cust_nm
      from cust_tbl
     where cust_id = pi_cust_id;
    
    return v_cust_nm;
  end;
```

```sql
  ...
  v_cust_nm := fun_get_cust_nm( v_cust_id );
  ...
```

**Giving the name a meaning**
Many modern naming paradigms concentrate on giving elements names that represent their purpose, function, behaviour so that the program code becomes more readable and gets closer to plain spoken language.
For me, one of the most difficult things in programming nowadays is giving things the right names.
That itself is a craft and a real craftsman pays attention to those little details that make the code more readable and maintainable.
**The prefixing/suffixing dilemma**
Would it be good prefix/suffix everything?
We might be proud at beginning once we name all views with \_VW, all synonyms with \_SYN and tables with \_TBL.
After some time we will realize however, that our code became quite resistant to changes.
We no longer can change the table to a view or a synonym as table is a \_TBL, and that suffix is spread around many places in our code.
There is a huge hidden cost connected with hard-coding the object type in its name. The object type cannot be changed without a need to change the name in each place that is referencing the object.
The change will also increases the risk for failure, as not all dependencies are easy to track.
An easy workaround to that might seem to be another standard that would say something like:
"No table can be referenced directly, but only through a view" or "No table can be referenced directly, but only through a synonym" and so on.
That however will increase development cost by bringing burden of creating and maintaining two or three database objects, where only one would suffice.
Both approaches make code resistant to change and by introducing a hard skeleton of either name standards or additional object structures.
Instead of allowing our code to be change-enabled, we make it change-resistant by applying such standards.
So what would happen if we would have tables/views/synonyms/users/packages etc. without type encoded into the object name?
Wouldn't it bring a total chaos? How would we know what is what?
We can name things in the with following pattern

- Database elements that represent data should be named by what they hold or represent (employees/customers etc.)
- Database elements that hold program code should be named by what they do (raise\_salary/get\_customer\_name/discount\_calculator)

We can also benefit from namespaces in Oracle (schemas).
We often tend to forget, that schema is a great way of isolating things. It is a separate namespace.
It allows very precise privileges management and we often try to push all we can into single schema.
Instead of having

```sql
table HR.EMPLOYEES_TAB 
view HR.EMPLOYEES_VW 
package HR.EMPLOYEE_SALARY_CALCULATOR_PCK
```

We could have

```sql
HR_DATA.EMPLOYEES
HR_API.EMPLOYEES
HR_CODE.EMPLOYEE_SALARY_CALCULATOR
```

Minimizing the number of abbreviated prefixes and suffixes in code makes it look much more like a human readable language.

```sql
  function get_customer_name( customer_id number) return customers.customer_name%type IS
    customer_name customers.customer_name%type;
  begin
    select c.customer_name
      into get_customer_name.customer_name
      from customers c
    where c.customer_id = get_customer_name.customer_id;
    
    return customer_name;
  end;
```

```sql
  ...
  customer_name := get_customer_name( custmer_id );
  ...
```
