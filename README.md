# DBIx::Wizard - Lightweight, chainable, zero-configuration ORM for Perl

## Synopsis

    use DBIx::Wizard;
    use DBIx::Wizard::DB;

    # Register a database
    DBIx::Wizard::DB->declare('mydb',
        'dbi:mysql:dbname=myapp', 'user', 'password',
        { RaiseError => 1 },
    );

    # Or via environment variable (no code needed)
    # export DBIW_DECLARE_MYDB="dbi:mysql:dbname=myapp|user|password"

    # Query
    my @active = dbiw('mydb:users')->where({ status => 'active' })->all;
    my @names  = dbiw('mydb:users')->order_by('name')->all('name');
    my $alice  = dbiw('mydb:users')->where({ name => 'Alice' })->one;
    my $count  = dbiw('mydb:users')->where({ status => 'active' })->count;

    # Insert
    my $id = dbiw('mydb:users')->insert({
        name  => 'Alice',
        email => 'alice@example.com',
    });

    # Update
    dbiw('mydb:users')->where({ id => 42 })
                       ->update({ status => 'inactive' });

    # Update with expressions
    dbiw('mydb:users')->where({ id => 42 })
                       ->update({ views => dbiw->col('views') + 1 });

    # Delete
    dbiw('mydb:users')->where({ status => 'deleted' })->delete;

    # Joins
    my @rows = dbiw('mydb:users')
        ->as('u')
        ->join('orders|o' => 'o.user_id = u.id')
        ->where({ 'u.status' => 'active' })
        ->order_by('-o.created_at')
        ->limit(20)
        ->all;

## Description

DBIx::Wizard is a thin, fluent database query executor for Perl. It sits
between raw DBI (too verbose) and DBIx::Class (too heavy), providing a
chainable API that requires zero schema definition.

