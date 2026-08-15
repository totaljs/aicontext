# Agent instructions

You are working in a **Total.js 5** project. This directory is the framework context.

1. Read [readme.md](readme.md)
2. Read [totaljs/architecture.md](totaljs/architecture.md)
3. Read [totaljs/anti-patterns.md](totaljs/anti-patterns.md)

Rules that override generic Node.js habits:

- Do not `require()` internal application files. Use `FUNC`, `MAIN`, `MODS`, `DATA`, `CONF`, `PATH`.
- Do not add Express/Nest layers (services, repositories, middleware stacks, DI).
- Prefer `DATA` over `DB()` and over `pg`.
- Prefer `ROUTE()`, `NEWSCHEMA` / `NEWACTION`, `AUTH()`, `CRON()`, `ON('service')`.
- Prefer `PATH` / `Total.Fs` over `require('fs')` / `require('path')`.
- Prefer `RESTBuilder` over `require('http')` / `require('https')`.

Do not copy another product's tables or schema names. Copy mechanisms, not domains.
