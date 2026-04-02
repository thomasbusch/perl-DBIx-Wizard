# DBIx::Wizard — Design Specification

## Project Goal

Build a clean, composable, and ergonomic database query executor that takes full advantage of SQL::Wizard's expression tree capabilities.

---

## Why This Module Is Needed

Perl has two extremes for database access and nothing good in between:

**Raw DBI** — maximum control, but verbose and error-prone. Building SQL by hand means string concatenation, manual bind parameter management, and no protection against injection mistakes. Even a simple filtered query with a join takes 10+ lines of boilerplate.

**DBIx::Class** — powerful ORM, but heavyweight. Requires defining a schema with result classes, relationships, and column metadata before you can run a single query. Excellent for large applications with well-defined models, but overkill when you just want to grab some rows from a table. Startup cost is high, the learning curve is steep, and the abstraction can fight you when you need non-trivial SQL.

There's nothing in between that lets you write:

```perl
my @users = dbiw('mydb:users')->find({ status => 'active' })->all;
```

without defining a schema, without 50 lines of setup, and without concatenating SQL strings. Ruby has ActiveRecord and Sequel. Python has SQLAlchemy. JavaScript has Knex. Perl has a gap.

**DBIx::Wizard fills that gap.** It's a thin, fluent query executor that:

- Requires zero schema definition — point it at a database and table, start querying
- Uses SQL::Wizard for safe, composable SQL generation — not string concatenation
- Returns plain hashrefs — no result objects, no inflation overhead unless you want it
- Supports the full SQL::Wizard feature set — joins, subqueries, expressions, CTEs, window functions — when you need them
- Stays out of the way — if you need raw SQL, drop down to DBI. If you need composable queries, use SQL::Wizard directly. DBIx::Wizard is the convenient layer on top, not a cage.

---

## Design Philosophy

### 1. One-liner friendly

The primary design goal is developer speed. Getting data from a database should be as concise as possible. The `dbiw('db:table')` factory and method chaining should make simple queries trivially short:

```perl
my @names = dbiw('mydb:users')->find({ status => 'active' })->all('name');
```

### 2. SQL::Wizard expressions everywhere

DBIx::Wizard accepts SQL::Wizard expression objects anywhere — in columns, in find() conditions, in sort(), in having(). The `dbiw` function doubles as a SQL::Wizard instance:

```perl
# dbiw without args returns a SQL::Wizard instance for expressions
dbiw->func('COUNT', '*')
dbiw->col('views') + 1
dbiw->now
dbiw->between('age', 18, 65)
```

### 3. Mutable chaining (not immutable)

Unlike SQL::Wizard's expression tree (which is immutable), DBIx::Wizard ResultSets are mutable. Each chained method modifies and returns `$self`. This is intentional — ResultSets are throwaway query builders, not reusable templates:

```perl
# This is one query being built, not a reusable base
dbiw('mydb:users')->find({ active => 1 })->sort('-created_at')->limit(10)->all;
```

### 4. Convention over configuration

- `dbiw('db:table')` — colon-separated database and table
- `table|alias` — pipe-separated alias (passed through to SQL::Wizard)
- DateTime inflation on by default
- Decimal fixing on by default
- Auto-detect primary key from table metadata

### 5. Thin layer over SQL::Wizard

DBIx::Wizard should be a thin execution layer. It should NOT reimplement SQL generation. The flow is:

```
dbiw('db:table')->find(...)->sort(...)->all(...)
      ↓
  Build SQL::Wizard query from accumulated state
      ↓
  $query->to_sql  →  ($sql, @bind)
      ↓
  DBI prepare + execute + fetch
      ↓
  Return results
```

---

## API Reference

### Entry Point

```perl
use DBIx::Wizard;

# Factory: returns a ResultSet tied to a database + table
my $rs = dbiw('mydb:users');

# Expression builder: returns a SQL::Wizard instance
my $expr = dbiw->col('views');
```

`dbiw` with a string argument creates a ResultSet. Without arguments (or as a class method), it returns a shared SQL::Wizard instance for building expressions.

---

### Database Registration

