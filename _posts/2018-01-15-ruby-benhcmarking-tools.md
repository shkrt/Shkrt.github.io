---
layout: default
title:  "Ruby benchmarking tools"
date:   2018-01-15 15:43:40 +0300
comments: true
tags: ruby
---

# Ruby benchmarking tools

When coming to a speed of Ruby applications, the first thing we going to do is to identify application's bottlenecks.
Usually, such a task is not imaginable without a benchmarking tool. In this article, I intend to make an overview of
typical tools for simplest ruby benchmarking tasks.

The first and most obvious tool is `benchmark` available from within Ruby's standard library. Let's try it to see performance
difference between `sort_by` and `shuffle` methods to shuffle the elements of an array of 1_000_000 elements.

{% highlight ruby %}
require 'benchmark'

arr = (1..1_000_000).to_a

Benchmark.bm do |x|
  x.report("sort_by") { arr.shuffle }
  x.report("shuffle") { arr.sort_by { rand } }
end

            user     system      total        real
shuffle 0.030000   0.000000   0.030000 (  0.030575)
sort_by 0.930000   0.000000   0.930000 (  0.936051)
{% endhighlight %}

As we can see, with this amount of data shuffling via `Array#shuffle` is about 30 times more efficient than the using `Array.sort_by`.

Another example will involve the performance comparison between `String#include` and `Regexp#match` to assert whether the given
string includes the provided substring or not.

{% highlight ruby %}
require 'benchmark'

string = "abc"

Benchmark.bm do |x|
  x.report("include?") { string.include?("a") }
  x.report("match") { string.match(/a/) }
end

              user     system      total        real
include?  0.000000   0.000000   0.000000 (  0.000012)
match     0.000000   0.000000   0.000000 (  0.000031)

{% endhighlight %}

This example shows that String#include? is about 2.6 times more performant than Regexp.match in this particular case.
But that is not very precise measurement. If we try to wrap our code blocks into loops, we would get different results
everytime we run the benchmark:

{% highlight ruby %}
Benchmark.bm do |x|
  x.report("include?") { 50_000.times { string.include?("a") } }
  x.report("match") { 50_000.times { string.match(/a/) } }
end
{% endhighlight %}

You can try this yourself, `include?` would always win, but the numbers will differ significantly between the benchmarks.
So, to make precise measurements, we have to choose the number of iterations.
Fortunately, the Ruby ecosystem has another tool, that makes this choice for us - it is the [evanphx/benchmark-ips](https://github.com/evanphx/benchmark-ips), which is the
enhanced version of benchmark.

Let's try our previous example with `benchmark-ips`:

{% highlight ruby %}
Benchmark.ips do |x|
  x.report("include?") { string.include?("a") }
  x.report("match") { string.match(/a/) }
  x.compare!
end
Warming up --------------------------------------
            include?   320.314k i/100ms
               match   128.064k i/100ms
Calculating -------------------------------------
            include?      7.550M (± 2.7%) i/s -     37.797M in   5.010589s
               match      1.752M (± 1.6%) i/s -      8.836M in   5.045055s

Comparison:
            include?:  7549975.8 i/s
               match:  1751951.7 i/s - 4.31x  slower
{% endhighlight %}

This kind of benchmark operates with `iterations per second` measurement and therefore its output shows the difference
between compared code blocks in a more clear and understandable way. Obviously, this tool is more popular for these reasons.

Let's compare the performance difference between Struct, OpenStruct, PORO, and a Hash using `benchmark/ips`, which now will
help us to build a really fancy benchmark report.

{% highlight ruby %}
require 'benchmark'
require 'benchmark/ips'
require 'ostruct'

class RectangularClass
  attr_accessor :width, :height

  def initialize(width, height)
    @width = width
    @height = height
  end
end

class RectangularStruct < Struct.new(:width, :height); end


p "Initialization"

Benchmark.ips do |x|
  x.report('PORO') { RectangularClass.new(15, 55) }
  x.report('Struct') { RectangularStruct.new(15, 55) }
  x.report('Hash') { { width: 15, height: 55 } }
  x.report('ostruct') { OpenStruct.new(width: 15, height: 55) }
  x.compare!
end

p "Reading"

poro = RectangularClass.new(15, 55)
struct = RectangularStruct.new(15, 55)
hash = { width: 15, height: 55 }
ostruct = OpenStruct.new(width: 15, height: 55)

Benchmark.ips do |x|
  x.report('PORO') { poro.width }
  x.report('Struct') { struct.width }
  x.report('Hash#[]') { hash[:width] }
  x.report('Hash#fetch') { hash.fetch(:width) }
  x.report('ostruct') { ostruct.width }
  x.compare!
end

p "Writing"

Benchmark.ips do |x|
  x.report('PORO') { poro.width = 43 }
  x.report('Struct') { struct.width = 43 }
  x.report('Hash#[]') { hash[:width] = 43 }
  x.report('ostruct') { ostruct.width = 43 }
  x.compare!
end
{% endhighlight %}

This script produces a very descriptive report about the performance of each data structure in this simple scenario.
As you may see from its output(obviously, truncated), the OpenStruct is really a performance destroyer, and Struct is as performant as the Plain
Old Ruby Object, when it comes to reading or initialization.

```
Initialization
Comparison:
              Struct:  5758806.7 i/s
                PORO:  5598489.7 i/s - same-ish: difference falls within error
                Hash:  3143382.6 i/s - 1.83x  slower
             ostruct:  1150810.4 i/s - 5.00x  slower

Reading
Comparison:
                PORO: 13365686.7 i/s
             Hash#[]: 12988414.6 i/s - same-ish: difference falls within error
              Struct: 12721531.5 i/s - same-ish: difference falls within error
          Hash#fetch: 10978007.0 i/s - 1.22x  slower
             ostruct:  7877572.2 i/s - 1.70x  slower

Writing
Comparison:
                PORO: 12676271.4 i/s
             Hash#[]: 10976062.0 i/s - 1.15x  slower
              Struct: 10609637.3 i/s - 1.19x  slower
             ostruct:  5380540.4 i/s - 2.36x  slower
```

Suggested Reading:

[benchmark](http://ruby-doc.org/stdlib-2.5.0/libdoc/benchmark/rdoc/Benchmark.html)
[benchmark-ips](https://github.com/evanphx/benchmark-ips)
