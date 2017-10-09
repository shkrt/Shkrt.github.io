---
layout: default
title:  "Dependency inversion with dry-container"
date:   2017-10-09 10:47:25 +0300
category: dry-rb
---

# Dependency inversion with dry-container

The [dependency inversion principle][wiki] is a "D" of SOLID principles. Shortly speaking, the principle is following:

* High-level modules should not depend on low-level modules. Both should depend on abstractions.
* Abstractions should not depend on details. Details should depend on abstractions.

To be more concrete, for example, we can paraphrase this as

* Business logic should not depend on implementation details

So, let's look at an example, and refactor it iteratively.

The first iteration:

{% highlight ruby %}
class PushNotification
  def initialize(payload)
    @message = payload.text
    @account_ids = payload.user_ids
  end

  def call
    MobileMessageService.deliver(@message, @account_ids)
  end
end
{% endhighlight %}

The main flaw of this example is the usage of hard-coded dependency, thus it violates both Open/Closed principle and Dependency Inversion Principle.
The business logic is simple - deliver messages to selected user accounts. But the business logic here depends on implementation details -
`call` method 'has knowledge' about which exact service class will be performing message delivery. MobileMessageService contains implementation
details, which we should replace by an abstraction. Also, this would allow us to introduce, for example, WebMessageService in the future releases,
which will contain another low-level implementation of this business logic. To refactor this example
to ensure usage of dependency inversion, we should replace hard-coded class by an abstraction.

{% highlight ruby %}
class PushNotification
  def initialize(payload)
    @message = payload.text
    @account_ids = payload.user_ids
  end

  def call(performer)
    performer.deliver(@message, @account_ids)
  end
end

# dummy implementation example
class MobileMessageService
  def self.deliver(message, account_ids)
    account_ids.each { |acc| p "#{acc} --> #{message} via mobile" }
  end
end

# dummy implementation example
class WebMessageService
  def self.deliver(message, account_ids)
    account_ids.each { |acc| p "#{acc} --> #{message} via web" }
  end
end

Payload = Struct.new(:text, :user_ids)

PushNotification.new(Payload.new('Some text', [1, 114, 56]))
                .call(MobileMessageService)

PushNotification.new(Payload.new('Some text', [23, 10, 16]))
                .call(WebMessageService)
{% endhighlight %}

Now we have replaced `MobileMessageService` by a variable, which we pass to the `PushNotification` class in `call`
method. Now we can pass to this class any other performer class, as long as it responds to `deliver` message with given signature.
This example illustrates, how implementation details can be replaced by abstraction.

So, let's move onto the third iteration, where we introduce a couple of new performer classes with the help of [dry-container][dry-container]
library

{% highlight ruby %}
require 'dry-container'

container = Dry::Container.new

container.register(:mobile) { MobileMessageService }
container.register(:web) { WebMessageService }
{% endhighlight %}

Here we initialized Dry::Container instance and registered our performer classes. Now we can refer to them using `resolve` method:

{% highlight ruby %}
performer = container.resolve(:mobile)
PushNotification.new(Payload.new('Some text', [1, 114, 56]))
                .call(performer)

performer = container.resolve(:web)
PushNotification.new(Payload.new('Some text', [23, 10, 16]))
                .call(performer)
{% endhighlight %}

And we can go even further and provide access to the app's all classes as small, component-like building blocks, using [dry-auto_inject][dry-auto_inject] gem:

{% highlight ruby %}
require 'dry-auto_inject'

class PerformerContainer
  extend Dry::Container::Mixin

  register('mobile') { MobileMessageService }
  register('web') { WebMessageService }
  register('push_notification') { PushNotification }
end

Import = Dry::AutoInject(PerformerContainer)
Payload = Struct.new(:text, :user_ids)

class App
  include Import['mobile', 'web', 'push_notification']

  def call(payload)
    [mobile, web].each do
      push_notification.new(payload).call(mobile)
    end
  end
end

payload = Payload.new('Some text', [1, 114, 56])
App.new.call(payload)
{% endhighlight %}

In this example, we have made our way from simple, but unconfident ruby class to the scalable code, consisting of loosely coupled, small building blocks. Of course, examples of code are all short and way contrived, but with the further growth of application, the necessity of decoupling using dependency inversion will become more and more obvious.

[dry-container]: http://dry-rb.org/gems/dry-container/
[dry-auto_inject]: http://dry-rb.org/gems/dry-auto_inject
[wiki]: https://en.wikipedia.org/wiki/Dependency_inversion_principle