```perl
use DBIx::Wizard::DB;

DBIx::Wizard::DB->declare('mydb',
    'dbi:mysql:dbname=myapp', 'user', 'password',
    { mysql_enable_utf8 => 1 },
);
```

Register databases by name before using them. Connection handles are cached and reused.

#### Environment-based declaration

Databases can also be declared via environment variables, requiring zero code setup. The format is:

```
DBIW_DECLARE_<NAME>=<dsn>|<user>|<password>
```

The `<NAME>` is uppercased in the environment but maps to the lowercase database key in `dbiw()`. Examples:

```bash
# Declare a database called "mydb"
export DBIW_DECLARE_MYDB="dbi:mysql:dbname=myapp|root|secret"

# Declare a database called "analytics"
export DBIW_DECLARE_ANALYTICS="dbi:Pg:dbname=analytics|readonly|pass123"

# No password
export DBIW_DECLARE_LOGS="dbi:SQLite:dbname=/tmp/logs.db||"
```

Then use directly without any `declare()` call:

```perl
use DBIx::Wizard;

my @users = dbiw('mydb:users')->all;          # connects via DBIW_DECLARE_MYDB
my @events = dbiw('analytics:events')->all;   # connects via DBIW_DECLARE_ANALYTICS
```

Environment declarations are checked lazily on first connection attempt. If both a `declare()` call and an environment variable exist for the same name, the `declare()` call takes precedence.

This makes it possible to write scripts and tests that need no database configuration code at all — everything comes from the environment, following the twelve-factor app principle.

---

### ResultSet — Query Building

All builder methods return `$self` for chaining.

```perl
dbiw('mydb:users')
    ->as('u')                                    # table alias
    ->find({ status => 'active' })               # WHERE (hash merges, array replaces)
    ->join('orders|o' => 'o.user_id = u.id')     # JOIN
    ->sort('-created_at', 'name')                 # ORDER BY (- prefix = DESC)
    ->group_by('department')                      # GROUP BY
    ->having({ dbiw->func('COUNT', '*') => { '>' => 5 } })  # HAVING
    ->limit(20)                                  # LIMIT
    ->offset(40)                                 # OFFSET
    ->inflate(1)                                 # DateTime inflation (default: on)
```

#### as(alias)

Sets the table alias. Translates to SQL::Wizard's `table|alias` syntax.

```perl
dbiw('mydb:users')->as('u')
# internally: -from => 'users|u'
```

#### find(condition)

Sets or extends the WHERE clause. Hashref merges with existing conditions. Arrayref replaces. Accepts SQL::Wizard expression objects:

```perl
->find({ status => 'active', age => { '>' => 18 } })
->find({ dbiw->func('LENGTH', 'name') => { '>' => 3 } })
->find([dbiw->between('age', 18, 65)])
```

#### join(table, on)

Adds a JOIN. Accepts same syntax as SQL::Wizard's `$q->join()`:

```perl
->join('orders|o' => 'o.user_id = u.id')             # INNER JOIN
->left_join('payments|p' => 'p.order_id = o.id')      # LEFT JOIN
```

#### sort(columns)

Sets ORDER BY. Accepts SQL::Wizard's `-column` shorthand for DESC:

```perl
->sort('-created_at', 'name')     # ORDER BY created_at DESC, name
->sort(dbiw->col('score')->desc)  # ORDER BY score DESC
```

#### group_by(columns)

```perl
->group_by('department', 'year')
```

#### having(condition)

```perl
->having({ dbiw->func('COUNT', '*') => { '>' => 5 } })
```

#### limit(n) / offset(n)

```perl
->limit(20)->offset(40)
```

---

### ResultSet — Data Retrieval

#### all(columns?)

Executes SELECT and returns all rows as hashrefs. Optional column spec:

```perl
my @rows  = dbiw('mydb:users')->all;                    # all columns, all rows
my @rows  = dbiw('mydb:users')->all(['id', 'name']);     # specific columns
my @names = dbiw('mydb:users')->all('name');             # single column → flat list
```

#### one(columns?)

Like `all()` but LIMIT 1, returns single hashref or scalar:

