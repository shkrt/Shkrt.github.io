---
layout: default
title:  "ROM-rb overview"
date:   2017-12-11 09:57:51 +0300
tags: ruby
---

# ROM-rb overview

ROM, the Ruby Object Mapper is a set of libraries developed by the same people who also authored the famous Dry-rb gems.
ROM, for example, used as a part of database-persistence and model layers in Hanami framework. It's backed by Sequel, which
performs lower level database interaction operations, but unlike Sequel and ActiveRecord, ROM is not simply another ORM.
The key concept of ROM is a separation of database-persistence logic from domain model logic and it has its roots in the
concept of domain-driven design for software applications. Being used to MVC frameworks, many of us have seen such an approach
to building an application, where one class performs all the business logic, all the database-related logic, and even form validations.
When applications grow, we tend to refactor these bloated classes, extracting form objects, validation objects and so on.
ROM also makes possible to completely decouple database-related actions from business logic. Of course, ROM is not an only
database-related thing. It also can be used to work with any data source, even some arbitrary structured files. ROM provides adapter abstraction,
that makes possible to write our own adapter for any possible data source.

### Example: use ROM to map to already existing legacy database

#### Getting up and running

Assuming we have a database with the following structure:

{% highlight sql %}
postgres@test_db=#  \dt
            List of relations
 Schema │    Name     │ Type  │  Owner
════════╪═════════════╪═══════╪══════════
 public │ authors     │ table │ postgres
 public │ books       │ table │ postgres
 public │ reviews     │ table │ postgres
 public │ users       │ table │ postgres
(5 rows)

authors: id, first_name, last_name, birthdate
books: id, title, release_year, author_id
reviews: id, user_id, book_id, text, rate
users: id, first_name, last_name
{% endhighlight %}

One author has many books, each book has many reviews, each review belongs to a user.

We can create mappings for this database using ROM.

First, the connection:

{% highlight ruby %}
# test.rb
require 'rom'
require 'rom-sql'

# connection string
rom = ROM.container(:sql, 'postgres://postgres:qwerty@localhost/roda_app_development') do |config|
  # define relations
  class Users < ROM::Relation[:sql]
    # infer schema from database
    schema(infer: true) do
      # override inferred schema by adding custom associations
      associations do
        has_many :reviews
      end
    end
  end

  class Books < ROM::Relation[:sql]
    schema(infer: true) do
      associations do
        has_many :reviews
        belongs_to :author
      end
    end

    # define query methods
    def for_authors(_assoc, authors)
      where(author_id: authors.map { |a| a[:id] })
    end

    def with_reviews
      combine(:reviews)
    end
  end

  class Reviews < ROM::Relation[:sql]
    schema(infer: true) do
      associations do
        belongs_to :book
      end
    end
  end

  class Authors < ROM::Relation[:sql]
    schema(infer: true) do
      associations do
        has_many :books
      end
    end

    def with_reviews
      combine(:reviews)
    end
  end

  # register defined relations
  config.register_relation(Users, Books, Reviews, Authors)
end
{% endhighlight %}

The following is not mandatory, I just assigned all relations to instance variables inside the ruby file, to make work with
the example easier:

{% highlight ruby %}
@books = rom.relations[:books]
@reviews = rom.relations[:reviews]
@authors = rom.relations[:authors]
@users = rom.relations[:users]
{% endhighlight %}

And now we can load this file from `pry` using

```
load 'files/test.rb'
```

And we have `@books`, `@reviews`, `@authors`, `@users` relations already available.

Now we can query the database for needed data:

{% highlight ruby %}
> @authors.where(id: 4).with_books.to_a
# there is no author with id: 4
=> []

> @authors.where(id: 1).with_books.to_a
# there are no books in our database for this particular author
=> [{:id=>1,
     :first_name=>"Charles",
     :last_name=>"Bukowski",
     :birthdate=>#<Date: 1920-08-16 ((2418405j,0s,0n),+0s,2299161j)>}]
     :books=>[]}]

# fetch by primary key
> @reviews.by_pk(1)
=> [{:id=>1, :user_id=>55, :book_id=>4, :text=>'Yrev very nice book by the way', :rate=>5}]

# fetch only associations
> @books.for_authors(@authors.associations[:books],@authors.where(id: 2)).to_a
=> [{:id=>5, :title=>"Ask the Dust", :release_year=>1939, :author_id=>2},
    {:id=>6, :title=>"The Road to Los Angeles", :release_year=>1936, :author_id=>2}]
{% endhighlight %}

Looks pretty neat. Using simple and intuitive DSL, we've set up database mapping, associations and query methods, and are
ready to work with data.

#### Commands concept

ROM uses commands to make changes in database, i.e. to perform `create`, `delete` and `update` operations. Command is an object.
As any object, we can instantiate it for later use.

{% highlight ruby %}
create_user = @users.command(:create)
create_user.(first_name: 'João', last_name: 'Gilberto')
=> {:id=>56, :first_name=>"João", :last_name=>"Gilberto"}

update_user = @users.by_pk(1).command(:update)
update_user.(first_name: 'Antonio', last_name: 'Jobim')
{% endhighlight %}

Commands are simple abstraction provided by ROM and are a foundation for more abstract one - changeset

#### Changesets concept

Changesets also serve for making changes in the database, but being built on top of commands, they provide more advanced features,
such as custom mapping, associating data. I won't touch these topics in this post, and you can find a brief description of them
in the documentation.

{% highlight ruby %}
author = @authors.changeset(:create, first_name: 'Howard', last_name: 'Lovecraft', birthdate: '1890-08-20')
author.commit
{% endhighlight %}

#### Repositories concept

Repository provides a set of abstractions to read and write complex data. It can be viewed as an implementation of repository pattern from
domain-driven design concepts and may sound familiar to those who acquainted with Hanami or Phoenix frameworks. The principles used in
repositories of both are the same, and ROM repositories are basically similar.

{% highlight ruby %}
class BookRepo < ROM::Repository[:books]
  commands :create, update: :by_pk, delete: :by_pk

  def titles
    books.pluck(:title)
  end

  def by_title(title)
    books.where(title: title)
  end
end

@book_repo = BookRepo.new(rom)
@book_repo.delete(9)
@book_repo.titles

# repositories provide data in Structs by default
@book_repo.by_title('Beyond the Wall of Sleep').one!.title
=> 'Beyond the Wall of Sleep'
{% endhighlight %}

Repositories can be used to perform complex data insertions:

{% highlight ruby %}
class AuthorRepo < ROM::Repository[:authors]
  commands :create, update: :by_pk, delete: :by_pk

  def create_with_books(author)
    authors.combine(:books).command(:create).(author)
  end
end

@author_repo = AuthorRepo.new(rom)

> @author_repo.create_with_books(first_name: 'Knut',
                                 last_name: 'Hamsun',
                                 books: [{title: 'Sult', release_year: 1890},
                                         {title: 'Mysterier', release_year: 1892}]

# which, as expected, creates an author with books
{% endhighlight %}

---

ROM can even be used in Rails application, for example, if you plan to refactor your app using domain-driven design principles.
When used inside Rails, it can be either run alongside ActiveRecord, or completely replace it, and offer a possibility to connect to multiple databases.

Suggested reading:

[ROM website](http://rom-rb.org)
