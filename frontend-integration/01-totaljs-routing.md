# Total.js API Routing — Concepts

## What is Total.js API Routing?

Total.js is a Node.js web framework. Its routing system — called **API Routing** — is architecturally different from conventional REST APIs. Instead of mapping resources to URL paths and HTTP verbs, it exposes **a single HTTP endpoint** and dispatches internally based on a `schema` string embedded in the request body.

This is an **action-based, schema-driven** routing model. If you come from a REST or GraphQL background, this section is the most important thing to read before writing any code.

---

## REST vs Total.js API Routing — side by side

Imagine a "posts" resource. Here is how a REST API and a Total.js API would expose the same operations:

**REST:**
```
GET    /posts           → list posts
GET    /posts/abc123    → read one post
POST   /posts           → create post
PUT    /posts/abc123    → update post
DELETE /posts/abc123    → delete post
```

**Total.js API Routing:**
```
POST /api/   { "schema": "Posts|list" }
POST /api/   { "schema": "Posts|read",   "data": { "id": "abc123" } }
POST /api/   { "schema": "Posts|create", "data": { ... } }
POST /api/   { "schema": "Posts|update", "data": { "id": "abc123", ... } }
POST /api/   { "schema": "Posts|remove", "data": { "id": "abc123" } }
```

| Concern | REST | Total.js API Routing |
|---------|------|----------------------|
| HTTP endpoint count | One per resource | **One for everything** |
| HTTP verbs | GET, POST, PUT, DELETE | **Always POST** |
| Resource identity | In the URL path | In the `data` object |
| Action | In the HTTP verb | In the `schema` string |
| Query params | On the URL | Appended to the `schema` string |
| Request body | Payload only | `{ schema, data }` envelope |

---

## The single endpoint

```
POST https://totaljsbackend.com/api/
Content-Type: application/json
x-token: <session_token>           ← omitted for public schemas

{
  "schema": "<schema_string>",
  "data":   { ... }                ← omitted when not needed
}
```

This is the only URL your client calls for **JSON API work**. File upload/download, health checks, and WebSockets are ordinary HTTP/WS routes and are documented separately. Do not invent a REST surface for CRUD.

The path is project-defined. Many backends use `/api/`. Some register API routing on the root path:

```javascript
ROUTE('API / +account_login --> Auth/login');
ROUTE('+API / -account_logout --> Auth/logout');
```

The client then calls `POST https://api.example.com/` with the same `{ schema, data }` envelope. Make the API path configurable instead of hard-coding `/api/`.

---

## The schema string

The schema string identifies a stable backend action. Input values such as IDs belong in `data`:

```
<Namespace>|<action>
<Namespace>|<action>?key=value&key=value
```

### Part 1 — Resource

The action namespace being addressed: `Posts`, `Users`, `Orders`, `Messages`, etc.

### Part 2 — Action

A verb describing what to do. Common conventions:

| Action name | Meaning |
|---------------|---------|
| `\|list` | Return all records (optionally filtered) |
| `\|read` | Return one record by ID |
| `\|create` | Create a new record |
| `\|insert` | Alias for create (used interchangeably) |
| `\|update` | Update an existing record |
| `\|remove` | Delete a record |
| `\|toggle_<field>` | Toggle a boolean field |
| `\|search` | Free-text or semantic search |
| `\|export` | Export data |
| `\|import` | Import data |

Custom actions beyond CRUD are common and encouraged — they make intent explicit:

```
account_logout
account_password          ← change password
account_verify            ← verify email
session_refresh
notifications_mark_read
```

### Input parameters

Declare required action input on the backend and send it in `data`:

```javascript
NEWACTION('Posts|read', {
  input: '*id',
  route: '+API /api/',
  action: function($, model) {
    // model.id
  }
});
```

```json
{ "schema": "Posts|read", "data": { "id": "abc123" } }
```

### Query parameters (optional)

Appended directly to the schema string — **not** to the HTTP URL:

```json
{ "schema": "Posts|list?page=2&limit=20&status=published" }
{ "schema": "Posts|search?limit=10&mode=semantic", "data": { "query": "..." } }
```

---

## Why this design makes client development easier

**1. One HTTP call setup — done.**

You configure one base URL, one endpoint path, and one auth header. That covers every operation in your application. There is no per-resource client initialization.

**2. One response parser — done.**

Every response uses the same envelope: `{ success, value, error }`. Write one parser and it works everywhere.

**3. One error handler — done.**

Because all calls go through the same endpoint and return the same shape, you handle errors in one place.

**4. No endpoint discovery.**

You do not need to explore or version an endpoint tree. The backend developer tells you the schema names; you call them.

**5. Actions are self-documenting.**

`orders_cancel/ord_123` is unambiguous. `DELETE /orders/ord_123` requires knowledge that DELETE means "cancel" in this context and not "archive" or "refund".

---

## Authentication model

Total.js API Routing uses **session tokens** passed via a custom HTTP header. The backend validates this header at the gateway level before dispatching to any schema handler.

```
x-token: <session_token>
```

- Public schemas (login, register, password reset) do not require this header.
- All protected schemas require it.
- A missing or invalid token results in HTTP `401`.
- The token is an opaque string — do not attempt to decode it client-side.

Do not infer frontend authorization from the `+` or `-` prefix inside a Total.js API route string. Those prefixes are Total.js routing/action metadata and are not a portable public/protected contract.

Frontend auth behavior should be documented separately by the backend team or verified from route middleware and actual responses. Mirror known public schemas in the frontend with an anonymous allowlist so login, registration, discovery, and OTP flows are not affected by old local sessions.

---

## Schema naming conventions summary

```
account_login            public auth
account_logout           protected auth
account                  get current user profile (protected)
account_update           update profile (protected)

Posts|list               list (protected or public depending on backend)
Posts|read               read one; send id in data
Posts|create             create
Posts|update             update; send id in data
Posts|remove             delete; send id in data

Posts|list?page=2        pagination via query param in schema
Posts|search?limit=5     search with filter params
```

Conventions may vary slightly per project. The backend developer defines the schema names — this guide describes the patterns used in a standard Total.js API Routing setup.
