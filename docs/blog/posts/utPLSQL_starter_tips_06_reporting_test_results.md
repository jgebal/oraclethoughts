---
title: "utPLSQL starter tips - reporting test results"
date:
  created: 2026-08-19
slug: utPLSQL_starter_tips_reporting_test_results
categories:
  - "PLSQL"
  - "utPLSQL"
  - "testing"
tags:
  - "utPLSQL"
  - "unit testing"
  - "ci-cd"
  - "sonarsource"
---

# Reporting test results with utPLSQL

utPLSQL comes with several output reporters for different use cases: human-readable text, JUnit XML, TeamCity, TFS for CI pipelines,
SonarQube-compatible coverage reports and more.

<!-- more -->

Reporters control the format of test output. The `ut.run()` command only allows executing tests with one reporter attached,
while `utplsql-cli`, `utplsql-maven-plugin` and SQL Developer/PLSQL Developer extensions allow for multiple reporters to be attached to a single run.

Multiple reporters can be used simultaneously to save results in different formats from a single test run.
Multi-reporting is most commonly used for simultaneous reporting of realtime test execution progress to console, saving test results into a JUnit XML file and generating coverage report data to be used in CI/CD.

```plsql
-- Documentation reporter (default)
begin
  ut.run();
end;
/

-- JUnit XML for CI pipelines
begin
  ut.run(a_reporter => ut_junit_reporter());
end;
/
```

## Available reporters

utPLSQL comes equipped with several reporters

**`ut_documentation_reporter`**
Creates a textual, pretty-print, human-readable report mirroring the suite hierarchy. Used for console runs, interactive development and log review.

**`ut_realtime_reporter`**

Provides live test execution progress that can be consumed from another session, enabling clients such as SQL Developer to show progress in real time as tests run.

**`ut_teamcity_reporter`**

Provides [TeamCity](https://confluence.jetbrains.com/display/TCD9/Build+Script+Interaction+with+TeamCity)
reporting-format that allows tracking of progress of a CI step/task as it executes.

**`ut_junit_reporter`**
JUnit XML format. Compatible with GitHub Actions, Jenkins, Azure Pipelines, GitLab CI, and any tool that reads JUnit XML test results.
Provides outcomes in a format conforming with JUnit 4 as defined [here](https://gist.github.com/kuzuha/232902acab1344d6b578)

**`ut_sonar_test_reporter`**

Generates an XML report providing detailed information on test execution.
Designed for [SonarQube](https://www.sonarsource.com/products/sonarqube/) to report test execution.
XML format returned conforms with the [Sonar specification](https://docs.sonarsource.com/sonarqube-server/analyzing-source-code/test-coverage/generic-test-data)

**`ut_coverage_sonar_reporter`**

Generates an XML coverage report providing information on code coverage with line numbers.
Designed for [SonarQube](https://www.sonarsource.com/products/sonarqube/) to report coverage.
XML format returned conforms with the [Sonar specification](https://docs.sonarsource.com/sonarqube-server/analyzing-source-code/test-coverage/generic-test-data).


**`ut_coverage_html_reporter`**

Generates HTML coverage report with summary and line by line information on code coverage.
Based on open-source simplecov-html coverage reporter for Ruby.
Includes source code in the report.


**`ut_coverage_cobertura_reporter`**

Generates Cobertura report on code coverage with line numbers.
Compatible with GitHub Actions, Jenkins, Azure Pipelines, GitLab CI, and any tool that reads the popular Cobertura format.
Cobertura Document Type Definition is located [here](https://github.com/cobertura/cobertura/blob/master/cobertura/src/site/htdocs/xml/coverage-04.dtd).


**`ut_tfs_junit_reporter`**

Provides outcomes in a format conforming with JUnit version for TFS / VSTS
as defined by [specification](https://learn.microsoft.com/en-us/azure/devops/pipelines/tasks/reference/publish-test-results-v2?view=azure-pipelines)
The implementation is based on [windyroad junit schema](https://github.com/windyroad/JUnit-Schema/blob/master/JUnit.xsd).


## Using reporters with utPLSQL-cli

When running tests from the command line, reporters are specified with the `-f` flags.
Each `-f` can be followed by a `-o` flag indicating the output filename.

```bash
utplsql run user/pass@//host:1521/SERVICE \
  -f=ut_junit_reporter \
  -o=test-results/junit.xml \
  -f=ut_documentation_reporter
```

Multiple `-f` flags produce multiple output formats from a single database test run.

## Further reading

- [Reporters reference](https://utplsql.org/utPLSQL/latest/userguide/reporters)
- [Code coverage reference](https://www.utplsql.org/utPLSQL/latest/userguide/coverage)
