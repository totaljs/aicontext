# API Routing Contract

The mobile app calls one endpoint and puts the operation in `schema`.

## Request shape

```http
POST /api/
Content-Type: application/json
x-token: <session_token>

{
  "schema": "orders_read/ord123?include=items",
  "data": { "optional": "payload" }
}
```

The path is project-defined (`/api/` or `/`). Configure it in the client.

## Schema string

```text
<operation>
<operation>/<id>
<operation>/<id>/<childid>
<operation>?key=value
<operation>/<id>?key=value
```

Examples:

```text
auth_login
orders_list?page=1&limit=20
orders_read/ord123
orders_update/ord123
```

## Naming

Readable, action-oriented, stable:

| Schema | Meaning |
|--------|---------|
| `auth_login` | login |
| `auth_me` | current user |
| `orders_list` | list |
| `orders_read/{id}` | detail |
| `orders_create` | create |

Do not leak table names (`tbl_order_select`). Do not rename schemas casually — they are as public as REST URLs.

## Query and params

Filters belong in the schema string, not on the HTTP URL.

```javascript
schema.action('list', {
	query: 'search:String,limit:Number,page:Number,sort:String',
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
ROUTE('+API /api/  -orders_read/{id} --> Orders/read');

schema.action('read', {
	params: '*id:UID',
	action: async function($) {
		var item = await DATA.read('view_order').id($.params.id).error(404).promise($);
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

When a contract changes, add `orders_list_v2` (or a clearly new name) and keep the old schema until old app builds die. Compatibility aliases (`logo` and `logoUrl`) are allowed if documented.
