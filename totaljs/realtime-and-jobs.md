# Realtime, cron, and workers

Use framework hooks. Do not add `node-cron`, `socket.io`, or `setInterval` job runners.

## WebSockets

```javascript
// controllers/realtime.js
exports.install = function() {
	ROUTE('SOCKET /realtime/', socket);
};

function socket() {
	var self = this;

	MAIN.ws_route = self;

	self.autodestroy(function() {
		if (MAIN.ws_route === self)
			MAIN.ws_route = null;
	});

	self.on('open', function(client) {
		client.$authed = false;
		client.$authTimer = setTimeout(function() {
			if (!client.$authed)
				client.close(4001, 'auth timeout');
		}, 15000);
	});

	self.on('close', function(client) {
		if (client.$authTimer)
			clearTimeout(client.$authTimer);
		FUNC.realtime_unbind(client);
	});

	self.on('message', async function(client, message) {
		var msg = message;
		if (typeof message === 'string') {
			try { msg = JSON.parse(message); } catch (e) { return; }
		}
		if (!msg || !msg.type)
			return;

		if (msg.type === 'auth') {
			var user = await FUNC.realtime_auth(client, msg.token);
			if (!user) {
				client.send(JSON.stringify({ type: 'auth_error' }));
				client.close(4003, 'unauthorized');
				return;
			}
			client.$authed = true;
			clearTimeout(client.$authTimer);
			client.send(JSON.stringify({ type: 'auth_ok', userid: user.id }));
			return;
		}

		if (!client.$authed)
			return;

		if (msg.type === 'ping')
			client.send(JSON.stringify({ type: 'pong', ts: new Date().toISOString() }));
	});
}
```

Keep connections on `MAIN`:

```javascript
MAIN.ws_users = MAIN.ws_users || {};

FUNC.realtime_bind = function(client, userid) {
	if (!MAIN.ws_users[userid])
		MAIN.ws_users[userid] = new Set();
	MAIN.ws_users[userid].add(client);
	client.$userid = userid;
};

FUNC.realtime_unbind = function(client) {
	var userid = client.$userid;
	if (!userid || !MAIN.ws_users[userid])
		return;
	MAIN.ws_users[userid].delete(client);
	if (!MAIN.ws_users[userid].size)
		delete MAIN.ws_users[userid];
};

FUNC.realtime_send = function(userid, event) {
	var set = MAIN.ws_users[userid];
	if (!set)
		return 0;
	var raw = JSON.stringify(event);
	var n = 0;
	set.forEach(function(client) {
		if (client && !client.isclosed) {
			client.send(raw);
			n++;
		}
	});
	return n;
};
```

`$.autodestroy(fn)` / `self.autodestroy(fn)` runs when the last client leaves (after a short delay). Use it to drop process-global references.

Do not add Socket.IO. Dynamic socket routes are just more `ROUTE('SOCKET /path/', handler)` entries.

Client-side: `WEBSOCKETCLIENT` exists on the global if you need an outbound socket.

## `ON('ready')`

Fires when the app is loaded. Safe place to warm caches or start optional connections.

```javascript
ON('ready', function() {
	console.log('Session store: memory');
});
```

If `ON('ready', fn)` is registered after boot, Total.js calls `fn` immediately.

## `ON('service')`

The framework service tick runs about once a minute and emits `service` with a counter.

```javascript
ON('service', function(counter) {
	if (counter % 5 === 0)
		FUNC.refresh_storage_usage();
	if (counter % 30 === 0)
		MAIN.sessions && MAIN.sessions.flush && MAIN.sessions.flush();
});
```

Good uses: expire sessions, refresh counters, light cleanup. Keep it fast.

## `CRON()`

```javascript
CRON('*/5 * * * *', function() {
	FUNC.run_escalations();
});

CRON('0 3 * * *', async function() {
	await FUNC.purge_tmp_files();
});
```

Standard 5-field cron. The returned object has `.remove()`.

Prefer `CRON()` or `ON('service')` over `setInterval`.

## Workers

Put isolated heavy work in `/workers/<name>.js`.

```javascript
var worker = NEWTHREAD('resize');     // worker_threads, file: workers/resize.js
var fork = NEWFORK('import');         // child_process.fork
var pool = NEWTHREADPOOL('resize', 4);
```

`NEWTHREAD()` / `NEWFORK()` without a name attach to the current worker process (`--worker`).

Most backends do not need workers. Prefer a queue table + `CRON()`.

Most backends do not need workers. A queue table + `CRON()` / `ON('service')` is enough:

```javascript
FUNC.process_queue = async function(limit) {
	var jobs = await DATA.find('tbl_job').where('status', 'queued').take(limit || 10).promise();
	for (var i = 0; i < jobs.length; i++) {
		try {
			await FUNC.process_job(jobs[i].id);
		} catch (e) {
			console.error('[job]', jobs[i].id, e.message);
		}
	}
};

CRON('* * * * *', function() {
	FUNC.process_queue(15);
});
```

## Events

```javascript
ON('orders.create', function(order) { ... });
EMIT('orders.create', order);
```

Use events for decoupled side effects (notify, realtime). Do not replace actions with an event bus.

## Multi-instance sockets

In-process `MAIN.ws_users` does not cross Node processes. If you run more than one instance, a small Redis pub/sub bridge inside a definition is a legitimate `require('redis')`. Keep the public API as `FUNC.realtime_send`.
