title: "Code Kata: Bloom Filter"
tags:
  - code-kata
  - algorithm
comments: true
date: 2016-04-26 19:11:11
---

If you're unfamiliar with what a Code Kata is, check out my previous post [Code Kata: TDD and Run-length Encoding](/2016/04/19/code-kata-tdd-and-run-length-encoding/)

The goal for this kata is to learn an unfamiliar data structure. It's called a bloom filter. I've read the Wikipedia article and have used them, but until I've made it myself I don't understand it deeply. The more fundamental my understanding, the more flexible I can be in applying a concept. It's just like calculus. There's a world of difference between merely memorizing a formula and having a deep, intuitive understanding.
<!-- more -->

# What is a bloom filter?
A bloom filter has two methods:

1. `add` - adds an element
2. `check` / `include` - false if an element is *definately not* in the filter, true if it *possibly* is

This is a _probabilistic_ data structure. It can say if an element is *definately not* in the set (no false negative), but it may say an element is in the set which isn't (false positive). The trade off for this uncertainty is that even for large sets the bloom filter is relatively much smaller and faster than storing looking up in a complete set.

I'm going to assume you're familiar with the technical aspects of the bloom filter. Others have already done a great job of explaining how they work.

* Easy to read description: http://billlaboon.com/probabilistic-data-structures-part-i-bloom-filters/
* Description plus a cool interactive visual aid: https://www.jasondavies.com/bloomfilter/

# When to use a bloom filter?
It's usefulness is not as obvious as an `Array` or `Hash`, but there are situations where expensive I/O operations can be skipped for elements definitely not in the filter, and only done if an element may be in the present.

Example time. Let's say you're analyzing Android applications and want to know which strings are unique to a given app. This is important for automated signature generation for malware detection. You must use strings _endemic_ to a particular sample so the signature won't false positive. Let's also assume you already have a few million Android apps and their strings. The simple approach is to extract and store strings and their frequencies. The distribution of string frequencies has a long tail. Some strings are in almost every app, but most are only in a small number of apps. Also, most of the strings for any given app are in almost every app. When analyzing an application, each string frequency is looked up to see if it's under some threshold. These lookups are expensive.

To improve on this, you create a bloom filter and add the common strings. Now, instead of checking the frequency of every string with an expensive table lookup, you check the bloom filter _first_. Most lookups are thus eliminated. The trade off is some genuinely uncommon strings are considered as common. In this situation, for this scenario, that's fine. If there's difficulty finding enough unique strings, you can skip the bloom filter and lookup each string. This won't happen often so it's an acceptable cost.

More example uses for bloom filters:

* https://www.quora.com/What-are-the-best-applications-of-Bloom-filters
* https://stackoverflow.com/questions/4282375/what-is-the-advantage-to-using-bloom-filters

# Kata: Bloom Filter

## Day 1
I'm only giving myself 30 minutes, so I'm going to keep it basic. First, I'll setup a Ruby project with RSpec. I've covered how to do this in my last kata, here: https://blog.srcclr.com/code-kata-tdd-run-length-encoding/

Normally, I'd start with tests, but as I found in my last kata, strict TDD of algorithms is hard. I'd rather prototype the algorithm then hit it with tests. If it were just engineering code, I'd consider testing first.

I start by quickly reviewing a few articles on how they work. That takes a few minutes. All I really need is an array, preferably a bitset, and some hashing methods. I don't want to spend time worrying about efficient storage, and I don't want to pull in any libraries until I get it to work. I'll just use a plain array. I pick 100 elements as a starting size because it's close to some numbers I've seen while reading about it.

```ruby
class BloomFilter
    def initialize(size = 100)
        @bitset = Array.new(size, 0)
    end
end
```

Now, I need some hashing methods. The only important property of the hash functions is that they are evenly distributed. A quick review of the String reveals `String#hash` which looks idea, and `String#sum` which looks less ideal. I figure it's good enough for now. Perfect is the enemy of good.

