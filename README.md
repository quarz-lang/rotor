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

Work in progress. What exists: the schema, queries and statements as values (`to_sql`), and a repository that runs them on PostgreSQL.

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

u = User.new(name: "Ann")   # User.new(name: 1) does not compile
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
  for `null: true`), the constructors (below), `with` (below), and the class methods `table_name`, `columns` (`Array[Rotor::Column]`),
  `primary_key` (`Symbol?`) and `associations`
  (`Array[Rotor::Association]`).
- A model holds its values in one hash (`to_h`) and remembers the
  columns assigned since it was made or loaded (`changed`, `changed?`;
  `clear_changes` gives it without them). `new` takes each column as
  an optional keyword, as `with` does (`User.new(name: "Ann", age: 3)`,
  checked when the program is compiled), and starts from the columns'
  defaults for the others: the columns given are changed, in the
  columns' order; reading a column that has no default and was never set
  raises `Rotor::MissingAttributeError` (a `null: true` one reads nil).
  `load(values)` checks each value against its column's type.
- A model, and a class between it and `Rotor::Model`, does not write
  `initialize`: `table` generates the model's constructors
  (`def_initialize`, Quarz's docs/14 §5), the keyword one and the one
  `with` and the repository build through
  (`initialize(point: Rotor::Checkpoint)`). One of the same parameters
  does not compile ("already has an `initialize` with the same
  parameters"), a written `initialize()` makes `new` without arguments
  ambiguous, and one in a class between makes the model's `table` an
  error.
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
- Every model runs its own `table`. Its superclass is `Rotor::Model` or
  a class between them without a `table` (for methods models share):
  a model whose superclass has a table does not compile (at its
  `table`, when the superclass's ran first: in a cog it depends on, or
  earlier in its own cog's order), and a subclass that inherits a table without one of its own
  raises `Rotor::SchemaError` at its first use ("Guest has no table; a
  model declares its own; single-table inheritance is not provided").
  The members `table` generates belong to the class it ran on (its
  `with` is typed so), and single-table inheritance (one table for a
  hierarchy, with a type column) is not provided.
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

### Models are immutable and Shareable

A model never changes: `with` gives a new one, the one it was called on
left as it was, and the repository's `insert`, `update` and `touch`
give the model as saved.

```ruby
u = repo.get(User, 1).as(User)
v = u.with(name: "Bob", email: nil)   # email: NULL; age left as it was
v.changed                             # => [:name, :email]
u.name                                # => "Ann"
v = repo.update(v)                    # saved: v.changed => []
```

- `with` takes each column as an optional keyword of the column's type
  (`T?` for `null: true`, so `email: nil` sets NULL, while a column left
  out keeps its value): `with(age: "x")`, `with(name: nil)` for a column
  that is not `null: true`, and an unknown column do not compile. The
  primary key may be given (`User.new(id: 1)`), but a model whose
  key was assigned is not updated or deleted by it (the row it named is
  not known).
- So models are `Shareable` (Quarz's docs/03): one can be captured by
  `Task.spawn`, sent on a `Channel` or held by an `Atom`, and read in
  another task with no copy and no lock.
- As in Ecto (structs and changesets) and Slick (case classes and
  `copy`), and unlike Active Record, a model is a value: what to save is
  the model given, not a state the object accumulated.
- A model class is `Shareable`, so it cannot declare a `var` field (the
  checker refuses it); a value derived from the columns is a method.
- Queries are not `Shareable` yet (Quarz's open item #72): a model
  crosses tasks, a query does not.

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
User.select { Summary.new(id, name) }        # Query[Summary]: fetches "id", "name"
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
- `select`'s block is evaluated in Quarz, not in SQL: the query fetches
  the columns the block reads (every column when it calls another
  method of the row or passes it on), and each row, loaded with those
  only, gives the block's value. Reading a column it did not fetch
  raises `Rotor::NotLoadedError`. Counting is `count`, a query's shape.

## Running them: `Rotor::Repo`

```ruby
PG::Pool.open("postgres://app@localhost/shop") do |pool|
  repo = Rotor::Repo.new(pool: pool)       # or connection: a PG::Connection
  u = repo.insert(User.new(name: "Ann"))   # key, defaults and timestamps loaded back
  repo.all(User.where { age >= 18 })       # Array[User]
  repo.one(User.where { email == e })      # User?; MultipleResultsError for more
  repo.one!(q); repo.first(q); repo.get(User, 1)
  repo.count(q); repo.exists?(q)
  u = repo.update(u.with(age: 31))         # the changed columns, and updated_at
  u = repo.touch(u)                        # updated_at alone
  repo.delete(u)                           # Bool
  repo.delete_all(q); repo.update_all(q) { {age: age + 1} }   # Int
  repo.transaction { |tx| tx.insert(other) }                  # a Repo on the transaction
  repo.exec("VACUUM members")                                 # rows changed
  repo.query(User, "SELECT * FROM users WHERE age > $1", [18])  # Array[User]
  repo.query("SELECT id, name FROM users") { |r| Summary.new(r.int(0), r.string(1)) }
end
```

- `run(statement)` gives the rows changed and `run(query)` the rows;
  the others are conveniences over them.
- Values are decoded by the columns' types through pg's readers, times
  in UTC.
- Timestamps are set by the repository, never by SQL: `insert` sets
  `created_at` (unless the model has one) and `updated_at` to now in
  UTC, to the microsecond (what PostgreSQL keeps, so a value read back
  equals the one written); `update` sets `updated_at` only when a
  column changed, and runs no UPDATE when none did; `touch` writes it
  alone. `update_all` and raw SQL set none.
- `insert`, `update` and `touch` give the model as saved, the one given
  unchanged (so a statement that fails leaves nothing to put back). An `update` of a row that no longer exists changes
  nothing and is not reported, for now. Transactions do not nest.
- `Rotor::Adapter` (`query`, `transaction`) is the boundary to the
  database; `Rotor::Adapters::PG` is pg's, over a pool or a connection.

## Developing

To work on rotor against a local checkout of pg, write `Cog.local.toml`
beside `Cog.toml` (git ignores it):

```toml
[patch]
PG = { path = "../quarz-pg" }
```

The lock ignores it: switching the local patch on or off changes no
committed file, and CI and users get pg's git main.

## Tests

`quarz test` (and `--release`, `--doc`). The repository's tests run on
PostgreSQL through pg's `PG::Testing`: `PG_TEST_URL` when it is set,
else a throwaway cluster where PostgreSQL's server programs are
installed; with neither they are omitted.
