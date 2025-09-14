# Safe Ecto Migrations

A non-exhaustive guide on common migration recipes and how to avoid trouble.

- [Adding an index](#adding-an-index)
- [Adding a reference or foreign key](#adding-a-reference-or-foreign-key)
- [Adding a column with a default value](#adding-a-column-with-a-default-value)
- [Changing a column's default value](#changing-a-columns-default-value)
- [Changing the type of a column](#changing-the-type-of-a-column)
- [Removing a column](#removing-a-column)
- [Renaming a column](#renaming-a-column)
- [Renaming a table](#renaming-a-table)
- [Adding a check constraint](#adding-a-check-constraint)
- [Setting NOT NULL on an existing column](#setting-not-null-on-an-existing-column)
- [Adding a JSON column](#adding-a-json-column)
- [Squashing migrations](#squashing-migrations)

Read more about safe migration techniques:

- [Migration locks in "Anatomy of a Migration"](./Anatomy.md)
- [How to backfill data and change data in bulk (aka: DML)](./Backfilling.md)

---

## Adding an index

Creating an index will [block writes](https://www.postgresql.org/docs/8.2/sql-createindex.html) to the table in Postgres.

MySQL is concurrent by default since [5.6](https://downloads.mysql.com/docs/mysql-5.6-relnotes-en.pdf) unless using `SPATIAL` or `FULLTEXT` indexes, which then it [blocks reads and writes](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html#online-ddl-index-syntax-notes).

**BAD ❌**

```elixir
def change do
  create index("posts", [:slug])

  # This obtains a ShareLock on "posts" which will block writes to the table
end
```

**GOOD ✅**

With Postgres, instead create the index concurrently which does not block writes.
There are two options:

**Option 1**

[Configure the Repo to use advisory locks](https://hexdocs.pm/ecto_sql/Ecto.Adapters.Postgres.html#module-migration-options) for locking migrations while running. Advisory locks are application-controlled database-level locks, and EctoSQL since v3.9.0 provides an option to use this type of lock. This is the safest option as it avoids the trade-off in Option 2.

Disable the DDL transaction in the migration to avoid a database transaction which is not compatible with `CONCURRENTLY` database operations.

```elixir
# in config/config.exs
config MyApp.Repo, migration_lock: :pg_advisory_lock

# in the migration
@disable_ddl_transaction true

def change do
  create index("posts", [:slug], concurrently: true)
end
```

If you're using Phoenix and PhoenixEcto, you will likely appreciate disabling
the migration lock in the CheckRepoStatus plug during dev to avoid hitting and
waiting on the advisory lock with concurrent web processes. You can do this by
adding `migration_lock: false` to the CheckRepoStatus plug in your
`MyAppWeb.Endpoint`.

**Option 2**

Disable the DDL transaction and the migration lock for the migration. By default, EctoSQL with Postgres will run migrations with a DDL transaction and a migration lock which also (by default) uses another transaction. You must disable both of these database transactions to use `CONCURRENTLY`. However, disabling the migration lock will allow competing nodes to try to run the same migration at the same time (eg, in a multi-node Kubernetes environment that runs migrations before startup). Therefore, some nodes may fail startup for a variety of reasons.

```elixir
@disable_ddl_transaction true
@disable_migration_lock true

def change do
  create index("posts", [:slug], concurrently: true)
end
```

For either option chosen, the migration may still take a while to run, but reads and updates to rows will continue to work. For example, for 100,000,000 rows it took 165 seconds to add run the migration, but SELECTS and UPDATES could occur while it was running.

**Do not have other changes in the same migration**; only create the index concurrently and separate other changes to later migrations.

---

## Adding a reference or foreign key

Adding a foreign key blocks writes on both tables.

**BAD ❌**

```elixir
def change do
  alter table("posts") do
    add :group_id, references("groups")
    # Obtains a ShareRowExclusiveLock which blocks writes on both tables
  end
end
```


**GOOD ✅**

In the first migration

```elixir
def change do
  alter table("posts") do
    add :group_id, references("groups", validate: false)
    # Obtains a ShareRowExclusiveLock which blocks writes on both tables.
  end
end
```

In the second migration

```elixir
def change do
  execute "ALTER TABLE posts VALIDATE CONSTRAINT group_id_fkey", ""
  # Obtains a ShareUpdateExclusiveLock which doesn't block reads or writes
end
```

These migrations can be in the same deployment, but make sure they are separate migrations.

**Note on empty tables**: when the table creating the referenced column is empty, you may be able to
create the column and validate at the same time since the time difference would be milliseconds
which may not be noticeable, no matter if you have 1 million or 100 million records in the referenced table.

**Note on populated tables**: the biggest difference depends on your scale. For 1 million records in
both tables, you may lock writes to both tables when creating the column for milliseconds
(you should benchmark for yourself) which could be acceptable for you. However, once your table has
100+ million records, the difference becomes seconds which is more likely to be felt and cause timeouts.
The differentiating metric is the time that both tables are locked from writes. Therefore, err on the side
of safety and separate constraint validation from referenced column creation when there is any data in the table.

---

## Adding a column with a default value

Adding a column with a default value to an existing table may cause the table to be rewritten. During this time, reads and writes are blocked in Postgres, and writes are blocked in MySQL and MariaDB. If the default column is an expression (volatile value) it will remain unsafe.

**BAD ❌**

Note: This becomes safe for non-volatile (static) defaults in:

- [Postgres 11+](https://www.postgresql.org/docs/release/11.0/). Default applies to INSERT since 7.x, and UPDATE since 9.3.
- MySQL 8.0.12+
- MariaDB 10.3.2+

```elixir
def change do
  alter table("comments") do
    add :approved, :boolean, default: false
    # This took 10 minutes for 100 million rows with no fkeys,

    # Obtained an AccessExclusiveLock on the table, which blocks reads and
    # writes.
  end
end
```

```elixir
def change do
  alter table("comments") do
    add :some_timestamp, :utc_datetime, default: fragment("now()")
    # A volatile value
  end
end
```

**GOOD ✅**

Add the column first, then alter it to include the default.

First migration:

```elixir
def change do
  alter table("comments") do
    add :approved, :boolean
    # This took 0.27 milliseconds for 100 million rows with no fkeys,
  end
end
```

Second migration:

```elixir
def change do
  execute "ALTER TABLE comments ALTER COLUMN approved SET DEFAULT false",
          "ALTER TABLE comments ALTER COLUMN approved DROP DEFAULT"
  # This took 0.28 milliseconds for 100 million rows with no fkeys,
end
```

Note: we cannot use [`modify/3`](https://hexdocs.pm/ecto_sql/Ecto.Migration.html#modify/3) as it will include updating the column type as
well unnecessarily, causing Postgres to rewrite the table. For more information,
[see this example](https://github.com/fly-apps/safe-ecto-migrations/issues/10).

Schema change to read the new column:

```diff
schema "comments" do
+ field :approved, :boolean, default: false
end
```

---

## Changing a column's default value

Changing an existing column's default may risk rewriting the table.

**BAD ❌**

```elixir
def change do
  alter table("comments") do
    # Previously, the default was `true`
    modify :approved, :boolean, default: false
    # This took 10 minutes for 100 million rows with no fkeys,

    # Obtained an AccessExclusiveLock on the table, which blocks reads and
    # writes.
  end
end
```

The issue is that we cannot use [`modify/3`](https://hexdocs.pm/ecto_sql/Ecto.Migration.html#modify/3) as it will include updating the column type as
well unnecessarily, causing Postgres to rewrite the table. For more information,
[see this example](https://github.com/fly-apps/safe-ecto-migrations/issues/10).

**GOOD ✅**

Execute raw sql instead to alter the default:

```elixir
def change do
  execute "ALTER TABLE comments ALTER COLUMN approved SET DEFAULT false",
          "ALTER TABLE comments ALTER COLUMN approved SET DEFAULT true"
  # This took 0.28 milliseconds for 100 million rows with no fkeys,
end
```

> ![NOTE]
> This will not update the values of rows previously-set by the old default. This value has been materialized at the time of insert/update and therefore has no distinction between whether it was set by the column `DEFAULT` or set by the original operation.
>
> If you want to update the default of already-written rows, you must distinguish them somehow and modify them with a [backfill](./Backfilling.md)

---

## Changing the type of a column

Changing the type of a column may cause the table to be rewritten. During this time, reads and writes are blocked in Postgres, and writes are blocked in MySQL and MariaDB.

**BAD ❌**

Safe in Postgres:

- increasing length on varchar or removing the limit
- changing varchar to text
- changing text to varchar with no length limit
- Postgres 9.2+ - increasing precision (NOTE: not scale) of decimal or numeric columns. eg, increasing 8,2 to 10,2 is safe. Increasing 8,2 to 8,4 is not safe.
- Postgres 9.2+ - changing decimal or numeric to be unconstrained
- Postgres 12+ - changing timestamp to timestamptz when session TZ is UTC

Safe in MySQL/MariaDB:

- increasing length of varchar from < 255 up to 255.
- increasing length of varchar from > 255 up to max.

```elixir
def change do
  alter table("posts") do
    modify :my_column, :boolean, from: :text
  end
end
```

**GOOD ✅**

Take a phased approach:

1. Create a new column
1. In application code, write to both columns
1. Backfill data from old column to new column
1. In application code, move reads from old column to the new column
1. In application code, remove old column from Ecto schemas.
1. Drop the old column.

---

## Removing a column

If Ecto is still configured to read a column in any running instances of the application, then queries will fail when loading data into your structs. This can happen in multi-node deployments or if you start the application before running migrations.

**BAD ❌**

```elixir
# Without a code change to the Ecto Schema

def change do
  alter table("posts") do
    remove :no_longer_needed_column
  end
end
```


**GOOD ✅**

Safety can be assured if the application code is first updated to remove references to the column so it's no longer loaded or queried. Then, the column can safely be removed from the table.

1. Deploy code change to remove references to the field.
1. Deploy migration change to remove the column.

First deployment:

```diff
# First deploy, in the Ecto schema

defmodule MyApp.Post do
  schema "posts" do
-   column :no_longer_needed_column, :text
  end
end
```

Second deployment:

```elixir
def change do
  alter table("posts") do
    remove :no_longer_needed_column
  end
end
```

---

## Renaming a column

Ask yourself: "Do I _really_ need to rename a column?". Probably not, but if you must, read on and be aware it requires time and effort.

If Ecto is configured to read a column in any running instances of the application, then queries will fail when loading data into your structs. This can happen in multi-node deployments or if you start the application before running migrations.

There is a shortcut: Don't rename the database column, and instead rename the schema's field name and configure it to point to the database column.

**BAD ❌**

```elixir
# In your schema
schema "posts" do
  field :summary, :text
end


# In your migration
def change do
  rename table("posts"), :title, to: :summary
end
```

The time between your migration running and your application getting the new code may encounter trouble.

**GOOD ✅**

**Strategy 1**

Rename the field in the schema only, and configure it to point to the database column and keep the database column the same. Ensure all calling code relying on the old field name is also updated to reference the new field name.

```elixir
defmodule MyApp.MySchema do
  use Ecto.Schema

  schema "weather" do
    field :temp_lo, :integer
    field :temp_hi, :integer
    field :precipitation, :float, source: :prcp
    field :city, :string

    timestamps(type: :naive_datetime_usec)
  end
end
```

```diff
## Update references in other parts of the codebase:
   my_schema = Repo.get(MySchema, "my_id")
-  my_schema.prcp
+  my_schema.precipitation
```

**Strategy 2**

Take a phased approach:

1. Create a new column
1. In application code, write to both columns
1. Backfill data from old column to new column
1. In application code, move reads from old column to the new column
1. In application code, remove old column from Ecto schemas.
1. Drop the old column.

---

## Renaming a table

Ask yourself: "Do I _really_ need to rename a table?". Probably not, but if you must, read on and be aware it requires time and effort.

If Ecto is still configured to read a table in any running instances of the application, then queries will fail when loading data into your structs. This can happen in multi-node deployments or if you start the application before running migrations.

There is a shortcut: rename the schema only, and do not change the underlying database table name.

**BAD ❌**

```elixir
def change do
  rename table("posts"), to: table("articles")
end
```

**GOOD ✅**

**Strategy 1**

Rename the schema only and all calling code, and don’t rename the table:

```diff
- defmodule MyApp.Weather do
+ defmodule MyApp.Forecast do
  use Ecto.Schema

  schema "weather" do
    field :temp_lo, :integer
    field :temp_hi, :integer
    field :precipitation, :float, source: :prcp
    field :city, :string

    timestamps(type: :naive_datetime_usec)
  end
end

# and in calling code:
- weather = MyApp.Repo.get(MyApp.Weather, “my_id”)
+ forecast = MyApp.Repo.get(MyApp.Forecast, “my_id”)
```

**Strategy 2**

Take a phased approach:

1. Create the new table. This should include creating new constraints (checks and foreign keys) that mimic behavior of the old table.
1. In application code, write to both tables, continuing to read from the old table.
1. Backfill data from old table to new table
1. In application code, move reads from old table to the new table
1. In application code, remove the old table from Ecto schemas.
1. Drop the old table.

---

## Adding a check constraint

Adding a check constraint blocks reads and writes to the table in Postgres, and blocks writes in MySQL/MariaDB while every row is checked.

**BAD ❌**

```elixir
def change do
  create constraint("products", :price_must_be_positive, check: "price > 0")
  # Creating the constraint with validate: true (the default when unspecified)
  # will perform a full table scan and acquires a lock preventing updates
end
```

**GOOD ✅**

There are two operations occurring:

1. Creating a new constraint for new or updating records
1. Validating the new constraint for existing records

If these commands are happening at the same time, it obtains a lock on the table as it validates the entire table and fully scans the table. To avoid this full table scan, we can separate the operations.

In one migration:

```elixir
def change do
  create constraint("products", :price_must_be_positive, check: "price > 0", validate: false)
  # Setting validate: false will prevent a full table scan, and therefore
  # commits immediately.
end
```

In the next migration:

```elixir
def change do
  execute "ALTER TABLE products VALIDATE CONSTRAINT price_must_be_positive", ""
  # Acquires SHARE UPDATE EXCLUSIVE lock, which allows updates to continue
end
```

These can be in the same deployment, but ensure there are 2 separate migrations.

---

## Setting NOT NULL on an existing column

Setting NOT NULL on an existing column blocks reads and writes while every row is checked.  Just like the Adding a check constraint scenario, there are two operations occurring:

1. Creating a new constraint for new or updating records
1. Validating the new constraint for existing records

To avoid the full table scan, we can separate these two operations.

**BAD ❌**

```elixir
def change do
  alter table("products") do
    modify :active, :boolean, null: false
  end
end
```

**GOOD ✅**

Add a check constraint without validating it, backfill data to satiate the constraint and then validate it. This will be functionally equivalent.

In the first migration:

```elixir
# Deployment 1
def change do
  create constraint("products", :active_not_null, check: "active IS NOT NULL", validate: false)
end
```

This will enforce the constraint in all new rows, but not care about existing rows until that row is updated.

You'll likely need a data migration at this point to ensure that the constraint is satisfied.

Then, in the next deployment's migration, we'll enforce the constraint on all rows:

```elixir
# Deployment 2
def change do
  execute "ALTER TABLE products VALIDATE CONSTRAINT active_not_null", ""
end
```

If you're using Postgres 12+, you can add the NOT NULL to the column after validating the constraint. From the Postgres 12 docs:

> SET NOT NULL may only be applied to a column provided
> none of the records in the table contain a NULL value
> for the column. Ordinarily this is checked during the
> ALTER TABLE by scanning the entire table; however, if
> a valid CHECK constraint is found which proves no NULL
> can exist, then the table scan is skipped.

**However** we cannot use [`modify/3`](https://hexdocs.pm/ecto_sql/Ecto.Migration.html#modify/3)
as it will include updating the column type as well unnecessarily, causing
Postgres to rewrite the table. For more information, [see this example](https://github.com/fly-apps/safe-ecto-migrations/issues/10).

```elixir
# **Postgres 12+ only**

def change do
  execute "ALTER TABLE products VALIDATE CONSTRAINT active_not_null",
          ""

  execute "ALTER TABLE products ALTER COLUMN active SET NOT NULL",
          "ALTER TABLE products ALTER COLUMN active DROP NOT NULL"

  drop constraint("products", :active_not_null)
end
```

If your constraint fails, then you should consider backfilling data first to cover the gaps in your desired data integrity, then revisit validating the constraint.

---

## Adding a JSON column

In Postgres, there is no equality operator for the json column type, which can cause errors for existing SELECT DISTINCT queries in your application.

**BAD ❌**

```elixir
def change do
  alter table("posts") do
    add :extra_data, :json
  end
end
```

**GOOD ✅**

Use jsonb instead. Some say it’s like “json” but “better.”

```elixir
def change do
  alter table("posts") do
    add :extra_data, :jsonb
  end
end
```

---

## Squashing Migrations

If you have a long list of migrations, sometimes it can take a while to migrate
each of those files every time the project is reset or spun up by a new
developer. Thankfully, Ecto comes with mix tasks to `dump` and `load` a database
structure which will represent the state of the database up to a certain point
in time, not including content.

- [mix ecto.dump](mix_ecto_dump)
- [mix ecto.load](mix_ecto_load)

Schema dumping and loading is only supported by external binaries `pg_dump` and
`mysqldump`, which are used by the Postgres, MyXQL, and MySQL Ecto adapters (not
supported in MSSQL adapter).

For example:

```
20210101000000 - First Migration
20210201000000 - Second Migration
20210701000000 - Third Migration <-- we are here now. run `mix ecto.dump`
```

We can "squash" the migrations up to the current day which will effectively
fast-forward migrations to that structure. The Ecto Migrator will detect that
the database is already migrated to the third migration, and so it begins there
and migrates forward.

Let's add a new migration:

```
20210101000000 - First Migration
20210201000000 - Second Migration
20210701000000 - Third Migration <-- `structure.sql` represents up to here
20210801000000 - New Migration <-- This is where migrations will begin
```

The new migration will still run, but the first-through-third migrations will
not need to be run since the structure already represents the changes applied by
those migrations. At this point, you can safely delete the first, second, and
third migration files or keep them for historical auditing.

Let's make this work:

1. Run `mix ecto.dump` which will dump the current structure into
   `priv/repo/structure.sql` by default. [Check the mix task for more
   options](mix_ecto_dump).
2. During project setup with an empty database, run `mix ecto.load` to load
   `structure.sql`.
3. Run `mix ecto.migrate` to run any additional migrations created after the
   structure was dumped.

To simplify these actions into one command, we can leverage mix aliases:

```elixir
# mix.exs

defp aliases do
  [
    "ecto.reset": ["ecto.drop", "ecto.setup"],
    "ecto.setup": ["ecto.load", "ecto.migrate"],
    # ...
  ]
end
```

Now you can run `mix ecto.setup` and it will load the database structure and run
remaining migrations. Or, run `mix ecto.reset` and it will drop and run setup.
Of course, you can continue running `mix ecto.migrate` as you create them.

[mix_ecto_dump]: https://hexdocs.pm/ecto_sql/Mix.Tasks.Ecto.Dump.html
[mix_ecto_load]: https://hexdocs.pm/ecto_sql/Mix.Tasks.Ecto.Load.html

---

# Credits

Created and written by David Bernheisel with recipes heavily inspired from Andrew Kane and his library [strong_migrations](https://github.com/ankane/strong_migrations).

[PostgreSQL at Scale by James Coleman](https://medium.com/braintree-product-technology/postgresql-at-scale-database-schema-changes-without-downtime-20d3749ed680)

[Strong Migrations by Andrew Kane](https://github.com/ankane/strong_migrations)

[Adding a NOT NULL CONSTRAINT on PG Faster with Minimal Locking by Christophe Escobar](https://medium.com/doctolib/adding-a-not-null-constraint-on-pg-faster-with-minimal-locking-38b2c00c4d1c)

[Postgres Runtime Configuration](https://www.postgresql.org/docs/current/runtime-config-client.html)

[Automatic and Manual Ecto Migrations by Wojtek Mach](https://dashbit.co/blog/automatic-and-manual-ecto-migrations)

Special thanks for sponsorship:

* Fly.io

Special thanks for these reviewers:

* Steve Bussey
* Stephane Robino
* Dennis Beatty
* Wojtek Mach
* Mark Ericksen
* And [all the contributors](https://github.com/fly-apps/safe-ecto-migrations/graphs/contributors)

# Reference Material

[Postgres Lock Conflicts](https://www.postgresql.org/docs/12/explicit-locking.html)

|  |  **Current Lock →** | | | | | | | |
|---------------------|-------------------|-|-|-|-|-|-|-|
| **Requested Lock ↓** | ACCESS SHARE | ROW SHARE | ROW EXCLUSIVE | SHARE UPDATE EXCLUSIVE | SHARE | SHARE ROW EXCLUSIVE | EXCLUSIVE | ACCESS EXCLUSIVE |
| ACCESS SHARE           |   |   |   |   |   |   |   | X |
| ROW SHARE              |   |   |   |   |   |   | X | X |
| ROW EXCLUSIVE          |   |   |   |   | X | X | X | X |
| SHARE UPDATE EXCLUSIVE |   |   |   | X | X | X | X | X |
| SHARE                  |   |   | X | X |   | X | X | X |
| SHARE ROW EXCLUSIVE    |   |   | X | X | X | X | X | X |
| EXCLUSIVE              |   | X | X | X | X | X | X | X |
| ACCESS EXCLUSIVE       | X | X | X | X | X | X | X | X |

- `SELECT` acquires a `ACCESS SHARE` lock
- `SELECT FOR UPDATE` acquires a `ROW SHARE` lock
- `UPDATE`, `DELETE`, and `INSERT` will acquire a `ROW EXCLUSIVE` lock
- `CREATE INDEX CONCURRENTLY` and `VALIDATE CONSTRAINT` acquires `SHARE UPDATE EXCLUSIVE`
- `CREATE INDEX` acquires `SHARE` lock

Knowing this, let's re-think the above table:

|  |  **Current Operation →** | | | | | | | |
|---------------------|-------------------|-|-|-|-|-|-|-|
| **Blocks Operation ↓** | `SELECT` | `SELECT FOR UPDATE` | `UPDATE` `DELETE` `INSERT` | `CREATE INDEX CONCURRENTLY`  `VALIDATE CONSTRAINT` | `CREATE INDEX` | SHARE ROW EXCLUSIVE | EXCLUSIVE | `ALTER TABLE` `DROP TABLE` `TRUNCATE` `REINDEX` `CLUSTER` `VACUUM FULL` |
| `SELECT` |   |   |   |   |   |   |   | X |
| `SELECT FOR UPDATE` |   |   |   |   |   |   | X | X |
| `UPDATE` `DELETE` `INSERT` |   |   |   |   | X | X | X | X |
| `CREATE INDEX CONCURRENTLY` `VALIDATE CONSTRAINT` |   |   |   | X | X | X | X | X |
| `CREATE INDEX` |   |   | X | X |   | X | X | X |
| SHARE ROW EXCLUSIVE |   |   | X | X | X | X | X | X |
| EXCLUSIVE |   | X | X | X | X | X | X | X |
| `ALTER TABLE` `DROP TABLE` `TRUNCATE` `REINDEX` `CLUSTER` `VACUUM FULL` | X | X | X | X | X | X | X | X |