```perl
my $user = dbiw('mydb:users')->find({ id => 42 })->one;
my $name = dbiw('mydb:users')->find({ id => 42 })->one('name');
```

#### cursor(columns?)

Returns a Cursor object for lazy iteration:

```perl
my $cursor = dbiw('mydb:users')->cursor;
while (my $row = $cursor->next) {
    # process row
}
```

#### count()

Returns COUNT(*):

```perl
my $n = dbiw('mydb:users')->find({ status => 'active' })->count;
```

#### sum(column)

Returns SUM(column):

```perl
my $total = dbiw('mydb:orders')->find({ user_id => 42 })->sum('amount');
```

---

### ResultSet — Data Modification

#### insert(data)

Inserts a row. Returns auto-generated primary key if present:

```perl
my $id = dbiw('mydb:users')->insert({
    name    => 'Alice',
    email   => 'alice@example.com',
    created => dbiw->now,
});
```

#### update(data)

Updates rows matching the current WHERE clause:

```perl
dbiw('mydb:users')->find({ status => 'inactive' })
                   ->update({ status => 'deleted' });

# With SQL::Wizard expressions
dbiw('mydb:users')->find({ id => 42 })
                   ->update({ views => dbiw->col('views') + 1 });
```

#### delete()

Deletes rows matching the current WHERE clause:

```perl
dbiw('mydb:users')->find({ status => 'deleted' })->delete;
```

#### truncate()

Truncates the table:

```perl
dbiw('mydb:temp_data')->truncate;
```

---

### Expression Helpers via dbiw

When called without arguments, `dbiw` acts as a SQL::Wizard expression builder:

```perl
dbiw->col('u.name')                    # column reference
dbiw->val(42)                          # bound value
dbiw->raw('NOW() - INTERVAL 1 DAY')   # raw SQL
dbiw->func('COUNT', '*')              # function call
dbiw->now                              # NOW()
dbiw->coalesce('nickname', dbiw->val('Anonymous'))
dbiw->between('age', 18, 65)
dbiw->exists($subquery)
dbiw->any($subquery)
dbiw->all($subquery)
dbiw->case(...)
dbiw->col('price') * dbiw->col('qty')  # arithmetic
```

These return SQL::Wizard expression objects that can be used in find(), having(), sort(), all(), insert(), and update().

---

## Module Structure

```
DBIx::Wizard              — exports dbiw(), creates ResultSet or SQL::Wizard
DBIx::Wizard::DB          — database connection management (declare, dbh, cached handles)
DBIx::Wizard::DB::Table   — table metadata (column names, types, auto PK, time/decimal cols)
DBIx::Wizard::ResultSet   — query builder + executor (find, sort, all, insert, update, etc.)
DBIx::Wizard::Cursor      — lazy row iterator
```

---

## What DBIx::Wizard Intentionally Does NOT Do

- **No schema definition** — it doesn't manage CREATE TABLE or migrations. Use your database's native tools.
- **No relationship mapping** — no `has_many`, `belongs_to`. Use joins explicitly.
- **No result object classes** — rows are plain hashrefs, not objects. This is by design for simplicity and performance.
- **No query caching** — each execution hits the database. Cache at the application level if needed.
- **No connection pooling** — relies on DBI's connection caching. Use DBIx::Connector or similar if needed.

---

## Debug Support

Controlled via environment variables:

```bash
DEBUG_DBIW=1                    # print SQL to STDOUT
DEBUG_DBIW_FILE=/tmp/sql.log    # log SQL to file
```

---

## Open Questions

1. **Should ResultSet support UNION/CTE?** The underlying SQL::Wizard supports it, but is it needed at the ResultSet level or should users drop down to SQL::Wizard for complex queries?

2. **Cursor column flattening** — should `cursor('name')` return scalars from `next()` instead of hashrefs, matching `all('name')` behavior?

3. **Transaction support** — should DBIx::Wizard provide `dbiw('db')->transaction(sub { ... })` or leave it to raw DBI?

4. **ClickHouse support** — ClickHouse has specific workarounds (ALTER TABLE UPDATE, COUNT() fallbacks). Should these be dialect-aware or separate methods?
