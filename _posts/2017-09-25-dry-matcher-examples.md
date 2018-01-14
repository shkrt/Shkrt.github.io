---
layout: default
title:  "Dry-matcher usage example"
date:   2017-09-25 12:25:54 +0300
comments: true
tags: dry-rb
---

# Dry-matcher usage example

The [dry-matcher][dry-matcher] is a part of so-called [dry-rb][dry-rb] libraries - set of ruby gems that are becoming more and more popular in
open source community right now.

Dry-matcher can be used to provide possibility to perform pattern-matching in Ruby. As one of its creators Piotr Solnica said,
"dry-matcher was extracted from dry-transaction as a low-level building block. It's not meant to be used as a first-class pattern matching library"

Assuming we have some code, that interacts with api, where api response can indicate either success or failure -  we can
pattern-match on the result of the response, and then perform some computations depending on the response. Conditional
logic, to be honest. Also there are multiple success cases.

Here is example response from our api:

success_cases:
```
{
  "current": {
    "temperature" : "25.67",
    "units": "celsius",
    "humidity": "54"
  }
}
```

```
{
  "forecast": {
    "temperature" : "25.67",
    "units": "celsius",
    "humidity": "54"
  }
}
```

failure case:

```
{
  "error": {
    "explain" : "no weather data for given city"
  }
}
```

And here is some service/interactor class, fetching weather info from external API:

{% highlight ruby %}
class WeatherFetchService
  def initialize(city)
    @base_url = url_for_city(city)
  end

  class WeatherFetchError < StandardError; end

  def perform
    results = Net::HTTP.get(@base_url)
    if results['current']
      Weather::Parse.new(results['current']).perform
    elsif results['forecast']
      Weather::Parse.new(results['forecast']).perform
    else
      WeatherFetchError.new("api interaction fail")
    end
  end

  private

  def url_for_city
    params = { city: city }
    URI::HTTP.build(host: "example.com",
                    path: 'weather',
                    query: URI.encode_www_form(params))
  end
end
{% endhighlight %}

So, now we'll refactor our conditional logic to use pattern-matching. First, we create a matcher, that will contain
two success cases and one fail case:

{% highlight ruby %}
require "dry-matcher"

module Matchers
  module WeatherApi
    # Match `{ "current": { some_value } } for first success case
    current_case = Dry::Matcher::Case.new(
      match: -> value { value.keys.first == 'current' },
      resolve: -> value { value['current'] }
    )

    # Match `{ "forecast" : { some_value } }` for second success case
    forecast_case = Dry::Matcher::Case.new(
      match: -> value { value.keys.first == 'forecast' },
      resolve: -> value { value['forecast'] }
    )

    # Match anything else for failure
    failure_case = Dry::Matcher::Case.new(
      match: -> value { true },
      resolve: -> _ {}
    )

    # Build the matcher
    matcher = Dry::Matcher.new(current: current_case,
                               forecast: forecast_case,
                               failure: failure_case)
  end
end
{% endhighlight %}

Our matcher consists of three cases, two success cases for each type of response repectively, and one failure case, that
matches if none of the success cases matched. Each Dry::Matcher::Case object has `match` key, where we describe matching
logic. Here, the pattern considered matched if the first key of response json is one of 'current' or 'forecast'. In the
`resolve` key we specify, how to deal with matched value, here we can parse the response, split it, return in another format
and so on.

Now we can get rid of conditional logic, and our perform method becomes:

{% highlight ruby %}
def perform
  results = Net::HTTP.get(@base_url)
  Matchers::WeatherApi.call(results) do |m|
    m.current { |v| Weather::Parse.new(v).perform }
    m.forecast { |v| Weather::Parse.new(v).perform }
    m.failure { |_v| WeatherFetchError.new("api interaction fail") }
  end
end
{% endhighlight %}


As we can see from this example, dry-matcher's api can seem to us a bit complicated at a first glance,
especially if you expected it to provide experience like in Elixir, Haskell or some other programming languages, where
pattern-matching is one of key features. But its separation of matching and resolving logic and multiple cases is lovely.

[dry-matcher]: http://dry-rb.org/gems/dry-matcher/
[dry-rb]: http://dry-rb.org/
