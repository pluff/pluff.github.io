---
layout: post
title: "Rails and background jobs"
description: "Some details on background jobs integration with Rails including after_save and after_commit callbacks, dirty attributes, promises concept and observers"
date: 2013-10-18 16:44
comments: true
categories:
keywords: ruby, rails, ruby on rails, ror, background jobs, delayed jobs, resque, background workers, callbacks, after_save, after_commit, after_update
tags: [ruby, rails ,background jobs, Resque]
---
{% img center /images/2013-10-18-rails-and-background-jobs.png %}

Background jobs concept is a great way to postpone some tasks to another thread.
Today I'll show you how we can use it with Rails and make our application a bit (or a lot) faster.

#### Initial state

Lets say we have user model with `email` attribute and we want to send email to
administrator each time user email changes.
```ruby
class User < ActiveRecord::Base
  after_save do |user|
    send_email if user.email_changed?
  end
end
```
<!-- more -->
Let's assume that `send_email` method represents "all things that should be done".
We don't care much about its implementation.
Example above works perfectly but has one weakness. What will happen if `send_email` method takes 10 seconds to run?
User will have to wait for ten seconds.
What can we do about that?

#### Resque to the resque!

We can move execution of our `send_email` method to separate process.
[Resque](https://github.com/resque/resque) is the most popular among the [plenty of gems](https://www.ruby-toolbox.com/categories/Background_Jobs) for background jobs.
Resque stores all background jobs in a queue (or multiple queues) in Redis and processes it later within another processes (workers).
Let's rewrite example code with Resque.

```ruby
class User < ActiveRecord::Base
  after_save do |user|
    Resque.enqueue(Mailer, user.id) if user.email_changed?
  end
end

module Mailer
  @queue = "mailer_queue"
  def self.perform(user_id)
    user = User.find(user_id)
    user.send_email
  end
end
```

When user email is changed Resque will enqueue `Mailer.perform` method call with model id as param.
`Mailer.perform` method will actually be called in another process.
Resque implementation is not the main topic of this post, so, if you want to learn more about Resque check this great [RailsCast](http://railscasts.com/episodes/271-resque).
From this point I'll assume that you understand the concept of background jobs.

#### New problems

Moving `send_email` method execution to separate process was a great idea, but it brought a small problem.
Sometimes when Resque tries to call `Mailer.perform` it throws an exception "User with id XXX was not found".
According to [ActiveRecord::Callbacks](http://api.rubyonrails.org/classes/ActiveRecord/Callbacks.html) `after_save`
callback is performed when DB transaction is not committed yet. That's why `User.find` can't find User.
As you might already understand I was too eager with enqueue.

__Always use after_commit instead of after_create\after_update\after_save for background jobs__.

That's the golden rule. `after_commit` ensures that transaction was successfully committed to DB and all records were stored.
Lets update the code!

```ruby
class User < ActiveRecord::Base
  after_commit do |user|
    Resque.enqueue(Mailer, user.id) if user.email_changed?
  end
end
#...
```

That should work. Oh wait... it doesn't!

#### after_commit limitations and the ways to deal with them

The code above never enqueues mail sending because `user.email_changed?` is always false.
For some reason [dirty](http://api.rubyonrails.org/classes/ActiveModel/Dirty.html) state of
our model resets before `after_commit` callback is executed, so every `xxx_changed?` returns false and every `xxx_was`
does NOT return old values.

In `after_save` we CAN'T enqueue because we can't be sure record is already saved to DB properly.
In `after_commit` we DON'T have access to previous state of our model.
Here is where the concept of promises comes to play.

Let's think about what we CAN do at each of these callbacks.
In `after_save` we can say if we should enqueue or not as we can check if email is changed or not.
In `after_commit` we can just enqueue a background job.
So, our solution is to promise to enqueue the job as soon as we can enqueue. Simple implementation looks like that:

```ruby
class User < ActiveRecord::Base
  attr_accessor :need_enqueue

  after_save do |user|
    need_enqueue = true if user.email_changed?
  end

  after_commit do |user|
    Resque.enqueue(Mailer, user.id) if user.need_enqueue
    user.need_enqueue = nil
  end
end
#...
```
Finally it works! I want to point out that the concept of promises can be skipped if we don't need any dirty attribute in our `after_commit` callback.
E.g. Lets assume than our business requirement was the following: "Send a email to admin each time user is created."
In this case implementation is as simple as
```ruby
class User < ActiveRecord::Base
  after_commit, on: :create do |user|
    Resque.enqueue(Mailer, user.id)
  end
end
#...
```

As my last point I want to mention that callback above can be moved to observer.
As for me I like observers a lot so I usually move all accompanying callbacks to observers (e.g. mail sending, logging, cache invalidation etc)

#### Summary
Sometimes background jobs make our response from servers considerably faster by cost of additional code complexity.
Next few rules can help you to avoid common mistakes with integration of background jobs.

1. Always use `after_commit` for background jobs callbacks to avoid race condition.
2. Use concept of promises in case you need dirty attributes for callback logic.
3. Use observers whenever you see that callback logic does not belong to observed model.




