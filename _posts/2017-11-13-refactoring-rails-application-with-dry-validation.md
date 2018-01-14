---
layout: default
title:  "Refactoring Rails application with dry-validation"
date:   2017-11-13 09:35:44 +0300
comments: true
tags: rails dry-rb refactoring
---

# Refactoring Rails application with dry-validation

When developing web applications, we often face the problem of accepting and validating user input, or some data,
coming from external sources. The so-called "Rails way" proposes following way to deal with this problem:

* First, restrict form parameters at a controller level, using `strong_parameters` idiom

* Then, validate permitted parameters at a model level, using `ActiveModel::Validations`

This approach has later evolved into another one, which suggests extracting validations from models into separate classes,
where all validation or even persistence-related actions are performed. This pattern is known as Form Object and usually
involves such solutions as [Virtus](https://github.com/solnic/virtus), [Reform](https://github.com/trailblazer/reform),
[Dry-Types](https://github.com/dry-rb/dry-types), [makandra/active_type](https://github.com/makandra/active_type).

Now we will try to refactor both strong_parameters and model validations using
[Dry-Validation](https://github.com/dry-rb/dry-validation) gem.

Our example application allows some users to register with a mobile phone, and some another group of users can register with an email.
The registration logic also has huge differences.

{% highlight ruby %}
class User < ApplicationRecord
  attr_acessor :mobile_registration

  # validations
  validates :phone,
            presence: true,
            uniqueness: true,
            format: { with: /\A\+\d*/,
                      message: I18n.t('errors.messages.phone_format_is_invalid') }
  validates :email,
            uniqueness: true,
            case_sensitive: false,
            unless: proc { |u| u.mobile_registration || u.email.blank? }
  validates :password, length: { minimum: 8 }, on: :create,
            unless: proc { |u| u.mobile_registration || u.email.blank? }
  validates :password, confirmation: true, presence: true, if: :password_present?
  validates :password_confirmation, presence: true, if: :password_present?
  validates :full_name, presence: true
end
{% endhighlight %}

So, this model contains a bunch of validations:

* `phone` is being validated always
* `email` is validated only if it's present and `mobile_registration` attribute is unset
* `password` is validated only on `create` action, and only if it's present and `mobile_registration` attribute is unset
* `password_confirmation` must be equal to password and must be present if password is present
* `full_name` is always validated

For me, this part of model's code seems way too complicated. The first step in a way to breaking down this complexity is to move validations
away from the model and invoke them only in particular controller actions, where there is no need for condition checking. For example,
if password validation is performed only on `create`, why not to move it to respective controller's `create` action? Also,
obviously we have separate actions that register a mobile user and regular users, so we won't invoke `email` and `password` validations
for users registering with a mobile phone.

Let's take a look at a controller actions (they are oversimplified purposefully):

{% highlight ruby %}
# The first one used to register users via web interface:
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    if @user.save
      redirect_to profile_path(@user)
    else
      render :new
    end
  end

  private

  def user_params
    params.require(:user).permit(:phone, :email, :password,
                                 :password_confirmation, :full_name)
  end
end

# The second one is for mobile registrations
class Mobile::UsersController < MobilesController
  def create
    @user = User.new(user_params)
    # note this assignment. It is needed only to bypass conditional validation
    @user.mobile_registration = true
    if @user.save
      redirect_to profile_path(@user)
    else
      render :new
    end
  end

  private

  def user_params
    params.require(:user).permit(:phone, :full_name, :mobile_registration)
  end
end
{% endhighlight %}

Using dry-validation, we can extract all validation related logic into separate classes and also get rid of strong parameters.
What's the matter with strong parameters? Just remember, how strong_parameters-related methods are growing and become bloated over the time.

So, let's write a new class for User model validation, that would be used to validate users registering from the web interface:

{% highlight ruby %}
require 'dry-validation'

class UserValidator
  UserSchema = Dry::Validation.Schema do
    # regular expression for phone validations
    PHONE_REGEX = /\A\+\d*/
    configure do
      # we need this to perform database-related validation, i.e. uniqueness
      option :record
      # custom error messages
      config.messages_file = File.join(Rails.root, 'config',
                                       'locales', 'validation_errors.en.yml')
      # sanitize input hash permitting only whitelisted parameters.
      # All parameters in this file will be whitelisted,
      # others will be filtered out
      config.input_processor = :sanitizer

      # universal uniqueness predicate
      def unique?(attr_name, value)
        !record.class.where.not(id: record.id).where(attr_name => value).exists?
      end

      # checking if value matches PHONE_REGEX
      def phone?(value)
        !PHONE_REGEX.match(value).nil?
      end
    end

    # wrap schema in :user, mimicking strong_parameters require method
    required(:user).schema do
      required(:full_name).filled
      required(:phone).filled(:phone?, unique?: :phone)
      optional(:password).filled(min_size?: 8)
      optional(:password_confirmation).filled
      optional(:email).filled(unique?: :email)

      # custom rules for password confirmation
      rule(password_confirmed?: [:password, :password_confirmation]) do |password, password_confirmation|
        password.filled?.then(password_confirmation.eql?(password))
      end

      rule(password_confirmation_filled?: [:password, :password_confirmation]) do |password, password_confirmation|
        password.filled?.then(password_confirmation.filled?)
      end
    end
  end
end
{% endhighlight %}

File with custom errors will look like this:

```
en:
  errors:
    unique?: 'Is not unique'
    phone: 'Phone should start with plus sign and contain only digits'
```

This code serves as a replacement for both strong parameters and ActiveModel::Validations. Let's examine, what it exactly does.

{% highlight ruby %}
# controller
user = User.new
result =  UserValidator::UserSchema.with(record: user)
                                   .call(user: { full_name: '',
                                                 email: 'admin@admin.test',
                                                 phone: '89',
                                                 password: '12345678',
                                                 password_confirmation: '1234567' })

result.success?

# => false

result.errors
=> {:user=>
  {:full_name=>["must be filled"],
   :phone=>["Phone should start with plus sign and contain only digits"],
   :email=>["Is not unique"],
   :password_confirmation=>["must be equal to 12345678"]}}

# let's provide valid parameters:
result =  UserValidator::UserSchema.with(record: user)
                                   .call(user: { full_name: 'John',
                                                 email: 'john@admin.test',
                                                 phone: '+89',
                                                 password: '12345678',
                                                 password_confirmation: '12345678',
                                                 active: true })

result.success?

# => true

result.output

# Notice that excessive key :active is not present in the output hash,
# because it was filtered out with sanitizer.

# {:user=>{:full_name=>"John", :phone=>"+89", :password=>"12345678",
# :password_confirmation=>"12345678", :email=>"john@admin.test"}}

# And what if do not provide user: {} hash?

UserValidator::UserSchema.with(record: user).call(something: {}).errors
# => {:user=>["is missing"]}

{% endhighlight %}

And resulting controller code will be something like this:

{% highlight ruby %}
class UsersController < ApplicationController
  def create
    @user = User.new
    validation = UserValidator::UserSchema.with(record: @user).call(params)
    if validation.success?
      @user.attributes = validation.output[:user]
      @user.save
      redirect_to profile_path(@user)
    else
      @errors = validation.errors
      render :new
    end
  end
# no strong parameters neeeded
end

# and the second one:
class Mobile::UsersController < MobilesController
  def create
    @user = User.new
    validation = MobileUserValidator::UserSchema.with(record: @user).call(params)
    if validation.success?
      @user.attributes = validation.output[:user]
      @user.save
      redirect_to profile_path(@user)
    else
      @errors = validation.errors
      render :new
    end
  end
{% endhighlight %}

I omit the code for the imaginary MobileUserValidator, because it mostly repeats UserValidator, except email,
password and password_confirmation validations. Making the conclusion, dry-validation provides a very convenient way to
replace both ActiveRecord and ActiveModel validations, and is also a much nicer replacement for strong_parameters, if needed.

Suggested reading:

[Documentation](http://dry-rb.org/gems/dry-validation/)

[Introducing dry-validation](http://solnic.eu/2015/12/07/introducing-dry-validation.html)
