# Total.js coding style

These are the usual Total.js backend conventions. Match them in application JavaScript.

## Formatting

- Tabs, not spaces (tab width 4)
- Semicolons at the end of statements
- Short strings in single quotes `'...'`
- No unnecessary whitespace
- Classic `for` loops over `forEach()` where possible
- Do not declare `const` inside method scopes. Use `var` (or `let` for block bindings that must rebind). Total.js codebases are `var`-first.

```javascript
// preferred
var user = await DATA.read('tbl_user').id($.user.id).promise($);
for (var i = 0; i < items.length; i++)
	FUNC.normalize(items[i]);

// avoid in application files
const user = await DATA.read('tbl_user').id($.user.id).promise($);
items.forEach((item) => FUNC.normalize(item));
```

`const` at the top of a module for a real npm import is acceptable. Do not turn every local into `const`.

## Names

Reuse the same field names across projects:

- `id`, `name`, `value`, `email`
- `items` or `arr` for collections
- `item`, `tmp`
- `dtcreated`, `dtupdated`

SQL and JS should use the same names. See [databases.md](databases.md).

## Files

- Lowercase filenames
- Kebab-case only when a module name is several words: `ai-engine.js`
- Plugin schema names are PascalCase: `NEWSCHEMA('Orders', ...)`
- Public action IDs use `Namespace|action`: `Orders|list`

## Comments

Explain non-obvious constraints, not what the next line does. Do not paste the same `$` field dump into every action.

## Errors and localization

User-facing messages use Total.js localization markup:

```javascript
$.invalid('@(Invalid credentials)');
```

Transport failures use status codes:

```javascript
$.invalid(401);
$.invalid(403);
$.invalid(404);
```