It uses [SQL::Wizard](https://metacpan.org/pod/SQL%3A%3AWizard) for SQL generation, inheriting its multi-layer
injection protection, composable expression trees, and support for joins,
subqueries, CTEs, window functions, and more.

Key properties:

- **Zero schema definition.** Point it at a database and table, start
querying. No result classes, no relationship declarations.
- **Fluent chaining.** Build queries with `->where->order_by->limit->all`.
- **SQL::Wizard expressions everywhere.** Use `dbiw->col(...)`,
`dbiw->func(...)`, `dbiw->now`, and arithmetic in any position.
- **Plain hashrefs.** Rows are returned as plain Perl hashrefs, not
objects. Fast and simple.
- **Pluggable DateTime inflation.** Automatically inflate time columns
to objects, with the class configurable at three levels.
- **Environment-based configuration.** Databases can be declared via
environment variables following the twelve-factor app principle.

## Exported Functions

### Dbiw

    # With argument: returns a ResultSet
    my $rs = dbiw('mydb:users');

    # Without argument: returns a SQL::Wizard instance
    my $expr = dbiw->col('u.name');
    my $now  = dbiw->now;
    my $func = dbiw->func('COUNT', '*');

When called with a `'db:table'` string, returns a
[DBIx::Wizard::ResultSet](https://metacpan.org/pod/DBIx%3A%3AWizard%3A%3AResultSet) object tied to that database and table.

When called without arguments, returns a shared [SQL::Wizard](https://metacpan.org/pod/SQL%3A%3AWizard) instance
for building expressions. These expressions can be used in `where()`,
`update()`, `order_by()`, `having()`, and column lists.

## Database Registration

Before querying, databases must be registered by name.

### Code-based Declaration

    use DBIx::Wizard::DB;

    DBIx::Wizard::DB->declare('mydb',
        'dbi:mysql:dbname=myapp',     # DSN
        'user',                        # username
        'password',                    # password
        {                              # DBI options
            RaiseError    => 1,
            inflate_class => 'My::DateTime',   # optional, see DATETIME INFLATION
        },
    );

### Environment-based Declaration

    export DBIW_DECLARE_MYDB="dbi:mysql:dbname=myapp|user|password"
    export DBIW_DECLARE_ANALYTICS="dbi:Pg:dbname=stats|readonly|pass"
    export DBIW_DECLARE_LOGS="dbi:SQLite:dbname=/tmp/logs.db||"

The format is `DBIW_DECLARE_<NAME>=<dsn>|<user>|<password>`. The
`<NAME>` is uppercased in the environment but maps to lowercase in
`dbiw()`. Empty user/password is supported with `||`.

Environment declarations are checked lazily on first connection. If both
`declare()` and an environment variable exist for the same name,
`declare()` takes precedence.

## Resultset — Query Building

All builder methods modify the ResultSet and return `$self` for chaining.

### as

    dbiw('mydb:users')->as('u')

Sets the table alias. The alias is used in the SQL FROM clause.

### Where

    ->where({ status => 'active' })
    ->where({ age => { '>' => 18 }, status => 'active' })
    ->where({ name => { -in => ['Alice', 'Bob'] } })

Sets or extends the WHERE clause. When called with a hashref, merges with
any existing conditions (AND). When called with an arrayref, replaces the
WHERE clause entirely.

Accepts all [SQL::Wizard](https://metacpan.org/pod/SQL%3A%3AWizard) WHERE syntax: hashrefs, arrayrefs with
`-and`/`-or`, operator hashrefs (`{ '>' => 5 }`), `{ -in => [...] }`,
`undef` for IS NULL, `{ '!=' => undef }` for IS NOT NULL, and
expression objects.

### join

    ->join('orders|o' => 'o.user_id = u.id')

Adds an INNER JOIN. The first argument is the table (with optional
`table|alias` syntax), the second is the ON condition. Multiple joins
can be chained:

    ->join('orders|o' => 'o.user_id = u.id')
    ->join('payments|p' => 'p.order_id = o.id')

### left\_join

    ->left_join('orders|o' => 'o.user_id = u.id')

Adds a LEFT JOIN. Same arguments as `join`.

### right\_join

    ->right_join('returns|r' => 'r.order_id = o.id')

Adds a RIGHT JOIN.

### full\_join

    ->full_join('archive|a' => 'a.user_id = u.id')

Adds a FULL OUTER JOIN.

### cross\_join

    ->cross_join('sizes')

Adds a CROSS JOIN (no ON condition).

### order\_by

    ->order_by('name')                           # ORDER BY name
    ->order_by('-created_at')                    # ORDER BY created_at DESC
    ->order_by('-created_at', 'name')            # ORDER BY created_at DESC, name
    ->order_by(dbiw->col('score')->desc)         # ORDER BY score DESC

Sets the ORDER BY clause. A leading `-` on a column name is shorthand for
DESC. Accepts expression objects.

### group\_by

    ->group_by('department')
    ->group_by('department', 'year')

Sets the GROUP BY clause.

### having

    ->having({ dbiw->raw('COUNT(*)') => { '>' => 5 } })

Sets the HAVING clause. Same syntax as `where()`.

### limit

    ->limit(20)

Sets the LIMIT. Must be a positive integer.

### offset

    ->offset(40)

Sets the OFFSET. Must be a positive integer.

### inflate

    ->inflate(1)    # enable (default)
    ->inflate(0)    # disable

Enables or disables DateTime inflation for time columns. See
["DATETIME INFLATION"](#datetime-inflation).

### inflate\_class

    ->inflate_class('My::DateTime')

Sets the DateTime inflation class for this ResultSet. See
["DATETIME INFLATION"](#datetime-inflation).

## Resultset — Data Retrieval

### all

    my @rows  = dbiw('mydb:users')->all;                   # all columns
    my @rows  = dbiw('mydb:users')->all(['name', 'email']); # specific columns
    my @names = dbiw('mydb:users')->all('name');            # flat list of values

Executes SELECT and returns all matching rows as hashrefs. When given a
single scalar column name (not an arrayref), returns a flat list of that
column's values instead.

### Distinct

    my @depts = dbiw('mydb:users')->distinct('department');
    my @pairs = dbiw('mydb:users')->distinct(['dept', 'city']);

Returns distinct values. Mirrors `all()`'s column argument: a scalar returns
a flat list of values, an arrayref returns hashrefs. Emits `SELECT DISTINCT`
under the hood.

A column argument is required — `->distinct` without columns would
produce `SELECT DISTINCT *`, which is almost never what you want.

### One

    my $row  = dbiw('mydb:users')->where({ id => 42 })->one;
    my $name = dbiw('mydb:users')->where({ id => 42 })->one('name');

Like `all()` but with LIMIT 1. Returns a single hashref, or a single
scalar value when given a column name.

### cursor

    my $cursor = dbiw('mydb:users')->cursor;
    while (my $row = $cursor->next) {
        # process row
    }

Returns a [DBIx::Wizard::Cursor](https://metacpan.org/pod/DBIx%3A%3AWizard%3A%3ACursor) for lazy row-by-row iteration. Useful
for large result sets that should not be loaded entirely into memory.

### count

    my $n = dbiw('mydb:users')->where({ status => 'active' })->count;

Returns the COUNT(\*) of matching rows.

### exists

    if (dbiw('mydb:users')->where({ email => $email })->exists) {
        ...
    }

Returns true if the query matches at least one row, false otherwise.
Equivalent to `count > 0`.

### Sum

    my $total = dbiw('mydb:orders')->where({ user_id => 42 })->sum('amount');

Returns the SUM of the given column.

## Resultset — Data Modification

### insert

    my $id = dbiw('mydb:users')->insert({
        name  => 'Alice',
        email => 'alice@example.com',
    });

Inserts a row and returns the auto-generated primary key if the table has
one (detected automatically via table metadata). Expression objects can be
used as values:

    dbiw('mydb:users')->insert({
        name       => 'Alice',
        created_at => dbiw->now,
    });

### update

    my $rows = dbiw('mydb:users')
                   ->where({ status => 'inactive' })
                   ->update({ status => 'deleted' });

Updates all rows matching the current WHERE clause. Returns the number of
affected rows. Expression objects work in the SET values:

    dbiw('mydb:counters')->where({ key => 'hits' })
                          ->update({ value => dbiw->col('value') + 1 });

### delete

    dbiw('mydb:users')->where({ status => 'deleted' })->delete;

Deletes all rows matching the current WHERE clause.

### Truncate

    dbiw('mydb:temp_data')->truncate;

Truncates the table. On SQLite (which does not support TRUNCATE TABLE),
falls back to `DELETE FROM`.

## Expressions

When called without arguments, `dbiw` returns a [SQL::Wizard](https://metacpan.org/pod/SQL%3A%3AWizard) instance.
This provides access to the full expression API:

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

These expression objects can be used in `where()`, `order_by()`, `having()`,
`all()` column lists, `insert()` values, and `update()` SET values.

See [SQL::Wizard](https://metacpan.org/pod/SQL%3A%3AWizard) for the complete expression API.

## Datetime Inflation

By default, columns with datetime, date, or timestamp types are inflated
to objects using the configured inflate class. The default class is
`Time::Moment`.

The inflate class is resolved in this order (first defined wins):

- 1. **ResultSet level** — `->inflate_class('My::DateTime')`
- 2. **Database level** — `inflate_class` option in `declare()`
- 3. **Package level** — `use DBIx::Wizard inflate_class => '...'`
or `DBIx::Wizard->default_inflate_class('...')`

The inflate class must implement `->new($datetime_string)`.

    # Package level (at use time)
    use DBIx::Wizard inflate_class => 'My::DateTime';

    # Package level (at runtime)
    DBIx::Wizard->default_inflate_class('My::DateTime');

    # Database level
    DBIx::Wizard::DB->declare('mydb', $dsn, $user, $pass, {
        inflate_class => 'My::DateTime',
    });

    # ResultSet level
    dbiw('mydb:users')->inflate_class('My::DateTime')->all;

    # Disable inflation entirely
    dbiw('mydb:users')->inflate(0)->all;

## Transactions

    dbiw('mydb')->transaction(sub {
        dbiw('mydb:users')->insert({ name => 'Alice', email => 'alice@test.com' });
        dbiw('mydb:orders')->insert({ user_id => 42, total => 99.95 });
    });

Wraps the code block in a BEGIN / COMMIT pair. If the block dies, the
transaction is rolled back and the exception is re-thrown.

Note: `dbiw('mydb')` (without a table) returns a database wrapper with
the `transaction` method.

### Nested Transactions

Nested calls to `transaction()` use savepoints. If the inner block dies,
only the inner savepoint is rolled back — the outer transaction can continue:

    dbiw('mydb')->transaction(sub {
        dbiw('mydb:users')->insert({ name => 'Dave' });

        eval {
            dbiw('mydb')->transaction(sub {
                dbiw('mydb:users')->insert({ name => 'Eve' });
                die "inner failure";
            });
        };
        # Dave is committed, Eve is rolled back
    });

## Debugging

SQL logging is controlled via environment variables:

    DEBUG_DBIW=1                    # print SQL to STDOUT
    DEBUG_DBIW_FILE=/tmp/sql.log    # log SQL to file

Each logged line includes the method name, the SQL statement, and the bind
parameters.

## cursor

[DBIx::Wizard::Cursor](https://metacpan.org/pod/DBIx%3A%3AWizard%3A%3ACursor) provides lazy row-by-row iteration:

    my $cursor = dbiw('mydb:users')->order_by('name')->cursor;
    while (my $row = $cursor->next) {
        print $row->{name}, "\n";
    }

`next()` returns a hashref for each row, with DateTime inflation and
decimal fixing applied. Returns `undef` when no more rows are available.

## Utility Methods

### dbh

    my $dbh = dbiw('mydb:users')->dbh;

Returns the raw [DBI](https://metacpan.org/pod/DBI) database handle for direct access.

## Inspiration

The closest equivalents in other languages are Ruby's Sequel, Python's
SQLAlchemy, and JavaScript's Knex. DBIx::Wizard brings that level of
ergonomic database access to Perl.

## See Also

- [SQL::Wizard](https://metacpan.org/pod/SQL%3A%3AWizard) — the SQL generation engine behind DBIx::Wizard
- [DBI](https://metacpan.org/pod/DBI) — the database interface DBIx::Wizard executes queries through
- [DBIx::Class](https://metacpan.org/pod/DBIx%3A%3AClass) — full ORM for when you need schema definitions and relationships
- [DBIx::Lite](https://metacpan.org/pod/DBIx%3A%3ALite) — Chained and minimal ORM
- [SQL::Abstract](https://metacpan.org/pod/SQL%3A%3AAbstract) — hash/arrayref to SQL, used by DBIx::Class

## Author

Thomas Busch <tbusch@cpan.org>

## License

MIT License

Copyright (c) 2026 Thomas Busch

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
