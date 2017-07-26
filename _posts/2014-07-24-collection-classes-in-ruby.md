---
layout:     post
title:      Collection classes in Ruby
date:       2014-07-24T11:15:00-04:00
categories: ruby
summary:    Sick of iterating over arrays? Create a Collection class!
permalink: /collection-classes-in-ruby
---
Iterating over collections in Ruby is fun. Methods like [`each`](http://ruby-doc.org/core-2.0/Array.html#method-i-each), [`map`](http://ruby-doc.org/core-2.0/Array.html#method-i-map), and [`inject`](http://ruby-doc.org/core-2.0/Enumerable.html#method-i-inject) are intuitive and easy to use. When I find myself doing the same iteration logic over and over, though, I get frustrated. Duplicate iteration logic is a sign of poor design, and loop-happy methods can be harder to read and harder to test.

To avoid these issues, I've started creating `Collection` classes to wrap collections of objects. For example, say we have a `Document` class that looks like this:

```ruby
class Document
  attr_reader :filename

  def initialize filename
    @filename = filename
  end

  def read
    @content ||= File.open(filename).read
  end
end
```

Suppose a user can have many documents. Let's write a method to read all documents for a user:

```ruby
class User
  def documents
    # returns an array of all the user's Document objects
  end

  def read_documents 
    self.documents.map do |doc|
      doc.read
    end.join
  end
end
```
While this does the job, what if we have a `Team` class that can also have documents? If we want a `read_documents` method for `Team`, we'll be rewriting the exact same loop.

Let's see how a `DocumentCollection` class could help us avoid this duplication.

## Creating the Collection

First, we create a simple class to wrap a collection of documents:

```ruby
class DocumentCollection
  def initialize documents=[]
    @documents = documents
  end
end
```

With our base class at hand, let's add a `read_all` method to mimic our `User#read_documents` method.

```ruby
class DocumentCollection
  def read_all
    @documents.map do |doc|
      doc.read
    end.join
  end
end
```

Great! Now we have a `read_all` method defined in a single place. Now we can update our `User#read_documents` object to use the new collection, like so:

```ruby
class User
  def read_documents
    DocumentCollection.new(self.documents).read_all
  end
end
```

Adding this logic to a `Team` class is now easy, and we are only defining the `read_all` logic in one place. So we're done, right?

Not exactly. Although our `DocumentCollection` class meets our current needs, how easy will it be to extend? For example, finding documents of a certain type might mean defining a `matching_type` method:

```ruby
  def matching_type type
    @documents.select do |doc|
      doc.type == type
    end
  end
```

This works fine, but notice how all our methods are just loops on our `@documents` collection. How could we clean this up? By using Ruby's wonderful [`Enumerable`](http://ruby-doc.org/core-2.0/Enumerable.html) module, that's how!

## Improving with Enumerable

Including `Enumerable` in a class gives you all the iterative powers of classes like `Array` without any extra work. All we have to do is define an `each` method to tell `Enumerable` how we want to iterate over our class.

Here's how our `DocumentCollection` class might look after including `Enumerable`:

```ruby
class DocumentCollection
  include Enumerable

  def initialize documents=[]
    @documents = documents
  end

  def each &amp;block
    @documents.each(&amp;block)
  end

  def read_all
    map {|doc| doc.read}.join
  end

  def matching_type type
    select {|doc| doc.type == type}
  end
end
```

See how much cleaner that is than our initial version? Thanks to `Enumerable`, adding iteration logic is a snap. Now we can encapsulate any collection logic (and testing) in our collection class and share it across our codebase. Oh, the magic of `Enumerable`!
