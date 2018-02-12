---
layout: default
title:  "My approach to refactoring fat Rails controllers"
date:   2018-02-12 21:06:59 +0300
comments: true
tags: rails
---

# My approach to refactoring fat Rails controllers

The problem of Rails controller actions growing bigger over time has been discussed many times, and refactoring
controller actions is not a secret for anyone. I want to share my approach, which combines a few approaches I have
learned from various developers with some of my personal researches.

Basically, every refactoring of controller action consists of only one significant step - 'Extract service object'.
In my opinion, service objects are applicable when the extracted code is responsible for interaction with third parties,
background queue managers, i.e performing some actions that have either success or failure status. The concept of
interactors is perfectly applicable here.

My approach differs from the conventional service object approach because, in my controllers, service objects are supposed
to return another object, and I already have seen some articles on the web, where people use "Result object" term, but
I prefer to call it "View object". View object has nothing to do with [cells](http://trailblazer.to/gems/cells/),
it consists of local variables, template and layout names, and status codes.

Here are the steps to refactor:

1. Inline controller callbacks

2. Change all instance variables to local variables

3. Extract all logic, except cookies/session manipulation and redirects to separate class

4. In newly created separate class, split querying logic into descriptive methods

5. Return object, responding to `status`, `locals`, `template`, `layout` methods, the last two are very optional

My credits to Arkency's Andrzej Krzywda, because I got some ideas from his brilliant
[Fearless Refactoring: Rails Controllers](http://rails-refactoring.com/) book, and to Ivan Shamatov, which has written
[this](https://mkdev.me/posts/kak-ispolzovat-query-objects-dlya-refaktoringa-sql-zaprosov-rails) inspiring article in Russian.

Let's start with example controller:

{% highlight ruby %}
class Desktop::OpinionsController < DesktopController
  ARTICLES_AMOUNT = 7
  RELATED_RATING_AMOUNT = 4

  before_action :apply_filter_cookies, only: :index

  def index
    @opinions = Opinion.published
                       .by_tenant(current_tenant)
                       .order(published_at: :desc)
                       .filter(params.slice(:by_columnist, :date, :period))
                       .offset(current_page * ARTICLES_AMOUNT - ARTICLES_AMOUNT)
                       .limit(ARTICLES_AMOUNT)
                       .includes(:columnist)
                       .map { |x| OpinionPresenter.new(x) }

    raise ActiveRecord::RecordNotFound if @opinions.empty?

    @partners = current_tenant.partners.map { |x| PartnerPresenter.new(x) }
    @person = Person.published
                    .by_tenant(current_tenant)
                    .where.not(is_preview: true)
                    .friendly.find(params[:id])
    @related_ratings: Rating.related_to(@person)
                            .select(:id, :rubric, :title, :slug, :published_at, :published_date, :status, :type)
                            .limit(RELATED_RATING_AMOUNT)

    render layout: 'desktop_special', template: 'desktop/articles'
  end

  private

  def apply_filter_cookies
    # some code, that has been omitted
  end
end

{% endhighlight %}

This example controller seems not to be very complicated, but it's fat enough and should be refactored.

## Inline controller callbacks

{% highlight ruby %}
class Desktop::OpinionsController < DesktopController
  ARTICLES_AMOUNT = 7
  RELATED_PERSON_AMOUNT = 4

  def index
    apply_filter_cookies

    # ...
  end
{% endhighlight %}

We simply remove before_action line and made it inline, inserting respective method call in the controller action body.

## Change all instance variables to local variables

{% highlight ruby %}
def index
  opinions = Opinion.published
                    .by_tenant(current_tenant)
                    .order(published_at: :desc)
                    .filter(params.slice(:by_columnist, :date, :period))
                    .offset(current_page * ARTICLES_AMOUNT - ARTICLES_AMOUNT)
                    .limit(ARTICLES_AMOUNT)
                    .includes(:columnist)
                    .map { |x| OpinionPresenter.new(x) }

  raise ActiveRecord::RecordNotFound if opinions.empty?

  partners = current_tenant.partners.map { |x| PartnerPresenter.new(x) }
  person = Person.published
                 .by_tenant(current_tenant)
                 .where.not(is_preview: true)
                 .friendly.find(params[:id])
  related_ratings: Rating.related_to(person)
                         .select(:id, :rubric, :title, :slug, :published_at, :published_date, :status, :type)
                         .limit(RELATED_RATING_AMOUNT)

  render layout: 'desktop_special', template: 'desktop/articles',
         locals: {
           opinions: opinions,
           partners: partners,
           person: person,
           related_ratings: related_ratings
         }
end
{% endhighlight %}

You also have to change all instance variables in the templates to the local ones, but this step thoroughly described
in Fearless Refactoring book, so I would not touch it here.

## Extract all logic, except cookies/session manipulation and redirects to separate class

For this step I prefer the following naming convention: make a separate directory for this kind of classes under `app/services`.
Then namespace all the classes exactly like they are namespaced in respective controllers, for this example,
the full path to the file will be `app/services/view_objects/desktop/opinions/index.rb`

{% highlight ruby %}
module ViewObjects
  module Desktop
    module Opinions
      class Index
        ARTICLES_AMOUNT = 7
        RELATED_PERSON_AMOUNT = 4

        attr_accessor :template, :layout, :locals, :status
        def initialize(tenant, page, params)
          @page = page
          @tenant = tenant
          @params = params
        end

        def call
          opinions = Opinion.published
                            .by_tenant(@tenant)
                            .order(published_at: :desc)
                            .filter(@params.slice(:by_columnist, :date, :period))
                            .offset(@page * ARTICLES_AMOUNT - ARTICLES_AMOUNT)
                            .limit(ARTICLES_AMOUNT)
                            .includes(:columnist)
                            .map { |x| OpinionPresenter.new(x) }

         raise ActiveRecord::RecordNotFound if opinions.empty?

         partners = @tenant.partners.map { |x| PartnerPresenter.new(x) }
         person = Person.published
                        .by_tenant(@tenant)
                        .where.not(is_preview: true)
                        .friendly.find(@params[:id])
         related_ratings: Rating.related_to(person)
                                .select(:id, :rubric, :title, :slug, :published_at, :published_date, :status, :type)
                                .limit(RELATED_RATING_AMOUNT)

          self.locals = {
            opinions: opinions,
            partners: partners,
            person: person,
            related_ratings: related_ratings
          }

          self.layout = 'desktop_special'
          self.template = 'desktop/articles'
          self
        end

      end
    end
  end
end
{% endhighlight %}

And controller becomes only this:

{% highlight ruby %}
class Desktop::OpinionsController < DesktopController
  def index
    apply_filter_cookies
    view_object = ViewObjects::Desktop::Opinions::Index.new(@current_tenant, @current_page, params).call
    render layout: view_object.layout, template: view_object.template
  end

  private

  def apply_filter_cookies
    # some code, that has been omitted
  end
end

{% endhighlight %}

Much better, but we have one bloated class for another, and this is the time for the next step.

## In newly created separate class, split querying logic into descriptive methods

{% highlight ruby %}
module ViewObjects
  module Desktop
    module Opinions
      class Index
        ARTICLES_AMOUNT = 7
        RELATED_PERSON_AMOUNT = 4

        attr_accessor :template, :layout, :locals, :status
        def initialize(tenant, page, params)
          @page = page
          @tenant = tenant
          @params = params
        end

        def call
          self.locals = find_locals
          self.layout = 'desktop_special'
          self.template = 'desktop/articles'
          self
        end

        private

        def find_locals
          person = find_person
          {
            opinions: opinions,
            partners: partners,
            person: person,
            related_ratings: related_ratings(person)
          }
        end

        def opinions
          opinions = Opinion.published
                            .by_tenant(@tenant)
                            .order(published_at: :desc)
                            .filter(@params.slice(:by_columnist, :date, :period))
                            .offset(@page * ARTICLES_AMOUNT - ARTICLES_AMOUNT)
                            .limit(ARTICLES_AMOUNT)
                            .includes(:columnist)
                            .map { |x| OpinionPresenter.new(x) }
          raise ActiveRecord::RecordNotFound if opinions.empty?
          opinions
        end

        def partners
          @tenant.partners.map { |x| PartnerPresenter.new(x) }
        end

        def find_person
          Person.published
                .by_tenant(@tenant)
                .where.not(is_preview: true)
                .friendly.find(@params[:id])
        end

        def related_ratings(person)
          Rating.related_to(person)
                .select(:id, :rubric, :title, :slug, :published_at, :published_date, :status, :type)
                .limit(RELATED_RATING_AMOUNT)
        end

      end
    end
  end
end
{% endhighlight %}

Here we just extracted methods from the `call` method, but we also should extract querying logic from the `opinions` method:

{% highlight ruby %}
module ViewObjects
  module Desktop
    module Opinions
      class Index
        # ...

        def opinions
          opinions = Opinion.published
          opinions = filter(opinions)
          opinions = paginate(opinions)
          opinions = decorate(opinions)
          raise ActiveRecord::RecordNotFound if opinions.empty?
          opinions
        end

        def filter(scope)
          scope.by_tenant(@tenant)
               .filter(@params.slice(:by_columnist, :date, :period))
               .includes(:columnist)
        end

        def paginate(scope)
          scope.order(published_at: :desc)
               .offset(@page * ARTICLES_AMOUNT - ARTICLES_AMOUNT)
               .limit(ARTICLES_AMOUNT)
        end

        def decorate(scope)
          scope.map { |x| OpinionPresenter.new(x) }
        end

        # ...
      end
    end
  end
end
{% endhighlight %}

## Return object, responding to `status`, `locals`, `template`, `layout` methods, the last two are very optional

Our so-called view object is already responding to all of these messages, but we can make a better use of the `status`
message. The `raise` being hidden in the private section of the service class adds implicitness to overall control flow,
and thus we can bring it back to the controller with the use of the `status`. I tend to leave in controller actions
all the things, that directly affect render/redirect cycle, and in other controllers, we can have more complicated
logic, more options for conditional redirects.

{% highlight ruby %}
module ViewObjects
  module Desktop
    module Opinions
      class Index

        # ...

        def call
          self.locals = find_locals
          self.status = :not_found if locals[:opinions].empty?
          self.layout = 'desktop_special'
          self.template = 'desktop/articles'
          self
        end

        # ...

        def opinions
          opinions = Opinion.published
          opinions = filter(opinions)
          opinions = paginate(opinions)
          opinions = decorate(opinions)
          opinions
        end

        # ...

        end
      end
    end
  end
end

{% endhighlight %}

And finally, controller action turns into this:

{% highlight ruby %}
class Desktop::OpinionsController < DesktopController
  def index
    apply_filter_cookies
    view_object = ViewObjects::Desktop::Opinions::Index.new(@current_tenant, @current_page, params).call
    raise ActiveRecord::RecordNotFound if view_object.status == :not_found
    render layout: view_object.layout, template: view_object.template
  end
{% endhighlight %}

Also. instead of status codes, you may prefer to raise custom errors inside the interactor/service classes, and then
rescue them in controllers, but my personal preference is to use as fewer rescues as possible because `rescue` in ruby
is known for being slow.

