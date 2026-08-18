# API Routing Contract

The mobile app calls one endpoint and puts the operation in `schema`.

## Request shape

```http
POST /api/
Content-Type: application/json
x-token: <session_token>

{
  "schema": "Orders|read?include=items",
  "data": { "id": "ord123" }
}
```

The path is project-defined (`/api/` or `/`). Configure it in the client.

## Schema string

```text
<Namespace>|<action>
<Namespace>|<action>?key=value
```

Examples:

```text
Account|login
Orders|list?page=1&limit=20
Orders|read
Orders|update
```

## Naming

Readable, action-oriented, stable:

| Schema | Meaning |
|--------|---------|
| `Account\|login` | login |
| `Account\|read` | current user |
| `Orders\|list` | list |
| `Orders\|read` | detail; `id` is declared input |
| `Orders\|create` | create |

Do not leak table names (`tbl_order_select`). Do not rename schemas casually — they are as public as REST URLs.

## Query and params

Filters belong in the schema string, not on the HTTP URL. Record IDs and other action inputs belong in `data`.

```javascript
NEWACTION('Orders|list', {
	query: 'search:String,limit:Number,page:Number,sort:String',
	route: '+API /api/',
	action: async function($) {
		var response = await DATA.list('view_order')
			.where('isremoved', false)
			.autoquery($.query, 'id:String,name:String,dtcreated:Date', 'dtcreated_desc', 50)
			.promise($);
		$.callback(response);
	}
});
```

```javascript
NEWACTION('Orders|read', {
	input: '*id:UID',
	route: '+API /api/',
	action: async function($, model) {
		var item = await DATA.read('view_order').id(model.id).error(404).promise($);
		$.success(item);
	}
});
```

## Errors

```javascript
$.invalid('@(Unsupported country)');
$.invalid(401);
$.invalid(404);
```

The client should normalize HTTP errors, `{ success: false, ... }`, validation errors, and string messages once.

## Versioning

When a contract changes, add `Orders|list_v2` (or a clearly new action name) and keep the old action until old app builds die. Compatibility aliases (`logo` and `logoUrl`) are allowed if documented.
