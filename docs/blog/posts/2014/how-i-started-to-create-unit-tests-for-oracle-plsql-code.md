---
title: "How I started to create Unit Tests for Oracle PL/SQL code"
date:
  created: 2014-01-24
slug: how-i-started-to-create-unit-tests-for-oracle-plsql-code
categories:
  - "testing"
tags:
  - "Oracle"
  - "PL/SQL"
  - "Ruby"
  - "TDD"
  - "unit testing"
---

Before I started working at [Pragmatists](http://www.pragmatists.pl) I never actually took time or effort to research the web for automating the testing process in Oracle databases. My previous job was more about delivering solutions, advisory, analysis, design and implementation using the traditional cascade product delivery approach.
I don't want to get into too many details on how it happened, that me, being a Oracle database developer, landed in Pragmatist, which is an Agile Software Development company specializing in Java, but all I need to say, is that nothing else but good came out of it for me.

<!-- more -->

Me and my colleague were the first Oracle developers hired by Pragmatists. We were pulled into a large ongoing project that combined both Data Warehousing and OLTP in one product.
The project was already rolling for over years, many different Oracle developers were involved and the product was in use by customers for a long time.
The first thing I learned was that there were lots of bugs, PL/SQL code was "owned" by different persons, so there was a common lack of transparency.
The good thing was, that the customer who owned the product and leaded the project, introduced a tool (Quest Code Tester) for test automation in Oracle database.
In the mean-time, Krzysztof Jelski made a one-day training about the Test Driven Development technique which greatly encouraged me to start writing automated tests for PL/SQL code that I deliver.
With enthusiasm we began to write our first automatic tests using Quest code teste, and then the bad things started happening. It turned out, that the tool is very cumbersome and slow, requires lot's of clicking and has lots of limitations. We spend over a week to configure a set of  three unit tests for quite simple thing.
This pushed me to look for different solutions. After some googling, I found that there is a free Unit Testing package with Oracle SQL Developer. It seemed a bit lighter solution than the Huge framework of obstacles presented by Quest.  I also came across a blog of [Raimond Simanovski](http://blog.rayapps.com/2010/10/05/ruby-plsql-spec-gem-and-code-coverage-reporting/) where he introduced his two Ruby gems to do all the work needed to start writing great Unit Tests in Ruby language.
After reading everything on his blog, as well as some basic information about Ruby and Rspec  and some talks with colleagues at work, I've decided to give it a shot. We've started to write Unit Tests in Ruby, a programming language, that we've never used before.
I can't say that it was easy. It is always hard when you try to learn something new. But it was definitely fun, and worth it.
The good things about Ruby is that it is so simple, that anyone can learn it, you may use it both as a scripting language and as a Object Oriented programming language, so there is something good for everyone.
Another great thing is that, since the unit testing is done 100% in a programming language, there are no limitations that are included in the fancy tools, you may implement just about any test that you need.
Within few months Me and Tomek created a set of over 400 Unit Tests that actually did some nice work, proving that the code we deliver is doing what it should.
We also managed to convince the customer, that using Ruby is for unit testing is much more efficient.
I must say I'm really grateful to Raimond for the wonderful work he did creating the [ruby-plsql](https://github.com/rsim/ruby-plsql) and [ruby-plsql-spec](https://github.com/rsim/ruby-plsql-spec) gems.
