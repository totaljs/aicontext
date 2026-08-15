# PostgreSQL + Total.js guidelines

Application code talks to PostgreSQL through **`DATA`** (QueryBuilder) after `querybuilderpg` is initialized. Do not add repositories, ORMs, or `require('pg')` inside actions.

```javascript
// definitions/db.js
require('querybuilderpg').init('', CONF.database, 1, ERROR('DB'));
```

```javascript
var user = await DATA.read('tbl_user').id(id).where('isremoved', false).error(404).promise($);
await DATA.insert('tbl_user', model).promise($);
await DATA.modify('tbl_user', { name: model.name, dtupdated: NOW }).id(id).error(404).promise($);
var list = await DATA.list('view_user')
	.autoquery($.query, 'id:String,name:String,dtcreated:Date', 'dtcreated_desc', 100)
	.promise($);
```

`DATA.list` returns `{ items, count }`. `DATA.find` returns an array. Use views for denormalized mobile/admin lists.

Parameterized SQL when QueryBuilder is not enough:

```javascript
var rows = await DATA.query('SELECT id, name FROM tbl_user WHERE email=$1', [email]).promise($);
```

Never concatenate request values into SQL. `builder.query('isremoved=FALSE')` is for trusted fragments only.

`DB()` creates a new QueryBuilder controller. Prefer `DATA`. See [globals.md](globals.md).

---

## SQL style

- __table names / field names should be all lowercase__
- use `tabs` (tab width `4`) instead of `spaces`
- remove all unnecessary white-spaces
- always use `;` semicolon at the end of a SQL command
- remove all unnecessary type-casting `name<>'something'::text` to `name<>'something'`
- format SQL scripts
- divide scripts TABLES, VIEWS, STORED PROCEDURES/FUNCTIONS, INDEXES
- table names starts with `tbl_` in singular --> `tbl_user`, `tbl_product`
- tables with codelists must start with `cl_` in singular --> `cl_type`, `cl_product`
- view names starts with `view_` in singular --> `view_user`
- stored procedures starts with `sp_` in singular --> `sp_user`, etc..
- functions starts with `fn_` in singular --> `fn_user`, etc..

__Fields creating__:

- dates starts with `dt` --> `dtcreated`, `dtupdated`, `dtpaid`, and always in UTC format
- booleans must starts with `is` --> `ispaid`, `isremoved`, `ispublished`, etc.. with few exceptions
- identifiers `id` (primary key), `userid` (foreign key), `productid` (foreign key), etc..
- use `text` data types instead of `varchar` ([more info](https://stackoverflow.com/questions/4848964/difference-between-text-and-varchar-character-varying))
- keep same names in different tables for example: `name`, `body`, etc..
- keep short names
- if the situation allows it, use always these columns `id`, `name` and `dtcreated`

__Keep the sort of fields below__:

```javascript
CREATE TABLE "public"."tbl_channel_message" (

    -- IDENTIFIERS FIRST
    "id" text NOT NULL,
    "userid" text,
    "channelid" text,
    "openplatformid" text,

    -- MAIN FIELDS
    "body" text,

    -- NUMBERS
    "countupdate" INT2 DEFAULT 0,
    "countviews" INT2 DEFAULT 0,

    -- BOOLEANS
    "isrobot" bool DEFAULT FALSE,
    "ismobile" bool DEFAULT FALSE,
    "ispinned" bool DEFAULT FALSE,
    "isremoved" bool DEFAULT FALSE,

    -- AND LAST DATES
    "dtupdated" timestamp,
    "dtremoved" timestamp,
    "dtcreated" timestamp DEFAULT timezone('utc'::text, now()),

    -- DO NOT FORGET FOR FOREIGN KEYS
    CONSTRAINT ...
    CONSTRAINT ...

    PRIMARY KEY ("id")
);
````