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

Work in progress. What exists: the schema, queries and statements as values (`to_sql`), a repository that runs them on PostgreSQL, and associations with preloading.

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
  for `null: true`), a getter for each association (Associations, below), the constructors (below), `with` (below), and the class methods `table_name`, `columns` (`Array[Rotor::Column]`),
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
  type arguments are not kept. Their getters and preloads are below
  (Associations).
- Every model runs its own `table`. Its superclass is `Rotor::Model` or
  a class between them without a `table` (for methods models share):
  a model whose superclass has a table does not compile (at its
  `table`, when the superclass's ran first: in a cog it depends on, or
  earlier in its own cog's order), and a subclass that inherits a table without one of its own
  raises `Rotor::SchemaError` at its first use ("Guest has no table: it
  inherits User's; a model declares its own; single-table inheritance
  is not provided").
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
- Queries and statements are `Shareable` too (below).

## Queries and statements

A query is a value, built from blocks that are never run: each is
received as its tree (Quarz's `Quoted`, docs/14 §6) and translated to
SQL when the query is built, with the values it captured as parameters.

```ruby
min = 18
q = User.where { age >= min && !email.nil? }.order { name }.desc.limit(10)
q.to_sql
# => ["SELECT \"id\", ... FROM \"users\" WHERE \"age\" >= $1 AND NOT \"email\" IS NULL ORDER BY \"name\" DESC LIMIT 10", [18]]

User.where { |u| ids.include?(u.id) }        # "id" = ANY($1), the array one parameter
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
  captured array or a literal one of literals (`= ANY($1)`, the array
  one parameter whatever its length; an array literal with other
  expressions is an `IN` list); `String#start_with?`,
  `end_with?` and `include?` of a string value (`LIKE`, escaped);
  `Rotor::SQL.like`, `ilike`, `lower`, `upper` and `not_nil` (below); literals and
  captured values. Anything else raises `Rotor::UntranslatableError`
  when the query is built, naming the method and where it is.
- `==` and `!=` keep Quarz's meaning of nil. Against a value they are
  decided by the value: `IS NULL` for nil, `=` otherwise (which an
  index serves), so a query's SQL can differ by what it captured (pg's
  statement cache prepares each text once); `!=` of a nullable column
  and a value is `IS DISTINCT FROM`, which counts nil as different.
  Between two columns, one of which admits NULL, they are `IS [NOT]
  DISTINCT FROM`; `include?` finds nil with `IS NULL`.
- Every form rotor translates keeps Quarz's meaning; the shapes it
  cannot are refused when the query is built
  (`Rotor::UntranslatableError`, saying what to write instead). A
  comparison of something that can be NULL (`email == e`, `=` in SQL;
  `ids.include?(email)`, `IN`) is NULL where Quarz has false, which is
  the same as false in a condition: a `where`, `filter` or join
  condition, an operand of `&&` and `||`, an `if`'s condition or branches. Anywhere else it
  is refused: under `!` (`!(email == e)` would leave out the rows Quarz
  takes: write `email != e`), and as a value (an `order` key, an
  `update` value, an operand of `==` or another call). `select`'s block
  is not translated (below): `select { email == e }` is false on a NULL
  row.
- `col.nil?` and `!col.nil?` are `IS NULL` and `NOT ... IS NULL`, the
  explicit form, and what the left side of a `&&` (`!email.nil? &&
  ...`), of a `||` (`email.nil? || ...`) or an `if`'s condition says
  of a column holds on the other side: there it is not NULL, so its
  comparisons are two-valued. SQL's `<>` on a nullable column, which an
  index can serve, is written out: `!email.nil? && email != e` is
  `NOT "email" IS NULL AND "email" <> $1` (where `email != e` alone is
  `IS DISTINCT FROM`), and `email.nil? || !(email == e)` translates.
- `Int#/` floors and `Int#%` takes the divisor's sign, as in Quarz
  (`floor(a::numeric / b)::bigint`, `(a % b + b) % b`); `age / 2.0`
  is a `Float`'s division, and `%` with a `Float` is refused
  (PostgreSQL has none). An `Int`'s `/` or `%` on what may be an
  `Int` or a `Float` (`(if b then age else score end) / 2`) is
  refused: which one SQL does would depend on the row. Its `+`, `-`
  and `*` are computed in double precision for both (see
  [Parameter types](#parameter-types)).
- Parameters are typed by what they meet, and cast by their value's
  class where PostgreSQL would take another type than Quarz's (see
  [Parameter types](#parameter-types)). An `if` that is nil either way is
  `NULL`; one whose condition is a captured value other than a `Bool`
  is the branch Quarz's truthiness takes; any other condition that is
  not a `Bool` (a column, an expression; of an `if`, `&&`, `||` or
  `!`) is tested as Quarz does, true unless nil: `email ? 1 : 0` is
  `CASE WHEN "email" IS NOT NULL THEN 1 ELSE 0 END` (one that may be a
  `Bool` or another value is refused). An `update` checks against the
  column the class of its value, as Quarz gives it: a known value's, a
  column's, an expression's (an `if`'s branches, arithmetic by its
  operands: `{qty: -f}` with a `Float`, `{qty: price}` and `{name: qty}`
  are refused, where PostgreSQL would round or write the number as
  text; an `Int` into a `Float` column is accepted). `a && b` and
  `a || b` are a `Bool` in SQL and one of their operands in Quarz: as
  a value (an `update`'s, an order key, an operand) they are refused
  unless both operands are `Bool`s; as a condition they test their
  operands as above. Arithmetic of two `integer`s, as PostgreSQL types `Int`
  literals and CASEs of them, casts the left one to a bigint:
  `2000000000 * 2` is `2000000000::bigint * 2`, where an `integer`
  would overflow; `age + 1` stays as it is.
- `include?` is `col = ANY($1)` with the array as one parameter (pg's
  array parameters): one statement and one translation for every
  length. A nil item is left out of the array and `OR col IS NULL`
  added (`= ANY` finds no NULL); an array of nils only is `col IS
  NULL`; the empty array is `{}`, in which `= ANY` finds nothing. The
  items are of one class, or `Int`s and `Float`s (sent as `Float`s);
  pg types an array of `Int`s, `Bool`s or `Bytes`, the column decides
  the others' (`Float`s beside a column that is not a `Float`'s are
  cast, `$1::double precision[]`, as a `Float` is). An array literal
  with expressions (`[age, 3].include?(id)`) is an `IN (...)` list
  instead: its SQL is fixed by the literal's length in the source, so
  it makes no statement per length (the reason a captured array is one
  parameter, `= ANY($1)`), and `IN` is the form every SQL dialect has,
  where `= ANY(array)` is PostgreSQL's.
- Comparing a nullable column: `where { age > 18 }` is allowed in query
  blocks, with `age` an `Int?`. Every query block is checked with `using
  Rotor::Compare` (Quarz's docs/01 §7), whose extensions give `nil` and
  `Int`, `Float`, `String` and `Time` the comparisons `<`, `<=`, `>` and
  `>=` with a nil (`Int` and `Float` mixed either way): nil compares
  false (Quarz's meaning in these blocks), which is what SQL's NULL does
  in a condition. With a literal nil (`level > nil`, or an `if` whose
  branches are literal nils, a missing `else` counting as one) such a
  comparison is always false, and refused: compare with a value, or use
  `nil?` (`level == nil` and `level != nil` are `IS NULL` and `IS NOT
  NULL`). A captured nil is an unset value, and kept (`> $1`, or `>
  NULL` through an `if`). Where Quarz's false and SQL's NULL would
  differ (`!(age > 18)`, an `order` key, an `update` value), the query
  is refused, as above. Outside query blocks `Int? > Int` is still a type error, and
  nil has no arithmetic (`age + 1` is one in a block too). (As for any
  column, SQL compares strings by the database's collation, Quarz by
  bytes.) `Rotor::SQL.not_nil(age)` is the column as a `T` for the
  explicit cases, where the condition before it says it is not NULL:
  `where { !age.nil? && Rotor::SQL.not_nil(age) + 1 > 18 }`; anywhere
  else it is refused.
- Identifiers are always quoted. The SQL is PostgreSQL's
  (`Rotor::Dialect`), the one dialect for now.
- A block's translation is kept for the process, by its call site
  (Quarz's `Quoted#site`) and the models of the query's tables: building
  a query again where it was built before translates nothing, and fills
  in the parameters with the values captured this time. The SQL may
  depend on what kind of values the block captured (nil is `IS NULL`, an
  array's items' class and whether it holds a nil, not its length), so
  each kind seen at a site is an entry of its own. A key (a call site
  over its models) keeps at most 8 entries, never evicted; a kind
  beyond that is translated on each build, as without the cache: the
  kinds of several captures multiply (four nullable ones make 16). A `where` of three
  conditions with an order and a limit builds in about 10 µs compiled
  (22 µs translating each time; 19 µs interpreted, 52 µs before).
- Queries are `Shareable` (Quarz's docs/03): a query can be a constant
  (`ACTIVE = User.where { active }`), be captured by `Task.spawn` or sent
  on a `Channel`, and run in another task; so can a `Rotor::Statement`.
- `select`'s block is evaluated in Quarz, not in SQL: the query fetches
  the columns the block reads (every column when it calls another
  method of the row or passes it on), and each row, loaded with those
  only, gives the block's value. Reading a column it did not fetch
  raises `Rotor::NotLoadedError`. Counting is `count`, a query's shape.

### Parameter types

PostgreSQL types a parameter by what it meets; rotor casts one where
that type would not be Quarz's. What follows from it:

1. **How a parameter is typed.** Beside a column it is left untyped and
   takes the column's type (`"age" > $1`; a `Float` against a `Float`
   column too, which may be `real` or `numeric`: `r == 0.1` finds the
   `real` 0.1). An `Int` that meets anything but a column (a literal, an
   expression: `age < big + 1`, where `$1` would be an `integer`) is
   cast `::bigint`, and a `Float` that meets anything but a `Float`
   column `::double precision` (`age > 25.5`, which would be read as a
   bigint). A parameter of any class is cast by its value's class
   (`::bigint`, `::double precision`, `::text`, `::boolean`,
   `::timestamptz`, `::bytea`) in a `CASE`'s branch or bare condition,
   under a unary minus (`age > -x`), beside another parameter (`lo <
   hi`, compared as text otherwise) and under `IS NULL` (a nil there as
   `::text`). In arithmetic a `Float` parameter is double precision
   against any column, and an `Int` one a bigint against any column but
   an `Int`'s, so that `r + 0.1` on a `real` column is computed as Quarz
   does on the value read. An array is one parameter: pg sends an array
   of `Int`s as a `bigint[]` (of `Bool`s, `boolean[]`; of `Bytes`,
   `bytea[]`), the column decides the others' type (`Float`s beside a
   column that is not a `Float`'s are cast, `$1::double precision[]`).
2. **`integer` columns of tables rotor did not create.** rotor's
   schema makes an `Int` column a `bigint`. A table made otherwise may
   have `integer` ones, and rotor, which does not know a column's SQL
   type, does not cast by it (decided 2026-11-07, #28). A parameter
   beside such a column is an `integer`: `n == x` is exact for an `x`
   in its range, and one beyond it is an error from the server
   (`value "3000000000" is out of range for type integer`) where Quarz
   finds no row; the column's arithmetic with such a parameter or an
   `Int` literal is an `integer`'s, which overflows the same way
   (`n * 100000000`). A parameter cast `::bigint` compares with the
   column through PostgreSQL's cross-type operators (`integer =
   bigint` is in the `integer_ops` btree family), exactly and served
   by an index on the column: `n > (flag ? x : y)` (`"n" > CASE WHEN
   $1::boolean THEN $2::bigint ELSE $3::bigint END`) and
   `xs.include?(n)` (`"n" = ANY($1)` of a `bigint[]`) both use an
   index on `n`; arithmetic with one is a `bigint`'s.
   (test/repo_test.qz checks these on such a table.)
3. **A `Time` in a `CASE`'s branch** is cast `::timestamptz`, the type
   rotor's schema gives a `Time` column. Against a `timestamp` (without
   time zone) column of a table rotor did not create, PostgreSQL
   compares through its cross-type operators, reading the column's
   values in the session's time zone. Keep such columns `timestamptz`,
   or set the session's `TimeZone` to the one their values are in.
   (Decided 2026-11-06, #31: left untyped, the comparison fails, as it
   did before; cast `::timestamp`, `timestamptz` columns would have the
   mirror problem.)
4. **Arithmetic on what may be an `Int` or a `Float`** (`(if b then
   age else score end) + 1`): its `+`, `-` and `*` are computed in
   double precision for both, as SQL's `CASE` of a bigint and a double
   precision is one, so an `Int` row differs from Quarz beyond 2**53
   (#31); its `/` and `%` are refused (above).

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
  repo.delete(u)                           # the model; StaleError when its row is gone
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
  unchanged (so a statement that fails leaves nothing to put back).
  `update`, `touch` and `delete` of a model whose row no longer exists
  (the statement changed no row) raise `Rotor::StaleError`, naming the
  model and its key (Ecto's `StaleEntryError`). Transactions do not
  nest.
- `Rotor::Adapter` (`query`, `transaction`) is the boundary to the
  database; `Rotor::Adapters::PG` is pg's, over a pool or a connection.

## Associations

```ruby
class User < Rotor::Model
  table "users" do
    column :id, Int, primary: true
    belongs_to Account                  # account_id; user.account -> Account?
    has_many Post                       # user.posts -> Rotor::HasMany[Post]
  end
end

u = repo.get(User, 1).as(User)
u.posts.loaded?                         # => false
u.posts.size                            # NotLoadedError: User#posts is not loaded
repo.all(u.posts.query.where { published })   # the association as a query
u = repo.preload(u) { [posts, account] }      # one query per association
u.posts.map { |p| p.title }             # Post's, typed
u.account                               # Account?
users = repo.preload(repo.all(User.query)) { nest(posts) { comments } }
users = repo.preload(users) { nest(posts) { [nest(comments) { author }, tags] } }   # any depth
repo.all(User.where { age > 18 }.preload { posts })   # preloaded after loading
```

- No lazy loading (as in Ecto): a getter gives what was preloaded and
  raises `Rotor::NotLoadedError` otherwise. `has_many`'s getter is a
  `Rotor::HasMany[T]`, `Enumerable` over the preloaded models, whose
  `query` is the association as a query (refined and run as any) and
  `loaded?` says whether they are there; `belongs_to`'s gives the model
  or nil, nil for a NULL key even when not preloaded.
- `repo.preload(models) { ... }` and `repo.preload(model) { ... }` load
  what the block names, the getters `table` generated, so that a
  misspelling does not compile: `{ posts }`, `{ [posts, account] }`, and
  with `nest` the associations of their models (`{ nest(posts) {
  comments } }`, `nest(account) { users }`), and so on at any depth
  (`{ nest(posts) { nest(comments) { author } } }`); a name that is not
  an association of its level's model is refused naming that model. One
  query per association and level, `WHERE "user_id" = ANY($1)` (the
  keys one array parameter, however many), a model's children in the
  order of their primary key; an association already preloaded is not
  loaded again, and one named twice is one preload. The models given are left as they
  were: preloading gives new ones.
- `query.preload { ... }` (and `User.preload { ... }`) has
  `repo.all`/`one`/`first` preload after loading, as Ecto's `preload:`;
  not with `select` (a projection has no models).
- Preloaded associations are carried by `with`, `restore` and the
  repository's `update`, and are not in `to_h`; a `with` that changes a
  key forgets what was preloaded for it (reading it raises
  `Rotor::NotLoadedError`): a `belongs_to`'s foreign key its
  association, the primary key every `has_many` (loaded for the old
  key). Ecto keeps the stale structs. A target's key column
  is checked when a query or a preload uses it (`Rotor::SchemaError`
  when it has none), not at the statement: the target's `table` may run
  later.
- An association's name may not be a column's or one of the model's
  own methods'.

### Many to many: `has_many ..., through:`

```ruby
class Post < Rotor::Model
  table do
    column :id, Int, primary: true
    has_many Tag, through: PostTag      # post.tags -> Rotor::HasMany[Tag]
  end
end

class PostTag < Rotor::Model             # the join table is a model
  table do
    column :id, Int, primary: true
    belongs_to Post
    belongs_to Tag
    column :position, Int, default: 0   # columns of its own are ordinary
  end
end

repo.all(post.tags.query.where { name.start_with?("q") })
posts = repo.preload(posts) { tags }      # one query
repo.insert(PostTag.new(post_id: post.id, tag_id: tag.id))   # writing: the join model
```

- The join table is always a model (no model-less join table). Its
  `belongs_to` the owner and its `belongs_to` the target give the keys;
  `source:` names the latter when it has several (`has_many User, name:
  :followed, through: Follow, source: :followee`). They are found when
  the association is first used (a query, a preload), when every
  `table` block has run, whatever their order (the owner names the join
  model, which names the owner); a join model without them raises
  `Rotor::SchemaError` naming both classes.
- Read-only (Ecto's `has_many :through`): `post.tags.query` is the
  target's table alone, `WHERE "id" IN (SELECT "tag_id" FROM
  "post_tags" WHERE "post_id" = $1)`, refined and run as any query; a
  preload is one query joining the join table, `SELECT "t0".*,
  "p1"."post_id" FROM "tags" AS "t0" INNER JOIN "post_tags" AS "p1" ON
  "p1"."tag_id" = "t0"."id" WHERE "p1"."post_id" = ANY($1)`, grouped by
  owner, nested as any level. A target appears once per owner, however
  many join rows pair them (the join rows themselves are the join
  model's). Join rows are written as the join model's (`insert`,
  `delete_all(PostTag.where { ... })`); replacing the set at once may
  come with changesets. Not in `join { ... }` (join the join model).

### Joins

A query joins other tables in one SELECT, each table with its alias
(`"u0"`, `"p1"`):

```ruby
# A `for` comprehension over several queries: an INNER JOIN on its
# conditions (the inner query's and the `when`), a CROSS JOIN for none.
for u in User.query, p in Post.where { published } when p.user_id == u.id do [u.name, p.title] end
# Through an association, joined by its key, with more conditions.
for u in User.query, p in u.posts.query when p.views > 10 do p end
# Explicit forms, giving pairs: by an association, or by a condition.
User.query.join { posts }.where { |u, p| p.published }   # Query[[User, Post]]
User.query.join(Post) { |u, p| p.editor_id == u.id }
```

- In a `for`, the inner query is a model's (`Post.where { ... }`, its
  `query`) or an association's (`u.posts.query`), with `where`, `filter`,
  `order` and `desc`, then a last `map` or another iterator: anything else
  (a `limit`, a query in a variable) raises `Rotor::UntranslatableError`.
  Orders come after the outer query's.
- A `for`'s rows are made by its body as Quarz code, with the columns it
  reads: the body runs for each distinct outer row, building the inner
  query again (its translation kept, about 29 µs a row compiled, 35 µs
  translating each time). For large results, prefer
  `join`, whose pairs are built directly; `where`, `order`, `select`
  (`select { |u, p| ... }`), `count` and `limit` follow it, with a block
  parameter per table.
- A column's getter of a model a block captured is its value:
  `Post.where { user_id == u.id }`.
- Joins are INNER only; LEFT joins and `through` associations come later.

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
