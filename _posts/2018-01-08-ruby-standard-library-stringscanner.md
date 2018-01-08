---
layout: default
title:  "Ruby Standard library: StringScanner"
date:   2018-01-08 15:43:40 +0300
tags: ruby
---

# Ruby Standard library: StringScanner

The StringScanner library is part of Ruby's standard library, you probably heard about it if you ever had to write a
text parser. Regular expressions are ok when we have to extract just a small parts of text according to patterns, but
if we are in a need for a fully charged lexical parser, the StringScanner comes in handy.

The StringScanner operates with pointers and regular expressions, which makes easier to extract information from loosely
structured or completely non-structured texts.

The real-world example can be found at the source code of Shrine library, or its `data_uri` plugin, to be more precise.
This plugin deals with data:URL encoded files, i.e. the image file that is encoded as a base64 string. As you may know,
the data:URL encoded files have a certain structure:

```
data:[<MIME-type>][;charset=<encoding>][;base64],<data>
```

This means that the data:URL string begins with `data:`, followed by content's MIME-type, followed by encoding,
followed by `;base64`, and in the end, comes the base64-encoded data itself.
The data:URL encoded file's string representation may look like this:

```
data:image/jpeg;base64,R0lGODdhMAAwAPAAAAAAAP///ywAAAAAMAAw
AAAC8IyPqcvt3wCcDkiLc7C0qwyGHhSWpjQu5yqmCYsapyuvUUlvONmOZtfzgFz
ByTB10QgxOR0TqBQejhRNzOfkVJ+5YiUqrXF5Y5lKh/DeuNcP5yLWGsEbtLiOSp
a/TPg7JpJHxyendzWTBfX0cxOnKPjgBzi4diinWGdkF8kjdfnycQZXZeYGejmJl
ZeGl9i2icVqaNVailT6F5iJ90m6mvuTS4OK05M0vDk0Q4XUtwvKOzrcd3iq9uis
F81M1OIcR7lEewwcLp7tuNNkM3uNna3F2JQFo97Vriy/Xl4/f1cf5VWzXyym7PH
hhx4dbgYKAAA7
```

This is the perfect ground for StringScanner. Let's look more thoroughly at the Shrine code.

First, StringScanner is `require`d:

{% highlight ruby %}
require "strscan"
{% endhighlight %}

Then the custom error is defined - this would be used when parser happens to find any invalid data:

{% highlight ruby %}
class ParseError < Error; end
{% endhighlight %}

Then, a number of constants are defined. This is a common practice for parsers.

The constant indicating that we have reached a `data:` part of the string:

{% highlight ruby %}
DATA_REGEXP = /data:/
{% endhighlight %}

The constant that serves as a mime-type extractor:

{% highlight ruby %}
MEDIA_TYPE_REGEXP = /[-\w.+]+\/[-\w.+]+(;[-\w.+]+=[^;,]+)*/
{% endhighlight %}

Base64 marker:

{% highlight ruby %}
BASE64_REGEXP = /;base64/
{% endhighlight %}

Indicating the only comma that separates `;base64` part from the data string

{% highlight ruby %}
CONTENT_SEPARATOR = /,/
{% endhighlight %}

These constants are defined accordingly to the data:URL format specification and each constant corresponds to the lexical
part of the encoded string.

Next, the parsing happens in the private `parse_data_uri` method.

First, initialize a new instance of StringScanner:

{% highlight ruby %}
scanner = StringScanner.new(uri)
{% endhighlight %}

Then, parsing goes step by step over data:URL encoded string according to data:URL specification:

Finding `data:` part, and raise error if the string does not contain it:

{% highlight ruby %}
scanner.scan(DATA_REGEXP) or raise ParseError, "data URI has invalid format"
{% endhighlight %}

Find a mime-type marker and store its value in a variable:

{% highlight ruby %}
media_type = scanner.scan(MEDIA_TYPE_REGEXP)
{% endhighlight %}

Finding `;base64` part to make a step over it and store its value in a variable:

{% highlight ruby %}
base64 = scanner.scan(BASE64_REGEXP)
{% endhighlight %}

Finally, scanner reaches a data string (or raises a custom error, if there is not `CONTENT_SEPARATOR` present, which is the comma):

{% highlight ruby %}
scanner.scan(CONTENT_SEPARATOR) or raise ParseError, "data URI has invalid format"
{% endhighlight %}

And then the method returns a hash, which contains `scanner.post_match` value, which in our case holds the encoded string.

{% highlight ruby %}
{
  content_type: content_type,
  base64:       !!base64,
  data:         scanner.post_match,
}
{% endhighlight %}

This example is very simple and educative, as for me. Let's go ahead to the next thing we can build using StringScanner.
This will be a simple calculator that will evaluate arithmetic operations from a string. So, the following test should pass:

{% highlight ruby %}
require 'rspec'

describe Calc do
  it 'performs addition' do
    string = "5+4"
    expect(Calc.call(string)).to eq(9)
  end

  it 'performs subtraction' do
    string = "5-4"
    expect(Calc.call(string)).to eq(1)
  end

  it 'performs multiplication' do
    string = "5*4"
    expect(Calc.call(string)).to eq(20)
  end

  it 'performs division' do
    string = "5/4"
    expect(Calc.call(string)).to eq(1)
  end

  it 'maintains spaces' do
    string = "5 + 50"
    expect(Calc.call(string)).to eq(55)
  end
end
{% endhighlight %}

StringScanner should help us reveal the following entities from given string:

1. The first operand of the arithmetic operation. It can be an arbitrary number of digits.

2. The operation itself, its symbol should be among `*/-+`

3. Second operand of the arithmetic operation. It can be also an arbitrary number of digits.

4. Spacings between operands - these are optional

First, we define a class with all constants.

{% highlight ruby %}
class Calc
  OPERATIONS = { "+" => :add, "-" => :sub, "*" => :mul, "/" => :div }
  SPACE = /\s+/
  DIGITS = /\d+/
  OPERATION_SYMS = /[-+\/*]/
end
{% endhighlight %}

And then we scan for each operand, spaces, and operation sequentially:

{% highlight ruby %}
class Calc
  class << self
    def call(str)
      scanner = StringScanner.new(str)
      first_operand = Integer(scanner.scan(DIGITS))
      scanner.scan(SPACE)
      operation = OPERATIONS[scanner.scan(OPERATION_SYMS)]
      scanner.scan(SPACE)
      second_operand = Integer(scanner.scan(DIGITS))
      send(operation, first_operand, second_operand)
    end
  end
end
{% endhighlight %}

The operation definitions are pretty straightforward then:

{% highlight ruby %}
private

def add(first, second)
  first + second
end

def sub(first, second)
  first - second
end

def mul(first, second)
  first * second
end

def div(first, second)
  first / second
end
{% endhighlight %}

I did not mention error management here because it's as simple as in the first Shrine library example. I hope these
two examples would be helpful to build your picture about the usefulness of StringScanner library.


Suggested reading:

[Docs](https://ruby-doc.org/stdlib-2.5.0/libdoc/strscan/rdoc/StringScanner.html)
