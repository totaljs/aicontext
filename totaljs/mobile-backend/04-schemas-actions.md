# Schemas, Actions, And Validation

`NEWSCHEMA()` files are the feature implementation. Routes only expose them.

```javascript
NEWSCHEMA('Orders', function(schema) {

	schema.action('read', {
		name: 'Read order',
		params: '*id:UID',
		action: async function($) {
			var response = await DATA.read('view_order').id($.params.id).error(404).promise($);
			$.success(response);
		}
	});
});
```

## Context

| Field | Meaning |
|-------|---------|
| `$` | Action context |
| `model` | Validated `input` |
| `$.params` | Path params |
| `$.query` | Query params |
| `$.user` | Session |
| `$.headers` | Headers |
| `$.files` | Uploads on upload routes |
| `$.ip` | IP |

Do not parse the raw body when `input` / `query` / `params` already did.

## Validation

```javascript
schema.action('create', {
	input: '*name:String,note:String,total:Number',
	action: async function($, model) {
		if (!FUNC.require_perm($, 'orders.manage'))
			return;
		model.id = UID();
		model.userid = $.user.id;
		model.dtcreated = NOW;
		await DATA.insert('tbl_order', model).promise($);
		$.success(model.id);
	}
});
```

`*` means required. Use it only when the action cannot continue without the field.

## Common patterns

List:

```javascript
schema.action('list', {
	query: 'search:String,page:Number,limit:Number',
	action: async function($) {
		var p = FUNC.paginate($.query);
		var result = await DATA.list('view_order')
			.where('isremoved', false)
			.autoquery($.query, 'id:String,name:String,dtcreated:Date', 'dtcreated_desc', 50)
			.paginate(p.page, p.limit)
			.promise($);
		$.callback(FUNC.list_payload(result, p.page, p.limit));
	}
});
```

Ownership:

```javascript
schema.action('check_owner', {
	params: '*id:UID',
	action: async function($) {
		var row = await DATA.read('tbl_order').id($.params.id).error(404).promise($);
		if (row.userid !== $.user.id && !FUNC.can($, '*')) {
			$.invalid(403);
			return;
		}
		$.success();
	}
});
```

## Helpers

- local function in the schema file for feature-only formatting
- `FUNC.*` for shared domain utilities
- `MODS.*` for integrations
- `definitions/` for boot and `AUTH`

Do not encode mobile UI in helpers. Return domain data.

## Errors

```javascript
$.invalid('@(Invalid phone number)');
$.invalid(401);
$.invalid(403);
$.invalid(404);
```
