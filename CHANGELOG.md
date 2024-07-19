# 2024-08-19

- Correction to "Adding an Index" section about what locks are obtained.
  Thanks @seanpascoe for the report!

# 2023-10-10

- Update the instructions for setting a columns' non-null constraint.
  Avoid using `modify/3` since that will include adjusting the type
  unnecessarily, which will cause Postgres to rewrite the table.
  Thanks @dhedlund for the report!

# 2023-05-10

- Add note about setting `migration_lock: false` on CheckRepoStatus when
  the Repo is using advisory locks for migrations. Thanks @cgrothaus for the
  report!

# 2022-10-06

- Ecto 3.9.0 supports advisory locks for migrations, which is a great way to
  manage migrations when you need to create indexes concurrently. Add a note about
  using that migration locking strategy.
