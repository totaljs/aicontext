# Total.js AI Context

Practical context for AI coding agents working on **Total.js 5** backends.

This repository exists for one reason: help an agent produce code that feels native to Total.js instead of generic Node.js. Total.js has its own runtime model, globals, auto-loaded files, routing style, schema actions, and plugin conventions. When an agent brings Express or NestJS habits into a Total.js project, the result may run, but it is usually harder to maintain.

Use these files as project context, onboarding material, or implementation guardrails for backend, API, mobile, and frontend integration work.

## Start Here

Read these three files before changing backend structure:

1. [Architecture](totaljs/architecture.md) explains how a Total.js application is organized.
2. [Anti-patterns](totaljs/anti-patterns.md) lists the abstractions and habits to avoid.
3. [Style](totaljs/style.md) gives naming, formatting, and implementation conventions.

Then open the guide that matches the task.

## Guide Map

| Need | Read |
| --- | --- |
| Understand framework globals such as `FUNC`, `MAIN`, `MODS`, `DATA`, `CONF`, `PATH`, and `Total` | [Globals](totaljs/globals.md) |
| Build schema actions and validation flows | [Actions and schemas](totaljs/actions.md) |
| Add routes, controllers, and API endpoints | [Controllers and routing](totaljs/controllers-and-routing.md) |
| Organize shared logic with definitions and modules | [Modules and definitions](totaljs/modules-and-definitions.md) |
| Work with PostgreSQL and QueryBuilder | [Databases](totaljs/databases.md) |
| Implement sessions, auth, and protected routes | [Auth](totaljs/auth.md) |
| Handle files, paths, uploads, and storage | [Filesystem](totaljs/filesystem.md) |
| Use or write Total.js plugins | [Plugins](totaljs/plugin.md) |
| Add WebSockets, cron work, jobs, and workers | [Realtime and jobs](totaljs/realtime-and-jobs.md) |
| Extend framework behavior with `Total.extend()` or `DEF` | [Extensions](totaljs/extensions.md) |
| Design a backend contract for mobile apps | [Mobile backend](totaljs/mobile-backend.md) |
| Connect web, React Native, or Flutter clients | [Frontend integration](frontend-integration/README.md) |
| Build jComponent interfaces | [jComponent UI](jcomponent/ui-component.md) |

## Core Rule

Before adding an import, wrapper, service layer, repository, dependency injection setup, middleware package, cron package, or WebSocket package, ask:

> Does Total.js already provide this through a global, definition, module, schema, route, plugin, or auto-loaded file?

If the answer is yes, use the Total.js mechanism. Internal `require()` calls and external abstractions should be rare, intentional, and easy to justify.

## How To Use This Repo

For a new Total.js project, copy or reference this repository as AI context before asking an agent to implement backend work. Point the agent to [AGENTS.md](AGENTS.md), then to the guide that matches the feature.

For an existing project, keep these docs near the codebase and treat them as the default engineering standard for Total.js work. Project-specific rules can still override them, but they should not reintroduce Express-style structure unless the project deliberately chose that architecture.

For mobile or frontend work, start with the backend contract first. The client guides assume the backend exposes schema-based API routes and uses session tokens consistently.

## Compatibility

These guides target:

- Total.js 5 (`total5`)
- Total.js API Routing with `{ schema, data }` requests
- Total.js globals such as `Total`, `FUNC`, `MAIN`, `MODS`, `DATA`, `CONF`, and `PATH`
- `querybuilderpg` for PostgreSQL
- Mobile clients using React Native, Flutter, or similar native stacks

Legacy Total.js 4 names are mentioned only where they still appear in real projects. Prefer the Total.js 5 globals and runtime conventions for new work.

## What This Is Not

This is not a generic Node.js architecture guide. It does not recommend Express controllers, Nest modules, repository classes, dependency injection containers, or custom middleware stacks as the default approach.

It is also not a replacement for official Total.js documentation. It is a focused implementation guide for agents and teams that need consistent, framework-native code.
