# Drizzle schema declaration

Distilled from `drizzle-orm` 0.45.x and the pre-v1 docs snapshot
(`vendor/drizzle-orm-docs` @ `11695b7`). Column-type detail:
`orm.drizzle.team/docs/column-types/pg` (and `/mysql`, `/sqlite`).

Everything here is **plain TypeScript**. A schema file exports `const`s; Drizzle
ORM imports them for types, Drizzle Kit imports them to diff migrations.
**Anything Kit must see has to be `export`ed** — a non-exported table just
vanishes from migrations with no error.

## Table builders

| Dialect | Builder | Column import |
| --- | --- | --- |
| PostgreSQL | `pgTable` | `drizzle-orm/pg-core` |
| MySQL | `mysqlTable` | `drizzle-orm/mysql-core` |
| SQLite | `sqliteTable` | `drizzle-orm/sqlite-core` |
| SingleStore | `singlestoreTable` | `drizzle-orm/singlestore-core` |
| Gel | `gelTable` | `drizzle-orm/gel-core` |

```ts
import { pgTable, integer, varchar, timestamp, boolean, index, uniqueIndex } from "drizzle-orm/pg-core";

export const users = pgTable(
  "users",
  {
    id: integer().primaryKey().generatedAlwaysAsIdentity(),
    email: varchar({ length: 256 }).notNull(),
    firstName: varchar("first_name", { length: 256 }),   // explicit DB name
    isActive: boolean("is_active").notNull().default(true),
    createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  },
  (t) => [
    uniqueIndex("users_email_uq").on(t.email),
    index("users_name_idx").on(t.firstName),
  ],
);
```

- **Column name resolution**: explicit string arg → `casing` option on
  `drizzle()` → verbatim TS key. `casing` (`"snake_case"` | `"camelCase"`) only
  rewrites keys with no explicit name. It changes emitted DDL and query SQL —
  choose before the first migration, never change later.
- The **third argument is a callback returning an array** of table-level
  extras: `index()`, `uniqueIndex()`, `primaryKey()`, `foreignKey()`,
  `check()`, `pgPolicy()`. (An object return still works but the array form is
  what current docs use.)
- Reuse column groups by spreading a plain object:
  ```ts
  const timestamps = {
    createdAt: timestamp("created_at").defaultNow().notNull(),
    updatedAt: timestamp("updated_at").$onUpdate(() => new Date()),
    deletedAt: timestamp("deleted_at"),
  };
  export const posts = pgTable("posts", { id: integer().primaryKey(), ...timestamps });
  ```
- `pgTableCreator((name) => \`app_${name}\`)` (also `mysqlTableCreator`,
  `sqliteTableCreator`) prefixes every table — pair with
  `tablesFilter: ["app_*"]` in the config to share one database between
  projects.

## Column types & the `mode` traps

### PostgreSQL

| Drizzle | SQL | Notes / inferred TS |
| --- | --- | --- |
| `integer()` `smallint()` `bigint({ mode })` | `integer` `smallint` `bigint` | `bigint()` **requires** a `mode`: `{ mode: "number" }` → JS `number` (only safe < 2^53), `{ mode: "bigint" }` → JS `BigInt`. |
| `serial()` `smallserial()` `bigserial({ mode })` | auto-increment | Legacy; prefer identity columns (below). |
| `boolean()` | `boolean` | |
| `text()` `varchar({ length })` `char({ length })` | | `{ enum: ["a","b"] }` narrows the **type** only (no runtime check). |
| `numeric({ precision, scale, mode })` / `decimal(...)` | `numeric` | **Returns `string` by default.** `mode: "number"` or `"bigint"` to map. |
| `real()` `doublePrecision()` | float4 / float8 | `number` |
| `json()` `jsonb()` | | `.$type<T>()` for the object shape (compile-time only). `jsonb().$type<string[]>().default({})` won't compile — good. |
| `uuid()` | `uuid` | `.defaultRandom()` → `gen_random_uuid()`. |
| `timestamp({ withTimezone, precision, mode })` | `timestamp` / `timestamptz` | **`mode: "date"` (default)** ⇄ JS `Date`; **`mode: "string"`** no mapping. Without `withTimezone` the DB drops offset info. With it, stored UTC, returned in the server's `TimeZone`. |
| `date({ mode })` | `date` | `"date"` → `Date`, `"string"` → `"YYYY-MM-DD"`. |
| `time({ withTimezone, precision })` `interval({ fields, precision })` | | strings |
| `point({ mode })` `line({ mode })` | geometric | `"tuple"` → `[x,y]`, `"xy"` → `{x,y}`. |
| `inet()` `cidr()` `macaddr()` `macaddr8()` `bytea()` | | strings / Buffer |
| enums: `pgEnum("mood", ["sad","happy"])` then `moodEnum()` | `CREATE TYPE ... AS ENUM` | Real DB enum. Renaming/reordering values is a migration headache — Kit generates `ALTER TYPE`. |

