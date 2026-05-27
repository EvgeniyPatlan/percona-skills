---
name: percona-javascript-stored-procedures
description: "Writing stored procedures and functions in JavaScript on Percona Server for MySQL via the js_lang component. Use when implementing stored routines whose logic is awkward in SQL/PSM, or when asked whether MySQL/Percona can run JavaScript in the database. IMPORTANT — this is a Percona Server feature (the js_lang component, V8-powered) and is TECH PREVIEW in 8.4 (not SLA-supported; test before production). It needs INSTALL COMPONENT plus the CREATE_JS_ROUTINE privilege; routines run in a sandbox — NO Node.js, require(), fetch, file/network I/O, triggers, or events. Code always runs in strict mode."
---

# Percona JavaScript Stored Procedures (`js_lang`)

*Last updated: 2026-05-27*

The `js_lang` component lets you write stored functions and procedures in JavaScript (V8 engine) instead of SQL/PSM — useful when the logic is easier in JS. It is **tech preview in Percona Server 8.4**: not yet production-ready, not covered by SLA. Routines run in a sandboxed V8 context with no I/O.

> **Version/status:** Percona Server **8.4**, **tech preview**. Requires `INSTALL COMPONENT` + the `CREATE_JS_ROUTINE` privilege.

## What LLMs Get Wrong

| What you might see | What's correct |
|---|---|
| Treating it as GA / production-ready | Tech preview in 8.4 — no SLA support; validate before relying on it |
| `CREATE TRIGGER … LANGUAGE JS` / a JS event | Not supported — JS can't back triggers or events, only procedures/functions |
| `require('x')`, `fetch()`, reading files in a routine | Sandboxed: no Node.js modules, no network, no file I/O, no DOM |
| Granting only `CREATE ROUTINE` | Also need `CREATE_JS_ROUTINE` (component privilege) — both are required |
| `CREATE FUNCTION … LANGUAGE JS` without installing the component | Run `INSTALL COMPONENT 'file://component_js_lang';` first |
| Naming a parameter `for`/`new`/etc. | Parameter names must be valid JS identifiers (not reserved words) |
| Expecting `await`/Promises to do real async work | Single-threaded with nothing to await — async is allowed but pointless |
| `console.table()` / `console.trace()` for output | No-ops here; use `console.log()` and read it via `JS_GET_CONSOLE_LOG()` |
| Disabling strict mode | Not possible — code always runs in strict mode |

## Install & Privileges

```sql
INSTALL COMPONENT 'file://component_js_lang';
GRANT CREATE_JS_ROUTINE ON *.* TO dev@'%';   -- plus the standard CREATE ROUTINE
```
`ALTER`/`DROP` of a JS routine don't need `CREATE_JS_ROUTINE`. `UNINSTALL COMPONENT` fails while any connection is executing a JS routine.

## Syntax

```sql
-- Function
CREATE FUNCTION f1(n INT) RETURNS INT LANGUAGE JS AS $$
    return n * 42;
$$;
SELECT f1(3);                       -- 126

-- Procedure with an OUT parameter
CREATE PROCEDURE p1(a INT, b INT, OUT r INT) LANGUAGE JS AS $$
    r = a * b;
$$;
CALL p1(6, 7, @res);  SELECT @res;  -- 42
```

## Type Conversions (SQL ↔ JS)

| SQL | JS |
|---|---|
| INT/TINYINT/…, FLOAT/DOUBLE, YEAR | `Number` |
| BIGINT, BIT(k>53) | `Number` if it fits, else `BigInt` |
| DECIMAL, DATE/TIME/DATETIME/TIMESTAMP | `String` |
| CHAR/VARCHAR/TEXT, ENUM, SET | `String` (converted to/from utf8mb4) |
| BINARY/VARBINARY/BLOB, GEOMETRY | `DataView` |
| JSON | `Object` (back via `JSON.stringify()`) |

SQL `NULL` ↔ JS `null`/`undefined` (both JS values map to SQL `NULL`).

## Limits, Memory & Debugging

- No triggers/events; no file/network I/O; always strict mode.
- Memory: `js_lang.max_mem_size` (default 8 MB soft limit, dynamic, 3 MB–1 GB). Avoid setting `js_lang.max_mem_size_hard_limit_factor` > 0 — on OOM it can terminate `mysqld`.
- Abort a runaway routine with `KILL QUERY` or the `MAX_EXECUTION_TIME` hint.
- Debug UDFs: `JS_GET_LAST_ERROR_INFO()` (message/line/column/stack), `JS_GET_MEMORY_USAGE_JSON()`, `JS_GET_CONSOLE_LOG()` / `JS_GET_CONSOLE_LOG_JSON()`, `JS_CLEAR_CONSOLE_LOG()`.
- Console API supports `log/info/warn/error/debug/assert/clear`, `count*`, `group*`, `time*`; `table/trace/dir/dirxml` are no-ops.

## Sources

- [js_lang overview](https://docs.percona.com/percona-server/8.4/js-lang-overview.html) · [install](https://docs.percona.com/percona-server/8.4/install-js-lang.html) · [procedures](https://docs.percona.com/percona-server/8.4/js-lang-procedures.html)
- [privileges](https://docs.percona.com/percona-server/8.4/js-lang-privileges.html) · [variables](https://docs.percona.com/percona-server/8.4/js-lang-variables.html) · [type conversions](https://docs.percona.com/percona-server/8.4/js-lang-type-conversions.html) · [Console API](https://docs.percona.com/percona-server/8.4/js-lang-console-api.html)

*Tech preview — confirm GA status and version at [docs.percona.com/percona-server](https://docs.percona.com/percona-server) before production use.*
