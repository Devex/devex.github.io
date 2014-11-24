---
layout: post
title: "One in a `has_many` - Access Special Association Objects in ActiveRecord"
date: 2014-11-24 15:12:48 +0100
author: dsager
comments: true
categories:
- Code

---

ActiveRecord models that define a `has_many` association often need access to a
specific entry of this list. Think of an user that has many email addresses but
only one that is his primary address. Or a Blog post with many comments of which
one is featured.

<!-- more -->

## How a lot of people do it

A pattern that seems to be quite common is to
[extend the association by implementing a method that gets you the specific record][ar-ass-extension]:

```ruby
class User < ActiveRecord::Base
  has_many :emails do
    def primary
      find(:first, conditions: 'is_primary')
    end
  end
end
```

This allows you to access the user's primary email address via `#emails.primary`.
So far so good, but what happens if we need to get a list of users with their
primary email address? Of course we do eager loading to reduce the amount of
database queries:

```ruby
User
  .includes(:emails)
  .find(:all)
  .each { |u| p "#{u.name}: #{u.emails.primary.email}" }
```

But when we look at the SQL queries that are actually executed we realize that
eager loading is happening but each primary email is queried separately
afterwards:

```
DEBUG: User Load (0.3ms) SELECT "users".* FROM "users"
DEBUG: Email Load (0.5ms) SELECT "emails".* FROM "emails"
                          WHERE "emails"."user_id" IN (1, 2, 3, 4, 5)
DEBUG: Email Load (0.6ms) SELECT "emails".* FROM "emails"
                          WHERE "emails"."user_id" = 1 AND (is_primary)
                          LIMIT 1
DEBUG: Email Load (0.3ms) SELECT "emails".* FROM "emails"
                          WHERE "emails"."user_id" = 2 AND (is_primary)
                          LIMIT 1
DEBUG: Email Load (0.3ms) SELECT "emails".* FROM "emails"
                          WHERE "emails"."user_id" = 3 AND (is_primary)
                          LIMIT 1
DEBUG: Email Load (0.3ms) SELECT "emails".* FROM "emails"
                          WHERE "emails"."user_id" = 4 AND (is_primary)
                          LIMIT 1
DEBUG: Email Load (0.3ms) SELECT "emails".* FROM "emails"
                          WHERE "emails"."user_id" = 5 AND (is_primary)
                          LIMIT 1
```

Ouch! This will screw up our app's performance as the user base grows!

## A better way

But there's another way of picking out one special instance of a `has_many`
association. A way that also allows eager loading. It's as simple as defining
just another association pointing to the same object.

```ruby
class User < ActiveRecord::Base
  has_many :emails
  has_one :primary_email, class_name: Email, conditions: 'is_primary'
end
```

Now you can access the user's primary email address by `#primary_email`. Let's
check the SQL log for a user list using eager loading:

```ruby
User
  .includes(:primary_email)
  .find(:all)
  .each { |u| p "#{u.name}: #{u.primary_email.email}" }
```

As we can see eager loading is now working properly for the primary email
addresses:

```
DEBUG: User Load (0.3ms) SELECT "users".* FROM "users"
DEBUG: Email Load (0.5ms) SELECT "emails".* FROM "emails"
                          WHERE "emails"."user_id" IN (1, 2, 3, 4, 5)
                          AND (is_primary)
```

Yay! Now all the millions of users out there can sign up on our page without
breaking the list of primary email addresses...

[ar-ass-extension]: http://api.rubyonrails.org/classes/ActiveRecord/Associations/ClassMethods.html#module-ActiveRecord::Associations::ClassMethods-label-Association+extensions
