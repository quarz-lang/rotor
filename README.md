# rotor

A functional-relational mapper for [Quarz](https://quarz.dev): models
whose columns are declared in their class body and turned into typed
accessors when the program is compiled, and (to come) queries written as
Quarz blocks that are translated to SQL. It runs on PostgreSQL through
[pg](https://github.com/quarz-lang/pg).

```toml
# Cog.toml of an application
[dependencies]
Rotor = { git = "https://github.com/quarz-lang/rotor", branch = "main" }
```

Work in progress. What exists: the schema.

```ruby
class User < Rotor::Model
  table "users" do                      # `table do`: `users` from the class's name
    column :id, Int, primary: true      # Int, Float, String, Bool, Time, Bytes
    column :name, String
    column :email, String, null: true   # its getter returns String?
    column :age, Int, default: 0
    timestamps                          # created_at, updated_at: Time
    belongs_to :account                 # account_id: Int
    has_many :posts                     # Post's user_id
  end
end

u = User.new
u.name = "Ann"         # u.name = 1 does not compile
u.email                # => nil
u.age                  # => 0
u.changed              # => [:name]
User.table_name        # => "users"
User.primary_key       # => :id
User.columns           # => [Rotor::Column(name: :id, sql: "bigint", ...), ...]
User.load({id: 1, name: "Bob"}).name   # => "Bob", unchanged
```

- `table` runs its block at compile time on a `Rotor::Schema`, then
  generates, for each column, a getter typed by the column (`T`, or `T?`
  for `null: true`) and a setter that records the change, and the
  class methods `table_name`, `columns` (`Array[Rotor::Column]`),
  `primary_key` (`Symbol?`) and `associations`
  (`Array[Rotor::Association]`).
- A model holds its values in one hash (`to_h`) and remembers the
  columns assigned since it was made, loaded or `clear_changes`
  (`changed`, `changed?`). `new` takes no arguments and starts from the
  columns' defaults; reading a column that has none and was never set
  raises `Rotor::MissingAttributeError` (a `null: true` one reads nil).
  `load(values)` checks each value against its column's type.
- `belongs_to` and `has_many` declare the association and, for
  `belongs_to`, its key column; the queries that follow them come with
  the queries.
- A subclass of a model shares its table, unless it has a `table`
  block of its own: then its columns are its own, and a setter it
  inherits for a column it does not have, or with another type, raises
  `ArgumentError`. Single-table inheritance (one table for a hierarchy,
  with a type column) is not provided.
- A `Time` column takes no `default:` (the compiler cannot bake a time
  into the program).
- A model without a `table` block raises `Rotor::SchemaError` at its
  first use; a second `table` block in one class does not compile
  (`already has a class method table_name`).

**Why one block.** Quarz runs the statements of a class body
independently of each other (its docs/14 §2): a statement does not see
what another generated, nor can it generate a member another already
did. A `column` statement of its own could make its accessors, but none
could know the other columns to make `columns` and `primary_key`; the
one `table` statement sees them all.

Models are mutable, so they are not `Shareable`: a task that needs one
loads its own.
