# Drizzle querying

`drizzle-orm` 0.45.x. Two APIs on the same `db`: **SQL-like** (mirrors SQL,
you map relations) and **relational** (`db.query.*`, nested trees, one
statement). Docs: `/docs/select`, `/docs/rqb`, `/docs/sql`, `/docs/operators`.

## SQL-like builder

### Select

```ts
await db.select().from(users);                                  // all columns, typed
await db.select({ id: users.id, n: users.name }).from(users);   // partial
await db.select({
  ...getTableColumns(users),
  postCount: sql<number>`count(${posts.id})`.mapWith(Number),
}).from(users).leftJoin(posts, eq(posts.authorId, users.id)).groupBy(users.id);

await db.selectDistinct().from(users);
await db.selectDistinctOn([users.city], { city: users.city }).from(users);   // pg only
```

- Drizzle **always lists columns explicitly** (never `select *`) to guarantee
  field order.
- **Conditional selection**: spread — `{ id: users.id, ...(withName ? { name: users.name } : {}) }`.
- `sql<T>` is a **type assertion, not a cast**. If the driver returns a string
  and you claimed `sql<number>`, the runtime value is still a string. Use
  `.mapWith(Number)` / `.mapWith(users.someColumn)` for an actual runtime map.

### Filters & operators

Import from `drizzle-orm`: `eq ne gt gte lt lte` · `inArray notInArray` (accept
an array **or** a subquery) · `between notBetween` · `like ilike notLike
notIlike` (`ilike` is pg-only) · `isNull isNotNull` · `exists notExists` ·
`and or not` · `arrayContains arrayContained arrayOverlaps` (pg arrays).

```ts
await db.select().from(users).where(and(eq(users.active, true), gt(users.age, 18)));
```

- **Each clause method runs once.** `.where(a).where(b)` is a type error — combine
  in `and(...)`.
- **`.where(undefined)` / `and()` with no args ⇒ no filter.** Optional-filter idiom:
  ```ts
  const filters: SQL[] = [];
  if (name) filters.push(ilike(users.name, name));
  if (minAge) filters.push(gte(users.age, minAge));
  await db.select().from(users).where(and(...filters));   // empty ⇒ selects all
  ```
- Anything Drizzle lacks: drop to `sql\`...\`` inside `.where()` /
  `.orderBy()` / `.having()` / `.groupBy()`.

### Insert / update / delete

```ts
await db.insert(users).values({ email: "a@b.c" });
await db.insert(users).values([{ email: "a@b.c" }, { email: "d@e.f" }]);
const [row] = await db.insert(users).values({ email: "a@b.c" }).returning();          // pg / sqlite
const ids  = await db.insert(users).values([...]).$returningId();                     // mysql / singlestore

await db.update(users).set({ name: "Dan", updatedAt: sql`now()` }).where(eq(users.id, 1));
await db.delete(users).where(eq(users.id, 1));
```

- **`RETURNING` is pg + sqlite only.** MySQL/SingleStore: `.$returningId()`
  gives back auto-increment / `$defaultFn` PKs as `{ id }[]` (or `{}[]` if no PK).
- **`.set({ x: undefined })` is skipped** — no `SET x = NULL`. Pass `null`.
- **No `.where()` ⇒ every row.** `db.delete(users)` truncates.
- `.limit()` / `.orderBy()` on `update`/`delete`: MySQL/SQLite/SingleStore only,
  not pg.

### Upsert

```ts
// pg / sqlite
await db.insert(users).values({ id: 1, name: "John" }).onConflictDoNothing();
await db.insert(users).values({ id: 1, name: "John" })
  .onConflictDoUpdate({ target: users.id, set: { name: sql`excluded.name` } });
await db.insert(users).values({ ... })
  .onConflictDoUpdate({ target: [users.a, users.b], targetWhere: sql`...`, set: {...}, setWhere: sql`...` });

// mysql / singlestore — target is implicit (PK + unique indexes)
await db.insert(users).values({ id: 1, name: "John" })
  .onDuplicateKeyUpdate({ set: { name: "John" } });
// mysql "do nothing": .onDuplicateKeyUpdate({ set: { id: sql`id` } })
```

### Joins

`leftJoin rightJoin innerJoin fullJoin crossJoin` (+ `*JoinLateral`).

```ts
const rows = await db.select().from(users).leftJoin(posts, eq(posts.authorId, users.id));
// rows: { users: {...}, posts: {...} | null }[]   ← grouped by table key
```

- **`leftJoin` ⇒ that table is `| null`** in the result type. `rightJoin` nulls
  the left table; `fullJoin` nulls both.
- **Partial select + join**: nest the shape so only the object goes null, not
  every field:
  ```ts
  db.select({ id: users.id, post: { id: posts.id, title: posts.title } })
    .from(users).leftJoin(posts, ...);
  // { id: number, post: { id, title } | null }[]
  ```
