# Total.js 5 Architecture

Total.js is not Express. It is not Nest. Do not redesign it around routers, DI containers, or `require()` graphs.

The running application is a **single global framework instance**. Files are auto-loaded by directory. Application code shares behavior through globals, not imports.

## Boot

`index.js` stays tiny. This is one of the few legitimate `require()` calls:

```javascript
require('total5');

const release = process.env.NODE_ENV === 'production';
const port = Number(process.env.PORT || 8000);

Total.run({ release, port });
```

Do not build an application container, logger factory, or config loader around boot. Total.js already loads `config`, `.env`, definitions, modules, plugins, controllers, schemas, and actions.

## Project shape

```text
backend/
  index.js                 # require('total5'); Total.run(...)
  config                   # key : value  → CONF
  config-debug             # debug overlay
  config-release           # production overlay (optional)
  definitions/             # AUTH, DATA init, FUNC.*, MAIN.*, CRON, ON()
  modules/                 # auto-loaded → MODS.<filename>
  controllers/             # HTTP / FILE / SOCKET routes, exports.install()
  plugins/<feature>/
    index.js               # metadata + ROUTE() registration
    schemas/*.js           # NEWSCHEMA() actions (auto-loaded)
  actions/                 # optional NEWACTION() files (auto-loaded)
  schemas/                 # optional shared NEWSCHEMA() files
  workers/                 # optional Total.js workers
  public/                  # static files
  views/                   # HTML views when needed
```

The framework loads JavaScript from these directories. You do not register them.

Load order that matters (verified from total5 `index.js`):

1. `.env` / `config` → `CONF`
2. `/modules/*.js` → `MODS` and `exports.install()`
3. `/actions`, `/schemas`, `/definitions`, `/controllers`
4. `/plugins/<id>/index.js` then that plugin’s schemas/definitions/controllers
5. each file’s `exports.install()` when that file is loaded

Definitions therefore see modules. Plugin schemas see `FUNC` assigned in `definitions/`. A module’s top-level body must not assume `FUNC.*` already exists; call `FUNC` from exported functions that run after boot.

## Global / singleton model

These objects already exist. Use them.

| Global | Role |
|--------|------|
| `Total` / `F` | Framework instance. `F` is the older alias. Prefer `Total` in new code. |
| `CONF` | Application configuration from `config` files |
| `FUNC` | Reusable application functions |
| `MAIN` | Shared runtime data (caches, socket maps, stores) |
| `MODS` | Auto-loaded `/modules/*.js` |
| `PLUGINS` | Auto-loaded `/plugins/<id>` |
| `DATA` | Shared QueryBuilder database interface |
| `DB()` | New QueryBuilder controller (batch/legacy). Prefer `DATA`. |
| `PATH` | Application paths and filesystem helpers |
| `CACHE` | In-memory TTL cache |
| `DEF` | Framework defaults and hooks |
| `DEBUG` | `true` unless `NODE_ENV === 'production'` |
| `NOW` | Current date, refreshed by the service timer |
| `REPO` | Framework repository bag (views/components). Not an app session store. |
| `TEMP` | Short-lived temporary bag |

There is no service locator to invent. `FUNC`, `MAIN`, and `MODS` are the locator.

## `require()` policy

`require()` is exceptional.

Legitimate:

```javascript
require('total5');                              // boot only
require('querybuilderpg').init('', CONF.database, 1, ERROR('DB'));
const { S3Client } = require('@aws-sdk/client-s3'); // real external SDK
```

Not legitimate inside controllers, schemas, definitions, or modules:

```javascript
// WRONG — internal sharing
const storage = require('../modules/storage');
const helper = require('./lib/db');
const Path = require('path');
const Fs = require('fs');
const https = require('https');
```

Use instead:

```javascript
MODS.storage.putFile(...);
FUNC.hash_password(password);
PATH.join(PATH.public(), 'uploads');
PATH.fs.readFileSync(filename);
RESTBuilder.POST(url, payload).promise($);
```

CLI scripts, one-off migrations, and tests that run *outside* `Total.run()` may `require()` Node built-ins and `pg`. Application runtime code must not.

## Where each kind of code goes

| Need | Put it here |
|------|-------------|
| Shared helper used by several features | `FUNC.*` in `definitions/` |
| Runtime map, cache, socket registry | `MAIN.*` |
| External integration with its own API | `/modules/<name>.js` and call `MODS.<name>` |
| Database driver init | `definitions/db.js` via `querybuilderpg` |
| Auth | `AUTH()` in `definitions/auth.js` |
| Feature HTTP API | `plugins/<feature>/index.js` + `schemas/` |
| Isolated action | `NEWACTION()` in `/actions/` or a plugin schema |
| Upload, health, webhook, OAuth redirect | `controllers/` + `ROUTE('GET|POST|FILE ...')` |
| Periodic work | `CRON()` or `ON('service')` in definitions |
| Heavy isolated process | `/workers/*.js` |
| Config value | `CONF.key` from `config` |

Do not add `src/`, `lib/`, `services/`, `repositories/`, `utils/`, or `middleware/` unless the framework directory already covers the need.

## Feature unit

A backend feature is a plugin:

```javascript
// plugins/orders/index.js
exports.name = '@(Orders)';
exports.icon = 'ti ti-receipt';
exports.position = 10;

exports.install = function() {
	ROUTE('+API /api/  -orders_list        --> Orders/list');
	ROUTE('+API /api/  -orders_read/{id}   --> Orders/read');
	ROUTE('+API /api/  +orders_create      --> Orders/create');
};
```

```javascript
// plugins/orders/schemas/orders.js
NEWSCHEMA('Orders', function(schema) {

	schema.action('list', {
		query: 'page:Number,limit:Number,search:String',
		action: async function($) {
			if (!FUNC.require_perm($, 'orders.read'))
				return;
			var p = FUNC.paginate($.query);
			var result = await DATA.list('tbl_order')
				.where('isremoved', false)
				.autoquery($.query, 'id:String,name:String,dtcreated:Date', 'dtcreated_desc', 100)
				.paginate(p.page, p.limit)
				.promise($);
			$.callback(FUNC.list_payload(result, p.page, p.limit));
		}
	});
});
```

The schema file is auto-loaded. Do not `require()` it from the plugin.

## `NEWACTION` vs `NEWSCHEMA`

Both are first-class Total.js 5 APIs.

- **`NEWSCHEMA('Name', ...)` + `schema.action()`** — group related actions. Preferred for plugin domains.
- **`NEWACTION('Name|action', { route, action })`** — standalone action, can declare its own route.

Do not add a third style (service class that both call). Do not put business rules in controllers when an action exists.

## API surface

Prefer one API Routing gateway for app/JSON work:

```http
POST /api/
Content-Type: application/json
x-token: <session>

{ "schema": "orders_list?page=1", "data": {} }
```

Use ordinary HTTP routes only for files, health, webhooks, SSO redirects, and WebSockets.

## Configuration

`config` is not JSON:

```text
name              : My App
database          : postgresql://user:pass@127.0.0.1:5432/app
salt              : change-me
allow_register    : true
```

Read it as `CONF.database`, `CONF.salt`. Do not wrap it. Environment variables are for deployment overrides, not a parallel config system. Prefer `config` typed keys such as `api_key (env) : MY_API_KEY` so the value still lands on `CONF`.

## What this architecture is not

- Not REST-resource controllers
- Not dependency injection
- Not one `require()` per file
- Not repository / unit-of-work classes
- Not Express middleware chains
- Not a generic Node.js folder layout with Total.js bolted on

If a pattern would make sense in Nest or Fastify, it is probably the wrong Total.js pattern.
