# Data Access, Lists, And Responses

Use `DATA`, not `DB()`, not `require('pg')`, not a repository.

## QueryBuilder and views

```javascript
var response = await DATA.list('view_order')
	.where('isremoved', false)
	.autoquery($.query, 'id:String,name:String,status:String,dtcreated:Date', 'dtcreated_desc', 50)
	.promise($);
$.callback(response);
```

Put joins and display names in SQL views. Do not make the mobile app stitch three endpoints to render a row.

## Field allowlists

```javascript
builder.fields('id,name,logo,status,dtcreated');
```

Never return password hashes, reset tokens, raw session rows, or third-party secrets.

## Public listing rules

```javascript
builder.where('isremoved', false);
builder.where('status', 'published');
```

Admin lists may be broader and stay behind `+API` plus admin permissions.

## List shapes

Simple:

```javascript
$.callback(items);
```

Paginated:

```javascript
$.callback({ items: items, count: count, page: page, limit: limit });
```

`DATA.list()` already returns `{ items, count }`. Document which shape each schema uses. Empty list is not an error.

## Response helpers

```javascript
$.success(item);
$.success(id);
$.success();
$.callback(result);
$.invalid('@(Message)');
$.invalid(404);
```

Do not invent `{ ok: true }` or `{ data: ... }`.

## Raw SQL

```javascript
var rows = await DATA.query(`
	SELECT id, name, saleprice
	FROM tbl_product
	WHERE isremoved = FALSE AND categoryid = $1
	LIMIT $2
`, [categoryid, limit]).promise($);
```

Dynamic filters: trusted fragments + parameter array. Never concatenate user input.

## Errors

| Situation | Backend |
|-----------|---------|
| missing token | `AUTH` → 401 |
| not owner | `$.invalid(403)` |
| missing row | `$.invalid(404)` or `.error(404)` |
| validation | `$.invalid('@(Message)')` |
| empty list | `[]` or `{ items: [], count: 0 }` |
