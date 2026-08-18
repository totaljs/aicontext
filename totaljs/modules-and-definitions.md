# Modules, definitions, FUNC, MAIN, MODS

This is how Total.js 5 shares application code. Internal `require()` is the wrong tool.

## Auto-loaded modules

Every `/modules/*.js` file is required by the framework and stored on `MODS`.

```javascript
// modules/storage.js
exports.install = function() {
	// optional startup
};

exports.putFile = async function(relPath, body, mime) {
	// talk to S3 / disk
};

exports.getUrl = async function(relPath) {
	// ...
};
```

The module key is the filename without `.js`. Override with `exports.id` if needed.

```javascript
// anywhere after boot — controllers, schemas, definitions, other modules
await MODS.storage.putFile(path, body, mime);
```

Do **not**:

```javascript
const storage = require('../modules/storage');
```

`exports.install()` runs when the module is loaded. Keep it short. Long work belongs in `ON('ready')`.

## What belongs in a module

A module is a reusable integration with its own surface:

- object storage
- mail provider wrapper that is more than `MAIL()`
- search / AI engine
- payment SDK
- push gateway

A module should expose functions on `exports`. It may read `CONF`. It may use `DATA` and `FUNC` from functions that run after boot, not at parse time if those assignments live in definitions.

A module is **not** a dumping ground for `paginate()`, permission checks, or SQL. Those are `FUNC` or actions.

## Definitions

`/definitions/*.js` is for cross-cutting startup:

| File (typical) | Assigns |
|----------------|---------|
| `db.js` | `querybuilderpg` → `DATA` |
| `auth.js` | `AUTH()` |
| `func.js` | `FUNC.*` |
| `sessions.js` | `MAIN.sessions` |
| `jobs.js` | `CRON()` / `ON('service')` |
| `realtime.js` | `FUNC.realtime_*`, `MAIN.ws_users` |

Definitions are auto-loaded. They do not need `exports.install()` unless they register routes.

## `FUNC`

Reusable application functions. Assign them in definitions.

```javascript
FUNC.hash_password = function(password) {
	return String(password).sha256(CONF.passwordizator);
};

FUNC.can = function($, perm) {
	var user = $.user;
	if (!user)
		return false;
	var perms = user.permissions || [];
	if (perms.includes('*') || (user.roles || []).includes('admin'))
		return true;
	return perms.includes(perm);
};

FUNC.require_perm = function($, perm) {
	if (!FUNC.can($, perm)) {
		$.invalid(403);
		return false;
	}
	return true;
};

FUNC.paginate = function(query) {
	var page = Math.max(1, Number(query.page) || 1);
	var limit = Math.min(100, Math.max(1, Number(query.limit) || 25));
	return { page: page, limit: limit, skip: (page - 1) * limit };
};

FUNC.list_payload = function(result, page, limit) {
	var items = result && result.items ? result.items : (result || []);
	var count = result && typeof result.count === 'number' ? result.count : items.length;
	return { items: items, page: page || 1, limit: limit || 25, count: count };
};

FUNC.audit = async function($, action, entity, entityid, payload) {
	try {
		await DATA.insert('tbl_audit', {
			id: UID(),
			userid: $.user ? $.user.id : null,
			entity: entity || null,
			entityid: entityid || null,
			action: action,
			payload: payload ? JSON.stringify(payload) : null,
			ip: $.ip,
			dtcreated: NOW
		}).promise();
	} catch (err) {
		console.error('audit failed', err.message);
	}
};
```

Call them from actions:

```javascript
if (!FUNC.require_perm($, 'users.manage'))
	return;
```

Good `FUNC` candidates: password hashing, permission checks, session create/refresh, safe user snapshots, pagination helpers, audit/notify, small domain calculators used by more than one plugin.

Bad `FUNC` candidates: a single plugin’s private formatter (keep it local), HTTP handlers, raw SQL dumps of a whole feature.

## `MAIN`

Shared **runtime** data. Not configuration. Not the database.

```javascript
MAIN.sessions = {
	async get(key) { return map.get(key) || null; },
	async set(key, value) { map.set(key, value); },
	async delete(key) { map.delete(key); }
};

MAIN.ws_users = {};     // userid → Set of websocket clients
```

Use `MAIN` for:

- session cache
- websocket connection maps
- short-lived boot flags (`MAIN.rt_redis_ready`)
- process-local registries

Do not put tenant business records in `MAIN` when they belong in PostgreSQL.

`REPO` exists as `{}` and is used by some Total.js UI/admin apps. Do not use it as an application session store. That is `MAIN`.

## `CACHE`

Framework in-memory cache with TTL:

```javascript
CACHE.set('user_' + id, snapshot, '10 minutes');
var snapshot = CACHE.get('user_' + id);
CACHE.remove('user_' + id);
CACHE.reset('user_');   // keys containing this string
```

Invalidate on update/logout. Do not authorize from stale cache if the DB says the user is gone.

## Load-order rule

```text
modules  →  definitions  →  controllers / plugin install()
```

Safe:

```javascript
// definitions/func.js
FUNC.media_url = function($, url) {
	if (!url)
		return url;
	if (url.indexOf('http') === 0)
		return url;
	return (CONF.url || '') + url;
};

// later, in a schema
url = FUNC.media_url($, url);
await MODS.storage.getUrl(path);
```

Unsafe:

```javascript
// modules/storage.js at top level
FUNC.something();   // FUNC.* from definitions may not exist yet
```

If a module needs definition helpers, call them from exported functions, not at load.

## External npm packages

A module is the right place to `require()` an SDK the framework does not provide:

```javascript
// modules/storage.js
const { S3Client, PutObjectCommand } = require('@aws-sdk/client-s3');
```

That is the boundary. Callers still use `MODS.storage`, never the SDK.

## Prefer `FUNC` / `MODS` over `Total.extend()`

Extend `$` only when many actions need the same *controller* helper. Ordinary functions belong on `FUNC`. See [extensions.md](extensions.md).
