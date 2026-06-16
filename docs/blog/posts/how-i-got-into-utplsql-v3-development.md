---
title: "How I got into utPLSQL v3 development"
date:
  created: 2017-09-24
slug: how-i-got-into-utplsql-v3-development
categories:
  - "PLSQL"
  - "utPLSQL"
  - "testing"
tags:
  - "Oracle"
  - "utPLSQL"
  - "unit testing"
---

Winter is coming and the 7th season of *Game of Thrones* now just a memory. While I do love watching TV series it was not them that dragged me away from my blog.
 
For last 18 months or so, I was heavily involved in design and development of new version of utPLSQL v3.
After a year of development, we've published utPLSQL version 3.0.0 in May 2017 and now we are at version 3.0.3.

<!-- more -->

#### Why I've decided to get involved in the project?

The reason for getting involved was  that there still was room for a good Unit Testing framework for Oracle SQL and PL/SQL.
I've used [ruby-plsql](https://github.com/rsim/ruby-plsql) a lot and I was very happy with it, however it has one big disadvantage - it's Ruby.
For some projects, adding Ruby language to development stack, just for the sake of Unit Testing is just too much. I still consider the framework a brilliant solution and it's definitely something that utPLSQL v3 is aspiring for.
Using frameworks in languages like Ruby/Java etc. to test database has one significant downside - performance. The fact that every interaction with database is a context switch between Java/Ruby and Oracle database significantly impacts the execution time of tests.
While using ruby-plsql, we had to be very careful when designing our tests to make them run fast.
The performance of tests for Oracle Database created in Ruby/Java will never match performance of the same test written in pure PLSQL.

#### Other frameworks

There is still SQL Dveloper Unit Testing, Quest(Dell) Code Tester for Oracle and more. For those two however, I see definite downsides of:

- Closed architecture
- Tight coupling with vendor UI
- UI limitations
- persistence of tests separated from persistence of code
- no clear concept of integrations with variety of CI/CD solutions
- licensing
- need to learn another tool outside of SQL and PLSQL

#### Why not utPLSQL v2?

There are many challenges around utPLSQL v2, for creating valuable Unit Tests. To name a few its:

- lack of support for many data-types
- noisy reporting
- insufficient integrations
- code in a hard to maintain state

#### What is so special about utPLSQL v3?

Here is a list with some of qualities utPLSQL version 3:

- free and open-source
- open for extensions
- self unit-tested
- continuously integrated in the cloud
- enables test driven development for PLSQL
- clean and consistent reporting
- CI/CD oriented and easy to integrate
- code coverage reporting out of the box
- follows best patterns of Ruby for test expectations
- follows best patterns of Java for annotations
- test suites are built on the fly
- no persistence needed - just install utPLSQL, compile code and tests and you're good to go
- all configuration is optional and defined when invoking tests[how-i-started-to-create-unit-tests-for-oracle-plsql-code.md](how-i-started-to-create-unit-tests-for-oracle-plsql-code.md)

#### Useful links

- [utPLSQL documentation site](http://utplsql.org/documentation/)
- [utPLSQL v3 cheat-sheet](https://www.cheatography.com/jgebal/cheat-sheets/utplsql-v3/)
- [utPLSQL releases](https://github.com/utPLSQL/utPLSQL/releases)
- [utPLSQL demo project sources](https://github.com/utPLSQL/utPLSQL-demo-project)
- [Sonarcloud integration](https://sonarcloud.io/dashboard?id=utPLSQL%3AutPLSQL-demo-project%3Adevelop) with demo project
- [Sonarcloud](https://sonarcloud.io/dashboard?id=utPLSQL%3Adevelop) for utPLSQL v3 itself
- [Builds](https://travis-ci.org/utPLSQL/utPLSQL/branches) for utPLSQL v3 on Travis CI
- Slides from my talk on Test Driven Development with utPLSQL v3 on Ireland OUG meetup in Dublin

#### Getting involved

The team developing utPLSQL v3 is a group of passionate Oracle engineers spending their spare time to deliver well designed, continuously tested, high quality product.
The hope is that,  utPLSQL v3 will bring Unit Testing and Test Driven Development practices into Oracle Developer community and we will finally catchup on engineering practives with Object Oriented programming world.
 
If you want to get involved, read the [contributing guide](https://github.com/utPLSQL/utPLSQL/blob/develop/CONTRIBUTING.md) and join our Slack channel - link available in the [projects readme.](https://github.com/utPLSQL/utPLSQL)
