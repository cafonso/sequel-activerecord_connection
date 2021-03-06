# sequel-activerecord_connection

This is an extension for [Sequel] that allows it to reuse an existing
ActiveRecord connection for database interaction.

This can be useful if you're using a library that uses Sequel for database
interaction (e.g. [Rodauth]), but you want to avoid creating a separate
database connection. Or if you're transitioning from ActiveRecord to Sequel,
and want the database connection to be shared.

It works on ActiveRecord 4.2+ and fully supports PostgresSQL, MySQL and SQLite
adapters, both the native ones and JDBC (JRuby). Other adapters might work too,
but their integration hasn't been tested.

## Installation

Add this line to your application's Gemfile:

```rb
gem "sequel-activerecord_connection", "~> 1.0"
```

And then execute:

```sh
$ bundle install
```

Or install it yourself as:

```sh
$ gem install sequel-activerecord_connection
```

## Usage

Assuming you've configured your ActiveRecord connection, you can initialize the
appropriate Sequel adapter and load the `activerecord_connection` extension:

```rb
require "sequel"

DB = Sequel.postgres(extensions: :activerecord_connection)
```

Now any Sequel operations that you make will internaly be done using the
ActiveRecord connection, so you should see the queries in your ActiveRecord
logs.

```rb
DB.create_table :posts do
  primary_key :id
  String :title, null: false
  Stirng :body, null: false
end

DB[:posts].insert(
  title: "Sequel::ActiveRecordConnection",
  body:  "Allows Sequel to reuse ActiveRecord's connection",
)
#=> 1

DB[:posts].all
#=> [{ title: "Sequel::ActiveRecordConnection", body: "Allows Sequel to reuse ActiveRecord's connection" }]

DB[:posts].update(title: "sequel-activerecord_connection")
#=> 1
```

The database extension supports `postgresql`, `mysql2` and `sqlite3`
ActiveRecord adapters, just make sure to initialize the corresponding Sequel
adapter before loading the extension.

```rb
Sequel.postgres(extensions: :activerecord_connection) # for "postgresql" adapter
Sequel.mysql2(extensions: :activerecord_connection)   # for "mysql2" adapter
Sequel.sqlite(extensions: :activerecord_connection)   # for "sqlite3" adapter
```

If you're on JRuby, you should be using the JDBC adapters:

```rb
Sequel.connect("jdbc:postgresql://", extensions: :activerecord_connection) # for "jdbcpostgresql" adapter
Sequel.connect("jdbc:mysql://", extensions: :activerecord_connection)      # for "jdbcmysql" adapter
Sequel.connect("jdbc:sqlite://", extensions: :activerecord_connection)     # for "jdbcsqlite3" adapter
```

### Transactions

This database extension keeps the transaction state of Sequel and ActiveRecord
in sync, allowing you to use Sequel and ActiveRecord transactions
interchangeably (including nesting them), and have things like ActiveRecord's
and Sequel's transactional callbacks still work correctly.

```rb
ActiveRecord::Base.transaction do
  DB.in_transaction? #=> true
end
```

Sequel's transaction API is fully supported:

```rb
DB.transaction(isolation: :serializable) do
  DB.after_commit { ... } # executed after transaction commits
  DB.transaction(savepoint: true) do # creates a savepoint
    DB.after_commit(savepoint: true) { ... } # executed if all enclosing savepoints have been released
  end
end
```

One caveat to keep in mind is that using Sequel's transaction/savepoint hooks
currently don't work if ActiveRecord holds the corresponding
transaction/savepoint. This is because it's difficult to be notified when
ActiveRecord commits or rolls back the transaction/savepoint.

```rb
DB.transaction do
  DB.after_commit { ... } # will get executed
end

DB.transaction do
  DB.transaction(savepoint: true) do
    DB.after_commit(savepoint: true) { ... } # will get executed
  end
end

ActiveRecord::Base.transaction do
  DB.after_commit { ... } # not allowed (will raise Sequel::ActiveRecordConnection::Error)
end

DB.transaction do
  ActiveRecord::Base.transaction(requires_new: true) do
    DB.after_commit(savepoint: true) { ... } # not allowed (will raise Sequel::ActiveRecordConnection::Error)
  end
end
```

### Model

By default, the connection configuration will be read from `ActiveRecord::Base`.
If you want to use connection configuration from a different model, you can
can assign it to the database object after loading the extension:

```rb
class MyModel < ActiveRecord::Base
  connects_to database: { writing: :animals, reading: :animals_replica }
end
```
```rb
DB.activerecord_model = MyModel
```

## Tests

You'll first want to run the rake tasks for setting up databases and users:

```sh
$ rake db_setup_postgres
$ rake db_setup_mysql
```

Then you can run the tests:

```sh
$ rake test
```

When you're done, you can delete the created databases and users:

```sh
$ rake db_teardown_postgres
$ rake db_teardown_mysql
```

## License

The gem is available as open source under the terms of the [MIT License](https://opensource.org/licenses/MIT).

## Code of Conduct

Everyone interacting in this project's codebases, issue trackers, chat rooms and mailing lists is expected to follow the [code of conduct](https://github.com/janko/sequel-activerecord-adapter/blob/master/CODE_OF_CONDUCT.md).

[Sequel]: https://github.com/jeremyevans/sequel
[Rodauth]: https://github.com/jeremyevans/rodauth
