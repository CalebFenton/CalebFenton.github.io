title: "Code Kata: TDD and Run-length Encoding"
tags:
  - code-kata
  - tdd
comments: true
date: 2016-04-23 19:26:04
---

A _kata_ is a martial arts training method. It's a set of detailed and choreographed movements and poses. The movements are performed repeatedly and internalized. A _code kata_ is is a training method for developing skill in programming. Take something you do frequently, or wish to do better, strip away everything not essential, and practice it repeatedly.
<!-- more -->

I was introduced to code katas by [@OngEmil](https://twitter.com/OngEmil) who's one of the best engineers I've ever met. The law of broscience teaches us that "Emil is good implies his advice is true", so I took it seriously and started practicing and I feel like they're a useful learning tool to understand.

Example code kata:

* Build a trivial Ruby project with Gemfile and specs
* Write a program which uses an unfamiliar data structure, e.g. <a href="http://billmill.org/bloomfilter-tutorial/" target="_blank">bloom filter</a>
* Implement an unfamiliar algorithm, e.g. <a href="https://en.wikipedia.org/wiki/Dijkstra%27s_algorithm#Pseudocode" target="_blank">shortest path</a>
* Create a new Android project from scratch
* Practice keyboard shortcuts for your favorite IDE

Practicing kata combines the essential elements of skill development:

1. break down a technique into the essentials
2. learn by repetition

The key elements of a code kata are:

1. short - should take fewer than 30 minutes, once competent
2. essential - develops a desired or critical skill, or improves a weakness
3. focused - depth rather than breadth
4. challenging - if it's not challenging, you either aren't focused, or the problem is too easy

# Why use katas?
You have to practice to get good at anything. Efficient practice focuses on just a few essentials -- do this kick five thousand times, play these scales for an hour, lift this weight eight times, etc. This is intuitive, yet with the skill _programming_, most never consider applying the same systematic approach. Many programmers think the key to good programming is about _intelligence_, or they think it's to just code a lot. It's neither. The key is _correct practice_. Intelligence helps, but programming is a craft, like carpentry. I doubt people think being intelligent is enough to be good at carpentry. Coding a lot also helps, but without guidance, effort is wasted. Code katas are an attempt to bring a systematic approach to developing programming skill.

# When to use katas?
There are three main types of problem I think are amenable to kata training:

1. common stuff - setting up a new project, keyboard shortcuts, IDE usage
2. new skills - new language, data structure, algorithm
3. weaknesses - something that slows you down

For more on the philosophy and more examples of katas check out:

* <a href="http://codekata.com/" target="_blank">codekata.com</a>
* <a href="http://www.codekatas.org/" target="_blank">www.codekatas.org</a>

# Kata: Run Length Encoding
For this kata, I'll be implementing <a href="https://en.wikipedia.org/wiki/Run-length_encoding" target="_blank">run-length encoding</a> using test-driven development. It's simple, but not too simple. The goal is not to practice the algorithm, but to practice writing RSpec tests, and setting up the project. Originally, I tried doing a kata on using TDD to come up with an edit distance algorithm but found it to be a little too complex. I don't think it's always possible to get to an algorithm using TDD. I'm also going to be pretending to not know <a href="http://rosettacode.org/wiki/Run-length_encoding#Ruby" target="_blank">regex</a>.

When performing a kata, be deliberate and focused. Take your time, and let these things sink in. Time box it to 30 minutes and stop if you go over. Start over again next time and you'll be faster.

First, create the directory and hop in:

```bash
mkdir run_length
cd run_length
```

Or, if you use *zsh*:

```bash
take run_length # now you're cool
```

I use <a href="https://rvm.io/" target="_blank">RVM</a> to manage having multiple Ruby and JRuby versions and multiple gemsets for each. A project can specify which Ruby version and gemset to use. When you navigate into that directory, RVM will automatically switch to that version.

```bash
echo 2.2.1 > .ruby-version
echo run_length > .ruby-gemset
cd .
```

Create a `Gemfile`:

```ruby
source 'https://rubygems.org'
gem 'rspec', '~> 3.0'
```

Since this is an empty gemset, need to install `bundler`:

```bash
gem install bundler
bundle install
```

Now, when can initialize rspec:

```bash
rspec --init
```

Create a *minimal* `lib/run_length.rb`:

```ruby
class RunLength
    def self.get(string)
    end
end
```

Create `spec/run_length_spec.rb` with one test. I opted for a simple test not because it's an edge case (empty input), but because it's simple. I'll avoid defensive programming, so I won't test for input of unicode, input of numbers, etc.

```ruby
require 'run_length'

describe RunLength do
    describe '#get' do
        let('get') { RunLength.get(string) }
        context 'with an empty string' do
            let('string') { '' }
            subject{ get }
            it { should eq '' }
        end
    end
end
```

What do you know? It fails:

```bash
╭─caleb@haxcalibur  ~/repos/code_kata/run_length ‹ruby-2.2.1@fuzzy_lcs›
╰─$ rspec
F

Failures:

  1) RunLength#get with an empty string should eq ''
     Failure/Error: it { should eq '' }

       expected: ''
            got: nil

       (compared using ==)
     # ./spec/run_length_spec.rb:11:in `block (4 levels) in <top (required)>'

Finished in 0.00109 seconds (files took 0.08408 seconds to load)
1 example, 1 failure

Failed examples:

rspec ./spec/run_length_spec.rb:11 # RunLength#get with an empty string should eq ''
```

I'm practicing <a href="http://www.jamesshore.com/Blog/Red-Green-Refactor.html" target="_blank">Red, Green, Refactor</a>. That means, to fix a test, I'll add just enough code to pass the tests *and nothing else*. Then, I'll clean it up if necessary. This is not something I normally get to practice. For comparison, my normal workflow is:

1. build a prototype
2. hack out some tests (sometimes at the same time as step 1)
3. realize the prototype is crap
4. scrap the prototype
5. rewrite the prototype (simultaneously with step 6)
6. refactor the tests, adding and removing where appropriate
7. repeat #3 - #6 until it meets requirements

You might call this "test assisted development" mixed with "merciless refactoring." I'm usually programming something weird or researchy, and it works well for me with those types of problems. I'll use tests to break the problem down and build intuition. Writing good unit tests _after_ writing a functioning unit usually leads to a lot of refactoring at first. I've found I can approach the design of a unit by thinking about how I would test it, and that cuts down on refactoring time. I use unwritten tests as opportunties to work when there's limited time. It can be hard to implement a new feature in 30 minutes, but I could probably add one or two test cases in that time.

After writing _just enough_ code to pass the test, it becomes:

```ruby
def self.get(string)
    ''
end
```

Why return `''` instead of `string`? Because empty quotes is more simple than a variable. Avoid adding any complexity. Before writing code, ask: "Can there be less and still pass the tests?"

Now, let's add a slightly less trivial test case:

```ruby
context 'with "a"' do
    let('string') { 'a' }
    subject{ get }
    it { should eq '1a' }
end
```

Code change seems a little much. Spent some time thinking about it, but couldn't think of anything.

```ruby
def self.get(string)
    return '' if string.empty?
    "1#{string}"
end
```

Next test adds a little more complexity:

```ruby
context 'with "aa"' do
    let('string') { 'aa' }
    subject{ get }
    it { should eq '2a' }
end
```

There's a pattern emerging. We can just add lots of `if/else`s for each test case, but that isn't going to webscale. It would appear we must violate the principle of "just pass the tests." But, lots of conditionals could be said to have a certain complexity. The challenge now is to write the code in a way that is more simple than having two conditionals and an else. Also, we can be certain there is a nearly 1 to 1 ratio of inputs to conditionals. At this point, I'm trying to avoid jumping ahead and solving the entire problem, since it's simple enough to guess.

The code becomes:

```ruby
def self.get(string)
    return '' if string.empty?
    "#{string.size}#{string[0]}"
end
```

It's not much more complex. It passes the tests. I keep feeling there's a smarter way to do it. That's fine. I'll practice this again for a few days and see if anything new jumps out at me.

Fourth test:

```ruby
context 'with "aab"' do
    let('string') { 'aab' }
    subject{ get }
    it { should eq '2a1b' }
end
```

Originally, I started writing a state machine that iterated over the string and juggled variables like character counts, last character processed, etc. This added a lot of complexity and was ugly. I want to add the least amount of complexity and still pass the tests. I thought about it for a few minutes and rewrote it into something more clear:

```ruby
def self.get(string)
    counts = Hash.new(0)
    string.each_char { |l| counts[l] += 1 }
    counts.map { |l, c| "#{c}#{l}" }.join
end
```

You could point out this still makes use of a state machine, but the difference is that this is more idiomatic, there's less state tracking, and it's relying on code I didn't have to write. Not writing code is the best way to not write bugs. This relies on the fact that Ruby > 1.9 maintains key insertion order for hashes. If it didn't, I could use a sorted hash, or a sorted set of just keys.

Some people might say we're done, but I'd like to add just _one_ complex case. This is not so much to test for correctness. It doesn't cover any additional code because no code changes are needed to make it pass, but it does _document the intent_ of the code.

```ruby
context 'with "aaaabcccccccccdddddz"' do
    let('string') { 'aaaabcccccccccdddddz' }
    subject{ get }
    it { should eq '4a1b9c5d1z' }
end
```

That's about 30 minutes for me. I'll stop here and practice again tomorrow. I'll do this for a week and next time I need to setup a Ruby + RVM + Rails project, I can get all the foundation laid without having to consult any documentation. If this problem is too short after a few days, I'll add a decoding method.

What do you think? Do you see something I could have done better? A better test? Cleaner, simpler solutions? Let me know. Have an idea for a kata? Share it here.
