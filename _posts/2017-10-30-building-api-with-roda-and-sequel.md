---
layout: default
title:  "Building API with Roda and Sequel"
date:   2017-10-30 18:33:09 +0300
category: ruby
---

# Building API with Roda and Sequel

Both [Roda](https://github.com/jeremyevans/roda) and [Sequel](https://github.com/jeremyevans/sequel) are authored by
[Jeremy Evans](https://github.com/jeremyevans). First is described as 'Routing Tree Web Toolkit' and offers web framework
experience without database persistence layer, which is, for example, can be provided by aforementioned Sequel.
If you're coming from Rails world, you might have been used to ActiveRecord as database ORM, and taking a glance at Sequel
is a must for you. Otherwise, you perhaps have used Sequel in Sinatra projects. It's also worth noting that both projects have zero issues on GitHub.

Let's start simple. Jeremy Evans has carefully prepared repo [roda-sequel-stack](https://github.com/jeremyevans/roda-sequel-stack) which contains
basically everything we need to have simplest web app up and running. I'm repeating first steps from the repo docs:

```
$ git clone https://github.com/jeremyevans/roda-sequel-stack
$ mv roda-sequel-stack my_app
$ cd my_app
$ rake setup[MyApp]
```

Then you should perform database creation manually, or create simple rake task for it:

{% highlight ruby %}
desc "database-related tasks"
namespace :db do
  desc "create database"
  task :create do
    logger.info("creating database")
    %x( createdb -E UTF8 -T template2 -U postgres --password -h localhost roda_app_development )
    logger.info("successfully created database")
  end
end
{% endhighlight %}

Database connection string is meant to be stored in `.env` file:

{% highlight ruby %}
ENV['DATABASE_URL'] = "postgres://postgres:qwerty@localhost/my_app_development
{% endhighlight %}

This will be automatically picked up at application start.

To start a blank application, you can just add the following to `config.ru` and then execute `rackup` from console:

{% highlight ruby %}
require "roda"

Roda.route { "Hello world!" }
run Roda.app
{% endhighlight %}

But suggested setup from `roda-sequel-stack` repository proposes better way to organize code, and we move framework code to `app.rb`,
and leave `config.ru` with [following code](https://github.com/jeremyevans/roda-sequel-stack/blob/f80298eaf726846d48f53bc50b908a371a90101d/config.ru)

Now we can add first routes into `app.rb`

{% highlight ruby %}
# app.rb
require_relative "models"
require "roda"

class App < Roda
  plugin :json, classes: [Array, Hash, Sequel::Model]

  route do |r|
    r.is 'books' do
      @books = Book.all
      @books.map(&:values).to_json
    end
  end
end
{% endhighlight %}

Of course, this piece of code contains not only routes but also some database querying and serializing logic.
Some important points:

* at line 6 we plugging in the `json` plugin, which is needed to return JSON responses.
* at line 8 we start matching routes. Matching works from top to bottom, and in this case, all requests to `/books` URL
will be matched.
* when matched, respective block is evaluated. Here we're setting up the @books instance variable by querying a database
for all Book records and then returning all the records in JSON format.

Of course, this would not work yet, because we have neither `Book` model, nor `books` table. So, let's create them.

{% highlight ruby %}
# migrate/001_create_books.rb
Sequel.migration do
  up do
    create_table(:books) do
      primary_key :id
      String :title, null: false
      String :image_data, text: true
      Integer :release_year
    end
  end

  down do
    drop_table(:books)
  end
end
{% endhighlight %}

This is pretty straightforward and does not need explanation, I think.

The model does not contain much code by now:

{% highlight ruby %}
# models/book.rb
class Book < Sequel::Model
end
{% endhighlight %}

Run `rackup` now, make GET request to our app's `/books` URL, and you should see an empty response. If you don't like it
empty, of course, Book records can be created, for example, by running `rake dev_irb`

```
$ curl localhost:9292/books
$ {"books":[[2,"Book title",null,2000]]
```

Let's go further and implement other CRUD actions.

{% highlight ruby %}
# app.rb
plugin :json_parser
plugin :all_verbs
plugin :halt
{% endhighlight %}

We have added three new plugins to app. 
* `json_parser` needed to parse incoming JSON requests because we need to serve
POST/PUT/PATCH requests that may contain JSON payload.
* `all_verbs` will add support for PUT/PATCH/DELETE/OPTIONS requests.
* `halt` needed to return from block earlier, i.e. when some error occurred.

{% highlight ruby %}
route do |r|
  # /books used to match get and post verbs, to provide respectively
  # books index and and book creation
  r.is 'books' do
    r.get do
      page = r.params[:page] || 1
      { books: Book.paginate(page, 20).map(&:values) }
    end

    r.post do
      @book = Book.create(book_params(r))
      { book: @book.values }
    end
  end

  r.is 'book', Integer do |book_id|
    # book/:id used to match get, put and delete request, to provide
    # getting book by id, updating and deleting book
    @book = Book[book_id]
    # use halt to return 404 without evaluating rest of the block
    r.halt(404) unless @book

    r.get do
      { book: @book.values }
    end

    r.put do
      @book.update(book_params(r))
      { book: @book.values }
    end

    r.delete do
      @book.destroy
      response.status = 204
      {}
    end
  end
end

private

def book_params(r)
  { release_year: r.params['release_year'], image_data: r.params['image_data'], title: r.params['title'] }
end
{% endhighlight %}

`book_params` method is a shortcut to DRY out parameters used for record changing.
Also, you may have noticed pagination was added at line 7
This is not yet implemented in the model, so let's add it:

{% highlight ruby %}
# models/book.rb
class Book < Sequel::Model
  class << self
    def paginate(page_no, page_size)
      ds = DB[:books]
      ds.extension(:pagination).paginate(page_no, page_size)
    end
  end
end
{% endhighlight %}

`pagination` is a Sequel extension, providing offset + limit pagination.

Now we have a very basic API application, that gives us access to one of our application's tables via very trivial CRUD interface.
We've almost built it from scratch, and though the sample setup from Jeremy Evans repo has been used, it is very simple and we can track every
performed action and break down an application into parts. This looks very advantageous when compared to frameworks providing some kind of
generators, where you're not controlling each aspect of app building. Also, there are no tons of middlewares, bloating and slowing down the request-response cycle (though
you can feel suspicious for plugins, they're nothing compared to Rails middlewares). In my opinion, the best option for one to try
roda-sequel stack without going all the way without Rails is to plug Roda API to Rails app via Rails's `config/routes.rb` file. If you had not known,
Rails is capable of mounting any Rack app within its routing tree, so Roda, being Rack-based, is perfectly fit for such use-cases. Using Sequel
instead of ActiveRecord is also an option, but this would be a subject of another article, I hope.
