# Total.js v5 Globals

Style rules: [style.md](style.md). Architecture: [architecture.md](architecture.md).

Total.js is a global/singleton framework. These names exist after `require('total5')`. Do not import them.

## Map

| Global | Purpose |
|--------|---------|
| `Total` / `F` | Framework instance. Prefer `Total`. `F` is the legacy alias. |
| `CONF` | Config from `config` / `config-debug` / `config-release` |
| `FUNC` | Application functions you assign |
| `MAIN` | Application runtime data you assign |
| `MODS` | Auto-loaded `/modules/*.js` |
| `PLUGINS` | Auto-loaded `/plugins/<id>` |
| `DATA` | Shared QueryBuilder. **Use this for SQL.** |
| `DB()` | New QueryBuilder controller. Prefer `DATA`. |
| `PATH` | Application directories and fs helpers |
| `CACHE` | In-memory TTL cache |
| `DEF` | Framework defaults (`onSuccess`, CSRF, …) |
| `DEBUG` | `process.env.NODE_ENV !== 'production'` |
| `NOW` | Current `Date`, refreshed by the service timer |
| `REPO` | Framework bag (UI/admin). Not your session store. |
| `TEMP` | Scratch object. **Wiped about every 5 minutes.** Not for sessions. |
| `U` / `Utils` | Total.js utility helpers (`F.TUtils`). Not `Total.Util` (that is `node:util`). |
| `EMPTYOBJECT` / `EMPTYARRAY` | Frozen empties |
| `RESTBuilder` | Outbound HTTP |
| `ErrorBuilder` | Structured errors |
| `AUTH` | Authorization delegate |
| `ROUTE` | Register HTTP / API / FILE / SOCKET |
| `CORS` | CORS origins |
| `NEWACTION` / `NEWSCHEMA` / `ACTION` | Actions |
| `CRON` | Cron jobs |
| `ON` / `ONCE` / `OFF` / `EMIT` | Events |
| `UID` / `GUID` / `HASH` | Identifiers and hashes |
| `ENCRYPT` / `DECRYPT` / `ENCRYPTREQ` / `DECRYPTREQ` | Crypto helpers |
| `MAIL` / `HTMLMAIL` / `LOGMAIL` | Mail |
| `FILESTORAGE` | Native file storage |
| `UNAUTHORIZED` | Permission helper used by action `permissions` |
| `BLOCKED` | Simple IP rate limit |
| `ERROR` | Error logger factory (`ERROR('DB')`) |
| `MIDDLEWARE` | Named middleware |
| `TRANSFORM` / `NEWTRANSFORM` | Value transforms |
| `PROXY` | HTTP proxy routes |
| `WEBSOCKETCLIENT` | Outbound websocket |
| `NEWTHREAD` / `NEWFORK` / `NEWTHREADPOOL` | Workers under `/workers/<name>.js` |
| `SUCCESS` | `DEF.onSuccess(value)` |

Details for `FUNC` / `MAIN` / `MODS`: [modules-and-definitions.md](modules-and-definitions.md).
Details for `PATH`: [filesystem.md](filesystem.md).
Details for `ROUTE` / `$`: [controllers-and-routing.md](controllers-and-routing.md).

## HASH(value, type);

Creates the hash from the string. Supports: sha1, sha256, sha512, md5 or crc32.

```javascript
HASH(text, [type]);
// returns {String} hashed value

let sha256 = HASH('my-secret-value', 'sha256');
let md5 = HASH('my-secret-value', 'md5');
let crc32 = HASH('my-secret-value', 'crc32');
```

Passwords in Total.js apps usually use the string helper plus a salt from `CONF`:

```javascript
FUNC.hash_password = function(password) {
	return String(password).sha256(CONF.passwordizator);
};
```

## GUID([length]);

- `GUID()` — a real 128-bit GUID string
- `GUID(n)` — random hash of length `n`

```javascript
GUID();      // 128-bit GUID
GUID(10);    // 10 character hash
GUID(40);    // useful for API keys
```

## UID();

Creates a unique identifier at least 12 characters long. Contains a timestamp, counter, and checksum. Use `UID()` for table primary keys.

```javascript
UID();
// returns {String}
```

## NOW;

Current date/time. Refreshed by the framework service timer (about every 5 seconds internally; the public `NOW` is updated on that cadence). Designed to reduce `new Date()` churn.

```javascript
NOW;
// returns {Date}

NOW.add('14 days');
NOW.add('15 minutes');
```

For “right now” in a security comparison, `new Date()` is still acceptable. For `dtcreated` / `dtupdated`, `NOW` is the Total.js convention.

## CONF;

Key/value from `config` files. Keys become lowercase identifiers.

```text
database          : postgresql://user:pass@127.0.0.1:5432/app
salt              : change-me
cookie_expires    : 14 days
allow_register    : true
```

```javascript
CONF.database
CONF.salt
CONF.allow_register
```

Booleans may arrive as `true` or `'true'`. Compare both when reading flags.

Typed config (framework-native, prefer this over a parallel env layer):

```text
allow_register (boolean) : true
api_key (env)            : MY_API_KEY
```

`(env)` copies `process.env.MY_API_KEY` into `CONF.api_key`.

## DATA;

Shared QueryBuilder. After `querybuilderpg` init, this is how you talk to PostgreSQL.

```javascript
// definitions/db.js — legitimate require of an npm package
require('querybuilderpg').init('', CONF.database, 1, ERROR('DB'));
```

Empty name registers the default connection used by `DATA`.

`DATA` is a singleton controller (`new Controller(true)`). Each call executes immediately.

`DB()` returns a **new** controller with a command queue. Prefer `DATA` in actions.

### Methods

