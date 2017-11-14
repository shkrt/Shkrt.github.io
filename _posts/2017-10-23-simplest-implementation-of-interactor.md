---
layout: default
title:  "Simplest implementation of interactor"
date:   2017-10-23 18:28:47 +0300
tags: ruby
---

# Simplest implementation of interactor

I had not found any well-known description for Interactors. For example, [Interactor gem's README](https://github.com/collectiveidea/interactor) provides following characteristic:
> An interactor is a simple, single-purpose object. Interactors are used to encapsulate your application's business logic. Each interactor represents one thing that your application does.

Looking at interactor gem and also [Hanami::Interactor](https://github.com/hanami/utils/blob/master/lib/hanami/interactor.rb) I've built my image of interactors accordingly to following principles:

* Interactor is a single-purpose class
* It returns boolean status attribute - success or failure
* It provides access to errors, collected during operation or initialization

So, this writing is dedicated to implementing interactor in the simplest of possible ways.

First, let's write the example part of the code that will use interactor logic.

{% highlight ruby %}
class EventsController < WebsiteController
  def fetch
    response = FetchFeedbacks.new('http://example.com', 4).call.success?
    if response.success?
      redirect_to events_path, notice: 'Feedbacks fetching started'
    else
      redirect_to events_feedback_path, alert: response.errors
    end
  end
end
{% endhighlight %}

This code calls FetchFeedbacks service class and then redirects or shows error messages accordingly to the result of the call. This is how should interactor work in my opinion.
Now let's imagine, how what exactly should FetchFeedbacks class be to meet our conditions.

{% highlight ruby %}
# fetch_feedbacks.rb
require 'uri'
require_relative 'interactor'

class FetchFeedbacks
  include Interactor

  def initialize(source, amount)
    @source = source
    @amount = amount
  end

  def call
    if valid_source?(@source)
      PostFeedbacks.perform_async(@source, @amount)
    else
      add_error('Invalid url passed')
    end
    self
  end

  private

  def valid_source?(src)
    src =~ URI.regexp
  end
end

class PostFeedbacks
  class << self
    def perform_async(src, amount)
      # ...
    end
  end
end
{% endhighlight %}

We had already put interactor logic into separate `Interactor` module, thus all the classes including that mixin would have `success?` and `add_error` methods.
In `call` method we validate input parameters, and if the condition is not met, we add error to interactor object's errors. In a successful scenario, some background job named `PostFeedbacks` is launched. In both scenarios, `self` is returned. It's worthful to note, that we are not limited to return the only that slim object,
containing success or failure status. We also can set custom instance variables for each object. Including Interactor module should only assure that we can call `success?`, `failure?` and `add_error` methods on the resulting object.

But we have not written Interactor module yet. Let's continue with writing specs for the FetchFeedbacks class. BTW I'm not a huge fan of writing tests for non-existent code, but in this case, we would benefit from doing this, because it would help us to have a clean representation of how Interactor module should be designed.
Also, some people would say, that we should write specs for Interactor module itself, but I think that writing specs for collaborator class can show us Interactor requirements more clearly.

{% highlight ruby %}
# test.rb

require 'rspec'
require_relative 'fetch_feedbacks'

RSpec.describe 'FetchFeedbacks' do
  context 'when valid source provided' do
    let(:subject) { FetchFeedbacks.new('http://example.com', 2).call }

    it 'has not failure status' do
      expect(subject.failure?).to be_falsy
    end

    it 'has success status' do
      expect(subject.success?).to be_truthy
    end

    it 'has no errors' do
      expect(subject.errors).to be_empty
    end

    it 'delegates work to PostFeedback class' do
      expect(PostFeedbacks).to receive(:perform_async)
      subject
    end
  end

  context 'when no valid source provided' do
    let(:subject) { FetchFeedbacks.new('1', 2).call }

    it 'has errors' do
      expect(subject.errors).not_to be_empty
    end

    it 'has failure status' do
      expect(subject.failure?).to be_truthy
    end

    it 'has not success status' do
      expect(subject.success?).to be_falsy
    end
  end
end
{% endhighlight %}

Now, let's make tests green.

{% highlight ruby %}
module Interactor
  # set up included hook
  def self.included(base)
    # use prepend to override base class' initialize method
    base.prepend(Initializer)
    # use class_eval to set attributes and methods
    base.class_eval do
      attr_accessor :success
      attr_reader :errors

      # this attribute will indicate successful status - true by default
      def success?
        errors.empty?
      end

      # this attribute will indicate failure status - false by default
      def failure?
        !success?
      end

      # method to add errors
      # wrapper around << method
      def add_error(msg)
        errors << msg
      end
    end
  end

  module Initializer
    def initialize(*args, &block)
      self.instance_variable_set(:@errors, [])
      super
    end
  end
end
{% endhighlight %}

This implementation successfully passes all tests. Someone might ask "what's the point of using wrapper method `add_error`, if we still have a possibility of using `<<`, `push`, `concat`, `+` methods on object's `errors` array?"
This method added to provide custom error raising logic. If we want to manage errors array directly, this does not break any functionality, because any interactor object that has errors will remain in the failed status.

About interactors:

[Hanami::Utils page - includes Interactor](https://github.com/hanami/utils)

[A couple of words about interactors](https://mkdev.me/en/posts/a-couple-of-words-about-interactors-in-rails)

[collectiveidea/interactor](https://github.com/collectiveidea/interactor)
