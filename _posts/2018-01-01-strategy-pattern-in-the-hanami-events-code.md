---
layout: default
title:  "Strategy pattern in the Hanami::Events code"
date:   2018-01-01 19:28:04 +0300
tags: ruby
---

# Strategy pattern in the Hanami::Events code

The [Hanami::Events](https://github.com/hanami/events) is a small experimental events framework for Hanami.
Its codebase is small, simple and and contains some real-world examples
of the strategy pattern usage - one of the GoF patterns for object-oriented design.

#### The Strategy

The main purpose of the strategy is to deal with situations when we have two or more different classes, that have to
interact with another class via a unified interface. The first thing that comes into my head when talking about strategy
is the code that has to perform some command via interaction with different third-party API.

Let's look at a simplified example of strategy:

{% highlight ruby %}
class SubscriberNotificator
  attr_accessor :notifier

  def deliver_message
    if @notifier == :telegram_api
      # telegram-specific implementation
    elsif @notifier == :email
      # email-specific implementation
    else
      # some fallback code, i.e
      # raise NotImplementedError
    end
  end
end
{% endhighlight %}

We can underline the following concepts regarding the strategy pattern:

* interface

The main public API that is visible to library's user. This may be a separate declarative class, or a completely virtual abstraction.
The latter may turn strategy into the template method.

* consumer

Any object using interfaces public API methods. It knows nothing about implementation details.

* implementation

Implementation of the interface's public API methods.

* concrete classes

Classes containing the implementation details.

These concepts are not taken from original GoF book, by the way.

Looking at the Hanami::Events code, you can easily spot the usage of the strategy pattern:

{% highlight ruby %}
module Hanami
  module Events
    class Base
      #..omitted
      def format(type)
        Formatter[type].new(subscribed_events).format
      end
    #..omitted
{% endhighlight %}

`Formatter[type].new` is a typical way to initialize the concrete class when using the Strategy pattern. The syntax assumes that
you use the dry-container library, but for simplicity, you can imagine that all available concrete classes are stored in some
data structure and being fetched from it by key lookup. Concrete classes themselves reside in lib/hanami/events/formatters
directory:

```
json.rb
plain_text.rb
xml.rb
```

{% highlight ruby %}
module Hanami
  module Events
    module Formatter
      class JSON
      #..omitted
      def initialize(events_meta)
        @events_meta = events_meta
      end

      def format
        { events: @events_meta }.to_json
      end
      #..omitted

# plain text formatter
# lib/hanami/events/plain_text.rb

      def format
        "Events:\n#{formatted_events}"
      end

# xml formatter
# lib/hanami/events/xml.rb

      def format
        XmlSimple.xml_out(events: @events_meta)
      end
{% endhighlight %}

There is only one interface method, `format`, which delegates its arguments to one of the concrete classes, and thus
we can have the output formatted by three different formatters.

This example recalls the respective example from Russ Olsens' "Design Patterns in Ruby" book, where the Strategy
pattern is also described via the example of some formatter.

Also, there is another example of the strategy in the Hanami::Events codebase. Though the naming may confuse you - the directory
named adapter, this is not an adapter pattern, but strategy. The key differences between strategy and adapter can be described as following:

The Adapter is used when you already have classes that have to collaborate, but their interfaces differ. Thus, the adapter provides
an additional layer to make one class support the given interface.
The Strategy is also about providing a single interface but from the point of interchangeability. The Adapter classes are a completely
different creatures, but the Strategy classes have a very similar behavior, differing only by implementation details.

The code in `lib/hanami/events/adapter` is all about interchangeability between classes that perform the same work, so this code is
definitely about strategy pattern.

{% highlight ruby %}
module Hanami
  module Events
    module Base
      attr_reader :adapter

      def initialize(adapter_name, options)
        @adapter = Adapter[adapter_name.to_sym].new(options)
      end

      def broadcast(event, **payload)
        adapter.broadcast(event, payload)
      end

      def subscribe(event_name, &block)
        adapter.subscribe(event_name, &block)
      end

      def subscribed_events
        adapter.subscribers.map(&:meta)
      end

      #... omitted
    end
  end
end
{% endhighlight %}

This method initializes the class with given concrete class and therefore, all the public API calls will be directed to that class:

```
def initialize(adapter_name, options)
  @adapter = Adapter[adapter_name.to_sym].new(options)
end
```

Then we have the three methods, that are forming the interface, `broadcast`, `subscribe`, `subscribers` (the latter is called from
`subscribed_events` method). All classes collaborating with `Hanami::Events::Base` class have to implements their version of these methods.

{% highlight ruby %}
# lib/hanami/events/adapter.rb
module Hanami
  module Events
    class Adapter
      extend Dry::Container::Mixin

      register(:memory_sync) do
        require_relative 'adapter/memory_sync'
        MemorySync
      end

      register(:memory_async) do
        require_relative 'adapter/memory_async'
        MemoryAsync
      end

      register(:redis) do
        require_relative 'adapter/redis'
        Redis
      end
    end
  end
end
{% endhighlight %}

This piece of code registers three concrete classes, MemorySync, MemoryAsync and Redis using dry-container.

And then each of these classes implement aforementioned interface methods:

{% highlight ruby %}
module Hanami
  module Events
    class Adapter
      class MemoryAsync
        attr_reader :subscribers

        def initialize(logger: nil, **)
          @logger = logger
          @subscribers = []
          @event_queue = Queue.new
        end

        def broadcast(event_name, payload)
          @event_queue << { id: SecureRandom.uuid, name: event_name, payload: payload }
        end

        def subscribe(event_name, &block)
          @subscribers << Subscriber.new(event_name, block, @logger)

          return if thread_spawned?
          thread_spawned!

          Thread.new do
            loop { call_subscribers }
          end
        end

        # ...omitted

{% endhighlight %}

The Redis and MemorySync classes will implement these methods in their own way.

The code we have seen in Hanami::Events library showed the real-world usage of the strategy pattern and helped to make the
difference between Adapter and Strategy patterns more clear.

Suggested reading:

[Design Patterns in Ruby](https://www.amazon.com/Design-Patterns-Ruby-Addison-Wesley-Professional-ebook/dp/B0010SEN1S)
