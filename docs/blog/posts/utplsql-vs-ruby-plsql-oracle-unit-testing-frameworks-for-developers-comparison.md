---
title: "utPLSQL v2 vs. ruby-plsql – Oracle unit testing frameworks for developers comparison"
date:
  created: 2015-08-19
slug: utplsql-v2-vs-ruby-plsql-oracle-unit-testing-frameworks-for-developers-comparison
categories:
  - "ruby-plsql"
  - "testing"
tags:
  - "PL/SQL"
  - "TDD"
  - "utPLSQL v2"
  - "ruby-plsql"
  - "unit testing"
---

[![UTPLSQL_vs_RSpec](../../images/UTPLSQL_vs_RSpec.png)](../posts/utplsql-vs-ruby-plsqlruby-plsql-spec-implicit-datatype-conversion-traps.md)
Last two months I was blogging quite a lot about UTPLSQL vs ruby-plsql.
There are lots of aspects that I did not manage to cover so far. I've had a ambitious plan to go through all of the details and dig into the darkest corners to show all the differences. Time is however one thing I'm really short on recently, so instead of going into all the details as planned I've decided to give a high level overview of main differences between UTPLSQL and ruby-plsql.
This will be a summary of the series for now, as I feel like moving into other topics. I might get back to it later if I find good reasons for doing so.

<!-- more -->
Currently I use UTPLSQL as it's "the tool of a choice" for unit testing my Oracle code. Before that I used ruby-plsql.
Interestingly, while unit testing with ruby-plsql I saw quite few people getting "test infected", using UTPLSQL I see many developers preferring test-avoidance technique instead. I'm not sure if it's a matter of skills, education or the tool of choice.
Through last years I've came to realize that writing good tests is really hard.
It's actually quite impossible to write good unit tests without one of those things happening:

- receiving solid portions of good mentoring
- lots of reading on unit testing and TDD
- lots of painful failures and refactorings of your tests

It took all of the above for me, before I've learned how to write tests in a way that they are

- usable
- useful
- maintainable

It was a long journey with many hard lessons, but it was and still is a journey to better. Today I can't imagine having a complex database with multiple packages and dependencies without unit-test coverage. It just doesn't feel safe any more. It actually never felt safe, but I just didn't realise how safe I can be doing changes in database.
So I've learned that writing good and maintainable tests is not simple. What I've also learned recently is that it's not just your skills but also the tools you choose, that impact quality of your tests and your productivity.
Yes, I do manage to get decent tests with UTPLSQL, but it is really hard effort and I'm still unable to get rid of the feeling that the code just smells bad and there's actually not much I can do about it.
Despite the facts that I used Ruby for nothing except unit testing of Oracle code, I cant resist the feeling that the power of the language is just amazing.
There were moments where using ruby-plsql for unit testing made my face as if I would be having a test-drive of Tesla P85D in Insane Mode.
http://www.youtube.com/watch?v=LpaLgF1uLB8
Ruby as a language surprised me with it's syntax and flexibility that I often made me happy as a sandboy.
It was a big challenge to categorize and articulate the exact reasons why I prefer ruby-plsql over UTPLSQL.
It was even a bigger challenge to put it into a comparison that would be dense and more focused on facts than gut-feeling and "it just feels nicer".
[![utplsql-vs-ruby-plsql-feature-comparison.600](../../images/jgebal_utplsql-vs-ruby-plsql-feature-comparison.600.jpg)](../../images/jgebal_utplsql-vs-ruby-plsql-feature-comparison1.pdf)
