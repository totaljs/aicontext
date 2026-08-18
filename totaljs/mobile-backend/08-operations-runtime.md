# Operations, Jobs, Integrations, And Runtime Hooks

See [../realtime-and-jobs.md](../realtime-and-jobs.md) and [../modules-and-definitions.md](../modules-and-definitions.md).

## Definitions

Use `definitions/` for:

- `querybuilderpg` / `DATA`
- `AUTH()`
- `FUNC.*`
- `MAIN.sessions`, `MAIN.ws_users`
- `CRON()` / `ON('service')`
- optional Redis / push bridges

Not for feature CRUD.

## `ON('ready')`

```javascript
ON('ready', function() {
	// warm code lists, log store type
});
```

Do not block boot with long jobs.

## `ON('service')` and `CRON()`

```javascript
ON('service', function(counter) {
	if (counter % 30 === 0 && MAIN.sessions.flush)
		MAIN.sessions.flush();
});

CRON('*/5 * * * *', function() {
	FUNC.run_maintenance();
});
```

Do not `setInterval` in definitions. Do not add `node-cron`.

## Shared globals

| Global | Use |
|--------|------|
| `CONF` | config |
| `FUNC` | helpers |
| `MAIN` | runtime maps/caches |
| `MODS` | auto-loaded modules |
| `DATA` | SQL |
| `NOW` | timestamps |
| `CACHE` | short TTL cache |

`REPO` is not an application session store.

## Integrations

Mail, push, S3, OAuth, maps: one module or `FUNC` wrapper. Callers never `require()` the SDK.

```javascript
await MODS.storage.putFile(path, body, mime);
var json = await RESTBuilder.POST(url, payload).header('Authorization', 'Bearer ' + CONF.token).promise($);
MAIL(user.email, '@(Welcome)', 'mail/welcome', user);
```

## Diagnostics

Protect log/debug routes with a token or admin permission. Do not log tokens or passwords.

## Tests

`.test.api` fixtures next to a plugin are useful. Do not commit live tokens.

```bash
node --check plugins/orders/schemas/orders.js
```
