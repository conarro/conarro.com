---
layout:     post
title:      The Ruby "splat" operator, nil, and you
date:       2014-01-26T07:58:00-05:00
summary:    Handling differences across Ruby versions
categories: ruby
permalink: /ruby-splat-nil
---

I recently started using the handy [Saddle gem](https://github.com/mLewisLogic/saddle) by Airbnb's [Mike Lewis](https://twitter.com/mLewisLogic/) to speed up the development of Ruby API clients. Since some of our code runs on Ruby 1.8, I quickly got to work adding 1.8 support for Saddle.

While working out compatibility issues, I came across a strange error when running the test suite. After digging through Faraday (the underlying HTTP client used in Saddle), and `FaradayMiddleware`, I finally uncovered the issue:

>The splat operator handles `nil` differently in Ruby 1.8 and Ruby 1.9

### A quick test
To confirm my suspicion, I did a quick test in both Ruby versions:

```ruby
def test_splat *args
  args
end
```
###### Ruby 1.8.7
```
irb> test_splat(nil)
=> [nil]
```

###### Ruby 1.9.3
```
irb> test_splat(nil)
=> []
```

Saddle originally relied on Ruby 1.9's treatment of `*nil` to return an empty array when passing arguments to `FaradayMiddleware`. Adding 1.8 support revealed this dependency and forced me to handle `*nil` in the Saddle code instead.

### Conclusion
When using Ruby's handy splat operator, be sure to account for differences in Ruby versions. In my case, the fix was simple, but finding the culprit took some time. 
