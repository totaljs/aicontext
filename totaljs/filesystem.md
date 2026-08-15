# Filesystem, PATH, and uploads

Total.js already wraps Node’s filesystem and path APIs. Application code should not `require('fs')` or `require('path')`.

## `PATH`

`PATH` is `Total.path` / `F.path`.

### Directory helpers

Each helper returns the directory, or joins a relative piece onto it:

```javascript
PATH.root()
PATH.root('package.json')

PATH.private()
PATH.public()
PATH.databases()
PATH.logs()
PATH.temp()          // alias: PATH.tmp()
PATH.modules()
PATH.plugins()
PATH.scripts()
PATH.templates()
```

```javascript
var dest = PATH.public('uploads/' + id + '.jpg');
var log = PATH.logs('audit.log');
```

### Join, exists, mkdir, unlink

```javascript
PATH.join(a, b, c);            // Total.Path.join
PATH.exists(filename);         // Promise → callback(exists, size, isFile)
PATH.exists(filename, function(exists, size, isFile) { ... });
PATH.mkdir(directory);         // recursive, sync
PATH.unlink(fileOrArray);
PATH.rmdir(dirOrArray);
```

`PATH.fs` is Node `fs` (`Total.Fs`). Use it when you need a raw fs method:

```javascript
PATH.fs.readFileSync(PATH.temp(name));
PATH.fs.writeFileSync(filename, buffer);
```

### `PATH.verify(directory)`

Ensures the directory exists (`mkdir` recursive) and remembers that it already did.

```javascript
PATH.verify(PATH.temp('exports'));
```

It is **not** a security check. It does not sanitize `../` or user-controlled segments.

When a path includes user or request data:

1. Generate the filename yourself (`UID()` + allowed extension)
2. Join it onto a known root (`PATH.public('uploads')`, `PATH.temp()`)
3. Reject absolute input and `..`
4. Then write

```javascript
var ext = (file.filename || 'bin').split('.').pop().toLowerCase();
if (['png', 'jpg', 'jpeg', 'webp', 'pdf'].indexOf(ext) === -1) {
	$.invalid('@(Unsupported file type)');
	return;
}
var filename = PATH.public('uploads/' + UID() + '.' + ext);
```

## Node built-ins on `Total`

Total.js 5 attaches Node core modules on the framework instance:

```javascript
Total.Fs        // node:fs        also PATH.fs
Total.Path      // node:path      also PATH.join
Total.Http      // node:http
Total.Https     // node:https
Total.Crypto    // node:crypto
Total.Stream    // node:stream
Total.Util      // node:util (not Total.js helpers — those are U / Utils)
Total.Os        // node:os
Total.Url       // node:url
Total.Zlib      // node:zlib
Total.Net
Total.Dns
Total.Tls
Total.Child     // child_process
Total.Worker    // worker_threads
```

Prefer `PATH.*` and `RESTBuilder` over reaching for these. Use `Total.Https` only when `RESTBuilder` cannot express the call.

Hashes and ids:

```javascript
HASH('value', 'sha256');
String(password).sha256(CONF.passwordizator);
UID();
GUID();
GUID(24);
```

## FileStorage

Total.js includes `FILESTORAGE(name)` / `$.filefs(name, id)` for framework-native blob storage (not S3).

```javascript
// store an upload
var meta = await file.fs('files', UID());

// send it back
$.filefs('files', id);
```

Use this when you want Total.js-managed files on disk. Use a `/modules/storage.js` module when you need S3 or another object store. Call it through `MODS.storage`, not `require()`.

## Upload routes

Multipart upload is a normal HTTP route, not an API Routing envelope.

```javascript
exports.install = function() {
	ROUTE('+POST /upload/', upload, ['upload'], 1024 * 10);
	ROUTE('GET /files/{id}/', download);
	ROUTE('FILE /download/*.*', files);
};

async function upload($) {
	var file = $.files && $.files[0];
	if (!file) {
		$.invalid('@(No file uploaded)');
		return;
	}

	if (!FUNC.can($, 'files.upload')) {
		$.invalid(403);
		return;
	}

	var id = UID();
	var buffer = await file.read();
	await MODS.storage.putFile('photos/' + id + '.jpg', buffer, file.type);

	await DATA.insert('tbl_file', {
		id: id,
		filename: file.filename,
		mime: file.type,
		size: file.size,
		ownerid: $.user.id,
		dtcreated: NOW
	}).promise($);

	$.success({ id: id, filename: file.filename });
}

async function download($) {
	var row = await DATA.read('tbl_file').id($.params.id).error(404).promise($);
	var obj = await MODS.storage.getObjectStream(row.path);
	if (obj.localPath)
		$.file(obj.localPath, row.filename);
	else
		$.stream(row.mime, obj.stream, row.filename);
}
```

Keep metadata in PostgreSQL. Keep bytes in storage. Return ids and URLs, never raw server paths.

## Signed download IDs

If you serve files from `FILE /download/*.*`, sign the id so clients cannot guess storage keys:

```javascript
var token = id.sign(CONF.salt);
// url: /download/{token}.jpg

function files($) {
	var name = $.split[1] || '';
	var index = name.lastIndexOf('.');
	var hash = index === -1 ? name : name.substring(0, index);
	var id = hash.substring(0, hash.indexOf('-', 10));
	if (hash === id.sign(CONF.salt))
		$.filefs('files', id);
	else
		$.invalid(404);
}
```

## Absolute URLs for mobile

Normalize media on the server:

```javascript
FUNC.media_url = function($, url) {
	if (!url)
		return url;
	if (url.indexOf('http') === 0)
		return url;
	var origin = CONF.url || (($.headers && ($.headers['x-forwarded-proto'] || 'http')) + '://' + $.headers.host);
	return origin.replace(/\/$/, '') + (url[0] === '/' ? url : ('/' + url));
};
```

Do not make every client concatenate hosts.

## Scripts vs runtime

`scripts/migrate.js` running as `node scripts/migrate.js` is not inside `Total.run()`. `require('fs')` is acceptable there. The same code copied into a controller is not.
