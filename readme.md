# Lazy Record
## About
Active Record is the 'model' portion of the Ruby on Rails MVC framework. Specifically, it is an Object Relational Mapping (ORM) framework that allows programmers to write leaner code when working with the database. This is an implementation of that Active Record ORM using Ruby to implement an `SQLObject` class that interfaces with the database. This `SQLObject` class makes use of `::insert`, `::save`, and `::update` to perform their respective functions on records in the database while the methods themselves abstract away table-specific details. Lastly, this Active Record Lite implements associations using metaprogramming to build an Associatable module.

## Technologies
* Ruby
* SQL
* SQLite(Database Testing)

## Key Features
### Searchable Module
This simple module proves extremely powerful as it is able to implement a `::where` method allowing the programmer to pass multiple qualifiers to the database query, without writing out long-winded SQL. By passing in the conditions for the query as a hash, the module is able to sort through the keys and values to build up the proper SQL query using a heredoc.
```
module Searchable
  def where(params)
    where_line = params.keys.map { |key| "#{key} = ?"}.join(" AND ")
    args = params.values
    results = DBConnection.execute(<<-SQL, *args)
      SELECT
        #{table_name}.*
      FROM
        #{table_name}
      WHERE
        #{where_line}
    SQL

    parse_all(results)
  end
```

### Associatable Module
This module uses the belongs_to and has_many syntax (similar to Active Record) to build out the assocations between `SQLObject::` objects. The `BelongsToOptions::` and `HasManyOptions::` both build out the defaults for the foreign key, primary key, and class name needed for their respective assocations. Ruby's metaprogramming method `::define_method` enables us to use the association name as a method call to query the database using `::where` with the information provided from either `BelongsToOptions::` or `HasManyOptions::`.
```
module Associatable
  def belongs_to(name, options = {})
    self.assoc_options[name] = BelongsToOptions.new(name, options)

    define_method(name) do
      options = self.class.assoc_options[name]
      key_val = self.send(options.foreign_key)
      options.model_class.where(options.primary_key => key_val).first
    end
  end

  def has_many(name, options = {})
    self.assoc_options[name] = HasManyOptions.new(name, self.name, options)

    define_method(name) do
      options = self.class.assoc_options[name]
      key_val = self.send(options.primary_key)
      options.model_class.where(options.foreign_key => key_val)
    end
  end

  def assoc_options
    @assoc_options ||= {}
  end
end
```
