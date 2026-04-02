# Flowgrate JSON Manifest Specification

This document defines the JSON manifest format — the contract between language SDKs and the Flowgrate Go CLI.

If you're building an SDK for a new language, your SDK must output manifests that conform to this spec.

---

## How it works

```
Your SDK (any language)
    → writes JSON manifests to stdout (one per migration)
    → flowgrate CLI reads them via stdin or by calling your project directly
    → Go CLI compiles operations to SQL and executes them
```

The CLI reads JSON from stdin if piped, otherwise invokes the SDK via the `run` command in config:

```bash
# piped — works with any SDK, no config needed
dotnet run --project ./Migrations | flowgrate up
python ./migrations/runner.py | flowgrate up

# direct — CLI calls the SDK itself using migrations.run from flowgrate.yml
flowgrate up
```

### Config (`flowgrate.yml`)

Generate with `flowgrate init`, then uncomment the `run` line that matches your SDK:

```yaml
database:
  url: postgres://user:pass@localhost/mydb

migrations:
  project: ./Migrations   # for 'flowgrate make' (where to generate files)
  sdk: csharp             # for 'flowgrate make' (which template)

  # Uncomment the command that invokes your SDK:
  # run: dotnet run --project ./Migrations
  # run: python ./Migrations/runner.py
  # run: php artisan flowgrate:export
  # run: poetry run python ./Migrations/runner.py
  # run: bundle exec rake flowgrate:dump
  # run: docker compose exec sdk dotnet run --project /migrations
```

If `run` is not set, the CLI falls back to built-in defaults:
- `sdk: csharp` → `dotnet run --project {project}`
- `sdk: python` → `python {project}`

---

## Top-level manifest

Each migration outputs **one JSON object** to stdout. Multiple migrations = multiple JSON objects (newline-delimited or whitespace-separated).

```json
{
  "migration": "20260402_163107_CreateUsersTable",
  "up": [ ...operations ],
  "down": [ ...operations ]
}
```

| Field       | Type              | Description                                      |
|-------------|-------------------|--------------------------------------------------|
| `migration` | `string`          | Filename without extension (timestamp + name)    |
| `up`        | `Operation[]`     | Operations to apply the migration                |
| `down`      | `Operation[]`     | Operations to roll back the migration            |

### Migration name format

```
YYYYMMDD_HHMMSS_MigrationName
```

Example: `20260402_163107_CreateUsersTable`

---

## Operations

Each operation has an `action` field that determines its type.

### `create_table`

```json
{
  "action": "create_table",
  "table": "users",
  "columns": [ ...Column ],
  "indexes": [ ...Index ],
  "foreign_keys": [ ...ForeignKey ]
}
```

### `alter_table`

```json
{
  "action": "alter_table",
  "table": "users",
  "columns": [ ...Column ]
}
```

Each column in an `alter_table` operation must have a `column_action` field (`"add"`, `"change"`, or `"drop"`).

### `drop_table`

```json
{
  "action": "drop_table",
  "table": "users",
  "if_exists": true
}
```

| Field       | Type      | Default  | Description                         |
|-------------|-----------|----------|-------------------------------------|
| `if_exists` | `boolean` | `false`  | Emit `DROP TABLE IF EXISTS`         |

---

## Column definition

```json
{
  "name": "email",
  "type": "string",
  "column_action": "add",
  "length": 100,
  "precision": 10,
  "scale": 2,
  "nullable": false,
  "primary": false,
  "auto_increment": false,
  "default": "active",
  "default_expression": "gen_random_uuid()",
  "comment": "User email address"
}
```

| Field                | Type      | Default   | Description                                                     |
|----------------------|-----------|-----------|-----------------------------------------------------------------|
| `name`               | `string`  | required  | Column name                                                     |
| `type`               | `string`  | required  | Canonical type (see table below)                                |
| `column_action`      | `string`  | `"add"`   | `add` / `change` / `drop` — only used in `alter_table`          |
| `length`             | `int`     | —         | For `string` type (default 255)                                 |
| `precision`          | `int`     | —         | For `decimal` type                                              |
| `scale`              | `int`     | —         | For `decimal` type                                              |
| `nullable`           | `bool`    | `false`   | Allow NULL                                                      |
| `primary`            | `bool`    | `false`   | PRIMARY KEY                                                     |
| `auto_increment`     | `bool`    | `false`   | Auto-incrementing PK (SERIAL / BIGSERIAL)                       |
| `default`            | `any`     | —         | Literal default value (string, number, boolean)                 |
| `default_expression` | `string`  | —         | Raw SQL expression default — emitted verbatim (e.g. `NOW()`)   |
| `comment`            | `string`  | —         | Column comment                                                  |

> **`default` vs `default_expression`**: use `default` for literal values (`"active"`, `true`, `0`). Use `default_expression` for SQL function calls (`gen_random_uuid()`, `NOW()`). Never set both.

