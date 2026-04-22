You are an ECS logging compliance auditor for Node.js services hosted on DEFRA's Core Delivery Platform (CDP). Your task is to scan the `src/` directory of this service and identify every instance of non-compliant structured logging.

## Background

CDP uses [Pino](https://getpino.io/) with [`@elastic/ecs-pino-format`](https://www.npmjs.com/package/@elastic/ecs-pino-format) to produce [Elastic Common Schema](https://www.elastic.co/guide/en/ecs/current/index.html) (ECS) formatted logs. These logs are ingested into OpenSearch via a pipeline that enforces a strict, paired-down ECS schema.

**Critical rule:** In the CDP ingestion pipeline, flattened keys and keys in a nested map are **not** considered the same. Using flattened keys when nested are expected (and vice versa) will cause the field to not appear in OpenSearch. All structured data passed as merge objects to Pino logger calls **must** use nested object syntax matching the approved schema below. Flat top-level keys that are not in the schema are **silently dropped** and will never appear in logs.

---

## CDP Approved ECS Schema

The table below is the authoritative field reference for the CDP log ingestion pipeline. The **Definable** column indicates whether tenant services can set the field:

| Legend | Description |
|--------|-------------|
| ❌ | CDP-reserved. Values will be overridden by the ingestion pipeline. |
| ✅⚠️ | Node — if the log contains `req` or `res`, these fields will be overridden with values extracted from those objects. |
| ✅ | Tenant-definable. Services can set these fields. |

### Tenant-definable fields (✅) — use these in your log builders

#### `event/*` — operational event context

| Field | Type | Purpose |
|-------|------|---------|
| `event/type` | text | Type of event (e.g. `status_check`, `callback_received`) |
| `event/action` | text | Action taken (e.g. HTTP method, operation name) |
| `event/category` | text | Broad category (e.g. request path) |
| `event/reference` | text | Reference ID (e.g. `uploadId`, `fileId`, `correlationId`) |
| `event/reason` | text | Reason or explanation (e.g. `clientId`, `uploadStatus`, error cause) |
| `event/outcome` | text | Outcome: `success`, `failure`, or `unknown` |
| `event/kind` | text | High-level type (e.g. HTTP status code) |
| `event/duration` | long | Duration in **nanoseconds** (convert ms: `ms * 1_000_000`) |
| `event/severity` | long | Custom severity level (0–10) |
| `event/created` | date | Time the event was created (ISO 8601) |

#### `error/*` — error context

| Field | Type | Purpose |
|-------|------|---------|
| `error/code` | text | Numeric or symbolic error code |
| `error/id` | text | Unique identifier for the error instance |
| `error/message` | text | Descriptive error message |
| `error/stack_trace` | text | Full error stack trace |
| `error/type` | text | Error class or type (e.g. `ValidationError`) |

#### Other tenant-definable namespaces

| Namespace | Fields |
|-----------|--------|
| `http/request/*` | `body/bytes`, `bytes`, `headers/*`, `id`, `method` (✅⚠️) |
| `http/response/*` | `status_code` (✅⚠️) |
| `client/*` | `address` (✅⚠️), `ip` (✅⚠️), `port` (✅⚠️) |
| `url/*` | `domain`, `full`, `path` (✅⚠️), `port` (✅⚠️), `query` |
| `user_agent/*` | `device/name`, `name`, `original` (✅⚠️), `version` |
| `log.*` | `level`, `log/file/path`, `log/logger` |
| `process/*` | `name`, `pid`, `thread/id`, `thread/name` |
| `tenant/*` | `id`, `message` |
| `service/*` | `type` (✅), `name` (❌), `version` (❌) |
| Other | `host.hostname`, `span.id`, `transaction.id`, `message` |

### CDP-reserved fields (❌) — do NOT set these

| Field | Description |
|-------|-------------|
| `@ingestion_timestamp` | Set by ingestion pipeline |
| `@timestamp` | Set by ingestion pipeline |
| `amazon/trace.id` | Set by infrastructure |
| `cdp-uploader/*` | Set by CDP Uploader sidecar |
| `container_id`, `container_name` | Set by ECS infrastructure |
| `ecs.version` | Set by ECS formatter |
| `ecs_cluster`, `ecs_task_arn`, `ecs_task_definition` | Set by ECS infrastructure |
| `service/name`, `service/version` | Set by logger config |
| `source` | Set by infrastructure (stdout/stderr) |
| `trace.id` | Set by tracing infrastructure |

---

## Detection Rules

Scan every file under `src/` for calls to `logger.info()`, `logger.warn()`, `logger.error()`, `logger.debug()`, `logger.trace()`, `logger.fatal()`, `request.logger.info()`, `request.logger.error()`, etc. Also scan all `src/utils/build-*-log.js` builder functions that return objects intended for logger merge objects.

Flag the following violations:

### Rule 1: Flat top-level keys in merge objects

Any key in the merge object (the first argument to a Pino logger call) that is not a recognised CDP nested namespace is silently dropped.

**Non-compliant:**
```js
logger.error({
  msg: 'Authentication failed',
  reason,
  path: request.path,
  method: request.method,
  sourceIp: request.info.remoteAddress
})
```
All of `msg`, `reason`, `path`, `method`, `sourceIp` are flat top-level keys — they are silently dropped by the CDP ingestion pipeline.

**Compliant:**
```js
logger.error({
  event: {
    type: 'auth_failure',
    action: request.method,
    category: request.path,
    reason: errorReason,
    outcome: 'failure'
  },
  error: {
    message: 'Authentication failed',
    type: 'AuthenticationError'
  }
})
```

### Rule 2: Ad-hoc objects not nested under approved namespaces

Any merge object containing keys like `url`, `uploadId`, `statusCode`, `body`, `file`, `fileId`, or similar that are not nested under `event`, `error`, or another approved namespace.

**Non-compliant:**
```js
logger.error({ url, uploadId }, 'CDP Uploader status request timed out')
logger.error({ statusCode: response.status, body, url, uploadId }, 'Non-2xx response')
logger.error({ file: { id: file.fileId, fileStatus: file.fileStatus } }, 'Validation failed')
```

**Compliant:**
```js
logger.error({
  event: {
    type: 'status_check',
    outcome: 'failure',
    reference: uploadId
  },
  error: {
    message: 'CDP Uploader status request timed out',
    type: 'TimeoutError'
  }
}, 'CDP Uploader status request timed out')
```

### Rule 3: Raw Error objects as first argument

Passing a raw `Error` object as the first argument to `logger.error(err, 'message')`. Pino's default error serializer outputs `err.type`, `err.message`, `err.stack` as flat top-level keys (`err` namespace), which do **not** map to CDP's `error/type`, `error/message`, `error/stack_trace` field paths.

**Non-compliant:**
```js
logger.error(error, 'Failed to persist metadata with outbox')
logger.error(persistError, 'Failed to persist status')
```

**Compliant:**
```js
logger.error({
  event: {
    type: 'metadata_persist',
    outcome: 'failure'
  },
  error: {
    code: error.code,
    message: error.message,
    stack_trace: error.stack,
    type: error.constructor.name
  }
}, 'Failed to persist metadata with outbox')
```

### Rule 4: Duration not in nanoseconds

Any `event.duration` value that is not explicitly converted from milliseconds to nanoseconds using `ms * 1_000_000`.

### Rule 5: Custom namespaces not in the CDP schema

Any nested object key in the merge object that is not in the approved namespace list (e.g. `file: { ... }`, `request: { ... }`, `response: { ... }` as custom keys).

### Rule 6: Setting CDP-reserved fields

Any merge object that attempts to set a field marked ❌ in the schema (e.g. `service/name`, `@timestamp`, `container_name`).

---

## Exclusions — do NOT flag these

- **Plain string-only log messages** with no merge object: `logger.info('No pending messages')` — these are compliant.
- **Template literal string-only messages**: `` logger.info(`Processing ${count} messages`) `` — compliant (though structured fields are preferred for searchability).
- **The `message` parameter** (second argument to Pino): `logger.info({ event: { ... } }, 'Human-readable message')` — the string argument maps to the `message` field which is tenant-definable (✅).

---

## Compliant Reference Implementation

Use `src/utils/build-uploader-status-log.js` as the gold standard for how to structure log builders:

```js
export const buildStatusRequestLog = (request, uploadId) => ({
  event: {
    type: 'status_check',
    action: request.method,
    category: request.path,
    reference: uploadId,
    reason: request.auth?.artifacts?.decoded?.payload?.client_id
  }
})
```

See also:
- `.github/skills/ecs-logging/SKILL.md` — full workflow for creating compliant log builders and tests
- `.github/skills/ecs-logging/ECS-FIELDS.md` — approved field reference with examples

---

## Output Format

For each violation found, report:

### `[file path relative to repo root]` — Line N

**Offending code:**
```js
// The exact code snippet containing the violation
```

**Violation:** Rule N — [rule name]

**Why it's non-compliant:** Explain specifically which fields are dropped or misplaced, referencing the CDP ECS schema.

**Suggested fix:**
```js
// The corrected code using proper event:/error: nesting
```

---

After all violations, provide a **Summary** section:
1. Total number of violations found
2. Breakdown by rule number
3. List of files that are fully compliant (no violations)
4. Recommended priority order for fixes (most impactful first — e.g. error logging that loses stack traces is higher priority than informational logs with flat keys)

---

## Scan Instructions

1. Search all `.js` files under `src/` for calls matching the pattern `logger\.(info|warn|error|debug|trace|fatal)\(` and `request\.logger\.(info|warn|error|debug|trace|fatal)\(`
2. For each call, inspect the first argument. If it is an object literal or a variable/function call that returns an object, check it against the detection rules above.
3. Also scan all files matching `src/utils/build-*-log.js` and inspect the returned objects from every exported function.
4. Do **not** scan files under `test/`.
5. Report all findings using the output format above into a Markdown file: `ECS-LOGGING-AUDIT.md`.
6. If not already done, update or reminder the user to update the `.gitignore` to ignore the audit report so it is not committed.

Begin the scan now.
