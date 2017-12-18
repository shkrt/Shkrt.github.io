---
layout: default
title:  "Transactions with Trailblazer::Operation"
date:   2017-12-18 20:48:26 +0300
tags: ruby trailblazer
---

# Transactions with Trailblazer::Operation

Trailblazer::Operation introduces the concept of `operation` - a programming pattern, used for wrapping a set of actions into a single object,
providing API to work with errors and results of these actions. The actions themselves are often the [interactors](/2017/10/23/simplest-implementation-of-interactor.html).
The similar approach is also used in a [dry-transaction](https://github.com/dry-rb/dry-transaction) library.

The most obvious properties of operation can be summarized as follows:

* the operation takes control of execution some service-object-alike actions one after another, step by step

* execution continues to the next action only if the result of the previous action was successful

* if the result of the previous action was not successful, the code has access to the errors from the previous action.

Some folks are calling the code written by respecting these conventions as "railway-oriented programming", but for me,
it simply looks like a monad.

Let's model a situation when there is a need to perform some action one step after another, and wrap it in a single transaction-like
object. Imagine a user-powered news delivery system, where most of the content is submitted by readers and readers also get rewarded for
worthy articles. The system has some AI-empowered service, that evaluates article's worthfulness and assigns a grade to user
submission. Also, users tend to violate service's rules, and in that case, they are fined instead of getting reward points.
The system also does not give reward points, if a submitted article has been graded too low. In this case, it notifies the user
that the submitted content needs a little more work.
Basically, we have a number of callable service objects for each task:

|---|---|
|`CreateNewsItem`|creates article from user submission|
|`ValidateTerms`| validates if article content does not violate the terms|
|`EvaluateGrade`|evaluates article's grade|
|`CalculateUserReward`|calculates reward points, accounting particular users submission history|
|`UpdateRewardPoints`|updates users reward points|
|`NotifyReward`| notifies user about reward|
|`ApplyFine`|applies fine if user has violated the rules|
|`NotifyUpdateExpected`| notifies user if article should be updated|

If we think in terms of operation, we can split this logic into the successful and unsuccessful tracks. First six operations build into a
successful track, and the last two are the matters of a failing scenario. This can be generalized as the following scheme:

#1. Successful scenario(general):

`CreateNewsItem` --> `ValidateTerms` --> `EvaluateGrade` --> `CalculateUserReward` --> `NotifyReward`

#2. Scenario, when user violated the terms of service

`CreateNewsItem` --> `ValidateTerms` --> `ApplyFine`

#3. Scenario, when user's submission got low grade

`CreateNewsItem` --> `ValidateTerms` --> `EvaluateGrade` --> `NotifyUpdateExpected`

By the use of operation concept, we can build a maintainable implementation of this logic, so let's look at how this can be done using
`Trailblazer:Operation`.

To start, we only need one gem installed:

{% highlight ruby %}
gem install trailblazer-operation

require 'trailblazer/operation'
{% endhighlight %}

Inherit the main service object, that will call further actions, from Trailblazer::Operation:

{% highlight ruby %}
class NewsItemSubmission < Trailblazer::Operation
end
{% endhighlight %}

First prototype looks like this:

{% highlight ruby %}
class NewsItemSubmission < Trailblazer::Operation
  step :create_news_item
  step :validate_terms
  failure :apply_fine, fail_fast: true
  step :evaluate_grade
  failure :notify_update_expected
  step :calculate_user_reward
  step :notify_reward

  def create_news_item(options, hash);  end

  def validate_terms(options, hash);  end

  def apply_fine(options, hash);  end

  def evaluate_grade(options, hash);  end

  def notify_update_expected(options, hash);  end

  def calculate_user_reward(options, hash);  end

  def notify_reward(options, hash);  end
end
{% endhighlight %}

Notice the `fail_fast` option passed to the `apply_fine` method. It ensures that execution of the failure path will not continue
after the first failure, which we need in this case. Without this option, `notify_update_expected` would have been called
after the `apply_fine` method.

Every method in our method chain receives two arguments, the first is a Trailblazer::Skill object, and the second is
params hash, containing additional parameters that can be passed to the method

If we inspect the first argument, the Trailblazer::Skill object, we can find a way to see the whole tree of calls:

{% highlight ruby %}
puts options['pipetree'].inspect(style: :row)

 0 ========================>operation.new
 1 =====================>create_news_item
 2 =======================>validate_terms
 3 <apply_fine===========================
 4 =======================>evaluate_grade
 5 <notify_update_expected===============
 6 ================>calculate_user_reward
 7 ========================>notify_reward
{% endhighlight %}

In Trailblazer terminology, successful and failure paths are called right and left tracks, respectively. So, this `inspect` output
contains a visual representation of left and right tracks. Also we can inspect the call tree another way, by calling the whole
operation as:

```
NewsItemSubmission.call['pipetree']
```

Here we used methods for steps, but instead of them we can use lambda or class with `call` class method, so let's
rewrite the class using a separate class instead of each method:

{% highlight ruby %}
class CreateNewsItem
  extend Uber::Callable
  def self.call(options, **); end
end

class ValidateTerms
  extend Uber::Callable
  def self.call(options, **); end
end

class ApplyFine
  extend Uber::Callable
  def self.call(options, **); end
end

class EvaluateGrade
  extend Uber::Callable
  def self.call(options, **); end
end

class NotifyUpdateExpected
  extend Uber::Callable
  def self.call(options, **); end
end

class CalculateUserReward
  extend Uber::Callable
  def self.call(options, **); end
end

class NotifyReward
  extend Uber::Callable
  def self.call(options, **); end
end

class NewsItemSubmission < Trailblazer::Operation
  step CreateNewsItem
  step ValidateTerms
  failure ApplyFine, fail_fast: true
  step EvaluateGrade
  failure NotifyUpdateExpected
  step CalculateUserReward
  step NotifyReward
end
{% endhighlight %}

How does Trailblazer::Operation decide, if the current step was successful or not? Based on its return value.
To continue on the right track, we have to return truthy value, to move to the failure track - falsy value. Now, all we have to do
to ensure that our service object is working as expected is to write specs, for each of the 3 scenarios.

{% highlight ruby %}
require 'rspec'

describe NewsItemSubmission do
  context 'when all actions are successful' do
    it 'calls only right-track actions' do
      expect(CreateNewsItem).to receive(:call).and_return(true)
      expect(ValidateTerms).to receive(:call).and_return(true)
      expect(ApplyFine).not_to receive(:call)
      expect(EvaluateGrade).to receive(:call).and_return(true)
      expect(NotifyUpdateExpected).not_to receive(:call)
      expect(CalculateUserReward).to receive(:call).and_return(true)
      described_class.call
    end
  end

  context 'when ValidateTerms fails' do
    it 'calls CreateNewsItem, then ValidateTerms, stops after ApplyFine' do
      expect(CreateNewsItem).to receive(:call).and_return(true)
      allow(ValidateTerms).to receive(:call).and_return(false)
      expect(ApplyFine).to receive(:call)
      expect(EvaluateGrade).not_to receive(:call)
      expect(NotifyUpdateExpected).not_to receive(:call)
      expect(CalculateUserReward).not_to receive(:call)
      described_class.call
    end
  end

  context 'when EvaluateGrade fails' do
    it 'calls CreateNewsItem, then ValidateTerms, then ApplyFine' do
      expect(CreateNewsItem).to receive(:call).and_return(true)
      expect(ValidateTerms).to receive(:call).and_return(true)
      expect(ApplyFine).not_to receive(:call)
      allow(EvaluateGrade).to receive(:call).and_return(false)
      expect(NotifyUpdateExpected).to receive(:call)
      expect(CalculateUserReward).not_to receive(:call)
      described_class.call
    end
  end
end
{% endhighlight %}

As we can see from the output of rspec, each one of our three scenarios is working perfectly.

Another basic concept of Trailblazer::Operation is the results.

Basically, results is a hash, that can be modified at every step and adds to a resulting Trailblazer::Skill object.
For example, if we want to save some intermediary results in an operation object, we can add following construct in every
chosen step:

{% highlight ruby %}
class CreateNewsItem
  extend Uber::Callable
  def self.call(options, **)
    options['result.create_news_item'] = current_user.id
    true
  end
end
{% endhighlight %}

These values are then accessible in each step of the operation and in the resulting object:

{% highlight ruby %}
NewsItemSubmission.()['result.create_news_item']
{% endhighlight %}

Suggested reading:

[Operation Overview](http://trailblazer.to/gems/operation/2.0/)

[Operation Basics](http://trailblazer.to/guides/trailblazer/2.0/01-operation-basics)

[Operation API](http://trailblazer.to/gems/operation/2.0/api.html)
