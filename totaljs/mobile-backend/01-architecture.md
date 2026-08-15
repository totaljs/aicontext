# Architecture And Project Shape

A mobile backend is still a Total.js 5 app. Do not add a second architecture for mobile.

## Reference structure

```text
backend/
  index.js                    # require('total5'); Total.run(...)
  config                      # CONF
  controllers/                # health, upload, FILE, SOCKET
  definitions/                # AUTH, db, FUNC, MAIN, CRON
  modules/                    # integrations → MODS.*
  plugins/<feature>/
    index.js                  # ROUTE()
    schemas/*.js              # NEWSCHEMA actions
  public/
```

Boot stays tiny:

```javascript
require('total5');
Total.run({ port: 8000 });
```

## Responsibility boundaries

| Layer | Responsibility |
|-------|----------------|
| `controllers/` | Health, multipart upload, file download, WebSocket, SSO redirects |
| `definitions/auth.js` | `AUTH()`, token parsing |
| `definitions/db.js` | `querybuilderpg` → `DATA` |
| `definitions/func.js` | `FUNC.*` |
| `modules/` | External SDKs; callers use `MODS.<name>` |
| `plugins/<feature>/index.js` | Route table |
| `plugins/<feature>/schemas/` | Validation, SQL, business rules |

Do not put feature rules in the mobile app. If a user may publish only what they own, that check is a backend action.

## Naming

- CommonJS files, Total.js globals, no internal `require()`
- tabs and semicolons
- PascalCase schema names: `NEWSCHEMA('Orders', ...)`
- snake_case public schemas: `orders_list`

```javascript
// WRONG
const mailer = require('../modules/mailer');

// RIGHT
MODS.mailer.send(...);
// or MAIL()
```

## What transfers to the next project

- API Routing with `{ schema, data }`
- plugin-owned routes
- `NEWSCHEMA` per domain
- `AUTH()` + encrypted session tokens
- documented public schema list
- `DATA.list().autoquery()`
- upload as a normal HTTP route
- `FUNC` / `MAIN` / `MODS` for sharing

Table names and product nouns do not transfer.
