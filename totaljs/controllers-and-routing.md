# Controllers, `$`, and `ROUTE()`

Controllers own HTTP, files, and sockets. They do not own domain rules.

## Controller file

```javascript
exports.install = function() {
	ROUTE('GET /', index);
	ROUTE('GET /health', health);
	ROUTE('+POST /upload/', upload, ['upload'], 1024 * 10);
	ROUTE('FILE /download/*.*', download);
	ROUTE('SOCKET /realtime/', socket);
};

function index($) {
	$.json({ name: CONF.name, api: '/api/' });
}

function health($) {
	$.json({ ok: true, ts: new Date().toISOString() });
}
```

`exports.install()` is called automatically.

## Two different `$` objects

This is a common agent mistake.

| Context | `$` is | Has |
|---------|--------|-----|
| `ROUTE('GET /', fn)` | HTTP **controller** | `$.json`, `$.view`, `$.file`, `$.html`, `$.plain`, `$.stream`, `$.filefs`, `$.redirect`, `$.success`, `$.invalid` |
| `NEWACTION` / `NEWSCHEMA` / `AUTH` | **Options** | `$.success`, `$.invalid`, `$.callback`, `$.redirect` — **not** `$.json` / `$.view` / `$.file` |

In an action, finish with `$.success(value)` or `$.callback(payload)`. Do not call `$.json()` there.

The rest of this page describes the HTTP controller. Action `$` is covered in [actions.md](actions.md).

### Request

| Field | Meaning |
|-------|---------|
| `$.query` | Query string object |
| `$.body` | Parsed body |
| `$.params` | Route params (`/files/{id}/` → `$.params.id`) |
| `$.headers` | Request headers |
| `$.files` | Uploaded files (`['upload']` routes) |
| `$.user` | Session from `AUTH()` + `$.success(session)` |
| `$.ip` | Client IP |
| `$.ua` / user-agent | Present on the controller / session helpers |
| `$.url` | Relative URL |
| `$.language` | Language when set |

On `NEWACTION` / `NEWSCHEMA` actions, `model` is the validated `input`. `$.model` is the same payload.

### Response

```javascript
$.json(obj);              // raw JSON
$.success(value);         // { success: true, value }
$.success();              // { success: true }
$.callback(payload);      // action callback (lists, query results)
$.invalid('@(Message)');  // validation / business error
$.invalid(401);
$.invalid(404);
$.redirect('/path');
$.view('index', model);
$.html(string);
$.plain(string);          // alias: $.text()
$.file('/abs/path', 'name.pdf');
$.filefs('files', id);
$.stream('application/pdf', stream, 'name.pdf');
$.proxy(opt);
$.cookie('name');                 // read
$.cookie('name', value, '7 days'); // write
```

`$.success(value)` uses `DEF.onSuccess` and returns `{ success: true, value }`. Do not rebuild that object by hand.

`$.done()` returns a Node-style callback that maps errors to `$.invalid` and success to the framework success envelope.

### Files on `$`

```javascript
var file = $.files[0];
if (!file) {
	$.invalid('@(No file uploaded)');
	return;
}

// Total.js uploaded file
var buffer = await file.read();
// or file.path / file.filename / file.type / file.size
```

Prefer `file.read()` or `file.fs('storage', id)` over `require('fs')`.

## `ROUTE()`

```javascript
ROUTE('GET /health', handler);
ROUTE('POST /hooks/stripe', handler);
ROUTE('+POST /upload/', handler, ['upload'], 1024 * 10);
ROUTE('FILE /documents/*.*', handler);
ROUTE('SOCKET /realtime/', handler);
```

Declare JSON API actions with `NEWACTION()` so the action ID, validation, and API route remain in one place.

### Auth flags

The first character of the method is an auth flag:

| Prefix | Meaning |
|--------|---------|
| `+API` / `+GET` | Authorized only (`AUTH` must `$.success(user)`) |
| `-API` / `-GET` | Unauthorized only |
| `API` / `GET` | Either |

This is evaluated from `AUTH()`. See [auth.md](auth.md).

### API Routing

`API` is POST plus a JSON envelope `{ schema, data }`.

```javascript
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

The `+API` prefix requires authentication. The presence of `input` controls validation; it does not change the public action ID.

Client call:

```http
POST /api/
x-token: ...
{ "schema": "Orders|read", "data": { "id": "abc123" } }
```

### Action composition

Call reusable preconditions explicitly from the public action with `ACTION('Orders|check', model)`, then return the final result. This keeps composition visible and avoids route strings that hide a pipeline of side effects.

### `NEWACTION` routes

```javascript
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

`route: '+API /api/'` registers the action on that gateway. `?` in a route string is replaced by `CONF.$api` (default `/api/`), so `'+API ?'` means `'+API /api/'`.

## What controllers should not do

- Parse auth tokens if `AUTH()` already did
- Contain SQL for a feature that has a plugin
- Invent `sendSuccess(res)` helpers
- `require()` modules or Node `fs`/`path`

Health, index, SSO redirects, webhooks, multipart upload, file download, and WebSocket gateways belong here.

## CORS

Once:

```javascript
// definitions/cors.js or controllers/default.js
CORS(); // allow all — fine for native mobile + token auth

// or explicit origins
CORS('https://app.example.com,http://localhost:3000');
```

Do not call `CORS()` in every plugin `install()`.
