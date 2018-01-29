---
layout: default
title:  "Writing Rack middleware"
date:   2018-01-29 13:54:10 +0300
comments: true
tags: ruby
---

# Writing Rack middleware

Rack middleware allows to work with HTTP requests on lower level than many of the frameworks allow, though it is
not a really low level and represented by the same Ruby objects we used to work with. The Rack itself can be considered as basically a wrapper toolkit
around the Ruby's Net::HTTP library. So, the Rack middleware can be used to perform, for example, the following set of tasks:

- Primitive, low-level authentication
- Adding headers and other info to the HTTP response
- Preventing throttling, attacks
- Various monitoring tasks

In this post, I will try to describe using a middleware for enhanced request logging(actually, not so enhanced). I think that in real-world conditions
this scenario is unlikely because such a middleware would create additional disk i/o and therefore slow down the request/response cycle.
But for the understanding of Rack and middlewares, it is ok.

Let's start with a bare minimum rack application, that only consists of 5 lines of code:

{% highlight ruby %}
# config.ru
require 'rack'

app = proc do
  [200, [], []]
end

run app
{% endhighlight %}

Run `rackup`, and the server will start listening at port 9292, serving empty 200 response for all requests.

Now let's add an empty middleware:

{% highlight ruby %}
# config.ru
require 'rack'
require_relative 'rack_logger'

# ...
use RackLogger
{% endhighlight %}

{% highlight ruby %}
# rack_logger.rb
class RackLogger; end
{% endhighlight %}

Launching application will now result in `wrong number of arguments (given 1, expected 0) (ArgumentError)`
Let's add explicit `initialize` definition in our middleware class:

{% highlight ruby %}
# rack_logger.rb
class RackLogger
  def initialize(app)
  end
end
{% endhighlight %}

Now the application starts, but visiting localhost:9292 will raise something like

```
NoMethodError at /
undefined method `call' for #<RackLogger:0x005619c2c17cf8>
```

Adding a `call` method definition will result in

```
ArgumentError: wrong number of arguments (given 1, expected 0)
```

So, I am tired of this guessing approach and now will take a look at a typical middleware code, for example, [Rack::Deflater](https://github.com/rack/rack/blob/master/lib/rack/deflater.rb)
From the code we can see the typical method signatures we need:

{% highlight ruby %}
# rack_logger.rb
class RackLogger
  def initialize(app)
    @app = app
  end

  def call(env)
    @app.call(env)
  end
end
{% endhighlight %}

After this, our requests are successfully processed by our no-op middleware and the application works again.

Now, to actually log something, we can use the env hash, provided as the first argument to `call` method.
For example, keys names `REQUEST_METHOD`, `REQUEST_PATH`, `QUERY_STRING` can provide interesting information for logging.
By the way, all of this info has already been outputted by Rack's default logger, so we can use them to somehow decorate
output and write it to the filesystem.

{% highlight ruby %}
# rack_logger.rb
# ...
  def call(env)
    puts [Time.now.to_i, env['REQUEST_METHOD'], env['REQUEST_PATH'], env['QUERY_STRING']].join(',')
    @app.call(env)
  end
{% endhighlight %}

Now, visiting the `http://localhost:9292/posts?by_date=today` URL will result in something like:

```
1517188859,GET,/posts,by_date=today
```

Which is obviously something easily consumable by CSV library.

Now we can add actual filesystem writing:

{% highlight ruby %}
# rack_logger.rb
class RackLogger
  def initialize(app)
    @app = app
    @filename = "server.log"
    File.open(@filename, 'a+') {}
  end

  def call(env)
    File.open(@filename, 'a+') do |f|
      f.puts [Time.now.to_i, env['REQUEST_METHOD'], env['REQUEST_PATH'], env['QUERY_STRING']].join(',')
    end
    @app.call(env)
  end
end
{% endhighlight %}

The middleware is working and does what expected. To make the code more reusable, we should
add a configuration layer. Let's do it the traditional way, via `configure` block:

{% highlight ruby %}
# config.ru
use RackLogger

RackLogger.configure do |config|
  config.logfile = 'log.txt'
end
{% endhighlight %}

And rewrite RackLogger class as following:

{% highlight ruby %}
# rack_logger.rb
require_relative 'configuration'

class RackLogger
  def initialize(app)
    @app = app
    @filename = RackLogger.config.logfile || "server.log"
    File.open(@filename, 'a+') {}
  end

  #...

  class << self
    def configure
      yield config
    end

    def config
      @config ||= Configuration.new
    end
  end
end
{% endhighlight %}

All we have left to do is to add the Configuration class itself:

{% highlight ruby %}
# configuration.rb
class Configuration
  attr_accessor :logfile
end
{% endhighlight %}

For real-world usage, all these also should be wrapped in module namespace, but this is beyond the scope of the article.

To learn more about middlewares, you can dig directly to the [rack codebase](https://github.com/rack/rack), which
contains a number of middlewares, which you probably have been implicitly using on an everyday basis, when writing web applications
in some of Ruby frameworks.

Suggested Reading:

[Rack source](https://github.com/rack/rack)
[Rack-attack source](https://github.com/kickstarter/rack-attack)
