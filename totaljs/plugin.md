# Creating a New Plugin

Plugins are the feature unit. Total.js auto-loads `/plugins/<id>/index.js` into `PLUGINS.<id>` and also auto-loads `plugins/<id>/schemas/*.js`, controllers, and definitions inside the plugin.

There are two common shapes:

1. **API feature plugin** (JSON backends, mobile APIs) — `index.js` registers `ROUTE()`, `schemas/` holds `NEWSCHEMA`. No UI required.
2. **Admin / UI plugin** — same server entry plus `public/` HTML for a Total.js client.

Do not `require()` a plugin schema from another file. Do not turn a plugin into an npm-style package of classes.

## Plugin Structure

```
plugins/
	todo/
		index.js            # metadata + ROUTE() in exports.install()
		schemas/
			todo.js         # NEWSCHEMA('Todo', ...)  (auto-loaded)
		public/             # optional admin/UI assets
			index.html
			form.html
```

## 1. `index.js` (Server-Side plugin definition)

This file serves as the entry point for your plugin on the server. It defines metadata, registers API routes, and sets up permissions. The plugin identifier is the directory name (`plugins/todo` → `PLUGINS.todo`).

```javascript
exports.name = '@(Todo)';
// A plugin name. The markup "@(your_text)" is automatically localized. It's the Total.js localization markup.

exports.icon = 'ti ti-check';
// A plugin icon.

exports.position = 1;
 // A position index in the navigation.

exports.hidden = false;
// Although this property "exports.hidden" can hide the plugin in the menu, but it will still be evaluated on the client side.

exports.import = 'extensions.html';
// Optional. This property can contain a file name from the "plugin/todo/public/" folder. The client-side library imports that file while the web app loads. It's intended for very specific cases, and most plugins don't use it.

exports.visible = function(user) {
	// In this handler, we can check the permissions for accessing this plugin on the client side. It's superior to the "exports.hidden" property.
	// IMPORTANT: The "user.sa {Boolean}" or "user.su {Boolean}" sees everything.
	return user.sa || user.permissions.includes('myitems_view');
};

exports.permissions = [{ id: 'myitems_view', name: 'View My Items' }, { id: 'myitems_edit', name: 'Edit My Items' }];
// Optional permission catalog for a Total.js admin/UI client.

exports.install = function() {
	ROUTE('+API /api/  -todo_list     --> Todo/list');
	ROUTE('+API /api/  +todo_create   --> Todo/create');
	// FILE / SOCKET / upload routes also belong here when they are this feature's
};
```

Call `CORS()` once for the app, not in every plugin.

__Syntax__:

- **`exports.name`** `String`: The display name of the plugin, we use localization markup in the `@(Plugin name)`.
- **`exports.icon`** `String`: Sets the icon displayed in the UI in the form `ti ti-icon_name`.
- **`exports.position`** `Number`: Determines the order in the navigation menu.
- **`exports.visible`** `Function(user_session)`: A function that returns `true` or `false` to control main visibility based on user properties (e.g., `user.sa` for super admin, `user.permissions`).
- **`exports.permissions`** `Object Array`: (Optional) An array of objects defining new permissions that this plugin introduces. Each object should have an `id` and a `name`.
- **`exports.hidden`** `Boolean`: Hides the plugin in the navigation on the client-side.
- **`exports.install`** `Function()`: This function is crucial for setting up special routes using the `ROUTE()`.

## Plugin example

```javascript
exports.name = '@(Todo)';
exports.icon = 'ti ti-check';
exports.position = 1;
exports.visible = user => user.permissions.includes('todo');
exports.permissions = [{ id: 'todo', name: 'ToDo' }];

// Action example
NEWACTION('Todo|list', {
	name: 'List of tasks',
	route: '+API ?',
	action: async function($, model) {

		// $ {Options} Documentation: https://docs.totaljs.com/total5/IbGpBV25x60f/
		// $.query {Object key:value}
		// $.user {Object key:value}
		// $.params {Object key:value}
		// $.model {Object key:value} or model "is prepared according to the input data schema"
		// $.headers {Object key:value}
		// $.ip {String}
		// $.files {Array}
		// $.invalid(error_or_http_status)
		// $.callback({ success: true });
		// $.success(data);
		// $.redirect(url);

		// It creates a list with pagination and filtering and sorting of the fields defined in the .autoquery() method.
		var response = await DATA.list('tbl_todo').autoquery($.query, 'id:String, name:String, body:String, iscompleted:Boolean, createdby:String, updatedby:String, completedby:String, dtcompleted:Date, dtcreated:Date, dtupdated:Date', 'dtcreated_desc', 100).promise($);

		$.callback(response);

	}
});
```

Put `NEWSCHEMA` in `plugins/todo/schemas/todo.js` rather than in `index.js` once the plugin has more than one or two actions. `index.js` stays the route table.

Conditional packs (optional product modules) can skip route registration:

```javascript
exports.install = function() {
	if (!FUNC.feature_enabled('todo'))
		return;
	ROUTE('+API /api/  -todo_list --> Todo/list');
};
```