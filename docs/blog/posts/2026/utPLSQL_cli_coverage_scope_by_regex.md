---
title: "Defining code coverage scope with regular expressions in utPLSQL-cli"
date:
  created: 2026-10-18
slug: utPLSQL_cli_coverage_scope_by_regex
categories:
  - "PLSQL"
  - "utPLSQL"
  - "testing"
tags:
  - "utPLSQL"
  - "code coverage"
  - "ci-cd"
  - "utPLSQL-cli"
  - "maven"
---

# Defining code coverage scope with regular expressions in utPLSQL-cli

A coverage report is only useful when the information it provides is meaningful.

Useful information would be: "How much of *MY application code* is covered by tests?"

Until now, obtaining precise coverage for your project with `utplsql-cli` was only possible by mapping project
files into database objects or by providing a complete list of objects to include or exclude when gathering coverage.

Since version 3.2.2, `utplsql-cli` lets you describe the scope of the coverage report with regular expressions,
so the included/excluded scope follows your naming conventions instead of an object list.

<!-- more -->

## Project files define the scope by default

When your code is kept in files, let utPLSQL map the project files to database objects. This is the preferred way of gathering coverage.
The report contains only the objects that were mapped to files from your project, so the scope is exactly your source code.

```bash
utplsql run user/pass@//dbhost:1521/service -p=app \
  -f=ut_coverage_sonar_reporter -o=coverage.xml \
  -source_path=src/main/plsql \
  -test_path=src/test/plsql
```

Nothing else is needed to narrow the report. New file in the project means new object in the report.

When project files are mapped, utPLSQL ignores all include/exclude filters, including the new regular expression filters.

## When project files cannot be mapped

Not every project can be mapped to files. Typical cases are:

- the code lives in the database and is not kept in a repository, or not in a layout that can be mapped to object names,
- sources and tests are kept in different repositories than the pipeline that runs the tests,
- the code is generated, or deployed from third-party packages.

In such a case coverage is gathered on database schemas. By default, that is every object in every schema that the tests are run on.

Until now the only ways to narrow it down were `--coverage-schemes` (a list of schemas)
and `-include` / `-exclude` (lists of `[schema.]object` names).
Lists work well for a small project. In a large project, adding each new package to the include or exclude list becomes a headache.

## Regular expression options

| Option                  | Meaning                                                           |
|-------------------------|-------------------------------------------------------------------|
| `-include-schema-expr`  | report coverage only on schemas whose name matches the expression |
| `-exclude-schema-expr`  | leave out schemas whose name matches the expression               |
| `-include-object-expr`  | report coverage only on objects whose name matches the expression |
| `-exclude-object-expr`  | leave out objects whose name matches the expression               |

The filters have been a feature of the utPLSQL framework for quite some time (parameters of ut.run). 
`utplsql-cli` version 3.2.2 is the first version that exposes them as run command options.

The `utplsql-maven-plugin` also has them available as `includeSchemaExpr`, `excludeSchemaExpr`, `includeObjectExpr` and `excludeObjectExpr`.

The rules:

- The expression is an Oracle regular expression, matched with `regexp_like` and the case-insensitive flag. There are no lookaheads.
- A schema expression is matched against the schema name. An object expression is matched against the object name **without** the schema.
- Expressions are not anchored. `-include-object-expr=ord` matches `ORDERS`, `RECORD_PKG` and `BORDER`. Use `^` and `$` to anchor the expression at the beginning or end.
- When an *include* expression is given, `--coverage-schemes` and `-include` are ignored.
  If both include expressions are given, an object has to match both.
- Include and exclude expressions can be combined. Both exclude expressions (schema and object) are applied.
  When any exclude expression is given, the `-exclude` list is ignored.

!!! note
    Characters like `^`, `$`, `|` and `(` mean something to your shell, so quote the expressions.
    Use single quotes in Bash and double quotes in Windows `cmd.exe`.

## Examples

Take a system split into `APP_ORDERS`, `APP_CUSTOMERS` and `APP_BILLING`, with tests kept in `APP_ORDERS_TEST`, `APP_CUSTOMERS_TEST` and `APP_BILLING_TEST`.

Report on all application schemas, but not on the test schemas:

```bash
utplsql run user/pass@//dbhost:1521/service \
  -p=app_orders_test,app_customers_test,app_billing_test \
  -f=ut_coverage_html_reporter -o=coverage.html \
  -include-schema-expr='^APP_' \
  -exclude-schema-expr='_TEST$'
```

When the team adds `APP_SHIPPING` next quarter, it shows up in the report the day it is created.
Notice that the report covers schemas that are not mentioned in `-p` at all.

Keep generated and temporary objects out of the report:

```bash
utplsql run user/pass@//dbhost:1521/service -p=app_orders_test \
  -f=ut_coverage_html_reporter -o=coverage.html \
  -exclude-object-expr='(_TMP|_GEN|_BAK)$'
```

Without an include expression the report covers the schemas that the tests are run on, minus everything that matches.

Cover one functional area, identified by a naming convention:

```bash
utplsql run user/pass@//dbhost:1521/service -p=app_orders_test \
  -f=ut_coverage_html_reporter -o=coverage.html \
  -include-schema-expr='^APP_ORDERS$' \
  -include-object-expr='^(ord|invoice)_'
```

Add the schema expression whenever you use an include object expression.
On its own the object expression is not limited to the schemas of your tests. It matches objects in any schema the database user can see.

## Maven

In `utplsql-maven-plugin` the same rules are configuration parameters.

```xml
<includeSchemaExpr>^APP_</includeSchemaExpr>
<excludeSchemaExpr>_TEST$</excludeSchemaExpr>
<excludeObjectExpr>(_TMP|_GEN)$</excludeObjectExpr>
```

The plugin maps project files by default (`src/main/plsql` and `src/test/plsql`), so the expressions have an effect only
when the project has no files to map.

## Further reading

- [Filtering objects in coverage reports](https://www.utplsql.org/utPLSQL/latest/userguide/coverage.html#filtering-objects-in-coverage-reports)
- [Project based coverage](https://www.utplsql.org/utPLSQL/latest/userguide/coverage.html#project-based-coverage)
- [utPLSQL-cli](https://www.utplsql.org/utPLSQL-cli.html)
- [utPLSQL-maven-plugin](https://www.utplsql.org/utPLSQL-maven-plugin.html)