Each method returns a QueryBuilder.

```javascript
DATA.find('table_name');
DATA.list('table_name');   // paginated { items, count }
DATA.read('table_name');   // one row
DATA.insert('table_name', { name: 'Created', dtcreated: NOW });
DATA.update('table_name', { name: 'Updated', dtupdated: NOW });
DATA.modify('table_name', payload); // alias of update
DATA.remove('table_name');
DATA.check('table_name');
DATA.count('table_name');
DATA.query('SELECT * FROM tbl_user WHERE id=$1', [id]);
DATA.scalar('table_name', 'avg', 'age');
```

### QueryBuilder methods

Filters combine with **AND** unless you use `.or()`.

```javascript
var builder = DATA.find('tbl_user');

builder.fields('id,name,dtcreated');
builder.id(value);                         // id = value
builder.userid(value);                     // userid = value
builder.where(column, value);
builder.where(column, '>', value);         // = > < >= <= <>
builder.between(column, a, b);
builder.in(column, array);
builder.search(column, value, [operator]); // ILIKE; operator: '*', 'beg', 'end'
builder.query('isremoved=FALSE');          // raw fragment — trusted SQL only
builder.sort('dtcreated', true);           // true / 'desc' → DESC
builder.take(50);
builder.skip(50);
builder.paginate(page, limit, maxlimit);
builder.or(function() {
	this.where('name', 'Peter');
	this.where('name', 'Anna');
});
builder.error(404);                        // error if empty / null
builder.error('@(Already exists)', true);  // reverse: error if exists
builder.autoquery(query, schema, default_sort, default_maxlimit);
builder.callback(function(err, response) { ... });
builder.promise($);                        // Promise; errors go to $.invalid
```

`autoquery()` reads `query` (`$.query`) and builds filters/sort/limit from a schema string such as `'name:String,age:Number,dtcreated:Date'`.

```javascript
var response = await DATA.list('tbl_user')
	.where('isremoved', false)
	.autoquery($.query, 'id:String,name:String,email:String,dtcreated:Date', 'dtcreated_desc', 100)
	.promise($);
```

### Examples

```javascript
DATA.find('tbl_user').where('isremoved', false).between('age', 20, 30).callback(function(err, response) {
	console.log(err, response);
});

var user = await DATA.read('tbl_user').id($.params.id).error(404).promise($);
```

Parameterized raw SQL:

```javascript
var rows = await DATA.query('SELECT id, name FROM tbl_user WHERE email=$1', [email]).promise($);
```

Never concatenate user input into SQL. More conventions: [databases.md](databases.md).

## RESTBuilder;

Outbound HTTP. Prefer this over `require('http')` / `require('https')`.

```javascript
RESTBuilder.POST(url, [payload]);
RESTBuilder.GET(url);
RESTBuilder.PUT(url, [payload]);
RESTBuilder.DELETE(url, [payload]);
RESTBuilder.PATCH(url, [payload]);
RESTBuilder.API(url, action, [payload]); // { schema: action, data: payload }
```

### RESTBuilderInstance

```javascript
var builder = RESTBuilder.GET('https://www.totaljs.com');

builder.keepalive();
builder.insecure();
builder.noparse();
builder.xhr();
builder.header(name, value);
builder.auth(user_or_token, [password]);
builder.urlencoded([payload]);
builder.timeout(timeout);          // ms, default 5000
builder.file(name, filename, [buffer]);
builder.cookie(name, value);
builder.callback(callback);
builder.promise($);
builder.stream(callback);
```

```javascript
RESTBuilder.GET('https://www.totaljs.com').xhr().callback(function(err, response) {
	console.log(err, response);
});

var profile = await RESTBuilder.GET(url)
	.header('Authorization', 'Bearer ' + token)
	.promise($);
```

## AUTH();

One delegate. See [auth.md](auth.md).

```javascript
AUTH(function($) {
	if ($.headers['x-token'] === '123456')
		$.success({ name: 'Token', sa: true });
	else
		$.invalid();
});
```

- `$.success(user)` → `+` routes
- `$.invalid()` → `-` routes

## ENCRYPTREQ / DECRYPTREQ

Bind a payload to the request (user-agent, optionally IP) and encrypt it with `CONF.salt`.

```javascript
var token = ENCRYPTREQ($, { id: session.id }, CONF.salt);
var data = DECRYPTREQ($, token, CONF.salt);
```

This is the usual session token. Do not introduce `jsonwebtoken` for the same job.

## CACHE

```javascript
CACHE.set('key', value, '10 minutes');
var value = CACHE.get('key');
CACHE.remove('key');
CACHE.reset('prefix');
```

## BLOCKED / ERROR

```javascript
if (BLOCKED($, 5, '15 minutes')) {
	$.invalid('@(Too many attempts)');
	return;
}

require('querybuilderpg').init('', CONF.database, 1, ERROR('DB'));
```

## PATH and Total.*

See [filesystem.md](filesystem.md). Short form:

```javascript
PATH.root()
PATH.public('uploads/a.jpg')
PATH.temp()
PATH.logs()
PATH.databases()
PATH.modules()
PATH.plugins()
PATH.join(a, b)
PATH.exists(filename)
PATH.mkdir(dir)
PATH.unlink(file)
PATH.verify(dir)     // ensure directory exists — not a sanitizer
PATH.fs              // node:fs

Total.Fs
Total.Path
Total.Http
Total.Https
Total.Crypto
Total.Stream
Total.Util          // node:util — helpers are U / Utils
Total.Os            // node:os
```

## Recommended field names

- `id`, `name`, `value`, `email`
- `items` / `arr`, `item`, `tmp`
- `dtcreated`, `dtupdated`
