# Total.js Mobile App Backend Guide

How to expose a Total.js 5 backend to a mobile app **without turning the backend into Express**.

The backend remains a Total.js application: plugins, `NEWSCHEMA` / `NEWACTION`, `DATA`, `FUNC`, `AUTH()`, `CONF`. The mobile client sees one HTTP envelope.

Read [architecture.md](architecture.md) first.

## Guide map

1. [Architecture And Project Shape](mobile-backend/01-architecture.md)
2. [API Routing Contract](mobile-backend/02-api-routing.md)
3. [Plugins And Route Registration](mobile-backend/03-plugins-routes.md)
4. [Schemas, Actions, And Validation](mobile-backend/04-schemas-actions.md)
5. [Authentication, Sessions, And Identity](mobile-backend/05-auth-sessions.md)
6. [Data Access, Lists, And Responses](mobile-backend/06-data-responses.md)
7. [Files, Uploads, Media, And Devices](mobile-backend/07-files-media-devices.md)
8. [Operations, Jobs, Integrations, And Runtime Hooks](mobile-backend/08-operations-runtime.md)
9. [Mobile Feature Checklist](mobile-backend/09-feature-checklist.md)

## Core contract

```http
POST /api/
Content-Type: application/json
x-token: <session_token>

{
  "schema": "products_list?page=1&limit=20",
  "data": { "optional": "payload" }
}
```

Some apps mount API Routing at `/` instead of `/api/`. The client must configure the path.

Guarantees:

- schema names are stable and action-oriented
- public vs protected is explicit (`-API` / `+API`)
- login/register can return `{ token, user }`
- lists are arrays or `{ items, count, page, limit }`
- errors go through `$.invalid()`
- ownership is enforced in actions, not in the app
- upload/download stay on ordinary HTTP routes

## Companion guides

- Frontend: [React Native Integration Guide](../frontend-integration/07-react-native-integration.md)
- Framework: [Actions](actions.md), [Plugins](plugin.md), [Databases](databases.md), [Globals](globals.md), [Auth](auth.md)