- For `sql` off a left-joined table, annotate `sql<string | null>` yourself.
- Self-join: `const parent = alias(users, "parent"); db.select().from(users).leftJoin(parent, eq(parent.id, users.parentId))`.
- Drizzle does **not** aggregate join rows into a tree — you `reduce()` yourself,
  or use the relational builder.
- MySQL index hints: `.from(users, { useIndex: idx })` / `ignoreIndex` / `forceIndex`.

### `sql` template

```ts
import { sql } from "drizzle-orm";
await db.execute(sql`select * from ${users} where ${users.id} = ${id}`);   // id → $1, parameterised
```

- **Interpolated tables/columns are escaped; interpolated values are
  parameterised.** This is injection-safe.
- **`sql.raw(str)` is NOT parameterised or escaped** — and **`sql.identifier()`
  / `sql.as()` were injection-vulnerable before `drizzle-orm@0.45.2`**. Never
  pass user input to `sql.raw` / `sql.identifier`; be on ≥ 0.45.2.
- `sql<T>\`...\`` type hint · `.as("alias")` (required when the field is
  referenced elsewhere, e.g. in a CTE) · `.mapWith(Number | usersTable.col | {mapFromDriverValue})`
  runtime map · `sql.placeholder("name")` for prepared statements ·
  `sql.join(chunks, sep)` / `sql.empty()` / `.append()` for building dynamically.
- Get SQL without running: `query.toSQL()` → `{ sql, params }`; standalone
  `new QueryBuilder()` from `drizzle-orm/pg-core` builds queries with no `db`.

### Aggregates & count

Helpers from `drizzle-orm`: `count() countDistinct() avg() avgDistinct() sum()
sumDistinct() max() min()`.

```ts
await db.select({ v: count() }).from(users);              // count(*)
await db.select({ v: count(users.id) }).from(users);
```

- **`count()` returns `bigint` in pg / `string` in mysql** (returned as string
  by the driver). The helper `.mapWith(Number)`s for you; a hand-written
  `sql\`count(*)\`` does not — add `.mapWith(Number)` or `cast(... as int)`.
- **`db.$count(table, where?)`** is the shortcut for a scalar count and works
  as a subquery / in `extras`.

### CTEs, subqueries, set operations

```ts
const sq = db.$with("sq").as(db.select().from(users).where(eq(users.id, 42)));
await db.with(sq).select().from(sq);                 // WITH ... SELECT
// insert/update/delete are also valid inside $with (RETURNING feeds the CTE)

const sub = db.select().from(users).where(eq(users.active, true)).as("sub");
await db.select().from(sub).leftJoin(...);           // subquery as a table

import { union, unionAll, intersect, except } from "drizzle-orm/pg-core";  // dialect-specific import
await union(db.select({ n: users.name }).from(users),
            db.select({ n: customers.name }).from(customers)).limit(10);
// or chained: db.select({...}).from(a).union(db.select({...}).from(b))
```

### Dynamic query building

Builders are single-call by design. To pass a query between functions, call
`.$dynamic()`:

```ts
function withPagination<T extends PgSelect>(qb: T, page = 1, size = 10) {
  return qb.limit(size).offset((page - 1) * size);
}
const q = db.select().from(users).where(eq(users.id, 1)).$dynamic();
withPagination(q, 2);
```
Generic param types: `PgSelect PgInsert PgUpdate PgDelete` (and `MySql*`,
`SQLite*`), plus `PgSelectQueryBuilder` for standalone `QueryBuilder`.

### Transactions

```ts
const newBalance = await db.transaction(async (tx) => {
  await tx.update(accounts).set({ balance: sql`${accounts.balance} - 100` }).where(eq(accounts.id, from));
  await tx.update(accounts).set({ balance: sql`${accounts.balance} + 100` }).where(eq(accounts.id, to));
  const [a] = await tx.select().from(accounts).where(eq(accounts.id, from));
  if (a.balance < 0) tx.rollback();          // throws → rolls back
  return a.balance;                          // return value propagates out
});
```

- **Use `tx` for every statement inside** — using the outer `db` escapes the
  transaction.
- Nested `tx.transaction(...)` ⇒ savepoints.
- Config: pg `{ isolationLevel, accessMode, deferrable }`; sqlite
  `{ behavior: "deferred" | "immediate" | "exclusive" }`; mysql
  `{ isolationLevel, accessMode, withConsistentSnapshot }`.
- **`neon-http` has no `transaction()` at all** (throws) — use `db.batch([...])`
  for atomic multi-statement. `neon-serverless` (WebSocket) and
  `drizzle-orm/planetscale` do support `transaction()`.
- The `cache` option is bypassed inside transactions.

### Prepared statements

