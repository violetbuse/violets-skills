# violets-skills

A [Claude Code plugin marketplace](https://docs.claude.com/en/docs/claude-code/plugins) of agent skills.

## Install

In Claude Code, add the marketplace once:

```
/plugin marketplace add violetbuse/violets-skills
```

Then install any of the plugins:

```
/plugin install celld@violets-skills
/plugin install drizzle-orm@violets-skills
/plugin install planetscale-postgres-serverless@violets-skills
```

To pick up later changes, re-sync the marketplace:

```
/plugin marketplace update violets-skills
```

## Layout

```
.claude-plugin/marketplace.json   # marketplace manifest, lists plugins
plugins/
  celld/
    .claude-plugin/plugin.json     # plugin manifest
    skills/
      celld/
        SKILL.md                    # the skill
        reference/                  # architecture, operations, cloudflare-compat
  drizzle-orm/
    .claude-plugin/plugin.json
    skills/
      drizzle-orm/
        SKILL.md
        reference/                  # schema, queries, kit-migrations, drivers-and-perf, cloudflare-d1-do, v1-migration
  planetscale-postgres-serverless/
    .claude-plugin/plugin.json
    skills/
      planetscale-postgres-serverless/
        SKILL.md
        reference/                  # neonconfig-and-frameworks
vendor/
  celld/                            # git submodule: denoland/celld source, for reference while authoring the skill
  drizzle-orm/                      # git submodule: drizzle-team/drizzle-orm @ 0.45.2
  drizzle-orm-docs/                 # git submodule: drizzle-team/drizzle-orm-docs @ 11695b7 (last pre-v1 snapshot)
  neon-serverless/                  # git submodule: neondatabase/serverless @ v1.1.0
```

Clone with submodules: `git clone --recurse-submodules <url>`, or after cloning:
`git submodule update --init`.

## Plugins

### `celld`

```
/plugin install celld@violets-skills
```

Working with [celld](https://github.com/denoland/celld) — self-hosted, distributed Durable Objects. Writing and deploying Cloudflare Workers apps to a celld fleet, the `celld` CLI, fleet operations, and celld's durability guarantees.

### `drizzle-orm`

```
/plugin install drizzle-orm@violets-skills
```

Working with [Drizzle ORM](https://github.com/drizzle-team/drizzle-orm) and Drizzle Kit — schema declaration, the SQL-like and relational query APIs, migrations, driver setup, serverless performance (incl. Cloudflare D1 & Durable Object SQLite), and the type/date/number mapping pitfalls. Targets `drizzle-orm` 0.45.x / `drizzle-kit` 0.31.x, with notes on the v1 RC.

### `planetscale-postgres-serverless`

```
/plugin install planetscale-postgres-serverless@violets-skills
```

Connecting to [PlanetScale Postgres](https://planetscale.com/docs/postgres/connecting/neon-serverless-driver) from serverless/edge runtimes with the [Neon serverless driver](https://github.com/neondatabase/serverless) — HTTP vs WebSocket mode, the required `neonConfig`, connection strings, framework setup (Workers/Vercel/Deno/Netlify), ORM integration, and the `pg` + PSBouncer alternative.
