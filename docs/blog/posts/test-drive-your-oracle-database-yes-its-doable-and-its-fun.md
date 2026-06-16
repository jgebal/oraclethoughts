---
title: "Test Drive your Oracle database. Yes, it's doable. And it's fun!"
date:
  created: 2015-05-04
slug: test-drive-your-oracle-database-yes-its-doable-and-its-fun
categories:
  - "testing"
tags:
  - "PL/SQL"
  - "TDD"
  - "unit testing"
---

![06_Red_Green_Refactor](../../images/06_Red_Green_Refactor-300x178.jpg)
**Foreword**
Over a month ago I've made a big decision and a shift in my life. I've decided to move to Ireland and start a new career there. Since I moved, I have significant amount of free time, as I no longer waste 3 hours each day on commuting. Also, since my wife still leaves in Poland (for now), I'm all on my own when I finish my work.
Well, maybe that's not 100% true, since now we have Internet with Skype/Hangouts/appear.in and others.
Anyway. I'm not feeling lonely, but rather take advantage of the time that is given to me, to catch up on things I've neglected for some time.

<!-- more -->
I spent some nice hours listening to speeches by Marting Fowler, Robert C. Martin, Gojko Adzic and John Seddon.
On [Norwegian Developers Conference 2012](http://lanyrd.com/2012/ndc2012/), Robert C. Martin (Uncle Bob) was giving a speech on the Single Responsibility Rule.
http://www.youtube.com/embed/Gt0M\_OHKhQE?autoplay=0&start=1339&end=1556&rel=0&showinfo=0&controls=1
I watch the Youtube speech and it was very interesting and educative. For some reason, the thing that got my focus was the Roman Numerals Kata mentioned by him that was done by Jason Gorman and is described at his [blog](http://codemanship.co.uk/parlezuml/blog/?postid=1021)
The experiment interested me to the extent, that I've decided to do a coding kata on my own for the first time.
Well It's not the first time I've seen or heard about coding kata. I've learned a lot about it while working [@Pragmatists](http://pragmatists.pl), it also was not the first time I've done such kata.
Thanks to Krzysztof Jelski's persistance, we've managed to make a TDD coding kata workshop for Oracle developers [@Pragmatists](http://pragmatists.pl) just before I left the company.
This experiment felt a bit different. It was about doing it on my own. It was about observing myself and the programming flow.
**The Test Driven Development Roman Numerals Kata in Oracle**
My personal goals for the kata were:

- to practice and document TDD cycles of Red-Green-Refactor (also know as Test-Code-Refactor)
- to see what I like about it
- to see what is difficult about it
- to come up with conclusions and feeling what is good/bad/easy/hard
  and finally, just to practice good programming patterns

For the experiment I used:

- Oracle 11g XE database on a VirtualBox machine
- GIT as a version control system
- ruby-plsql-spec as a unit testing framework of a choice

The approach I took for the exercise was following:

1. The Red phase.
   Write a test, to express a new requirement.
   Execute all tests and expect new test to fail, as there is no implementation of requirement in place.
   Commit the changes to document the phase.
2. The Green phase.
   Write the minimal implementation that makes the test pass, don't bother with styling and code quality, just implement the functionality with minimal cost.
   Execute all tests and expect all tests to pass, as the code should cover all the requirements expressed by tests.
   Commit the changes to document the phase.
3. The Refactor phase.
   Review both test and implementation code. If anything can be improved, do it.
   Execute all tests and expect all tests to pass.The refactoring should not affect the functional aspects of the code.
   Commit the changes if there are any.
4. If there is another requirement to implement, start over from point 1.

That's really all there is.
Even though the rules are simple, they are a magic discovery.
**First steps of the kata**
The Red phase

```ruby
require_relative 'spec_helper'

describe 'Decimal to roman numeral converter' do

  it 'should return I for 1' do
    expect( plsql.to_roman_numeral(1) ).to eq('I')
  end

end
```

The Green phase

```sql
CREATE OR REPLACE FUNCTION to_roman_numeral( pv_decimal_number INTEGER ) RETURN VARCHAR2 IS
BEGIN
  RETURN 'I';
END;
/
```

The Refactor phase

```
Nothing done
```

The Red phase

```ruby
require_relative 'spec_helper'

describe 'Decimal to roman numeral converter' do

  it 'should return I for 1' do
    expect( plsql.to_roman_numeral(1) ).to eq('I')
  end

  it 'should return II for 2' do
    expect( plsql.to_roman_numeral(2) ).to eq('II')
  end

end
```

The Green phase

```sql
CREATE OR REPLACE FUNCTION to_roman_numeral( pv_decimal_number INTEGER ) RETURN VARCHAR2 IS
BEGIN
  RETURN
    CASE pv_decimal_number
      WHEN 1 THEN 'I'
      WHEN 2 THEN 'II'
    END;
END;
/
```

The Refactor phase

```
Nothing done
```

**The end results**
Unit tests

```ruby
require_relative 'spec_helper'
require_relative 'to_roman_numeral'

describe 'Decimal to roman numeral converter' do

  roman_numbers = %w{I II III IV V VI VII VIII IX X XI XII XIII XIV XV XVI XVII XVIII XIX XX
                     XXI XXII XXIII XXIV XXV XXVI XXVII XXVIII XXIX
                     XXX XXXI XXXII XXXIII XXXIV XXXV XXXVI XXXVII XXXVIII XXXIX XL
                     XLI XLII XLIII XLIV XLV XLVI XLVII XLVIII XLIX
                     L LI LII LIII LIV LV LVI LVII LVIII LIX LX
                     LXI LXII LXIII LXIV LXV LXVI LXVII LXVIII LXIX LXX LXXI LXXII
                     LXXIII LXXIV LXXV LXXVI LXXVII LXXVIII LXXIX LXXX
                     LXXXI LXXXII LXXXIII LXXXIV LXXXV LXXXVI LXXXVII LXXXVIII LXXXIX
                     XC XCI XCII XCIII XCIV XCV XCVI XCVII XCVIII XCIX C
                    }
  (1..roman_numbers.size).each do |given|

    expected = roman_numbers[given-1]

    it "should return #{expected} for #{given}" do
      expect( plsql.to_roman_numeral(given) ).to eq( expected )
    end
  end

  [
    [200	,'CC'],
    [300	,'CCC'],
    [400	,'CD'],
    [500	,'D'],
    [600	,'DC'],
    [700	,'DCC'],
    [800	,'DCCC'],
    [900	,'CM'],
    [2015, 'MMXV'],
  ].each do |given, expected|

    it "should return #{expected} for #{given}" do
      expect( plsql.to_roman_numeral(given) ).to eq( expected )
    end
  end

  it 'should raise exception for 3001' do
    expect{ plsql.to_roman_numeral(3001) }.to raise_exception(/works only up to 3000/)
  end

end
```

And the code

```sql
CREATE OR REPLACE TYPE decimal_to_roman IS OBJECT (
  divisor INTEGER,
  symbol VARCHAR2(2),
  MEMBER FUNCTION get( pv_number IN OUT NOCOPY INTEGER ) RETURN VARCHAR2
);
/

CREATE OR REPLACE TYPE BODY decimal_to_roman IS
  MEMBER FUNCTION get( pv_number IN OUT NOCOPY INTEGER ) RETURN VARCHAR2 IS
    lv_result VARCHAR2(100);
    lv_times  INTEGER := FLOOR( ( pv_number ) / divisor );
  FUNCTION repeat( pv_what VARCHAR2, pv_times INTEGER ) RETURN VARCHAR2 IS
    BEGIN
      RETURN RTRIM(LPAD( ' ', pv_times * LENGTH( pv_what ) + 1, pv_what ));
    END;
  BEGIN
    IF pv_number >= divisor THEN
      lv_result := repeat( symbol, lv_times );
      pv_number := MOD( pv_number, divisor );
    END IF;
    RETURN lv_result;
  END;
END;
/

CREATE OR REPLACE FUNCTION to_roman_numeral( pv_decimal_number INTEGER ) RETURN VARCHAR2 IS
  lv_decimal_number INTEGER := pv_decimal_number;
  lv_roman_numeral  VARCHAR2(100);

  TYPE la_decimal_roman_map IS TABLE OF decimal_to_roman;
  lv_decimal_roman_map la_decimal_roman_map;

BEGIN
  IF lv_decimal_number > 3000 THEN
    raise_application_error(-20000, 'Function works only up to 3000 decimal number');
  END IF;
  lv_decimal_roman_map
    := la_decimal_roman_map(
      decimal_to_roman(1000,'M'),
      decimal_to_roman(900,'CM'), decimal_to_roman(500,'D'), decimal_to_roman(400,'CD'),
      decimal_to_roman(100,'C'), 
      decimal_to_roman(90,'XC'), decimal_to_roman(50,'L'), decimal_to_roman(40,'XL'),
      decimal_to_roman(10,'X'),
      decimal_to_roman(9,'IX'), decimal_to_roman(5,'V'), decimal_to_roman(4,'IV'),
      decimal_to_roman(1,'I')
      );
  FOR i IN lv_decimal_roman_map.FIRST .. lv_decimal_roman_map.LAST LOOP
    lv_roman_numeral := lv_roman_numeral || lv_decimal_roman_map(i).get(lv_decimal_number);
  END LOOP;
  RETURN lv_roman_numeral;
END;
/
```

Full commits history can be seen on my [github project page](https://github.com/jgebal/oracle_testing_tdd_roman_numerals_kata/commits/master)
**Observations**
The things that made me feel very comfortable during the exercise were the facts that I was able to:

- pick one requirement at a time and focus on it
- write test for the requirement, not for the code
- focus on implementation of a small piece of functionality

Switching of focus felt really productive and was driving away boredomness and tiredness.
The fact that each step was really small and pretty quick to go through was brilliant.
It feels like playing a really **cool computer game**, where every minute your score is raising.

- Have a failing test -> point
- Have a working code and passing test -> point
- Make refactoring and have all tests green -> whoa!! That's a bonus!!!

This is why I think TDD is awesome fun! It gave me a huge amount of satisfaction of a job done really well, though it was just a small kata.
**Some more thoughts**
When I was implementing a test I was really focused on one small feature. When I was implementing code I was really focused on getting the test to work. It was so much easier to keep focus. I didn't have to jump from one thing to another, just stay on track of the current step.
I dare saying that the TDD approach is a way of having well-focused creativity that brings extraordinary result.
It also removes the stressful questions like:

- Do I really know if what I did was right?
- Can I change that code and be sure that nothing else will get broken?

On the other hand, it was pretty hard to follow the rules. It was very tempting to do jump forward with implementation beyond test coverage or do some small improvements to the existing code while implementing a new functionality.
In fact, I broke the rules. It happened when my code grew, I implemented another test and realized, that following the implementation pattern I have chosen is no longer a good approach. So I have had a Red (failing) test and I started to refactor existing code.
Still, even with a rule broken, the refactoring felt really safe. I already had unit tests to cover all the implemented functionalities.
**Summary**
It took me about 4 hours to complete the exercise, mainly because I've done this particular exercise for the first time.
The kata was published on the [Roman Numeral Katas github project](https://github.com/froderik/roman_numeral_katas) (thank you Fred) and joined several implementations done in different languages for the kata.
Now, when I finished the kata, I wonder if it would not have been better, to just put all of the code to the trash and start over, when I realized that the implementation approach was pretty ugly. Learning to let go and throw the code away is another exercise that in my opinion each developer should do.
Thomas Edison said:
![thomas-edison-quote-jpg](../../images/thomas-edison-quote-jpg-300x150.jpg)
Following this, if you fail to admit what you're doing doesn't work, you will not progress.
**Foot notes** on how to setup your local environment for unit testing with ruby-plsql-spec
The easiest way seems to follow the instructions provided with ruby-plsql (https://github.com/rsim/ruby-plsql)
For Windows setup, you will need to the following:

- Install ruby
- Have Oracle client (32 bit, as ruby 1.9.3 comes in 32bit version on Windows and needs a 32bit OCI)
- Install ([ruby-plsql](https://github.com/rsim/ruby-plsql))
- download oracle-xe-11.2.0-1.0.x86\_64.rpm.zip
- bring up the VM: vagrant up ([as described in doc](https://github.com/rsim/ruby-plsql))

I have done all of the above steps on my local machine a long long time ago, so don't know if I have skipped any.
But once you have your env up and running:

- create a project directory
- initialize git repository: git init
- initialize ruby-plsql-spec: plsql-spec init
- setup connection credentials in spec/database.yml

and you're good to go :)
***Good luck and have as much fun as I did.***
If you like this post, you might also enjoy my other posts on [ruby-plsql, UTPLSQL](../posts/utplsql-vs-ruby-plsqlruby-plsql-spec-part-one.md) and [Continuous Integration](../posts/utplsql-vs-ruby-plsql-running-oracle-unit-tests-on-jenkins-ci.md).
