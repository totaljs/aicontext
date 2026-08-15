# Files, Uploads, Media, And Devices

Uploads are ordinary HTTP routes. See [../filesystem.md](../filesystem.md).

## Upload

```javascript
exports.install = function() {
	ROUTE('+POST /upload/', upload, ['upload'], 1024 * 5);
	ROUTE('FILE /download/*.*', files);
};

async function upload($) {
	var output = [];
	for (var i = 0; i < $.files.length; i++) {
		var file = $.files[i];
		var response = await file.fs('files', UID());
		response.url = FUNC.media_url($, '/download/' + response.id.sign(CONF.salt) + '.' + response.ext);
		output.push(response);
	}
	$.json(output);
}
```

If storage is S3 or another object store, put the SDK in `/modules/storage.js` and call `MODS.storage`.

## Download

Sign ids. Do not return filesystem paths.

```javascript
function files($) {
	var name = ($.split && $.split[1]) || '';
	var index = name.lastIndexOf('.');
	var hash = index === -1 ? name : name.substring(0, index);
	var dash = hash.indexOf('-', 10);
	var id = dash === -1 ? '' : hash.substring(0, dash);
	if (id && hash === id.sign(CONF.salt))
		$.filefs('files', id);
	else
		$.invalid(404);
}
```

## Absolute URLs

Normalize on the server with `FUNC.media_url($, url)`. Mobile clients should not concatenate hosts for every image.

## Native devices

`localhost` on a phone is the phone. Document LAN IP, tunnel, or staging URL. CORS matters for Expo web and dashboards, not for native HTTP.

```javascript
CORS(); // or an explicit origin list, once
```

## Upload auth

`+POST /upload/` so `AUTH()` fills `$.user`. Do not re-implement token decryption in the upload handler. Do not put admin API secrets in the mobile bundle.

A dedicated upload token is acceptable only if it is scoped to upload and documented as such.
