---
layout: default
title:  " How to describe issues in a clear way"
date:   2017-10-02 13:27:12 +0300
category: workflow
---

# How to describe issues in a clear way

Often, when approaching coding problems, we face some issues with a framework or some libraries. First, we try to solve
them ourselves, digging into the source code, then begin to ask questions on GitHub, related Slack channels, searching
StackOverflow and what not. Usually, I notice that many developers ask that kind of questions in a manner, that makes
hard to understand, what exactly is going on. In such cases, there is often even no way to reproduce the described bug.
So, let's look at an example. Let's imagine, that we have some web application, written using Ruby on Rails, backed by
ActiveRecord ORM. We tried to implement image uploads using [Shrine][shrine] library, and something just does not work
as expected. How do we make the description of this situation understandable and give readers the possibility to reproduce
described behavior?

#### We can point them directly to our repository

so they can clone it, launch project and make suggestions. This approach has two main drawbacks:
* cloning and setting up a project can take amounts of time
* the project may live in a private repository, making the project inaccessible for third-party
Of course, these two are not the case when you asking from your teammates, with which you are currently working on the
described project.

#### We can setup everything in just a single file

The file will load only needed libraries, and replace some external services by their mocks.
So this is the example of the second approach.

{% highlight ruby %}
# load activerecord, Shrine, mini-magick
require "active_record"
require "shrine"
require "shrine/storage/file_system"
require "image_processing/mini_magick"

# use in-memory sqlite instead of postgresql
# or other disk-persisting solution
ActiveRecord::Base.establish_connection(adapter: "sqlite3",
                                        database: ":memory:")

ActiveRecord::Migration.create_table(:articles) do |t|
  t.text :picture_data
end

# set up Shrine
Shrine.storages = {
  cache: Shrine::Storage::FileSystem.new("uploads/cache"),
  store: Shrine::Storage::FileSystem.new("uploads/store"),
}

Shrine.plugin :activerecord
Shrine.plugin :pretty_location
Shrine.plugin :remote_url, max_size: 10*1024*1024

# set up uploader
class PictureUploader < Shrine
  include ImageProcessing::MiniMagick

  def generate_location(io, _context)
    "pictures/#{super}"
  end
end

# our model
class Article < ActiveRecord::Base
  include PictureUploader[:picture]
end

# here we are "testing" file upload
article = Article.create(picture_remote_url:
                         "http://placehold.it/300x300.jpg")

# and here conclusions go
p article.image_data
{% endhighlight %}

In the given examples we loaded all needed libraries manually, set up in-memory SQLite database, set our uploader
and model and then demonstrated image upload. There are really no any issue or bug, but that way we can describe
most of the problems that we face during the development process. We can even add a testing library to this setup,
but in this case, everything is clear and reproducible even without minitest or rspec.


[shrine]: http://shrinerb.com/
