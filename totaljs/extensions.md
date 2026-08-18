# Framework extension

`Total` is the current global framework instance. `F` is the older alias (`global.F = global.Total`). Prefer `Total` in new examples. Both work.

## `Total.extend()`

Adds a method to a framework prototype.

```javascript
Total.extend('$', 'mail', function(address, subject) {
	var $ = this;
	return MAIL(address, subject, 'mail/notif', { user: $.user });
});

// later in an action
$.mail(user.email, '@(Welcome)');
```

Supported first arguments (from Total.js 5 source):

| proto | Target |
|-------|--------|
| `'$'` / `'options'` | Action/controller options (`$`) |
| `'controller'` | HTTP and WebSocket controllers |
| `'error'` / `'errorbuilder'` | `ErrorBuilder` |
| `'restbuilder'` | `RESTBuilder` |
| `'mail'` / `'email'` | Mail message |
| `'view'` | View engine |
| `'flowstream'` / `'message'` | FlowStream |
| `'querybuilder'` / `'database'` | QueryBuilder (see caveat) |

Use this when many actions need the same **context** helper (`this` is `$` or a controller).

Do not use it for ordinary functions:

```javascript
// WRONG — not a controller concern
Total.extend('$', 'hashPassword', function(pw) { ... });

// RIGHT
FUNC.hash_password = function(pw) {
	return String(pw).sha256(CONF.passwordizator);
};
```

Do not invent a utility module of prototype patches. Two or three extensions are plenty.

### Caveat

In total5 `0.0.17`, `Total.extend('querybuilder', ...)` references `T.TQueryBuilder` in source. Prefer `FUNC` helpers that take a builder over extending QueryBuilder until you have verified the extend target in your installed version.

## `DEF`

Framework defaults. Already populated. Override rarely.

```javascript
DEF.onSuccess = function(value) {
	return { success: true, value: value };
};
```

Changing `DEF.onSuccess` changes `$.success()` globally. Do not replace it with `{ ok: true, data }` unless the whole product agrees.

Other hooks exist for CSRF, localization, and similar. Read `index.js` around `DEF.` before overriding.

## `MIDDLEWARE()`

Total.js has `MIDDLEWARE(name, fn)` and route flags `#name`. Use it for rare cross-cutting HTTP concerns that `AUTH()` and actions cannot express.

Do not rebuild Express middleware. Auth, permissions, and validation already have framework homes.

## Prototype extensions vs `FUNC` vs modules vs actions

| Need | Mechanism |
|------|-----------|
| Pure helper | `FUNC.x = function(...)` |
| Integration | `/modules/x.js` → `MODS.x` |
| Request/response sugar | `Total.extend('$', 'x', fn)` |
| Domain operation | `NEWSCHEMA` / `NEWACTION` |
| Boot / auth / db | `definitions/` |

When unsure, use `FUNC`.