```ts
const p = db.select().from(users).where(eq(users.id, sql.placeholder("id"))).prepare("by_id");  // name required in pg
await p.execute({ id: 10 });
```
Biggest win on serverless (hoist the prepared statement to module scope so the
driver reuses the compiled plan across invocations). Placeholders work in
`where`, `limit`, `offset`.

### Batch

`db.batch([...])` — one network round-trip, implicit transaction — for
**libSQL, Neon, D1** only. Array of any builders (`select`/`insert`/`update`/
`delete`/`db.query.*.findMany`/`db.get`/…); result is a tuple typed per
statement.

## Relational query builder (`db.query.*`)

### Setup

1. Define relations in a schema file (or sibling) with `relations()`:
   ```ts
   import { relations } from "drizzle-orm";
   export const usersRelations = relations(users, ({ one, many }) => ({
     posts: many(posts),
     profile: one(profiles),                              // FK lives on profiles → nullable
   }));
   export const postsRelations = relations(posts, ({ one }) => ({
     author: one(users, { fields: [posts.authorId], references: [users.id] }),
   }));
   ```
2. Pass **the whole schema module** to `drizzle`:
   `drizzle(url, { schema })` (multi-file: `{ schema: { ...s1, ...s2 } }`).

`relations()` is **application-level only** — no FK, not in migrations. Define
FKs separately if you want the DB constraint.

### Querying

```ts
await db.query.users.findMany({
  where: (u, { eq, and }) => and(eq(u.active, true)),
  columns: { password: false },                    // false-exclude OR true-include, not both
  with: {
    posts: {
      columns: { authorId: false },
      where: (p, { lt }) => lt(p.createdAt, new Date()),
      limit: 5,
      orderBy: (p, { desc }) => [desc(p.id)],
      with: { comments: true },                     // nest arbitrarily deep
    },
  },
  extras: {
    fullName: (u, { sql }) => sql<string>`${u.firstName} || ' ' || ${u.lastName}`.as("full_name"),
  },
  limit: 10,
  offset: 20,                                       // top level only on 0.45
});
await db.query.users.findFirst({ with: { posts: true } });   // adds LIMIT 1
```

- **One SQL statement**, always — nested `with` become lateral-joined
  `json_agg` subqueries.
- 0.45 callback style is `(table, operators)`. `extras` fields **must** be
  `.as("...")`d. Aggregations aren't allowed in `extras` — use a core query or
  `db.$count`.
- `columns: {}` + `with` returns only the relations.

### Relation shapes

- **one-to-one**: `one(target, { fields, references })` on the side holding the
  FK; `one(target)` (no config) on the other side ⇒ nullable.
- **one-to-many**: `many(child)` on the parent, `one(parent, { fields,
  references })` on the child.
- **many-to-many**: you must declare the junction table and relate through it
  from **both** sides:
  ```ts
  export const usersToGroups = pgTable("users_to_groups", {
    userId: integer("user_id").notNull().references(() => users.id),
    groupId: integer("group_id").notNull().references(() => groups.id),
  }, (t) => [primaryKey({ columns: [t.userId, t.groupId] })]);

  export const usersRelations = relations(users, ({ many }) => ({ usersToGroups: many(usersToGroups) }));
  export const groupsRelations = relations(groups, ({ many }) => ({ usersToGroups: many(usersToGroups) }));
  export const utgRelations = relations(usersToGroups, ({ one }) => ({
    user: one(users, { fields: [usersToGroups.userId], references: [users.id] }),
    group: one(groups, { fields: [usersToGroups.groupId], references: [groups.id] }),
  }));
  ```
  Query goes through the junction: `with: { usersToGroups: { with: { group: true } } }`.
  (The `through` shortcut that skips the junction is a **v1** feature.)
- **Multiple relations between the same two tables** (author + reviewer): give
  each a matching `relationName` on both sides.

### Relational gotchas

- **PlanetScale + `mysql2`**: `drizzle({ client, schema, mode: "planetscale" })`
  — PlanetScale rejected lateral joins historically; wrong `mode` ⇒ runtime
  error. Native `drizzle-orm/planetscale` handles it.
- **`offset` in nested `with`** — not supported on 0.45 (top-level only).
- **`db.$cache` / `$withCache` don't cover `db.query.*`** yet.
- Index the FK columns the relational builder joins on — `authorId` on the
  "many" side, both columns + a composite index on a junction table.
- Prepared relational queries: `db.query.users.findMany({...}).prepare("name")`,
  placeholders in `where` / `limit` / `offset`.

## Iterators & raw

- `db.select().from(users).iterator()` — async iterator for huge result sets
  (MySQL fully; pg/sqlite WIP).
- `db.execute(sql\`...\`)` (pg/mysql), `db.all/get/values/run(sql\`...\`)`
  (sqlite) for anything the builder can't express.
- `drizzle.mock()` — a `db` with no connection, for unit-testing SQL output.
