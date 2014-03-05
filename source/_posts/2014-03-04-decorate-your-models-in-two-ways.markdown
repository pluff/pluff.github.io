---
layout: post
title: "Decorate your models in two ways"
date: 2014-03-04 23:08
comments: true
categories:
description: "Common decorator implementations under the scope. Step by step implementation,  pros and cons, benchmarks, tips & tricks."
keywords: ruby, decorator pattern, decorator, presenters, benchmark, graper, active_decorator, ruby decorator gem, mvc
tags: [ruby, decorator, mvc]
---

<img src="/images/2014-03-04-decorate-your-models-in-two-ways.png" class="center" itemprop="image">

#### Intro

[Decorator](http://en.wikipedia.org/wiki/Decorator_pattern) is a common design pattern which allows us to add behavior to individual objects.
Ruby simplicity and flexibility provides several ways to implement it. Today we will write a couple of implementations and will do an overview of existing gems.

#### Keep our model as clean as possible

In MVC world decorators are widely used for adding presentation logic methods to a model without polluting a model class.
Let's say we have very simple `User` model and special logic for displaying user's full name.
```ruby
class User
  attr_accessor :first_name, :last_name, :title

  def full_name
    if title == "Doctor"
      "#{title} #{first_name} #{last_name}"
    elsif title
      "#{first_name} #{last_name} #{title}"
    else
      "#{first_name} #{last_name}"
    end
  end
end
```

Short and simple, however it provides one nasty side effect as your project grows.
<!-- more -->

Imagine there are 50 similar methods for various view-related stuff. Your model will become a mess in a blink of an eye!
If you came from Rails world you can use Rails helpers and move all representation methods there:
```ruby
module MyHelpers
  def user_full_name(user)
   #code
  end
  #...
end
#and then in views <%= user_full_name(user) %>
```
That will work, however it's hard to maintain and it goes against Ruby object oriented nature.
Another solution could be to move all extra methods to a separate module and include in our class
```ruby
class User
  attr_accessor :first_name, :last_name, :title

  include UserViewMethods
end
```
This solutions works fine, but it still adds plenty of methods to your User class even when you might not need it.
We can do better than that! We can keep our models as clean as we can and move our view helpers to another layer.

#### Wrap me a model, please

Lets create a new class `UserDecorator` that wraps our User object and adds additional method on top of it.
```ruby
class UserDecorator
  attr_accessor :user

  def initialize(user)
    @user = user
  end

  def full_name
    if @user.title == 'Doctor'
      "#{@user.title} #{@user.first_name} #{@user.last_name}"
    elsif title
      "#{@user.first_name} #{@user.last_name} #{@user.title}"
    else
      "#{@user.first_name} #{@user.last_name}"
    end
  end
end
#Usage example
decorated_user = UserDecorator.new(User.new)
decorated_user.user
decorated_user.full_name
```
Whenever we want to call a method on a model itself we have to write `decorated_user.user.my_method` which is so inconvinient!
We can get rid of it with mighty `method_missing`!
```ruby
class UserDecorator
  #...

  def method_missing(method, *args, &block)
    @user.public_send(method, *args, &block)
  end
end
#Usage example
decorated_user = UserDecorator.new(User.new)
decorated_user.first_name #delegates to User#first_name
decorated_user.full_name
```
Feels much better! Now our decorator pretends to be a User and delegates all undefined method calls to user object. Oh, wait a second...
`decorated_user.respond_to? :first_name #=> false` ouch!
`respond_to?` returns false unless there is a defined method on our class,
so we need UserDecorator to be a even more like User and treat all User.instance_methods as its own.
```ruby
class UserDecorator
  #...

  def method_missing(method, *args, &block)
    @user.public_send(method, *args, &block)
  end

  def respond_to?(method)
    super || @user.respond_to?(method)
  end
end
```
One step closer to success! Obviously we can delegate class methods and constants to User class as well.
This approach is implemented in the most popular presenters gem for Rails called [Draper](https://github.com/drapergem/draper).
You can check [this file](https://github.com/drapergem/draper/blob/594816661e0c6c48d6137f4472b8c731c3dd8df8/lib/draper/automatic_delegation.rb) for details.
Draper is a great and very rich featured gem but imvho it has a bit of "swiss army knife" syndrome.

There is a couple weak places in this approach:

1. Our decorator class pretends to be a User but it's not a User! `decorated_user.is_a? User #=> false`
2. `method_missing` is relatively slow. See below for benchmarks.

Let's think how we can decorate our model even better.

#### Extend me if you want!

As it was mentioned before we can create a module and move all presentation methods there. But should we include it in class definition?
Can we add these methods to our model only when we really need it? Yes we can!
```ruby
class User
  attr_accessor :first_name, :last_name, :title
end

module UserDecorator
  def full_name
    if title == 'Doctor'
      "#{title} #{first_name} #{last_name}"
    elsif title
      "#{first_name} #{last_name} #{title}"
    else
      "#{first_name} #{last_name}"
    end
  end
end
```
Now we can add presentation methods to our object right before we need them(e.g. right before rendering a view).
```
user = User.new
# do something with clean model object
user.extend UserDecorator #now we have additional methods in this particular instance
user.full_name #=> String object
another_user = User.new
another_user.full_name #=> exception. User class has no such method.
```
You can also overwrite base User methods if you want.
This approach allows us to add presentational methods per instance and keep our model class clean.
Our decorated object does not pretend to be a User anymore, it is a User!
Second most popular gem for Rails presenters called [active_decorator](https://github.com/amatsuda/active_decorator) has similar implementation with some additional features.
You can check [this file](https://github.com/amatsuda/active_decorator/blob/af3b6c60718106496b4acacd816325b00cfb873a/lib/active_decorator/decorator.rb#L31) for details.

Of course there is no free cheese in this world and we need to pay the price.

Weak places of this approach:

1. calling Object#extend is expensive.

#### Passing additional params to our decorators

In real world there are use cases when you need additional params in your representation methods.
With decoration "wrapper" class it's as easy as adding additional parameters to initialize method.
With "extending" it's slightly more complex since you can't pass additional params to extend method, so you have to add them after extending.
```
class User
  attr_accessor :first_name, :last_name, :title
end

module UserDecorator
  attr_accessor :my_param
  #....
end

u = User.new
u.extend UserDecorator
u.my_param = 1
```
Also we can encapsulate decoration logic in separate method (e.g. `decorate`)
```
class User
   #...
   def decorate
     self.extend UserDecorator
     yield self if block_given?
   end
end
user = User.new
user.decorate { |u| u.my_param = 1 }
```

#### Benchmarks. Can I take a nap while it works?

Are you curious what's the real performance numbers for both approaches? So do I! Here is the test I used for benchmarking:
```ruby
require 'benchmark'

class User
  def base_method
    1
  end
end

class UserDecoratorClass
  def initialize(user)
    @user = user
  end

  def method_missing(method, *args, &block)
    @user.send method, *args, &block
  end
end

module UserDecoratorModule
end

n = 1_000_000
Benchmark.bm do |x|
  x.report("User.new") { n.times { u = User.new; u } }
  x.report("Wrap    ") { n.times { u = User.new; UserDecoratorClass.new(u) } }
  x.report("Extend  ") { n.times { u = User.new; u.extend UserDecoratorModule } }
end

u_wrap = UserDecoratorClass.new User.new
u_ext = User.new.extend UserDecoratorModule

Benchmark.bm do |x|
  x.report("Wrap    ") { n.times { u_wrap.base_method } }
  x.report("Extend  ") { n.times { u_ext.base_method } }
end
```
First benchmark row shows us the time Ruby spends for creating simple object.
Second one represents time spent for decorating via "wrapping".
Third row represents time spent for decorating via extending.
Last 2 rows represent time spent for calling `User#base_method` for "wrapping" and extending approaches respectively.
Here is the results:
```
#Ruby 1.9.3
       user     system      total        real
User.new  0.230000   0.000000   0.230000 (  0.236019)
Wrap      0.530000   0.000000   0.530000 (  0.531092)
Extend    1.390000   0.010000   1.400000 (  1.391936)
       user     system      total        real
Wrap      0.330000   0.000000   0.330000 (  0.329640)
Extend    0.100000   0.000000   0.100000 (  0.102596)
#Ruby 2.1.0
       user     system      total        real
User.new  0.170000   0.000000   0.170000 (  0.170953)
Wrap      0.380000   0.000000   0.380000 (  0.383857)
Extend    2.140000   0.010000   2.150000 (  2.144567)
       user     system      total        real
Wrap      0.250000   0.000000   0.250000 (  0.241658)
Extend    0.080000   0.000000   0.080000 (  0.083517)
```
As we can see "wrapping" an object is almost 4 times faster (more than 9 times faster with Ruby 2.1.0).
On the other side method calls from undecorated base class is roughly 3 times faster with "extend" approach.
If your views use a lot of method calls per object then extending will work faster.
If you have plenty of small objects and low method calls frequency then "wrapping" will work faster for you.

#### Conclusion

There are several ways how to deal with presentation logic in Ruby. Using decoration classes is more classic well-known implementation, however it doesn't fits well with Ruby ideology with duck-typing and all that jazz.
Using modules and "Object#extend" provides us another way how to decorate our models with additional methods. It uses full power of Ruby object model and will come out with better performance in majority of use-cases.
Both methods have their weaknesses and you just need to choose what fits you better.

Thanks, and see you next time!








