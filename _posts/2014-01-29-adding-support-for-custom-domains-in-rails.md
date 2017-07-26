---
layout:     post
title:      Adding support for custom domains in Rails
date:       2014-01-29T15:41:00-05:00
summary:    How to use routes and constraints in your Rails app to support custom domains
categories: ruby rails software
permalink: /rails-custom-domain-support
---

A [fellow villager](http://atlantatechvillage.com) recently asked me how we had implemented custom domain support on [Rigor's public status pages](http://rigor.com/blog/2012/07/new-feature-public-status-page). I tried to find a decent resource that walked through the steps, but I noticed that it was hard to find relevant search results, so I figured I'd write about it.

To clarify, the question at hand is this: 

> As a Rails developer, how can I let my users point their custom domains to my app?

There are three main steps necessary for adding custom domain support to your Rails project:

1. Have your users __add a CNAME record__ pointing their domain to yours 
2. __Update your routes__ to handle any custom domains.
3. __Add the controller logic__ to find resources using the custom domain

For this example, let's assume your user wants to use their domain at __blog.company.com__ to point to their blog hosted on your Rails app at __myapp.com/blogs/:id__.

## Add a CNAME record

Although this step actually occurs last (once you've implemented the logic in your app), it serves as a more logical starting point for this walkthrough. Think about it: for a custom domain to go to your app, the first step is to connect the two domains. Adding a CNAME record does just that.

Have your user add a CNAME record for their domain pointing to your domain. For this example, we'll point __blog.company.com__ to your app domain __myapp.com__.

Here is what the CNAME record looks like on [DNSimple](https://dnsimple.com/):
![DNSimple CNAME setup](https://silvrback.s3.amazonaws.com/uploads/fad66045-c410-4408-87e9-45024670785b/dnsimple_cname_large.png)

## Update your routes

In order for your users to be directed to the right place when they visit their custom domain, you'll need to update the routes in your Rails app. In this case, we can add another __root__ route that sends requests to your `BlogsController`, but constrain it to the __blog__ subdomain.

```ruby
# config/routes.rb

# requests to custom.myapp.com should go to the blogs#show action
root to: 'blogs#show', constraints: { subdomain: 'blog' }

# keep your regular resource routing (you probably already have some version of this part in place)
resources :blogs
```

This would work fine for users using __blog__ as their subdomain, but what if we want to support _any_ custom subdomain? Enter advanced routing constraints.

#### Use advanced routing constraints
[Rails advanced constraints](http://guides.rubyonrails.org/routing.html#advanced-constraints) allow for more powerful routing logic. In our case, we can use an advanced constraint to add support for any custom domain. To use an advanced constraint:

1. Define an object that implements the `matches?` method:

```ruby
# lib/custom_domain_constraint.rb

class CustomDomainConstraint
  def self.matches? request
    request.subdomain.present? && matching_blog?(request)
  end

  def self.matching_blog? request
    Blog.where(:custom_domain =&gt; request.host).any?
  end
end
```

2. Pass the object to the constraint in your `routes.rb`:

```ruby
root to: 'blogs#show', constraints: CustomDomainConstraint

# or use the newer constraint syntax
constraints CustomDomainConstraint do
  root to: 'blogs#show'
end
```

## Add the controller logic

With the new `CustomDomainConstraint` in place, any request that has a subdomain and a matching `Blog` record will get routed to the `BlogsController#show` action. To finish our implementation, we need to add logic in `BlogsController` that finds the correct blog to render.

Assuming your `Blog` model already has a `custom_domain` field, adding the logic is easy:

```ruby
# app/controllers/blogs_controller.rb

def show
  @blog = Blog.find_by(custom_domain: request.host)
  # render stuff
end
```

For this to work properly, your user will need to set their blog's `custom_domain` to __blog.company.com__ in your app. With that in place, the request flow looks like this:

1. A user visits __blog.company.com__, which points to __myapp.com__
2. Your app handles the request from the `custom` subdomain, routing the request to the `#show` action in `BlogsController`
3. Your controller looks up the blog with __blog.company.com__ as the `custom_domain` and renders it

And just like that, your Rails app now supports custom domains!

*Note for Heroku users:*

If you're using [Heroku](https://www.heroku.com/) to host your Rails app, you'll need an additional bit of logic to make this work. Heroku's routing requires every domain to exist as a 'custom domain' in your Heroku app's settings. [This post](http://www.mccartie.com/2016/04/04/letting-users-add-custom-domains-on-heroku.html) outlines a way to automate this step via Heroku's API and Rails background workers.

## Going above and beyond

##### Keep standard route support
To make sure the standard blog routes still work (`/blogs/:id`), make sure your `BlogsController` still supports finding blogs by id:

 ```ruby
# app/controllers/blogs_controller.rb

def show
  @blog = Blog.find_by(custom_domain: request.host) || Blog.find(params[:id])
  # render stuff
end
```

To clean things up a bit, you might consider moving this into a `before_filter`:

 ```ruby
# app/controllers/blogs_controller.rb

before_filter :find_blog, only: :show

private

def find_blog
  # find the blog by domain or ID
end
```

While these tweaks aren't *required* for custom domains to work, they do improve the `BlogsController` logic to be cleaner and more intuitive.
