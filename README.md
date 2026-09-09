# violets-skills

A [Claude Code plugin marketplace](https://docs.claude.com/en/docs/claude-code/plugins) of agent skills.

## Add this marketplace

```
/plugin marketplace add violetbuse/violets-skills
```

Then install a plugin:

```
/plugin install celld@violets-skills
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
vendor/
  celld/                            # git submodule: denoland/celld source, for reference while authoring the skill
```

Clone with submodules: `git clone --recurse-submodules <url>`, or after cloning:
`git submodule update --init`.

## Plugins

| Plugin | Description |
| ------ | ----------- |
| `celld` | Working with [celld](https://github.com/denoland/celld) — self-hosted, distributed Durable Objects. Writing and deploying Cloudflare Workers apps to a celld fleet, the `celld` CLI, fleet operations, and celld's durability guarantees. |
