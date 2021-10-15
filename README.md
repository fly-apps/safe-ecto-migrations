# Safe Ecto Migrations

A non-prescriptive guide on common migration recipes and how to avoid trouble.

- [Adding an index](#adding-an-index)
- [Adding a reference or foreign key](#adding-a-reference-or-foreign-key)
- [Adding a column with a default value](#adding-a-column-with-a-default-value)
- [Changing the type of a column](#changing-the-type-of-a-column)
- [Removing a column](#removing-a-column)
- [Renaming a column](#renaming-a-column)
- [Renaming a table](#renaming-a-table)
- [Adding a check constraint](#adding-a-check-constraint)
- [Setting NOT NULL on an existing column](#setting-not-null-on-an-existing-column)
- [Adding a JSON column](#adding-a-json-column)

---
 
## Adding an index

Creating an index will block both reads and writes.

### BAD ❌

```elixir
def change do
  create index("posts", [:slug])

  # This obtains a ShareLock on "posts" which will block writes to the table
end
```

### GOOD ✅

Instead, have Postgres create the index concurrently which does not block reads. You will need to disable the migration transactions to use `CONCURRENTLY`.

```elixir
@disable_ddl_transaction true
@disable_migration_lock true

def change do
  create index("posts", [:slug], concurrently: true)
end
```

The migration may still take a while to run, but reads and updates to rows will continue to work. For example, for 100,000,000 rows it took 165 seconds to add run the migration, but SELECTS and UPDATES could occur while it was running.

**Do not have other changes in the same migration**; only create the index concurrently and separate other changes to later migrations.

---

## Adding a reference or foreign key

Adding a foreign key blocks writes on both tables.

### BAD ❌

```elixir
def change do
  alter table("posts") do
    add :group_id, references("groups")
  end
end
```


### GOOD ✅

In the first migration

```elixir
def change do
  alter table("posts") do
    add :group_id, references("groups", validate: false)
  end
end
```

In the second migration

```elixir
def change do
  execute "ALTER TABLE posts VALIDATE CONSTRAINT group_id_fkey", ""
end
```
 
 These migrations can be in the same deployment, but make sure they are separate migrations.
 
---
 
## Adding a column with a default value

Adding a column with a default value to an existing table may cause the table to be rewritten. During this time, reads and writes are blocked in Postgres, and writes are blocked in MySQL and MariaDB.

### BAD ❌

Note: This becomes safe in:

- Postgres 11+
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

### GOOD ✅

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
  alter table("comments") do
    modify :approved, :boolean, default: false
    # This took 0.28 milliseconds for 100 million rows with no fkeys,
  end
end
```

Schema change to read the new column:

```diff
schema "comments" do
+ field :approved, :boolean, default: false
end
```

---

## Changing the type of a column

Changing the type of a column may cause the table to be rewritten. During this time, reads and writes are blocked in Postgres, and writes are blocked in MySQL and MariaDB.

### BAD ❌

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
    modify :my_column, :boolean, :text
  end
end
```

### GOOD ✅

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

### BAD ❌

```elixir
# Without a code change to the Ecto Schema

def change
  alter table("posts") do
    remove :no_longer_needed_column
  end
end
```


### GOOD ✅

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
def change
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

### BAD ❌

```elixir
# In your schema

schema "posts" do
  field :summary, :text
end
```

# In your migration

```elixir
def change do
  rename table("posts"), :title, to: :summary
end
```

The time between your migration running and your application getting the new code may encounter trouble.

### GOOD ✅

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

### BAD ❌

```elixir
def change do
  rename table("posts"), to: table("articles")
end
```

### GOOD ✅

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

### BAD ❌

```elixir
def change do
  create constraint("products", :price_must_be_positive, check: "price > 0")
  # Creating the constraint with validate: true (the default when unspecified)
  # will perform a full table scan and acquires a lock preventing updates
end
```

### GOOD ✅

There are two operations occurring:

1. Creating a new constraint for new or updating records
1. Validating the new constraint for existing records

If these commands are happening at the same time, it obtains a lock on the table as it validates the entire table and fully scans the table. To avoid this full table scan, we can separate the operations.

In one migration:

```elixir
def change do
  create constraint("products", :price_must_be_positive, check: "price > 0"), validate: false
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

### BAD ❌

```elixir
def change do
  alter table("products") do
    modify :active, :boolean, null: false
  end
end
```

### GOOD ✅

Add a check constraint without validating it, backfill data to satiate the constraint and then validate it. This will be functionally equivalent.

In the first migration:

```elixir
# Deployment 1
def change do
  create constraint("products", :active_not_null, check: "active IS NOT NULL"), validate: false
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

```elixir
# **Postgres 12+ only**

def change do
  execute "ALTER TABLE products VALIDATE CONSTRAINT active_not_null", ""

  alter table("products") do
    modify :active, :boolean, null: false
  end

  drop constraint("products", :active_not_null)
end
```

If your constraint fails, then you should consider backfilling data first to cover the gaps in your desired data integrity, then revisit validating the constraint.

---
 
## Adding a JSON column

In Postgres, there is no equality operator for the json column type, which can cause errors for existing SELECT DISTINCT queries in your application.

### BAD ❌

```elixir
def change do
  alter table("posts") do
    add :extra_data, :json
  end
end
```

### GOOD ✅

Use jsonb instead. Some say it’s like “json” but “better.”

```elixir
def change do
  alter table("posts") do
    add :extra_data, :jsonb
  end
end
```

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
