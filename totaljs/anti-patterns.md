# Anti-patterns

Agents produce these constantly. Do not.

## Sharing code with `require()`

```javascript
// WRONG
const storage = require('../modules/storage');
const mailer = require('../../lib/mailer');
const { can } = require('../services/acl');

// RIGHT
MODS.storage.putFile(...);
FUNC.can($, 'orders.read');
MAIL(address, subject, 'template', model);
```

Internal modules are auto-loaded. Access them through `MODS`. Shared functions go on `FUNC`.

## Node built-in imports inside the app

```javascript
// WRONG
const Fs = require('fs');
const Path = require('path');
const https = require('https');
const crypto = require('crypto');

// RIGHT
PATH.fs.readFileSync(PATH.temp(name));
PATH.join(PATH.public(), 'uploads', name);
PATH.mkdir(PATH.temp('exports'));
Total.Crypto / HASH() / String.sha256()
RESTBuilder.GET(url).promise($);
Total.Https.request(...)   // only if RESTBuilder cannot do it
```

`PATH.verify(dir)` creates the directory if missing (cached). It is **not** a path-traversal sanitizer. Never join user input onto a filesystem path without constraining it first.

## Configuration wrappers

```javascript
// WRONG
const config = require('../config.json');
const settings = require('./env');

// RIGHT
CONF.database
CONF.salt
CONF.allow_register
```

## Database repositories

```javascript
// WRONG
class UserRepository {
	findById(id) { return this.db.query('SELECT ...'); }
}

// RIGHT
var user = await DATA.read('tbl_user').id(id).where('isremoved', false).error(404).promise($);
```

`DATA` is the database API. Views (`view_user`) belong in SQL, not in a JS repository.

## Express response wrappers

```javascript
// WRONG
function sendOk(res, data) { res.json({ ok: true, data }); }
function authMiddleware(req, res, next) { ... }

// RIGHT
$.success(user);
$.callback(items);
$.invalid(401);
AUTH(async function($) { ... });
```

## Re-implementing `AUTH()` on every upload route

If a route is registered without `+` / `-`, Total.js still runs `AUTH()`, but `+GET` / `+POST` is how you require a session. Prefer:

```javascript
ROUTE('+POST /upload/  *  --> upload', ['upload'], 1024 * 10);
```

Do not decrypt tokens again in the handler when `$.user` is already set.

## `setInterval` instead of framework timers

```javascript
// WRONG in definitions
setInterval(tick, 5 * 60 * 1000);

// RIGHT
CRON('*/5 * * * *', function() {
	FUNC.refresh_usage();
});

ON('service', function(counter) {
	if (counter % 5 === 0)
		FUNC.run_jobs();
});
```

`ON('service')` fires about once a minute. `CRON()` is for cron expressions.

## Custom HTTP clients

```javascript
// WRONG
https.request({ hostname: 'api.example.com', ... })

// RIGHT
var response = await RESTBuilder.POST('https://api.example.com/v1', payload)
	.header('Authorization', 'Bearer ' + CONF.api_token)
	.promise($);
```

## `DB()` as the default

`DB()` creates a **new** QueryBuilder controller. `DATA` is the shared singleton.

```javascript
// preferred
await DATA.find('tbl_user').where('isactive', true).promise($);

// only when you need a fresh controller / command batch
var builder = DB();
```

Older docs and some apps use `DB()` everywhere. Do not copy that into new Total.js 5 code.

## `process.env` instead of `CONF`

Boot may read `process.env.PORT` / `NODE_ENV`. Feature code reads `CONF`. If a value must come from the environment, put it in `config` or assign it once in a definition:

```javascript
if (process.env.FEATURE_X != null)
	CONF.feature_x = process.env.FEATURE_X;
```

## God-file `definitions/func.js`

`FUNC` is correct. One 2,000-line file of unrelated domain math is not. Split by concern:

```text
definitions/func.js        # generic helpers (paginate, can, audit)
definitions/func-mail.js   # or assign more FUNC.* from a second definition
```

Definitions are auto-loaded. Multiple files may assign `FUNC.*`.

## In-memory table scans

```javascript
// WRONG
var all = await DATA.find('tbl_order').promise();
var mine = all.filter(w => w.userid === id);

// RIGHT
var mine = await DATA.find('tbl_order').where('userid', id).where('isremoved', false).promise($);
```

Filter in QueryBuilder. Do not load a table to filter it in JavaScript.

## N+1 `DATA.read` loops

```javascript
// WRONG
for (var i = 0; i < ids.length; i++)
	map[ids[i]] = await DATA.read('tbl_user').id(ids[i]).promise();

// RIGHT
var rows = await DATA.find('tbl_user').in('id', ids).promise($);
```

## Ad hoc response envelopes

```javascript
// WRONG
$.json({ ok: true, data: item });
$.json({ success: true, value: item }); // hand-rolled

// RIGHT
$.success(item);     // → { success: true, value: item }
$.callback(item);    // raw payload (lists, query results)
```

## Calling `CORS()` in every plugin

Call `CORS()` once from a definition or default controller.

## Copying another product's domain

Do not recreate another application's tables or schema names. Copy the **mechanism** (plugin + `NEWSCHEMA` + `DATA` + `FUNC`), not the domain.
