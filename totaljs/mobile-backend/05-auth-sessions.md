# Authentication, Sessions, And Identity

See also [../auth.md](../auth.md). This page is the mobile contract.

## Headers

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

Clients send `x-token`. They never decode it.

## Login response

```javascript
$.success({ token: result.token, user: result.user });
```

`user` is a safe snapshot: id, name, email/phone, roles, permissions, language, photo. Never password hashes, reset tokens, or integration secrets.

Create the token with `ENCRYPTREQ($, { id: session.id }, CONF.salt)` and store the session with `DATA` + `MAIN.sessions`.

## `AUTH()`

One delegate. Path-based identity splits are optional and only justified when those are actually different principals. Prefer one user and permissions.

```javascript
AUTH(async function($) {
	var token = FUNC.read_header_token($);
	if (!token) {
		$.invalid();
		return;
	}
	var data = DECRYPTREQ($, token, CONF.salt);
	if (!data || !data.id) {
		$.invalid();
		return;
	}
	var session = await MAIN.sessions.get(data.id) || await DATA.read('tbl_session').id(data.id).promise();
	if (!session) {
		$.invalid();
		return;
	}
	var user = await FUNC.refresh_session(session);
	user ? $.success(user) : $.invalid();
});
```

## Public vs optional auth

Public discovery uses `-API`. Optional personalization uses unprefixed `API` and reads `$.user` if present. A failed optional token must not become 401.

## Logout

Delete the server session and cache entry. Return success even if the client will drop the token anyway.

```javascript
await MAIN.sessions.delete(sessionid);
await DATA.remove('tbl_session').id(sessionid).promise();
$.success(true);
```

## Authorization

Authentication is identity. Authorization is `FUNC.can` / `FUNC.require_perm` / ownership queries. Hide a screen in the app if you want; the backend still decides.