```ruby
def add(s)
    hash(s).each { |h| @bitset[h] = 1 }
end

def hash(s)
    h1 = s.hash % @bitset.size
    h2 = s.sum % @bitset.size
    [h1, h2]
end
```

To wrap it up, add a method to check if an element is possibly in the filter:

```ruby
def include?(s)
    hash(s).each { |h| return false unless @bitset[h] == 1 }
    true
end
```

With only a few minutes left, I wrote a little benchmarking code to make sure it all worked:

```ruby
# Add lots of random strings
length = 10
strings = []
100.times { strings << rand(36 ** length).to_s(36) }

b = BloomFilter.new
strings.each { |s| b.add s }

# Make sure they're all possibly included
strings.each { |s| puts "#{s} should be included but isn't" unless b.include? s }

# Rough check of false positives
other_strings = []
100.times do
    string = rand(36 ** (1 + rand(100))).to_s(36)
    next if strings.include? string
    other_strings << string
end

fps = 0.0
other_strings.each { |s| fps += 1.0 if b.include? s }
puts "False positive %: #{100 * (fps / other_strings.size)}"
```

## Day 2
I read a bit more about bloom filters. I learn there are a few tunable parameters, and some advanced implementations that allow for removal of elements. The advanced stuff looks way out of the scope of a 30 minute session. I'm thinking that a 30 minute kata isn't a good way to learn anything complicated. You'd need a few solid hours of uninterrupted study to digest anything heavy. For my goal of deepening my understanding of bloom filters, it's enough. There are two main tunable aspects of the bloom filter:

1. Size of the array, usually _m_. ~16 bits per element regardless of element size to give 0.1% error rate.
2. Number of hashing methods, usually _k_. Three seems common.

I wanted to add some tests with randomly generated data. For example, generate 1,000 strings, add them all to a filter, and make sure they're all possibly present. Then, generate 1,000 strings not in the original set and make sure none of them are present. I feel like that's not a good unit test, because you'd have to `add` and `include?` in the same test. I spent a lot of time trying to learn the relevant RSpec syntax to make it work.

```ruby
require 'bloom_filter'

describe BloomFilter do
    let('bf') { BloomFilter.new }

    describe '#include?' do
        it 'should include strings once added' do
            strings = []
            100.times { strings << rand(36 ** (1 + rand(100))).to_s(36) }
            strings.each { |s| bf.add s }
            strings.each { |s| expect(bf.include?(s)).to be true }
        end

        it 'should not include strings that are not added' do
            strings = []
            100.times { strings << rand(36 ** (1 + rand(100))).to_s(36) }
            strings.each { |s| expect(bf.include?(s)).to be false }
        end
    end
end
```

I also moved the benchmark code out of the `BloomFilter` class file and into it's own file.

## Day 3
Rewrote everything. Such practice. Much fast.

## Day 4
Rewrote everything. Added an alias for `#<<` so addition can be like `Array#<<`. Renamed `include?` to `test` because it's more honest, and shorter than `may_include?`.

## Day 5
Optimization time. I updated the benchmarking code and saw that I wasn't getting the false positive rate as low as I wanted. I suspected it was my hash functions. I replaced `String#sum` with two over-kill cryptographic digests `MD5` and `SHA1`. I could have gotten away with something simpler, like `CRC32` or `ADLER`, but I had no internet and this is all I could divine by grepping through the Ruby installation.

The effect was good though. It brought the error rate down an order of magnitude.


# Final Thoughts

I feel like I understand the bloom filter now. This includes a deeper appreciation for how much I don't yet understand of advanced implementations of bloom filters. The simple, mindful practice of carefully writing Ruby was useful. Each time I rewrote it, it came faster and more easily and little optimizations were more apparent.

The full project can be viewed here:
https://github.com/CalebFenton/code_kata/tree/master/bloom_filter
