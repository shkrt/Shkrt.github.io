---
layout: default
title:  "Differences between RSpec doubles, spies and stubs"
date:   2017-12-25 21:18:27 +0300
comments: true
tags: ruby rspec
---

# Differences between RSpec doubles, spies and stubs

RSpec's built-in 'rspec-mocks' library contains various abstractions to make available the use of test doubles in our
test suites. These abstractions are known as `doubles`, `spies` and `stubs`. They are extensively used to make test suites
and examples isolated, to decouple tested classes from their dependencies. During isolated test examples test doubles
are staying for collaborator classes, make collaborator classes not involved at all.

#### The concept of double

The double is an "empty" object that (theoretically) can stand for any other object.

{% highlight ruby %}
user = double("User")

user.class
=> RSpec::Mocks::Double
{% endhighlight %}

this object is not in any way connected to the real User class and knows nothing about it. The name is just a name.
Basically, this object cannot do anything really useful, except that it will raise an exception if it received the
unexpected message. In this particular case, any message passed to `user` would be unexpected.

{% highlight ruby %}
user.load
RSpec::Mocks::MockExpectationError: #<Double "User"> received unexpected message :load with (no args)
{% endhighlight %}

To make this object actually receive some method calls without an exception, we can initialize the double with predefined
values:

{% highlight ruby %}
user = double("User", load: '1111')
user.load
=> '1111'
user.load(amount: 5)
=> '1111'
{% endhighlight %}

Now the test double can respond to `load` message and ignores any arguments, passed along with the message.

Another way to deal with messages is to declare a `stub` on a test double.

#### Stubs

By declaring a stub, we define, which methods can the double respond to, alongside with their respective arguments
and return values:

{% highlight ruby %}
allow(user).to receive(:load).and_return('1111')
user.load
=> '1111'

allow(user).to receive(:load).with(amount: 5).and_return('5')
user.load(amount: 5)
=> '5'
{% endhighlight %}

Another and more useful side of stubs is that they can be declared over the existing, "real" class instances and classes:

{% highlight ruby %}
class User
  def load
    'real instance method'
  end

  def self.show
    'real class method'
  end
end

user = User.new
user.load
=> 'real instance method'

allow(user).to receive(:load).and_return('1111')
user.load
=> '1111'

User.show
=> 'real class method'

allow(User).to receive(:show).and_return('1111')
User.show
=> '1111'

user_2 = User.new
user_2.load
=> 'real instance method'

allow_any_instance_of(User).to receive(:load).and_return('1111')
user_2.load
=> '1111'
{% endhighlight %}

One of the specifics of the stubs is the working of the `expect` clause. To make the test pass, you should first set
the expectation for the message, and then perform the actions that would invoke the message:

{% highlight ruby %}
class User
  # ... skipped
  def self.setup(instance)
    instance.load
  end
end

# test.rb
require 'rspec'

describe 'User' do
  it 'receives load with one argument' do
    allow_any_instance_of(User).to receive(:load)
    expect_any_instance_of(User).to receive(:load).with('5')
    User.setup(User.new)
  end
end

$ rspec test.rb

1) User receives load iwth one argument
   Failure/Error: instance.load

     #<User:0x005611a1f08350> received :load with unexpected arguments
       expected: ("5")
            got: (no args)
{% endhighlight %}

If we had put the expectation clause after the invocation of the `setup` method, the test would fail for another reason,
because the message would not be received at all. This order of things can seem unnatural at first, because we used to set
the expectations at the end of the test example. To do so, we can use the spies.

#### Test spies

The test double has another form of initialization, that makes the test double completely silent regarding the error messages.

{% highlight ruby %}
user = double('User').as_null_object

user.load
=> #<Double "User">
{% endhighlight %}

There is a shorthand for `.as_null_object`:

{% highlight ruby %}
user = spy('User')
{% endhighlight %}

The spy simply swallows all the messages passed to it, without raising an exception and also without any message at all.
But instead, it 'remembers' all the messages passed to it during the test example, and therefore allows checking if the
messages had been actually received at the end of the example.

{% highlight ruby %}
describe 'User' do
  it 'receives load with one argument' do
    user = spy('user')
    user.load(4)
    expect(user).to have_received(:load).with('5')
  end
end

1) User receives load with one argument
   Failure/Error: expect(user).to have_received(:load).with('5')

     #<Double "user"> received :load with unexpected arguments
       expected: ("5")
            got: (4)
{% endhighlight %}

#### Verifying doubles - safely mocking existing classes

All of this mocking patterns and techniques would seem worthless if they cannot provide a possibility to stay in touch
with actually tested interfaces. For example, we can mock the `User` class, by creating a double and allowing it to receive
`load` method, and then in one of the days, we simply change the name of the method. The test example would remain fully
working, because it works with double, and does not care about changes in the actual class.

{% highlight ruby %}
class User

# this was formerly called 'load'
def preload
  'real instance method'
end

user = double(User)
allow(user).to receive(:load).and_return('5')

user.load
=> '5'
{% endhighlight %}

To prevent these kinds of failing scenarios, there are verifying doubles concept. The verifying double will check if
the mocked class actually can respond to given messages.

{% highlight ruby %}
user = double(User)
allow(user).to receive(:load).and_return('5')

=> RSpec::Mocks::MockExpectationError: #<InstanceDouble(User) (anonymous)> received unexpected message :load with (no args)
{% endhighlight %}

The same way we can define verifying double on a class-level messages:

{% highlight ruby %}
user = class_double(User)
allow(user).to receive(:down).and_return('ok')

user.down
=> RSpec::Mocks::MockExpectationError: the User class does not implement the class method: down
{% endhighlight %}

There is also `instance_spy` shortcut for `instance_double(Klass).as_null_object`,

In this writing, I tried to make a brief overview of the test double concepts available in the RSpec testing framework. All the listed
terminology - spy, double, mock is understood differently by different people and it seems that there is no single, unified point
of view regarding the naming and usage of different patterns of testing using test doubles. As you may have seen from these
short examples, the test doubles are extremely useful to do pure unit-testing, when we have to completely isolate a
class from all of its collaborators, yet still having access to their public interface.

Suggested reading:

[xUnit Patterns](http://xunitpatterns.com/Test%20Double%20Patterns.html)

[Effectively Testing with RSpec 3](https://pragprog.com/book/rspec3/effective-testing-with-rspec-3)