**Arrays (pg)**: `.array()` on any column → `type[]`; chain for multi-dim:
`integer().array()` → `integer[]`, `integer().array().array()` → `integer[][]`.
Query with `arrayContains` / `arrayContained` / `arrayOverlaps`. (v1 replaces the
chain with a string arg: `.array("[][]")`.)

**Identity columns** (preferred over `serial`):
```ts
id: integer().primaryKey().generatedAlwaysAsIdentity(),          // DB always generates
id: integer().primaryKey().generatedByDefaultAsIdentity(),       // manual override allowed
id: integer().generatedAlwaysAsIdentity({ startWith: 1000, name: "my_seq" }),
```

### MySQL

`int()` `tinyint()` `bigint({ mode, unsigned })` `serial()` (→ `bigint unsigned
auto_increment`), `boolean()` (stored as `tinyint(1)`), `varchar({ length })`
(**length required**), `text()`/`mediumtext()`/`longtext()`, `decimal()`
(string), `float()`/`double()`, `json()` (`.$type<T>()`), `datetime({ mode })`
/ `timestamp({ mode })` / `date({ mode })` (`"date"` ⇄ `Date`, `"string"` raw),
`mysqlEnum("col", ["a","b"])` (inline enum), `binary()`/`varbinary()`.
Auto-increment PK: `.primaryKey().autoincrement()`.

### SQLite

Storage classes are `NULL INTEGER REAL TEXT BLOB` — everything is a `mode` on
one of five builders:

```ts
integer()                              // number
integer({ mode: "boolean" })           // 0/1  ⇄  boolean   ← SQLite has no bool
integer({ mode: "timestamp" })         // unix seconds  ⇄  Date
integer({ mode: "timestamp_ms" })      // unix millis  ⇄  Date
integer({ mode: "number" }).primaryKey({ autoIncrement: true })
text()                                 // string
text({ enum: ["a", "b"] })             // type-narrowed
text({ mode: "json" }).$type<T>()      // JSON — prefer this over blob for JSON
real()
blob({ mode: "buffer" | "bigint" | "json" })
numeric({ mode: "number" | "bigint" })
```

**SQLite `TEXT` rejects invalid UTF-8** — store raw bytes as `blob`.

### Custom types

`customType<{ data; driverData?; config? }>({ dataType(config), toDriver?, fromDriver? })`
lets you define any DB type Drizzle lacks (e.g. `tsvector`, `citext`, PostGIS
`geometry`). `dataType()` returns the SQL type string used in migrations;
`toDriver`/`fromDriver` map values at runtime.

```ts
const tsvector = customType<{ data: string }>({ dataType: () => "tsvector" });
```

## Runtime defaults & hooks

| API | When it runs | Affects migrations? |
| --- | --- | --- |
| `.default(value)` / `.default(sql\`now()\`)` | DB-side `DEFAULT` clause | **Yes** — emitted in DDL |
| `.defaultNow()`, `.defaultRandom()` | DB-side (`now()`, `gen_random_uuid()`) | Yes |
| `.$defaultFn(() => ...)` / `.$default(...)` | Drizzle, at **insert** time in JS | **No** — runtime only |
| `.$onUpdateFn(() => ...)` / `.$onUpdate(...)` | Drizzle, at **update** time (and insert if no other default) | No |

Use `.$defaultFn` for app-generated IDs (`cuid2`, `nanoid`, `uuidv7`):
```ts
import { createId } from "@paralleldrive/cuid2";
id: text().primaryKey().$defaultFn(() => createId()),
```

## Constraints

- **Not null / unique**: `.notNull()`, `.unique()`, `.unique("name", { nulls: "not distinct" })`.
- **Composite unique / PK**: table-level `unique().on(t.a, t.b)` /
  `primaryKey({ columns: [t.a, t.b] })`.
