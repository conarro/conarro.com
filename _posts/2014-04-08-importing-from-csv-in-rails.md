---
layout:     post
title:      Importing from CSV in Rails
date:       2014-04-08T05:58:00-04:00
summary:    Using ActiveModel::Model to keep things Railsy
categories: ruby rails
permalink: /importing-from-csv-in-rails
---

Building a CSV import option into your application can be very helpful for getting a lot of records into your database in one step. While building out some of [our recent reporting additions](http://wp.me/p45I00-18u) at [Rigor](http://rigor.com), I wanted to include the option to import CSV data to help our team migrate records from one service to another.

Using Ruby's standard CSV library makes reading CSV files a breeze. Implementing the import function in a Rails-like way, however, can be more difficult. In general, Rails controller actions look like:

````ruby
def index
  @my_model = MyModel.new(params[:my_model])
 
  if @my_model.save
    # yay, it worked! render some success page
  else
    # fooey, render some helpful error messages
  end
end
````

Massaging a CSV import into a controller action of this format isn't too terribly difficult, but it may not be completely obvious at first. For my import, I opted to lean on the `ActiveModel::Model` module (say that five times fast) in Rails 4 to create a model-like wrapper around the CSV import functionality.

I started by creating a class in my models directory and including the module:

````ruby
class MyAwesomeImporter
  include ActiveModel::Model  
end
````

To get the Rails model-like behavior, we have to define a few model methods:

````ruby
class MyAwesomeImporter
  include ActiveModel::Model
  
  def persisted?
    false    # since this model isn't ever persisted, just return false
  end

  def valid?
    # logic to determine if import is valid
  end 
end
````

For the `valid?` method, define what a valid import should look like and test it there, returning true or false. For example:

````ruby
def valid?
  record_attributes = read_stuff_from_csv
  import_records = record_attributes.map {|attrs| MyModel.new(attrs)}
  import_records.map(&amp;:valid?).all?
end
````

With the model-y parts out of the way, now we just have to set up our model with access to the CSV file and define `read_stuff_from_csv` and then we can use our new class in a controller action just like we always do:

````ruby
# my_awesome_importer.rb
def initialize(file)
  @file = file
end

def read_stuff_from_csv
  CSV.new(@file, headers: :first_row).each do |row|
    # do stuff with the row data
  end
  # return some useful data for making records
end
````

````ruby
# somewhere in my controller
def import
  @my_awesome_importer = MyAwesomeImporter.new(params[:csv_file])
  if @my_awesome_importer.save
    # let the user know the import worked
  else
    # boo, return some errors
  end
end
````

In addition to being easy to read and understand, using a symbolic model to wrap our import makes it easy to add helpful errors to users when an import fails. Since we included the `ActiveModel::Model`, adding validation errors is simple. For example, if we want to make sure the imported CSV isn't empty, we can just add the following:

````ruby
def csv_empty?
  if CSV.new(File.open(file2), headers: :first_row).to_a.empty?
    # add a helpful error message for the user
    errors.add :base, "CSV is empty"
    true
  end
end
````

Then we can update our `valid?` definition to check if the file is empty first:

````ruby
def valid?
  return false if csv_empty?
  # original validations
end
````

By leveraging Rails `ActiveModel::Model` module, we can incorporate the helpful features of a model into our simple Ruby class. When working with objects that span the MVC pattern, consider creating a model-like object to keep your controllers Railsy and keep form error-handling simple.
