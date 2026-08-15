# Mobile Feature Checklist

## Design

- Feature lives in one plugin
- Schema names are stable and readable
- Public vs protected is explicit
- Mobile-specific schemas exist only when the contract differs
- Old app builds still have a path

## Routes

- Registered in `plugins/<feature>/index.js`
- Public: `-API`
- Protected: `+API`
- Ownership in the action (or a composed check)
- HTTP routes only for files, webhooks, health, sockets

## Actions

- `NEWSCHEMA` in `plugins/<feature>/schemas/`
- `input` / `query` / `params` declared
- `$.invalid()` / `$.success()` / `$.callback()`
- Shared helpers on `FUNC` or `MODS`, not `require()`

## Auth

- Public schemas documented
- Protected schemas 401 without a token
- Server-side ownership
- Optional personalization does not 401
- Logout clears `MAIN` + `DATA` session

## Data

- `DATA`, not repositories
- List shape documented
- Empty list is not an error
- Missing row is 404
- Public lists hide removed/unpublished rows
- Secrets never in payloads
- Media URLs absolute or documented

## Files

- Upload route and size limit documented
- Download ids signed or authorized
- No raw filesystem paths

## Operations

- Slow work in `CRON()` / `ON('service')` / a job table
- Cache invalidation on update/logout
- Diagnostic routes protected

## Verification

```bash
node --check path/to/edited-file.js
```

Exercise the schema with a fixture or HTTP call. If a mobile client changed, type-check that client too.