- **Check**: `check("age_ck", sql\`${t.age} >= 0\`)`.
- **Foreign keys**:
  ```ts
  authorId: integer("author_id").references(() => users.id, { onDelete: "cascade", onUpdate: "no action" }),
  ```
  Actions: `"cascade" | "restrict" | "no action" | "set null" | "set default"`.
  Self-reference → `references((): AnyPgColumn => users.id)` **or** the
  standalone `foreignKey({ columns, foreignColumns, name }).onDelete("cascade")`.
  Multi-column FKs require the standalone `foreignKey`.

## Indexes

```ts
(t) => [
  index("name_idx").on(t.name),
  uniqueIndex("email_idx").on(t.email),
  index("multi").on(t.a.asc(), t.b.desc().nullsLast()),
  index("expr").on(sql`lower(${t.email})`),                 // ← must name it manually
  index("gin_idx").using("gin", t.searchVector),
  index("partial").on(t.status).where(sql`${t.status} = 'active'`),
  index("fill").on(t.id).with({ fillfactor: "70" }),
]
```

- **An index on an expression MUST have an explicit name** — `index().on(sql\`...\`)`
  throws; `index("my_name").on(sql\`...\`)` is fine.
- **`drizzle-kit push` ignores changes** to an existing index's `.on()`
  expressions, `.using()`, `.where()`, or `.op()` operator classes. To change
  those with `push`: comment the index out → push → uncomment with the change →
  push again. `drizzle-kit generate` detects them normally.

## Generated columns

```ts
searchVector: tsvector("search").generatedAlwaysAs(
  (): SQL => sql`to_tsvector('english', ${test.content})`
),
// MySQL/SQLite also take { mode: "stored" | "virtual" }
```

- Postgres: `STORED` only, no default, can't reference other generated
  columns, can't be in PK/FK/unique.
- **`push` cannot alter a generated expression** (pg/mysql) or a `stored`
  generated column at all (sqlite) — drop & recreate the column. `generate`
  handles it.

## Views

```ts
export const activeUsers = pgView("active_users").as((qb) =>
  qb.select().from(users).where(eq(users.isActive, true)));

// raw SQL view — you must declare the column shape
export const v = pgView("v", { id: integer(), name: text() }).as(sql`select ...`);

// view that already exists — Kit will not emit CREATE VIEW
export const external = pgView("external", { id: integer() }).existing();

// materialized (pg only)
export const mv = pgMaterializedView("mv").as((qb) => qb.select().from(users));
await db.refreshMaterializedView(mv).concurrently();
```
RLS on a view: `.with({ securityInvoker: true })`.

## Postgres schemas, sequences, RLS

- **Schema** (namespace): `const s = pgSchema("billing"); export const inv = s.table("invoices", {...})`.
  Also `s.enum(...)`, `s.sequence(...)`. Queries get `"billing"."invoices"`.
  (MySQL `mysqlSchema` = database and is **not** tracked by Kit.)
- **Sequence**: `pgSequence("name", { startWith, increment, minValue, maxValue, cycle, cache })`.
- **RLS**: `pgTable(..., (t) => [pgPolicy("p", { as, to, for, using, withCheck })])`
  (adding a policy auto-enables RLS), `pgRole("admin", {...})`,
  `.enableRLS()` for policy-less default-deny. `drizzle-orm/neon` exposes
  `crudPolicy()` + `authenticatedRole`; `drizzle-orm/supabase` exposes
  `authenticatedRole` / `serviceRole` / `authUsers` / `authUid`. Tell Kit which
  roles to leave alone with `entities.roles` in the config.

## Type helpers (in app code)

```ts
type User    = typeof users.$inferSelect;    // row shape
type NewUser = typeof users.$inferInsert;    // insert shape (optionals, defaults)
import { getTableColumns } from "drizzle-orm";
const { password, ...safe } = getTableColumns(users);
await db.select({ ...safe }).from(users);     // select all but password
import { getTableConfig } from "drizzle-orm/pg-core";  // columns, indexes, fks, ...
```
(`InferSelectModel<typeof users>` / `InferInsertModel` are equivalents.
`getColumns` is the **v1** rename of `getTableColumns` — don't use it on 0.45.)
