# sqltyper - Type your SQL queries!

SQL is a typed language. sqltyper takes raw PostgreSQL queries and
generates TypeScript functions that run those queries AND are typed
correctly, based on the database schema.

For example, given the following schema:

```sql
CREATE TABLE person (
  name text NOT NULL,
  age integer NOT NULL,
  shoe_size integer
)
```

The following SQL query in `find-persons.sql`:

```sql
SELECT initcap(name) as name_capitalized, age, shoe_size
FROM person
WHERE
    name LIKE ${namePattern} AND
    age > ${minimumAge}
```

Converts to `find-persons.ts`:

```typescript
import { ClientBase } from 'pg'

export function findPersons(
  client: ClientBase,
  params: {
    namePattern: string
    minimumAge: number
  },
): Promise<Array<{
  name_capitalized: string
  age: number
  shoe_size: number | null
}>> { ... }
```

sqltyper does this without actually executing your query, so it's
perfectly safe to use in any environment.


## Installation

```
npm install --save pg
npm install --save-dev sqltyper
```

Or:

```
yarn add pg
yarn add --dev sqltyper
```

sqltyper generates TypeScript code, so it isn't needed on
application runtime. However, the generated TypeScript code uses
[node-postgres] to execute the queries, so `pg` is a required runtime
dependency.


[node-postgres]: https://node-postgres.com/


## Tutorial

Assuming you have a TypeScrip app and a bunch of SQL queries, put them
in files in a single directory, like this:

```
src/
|-- app.ts
|-- ...
`-- sqls/
    |-- my-query.sql
    `-- other-query.sql
```

Run sqltyper on the `sqls` directory:

```
yarn sqltyper --database postgres://user:pass@host/dbname src/sqls 

# or npx sqltyper, or ./node_modules/.bin/sqltyper, ...
```

sqltyper connects to the PostgreSQL database you give in the
`--database` option, finds out the input and output types of each of
the SQL queries, and outputs the corresponding TypeScript functions in
the same directory.

You should now have the following files:
```
src/
|-- app.ts
|-- ...
`-- sqls/
    |-- index.ts
    |-- my-query.sql
    |-- my-query.ts
    |-- other-query.sql
    `-- other-query.ts
```

Each `.sql` file got a `.ts` file next to it. Each `.ts` file exports
a single function, whose name is the `.sql` file name with the
extension removed and camelCased. Furthermore, it generates an
`index.ts` file that re-exports all these functions.

In `app.ts`, import the SQL query functions:

```
import * as sql from './sql'
```

And that's it! Now you can use `sql.myQuery()` and `sql.otherQuery()`
to run the queries in a type-safe manner.

These functions a `Client` or `Pool` from [node-postgres] as the first
argument and possible query parameters as the second parameter.

It will return one of the following, wrapped in a `Promise`:

- An array of result objects, with object keys corresponding to output
  column names. Note that all of the output columns in your query must
  have a unique name, because otherwise some of them would be not
  accessible.

- A single result row or `null` if the query only ever returns zero or
  one row (e.g. `SELECT` query with `LIMIT 1`).

- A number which denotes the number of affected rows (`INSERT`,
  `UPDATE` or `DELETE` without a `RETURNING` clause).


## CLI

```
sqltyper [options] DIRECTORY...
```

Generate TypeScript functions for SQL statements in all files in the
given directories. For each input file, the output file name is
generated by removing the file extension and appending `.ts`.

Each output file will export a single function whose name is a
camelCased version of the basename of the input file.

sqltyper connects to the database to infer the parameter and output
column types of each SQL statement. It does this without actually
executing the SQL queries, so it's safe to run against any database.

Options:

`--database`, `-d`

Database URI to connect to, e.g. `-d postgres://user:pass@localhost:5432/mydb`.
By default, uses the [connecting logic] of node-postgres that
relies on environment variables.

`--ext`, `-e`

File extensions to consider, e.g. `-e sql,psql`. Default: `sql`.

`--verbose`, `-v`

Give verbose output about problems with inferring statement
nullability. Default: `false`.

`--watch`, `-w`

Watch files and run the conversion when something changes. Default:
`false`.

`--prettify`, `-p`

Apply `prettier` to output TypeScript files. `prettier` must be
installed and configured for your project. Default: `false`.

`--index`

Whether to generate and `index.ts` file that re-exports all the
generated functions. Default: `true`.

`--pg-module`

Where to import node-postgres from. Default: `pg`.

[connecting logic]: https://node-postgres.com/features/connecting


## How does it work?

sqltyper connects to your database to look up the schema: which
types there are, which tables there are, what columns and constraints
the tables have, etc. The only queries it executes look up this
information from various `pg_catalog.*` tables.

First, it substitutes any `${paramName}` strings with `$1`, `$2`, etc.

Then, it creates a prepared statement from the query, and then asks
PostgreSQL to describe the prepared statement. PostgreSQL will reply
with parameter types for `$1`, `$2`, etc., and columns types of the
result rows.

However, this is not enough! In SQL basically anything anywhere can be
`NULL`, so if sqltyper stopped here all the types would have to be
e.g. `integer | null`, `string | null` and so on. For this reason,
sqltyper also parses the SQL query with its built-in SQL parser and
then starts finding out which expressions can never be `NULL`. It
employs `NOT NULL` constraints, nullability guarantees of functions
and operators, `WHERE` clause expressions, etc. to rule out as many
possibilities of `NULL` as possible, and amends the original statement
description with this information.

It also uses the parsing result to find out the possible number of
results. For example, `UPDATE`, `DELETE` and `INSERT` queries without
a `RETURNING` clause will return the number of affected rows instead
of any columns. Furthermore, a `SELECT` query with `LIMIT 1` will
return `{ ... } | null` instead of `Array<{ ... }>`.

Then, it outputs a TypeScript function that is correctly typed, and
when run, executes your query and converts input and output data
to/from PostgreSQL.


## Prior art

The main motivator for sqltyper was [sqlτyped] by @jonifreeman. It
does more or less the same as sqltyper, but for Scala, and meant to be
used with MySQL. It uses JDBC, and is implemented as a Scala macro
rather than an offline code generation tool.

[sqlτyped]: https://github.com/jonifreeman/sqltyped
