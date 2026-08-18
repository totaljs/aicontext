# Authentication and authorization

Total.js authorization is one delegate: `AUTH()`. It runs on every request except static files.

## `AUTH()`

```javascript
// definitions/auth.js
AUTH(async function($) {

	var token = FUNC.read_header_token($);
	if (!token || token.length < 20) {
		$.invalid();
		return;
	}

	var data = null;
	try {
		data = DECRYPTREQ($, token, CONF.salt);
	} catch (e) {
		data = null;
	}

	if (!data || !data.id) {
		$.invalid();
		return;
	}

	var session = await MAIN.sessions.get(data.id);
	if (!session)
		session = await DATA.read('tbl_session').id(data.id).promise();

	if (!session || (session.expires && NOW > new Date(session.expires))) {
		$.invalid();
		return;
	}

	var user = await FUNC.refresh_session(session);
	if (!user) {
		$.invalid();
		return;
	}

	$.success(user);
});
```

- `$.success(user)` → `$.user` is set; `+` routes match
- `$.invalid()` → unauthorized; `-` routes match
- No call / both → flags `API` / `GET` without `+`/`-` still run

Use `$.invalid(401)` when you want an explicit status from an **action**. Inside `AUTH()`, `$.invalid()` is enough to mark the request unauthorized.

## Token headers

Accept a small compatibility set. Prefer `x-token`.

```javascript
FUNC.read_header_token = function($) {
	var headers = $.headers || EMPTYOBJECT;
	var token = headers['x-token'] || headers.token || '';
	var auth = headers.authorization || '';
	if (!token && auth && auth.substring(0, 7).toLowerCase() === 'bearer ')
		token = auth.substring(7).trim();
	return token || '';
};
```

Tokens are opaque. Encrypt the session id with `ENCRYPTREQ($, { id: session.id }, CONF.salt)`. Do not put roles inside a JWT that the client can read and the server trusts blindly.

## Sessions

Create in `FUNC`, cache on `MAIN`, persist in `DATA`:

```javascript
FUNC.create_session = async function($, userid) {
	var session = {
		id: UID(),
		userid: userid,
		dtcreated: NOW,
		ip: $.ip || null,
		expires: NOW.add(CONF.cookie_expires || '14 days')
	};
	session.token = ENCRYPTREQ($, { id: session.id }, CONF.salt);
	await DATA.insert('tbl_session', session).promise();

	var snapshot = await FUNC.refresh_session(session);
	if (snapshot)
		await MAIN.sessions.set(session.id, snapshot);

	return { token: session.token, user: FUNC.user_safe(snapshot) };
};
```

Login action:

```javascript
schema.action('login', {
	input: '*email:Email,*password:String',
	action: async function($, model) {
		var user = await DATA.read('tbl_user')
			.where('email', model.email.toLowerCase().trim())
			.where('password', FUNC.hash_password(model.password))
			.where('isremoved', false)
			.promise();

		if (!user) {
			$.invalid('@(Invalid credentials)');
			return;
		}

		var result = await FUNC.create_session($, user.id);
		$.success(result);
	}
});
```

`FUNC.user_safe(user)` must strip password hashes, reset tokens, and secrets.

## Route flags

```javascript
ROUTE('-API /api/  +auth_login   --> Auth/login');   // public
ROUTE('+API /api/  -auth_me      --> Auth/me');      // session required
ROUTE('+API /api/  +auth_logout  --> Auth/logout');
```

Public login/register stay on `-API`. Protected work stays on `+API`.

Optional personalization (public list that can use a user if present) uses an unprefixed `API` route and reads `$.user` if `AUTH()` succeeded.

## Permissions

`AUTH()` answers who. Actions answer whether.

```javascript
if (!FUNC.require_perm($, 'orders.manage'))
	return;
```

`NEWACTION` / schema options also support:

```javascript
NEWACTION('Orders|remove', {
	permissions: 'orders.manage',
	sa: true,          // requires $.user.sa or $.user.su
	user: true,        // requires $.user
	route: '+API /api/',
	action: async function($) { ... }
});
```

`permissions` calls `UNAUTHORIZED($, permissions)`. Custom RBAC beyond that belongs in `FUNC.can`.

## API keys

If you accept `x-api-key`, resolve it **inside** `AUTH()`, not in a second middleware package. Map the key to the same session shape (`id`, `permissions`, `auth_source: 'api'`).

## WebSocket auth

`AUTH()` does not apply to sockets the same way. Authenticate on the first message and store the user on the client. See [realtime-and-jobs.md](realtime-and-jobs.md).

## Do not

- Implement Passport / jsonwebtoken middleware
- Decode tokens in every controller
- Store the password hash on `$.user`
- Trust the client’s `role` field
- Split identity only by URL path unless you truly have separate apps (admin vs customer). Prefer one user with roles/permissions.
