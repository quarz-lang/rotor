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

Work in progress. What exists: the schema, and queries and statements as values (`to_sql`); running them comes next.

```ruby
class User < Rotor::Model
  table "users" do                      # `table do`: `users` from the class's name
    column :id, Int, primary: true      # Int, Float, String, Bool, Time, Bytes
    column :name, String
    column :email, String, null: true   # its getter returns String?
    column :age, Int, default: 0
    timestamps                          # created_at, updated_at: Time
    belongs_to Account                  # account_id: Int
    has_many Post                       # posts, through Post's user_id
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
- `belongs_to Account` and `has_many Post` take the associated model's
  class (compile-time code cannot turn a name into one), and declare
  the association (`account`, `posts`; or `name:`) and, for
  `belongs_to`, its key column (`account_id`; or `foreign_key:`, as for
  `has_many`'s key in the other table, by default `user_id` from this
  model's name). Two models may name each other, in one file or two.
  A target without a `table` block compiles, and raises
  `Rotor::SchemaError` when its metadata is read; a generic target's
  type arguments are not kept.
  The queries that follow them come with the queries.
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

## Queries and statements

A query is a value, built from blocks that are never run: each is
received as its tree (Quarz's `Quoted`, docs/14 §6) and translated to
SQL when the query is built, with the values it captured as parameters.

```ruby
min = 18
q = User.where { age >= min && !email.nil? }.order { name }.desc.limit(10)
q.to_sql
# => ["SELECT \"id\", ... FROM \"users\" WHERE \"age\" >= $1 AND NOT \"email\" IS NULL ORDER BY \"name\" DESC LIMIT 10", [18]]

User.where { |u| ids.include?(u.id) }        # "id" IN ($1, $2, ...)
User.where { name.start_with?(prefix) }      # "name" LIKE $1, the prefix escaped
User.select { Summary.new(id, name) }        # Query[Summary]: SELECT "id", "name"
User.where { admin }.count                   # Query[Int]: SELECT count(*) ...
User.where { admin }.exists                  # Query[Bool]: SELECT EXISTS (...)
User.where { age < 13 }.delete               # Statement: DELETE FROM ...
User.where { admin }.update { {age: age + 1, name: "x"} }   # UPDATE ... SET
Rotor::Statement.insert(u).returning(:id)
Rotor::Statement.update(u)                    # the changed columns, nil for none
Rotor::Statement.delete(u)                    # by the primary key (refused once it was assigned)
```

- The row is the block's receiver (`where { age > 18 }`) or its
  parameter (`where { |u| u.age > 18 }`, the form `for` comprehensions
  use). A block that names nothing of the row (`where { true }`) fits
  both and does not compile: write `where { |u| true }`.
- What translates: the columns' getters; `== != < <= > >=`, `+ - * / %`
  (`+` of strings is `||`) on numbers, strings, booleans and times;
  `&&`, `||`, `!`; `nil?`; `if`/`?:` (`CASE WHEN`); `include?` on a
  captured or literal array (an `IN` list, `FALSE` when empty; one
  parameter per item, within PostgreSQL's 65535); `String#start_with?`,
  `end_with?` and `include?` of a string value (`LIKE`, escaped);
  `Rotor::SQL.like`, `ilike`, `lower` and `upper`; literals and
  captured values. Anything else raises `Rotor::UntranslatableError`
  when the query is built, naming the method and where it is.
- `==` and `!=` keep Quarz's meaning of nil. Against a value they are
  decided by the value: `IS NULL` for nil, `=` otherwise (which an
  index serves), so a query's SQL can differ by what it captured (pg's
  statement cache prepares each text once); `!=` of a nullable column
  and a value is `IS DISTINCT FROM`, which counts nil as different.
  Between two columns, one of which admits NULL, they are `IS [NOT]
  DISTINCT FROM`; `include?` finds nil with `IS NULL`. What SQL does
  not keep: `!` over a comparison of a nullable column (`!(email ==
  "x")` leaves the nil rows out, where Quarz takes them), and a
  comparison's value in `select`, NULL where Quarz has false.
- `Int#/` floors and `Int#%` takes the divisor's sign, as in Quarz
  (`floor(a::numeric / b)::bigint`, `(a % b + b) % b`).
- An `include?` list is one parameter per item, and PostgreSQL takes at
  most 65535 in a statement (not checked when the query is built). When
  pg encodes arrays, it becomes `= ANY($1)`.
- A nullable column's getter returns `T?`, which `<` does not take (a
  block cannot narrow a getter's result): such a column is compared
  with `==`, `!=` and `nil?` only, for now.
- Identifiers are always quoted. The SQL is PostgreSQL's
  (`Rotor::Dialect`), the one dialect for now.
- A query is translated each time it is built: about 26 µs for a
  `where` of three conditions with an order and a limit, compiled
  (59 µs interpreted). Queries are not `Shareable` yet (their captured
  values are `Object`s), so one cannot be a constant.
