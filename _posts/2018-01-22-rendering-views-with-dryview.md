---
layout: default
title:  "Rendering views with dry-view"
date:   2018-01-22 23:54:10 +0300
comments: true
tags: ruby
---

# Rendering views with dry-view

Dry-view is another view rendering system, built primarily for dry-web stack and to use alongside with dry-system, but
being completely standalone, it can be considered as a replacement for another view rendering systems, such as cells,
actionview, and built-in rendering systems of Sinatra and Roda.

For a brief example, let's consider introducing the dry-view into a following traditionally oversimplified Sinatra application:

Our app has only 4 files:

{% highlight ruby %}
# Gemfile
source 'https://rubygems.org'
ruby '2.3.4'

gem 'sinatra'
gem 'slim'
gem 'faker'
{% endhighlight %}

Main application class:

{% highlight ruby %}
# app.rb
require 'sinatra'
require 'faker'

class App < Sinatra::Base
  get '/' do
    users = (1..10).map do |a|
      { id: a, name: Faker::Name.name, occupation: Faker::Job.title }
    end

    slim :index, locals: { users: users }
  end
end
{% endhighlight %}

config.ru to launch the app via rackup:

{% highlight ruby %}
# config.ru
dev = ENV['RACK_ENV'] == 'development'

if dev
  require 'logger'
  logger = Logger.new($stdout)
end

require_relative('app')
run(App)
{% endhighlight %}

and our only template:

{% highlight ruby %}
h1 Users list

table
  thead
    th ID
    th Name
    th Occupation
  tbody
    - users.each do |user|
      tr
        td = user[:id]
        td = user[:name]
        td = user[:occupation]
{% endhighlight %}

As everyone may understand from this code, the application only renders table with some fake data, i.e. table of usernames
with their respective ids and occupation information.

Replacing Sinatra's built-in renderer takes a few steps:

First, install dry-view:

{% highlight ruby %}
# Gemfile
gem 'dry-view'
#...
{% endhighlight %}

Then, make the separate directories for view objects and templates(just like you may have seen in Hanami or Phoenix):

```
app.rb
config.ru
Gemfile
Gemfile.lock
templates/
views/
```

Remove Sinatra rendering from the app:

{% highlight ruby %}
# app.rb
#...
require_relative 'views/user_index_view'

class App < Sinatra::Base
  get '/' do
    users = (1..10).map do |a|
      { id: a, name: Faker::Name.name, occupation: Faker::Job.title }
    end

    view = UserIndexView.new
    view.call(users: users)
  end
end
{% endhighlight %}

create UserIndexView functional object:

{% highlight ruby %}
# views/user_index_view.rb
require 'dry-view'

class UserIndexView < Dry::View::Controller
  configure do |config|
    config.paths = [File.join("templates")]
    config.template = "index"
  end

  expose :users
end
{% endhighlight %}

And create `index.html.slim` template. If we name it `index.slim`, it wouldn't be picked up by `tilt`, which dry-view
uses underneath for the rendering purposes, because by default it relies on the file name structure consisting of three parts.

{% highlight ruby %}
# templates/index.html.slim
# contents are the same as in the original template
#...
{% endhighlight %}

And this simply works.

So why in the world one need to use dry-view? Even in this simplest example, it can become obvious.

##### 1. Tidy up the routes

We don't need the user fetching logic in the routes anymore - it can go into view object. The only thing we need to provide
to object when using `view.call` is the input parameters from URL of forms. In our example users fetching logic goes
completely inside the view object.

{% highlight ruby %}
# views/user_index_view.rb
private

def users
  (1..10).map do |a|
    { id: a, name: Faker::Name.name, occupation: Faker::Job.title }
  end
end
{% endhighlight %}

And Sinatra route becomes only:

{% highlight ruby %}
# app.rb
get '/' do
  UserIndexView.new.call
end
{% endhighlight %}

##### 2. Increase testability

Dry-view approach not only removes logic from routing layer(in case of Sinatra or Roda) but also cleans up templates
and thus the view logic becomes much more testable. Remember the testing of Rails views or imagine a testing process of logic
embedded in Sinatra routing - both are a pain. When dealing with functional objects, we can use a unit-test approach or
test class's outcome with given input. For example, if we assume, that `user` objects are coming from the database:

{% highlight ruby %}
require 'rspec'
require_relative 'views/user_index_view'

describe UserIndexView do
  it 'exposes users' do
    view_object = described_class.new.call
    User.take(10) do |user|
      expect(view_object)..to include(user.occupation)
    end
  end
end
{% endhighlight %}

Dry-view has also another notable use cases, and the most important of them is that it perfectly plays with dependency injection
scenarios, but this is beyond the scope of the article.

Suggested reading:

[Dry-View](http://dry-rb.org/gems/dry-view/)
