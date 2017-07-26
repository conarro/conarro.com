---
layout:     post
title:      Adding support for custom domains in Rails
date:       2014-01-29T15:41:00-05:00
summary:    How to use routes and constraints in your Rails app to support custom domains
categories: ruby rails software
permalink: /rails-custom-domain-support
---
At [Rigor](http://rigor.com), we use [Intercom](https://www.intercom.com/) to manage support requests. I really like their chat-oriented approach, and the in-app help makes it very easy for users to talk to us. But do we really want to wait for users to encounter bugs to find out about them?

# Error reporting with Rollbar
Enter Rollbar. We use Rollbar across our stack, from our Ruby/Rails back-end applications to our client-side JavaScript. Their error aggregation and reporting and integrations with notification and ticketing services make it easy to incorporate in your development process.

With Rollbar's client-side library, we get notified of JavaScript errors _as they happen_. We've configured the library to report information about the user who encountered the error, giving us the ability to proactively reach out _before_ the user writes in.

When we see errors, we can quickly create a ticket in JIRA to investigate and fix the underlying bug. But oftentimes there's work involved in reproducing the user's actions. What did they click? What were they trying to do?

# Session recordings with Fullstory

We already use [Fullstory]() to explore user activity and research UX improvements. We can search for the user's email and try to find the matching session recording, but that can be challenging for more active users.

Luckily, Fullstory provides access to the session URL directly from their client-side library(link). Using custom payload information in [our Rollbar configuration](), we were able to pull in the link to the exact session when reporting errors to Rollbar.

[screenshot]

# The workflow
Now, when a user runs into a client-side error, this is what happens:

`User triggers error > Rollbar notifies Slack > Developer views error + Fullstory session`

Minimizing the steps to resolution creates the appearance of super powers. Ideally, a user never runs into bugs, but

> I don't usually 
