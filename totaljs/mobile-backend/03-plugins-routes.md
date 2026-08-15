# Plugins And Route Registration

A plugin is the feature unit. Routes live in `exports.install()`. Actions live in auto-loaded `schemas/`.

```javascript
exports.icon = 'ti ti-box';
exports.name = '@(Orders)';
exports.position = 5;
exports.visible = function(user) {
	return user.sa || (user.permissions || []).includes('orders');
};

exports.install = function() {
	ROUTE('+API /api/  -orders_list     --> Orders/list');
	ROUTE('+API /api/  -orders_read/{id} --> Orders/read');
	ROUTE('+API /api/  +orders_create   --> Orders/create');
};
```

Do not scatter feature routes through controllers. Controllers keep gateway-level HTTP (upload, health, sockets).

## Auth vs operation prefix

```javascript
ROUTE('+API /api/  -orders_list   --> Orders/list');
ROUTE('-API /api/  +auth_login    --> Auth/login');
```

- `+API` / `-API` — session required / public
- `-orders_list` / `+orders_create` — Total.js read/write convention; the public name is without the prefix

## Composition

```javascript
ROUTE('+API /api/  +orders_create --> Orders/check Orders/insert (response)');
```

Use for reusable preconditions (uniqueness, ownership). Do not hide a workflow behind six silent actions.

## Public allowlist

Publish the public schema names. The mobile app should not guess from `+`/`-`.

```text
auth_login
auth_register
api_ping
catalog_list
catalog_read
```

Protected schemas return 401 without a token. Public schemas must not log the user out if a stale token is sent — either use `-API` or ignore a bad token on those operations.

## Extra HTTP routes

```javascript
ROUTE('GET /health', health);
ROUTE('+POST /upload/', upload, ['upload'], 1024 * 5);
ROUTE('FILE /download/*.*', files);
ROUTE('SOCKET /realtime/', socket);
```

Keep these rare. Most mobile work stays in API Routing.

## Optional feature packs

```javascript
exports.install = function() {
	if (!FUNC.pack_enabled('inventory'))
		return;
	ROUTE('+API /api/  -inventory_list --> Inventory/list');
};
```

Gate on `CONF` / `FUNC`, not on a second codebase.
