# Total.js AI Context

Coding-agent guidelines for **Total.js 5** backends.

This is not generic Node.js advice. Total.js is a global/singleton framework. Agents that import Express/Nest habits (`require()` for internals, service containers, repositories, middleware stacks) produce non-idiomatic code even when it runs.

## Read this first

1. [Architecture](totaljs/architecture.md) — how Total.js expects code to be structured
2. [Anti-patterns](totaljs/anti-patterns.md) — what not to invent
3. [Style](totaljs/style.md) — formatting and naming conventions

Then open the topic you are implementing:

| Topic | File |
|-------|------|
| Globals (`FUNC`, `MAIN`, `MODS`, `DATA`, `CONF`, `PATH`, `Total`) | [totaljs/globals.md](totaljs/globals.md) |
| Actions and schemas | [totaljs/actions.md](totaljs/actions.md) |
| Controllers, `$`, `ROUTE()` | [totaljs/controllers-and-routing.md](totaljs/controllers-and-routing.md) |
| Modules, definitions, `FUNC`, `MODS` | [totaljs/modules-and-definitions.md](totaljs/modules-and-definitions.md) |
| PostgreSQL + QueryBuilder | [totaljs/databases.md](totaljs/databases.md) |
| Auth and sessions | [totaljs/auth.md](totaljs/auth.md) |
| Filesystem, `PATH`, uploads | [totaljs/filesystem.md](totaljs/filesystem.md) |
| Plugins | [totaljs/plugin.md](totaljs/plugin.md) |
| WebSockets, cron, workers | [totaljs/realtime-and-jobs.md](totaljs/realtime-and-jobs.md) |
| `Total.extend()` and `DEF` | [totaljs/extensions.md](totaljs/extensions.md) |
| Mobile/API contract | [totaljs/mobile-backend.md](totaljs/mobile-backend.md) |
| Frontend clients | [frontend-integration/README.md](frontend-integration/README.md) |
| jComponent UI | [jcomponent/ui-component.md](jcomponent/ui-component.md) |

## The one rule that prevents most bad code

Before adding an import, wrapper, service, repository, middleware package, cron package, or WebSocket package, ask:

> Does Total.js already expose this as a global or auto-loaded mechanism?

If yes, use that. `require()` is exceptional.

## Compatible with

- Total.js 5 (`total5`)
- `querybuilderpg` for PostgreSQL
- API Routing (`ROUTE('API /api/ ...')` + `{ schema, data }`)

Legacy Total.js 4 names (`F` as the only entry, `DB()` as the default query API) are documented only where they still exist. Prefer the Total.js 5 globals: `Total`, `DATA`, `FUNC`, `MAIN`, `MODS`, `PATH`.
