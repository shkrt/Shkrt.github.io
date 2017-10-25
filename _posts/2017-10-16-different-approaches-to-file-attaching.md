---
layout: default
title:  "Different approaches to file attaching"
date:   2017-10-16 18:39:41 +0300
category: ruby
---

# Different approaches to file attaching

This time I'm making an overview of different approaches to file attaching problem, that we face in most applications. Let's begin by looking at the popular gems and how this problem is solved in these gems' codebases.

### Paperclip

For non-Rails usage, we should mix in `Paperclip::Glue` (for Rails it's mixed in during autoload), which itself mixes in `Paperclip::Callbacks`, `Paperclip::Schema` and `Paperclip::Validators`, each of which makes available, respectively, callbacks, database_column projection methods, and validators [(Github)](https://github.com/thoughtbot/paperclip/blob/8f7f29fc109c0f1c9189ca64e7d412e8f96c761d/lib/paperclip/glue.rb).
In [Paperclip::Schema](https://github.com/thoughtbot/paperclip/blob/8f7f29fc109c0f1c9189ca64e7d412e8f96c761d/lib/paperclip/schema.rb)
we see definition of four columns, that will be used to store attachment's properties:

{% highlight ruby %}
COLUMNS = { :file_name    => :string,
            :content_type => :string,
            :file_size    => :integer,
            :updated_at   => :datetime }
{% endhighlight %}

Then the main work is done in `Paperclip::HasAttachedFile`, which defines all needed methods using lots of metaprogramming.
Our model has now four attributes, that store attached file's metadata (to actually display attached file, we need only `:file_name`). Also, we have meta-programmatically defined methods to add and remove attachments, drop an attached file.
As a backend, `ImageMagick` and `file` utilities are used to manipulate images and checking content-types.

### Carrierwave

This one is the second most popular file attachment solution. It uses the concept of uploaders - special classes, providing overrides of default behavior for each class, using special `mount_uploader` and `mount_uploaders` macro.
In [CarrierWave::Mount module](https://github.com/carrierwaveuploader/carrierwave/blob/16c0eb5a7869d49930e165bc6a519033f067e9c6/lib/carrierwave/mount.rb), we see a bunch of methods being defined using `class_eval`.
The second argument to `mount_uploader` is the uploader class name, which maps lots of methods to the model, making available adding and deleting attachments, as well as other operations.
That way, when we refer to our image attribute, we've got deserialized uploader class:

{% highlight ruby %}
def #{column}
  _mounter(:#{column}).uploaders[0] ||= _mounter(:#{column}).blank_uploader
end
{% endhighlight %}

Then we use uploader class to override default methods.
Just like Paperclip, Carrierwave processes files in the upload callbacks, using ImageMagick as a backend for image processing, but Carrierwave also has an adapter layer, which is represented by RMagick or MiniMagick libraries.

### Dragonfly

This gem introduces the concept of separate rack app that is used to process file requests.
For Rails app, we can start by adding `dragonfly_accessor :photo` macro to model class. As we have seen from previous examples, such macros usually used to define various accessor methods using meta-programming
[(Github)](https://github.com/markevans/dragonfly/blob/b8af810e647fc21e43ccc42b69beb6c9baa40abe/lib/dragonfly/model/class_methods.rb#L25).
It also stores attached file attributes in a database, therefore, respective methods are [used](https://github.com/markevans/dragonfly/blob/b8af810e647fc21e43ccc42b69beb6c9baa40abe/lib/dragonfly/model/attachment.rb#L206):

{% highlight ruby %}
def model_uid=(uid)
  model.send("#{attribute}_uid=", uid)
end
{% endhighlight %}

There is no definition for that methods, gem relies on column names. When accessing images, Dragonfly's backend server processes requests, deserializes model's attachment attribute and directs further processing to ImageMagick (in case of images).

### Refile

This library also relies on a separate app for processing uploads. It mounts Sinatra application inside the host app:
{% highlight ruby %}
if Refile.automount
  Rails.application.routes.draw do
    mount Refile.app, at: Refile.mount_point, as: :refile_app
  end
end
{% endhighlight %}

Then goes already familiar concept of a macro:

`attachment :profile_image`

which also serves as accessor method definition [helper](https://github.com/refile/refile/blob/master/lib/refile/attachment.rb#L39).

Accessing attachments results in a request to Sinatra app. Request url is constructed by deserializing attachment's [metadata](https://github.com/refile/refile/blob/36646017d183239b898b21dbd443f2ed6799d088/lib/refile.rb#L309):

{% highlight ruby %}
def file_url(file, *args, host: nil, prefix: nil, filename:, format: nil)
  return unless file
  host ||= Refile.cdn_host
  backend_name = Refile.backends.key(file.backend)
  filename = Rack::Utils.escape(filename)
  filename << "." << format.to_s if format
  base_path = ::File.join("", backend_name, *args.map(&:to_s), file.id.to_s, filename)
  ::File.join(app_url(prefix: prefix, host: host), token(base_path), base_path)
end
{% endhighlight %}

The `token` method is used to protect request url from flood attacks by adding a unique hash to every attachment's url. Hash is simply constructed from combining arbitrary secret key with file path

Refile does not store processed versions of uploaded files - this task is simply outsourced to a content delivery network.
Approach to defining custom processing options is also interesting - it is supposed to be done via monkey-patching using initializer files.

### Shrine

This one is my personal favorite, because of its being actively maintained and supporting some ORMs besides ActiveRecord. It also has its own unique concept of plugins, each of which serves some particular purpose.
File data is as usual stored in database text field. To make this data accessible in our app, we mix in uploader class into the model:
{% highlight ruby %}
include ImageUploader::Attachment.new(:image)
{% endhighlight %}
`ImageUploader` in this case is a subclass of `Shrine` class, which in turn leads us to another meta-programming defined accessor [methods](https://github.com/janko-m/shrine/blob/master/lib/shrine.rb#L390).
Shrine plugins are just modules, adding new methods or overriding existing ones. For example, [processing](https://github.com/janko-m/shrine/blob/master/lib/shrine/plugins/processing.rb) plugin.
Here we see that custom entries are being added in `uploader.opts` hash:
{% highlight ruby %}
def self.configure(uploader)
  uploader.opts[:processing] = {}
end
{% endhighlight %}

And then `process` method calls given action on the io object:

{% highlight ruby %}
def process(io, context = {})
  pipeline = opts[:processing][context[:action]] || []
  result = pipeline.inject(io) do |input, processing|
    instance_exec(input, context, &processing) || input
  end
  result unless result == io
end
{% endhighlight %}

The processing logic is simply wrapped in a block and defined by the user in uploader class.

### Attache

This solution also relies on dynamically processing uploads. But this also goes further and provides scalability by deploying processor app onto a separate server.
There are no any ORM bindings, neither model accessor methods. Attache simply provides file processing operations and storing files (or promoting them to external storage). Let's look. for example, at `Attache::Upload` [class](https://github.com/choonkeat/attache/blob/master/lib/attache/upload.rb).
Received file is simply being written into the cache, which, in this case, is disk.

{% highlight ruby %}
Attache.cache.write(cachekey, request.body)
# ...
[200, config.headers_with_cors.merge('Content-Type' => 'text/json'), [json_of(relpath, cachekey, config)]]
{% endhighlight %}

Attache app just writes the unprocessed file on the disk and then returns its metadata, whether it would be disk or some other storage. Storing of metadata and attaching it to the model
is user's responsibility.
Processing itself is performed while the file is being downloaded [(Github)](https://github.com/choonkeat/attache/blob/master/lib/attache/download.rb).
In this case even background jobs are involved. Interesting point was that the Attache uses Paperclip internally as image processing proxy to ImageMagick.

So, as a conclusion for this overview, we can outline following points:
* File upload solution consists of three responsibilities: processing the uploaded file, storing its metadata and attaching this data to our model class.
* Processing uploads can be done by introducing separate web-server or directly during file upload, synchronously or in background.
* As we see, most of the file uploading solutions tend to separate out this three responsibilities and some of them, like Attache, even provide separate deployment option out-of-box. This is a must for the sake of scalability.
* The most popular pattern to provide accessor interface to model's uploaded files is defining some macro, that internally defines all needed accessor methods and then use special classes designed to give the user access to customize some accessor methods. These classes perform serializing and deserializing of stored attributes.
