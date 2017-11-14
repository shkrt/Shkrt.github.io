---
layout: default
title:  "Comparing Sequel and ActiveRecord"
date:   2017-11-06 18:33:09 +0300
tags: ruby sequel activerecord
---

# Comparing Sequel and ActiveRecord

[Sequel](http://sequel.jeremyevans.net) is a database ORM for Ruby applications, authored by Jeremy Evans, also an author of Roda.
Many modern web application building toolkits, such as Hanami, Phoenix, Dry-Web etc. tend to stay away from ActiveRecord for different reasons,
and most of them do not utilize even active record pattern. Sequel from this point of view can be considered as an alternative ORM,
using the same active record pattern, as the ActiveRecord itself does. Looking at the Sequel's documentation, transition from ActiveRecord to
Sequel should not be that difficult.

This is how our ActiveRecord setup looks like:

{% highlight ruby %}
require "active_record"
require "faker"

ActiveRecord::Base.establish_connection(adapter: "sqlite3",
                                        database: ":memory:")

ActiveRecord::Migration.create_table(:writers) do |t|
  t.string :full_name
end

ActiveRecord::Migration.create_table(:books) do |t|
  t.integer :writer_id
  t.string :title
end

ActiveRecord::Migration.create_table(:genres) do |t|
  t.string :title
end

ActiveRecord::Migration.create_table(:book_genres) do |t|
  t.integer :book_id
  t.integer :genre_id
end

class Writer < ActiveRecord::Base
  has_many :books
end

class Book < ActiveRecord::Base
  belongs_to :writer
  has_many :book_genres
  has_many :genres, through: :book_genres
end

class BookGenre < ActiveRecord::Base
  belongs_to :book
  belongs_to :genre
end

class Genre < ActiveRecord::Base
  has_many :book_genres
  has_many :books, through: :book_genres
end

Writer.create(full_name: Faker::Name.name)

2.times do
  Book.create(title: Faker::Book.title, writer: Writer.last)
  Genre.create(title: Faker::Book.genre)
end

Book.first.genres = Genre.all
Book.last.genres << Genre.last

{% endhighlight %}

Let's achieve the same functionality via Sequel:

{% highlight ruby %}
require "sequel"
require "faker"

DB = Sequel.sqlite

DB.create_table :writers do
  primary_key :id
  String :full_name
end

DB.create_table :books do
  primary_key :id
  String :title
  foreign_key(:writer_id)
end

DB.create_table :genres do
  primary_key :id
  String :title
end

DB.create_table :books_genres do
  primary_key :id
  foreign_key(:book_id)
  foreign_key(:genre_id)
end

class Writer < Sequel::Model
  one_to_many :books
end

class Book < Sequel::Model
  many_to_one :writer
  many_to_many :genres
end

class Genre < Sequel::Model
  many_to_many :books
end

Writer.create(full_name: Faker::Name.name)

2.times do
  Book.create(title: Faker::Book.title, writer: Writer.last)
  Genre.create(title: Faker::Book.genre)
end

Book.last.add_genre(Genre.last)

{% endhighlight %}

It seems like migrating a simple ActiveRecord application to sequel won't cause many problems.

Let's take a look at some of the mostly used queries:

|AR|Sequel|SQL(AR)|SQL(Sequel)|
|---|---|---|
| `Book.find(1)` | `Book[1]` | `SELECT "books"."*" FROM "books" WHERE "books"."id" = 1 LIMIT 1` | `SELECT * FROM "books" WHERE ("id"=1)`|
| `Book.last` | `Book.last` | `SELECT "books"."*" FROM "books" ORDER BY "books"."id" DESC LIMIT 1` | `SELECT * FROM "books" ORDER BY "id" LIMIT 1` |
| `Writer.includes(:books)` | `Writer.eager(:books)`| `SELECT "writers"."*" FROM "writers"` `SELECT "books"."*" FROM "books" WHERE "books"."writer_id" IN (1)`| |
| `Book.select(:id)` | `Book.select(:id)` | `SELECT "books"."id" FROM "books"` | `SELECT "id" FROM "books"`|
| `Writer.joins(:books)` | `Writer.association_join(:books)` | `SELECT "writers"."*" FROM "writers" INNER JOIN "books" ON "books"."writer_id" = "writers"."id"` | `SELECT * FROM "writers" INNER JOIN "books" ON ("books"."writer_id" = "writers"."id")`|
| `Writer.joins('LEFT JOIN books ON books.writer_id = writers.id')` | `Writer.aasociation_left_join(:books)` | `SELECT "writers"."*" FROM "writers" LEFT JOIN "books" ON "books"."writer_id" = "writers"."id"` |`SELECT * FROM "writers" LEFT JOIN "books" ON ("books"."writer_id" = "writers"."id")` |
| `Writer.order(full_name: :desc)` | `Writer.reverse_order(:full_name)` | `SELECT "writers"."*" FROM "writers" ORDER BY "writers"."full_name" DESC` | `SELECT * FROM "writers" ORDER BY "writers"."full_name" DESC` |

As we see, Sequel's API offers very convenient interface for ordering, eager loading, joining and many more typical orm-related tasks. To actually use Sequel with Rails
instead of ActiveRecord, you will have to use [sequel-rails](https://github.com/TalentBox/sequel-rails) gem. If you're not in need of full orm support, including generators, migrations, or if
you just want to do some experiments in command line, you can inherit existing Rails models from Sequel::Model, and set up all associations from scratch. That way you can also migrate
your application part by part, iteratively changing connected models to Sequel::Model.

Suggested reading:

[Sequel official documentation](http://sequel.jeremyevans.net/)

[Ode to Sequel](https://twin.github.io/ode-to-sequel/)

[Why you should stop using ActiveRecord and start using Sequel](https://mrbrdo.wordpress.com/2013/10/15/why-you-should-stop-using-activerecord-and-start-using-sequel/)
