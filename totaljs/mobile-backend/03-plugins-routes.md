# Plugins And Route Registration

A plugin is the feature unit. Public JSON contracts use `NEWACTION()` so each route and its validation live with the action ID. Keep ordinary HTTP, file, and socket routes in `exports.install()`.

```javascript
exports.icon = 'ti ti-box';
exports.name = '@(Orders)';
exports.position = 5;
exports.visible = function(user) {
	return user.sa || (user.permissions || []).includes('orders');
};

NEWACTION('Orders|list', {
	route: '+API /api/',
	action: async function($) {
		$.callback(await DATA.list('tbl_order').promise($));
	}
});

NEWACTION('Orders|read', {
	input: '*id:UID',
	route: '+API /api/',
	action: async function($, model) {
		$.callback(await DATA.read('tbl_order').id(model.id).error(404).promise($));
	}
});

NEWACTION('Orders|create', {
	input: '*name:String',
	route: '+API /api/',
	action: async function($, model) {
		model.id = UID();
		await DATA.insert('tbl_order', model).promise($);
		$.success(model.id);
	}
});
```

Do not scatter feature routes through controllers. Controllers keep gateway-level HTTP (upload, health, sockets).

## Auth route prefix

```javascript
NEWACTION('Orders|list', {
	route: '+API /api/',
	action: function($) {
		// Return the authorized user's order list.
	}
});

NEWACTION('Account|login', {
	input: '*email:Email,*password:String',
	route: '-API /api/',
	action: function($, model) {
		// Validate credentials and return the session.
	}
});
```

- `+API` / `-API` — session required / public
- `input` — validated client data; it does not alter the stable public action ID

## Composition

Call reusable preconditions explicitly with `ACTION('Orders|check', model)` inside `Orders|create`. Do not hide a workflow behind a route string containing six silent actions.

## Public allowlist

Publish the public schema names. The mobile app should not guess from `+`/`-`.

```text
Account|login
Account|create
API|ping
Catalog|list
Catalog|read
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
	NEWACTION('Inventory|list', {
		route: '+API /api/',
		action: async function($) {
			$.callback(await DATA.list('tbl_inventory').promise($));
		}
	});
};
```

Gate on `CONF` / `FUNC`, not on a second codebase.