---

## Column types

These are canonical type names. Each database adapter maps them to its native type.

| Manifest type   | PostgreSQL         | Description                              |
|-----------------|--------------------|------------------------------------------|
| `string`        | `VARCHAR(n)`       | Variable-length string, `length` sets n  |
| `text`          | `TEXT`             | Unlimited text                           |
| `small_integer` | `SMALLINT`         | 2-byte integer                           |
| `integer`       | `INTEGER`          | 4-byte integer                           |
| `big_integer`   | `BIGINT`/`BIGSERIAL` | 8-byte integer; BIGSERIAL if `auto_increment` |
| `decimal`       | `NUMERIC(p, s)`    | Exact numeric, requires `precision`+`scale` |
| `float`         | `REAL`             | 4-byte floating point                    |
| `double`        | `DOUBLE PRECISION` | 8-byte floating point                    |
| `boolean`       | `BOOLEAN`          |                                          |
| `date`          | `DATE`             |                                          |
| `time`          | `TIME`             |                                          |
| `timestamp`     | `TIMESTAMP`        |                                          |
| `uuid`          | `UUID`             |                                          |
| `json`          | `JSON`             |                                          |
| `jsonb`         | `JSONB`            | PostgreSQL only — binary JSON            |
| `binary`        | `BYTEA`            | Raw binary data                          |

---

## Index definition

```json
{
  "columns": ["email", "tenant_id"],
  "unique": true,
  "name": "uq_users_email_tenant"
}
```

| Field     | Type       | Default | Description                                              |
|-----------|------------|---------|----------------------------------------------------------|
| `columns` | `string[]` | required | Column names in the index                               |
| `unique`  | `bool`     | `false` | UNIQUE constraint                                        |
| `name`    | `string`   | —       | Custom index name (auto-generated from table+columns if omitted) |

---

## Foreign key definition

```json
{
  "column": "role_id",
  "references_table": "roles",
  "references_column": "id",
  "on_update": "cascade",
  "on_delete": "set null"
}
```

| Field               | Type     | Default | Description                          |
|---------------------|----------|---------|--------------------------------------|
| `column`            | `string` | required | The FK column in this table         |
| `references_table`  | `string` | required | Referenced table                    |
| `references_column` | `string` | `"id"`  | Referenced column                    |
| `on_update`         | `string` | —       | `cascade` / `set null` / `restrict`  |
| `on_delete`         | `string` | —       | `cascade` / `set null` / `restrict`  |

---

## Complete example

```json
{
  "migration": "20260402_163107_CreateUsersTable",
  "up": [
    {
      "action": "create_table",
      "table": "users",
      "columns": [
        { "name": "id",         "type": "big_integer", "primary": true, "auto_increment": true },
        { "name": "public_id",  "type": "uuid",        "default_expression": "gen_random_uuid()" },
        { "name": "name",       "type": "string",      "length": 255 },
        { "name": "email",      "type": "string",      "length": 100 },
        { "name": "score",      "type": "decimal",     "precision": 10, "scale": 2, "nullable": true },
        { "name": "active",     "type": "boolean",     "default": true },
        { "name": "deleted_at", "type": "timestamp",   "nullable": true },
        { "name": "created_at", "type": "timestamp",   "default_expression": "NOW()" },
        { "name": "updated_at", "type": "timestamp",   "default_expression": "NOW()" }
      ],
      "indexes": [
        { "columns": ["email"], "unique": true, "name": "uq_users_email" }
      ],
      "foreign_keys": [
        { "column": "role_id", "references_table": "roles", "references_column": "id", "on_delete": "cascade" }
      ]
    }
  ],
  "down": [
    {
      "action": "drop_table",
      "table": "users",
      "if_exists": true
    }
  ]
}
```

---

## Implementing an SDK

To implement Flowgrate for a new language:

1. **Define a `Migration` base class/interface** with `up()` and `down()` methods that call your fluent API.
2. **Build a fluent Blueprint API** that records operations (create table, alter table, drop table) and column definitions using the canonical types above.
3. **Discover all migrations** via reflection, file scanning, or explicit registration — sorted by filename (timestamp prefix ensures order).
4. **Serialize to JSON** following this spec and write each manifest to **stdout** as a JSON object.

The CLI reads your output via stdin:
```bash
your-sdk-runner | flowgrate up
```

### SDK requirements checklist

- [ ] Invocable via a single shell command (configurable in `migrations.run`)
- [ ] Outputs one JSON object per migration to stdout
- [ ] Migrations are ordered by the timestamp in their filename
- [ ] All canonical column types are supported
- [ ] `default` and `default_expression` are kept separate
- [ ] `column_action` is set correctly for `alter_table` operations
- [ ] `if_exists` is set on `drop_table` when using DropIfExists equivalent

### Reference implementations

- **C#**: [github.com/flowgrate/dotnet](https://github.com/flowgrate/dotnet)
